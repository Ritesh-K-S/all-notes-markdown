# Chapter 36: ARM Templates & Bicep

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Infrastructure as Code Fundamentals](#part-1-infrastructure-as-code-fundamentals)
- [Part 2: ARM Template Structure](#part-2-arm-template-structure)
- [Part 3: Bicep Language](#part-3-bicep-language)
- [Part 4: Parameters & Variables](#part-4-parameters--variables)
- [Part 5: Modules](#part-5-modules)
- [Part 6: Deploying Templates (Portal & CLI)](#part-6-deploying-templates-portal--cli)
- [Part 7: What-If & Deployment Stacks](#part-7-what-if--deployment-stacks)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Part 9: az CLI Reference](#part-9-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

ARM (Azure Resource Manager) templates and Bicep are Azure's native Infrastructure as Code (IaC) tools. ARM templates use JSON; Bicep is a simpler, human-friendly language that compiles to ARM JSON. Both let you define your entire Azure infrastructure in code — so you can version it, review it, and deploy it consistently every time.

```
What you'll learn:
├── Infrastructure as Code Fundamentals
│   ├── Why IaC (repeatable, version controlled)
│   ├── ARM Templates (JSON) vs Bicep vs Terraform
│   └── Declarative vs Imperative
├── ARM Template Structure (JSON format)
├── Bicep Language (cleaner syntax)
├── Parameters & Variables (make templates reusable)
├── Modules (organize large deployments)
├── Deploying Templates (Portal, CLI, PowerShell)
├── What-If & Deployment Stacks
├── Real-World Patterns
└── az CLI reference
```

---

## Part 1: Infrastructure as Code Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHY INFRASTRUCTURE AS CODE?                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Without IaC (manual):                                                │
│ ├── Click through portal to create resources                     │
│ ├── Forget a setting → inconsistent environments                │
│ ├── Can't easily reproduce → "works on my Azure" problem       │
│ └── No audit trail of infrastructure changes                    │
│                                                                       │
│ With IaC:                                                            │
│ ├── Define everything in code files                               │
│ ├── Store in Git → version history, code review                 │
│ ├── Deploy consistently → dev = staging = prod                 │
│ └── Destroy and recreate anytime                                 │
│                                                                       │
│ ARM JSON vs Bicep vs Terraform:                                     │
│ ┌──────────────┬───────────┬──────────┬─────────────┐            │
│ │ Feature      │ ARM JSON  │ Bicep    │ Terraform   │            │
│ ├──────────────┼───────────┼──────────┼─────────────┤            │
│ │ Syntax       │ Verbose   │ Clean    │ Clean (HCL) │            │
│ │ Learning     │ Hard      │ Easy     │ Medium      │            │
│ │ Azure native │ Yes ✅    │ Yes ✅   │ No (3rd pty)│            │
│ │ Multi-cloud  │ No ❌     │ No ❌    │ Yes ✅      │            │
│ │ State file   │ No        │ No       │ Yes         │            │
│ │ What-if      │ Built-in  │ Built-in │ terraform plan│           │
│ │ Tooling      │ VS Code   │ VS Code  │ VS Code     │            │
│ │ Ecosystem    │ Azure     │ Azure    │ All clouds  │            │
│ └──────────────┴───────────┴──────────┴─────────────┘            │
│                                                                       │
│ ⚡ Bicep is recommended for Azure-only projects.                  │
│ ⚡ Terraform is recommended for multi-cloud projects.             │
│ ⚡ ARM JSON is still used but Bicep is its replacement.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: ARM Template Structure

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "type": "string",
      "metadata": {
        "description": "Name of the storage account"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    }
  },
  "variables": {
    "storageSku": "Standard_LRS"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[variables('storageSku')]"
      },
      "kind": "StorageV2",
      "properties": {
        "accessTier": "Hot"
      }
    }
  ],
  "outputs": {
    "storageId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
    }
  }
}
```

```
ARM Template sections:
├── $schema → Template version (don't change)
├── contentVersion → Your template version
├── parameters → Input values (passed at deployment)
├── variables → Computed values (used within template)
├── resources → The actual Azure resources to create
├── outputs → Values returned after deployment
└── functions → Custom reusable functions (rarely used)
```

---

## Part 3: Bicep Language

The same storage account in Bicep (much simpler!):

```bicep
// main.bicep
param storageAccountName string
param location string = resourceGroup().location

var storageSku = 'Standard_LRS'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
  }
}

output storageId string = storageAccount.id
```

```
Bicep vs ARM JSON comparison:
├── No curly braces everywhere (cleaner)
├── Type inference (less boilerplate)
├── IntelliSense in VS Code (great autocomplete)
├── Compile to ARM: az bicep build --file main.bicep
├── Decompile ARM: az bicep decompile --file template.json
└── Bicep compiles to ARM JSON behind the scenes
```

### More Bicep Examples

```bicep
// App Service Plan + Web App
param appName string
param location string = resourceGroup().location

resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: '${appName}-plan'
  location: location
  sku: {
    name: 'B1'
    tier: 'Basic'
  }
  kind: 'linux'
  properties: {
    reserved: true  // Linux
  }
}

