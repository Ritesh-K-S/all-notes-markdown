# Chapter 56: Azure Synapse Analytics

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Synapse Fundamentals](#part-1-synapse-fundamentals)
- [Part 2: Creating a Synapse Workspace (Portal Walkthrough)](#part-2-creating-a-synapse-workspace-portal-walkthrough)
- [Part 3: SQL Pools](#part-3-sql-pools)
- [Part 4: Spark Pools](#part-4-spark-pools)
- [Part 5: Synapse Pipelines](#part-5-synapse-pipelines)
- [Part 6: Serverless SQL Pool](#part-6-serverless-sql-pool)
- [Part 7: Terraform & az CLI Reference](#part-7-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Synapse Analytics is a unified analytics platform that brings together data warehousing (SQL pools), big data processing (Spark pools), and data integration (pipelines) into one service. Think of it as a one-stop-shop for all analytics workloads.

```
What you'll learn:
├── Synapse Fundamentals
│   ├── What is a data warehouse
│   ├── Synapse vs SQL Database vs Databricks
│   └── Workspace architecture
├── Creating a Synapse Workspace (Portal)
├── SQL Pools (dedicated — data warehouse)
├── Spark Pools (big data processing)
├── Synapse Pipelines (ETL/ELT)
├── Serverless SQL Pool (query files directly)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Synapse Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           SYNAPSE ANALYTICS OVERVIEW                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Synapse = Data Warehouse + Big Data + Data Integration            │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Synapse Workspace (web.azuresynapse.net)                      │   │
│ │                                                               │   │
│ │ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ │   │
│ │ │Dedicated  │ │Serverless │ │ Spark     │ │ Pipelines │ │   │
│ │ │SQL Pool   │ │SQL Pool   │ │ Pool      │ │ (ETL)     │ │   │
│ │ │(DW)       │ │(on-demand)│ │(big data) │ │           │ │   │
│ │ └───────────┘ └───────────┘ └───────────┘ └───────────┘ │   │
│ │                                                               │   │
│ │ Linked Services → Data Lake, Blob, SQL, Cosmos, etc.       │   │
│ │ Synapse Studio → Web IDE for SQL, Spark, Pipelines         │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Dedicated SQL Pool: Pre-provisioned DW (pay always)         │
│ ├── Serverless SQL Pool: Query files on demand (pay per query)  │
│ ├── Spark Pool: Apache Spark for Python/Scala/R processing      │
│ ├── Pipelines: Data Factory-compatible ETL orchestration        │
│ └── Data Lake: ADLS Gen2 (primary storage for the workspace)   │
│                                                                       │
│ Pricing:                                                             │
│ ├── Dedicated SQL: ~$1.20/DWU/hour (100 DWU min = $2.88/hr)   │
│ ├── Serverless SQL: ~$5 per TB scanned                          │
│ ├── Spark: ~$0.16/vCore/hour                                    │
│ └── Pipelines: Same as Data Factory pricing                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Synapse Workspace (Portal Walkthrough)

```
Console → Azure Synapse Analytics → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE SYNAPSE WORKSPACE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-analytics ▼]                                   │
│ Workspace name: [syn-mycompany-prod]                               │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Data Lake Storage:                                                   │
│ Account name: [adlsmycompany ▼] (ADLS Gen2)                      │
│ File system: [synapse] (container for workspace data)             │
│                                                                       │
│ Security:                                                            │
│ SQL admin username: [sqladminuser]                                  │
│ SQL admin password: [••••••••]                                     │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
│ After creation → Open Synapse Studio                              │
│ URL: https://web.azuresynapse.net                                  │
│                                                                       │
│ Synapse Studio = Web-based IDE with:                               │
│ ├── Data hub (explore data sources)                               │
│ ├── Develop hub (SQL scripts, notebooks, pipelines)              │
│ ├── Integrate hub (pipelines / data flows)                       │
│ ├── Monitor hub (pipeline runs, SQL queries)                     │
│ └── Manage hub (SQL pools, Spark pools, linked services)         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: SQL Pools

```
Dedicated SQL Pool = Enterprise data warehouse

Synapse Studio → Manage → SQL pools → [+ New]

Name: [dwh_sales]
Performance level: [DW100c] (100 DWU — smallest)

DWU (Data Warehouse Units) = CPU + Memory + IO bundle
├── DW100c: Dev/test (~$1.20/hr)
├── DW500c: Small production
├── DW1000c: Medium workloads
├── DW30000c: Maximum scale
└── Can pause when not in use! (stop paying)

Key features:
├── MPP (Massively Parallel Processing)
│   Query distributed across 60 distributions
├── Columnstore indexes (fast analytics queries)
├── Workload management (classify & prioritize queries)
├── Result set caching (instant repeat queries)
├── Materialized views (pre-computed aggregations)
└── Pause/Resume (stop billing when not querying)

Pause SQL Pool:
  Synapse Studio → Manage → SQL pools → dwh_sales → [Pause]
  ⚡ No charges while paused! Great for dev/test.

Example query:
  SELECT
    ProductCategory,
    SUM(SalesAmount) as TotalSales,
    COUNT(*) as TransactionCount
  FROM dbo.FactSales
  JOIN dbo.DimProduct ON FactSales.ProductKey = DimProduct.ProductKey
  GROUP BY ProductCategory
  ORDER BY TotalSales DESC
```

---

## Part 4: Spark Pools

```
Spark Pool = Apache Spark for big data processing

Synapse Studio → Manage → Apache Spark pools → [+ New]

Name: [sparkpool1]
Node size: [Small (4 vCores, 32 GB)]
Autoscale: ☑ Enabled (3-10 nodes)
Auto-pause: ☑ After [15] minutes idle

Use Spark for:
├── Processing large files (Parquet, CSV, JSON) in Data Lake
├── Machine learning (SparkML, scikit-learn)
├── Data transformation (Python/Scala/R notebooks)
└── Complex ETL that SQL can't handle

Notebook example (PySpark):
  # Read from Data Lake
  df = spark.read.parquet("abfss://data@adlsmycompany.dfs.core.windows.net/sales/")

  # Transform
  result = df.groupBy("category").agg(
      sum("amount").alias("total_sales"),
      count("*").alias("transaction_count")
  )

  # Write to dedicated SQL pool
  result.write \
      .synapsesql("dwh_sales.dbo.SalesSummary") \
      .mode("overwrite")

⚡ Auto-pause: Cluster shuts down after idle period (saves cost)
⚡ Auto-scale: Nodes scale up/down based on workload
```

---

## Part 5: Synapse Pipelines

```
Pipelines = Data Factory-compatible ETL/ELT orchestration

Synapse Studio → Integrate → [+] → Pipeline

Same as Azure Data Factory:
├── Activities: Copy, Data Flow, Notebook, SQL script
├── Triggers: Schedule, tumbling window, event-based
├── Linked Services: Connect to any data source
├── Datasets: Define data structure
└── Data Flows: Visual data transformation (no code)

Example pipeline:
  [Copy Activity]                    [Notebook Activity]
  Copy CSV from Blob           →    Transform with Spark
  to Data Lake                       (clean, aggregate)
       │                                    │
       └──────────── → ──────────── → ─────┘
                                            │
                                     [SQL Script]
                                     Load into DW

⚡ Pipelines are essentially Data Factory built into Synapse
⚡ If you already know Data Factory, you know Synapse Pipelines
```

---

## Part 6: Serverless SQL Pool

```
Serverless SQL Pool = Query files directly, no infrastructure!

Built-in (always available, no provisioning needed)
Pay only for data scanned (~$5 per TB)

Query Parquet files in Data Lake:
  SELECT
    category,
    SUM(amount) as total
  FROM OPENROWSET(
    BULK 'https://adlsmycompany.dfs.core.windows.net/data/sales/*.parquet',
    FORMAT = 'PARQUET'
  ) AS sales
  GROUP BY category

Query CSV files:
  SELECT *
  FROM OPENROWSET(
    BULK 'https://adlsmycompany.dfs.core.windows.net/data/logs/*.csv',
    FORMAT = 'CSV',
    HEADER_ROW = TRUE
  ) AS logs
  WHERE status_code = 500

Create external tables (reusable views over files):
  CREATE EXTERNAL TABLE dbo.Sales
  WITH (
    LOCATION = 'data/sales/',
    DATA_SOURCE = DataLake,
    FILE_FORMAT = ParquetFormat
  )

⚡ No infrastructure to manage!
⚡ Great for ad-hoc exploration of data in the lake
⚡ Supports Parquet, CSV, JSON, Delta Lake formats
```

---

## Part 7: Terraform & az CLI Reference

### Terraform

```hcl
resource "azurerm_synapse_workspace" "main" {
  name                                 = "syn-mycompany-prod"
  resource_group_name                  = azurerm_resource_group.main.name
  location                             = azurerm_resource_group.main.location
  storage_data_lake_gen2_filesystem_id = azurerm_storage_data_lake_gen2_filesystem.main.id
  sql_administrator_login              = "sqladminuser"
  sql_administrator_login_password     = var.sql_password

  identity {
    type = "SystemAssigned"
  }
}

resource "azurerm_synapse_sql_pool" "dwh" {
  name                 = "dwh_sales"
  synapse_workspace_id = azurerm_synapse_workspace.main.id
  sku_name             = "DW100c"
  create_mode          = "Default"
}

resource "azurerm_synapse_spark_pool" "spark" {
  name                 = "sparkpool1"
  synapse_workspace_id = azurerm_synapse_workspace.main.id
  node_size_family     = "MemoryOptimized"
  node_size            = "Small"
  node_count           = 3

  auto_pause {
    delay_in_minutes = 15
  }

  auto_scale {
    min_node_count = 3
    max_node_count = 10
  }
}
```

### Bicep

```bicep
// Synapse Workspace
resource synapse 'Microsoft.Synapse/workspaces@2021-06-01' = {
  name: 'syn-mycompany-prod'
  location: resourceGroup().location
  identity: { type: 'SystemAssigned' }
  properties: {
    defaultDataLakeStorage: {
      accountUrl: 'https://${dataLakeAccount.name}.dfs.core.windows.net'
      filesystem: 'synapse'
    }
    sqlAdministratorLogin: 'sqladmin'
    sqlAdministratorLoginPassword: sqlPassword
  }
}

// Dedicated SQL Pool
resource sqlPool 'Microsoft.Synapse/workspaces/sqlPools@2021-06-01' = {
  parent: synapse
  name: 'dwh-prod'
  location: resourceGroup().location
  sku: { name: 'DW100c' }
  properties: {
    createMode: 'Default'
    collation: 'SQL_Latin1_General_CP1_CI_AS'
  }
}

// Spark Pool
resource sparkPool 'Microsoft.Synapse/workspaces/bigDataPools@2021-06-01' = {
  parent: synapse
  name: 'sparkpool1'
  location: resourceGroup().location
  properties: {
    nodeSize: 'Small'
    nodeSizeFamily: 'MemoryOptimized'
    nodeCount: 3
    autoScale: { enabled: true, minNodeCount: 3, maxNodeCount: 10 }
    autoPause: { enabled: true, delayInMinutes: 15 }
    sparkVersion: '3.3'
  }
}
```
  --sql-admin-login-user sqladminuser \
  --sql-admin-login-password "SecureP@ss123!"

# Create dedicated SQL pool
az synapse sql pool create \
  --name dwh_sales \
  --workspace-name syn-mycompany-prod \
  --resource-group rg-analytics \
  --performance-level DW100c

# Pause SQL pool (stop billing)
az synapse sql pool pause --name dwh_sales \
  --workspace-name syn-mycompany-prod --resource-group rg-analytics

# Resume SQL pool
az synapse sql pool resume --name dwh_sales \
  --workspace-name syn-mycompany-prod --resource-group rg-analytics

# Create Spark pool
az synapse spark pool create \
  --name sparkpool1 \
  --workspace-name syn-mycompany-prod \
  --resource-group rg-analytics \
  --node-size Small --node-count 3 \
  --enable-auto-pause --delay 15 \
  --enable-auto-scale --min-node-count 3 --max-node-count 10

# Delete workspace
az synapse workspace delete --name syn-mycompany-prod --resource-group rg-analytics --yes
```

---

## Real-World Patterns

### Pattern 1: Modern Data Warehouse

```
┌─────────────────────────────────────────────────┐
│  End-to-End Analytics Pipeline                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  Sources          Ingest        Transform       │
│  ┌────────┐                                      │
│  │ SQL DB  │──┐                                  │
│  └────────┘  │  Synapse      Spark Pool     │
│  ┌────────┐  ├─→ Pipeline ─→ (clean/enrich)   │
│  │ APIs    │──┤                  │               │
│  └────────┘  │                  ▼               │
│  ┌────────┐  │          Dedicated SQL Pool   │
│  │ Files   │──┘          (serve to Power BI)    │
│  └────────┘                                      │
│                                                 │
│  Serverless SQL: Ad-hoc queries on Data Lake    │
│  Spark Pool: ML model training on same data     │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Synapse = Data Warehouse + Big Data + Data Integration (one platform)

Components:
├── Dedicated SQL Pool: MPP data warehouse (DWU-based, can pause)
├── Serverless SQL Pool: Query files directly ($5/TB scanned)
├── Spark Pool: Apache Spark (PySpark/Scala notebooks, auto-pause)
├── Pipelines: Data Factory-compatible ETL/ELT
└── Synapse Studio: Web IDE for everything

Pricing:
├── Dedicated SQL: ~$1.20/DWU/hr (pause when not in use!)
├── Serverless SQL: ~$5/TB scanned
├── Spark: ~$0.16/vCore/hr (auto-pause saves cost)
└── Pipelines: Per activity run

Use Dedicated SQL Pool for: Structured analytics, dashboards, BI
Use Serverless SQL for: Ad-hoc queries on Data Lake files
Use Spark for: ML, complex transformations, large-scale processing
```

---

## What's Next?

Next chapter: [Chapter 57: Azure Data Factory](57-data-factory.md) — Cloud-based ETL/ELT service for data integration and orchestration.
