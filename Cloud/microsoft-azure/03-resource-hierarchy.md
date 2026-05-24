# Chapter 3: Resource Hierarchy & Management (Azure)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Resource Hierarchy (Deep Dive)](#part-1-azure-resource-hierarchy-deep-dive)
- [Part 2: Resource Groups (Deep Dive)](#part-2-resource-groups-deep-dive)
- [Part 3: How Resources Are Identified (Resource IDs)](#part-3-how-resources-are-identified-resource-ids)
- [Part 4: Resource Naming Conventions](#part-4-resource-naming-conventions)
- [Part 5: Tagging Strategy](#part-5-tagging-strategy)
- [Part 6: Resource Locks](#part-6-resource-locks)
- [Part 7: Azure Resource Graph (Query All Resources)](#part-7-azure-resource-graph-query-all-resources)
- [Part 8: Service Quotas & Limits](#part-8-service-quotas--limits)
- [Part 9: Moving Resources](#part-9-moving-resources)
- [Part 10: Real-World Resource Management Patterns](#part-10-real-world-resource-management-patterns)
- [Quick Reference: Console Navigation](#quick-reference-console-navigation)
- [What's Next?](#whats-next)

---

## Overview

This chapter covers how resources are organized, identified, and managed within Azure — including Resource Groups, resource IDs, tagging strategies, resource locks, Azure Resource Graph, and real-world management patterns.

```
What you'll learn:
├── Azure resource hierarchy (complete picture)
├── Resource Groups (mandatory container)
├── How resources are identified (Resource IDs)
├── Resource naming conventions
├── Tagging strategy (critical for companies)
├── Resource Locks (prevent accidental deletion)
├── Azure Resource Graph (query all resources)
├── Azure Policy for tag enforcement
├── Service limits & quotas
├── Resource lifecycle management
└── Real-world patterns
```

---

## Part 1: Azure Resource Hierarchy (Deep Dive)

### The Complete Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│              AZURE RESOURCE HIERARCHY - COMPLETE                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Level 0: ENTRA ID TENANT (Identity Foundation)                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Tenant: techcorp.onmicrosoft.com                             │    │
│  │ Tenant ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx              │    │
│  │                                                               │    │
│  │ Contains: Users, Groups, App Registrations, Roles            │    │
│  │ NOT part of resource hierarchy, but foundation for identity  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │ Trust relationship                                           │
│       ▼                                                              │
│  Level 1: MANAGEMENT GROUPS                                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Tenant Root Group (auto-created)                             │    │
│  │                                                               │    │
│  │ What lives here:                                             │    │
│  │ ├── Azure Policy assignments                                │    │
│  │ ├── RBAC role assignments                                   │    │
│  │ └── Compliance scope                                        │    │
│  │                                                               │    │
│  │ ├── MG: Platform                                            │    │
│  │ ├── MG: Landing Zones                                       │    │
│  │ │   ├── MG: Production                                     │    │
│  │ │   └── MG: Non-Production                                 │    │
│  │ └── MG: Sandbox                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                              │
│       ▼                                                              │
│  Level 2: SUBSCRIPTIONS                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Subscription: Prod-WebApp                                    │    │
│  │ Subscription ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx        │    │
│  │                                                               │    │
│  │ What lives here:                                             │    │
│  │ ├── Billing boundary                                        │    │
│  │ ├── Azure Policy (inherited + own)                          │    │
│  │ ├── RBAC (inherited + own)                                  │    │
│  │ ├── Resource providers (registered)                         │    │
│  │ └── Service quotas                                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                              │
│       ▼                                                              │
│  Level 3: RESOURCE GROUPS (mandatory container)                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Resource Group: rg-prod-webapp-compute                       │    │
│  │ Location: Central India (metadata location, not a constraint)│    │
│  │                                                               │    │
│  │ What lives here:                                             │    │
│  │ ├── Logical container for related resources                 │    │
│  │ ├── RBAC (inherited + own)                                  │    │
│  │ ├── Tags (can be different from resource tags)              │    │
│  │ ├── Locks (inherited by resources)                          │    │
│  │ └── Resources can be from ANY region                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                              │
│       ▼                                                              │
│  Level 4: RESOURCES                                                  │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Resources inside the Resource Group:                         │    │
│  │ ├── Virtual Machine: prod-web-01 (Central India)            │    │
│  │ ├── Managed Disk: prod-web-01-osdisk (Central India)        │    │
│  │ ├── NIC: prod-web-01-nic (Central India)                    │    │
│  │ ├── Public IP: prod-web-01-pip (Central India)              │    │
│  │ └── NSG: prod-web-nsg (Central India)                       │    │
│  │                                                               │    │
│  │ Each resource has:                                           │    │
│  │ ├── Resource ID (unique identifier)                         │    │
│  │ ├── Tags                                                    │    │
│  │ ├── Location (region)                                       │    │
│  │ ├── SKU/Tier                                                │    │
│  │ └── Resource-specific properties                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  INHERITANCE FLOW:                                                   │
│  MG → Subscription → Resource Group → Resource                      │
│  ├── RBAC: Inherited (additive, union of all levels)                │
│  ├── Azure Policy: Inherited (all policies apply)                   │
│  ├── Locks: Inherited (RG lock applies to all resources)            │
│  └── Tags: NOT inherited! (must be applied at each level)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Inheritance Rules

```
⚠️ CRITICAL RULES:

1. RBAC INHERITS downward (and is additive):
   ├── MG: Contributor → applies to all subs, RGs, resources below
   ├── Cannot REMOVE inherited access at lower level
   └── Can only ADD more access at lower levels

2. AZURE POLICY INHERITS downward:
   ├── Policy at MG → all subs below must comply
   ├── Cannot exempt lower levels (unless explicitly excluded)
   └── Policies merge (all applicable policies are evaluated)

3. TAGS DO NOT INHERIT:
   ├── RG tags are NOT copied to resources inside it
   ├── Subscription tags are NOT copied to RGs
   ├── Must tag each level independently
   └── 💡 Use Azure Policy "Inherit tag from RG" to auto-copy
   
4. LOCKS INHERIT downward:
   ├── Lock on RG → all resources inside are locked
   └── Cannot override lock at resource level (must remove from RG)
```

---

## Part 2: Resource Groups (Deep Dive)

### What is a Resource Group?

A Resource Group (RG) is a **mandatory** logical container for Azure resources. Every resource must be in exactly one RG.

```
┌─────────────────────────────────────────────────────────────────┐
│                     RESOURCE GROUP                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Properties:                                                      │
│ ├── Name: rg-prod-webapp-compute                                │
│ ├── Subscription: Prod-WebApp                                   │
│ ├── Location: centralindia (metadata only!)                     │
│ ├── Tags: {Environment: production, Team: backend}              │
│ └── Resource Group ID: /subscriptions/xxx/resourceGroups/rg-xxx │
│                                                                   │
│ Rules:                                                           │
│ ├── ⚠️ Every resource MUST be in a RG                           │
│ ├── A resource can be in only ONE RG                            │
│ ├── RG can contain resources from DIFFERENT regions             │
│ ├── Resources can be moved between RGs (with limitations)       │
│ ├── Deleting a RG deletes ALL resources inside                  │
│ ├── RG location ≠ resource location (it's just metadata)       │
│ ├── RG name must be unique within a subscription                │
│ └── Max 980 RGs per subscription                                │
│                                                                   │
│ 💡 The RG location determines where the metadata is stored      │
│    (important for compliance — metadata includes resource names, │
│     types, and configuration)                                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Resource Group

```
Portal → Resource Groups → Create

┌─────────────────────────────────────────────────────────────────┐
│                  CREATE RESOURCE GROUP                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Basics tab:                                                      │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Subscription:    [Prod-WebApp ▼]                          │   │
│ │                                                            │   │
│ │ Resource group:  [rg-prod-webapp-compute]                  │   │
│ │                  Rules: 1-90 chars, alphanumeric,          │   │
│ │                  underscores, hyphens, periods, parens     │   │
│ │                  Cannot end with period                    │   │
│ │                                                            │   │
│ │ Region:          [Central India ▼]                         │   │
│ │                  (Where RG metadata is stored)             │   │
│ │                  (Resources inside can be in ANY region)   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Tags tab:                                                        │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Name          │ Value                                      │   │
│ ├───────────────────────────────────────────────────────────┤   │
│ │ Environment   │ production                                │   │
│ │ Team          │ backend                                   │   │
│ │ Application   │ webapp                                    │   │
│ │ CostCenter    │ CC-1234                                   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Review + create] → [Create]                                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  az group create \
    --name rg-prod-webapp-compute \
    --location centralindia \
    --tags Environment=production Team=backend Application=webapp

PowerShell:
  New-AzResourceGroup `
    -Name "rg-prod-webapp-compute" `
    -Location "centralindia" `
    -Tag @{Environment="production"; Team="backend"}
```

### Resource Group Design Strategies

```
Strategy 1: By Application Tier (Most Common)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subscription: Prod-WebApp
├── rg-prod-webapp-compute     (VMs, AKS, App Service)
├── rg-prod-webapp-data        (SQL, Cosmos DB, Storage)
├── rg-prod-webapp-networking  (VNet, NSG, Load Balancer)
└── rg-prod-webapp-monitoring  (Log Analytics, App Insights)

✅ Good for: Clear separation of compute/data/network teams
✅ Good for: Different RBAC per tier (DBAs → data RG only)

Strategy 2: By Lifecycle (Deploy/Delete together)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subscription: Prod-WebApp
├── rg-prod-webapp-shared      (VNet, DNS — long-lived)
├── rg-prod-webapp-v2          (Current app version)
└── rg-prod-webapp-v1          (Old version — delete when done)

✅ Good for: Blue-green deployments
✅ Good for: Easy cleanup (delete entire RG)

Strategy 3: By Application (Each app gets its own RGs)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subscription: Production
├── rg-prod-orders-api
├── rg-prod-users-api
├── rg-prod-frontend
├── rg-prod-shared-networking
└── rg-prod-shared-data

✅ Good for: Microservices architecture
✅ Good for: Per-service cost tracking

Strategy 4: By Environment + Application
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subscription: Production
├── rg-prod-app1
├── rg-prod-app2

Subscription: Development  
├── rg-dev-app1
├── rg-dev-app2

✅ Good for: Simple structure with separate subs per env
```

---

## Part 3: How Resources Are Identified (Resource IDs)

### Azure Resource IDs

Every Azure resource has a unique **Resource ID** that follows a hierarchical path format.

```
Resource ID Format:
━━━━━━━━━━━━━━━━━━━
/subscriptions/{subscriptionId}/resourceGroups/{rgName}/providers/{resourceProvider}/{resourceType}/{resourceName}

Parts:
┌──────────────────────────────────────────────────────────────────┐
│ Part                │ Example                                     │
├──────────────────────────────────────────────────────────────────┤
│ /subscriptions/     │ Fixed prefix                                │
│ {subscriptionId}    │ a1b2c3d4-e5f6-g7h8-i9j0-k1l2m3n4o5p6     │
│ /resourceGroups/    │ Fixed prefix                                │
│ {rgName}            │ rg-prod-webapp-compute                     │
│ /providers/         │ Fixed prefix                                │
│ {resourceProvider}  │ Microsoft.Compute                           │
│ {resourceType}      │ virtualMachines                             │
│ {resourceName}      │ prod-web-01                                │
└──────────────────────────────────────────────────────────────────┘
```

### Resource ID Examples

```
Virtual Machine:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-compute/providers/Microsoft.Compute/virtualMachines/prod-web-01

Storage Account:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-data/providers/Microsoft.Storage/storageAccounts/techcorpprodstorage

SQL Database:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-data/providers/Microsoft.Sql/servers/prod-sql-server/databases/users-db

App Service:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-compute/providers/Microsoft.Web/sites/prod-webapp

AKS Cluster:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-compute/providers/Microsoft.ContainerService/managedClusters/prod-aks

Key Vault:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-security/providers/Microsoft.KeyVault/vaults/prod-keyvault

VNet:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-network/providers/Microsoft.Network/virtualNetworks/prod-vnet

NSG:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-network/providers/Microsoft.Network/networkSecurityGroups/prod-web-nsg

Load Balancer:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-network/providers/Microsoft.Network/loadBalancers/prod-lb

Cosmos DB:
  /subscriptions/a1b2c3d4/resourceGroups/rg-prod-data/providers/Microsoft.DocumentDB/databaseAccounts/prod-cosmos

Log Analytics Workspace:
  /subscriptions/a1b2c3d4/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/prod-logs
```

### Where Resource IDs are Used

```
Resource IDs appear in:
├── Portal URL (when viewing a resource)
├── Activity Log entries
├── Azure Policy compliance reports
├── RBAC role assignments (scope)
├── ARM templates / Bicep (resource references)
├── CLI commands: az resource show --id "/subscriptions/..."
├── Diagnostic settings (source resource)
├── Azure Monitor alerts (target resource)
└── Terraform (import existing resources)

Useful CLI commands:
  # Get resource ID
  az vm show --name prod-web-01 -g rg-prod-compute --query id -o tsv
  
  # Describe any resource by ID
  az resource show --ids "/subscriptions/a1b2c3d4/..."
  
  # List all resources in a RG with IDs
  az resource list -g rg-prod-compute --query "[].{Name:name, Type:type, ID:id}" -o table
```

---

## Part 4: Resource Naming Conventions

### Azure Naming Rules (Vary by Resource!)

```
⚠️ Each Azure resource type has DIFFERENT naming rules:

┌──────────────────────────────────────────────────────────────────────┐
│ Resource Type      │ Length │ Valid Characters        │ Unique Scope  │
├──────────────────────────────────────────────────────────────────────┤
│ Resource Group     │ 1-90  │ Alpha, digits, _-.(/)   │ Subscription  │
│ Storage Account    │ 3-24  │ Lowercase + digits ONLY │ GLOBAL        │
│ VM                 │ 1-64  │ Alpha, digits, -, _     │ Resource Group│
│ App Service        │ 2-60  │ Alpha, digits, -        │ GLOBAL (.azurewebsites.net)│
│ SQL Server         │ 1-63  │ Lowercase, digits, -    │ GLOBAL (.database.windows.net)│
│ Key Vault          │ 3-24  │ Alpha, digits, -        │ GLOBAL (.vault.azure.net)│
│ AKS Cluster        │ 1-63  │ Alpha, digits, -, _     │ Resource Group│
│ Cosmos DB Account  │ 3-44  │ Lowercase, digits, -    │ GLOBAL        │
│ VNet               │ 2-64  │ Alpha, digits, -, _, .  │ Resource Group│
│ NSG                │ 1-80  │ Alpha, digits, -, _, .  │ Resource Group│
│ Public IP          │ 1-80  │ Alpha, digits, -, _, .  │ Resource Group│
│ Load Balancer      │ 1-80  │ Alpha, digits, -, _, .  │ Resource Group│
│ Function App       │ 2-60  │ Alpha, digits, -        │ GLOBAL        │
│ Container Registry │ 5-50  │ Alpha + digits ONLY     │ GLOBAL (.azurecr.io)│
└──────────────────────────────────────────────────────────────────────┘

⚠️ GLOBALLY UNIQUE resources: Storage Account, App Service, SQL Server,
   Key Vault, Cosmos DB, Container Registry, Function App
   → These need a unique prefix (company name + env + random)
```

### Recommended Naming Convention

```
Microsoft's recommended abbreviation-based convention:
  <resource-type-abbr>-<workload/app>-<environment>-<region>-<instance>

Resource type abbreviations (Microsoft official):
┌────────────────────────────────────────────────────────┐
│ Resource              │ Abbreviation │ Example          │
├────────────────────────────────────────────────────────┤
│ Resource Group        │ rg           │ rg-webapp-prod-ci│
│ Virtual Network       │ vnet         │ vnet-prod-ci-001 │
│ Subnet                │ snet         │ snet-web-prod    │
│ NSG                   │ nsg          │ nsg-web-prod     │
│ Public IP             │ pip          │ pip-web-prod-ci  │
│ Load Balancer         │ lbi / lbe    │ lbe-web-prod-ci  │
│ App Gateway           │ agw          │ agw-web-prod-ci  │
│ Virtual Machine       │ vm           │ vm-web-prod-01   │
│ Storage Account       │ st           │ sttcproddata001  │
│ App Service           │ app          │ app-orders-prod  │
│ Function App          │ func         │ func-process-prod│
│ SQL Server            │ sql          │ sql-tc-prod-ci   │
│ SQL Database          │ sqldb        │ sqldb-users-prod │
│ Cosmos DB             │ cosmos       │ cosmos-tc-prod   │
│ Key Vault             │ kv           │ kv-tc-prod-ci    │
│ AKS                   │ aks          │ aks-backend-prod │
│ Container Registry    │ cr           │ crtcprod001      │
│ Log Analytics         │ log          │ log-prod-ci      │
│ App Insights          │ appi         │ appi-webapp-prod │
│ Service Bus           │ sb           │ sb-tc-prod       │
│ Event Hub             │ evh          │ evh-tc-prod      │
│ Front Door            │ afd          │ afd-tc-prod      │
│ Azure Firewall        │ afw          │ afw-hub-prod-ci  │
│ Managed Identity      │ id           │ id-webapp-prod   │
│ Private Endpoint      │ pep          │ pep-sql-prod     │
└────────────────────────────────────────────────────────┘

Abbreviations for environments: prod, stg, dev, sand
Abbreviations for regions: ci (Central India), eu (East US), we (West Europe)
```

---

## Part 5: Tagging Strategy

### What are Tags in Azure?

Tags are **key-value pairs** applied to resources, resource groups, and subscriptions for organization, cost management, and automation.

```
Tag structure:
┌──────────────────────────────────────────────────┐
│ Name (Key)          │ Value                       │
├──────────────────────────────────────────────────┤
│ Environment         │ Production                  │
│ Team                │ Backend                     │
│ Application         │ OrderService                │
│ CostCenter          │ CC-1234                     │
│ Owner               │ john.doe@company.com        │
│ ManagedBy           │ Terraform                   │
│ CreatedDate         │ 2026-05-16                  │
│ DataClassification  │ Confidential                │
└──────────────────────────────────────────────────┘

Azure tag constraints:
├── Max 50 tags per resource/RG/subscription
├── Tag name: max 512 characters
├── Tag value: max 256 characters
├── Tag names: case-insensitive for operations (but stored as entered)
├── Tag values: case-sensitive
├── NOT inherited from RG to resources (must apply separately)
├── Storage accounts: max 128 char names, 256 char values
└── Can use Unicode characters
```

### Tag Categories

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TAG CATEGORIES                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. MANDATORY TAGS (enforce via Azure Policy):                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Tag               │ Values               │ Purpose            │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ Environment       │ Production/Staging/Dev│ Environment ID     │    │
│  │ Application       │ app-name              │ App association     │    │
│  │ Owner             │ email@company.com     │ Accountability     │    │
│  │ CostCenter        │ CC-XXXX               │ Billing chargeback │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  2. RECOMMENDED TAGS:                                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Team              │ backend/frontend/data │ Team ownership    │    │
│  │ ManagedBy         │ terraform/bicep/manual│ IaC tracking      │    │
│  │ CreatedDate       │ 2026-05-16           │ Age tracking       │    │
│  │ BusinessUnit      │ retail/finance       │ BU attribution     │    │
│  │ Criticality       │ high/medium/low      │ Priority level     │    │
│  │ DR                │ essential/important   │ DR priority        │    │
│  │ DataClassification│ public/internal/conf  │ Data sensitivity   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  3. AUTOMATION TAGS:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ AutoShutdown      │ true/false           │ Stop at night     │    │
│  │ StartupSchedule   │ 08:00-IST            │ When to start     │    │
│  │ BackupPolicy      │ daily/weekly         │ Backup schedule   │    │
│  │ PatchWindow       │ Sun-02:00-IST        │ Maintenance time  │    │
│  │ ExpiresOn         │ 2026-08-16           │ Temp resources    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Enforcing Tags with Azure Policy

```
Azure Policy provides built-in tag enforcement policies:

1. "Require a tag and its value on resources"
   Effect: Deny (block creation without tag)
   
2. "Require a tag on resources"
   Effect: Deny (block creation without tag key, any value OK)

3. "Inherit a tag from the resource group"
   Effect: Modify (auto-copy RG tag to resources)
   💡 THIS IS VERY USEFUL! Solves the "tags don't inherit" problem

4. "Inherit a tag from the subscription"
   Effect: Modify (auto-copy subscription tag to RGs)

5. "Add a tag to resources"
   Effect: Modify (auto-add a specific tag to all resources)

Example: Enforce "Environment" tag on all resources
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Portal → Policy → Assign → "Require a tag and its value on resources"

  Scope: MG: Production (applies to all production subscriptions)
  
  Parameters:
    Tag Name: Environment
    Tag Value: Production
  
  Effect: Deny
  
  Non-compliance message: 
  "All resources in Production must have the tag 
   Environment=Production. Contact platform-team for help."

Result: Any resource created without Environment=Production → BLOCKED

Example: Auto-inherit tags from Resource Group
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Portal → Policy → Assign → "Inherit a tag from the resource group"

  Parameters:
    Tag Name: CostCenter
  
  Effect: Modify
  
  ✅ Create remediation task: Yes
  ✅ Create managed identity: Yes

Result: 
├── New resources auto-get CostCenter tag from their RG
├── Remediation task retroactively tags existing resources
└── No manual tagging needed for CostCenter!
```

### How to Apply Tags

```
Portal: Any resource → Tags (left menu) → Add tags

CLI:
  # Tag a resource group
  az group update -n rg-prod-compute \
    --tags Environment=Production Team=Backend CostCenter=CC-1234

  # Tag a VM
  az vm update -n prod-web-01 -g rg-prod-compute \
    --set tags.Environment=Production tags.Team=Backend

  # Tag any resource by ID
  az tag update --resource-id "/subscriptions/xxx/..." \
    --operation merge \
    --tags Environment=Production

  # Bulk tag — all resources in a RG
  az resource list -g rg-prod-compute --query "[].id" -o tsv | \
    xargs -I {} az tag update --resource-id {} \
    --operation merge --tags Environment=Production

PowerShell:
  # Tag a resource group
  Set-AzResourceGroup -Name "rg-prod-compute" `
    -Tag @{Environment="Production"; Team="Backend"}

  # Merge tags (don't overwrite existing)
  $rg = Get-AzResourceGroup -Name "rg-prod-compute"
  $rg.Tags.Add("CostCenter", "CC-1234")
  Set-AzResourceGroup -Tag $rg.Tags -Name "rg-prod-compute"

Bicep:
  resource vm 'Microsoft.Compute/virtualMachines@2023-09-01' = {
    name: 'prod-web-01'
    location: resourceGroup().location
    tags: {
      Environment: 'Production'
      Team: 'Backend'
      Application: 'WebApp'
      CostCenter: 'CC-1234'
      ManagedBy: 'Bicep'
    }
    properties: { ... }
  }
```

---

## Part 6: Resource Locks

### What are Resource Locks?

Resource Locks prevent accidental deletion or modification of critical resources.

```
┌─────────────────────────────────────────────────────────────────┐
│                     RESOURCE LOCKS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Lock Types:                                                     │
│  ┌───────────────┬────────────────────────────────────────────┐ │
│  │ CanNotDelete  │ Can read & modify, but CANNOT delete       │ │
│  │               │ The resource is protected from deletion    │ │
│  │               │ Users can still update/modify the resource │ │
│  ├───────────────┼────────────────────────────────────────────┤ │
│  │ ReadOnly      │ Can read only, CANNOT modify or delete     │ │
│  │               │ Similar to Reader role (even for admins!)  │ │
│  │               │ ⚠️ Very restrictive — use with caution!    │ │
│  └───────────────┴────────────────────────────────────────────┘ │
│                                                                   │
│  Can be applied at:                                              │
│  ├── Subscription level → all RGs & resources locked           │
│  ├── Resource Group level → all resources in RG locked         │
│  └── Resource level → only that specific resource              │
│                                                                   │
│  ⚠️ Locks apply to ALL users, including admins!                  │
│     Even Owner role cannot delete a locked resource.            │
│     Must remove the lock first (requires lock management permission)│
│                                                                   │
│  Who can manage locks:                                           │
│  ├── Owner role                                                 │
│  ├── User Access Administrator role                             │
│  └── Custom role with Microsoft.Authorization/locks/* permission│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating Locks

```
Portal: Any resource or RG → Locks (left menu) → Add

┌─────────────────────────────────────────────────────────────────┐
│ Lock name:  [do-not-delete-production-db]                        │
│ Lock type:  ○ Read-only  ○ Delete                               │
│ Notes:      [Critical production database. Contact DBA team     │
│              before removing this lock.]                         │
│                                                                   │
│ [OK]                                                             │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Lock a resource group (prevent deletion)
  az lock create \
    --name "do-not-delete" \
    --resource-group rg-prod-data \
    --lock-type CanNotDelete \
    --notes "Production data — contact DBA team"

  # Lock a specific resource
  az lock create \
    --name "protect-prod-db" \
    --resource-group rg-prod-data \
    --resource-name prod-sql-server \
    --resource-type Microsoft.Sql/servers \
    --lock-type CanNotDelete

  # List locks
  az lock list --resource-group rg-prod-data -o table

  # Remove lock (when you need to make changes)
  az lock delete --name "do-not-delete" --resource-group rg-prod-data

Best practice: Lock critical resources:
├── Production databases (CanNotDelete)
├── Hub VNet (CanNotDelete)
├── Key Vaults (CanNotDelete)
├── DNS zones (CanNotDelete)
├── Log Analytics workspace (CanNotDelete)
└── Shared services RGs (CanNotDelete)
```

---

## Part 7: Azure Resource Graph (Query All Resources)

### What is Resource Graph?

Azure Resource Graph provides fast, efficient querying of resources across ALL subscriptions using Kusto Query Language (KQL).

```
Portal → Resource Graph Explorer

┌─────────────────────────────────────────────────────────────────┐
│                   RESOURCE GRAPH EXPLORER                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Query: (KQL - Kusto Query Language)                              │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Resources                                                  │   │
│ │ | where type == "microsoft.compute/virtualmachines"       │   │
│ │ | where tags.Environment == "Production"                  │   │
│ │ | project name, location, resourceGroup,                 │   │
│ │          properties.hardwareProfile.vmSize                │   │
│ │ | order by name asc                                      │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Run query]                                                      │
│                                                                   │
│ Results: (Instant — much faster than ARM API!)                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Name          │ Location     │ RG              │ Size     │   │
│ ├───────────────────────────────────────────────────────────┤   │
│ │ prod-web-01  │ centralindia │ rg-prod-compute │ Standard_D2s│ │
│ │ prod-web-02  │ centralindia │ rg-prod-compute │ Standard_D2s│ │
│ │ prod-api-01  │ centralindia │ rg-prod-compute │ Standard_D4s│ │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Useful Resource Graph Queries

```
-- All resources without required tags
Resources
| where isempty(tags.Environment) or isempty(tags.CostCenter)
| project name, type, resourceGroup, subscriptionId
| order by type asc

-- Count resources by type
Resources
| summarize count() by type
| order by count_ desc
| take 20

-- VMs by size (find over-provisioned)
Resources
| where type == "microsoft.compute/virtualmachines"
| extend vmSize = tostring(properties.hardwareProfile.vmSize)
| summarize count() by vmSize
| order by count_ desc

-- Resources by region
Resources
| summarize count() by location
| order by count_ desc

-- Find resources with public IPs
Resources
| where type == "microsoft.network/publicipaddresses"
| extend ipAddress = properties.ipAddress
| project name, ipAddress, resourceGroup

-- All resources in a specific resource group
Resources
| where resourceGroup == "rg-prod-compute"
| project name, type, location, tags

-- Resources missing locks (not locked)
Resources
| join kind=leftouter (
    ResourceContainers
    | where type == "microsoft.resources/subscriptions/resourcegroups"
    | project rgName=name, rgId=id
) on $left.resourceGroup == $right.rgName
| project name, type, resourceGroup

-- Cost center breakdown
Resources
| where isnotempty(tags.CostCenter)
| summarize count() by tostring(tags.CostCenter)
| order by count_ desc
```

---

## Part 8: Service Quotas & Limits

```
Console → Subscriptions → Select → Usage + quotas

┌─────────────────────────────────────────────────────────────────┐
│                   COMMON AZURE QUOTAS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Compute:                                                         │
│ ├── Total Regional vCPUs: 20-350 (per region, per sub)         │
│ ├── Per VM family: 10-350 vCPUs                                │
│ ├── Availability Sets: 2,500 per subscription                  │
│ ├── VM Scale Sets: 2,500 per region                            │
│ └── Managed Disks per subscription: 50,000                     │
│                                                                   │
│ Networking:                                                      │
│ ├── VNets per subscription: 1,000                              │
│ ├── Subnets per VNet: 3,000                                   │
│ ├── NSGs per subscription: 5,000                               │
│ ├── NSG rules per NSG: 1,000                                  │
│ ├── Public IPs: 1,000 per subscription                         │
│ ├── Load Balancers: 1,000 per subscription                     │
│ └── Network Interfaces: 65,536 per subscription                │
│                                                                   │
│ Storage:                                                         │
│ ├── Storage accounts per region: 250                           │
│ ├── Max storage per account: 5 PiB                             │
│ ├── Max blob size: ~190.7 TiB (block blob)                     │
│ └── Max file share: 100 TiB                                   │
│                                                                   │
│ Databases:                                                       │
│ ├── SQL servers per subscription: 150                          │
│ ├── Databases per server: 5,000                                │
│ ├── Cosmos DB account per subscription: 50                     │
│ └── Redis Cache instances: 150 per subscription                │
│                                                                   │
│ App Service:                                                     │
│ ├── App Service Plans: 100 per RG                              │
│ ├── Apps per plan: varies by tier (100 for Standard)           │
│ └── Deployment slots: 20 (Standard+)                           │
│                                                                   │
│ AKS:                                                             │
│ ├── Clusters per subscription: 5,000                           │
│ ├── Nodes per cluster: 5,000                                  │
│ ├── Pods per node: 250                                        │
│ └── Node pools per cluster: 100                               │
│                                                                   │
│ Subscription limits:                                             │
│ ├── Resource Groups: 980                                       │
│ ├── Deployments per RG (history): 800                          │
│ ├── Tags per resource: 50                                     │
│ └── Role assignments per subscription: 4,000                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Requesting quota increase:
Portal → Subscriptions → Usage + quotas → Select → Request increase
→ Most requests auto-approved within minutes
→ Larger requests may need review (1-3 business days)

CLI:
  az quota update --resource-name "StandardDSv3Family" \
    --scope "/subscriptions/xxx/providers/Microsoft.Compute/locations/centralindia" \
    --limit-object value=100 limit-object-type=LimitValue
```

---

## Part 9: Moving Resources

### Moving Resources Between Resource Groups

```
Portal → Resource → Move → Move to another resource group

┌─────────────────────────────────────────────────────────────────┐
│                    MOVE RESOURCES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Move types:                                                      │
│ ├── Between Resource Groups (same subscription)                 │
│ ├── Between Subscriptions (same tenant)                         │
│ └── Between Regions (limited — most need recreate)              │
│                                                                   │
│ Resources that CAN be moved:                                     │
│ ├── Virtual Machines (+ disks, NICs, public IPs)               │
│ ├── Storage Accounts                                            │
│ ├── VNets (most scenarios)                                      │
│ ├── App Service Plans + Web Apps                                │
│ ├── SQL Servers + Databases                                     │
│ ├── Key Vault                                                   │
│ └── Most PaaS services                                          │
│                                                                   │
│ Resources that CANNOT be moved:                                  │
│ ├── Azure Active Directory resources                            │
│ ├── Azure Backup vault (with data)                              │
│ ├── Some networking resources (ExpressRoute circuits)           │
│ ├── Azure Databricks workspaces                                 │
│ └── Resources with locks (must remove lock first)               │
│                                                                   │
│ ⚠️ Moving changes the Resource ID!                               │
│    (resourceGroup part changes)                                  │
│    Update any references/scripts that use the old ID.           │
│                                                                   │
│ ⚠️ Move is NOT instantaneous — can take minutes to hours         │
│    Source and target RGs are locked during move.                 │
│                                                                   │
│ CLI:                                                             │
│ az resource move \                                               │
│   --destination-group rg-new-location \                          │
│   --ids "/subscriptions/xxx/.../vm-name"                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Real-World Resource Management Patterns

### Pattern 1: Tag-Based Cost Allocation

```
Company requirement: "Show spending per team per environment"

Implementation:
1. Define mandatory tags via Azure Policy (Deny effect):
   ├── Environment (Production/Staging/Development)
   ├── Team (backend/frontend/data/platform)
   ├── Application (app name)
   └── CostCenter (CC-XXXX)

2. Auto-inherit tags from RG using Azure Policy (Modify effect):
   → Reduces manual tagging burden

3. Cost Analysis:
   Portal → Cost Management → Cost analysis
   → Group by: Tag (Team)
   → Filter: Tag (Environment = Production)
   → Shows per-team production costs

4. Budget per team:
   Portal → Cost Management → Budgets
   → Filter by Tag: Team = backend
   → Set $5,000/month limit → Alert at 80%, 100%

5. Monthly report:
   Export cost data → Power BI dashboard → Share with leadership
```

### Pattern 2: Governance with Policy + Locks

```
Production environment governance:

Azure Policy (at MG: Production):
├── Deny: Resources without required tags
├── Deny: Resources outside approved regions
├── Deny: VM sizes not in approved list
├── Modify: Auto-add Environment=Production tag
├── Audit: Resources without diagnostics settings
└── DeployIfNotExists: Deploy Microsoft Defender agent

Resource Locks (on critical RGs):
├── rg-prod-data: CanNotDelete lock
├── rg-prod-networking: CanNotDelete lock
└── rg-prod-security: CanNotDelete lock

RBAC (tightly scoped):
├── Platform Team: Contributor on Connectivity subscription
├── App Team: Contributor on their specific RGs only
├── DBA: SQL DB Contributor on data RGs only
├── Security: Security Reader everywhere
└── Everyone: Reader on all production (for visibility)
```

### Pattern 3: Resource Lifecycle with Automation

```
Scenario: Auto-shutdown dev VMs at night

Tag: AutoShutdown=true, ShutdownTime=20:00-IST

Azure Automation Account:
├── Runbook: "Stop-TaggedVMs" (PowerShell)
├── Schedule: Daily at 20:00 IST
├── Logic:
│   $vms = Get-AzVM | Where-Object {$_.Tags.AutoShutdown -eq "true"}
│   foreach ($vm in $vms) { Stop-AzVM -Name $vm.Name -ResourceGroupName $vm.RG }
├── Another runbook: "Start-TaggedVMs" at 08:00 IST
└── Savings: ~58% on dev/staging VM costs

Alternative: Use built-in Auto-shutdown on individual VMs
Portal → VM → Operations → Auto-shutdown → Configure schedule
(Simpler but per-VM, not centralized)
```

---

## Quick Reference: Console Navigation

| Task | Portal Path |
|------|------------|
| Create Resource Group | Resource Groups → Create |
| View all resources | All Resources (left menu) |
| Tag a resource | Resource → Tags (left menu) |
| Create resource lock | Resource or RG → Locks (left menu) |
| Resource Graph query | Resource Graph Explorer |
| View quotas | Subscriptions → Usage + quotas |
| Move resources | Resource → Move |
| Azure Policy | Policy → Definitions / Assignments |
| Cost analysis by tag | Cost Management → Cost analysis → Group by Tag |
| Activity log | Resource → Activity log |

---

## What's Next?

In the next chapter, we'll cover Microsoft Entra ID (Azure AD) in depth — users, groups, app registrations, roles, conditional access, PIM, and managed identities.

→ Next: [Chapter 4: Microsoft Entra ID (Azure AD)](04-entra-id.md)

---

*Last Updated: May 2026*