resource webApp 'Microsoft.Web/sites@2023-01-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
    }
  }
}
```

---

## Part 4: Parameters & Variables

```bicep
// Parameters — values provided at deployment time

// Simple parameter
param environment string

// With allowed values
@allowed(['dev', 'staging', 'prod'])
param environmentName string

// With default value
param location string = resourceGroup().location

// With constraints
@minLength(3)
@maxLength(24)
param storageAccountName string

// Secure (won't show in logs)
@secure()
param adminPassword string

// Parameter file (main.parameters.json or .bicepparam)
// Provides values for parameters at deployment time
```

```json
// main.parameters.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "value": "mystorageaccount123"
    },
    "environmentName": {
      "value": "prod"
    }
  }
}
```

```bicep
// Bicep parameter file (.bicepparam) — new format
using 'main.bicep'

param storageAccountName = 'mystorageaccount123'
param environmentName = 'prod'
```

```bicep
// Variables — computed values used within the template
var storageName = 'st${environmentName}${uniqueString(resourceGroup().id)}'
var isProduction = environmentName == 'prod'
var tags = {
  environment: environmentName
  managedBy: 'bicep'
}
```

---

## Part 5: Modules

```
Modules let you break large templates into reusable pieces:

main.bicep
├── modules/
│   ├── network.bicep      (VNet, subnets, NSGs)
│   ├── database.bicep     (SQL Server, database)
│   ├── webapp.bicep       (App Service Plan, Web App)
│   └── monitoring.bicep   (Log Analytics, App Insights)
```

```bicep
// modules/webapp.bicep
param appName string
param location string
param appServicePlanId string

resource webApp 'Microsoft.Web/sites@2023-01-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlanId
  }
}

output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
```

```bicep
// main.bicep — using modules
param environment string = 'prod'
param location string = resourceGroup().location

module network 'modules/network.bicep' = {
  name: 'networkDeploy'
  params: {
    environment: environment
    location: location
  }
}

module database 'modules/database.bicep' = {
  name: 'databaseDeploy'
  params: {
    subnetId: network.outputs.dbSubnetId
    location: location
  }
}

module webapp 'modules/webapp.bicep' = {
  name: 'webappDeploy'
  params: {
    appName: 'myapp-${environment}'
    location: location
    appServicePlanId: network.outputs.appServicePlanId
  }
}
```

```
Bicep Registry (share modules):
├── Public: Azure Verified Modules (community templates)
│   module storage 'br/public:avm/res/storage/storage-account:0.9.0' = {
│     params: { ... }
│   }
├── Private: Your own ACR as module registry
│   module myModule 'br:myacr.azurecr.io/bicep/modules/webapp:v1' = {
│     params: { ... }
│   }
└── Template Specs: Store templates in Azure for sharing
```

---

## Part 6: Deploying Templates (Portal & CLI)

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEPLOYING TEMPLATES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Method 1: Azure CLI                                                 │
│ az deployment group create \                                        │
│   --resource-group rg-myapp-prod \                                 │
│   --template-file main.bicep \                                     │
│   --parameters @main.parameters.json                               │
│                                                                       │
│ Method 2: PowerShell                                                │
│ New-AzResourceGroupDeployment `                                    │
│   -ResourceGroupName rg-myapp-prod `                               │
│   -TemplateFile main.bicep `                                       │
│   -TemplateParameterFile main.parameters.json                     │
│                                                                       │
│ Method 3: Azure Portal                                              │
│ Search → "Deploy a custom template" → Build your own template  │
│ Paste JSON → Enter parameters → Deploy                           │
│                                                                       │
│ Method 4: CI/CD Pipeline                                            │
│ - task: AzureResourceManagerTemplateDeployment@3                  │
│   inputs:                                                            │
│     deploymentScope: 'Resource Group'                              │
│     azureResourceManagerConnection: 'Azure-SC'                   │
│     resourceGroupName: 'rg-myapp-prod'                            │
│     location: 'centralindia'                                       │
│     templateLocation: 'Linked artifact'                            │
│     csmFile: 'infra/main.bicep'                                   │
│     csmParametersFile: 'infra/main.parameters.json'              │
│                                                                       │
│ Deployment scopes:                                                   │
│ ├── Resource Group: Most common (az deployment group create)     │
│ ├── Subscription: Create resource groups (az deployment sub)     │
│ ├── Management Group: Cross-subscription policies               │
│ └── Tenant: Tenant-level resources                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: What-If & Deployment Stacks

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT-IF (Preview changes before deploying)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ az deployment group what-if \                                       │
│   --resource-group rg-myapp-prod \                                 │
│   --template-file main.bicep                                       │
│                                                                       │
│ Output:                                                              │
│ Resource and property changes detected:                             │
│                                                                       │
│   + Microsoft.Storage/storageAccounts/st123 [CREATE]              │
│   ~ Microsoft.Web/sites/myapp [MODIFY]                            │
│     - properties.siteConfig.appSettings[0].value: "old" → "new" │
│   - Microsoft.Cache/Redis/mycache [DELETE]                        │
│                                                                       │
│ ⚡ Like terraform plan but for ARM/Bicep!                         │
│ ⚡ Shows what will be created, modified, or deleted.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           DEPLOYMENT STACKS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Deployment Stack = A managed collection of resources from a template│
│                                                                       │
│ Problem: Normal deployments don't track what to delete.            │
│ If you remove a resource from your template and redeploy,         │
│ the old resource stays (orphaned).                                  │
│                                                                       │
│ Solution: Deployment Stacks track all resources.                   │
│ Remove from template → Stack can delete the orphaned resource.  │
│                                                                       │
│ az stack group create \                                              │
│   --name myapp-stack \                                              │
│   --resource-group rg-myapp-prod \                                 │
│   --template-file main.bicep \                                     │
│   --deny-settings-mode denyDelete \                                │
│   --action-on-unmanage deleteAll                                   │
│                                                                       │
│ deny-settings-mode:                                                  │
│ ├── none → No protection                                         │
│ ├── denyDelete → Prevent deletion of managed resources           │
│ └── denyWriteAndDelete → Prevent modification AND deletion      │
│                                                                       │
│ action-on-unmanage (when resource removed from template):          │
│ ├── detachAll → Just stop tracking (keep resource)              │
│ └── deleteAll → Delete the resource                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Real-World Patterns

