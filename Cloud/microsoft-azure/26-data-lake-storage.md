# Chapter 26: Azure Data Lake Storage Gen2

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Data Lake Storage Fundamentals](#part-1-data-lake-storage-fundamentals)
- [Part 2: Enabling Data Lake Storage Gen2 (Portal)](#part-2-enabling-data-lake-storage-gen2-portal)
- [Part 3: Containers & File Systems](#part-3-containers--file-systems)
- [Part 4: Access Control (ACLs)](#part-4-access-control-acls)
- [Part 5: Integration with Analytics](#part-5-integration-with-analytics)
- [Part 6: Terraform & Bicep](#part-6-terraform--bicep)
- [Part 7: az CLI Reference](#part-7-az-cli-reference)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Data Lake Storage Gen2 (ADLS Gen2) is a set of capabilities built on top of Azure Blob Storage specifically designed for big data analytics. It adds a hierarchical namespace (real folders!) and fine-grained access control (POSIX ACLs) to standard blob storage. Think of it as "Blob Storage + real folder structure + granular permissions."

```
What you'll learn:
├── Data Lake Storage Gen2 Fundamentals
│   ├── What is ADLS Gen2 (blob storage with hierarchical namespace)
│   ├── Why it matters for analytics (real directories, ACLs)
│   └── ADLS Gen2 vs regular Blob Storage
├── Enabling ADLS Gen2 (during storage account creation)
├── Containers (file systems) & directory management
├── Access Control (RBAC + POSIX ACLs)
├── Integration with analytics services
│   ├── Azure Synapse Analytics
│   ├── Azure Databricks
│   ├── Azure Data Factory
│   └── HDInsight
├── Terraform, Bicep, az CLI
└── Real-world patterns (data lake architecture)
```

---

## Part 1: Data Lake Storage Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           DATA LAKE STORAGE GEN2 CONCEPT                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is a Data Lake?                                                │
│ ├── A centralized repository that stores ALL your data            │
│ ├── Structured data (tables, CSV) + unstructured (images, logs) │
│ ├── Store data "as-is" — transform later when needed            │
│ ├── Used by data engineers, data scientists, analysts            │
│ └── Think: A huge storage pool for your entire organization     │
│                                                                       │
│ What makes ADLS Gen2 different from regular Blob Storage?          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ Regular Blob Storage:                                        │  │
│ │ ├── Flat namespace (virtual folders using / in names)      │  │
│ │ ├── Renaming a "folder" = rename every blob inside it      │  │
│ │ ├── No real directory operations                            │  │
│ │ └── Access control at container or blob level only         │  │
│ │                                                              │  │
│ │ ADLS Gen2 (Hierarchical Namespace enabled):                  │  │
│ │ ├── REAL directories (actual folder structure!)            │  │
│ │ ├── Renaming a folder = atomic operation (instant!)        │  │
│ │ ├── Directory-level operations (rename, delete, move)      │  │
│ │ ├── POSIX ACLs (granular file/folder permissions)         │  │
│ │ ├── Same blob APIs + new DFS (Distributed File System) API│  │
│ │ └── Optimized for analytics workloads (Spark, Synapse)    │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ ADLS Gen2 = Blob Storage + hierarchical namespace enabled.     │
│   Same storage account, same pricing, extra features!            │
│                                                                       │
│ Example data lake structure:                                         │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Container: datalake                                          │  │
│ │ ├── raw/           (raw ingested data)                     │  │
│ │ │   ├── sales/                                             │  │
│ │ │   │   ├── 2024/01/sales-data.csv                       │  │
│ │ │   │   └── 2024/02/sales-data.csv                       │  │
│ │ │   ├── logs/                                              │  │
│ │ │   │   └── 2024/01/app-logs.json                        │  │
│ │ │   └── images/                                            │  │
│ │ │       └── products/                                      │  │
│ │ ├── processed/     (cleaned, transformed data)             │  │
│ │ │   ├── sales/                                             │  │
│ │ │   │   └── 2024/sales-clean.parquet                     │  │
│ │ │   └── aggregated/                                        │  │
│ │ │       └── monthly-summary.parquet                       │  │
│ │ └── curated/       (business-ready data)                  │  │
│ │     ├── reports/                                            │  │
│ │     └── dashboards/                                         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ AWS equivalent: S3 + AWS Lake Formation                         │
│ ⚡ GCP equivalent: Cloud Storage + BigLake                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Enabling Data Lake Storage Gen2 (Portal)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CREATING ADLS GEN2 STORAGE ACCOUNT                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Storage accounts → Create                               │
│                                                                       │
│ Basics tab:                                                          │
│ ├── Name: stdatalakeprod                                          │
│ ├── Performance: Standard                                         │
│ ├── Redundancy: ZRS or GRS                                       │
│                                                                       │
│ Advanced tab:                                                        │
│ ├── ☑ Enable hierarchical namespace  ← THIS ENABLES ADLS Gen2 │
│ │   ⚠️ Cannot be disabled once enabled!                        │
│ │   ⚠️ Some blob features may behave differently               │
│ └── Access tier: Hot (default for data lake)                     │
│                                                                       │
│ ⚡ That's it! Checking "hierarchical namespace" transforms        │
│   regular blob storage into ADLS Gen2.                            │
│                                                                       │
│ Endpoints (after creation):                                          │
│ ├── Blob:  https://stdatalakeprod.blob.core.windows.net         │
│ ├── DFS:   https://stdatalakeprod.dfs.core.windows.net  ← new! │
│ └── DFS endpoint is used by analytics tools (Spark, Synapse)   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Containers & File Systems

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONTAINERS & DIRECTORIES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ In ADLS Gen2, containers are called "file systems" (same thing): │
│                                                                       │
│ Portal → Storage Account → Containers                             │
│ [+ Container]                                                       │
│ Name: [datalake]                                                    │
│ Access level: Private                                                │
│                                                                       │
│ Inside the container:                                                │
│ [+ Add Directory]  [Upload]                                        │
│                                                                       │
│ ⚡ Unlike regular Blob Storage:                                    │
│   ├── Directories are REAL (not just name prefixes)              │
│   ├── Rename directory = instant (atomic operation)              │
│   ├── Delete directory = deletes all contents                    │
│   ├── Move directory = atomic move                               │
│   └── ACLs can be set at directory level                        │
│                                                                       │
│ Managing directories:                                                │
│ ├── Create: Portal → Container → + Add Directory              │
│ ├── Rename: Right-click → Rename (instant!)                    │
│ ├── Delete: Right-click → Delete (all contents deleted)       │
│ └── Properties: View/set ACLs, metadata                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Access Control (ACLs)

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACCESS CONTROL (RBAC + ACLs)                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ADLS Gen2 supports TWO layers of access control:                    │
│                                                                       │
│ Layer 1: RBAC (Role-Based Access Control)                           │
│ ├── Coarse-grained (entire storage account or container)         │
│ ├── Roles: Storage Blob Data Reader/Contributor/Owner            │
│ └── Same as regular blob storage                                 │
│                                                                       │
│ Layer 2: POSIX ACLs (fine-grained, per directory/file)            │
│ ├── Permissions: Read (r), Write (w), Execute (x)               │
│ ├── For: Owner, Owning group, Other                              │
│ ├── Like Linux file permissions (rwxr-xr--)                     │
│ └── Set at each directory/file individually                      │
│                                                                       │
│ How they work together:                                              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ RBAC check → If pass → FULL ACCESS (ACLs not checked)      │  │
│ │ RBAC check → If no role → Check ACLs for specific access  │  │
│ │                                                              │  │
│ │ ⚡ If user has RBAC "Storage Blob Data Owner" role,         │  │
│ │   ACLs are SKIPPED (RBAC wins). Use ACLs for fine-grained │  │
│ │   control WHEN users don't have broad RBAC roles.          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Setting ACLs in Portal:                                              │
│ Portal → Container → Directory → Manage ACL                    │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Access ACL (applied to this item):                           │  │
│ │ ┌───────────────────┬───────┬───────┬─────────┐            │  │
│ │ │ Entity            │ Read  │ Write │ Execute │            │  │
│ │ ├───────────────────┼───────┼───────┼─────────┤            │  │
│ │ │ Owner (user1)     │ ✅   │ ✅   │ ✅     │            │  │
│ │ │ Group (data-team) │ ✅   │ ❌   │ ✅     │            │  │
│ │ │ Other             │ ❌   │ ❌   │ ❌     │            │  │
│ │ │ user2@company.com │ ✅   │ ❌   │ ✅     │ (named)    │  │
│ │ └───────────────────┴───────┴───────┴─────────┘            │  │
│ │                                                              │  │
│ │ Default ACL (inherited by new children):                    │  │
│ │ Same structure — applied to files/dirs created inside.     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ Execute permission on directories = list contents              │
│ ⚡ Default ACLs = inheritance (new files get these ACLs)          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Integration with Analytics

```
┌─────────────────────────────────────────────────────────────────────┐
│           ANALYTICS INTEGRATION                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ADLS Gen2 is the central storage for Azure analytics services:      │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │           ┌─────────────────────────────────────┐           │  │
│ │           │      ADLS Gen2 (Data Lake)          │           │  │
│ │           │  abfss://datalake@stdatalakeprod    │           │  │
│ │           └──────────────┬──────────────────────┘           │  │
│ │                          │                                   │  │
│ │   ┌─────────┬───────────┼────────────┬──────────┐          │  │
│ │   │         │           │            │          │          │  │
│ │   ▼         ▼           ▼            ▼          ▼          │  │
│ │ Synapse  Databricks  Data Factory  HDInsight  Power BI    │  │
│ │ (SQL+    (Spark      (ETL/ELT     (Hadoop     (Reports   │  │
│ │  Spark)   notebooks)  pipelines)   clusters)   dashboards)│  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Connection URL format:                                               │
│ abfss://<container>@<account>.dfs.core.windows.net/<path>         │
│ Example: abfss://datalake@stdatalakeprod.dfs.core.windows.net/raw/│
│                                                                       │
│ ⚡ "abfss" = Azure Blob File System Secure (ADLS Gen2 protocol)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_storage_account" "datalake" {
  name                     = "stdatalakeprod"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "ZRS"
  account_kind             = "StorageV2"
  is_hns_enabled           = true  # Enables ADLS Gen2!

  blob_properties {
    delete_retention_policy {
      days = 7
    }
  }
}

resource "azurerm_storage_data_lake_gen2_filesystem" "main" {
  name               = "datalake"
  storage_account_id = azurerm_storage_account.datalake.id
}

resource "azurerm_storage_data_lake_gen2_path" "raw" {
  path               = "raw"
  filesystem_name    = azurerm_storage_data_lake_gen2_filesystem.main.name
  storage_account_id = azurerm_storage_account.datalake.id
  resource           = "directory"
}
```

### Bicep

```bicep
// ADLS Gen2 Storage Account (HNS enabled)
resource dataLakeAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'stdatalakeprod'
  location: resourceGroup().location
  sku: { name: 'Standard_ZRS' }
  kind: 'StorageV2'
  properties: {
    isHnsEnabled: true  // Enables hierarchical namespace (ADLS Gen2)
    accessTier: 'Hot'
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
  }
}

// File system (container with HNS)
resource filesystem 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  name: '${dataLakeAccount.name}/default/datalake'
  properties: {
    publicAccess: 'None'
  }
}
```
  --name stdatalakeprod \
  --resource-group rg-datalake-prod \
  --location centralindia \
  --sku Standard_ZRS \
  --kind StorageV2 \
  --hns true  # hierarchical namespace

# Create file system (container)
az storage fs create --name datalake --account-name stdatalakeprod --auth-mode login

# Create directory
az storage fs directory create \
  --name raw \
  --file-system datalake \
  --account-name stdatalakeprod \
  --auth-mode login

# Upload file
az storage fs file upload \
  --source ./data.csv \
  --path raw/2024/01/data.csv \
  --file-system datalake \
  --account-name stdatalakeprod \
  --auth-mode login

# List directory contents
az storage fs file list \
  --file-system datalake \
  --path raw/ \
  --account-name stdatalakeprod \
  --auth-mode login \
  --output table

# Set ACL on directory
az storage fs access set \
  --acl "user::rwx,group::r-x,other::---" \
  --path raw \
  --file-system datalake \
  --account-name stdatalakeprod \
  --auth-mode login

# Delete file system
az storage fs delete --name datalake --account-name stdatalakeprod --yes
```

---

## Part 8: Real-World Patterns

```
Pattern: Medallion Architecture (Bronze-Silver-Gold)
├── Container: datalake
│   ├── bronze/ (raw data — as ingested)
│   │   └── Data Factory ingests from sources
│   ├── silver/ (cleaned, validated data)
│   │   └── Databricks/Synapse transforms bronze → silver
│   └── gold/ (business-ready, aggregated)
│       └── Power BI reads from gold for dashboards
│
├── Access control:
│   ├── Data Engineers: rwx on bronze, silver, gold
│   ├── Data Scientists: r-x on silver, gold
│   └── Business Analysts: r-x on gold only
│
└── Lifecycle: Archive gold data older than 1 year
```

---

## Quick Reference

```
ADLS Gen2 = Blob Storage + hierarchical namespace (real folders)
Enable = Check "hierarchical namespace" during storage account creation
Cannot be disabled once enabled!

DFS endpoint: https://<account>.dfs.core.windows.net
Connection: abfss://<container>@<account>.dfs.core.windows.net/<path>

Access control:
  RBAC = Coarse-grained (account/container level)
  POSIX ACLs = Fine-grained (directory/file level, rwx)
  RBAC wins over ACLs if user has RBAC role

Key use case: Central data lake for analytics
  Synapse, Databricks, Data Factory, HDInsight, Power BI
```

---

## What's Next?

Next chapter: [Chapter 27: Azure SQL Database](27-azure-sql.md) — The fully managed relational database service for SQL Server in the cloud.
