# Chapter 37: Terraform on Azure

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Terraform Fundamentals](#part-1-terraform-fundamentals)
- [Part 2: Setting Up Terraform for Azure](#part-2-setting-up-terraform-for-azure)
- [Part 3: AzureRM Provider](#part-3-azurerm-provider)
- [Part 4: State Management](#part-4-state-management)
- [Part 5: Modules](#part-5-modules)
- [Part 6: CI/CD Integration](#part-6-cicd-integration)
- [Part 7: Real-World Project Structure](#part-7-real-world-project-structure)
- [Part 8: Common Commands](#part-8-common-commands)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Terraform is a popular open-source Infrastructure as Code tool by HashiCorp. Unlike Bicep/ARM (Azure-only), Terraform works with any cloud provider (Azure, AWS, GCP) using the same language (HCL). You define what you want, and Terraform figures out how to create it.

```
What you'll learn:
├── Terraform Fundamentals
│   ├── What is Terraform (declarative IaC)
│   ├── Terraform vs Bicep/ARM
│   └── Core workflow: init → plan → apply → destroy
├── Setting Up Terraform for Azure
├── AzureRM Provider (authentication, features)
├── State Management (remote state in Azure Storage)
├── Modules (reusable infrastructure blocks)
├── CI/CD Integration (Azure Pipelines + GitHub Actions)
├── Real-World Project Structure
└── Common commands
```

---

## Part 1: Terraform Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           TERRAFORM CORE CONCEPTS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ How Terraform works:                                                 │
│                                                                       │
│ 1. Write: Define infrastructure in .tf files (HCL language)       │
│ 2. Plan: Terraform compares desired state vs current state        │
│ 3. Apply: Terraform makes changes to reach desired state          │
│ 4. State: Terraform tracks what it manages in a state file       │
│                                                                       │
│ Core workflow:                                                       │
│ terraform init → Downloads providers & modules                   │
│ terraform plan → Shows what will change (like what-if)           │
│ terraform apply → Creates/modifies/deletes resources             │
│ terraform destroy → Deletes everything                            │
│                                                                       │
│ Key difference from Bicep:                                          │
│ ├── Terraform has a STATE FILE that tracks resources             │
│ │   ⚡ Knows exactly what exists, what to add/remove            │
│ │   ⚡ Must be stored safely (Azure Storage, Terraform Cloud)   │
│ ├── Multi-cloud: Same language for Azure + AWS + GCP            │
│ └── Larger ecosystem: 3000+ providers                            │
│                                                                       │
│ Terraform vs Bicep:                                                 │
│ ├── Azure-only project → Use Bicep (simpler, native)           │
│ ├── Multi-cloud project → Use Terraform                         │
│ ├── Team knows Terraform → Use Terraform                       │
│ └── Both are great choices for Azure!                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Setting Up Terraform for Azure

```
Step 1: Install Terraform
├── Windows: choco install terraform  OR  winget install HashiCorp.Terraform
├── macOS: brew install terraform
├── Linux: sudo apt-get install terraform
└── Verify: terraform --version

Step 2: Install Azure CLI
├── Already covered in earlier chapters
└── Verify: az --version

Step 3: Authenticate to Azure
├── Option A: Azure CLI (recommended for development)
│   az login
│   az account set --subscription "My Subscription"
│
├── Option B: Service Principal (recommended for CI/CD)
│   az ad sp create-for-rbac --name "terraform-sp" --role Contributor \
│     --scopes /subscriptions/<sub-id>
│   Export the returned values:
│   export ARM_CLIENT_ID="<appId>"
│   export ARM_CLIENT_SECRET="<password>"
│   export ARM_SUBSCRIPTION_ID="<subscription-id>"
│   export ARM_TENANT_ID="<tenant>"
│
└── Option C: Managed Identity (for Azure-hosted CI/CD agents)
    No credentials needed — identity assigned to the VM/container
```

---

## Part 3: AzureRM Provider

```hcl
# providers.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

provider "azurerm" {
  features {
    # Required (even if empty)
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
    key_vault {
      purge_soft_delete_on_destroy = true
    }
  }

  # Optional: specify subscription
  subscription_id = var.subscription_id
}
```

```hcl
# main.tf — Example: Create a resource group + storage account
resource "azurerm_resource_group" "main" {
  name     = "rg-myapp-prod"
  location = "centralindia"

  tags = {
    environment = "production"
    managed_by  = "terraform"
  }
}

resource "azurerm_storage_account" "main" {
  name                     = "stmyappprod123"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  tags = azurerm_resource_group.main.tags
}
```

```hcl
# variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "centralindia"
}
```

```hcl
# outputs.tf
output "storage_account_id" {
  description = "Storage account resource ID"
  value       = azurerm_storage_account.main.id
}

output "storage_primary_endpoint" {
  description = "Primary blob endpoint"
  value       = azurerm_storage_account.main.primary_blob_endpoint
}
```

```hcl
# terraform.tfvars — variable values
environment = "prod"
location    = "centralindia"
```

---

## Part 4: State Management

```
┌─────────────────────────────────────────────────────────────────────┐
│           TERRAFORM STATE                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is state?                                                       │
│ ├── A JSON file (terraform.tfstate) that tracks your resources  │
│ ├── Maps .tf config to real Azure resources                      │
│ ├── Knows resource IDs, attributes, dependencies                │
│ └── WITHOUT state, Terraform can't manage existing resources    │
│                                                                       │
│ Local state (default — NOT for teams!):                             │
│ terraform.tfstate → File on your local disk                      │
│ ⚡ Problem: If two people run terraform at the same time,       │
│   they overwrite each other's state!                              │
│                                                                       │
│ Remote state (recommended — use Azure Storage!):                   │
│ ├── State stored in Azure Blob Storage                           │
│ ├── State locking (prevents concurrent modifications)           │
│ └── Shared across team members and CI/CD                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

```bash
# Create storage account for Terraform state
az group create --name rg-terraform-state --location centralindia

az storage account create \
  --name sttfstate12345 \
  --resource-group rg-terraform-state \
  --sku Standard_LRS \
  --encryption-services blob

az storage container create \
  --name tfstate \
  --account-name sttfstate12345
```

```hcl
# backend.tf — Remote state configuration
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "sttfstate12345"
    container_name       = "tfstate"
    key                  = "myapp/prod/terraform.tfstate"
  }
}
```

```
State commands:
├── terraform state list → Show all managed resources
├── terraform state show azurerm_storage_account.main → Details
├── terraform state rm azurerm_storage_account.main → Stop managing
├── terraform import azurerm_storage_account.main /subscriptions/.../... → Import existing
└── terraform state mv → Rename/move resources in state
```

---

## Part 5: Modules

```
Modules = Reusable Terraform packages (like functions)

project/
├── main.tf          (calls modules)
├── variables.tf
├── outputs.tf
├── providers.tf
├── backend.tf
└── modules/
    ├── networking/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── database/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── webapp/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

```hcl
# modules/webapp/main.tf
variable "app_name" {
  type = string
}
variable "location" {
  type = string
}
variable "resource_group_name" {
  type = string
}

resource "azurerm_service_plan" "main" {
  name                = "${var.app_name}-plan"
  resource_group_name = var.resource_group_name
  location            = var.location
  os_type             = "Linux"
  sku_name            = "B1"
}

resource "azurerm_linux_web_app" "main" {
  name                = var.app_name
  resource_group_name = var.resource_group_name
  location            = var.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      node_version = "20-lts"
    }
  }
}

output "web_app_url" {
  value = "https://${azurerm_linux_web_app.main.default_hostname}"
}
```

```hcl
# main.tf — Using modules
module "webapp" {
  source              = "./modules/webapp"
  app_name            = "myapp-${var.environment}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
}

module "database" {
  source              = "./modules/database"
  server_name         = "sql-${var.environment}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
}

# Use module from Terraform Registry
module "naming" {
  source  = "Azure/naming/azurerm"
  version = "0.4.0"
  prefix  = ["myapp"]
}
```

---

## Part 6: CI/CD Integration

### Azure Pipelines

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: terraform-secrets  # ARM_CLIENT_ID, ARM_CLIENT_SECRET, etc.

stages:
  - stage: Plan
    jobs:
      - job: TerraformPlan
        steps:
          - task: TerraformInstaller@0
            inputs:
              terraformVersion: 'latest'

          - script: terraform init
            workingDirectory: infra/
            displayName: 'Terraform Init'

          - script: terraform plan -out=tfplan
            workingDirectory: infra/
            displayName: 'Terraform Plan'
            env:
              ARM_CLIENT_ID: $(ARM_CLIENT_ID)
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
              ARM_TENANT_ID: $(ARM_TENANT_ID)

          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: 'infra/tfplan'
              artifactName: 'tfplan'

  - stage: Apply
    dependsOn: Plan
    jobs:
      - deployment: TerraformApply
        environment: 'production'  # Requires approval!
        strategy:
          runOnce:
            deploy:
              steps:
                - script: |
                    terraform init
                    terraform apply -auto-approve tfplan
                  workingDirectory: infra/
                  displayName: 'Terraform Apply'
```

### GitHub Actions

```yaml
# .github/workflows/terraform.yml
name: Terraform
on:
  push:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    env:
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - run: terraform init
        working-directory: infra/

      - run: terraform plan
        working-directory: infra/

      - run: terraform apply -auto-approve
        working-directory: infra/
        if: github.ref == 'refs/heads/main'
```

---

## Part 7: Real-World Project Structure

```
terraform-azure-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf          (calls modules with dev values)
│   │   ├── backend.tf       (dev state file location)
│   │   └── terraform.tfvars (dev variable values)
│   ├── staging/
│   │   ├── main.tf
│   │   ├── backend.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── backend.tf
│       └── terraform.tfvars
├── modules/
│   ├── networking/
│   ├── compute/
│   ├── database/
│   └── monitoring/
└── README.md

Deploy dev:  cd environments/dev  && terraform apply
Deploy prod: cd environments/prod && terraform apply

⚡ Each environment has its own state file!
⚡ Modules are shared across environments!
```

---

## Part 8: Common Commands

```bash
# Initialize (download providers, configure backend)
terraform init

# Preview changes
terraform plan

# Apply changes
terraform apply

# Apply without confirmation
terraform apply -auto-approve

# Destroy all resources
terraform destroy

# Format code
terraform fmt -recursive

# Validate syntax
terraform validate

# Show current state
terraform state list
terraform state show azurerm_resource_group.main

# Import existing Azure resource into Terraform
terraform import azurerm_resource_group.main /subscriptions/.../resourceGroups/rg-myapp

# Create a plan file
terraform plan -out=tfplan
terraform apply tfplan

# Target specific resource
terraform apply -target=azurerm_storage_account.main

# Refresh state from Azure
terraform refresh

# Show outputs
terraform output
```

---

## Quick Reference

```
Terraform = Multi-cloud IaC tool (HCL language)
AzureRM provider = Terraform plugin for Azure
State file = Tracks managed resources (store in Azure Blob!)

Workflow: init → plan → apply → (destroy)
Auth: az login (dev) | Service Principal (CI/CD) | Managed Identity

Key files:
  main.tf        → Resources
  variables.tf   → Input variables
  outputs.tf     → Output values
  providers.tf   → Provider config
  backend.tf     → Remote state config
  terraform.tfvars → Variable values

Modules: Reusable infrastructure packages
State: terraform state list/show/rm/import/mv
CI/CD: Azure Pipelines or GitHub Actions + Service Principal
```

---

## What's Next?

Next chapter: [Chapter 38: Azure Monitor](38-azure-monitor.md) — Metrics, alerts, action groups, and dashboards for monitoring your Azure resources.