```
Pattern 1: Environment-specific deployments
├── infra/
│   ├── main.bicep
│   ├── parameters/
│   │   ├── dev.parameters.json
│   │   ├── staging.parameters.json
│   │   └── prod.parameters.json
│   └── modules/
│       ├── network.bicep
│       └── compute.bicep

Deploy dev:  az deployment group create ... --parameters @parameters/dev.parameters.json
Deploy prod: az deployment group create ... --parameters @parameters/prod.parameters.json

Pattern 2: Conditional resources
resource redis 'Microsoft.Cache/Redis@2023-08-01' = if (isProduction) {
  name: 'redis-${environment}'
  ...
}
⚡ Redis only created in production, not in dev!

Pattern 3: Loops
param locations array = ['centralindia', 'eastus', 'westeurope']

resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for loc in locations: {
  name: 'st${uniqueString(resourceGroup().id, loc)}'
  location: loc
  ...
}]
⚡ Creates 3 storage accounts, one in each region!
```

---

## Part 9: az CLI Reference

```bash
# Deploy Bicep template
az deployment group create \
  --resource-group rg-myapp-prod \
  --template-file main.bicep \
  --parameters @main.parameters.json

# Preview changes (what-if)
az deployment group what-if \
  --resource-group rg-myapp-prod \
  --template-file main.bicep

# List deployments
az deployment group list --resource-group rg-myapp-prod --output table

# Show deployment details
az deployment group show --resource-group rg-myapp-prod --name mainDeploy

# Export existing resource group as ARM template
az group export --name rg-myapp-prod --output json > exported.json

# Compile Bicep to ARM JSON
az bicep build --file main.bicep

# Decompile ARM JSON to Bicep
az bicep decompile --file template.json

# Install/upgrade Bicep CLI
az bicep install
az bicep upgrade

# Create deployment stack
az stack group create \
  --name myapp-stack \
  --resource-group rg-myapp-prod \
  --template-file main.bicep \
  --action-on-unmanage deleteAll \
  --deny-settings-mode denyDelete

# Delete deployment (metadata only, resources stay)
az deployment group delete --resource-group rg-myapp-prod --name mainDeploy
```

---

## Quick Reference

```
ARM Template = Azure IaC in JSON (verbose)
Bicep = Azure IaC in clean syntax (compiles to ARM JSON)

Template sections: parameters, variables, resources, outputs
Bicep file: .bicep | ARM file: .json

Deploy: az deployment group create --template-file main.bicep
Preview: az deployment group what-if --template-file main.bicep

Modules: Break templates into reusable pieces
Parameter files: .parameters.json or .bicepparam
Deployment Stacks: Track & protect managed resources

Scopes: Resource Group > Subscription > Management Group > Tenant
Bicep Registry: Share modules via ACR or public registry
```

---

## What's Next?

Next chapter: [Chapter 37: Terraform on Azure](37-terraform-azure.md) — Using Terraform's AzureRM provider, state management, modules, and CI/CD integration.
