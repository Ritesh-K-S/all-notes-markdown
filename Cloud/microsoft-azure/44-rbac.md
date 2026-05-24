# Chapter 44: Azure RBAC & Custom Roles

---

## Table of Contents

- [Overview](#overview)
- [Part 1: RBAC Fundamentals](#part-1-rbac-fundamentals)
- [Part 2: Built-in Roles](#part-2-built-in-roles)
- [Part 3: Assigning Roles (Portal Walkthrough)](#part-3-assigning-roles-portal-walkthrough)
- [Part 4: Custom Roles](#part-4-custom-roles)
- [Part 5: Scope Levels](#part-5-scope-levels)
- [Part 6: Deny Assignments & Locks](#part-6-deny-assignments--locks)
- [Part 7: Terraform & az CLI Reference](#part-7-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure RBAC (Role-Based Access Control) controls who can do what in Azure. Instead of giving everyone full admin access, you assign specific roles (like "Virtual Machine Contributor" or "Storage Blob Data Reader") at the right scope (subscription, resource group, or individual resource). This follows the principle of least privilege.

```
What you'll learn:
├── RBAC Fundamentals
│   ├── What is RBAC (who + what + where)
│   ├── Security principal, role, scope
│   └── RBAC vs Entra ID roles
├── Built-in Roles (Owner, Contributor, Reader, etc.)
├── Assigning Roles (Portal)
├── Custom Roles (create your own)
├── Scope Levels (Management Group → Subscription → RG → Resource)
├── Deny Assignments & Resource Locks
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: RBAC Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           RBAC CONCEPT                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ RBAC answers three questions:                                       │
│                                                                       │
│ WHO (Security Principal)   +   WHAT (Role)   +   WHERE (Scope)   │
│                                                                       │
│ WHO can access? (Security Principal)                                │
│ ├── User (john@company.com)                                      │
│ ├── Group (sg-developers)                                        │
│ ├── Service Principal (app identity)                              │
│ └── Managed Identity (Azure resource identity)                   │
│                                                                       │
│ WHAT can they do? (Role Definition)                                 │
│ ├── Owner (full access + assign roles)                           │
│ ├── Contributor (full access, no role assignment)                │
│ ├── Reader (view only)                                           │
│ ├── Specialized roles (VM Contributor, SQL DB Contributor, etc.)│
│ └── Custom roles (your own definition)                           │
│                                                                       │
│ WHERE does it apply? (Scope)                                       │
│ ├── Management Group (all subscriptions under it)               │
│ ├── Subscription (all resources in subscription)                │
│ ├── Resource Group (all resources in group)                     │
│ └── Resource (single specific resource)                         │
│                                                                       │
│ Example: "Give sg-developers group the Contributor role on       │
│          resource group rg-myapp-dev"                              │
│                                                                       │
│ Role Assignment = WHO + WHAT + WHERE                               │
│                                                                       │
│ Inheritance: Permissions cascade DOWN the scope hierarchy.       │
│ Owner at subscription → Owner on ALL resource groups + resources│
│                                                                       │
│ ⚡ RBAC is additive — permissions are UNIONED (not intersected). │
│ If you have Reader at subscription AND Contributor at RG,        │
│ you're effectively Contributor at that RG level.                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Built-in Roles

```
┌──────────────────────────────────────────────────────────────────┐
│ General-purpose roles:                                            │
│ ├── Owner → Full access + manage role assignments              │
│ ├── Contributor → Full access, cannot assign roles            │
│ ├── Reader → View everything, change nothing                  │
│ └── User Access Administrator → Only manage role assignments │
│                                                                    │
│ Compute roles:                                                    │
│ ├── Virtual Machine Contributor → Manage VMs (not VNet/storage)│
│ ├── Virtual Machine Administrator Login → Login as admin     │
│ └── Virtual Machine User Login → Login as regular user       │
│                                                                    │
│ Storage roles:                                                    │
│ ├── Storage Account Contributor → Manage storage accounts    │
│ ├── Storage Blob Data Contributor → Read/write blob data    │
│ ├── Storage Blob Data Reader → Read blob data only          │
│ └── Storage Queue Data Contributor → Read/write queue data  │
│                                                                    │
│ Database roles:                                                   │
│ ├── SQL DB Contributor → Manage SQL databases (not security)│
│ ├── SQL Server Contributor → Manage SQL servers             │
│ └── Cosmos DB Account Reader Role → Read Cosmos DB data    │
│                                                                    │
│ Networking roles:                                                 │
│ ├── Network Contributor → Manage networks                    │
│ └── DNS Zone Contributor → Manage DNS zones                 │
│                                                                    │
│ Security roles:                                                   │
│ ├── Key Vault Secrets User → Read secrets                   │
│ ├── Key Vault Administrator → Full Key Vault access         │
│ ├── Security Admin → Read security policy + manage alerts  │
│ └── Security Reader → View security features               │
│                                                                    │
│ Kubernetes roles:                                                 │
│ ├── Azure Kubernetes Service Cluster Admin Role             │
│ ├── Azure Kubernetes Service Cluster User Role              │
│ └── Azure Kubernetes Service RBAC Admin                     │
│                                                                    │
│ ⚡ There are 400+ built-in roles! These are the most common.   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Assigning Roles (Portal Walkthrough)

```
Any resource → Access control (IAM) → [+ Add] → Add role assignment

┌─────────────────────────────────────────────────────────────────────┐
│           ADD ROLE ASSIGNMENT                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Select role                                                 │
│ Search: [Contributor ▼]                                            │
│ ┌────────────────────────────────────────────┐                    │
│ │ ○ Owner                                     │                    │
│ │ ● Contributor                               │                    │
│ │ ○ Reader                                    │                    │
│ │ ○ Storage Blob Data Reader                 │                    │
│ │ ○ Virtual Machine Contributor              │                    │
│ │ ...                                         │                    │
│ └────────────────────────────────────────────┘                    │
│ [Next]                                                               │
│                                                                       │
│ Step 2: Select members                                              │
│ Assign access to: ● User, group, or service principal            │
│                    ○ Managed identity                              │
│ [+ Select members]                                                  │
│ Search: [sg-developers]                                            │
│ Selected: sg-developers (Security group)                          │
│ [Next]                                                               │
│                                                                       │
│ Step 3: Conditions (optional, for data actions)                    │
│ For some roles, you can add conditions:                            │
│ "Only allow read access to blobs with tag=public"                │
│                                                                       │
│ Step 4: Review + assign                                             │
│ Role: Contributor                                                    │
│ Members: sg-developers                                              │
│ Scope: /subscriptions/.../resourceGroups/rg-myapp-dev             │
│                                                                       │
│ [Review + assign]                                                   │
│                                                                       │
│ Check existing assignments:                                         │
│ Access control (IAM) → Role assignments tab                      │
│ ⚡ See WHO has WHAT access at this scope level.                  │
│                                                                       │
│ Check your own access:                                              │
│ Access control (IAM) → Check access → [Enter user/group]       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Custom Roles

```
┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM ROLES                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ When built-in roles don't fit, create your own!                    │
│                                                                       │
│ Access control (IAM) → [+ Add] → Add custom role                │
│ OR: Clone an existing role and modify                              │
│                                                                       │
│ Custom role name: [VM Operator]                                    │
│ Description: [Can start, stop, and restart VMs but not delete]   │
│ Baseline: ○ Start from scratch  ● Clone a role  ○ Start from JSON│
│                                                                       │
│ Permissions:                                                         │
│ ☑ Microsoft.Compute/virtualMachines/read                         │
│ ☑ Microsoft.Compute/virtualMachines/start/action                 │
│ ☑ Microsoft.Compute/virtualMachines/restart/action               │
│ ☑ Microsoft.Compute/virtualMachines/powerOff/action              │
│ ☑ Microsoft.Compute/virtualMachines/deallocate/action            │
│ ☐ Microsoft.Compute/virtualMachines/delete ← NOT included!     │
│ ☐ Microsoft.Compute/virtualMachines/write ← NOT included!      │
│                                                                       │
│ Assignable scopes: [/subscriptions/<subscription-id>]            │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Custom Role JSON

```json
{
  "Name": "VM Operator",
  "Description": "Can start, stop, restart VMs but not create or delete",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/powerOff/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Resources/subscriptions/resourceGroups/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/<subscription-id>"
  ]
}
```

```
Permission types:
├── Actions → Control plane (manage resources via ARM)
│   Example: Microsoft.Compute/virtualMachines/write
├── DataActions → Data plane (access data within resources)
│   Example: Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read
├── NotActions → Exclude specific actions from Actions
└── NotDataActions → Exclude specific data actions
```

---

## Part 5: Scope Levels

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCOPE HIERARCHY                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Management Group (highest scope)                                    │
│ └── Subscription                                                   │
│     └── Resource Group                                             │
│         └── Resource (lowest scope)                               │
│                                                                       │
│ Example:                                                             │
│ Management Group: mg-production                                    │
│ ├── Subscription: prod-subscription                               │
│ │   ├── Resource Group: rg-webapp                                │
│ │   │   ├── App Service: myapp-prod                             │
│ │   │   ├── SQL Database: sqldb-prod                            │
│ │   │   └── Storage Account: stprod123                          │
│ │   └── Resource Group: rg-infra                                │
│ │       ├── VNet: vnet-prod                                     │
│ │       └── Key Vault: kv-prod                                  │
│ └── Subscription: dev-subscription                               │
│     └── Resource Group: rg-dev                                   │
│         └── ...                                                    │
│                                                                       │
│ Assign Contributor at rg-webapp → Access to myapp, sqldb, st   │
│ Assign Reader at prod-subscription → View ALL resources in sub │
│ Assign Owner at mg-production → Full control of everything!    │
│                                                                       │
│ Best practice:                                                       │
│ ├── Assign at RESOURCE GROUP level (most common)                │
│ ├── Use GROUPS, not individual users                             │
│ ├── Follow least privilege (smallest scope + role needed)       │
│ └── Avoid Owner/Contributor at subscription level               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Deny Assignments & Locks

```
Deny Assignments:
├── Override (deny) specific permissions even if user has them
├── Created by Azure Blueprints and Deployment Stacks
├── Cannot be created directly by users
└── Example: Blueprint denies deletion of critical resources

Resource Locks:
├── Resource → Settings → Locks → [+ Add]
├── Lock types:
│   ├── ReadOnly → Can read, cannot modify or delete
│   └── Delete → Can read and modify, cannot delete
├── Inheritance: Lock on RG applies to all resources in it
├── Even OWNER cannot delete a locked resource without removing lock first!
└── Use for: Production resources you never want accidentally deleted

az lock create --name no-delete --lock-type CanNotDelete \
  --resource-group rg-production
```

---

## Part 7: Terraform & az CLI Reference

### Terraform

```hcl
# Role assignment
resource "azurerm_role_assignment" "dev_contributor" {
  scope                = azurerm_resource_group.dev.id
  role_definition_name = "Contributor"
  principal_id         = azuread_group.developers.object_id
}

# Custom role
resource "azurerm_role_definition" "vm_operator" {
  name        = "VM Operator"
  scope       = data.azurerm_subscription.current.id
  description = "Can start, stop, restart VMs"

  permissions {
    actions = [
      "Microsoft.Compute/virtualMachines/read",
      "Microsoft.Compute/virtualMachines/start/action",
      "Microsoft.Compute/virtualMachines/restart/action",
      "Microsoft.Compute/virtualMachines/powerOff/action",
      "Microsoft.Compute/virtualMachines/deallocate/action",
    ]
  }

  assignable_scopes = [data.azurerm_subscription.current.id]
}

# Resource lock
resource "azurerm_management_lock" "rg_lock" {
  name       = "no-delete"
  scope      = azurerm_resource_group.prod.id
  lock_level = "CanNotDelete"
}
```

### Bicep

```bicep
// Role assignment
resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, developersGroupId, 'Contributor')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'b24988ac-6180-42a0-ab88-20f7382dd24c') // Contributor
    principalId: developersGroupId
    principalType: 'Group'
  }
}

// Custom role definition
resource customRole 'Microsoft.Authorization/roleDefinitions@2022-04-01' = {
  name: guid(subscription().id, 'VM Operator')
  properties: {
    roleName: 'VM Operator'
    description: 'Can start, stop, restart VMs'
    type: 'CustomRole'
    permissions: [
      {
        actions: [
          'Microsoft.Compute/virtualMachines/read'
          'Microsoft.Compute/virtualMachines/start/action'
          'Microsoft.Compute/virtualMachines/restart/action'
          'Microsoft.Compute/virtualMachines/powerOff/action'
          'Microsoft.Compute/virtualMachines/deallocate/action'
        ]
      }
    ]
    assignableScopes: [subscription().id]
  }
}

// Resource lock
resource lock 'Microsoft.Authorization/locks@2020-05-01' = {
  name: 'no-delete'
  properties: {
    level: 'CanNotDelete'
    notes: 'Prevent accidental deletion of production resources'
  }
}
```
# Assign role
az role assignment create \
  --assignee sg-developers \
  --role "Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-dev

# List role assignments
az role assignment list --resource-group rg-myapp-dev --output table

# List all built-in roles
az role definition list --output table

# Show specific role
az role definition list --name "Contributor"

# Create custom role from JSON
az role definition create --role-definition @vm-operator.json

# Create resource lock
az lock create --name no-delete --lock-type CanNotDelete \
  --resource-group rg-production

# Remove role assignment
az role assignment delete \
  --assignee sg-developers \
  --role "Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-dev

# Delete custom role
az role definition delete --name "VM Operator"
```

---

## Real-World Patterns

### Pattern 1: Least-Privilege Access for Teams

```
┌─────────────────────────────────────────────────┐
│  Team-Based RBAC Architecture                   │
├─────────────────────────────────────────────────┤
│                                                 │
│  Entra ID Group        Scope         Role       │
│  ────────────────    ──────────    ───────── │
│  SG-Platform-Admins  → Subscription → Contributor│
│  SG-Dev-Team-A       → RG-App-A    → Contributor│
│  SG-Dev-Team-B       → RG-App-B    → Contributor│
│  SG-Data-Engineers   → RG-Data     → Custom Role│
│  SG-Security-Audit   → Subscription → Reader    │
│  SG-DB-Admins        → SQL Servers → SQL Contrib│
│                                                 │
│  Rules:                                         │
│  - Assign to groups, NEVER individual users     │
│  - Use narrowest scope possible                 │
│  - Custom roles only when built-in won't work   │
│  - Review access quarterly with Access Reviews  │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
RBAC = WHO (principal) + WHAT (role) + WHERE (scope)

General roles: Owner > Contributor > Reader
  Owner: Full access + assign roles
  Contributor: Full access, no role assignment
  Reader: View only

Scope: Management Group > Subscription > Resource Group > Resource
Inheritance: Permissions cascade DOWN

Best practices:
  Use groups (not individual users)
  Assign at resource group level (not subscription)
  Follow least privilege (minimum role + scope)
  Use custom roles when built-in don't fit

Locks: ReadOnly | Delete (even Owner can't bypass!)
Custom roles: Actions + DataActions (JSON definition)
400+ built-in roles available
```

---

## What's Next?

Next chapter: [Chapter 45: Azure Policy](45-azure-policy.md) — Enforce organizational standards and compliance with policy definitions, initiatives, and compliance reporting.
