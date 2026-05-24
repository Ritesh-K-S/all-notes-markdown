# Chapter 58: Azure Databricks

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Databricks Fundamentals](#part-1-databricks-fundamentals)
- [Part 2: Creating a Databricks Workspace (Portal Walkthrough)](#part-2-creating-a-databricks-workspace-portal-walkthrough)
- [Part 3: Clusters & Compute](#part-3-clusters--compute)
- [Part 4: Notebooks & Jobs](#part-4-notebooks--jobs)
- [Part 5: Unity Catalog & Delta Lake](#part-5-unity-catalog--delta-lake)
- [Part 6: Terraform & az CLI Reference](#part-6-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Databricks is a Spark-based analytics platform built in collaboration with Databricks. It provides notebooks, clusters, jobs, and a lakehouse architecture (Delta Lake) for big data processing, machine learning, and data engineering.

```
What you'll learn:
├── Databricks Fundamentals
│   ├── What is a Lakehouse
│   ├── Databricks vs Synapse vs HDInsight
│   └── Pricing (DBU-based)
├── Creating a Workspace (Portal)
├── Clusters & Compute
├── Notebooks & Jobs
├── Unity Catalog (governance) & Delta Lake (storage)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Databricks Fundamentals

```
Databricks = Apache Spark + Collaborative workspace + Lakehouse

Lakehouse = Data Lake + Data Warehouse features
├── Store data as files (Parquet/Delta) in Data Lake ← Lake
├── Get SQL, ACID transactions, schema enforcement ← Warehouse
└── One copy of data for both BI and ML

Databricks vs others:
┌──────────────┬──────────────┬──────────────┬──────────────┐
│              │ Databricks   │ Synapse      │ HDInsight    │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Best for     │ Data eng + ML│ DW + analytics│ Hadoop      │
│ Compute      │ Spark clusters│ SQL/Spark pools│ Hadoop/Spark│
│ Notebook     │ Excellent ✅ │ Good          │ Jupyter      │
│ Delta Lake   │ Native ✅   │ Supported     │ Manual       │
│ ML           │ MLflow ✅   │ Limited       │ Limited      │
│ SQL          │ SQL Warehouse│ Dedicated pool│ Hive         │
│ Collaboration│ Excellent   │ Good          │ Basic        │
│ Cost model   │ DBU/hour    │ DWU/vCore-hr  │ VM/hour      │
└──────────────┴──────────────┴──────────────┴──────────────┘

Pricing (DBU = Databricks Unit):
├── Jobs Compute: ~$0.15/DBU (batch workloads)
├── All-Purpose Compute: ~$0.40/DBU (interactive/dev)
├── SQL Warehouse: ~$0.22/DBU (BI queries)
└── + Azure VM cost underneath
```

---

## Part 2: Creating a Databricks Workspace (Portal Walkthrough)

```
Console → Azure Databricks → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE AZURE DATABRICKS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-databricks ▼]                                  │
│ Workspace name: [dbw-mycompany-prod]                               │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Pricing tier:                                                        │
│ ○ Standard (no Unity Catalog, basic features)                     │
│ ● Premium (Unity Catalog, RBAC, audit logs) ← Recommended       │
│ ○ Trial (Premium features, 14 days free)                          │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
│ After creation → [Launch Workspace]                               │
│ Opens: https://adb-1234567890.azuredatabricks.net                 │
│                                                                       │
│ Workspace UI:                                                        │
│ ├── Workspace: Notebooks, folders                                 │
│ ├── Repos: Git integration                                        │
│ ├── Compute: Clusters, SQL Warehouses                             │
│ ├── Workflows: Scheduled jobs                                     │
│ ├── Data: Catalog, databases, tables                              │
│ └── SQL Editor: Write and run SQL                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Clusters & Compute

```
Compute → Create compute

All-Purpose Cluster (interactive development):
├── Name: [dev-cluster]
├── Runtime: [14.3 LTS (Spark 3.5, Scala 2.12)]
├── Node type: [Standard_D4s_v3 (4 cores, 16 GB)]
├── Worker type: ● Fixed [2] workers  ○ Autoscale [2-8]
├── Auto termination: [120] minutes (saves cost!)
└── [Create Compute]

Job Cluster (batch processing):
├── Created automatically when a job runs
├── Terminated automatically after job completes
├── Cheaper DBU rate (~$0.15/DBU vs $0.40/DBU)
└── Always use job clusters for production workloads!

SQL Warehouse (BI/SQL queries):
├── Compute → SQL Warehouses → Create
├── Size: [Small (2X-Small to 4X-Large)]
├── Auto stop: [10] minutes
├── Serverless: ☑ (instant start, no warm-up)
└── Use for: Dashboards, BI tools (Power BI, Tableau)

⚡ Always set auto-termination to avoid runaway costs!
⚡ Use job clusters for scheduled workloads (cheaper)
⚡ Spot instances: Save up to 80% for fault-tolerant workloads
```

---

## Part 4: Notebooks & Jobs

```
Notebooks:
├── Workspace → Create → Notebook
├── Languages: Python, Scala, SQL, R (switch per cell!)
├── Attach to cluster, then run cells interactively
├── Visualizations built-in (charts, dashboards)
├── Collaboration: Multiple users edit simultaneously
└── Git integration: Sync notebooks to repos

Example notebook cells:
  # Cell 1 (Python)
  df = spark.read.format("delta").load("/mnt/data/sales")
  display(df.limit(10))

  -- Cell 2 (SQL)
  SELECT category, SUM(amount) as total
  FROM delta.`/mnt/data/sales`
  GROUP BY category

  # Cell 3 (Python)
  from pyspark.ml.feature import VectorAssembler
  from pyspark.ml.regression import LinearRegression
  # ... build ML model

Jobs (scheduled execution):
├── Workflows → Create job
├── Task: Notebook, Python script, JAR, SQL, dbt
├── Schedule: Cron expression or manual trigger
├── Cluster: New job cluster (cheaper!) or existing
├── Alerts: Email on failure
├── Multi-task jobs: DAG of dependent tasks
│
│   [Extract] → [Transform] → [Load]
│       │                        │
│       └──── [Quality Check] ───┘
```

---

## Part 5: Unity Catalog & Delta Lake

```
Unity Catalog = Centralized governance for all data

Hierarchy:
├── Metastore (one per region)
│   ├── Catalog (e.g., "production", "development")
│   │   ├── Schema (e.g., "sales", "marketing")
│   │   │   ├── Table (e.g., "orders")
│   │   │   ├── View
│   │   │   ├── Function
│   │   │   └── Volume (files)
│   │   └── ...
│   └── ...

Features:
├── Fine-grained access control (GRANT/REVOKE on tables)
├── Data lineage (see where data comes from/goes to)
├── Audit logs (who accessed what data)
├── Cross-workspace sharing (share tables between workspaces)
└── Row/column-level security

Delta Lake = Open-source storage format
├── ACID transactions on data lake files
├── Schema enforcement and evolution
├── Time travel (query historical versions)
│   SELECT * FROM orders TIMESTAMP AS OF '2024-01-01'
│   SELECT * FROM orders VERSION AS OF 5
├── MERGE (upsert) support
│   MERGE INTO target USING source
│   ON target.id = source.id
│   WHEN MATCHED THEN UPDATE SET ...
│   WHEN NOT MATCHED THEN INSERT ...
├── OPTIMIZE: Compacts small files
├── VACUUM: Removes old files
└── Z-ORDER: Co-locate related data for faster queries
```

---

## Part 6: Terraform & az CLI Reference

### Terraform

```hcl
resource "azurerm_databricks_workspace" "main" {
  name                = "dbw-mycompany-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "premium"

  managed_resource_group_name = "rg-databricks-managed"
}
```

### Bicep

```bicep
// Databricks Workspace
resource databricks 'Microsoft.Databricks/workspaces@2023-02-01' = {
  name: 'dbw-mycompany-prod'
  location: resourceGroup().location
  sku: { name: 'premium' }
  properties: {
    managedResourceGroupId: subscriptionResourceId('Microsoft.Resources/resourceGroups', 'rg-databricks-managed')
    parameters: {
      enableNoPublicIp: { value: false }
    }
  }
}
```

### az CLI

```bash
# Create workspace
az databricks workspace create \
  --name dbw-mycompany-prod \
  --resource-group rg-databricks \
  --location centralindia \
  --sku premium

# Show workspace
az databricks workspace show \
  --name dbw-mycompany-prod \
  --resource-group rg-databricks

# List workspaces
az databricks workspace list --resource-group rg-databricks -o table

# Update workspace (e.g., enable no-public-IP)
az databricks workspace update \
  --name dbw-mycompany-prod \
  --resource-group rg-databricks \
  --enable-no-public-ip true

# Delete workspace
az databricks workspace delete \
  --name dbw-mycompany-prod \
  --resource-group rg-databricks --yes

# Wait for workspace to be provisioned
az databricks workspace wait \
  --name dbw-mycompany-prod \
  --resource-group rg-databricks \
  --created
```

---

## Real-World Patterns

### Pattern 1: Lakehouse Architecture

```
┌─────────────────────────────────────────────────┐
│  Databricks Lakehouse                           │
├─────────────────────────────────────────────────┤
│                                                 │
│  Raw Data ─→ Bronze ─→ Silver ─→ Gold           │
│  (landing)   (clean)   (enrich)  (serve)       │
│                                                 │
│  Bronze: Ingest raw JSON/CSV with Auto Loader  │
│  Silver: Schema enforcement, dedup, join        │
│  Gold: Aggregated business tables               │
│                                                 │
│  All layers stored as Delta Lake tables         │
│  - ACID transactions                            │
│  - Time travel (version history)                │
│  - Schema evolution                             │
│                                                 │
│  Unity Catalog: Governance across all tables    │
│  Power BI: Direct query on Gold tables          │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Databricks = Spark + Lakehouse + Collaborative Workspace

Compute types:
├── All-Purpose: Interactive dev (~$0.40/DBU) — always auto-terminate!
├── Job Cluster: Batch processing (~$0.15/DBU) — auto-created/destroyed
└── SQL Warehouse: BI queries (~$0.22/DBU) — serverless option

Delta Lake: ACID on data lake, time travel, MERGE, OPTIMIZE, VACUUM
Unity Catalog: Governance (access control, lineage, audit)

Notebooks: Python/Scala/SQL/R, collaborative, Git-integrated
Jobs: Scheduled notebook/script execution with DAG dependencies

Premium tier: Required for Unity Catalog, RBAC, audit logs

⚡ Set auto-termination on all clusters!
⚡ Use job clusters for production (cheaper)
⚡ Spot instances save up to 80%
```

---

## What's Next?

Next chapter: [Chapter 59: Azure Stream Analytics](59-stream-analytics.md) — Real-time stream processing with SQL-like query language.
