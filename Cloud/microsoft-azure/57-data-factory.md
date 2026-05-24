# Chapter 57: Azure Data Factory

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Data Factory Fundamentals](#part-1-data-factory-fundamentals)
- [Part 2: Creating a Data Factory (Portal Walkthrough)](#part-2-creating-a-data-factory-portal-walkthrough)
- [Part 3: Linked Services & Datasets](#part-3-linked-services--datasets)
- [Part 4: Pipelines & Activities](#part-4-pipelines--activities)
- [Part 5: Data Flows](#part-5-data-flows)
- [Part 6: Triggers & Monitoring](#part-6-triggers--monitoring)
- [Part 7: Terraform & az CLI Reference](#part-7-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Data Factory (ADF) is a cloud ETL/ELT service for data integration. It moves and transforms data between 100+ sources — databases, files, APIs, SaaS apps — using visual pipelines. No coding required for most scenarios.

```
What you'll learn:
├── Data Factory Fundamentals
│   ├── ETL vs ELT
│   ├── Key concepts (pipeline, activity, dataset, linked service)
│   └── Pricing
├── Creating a Data Factory (Portal)
├── Linked Services & Datasets
├── Pipelines & Activities
├── Data Flows (visual transformations)
├── Triggers & Monitoring
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Data Factory Fundamentals

```
ETL = Extract, Transform, Load
  Source → Transform data → Load to destination

ELT = Extract, Load, Transform
  Source → Load raw data → Transform in destination (e.g., in Synapse)

Data Factory handles both patterns!

Key concepts:
┌─────────────────────────────────────────────────────┐
│ Data Factory                                          │
│                                                       │
│ Linked Service = Connection string to a data store  │
│ (e.g., "connect to this SQL Server" or "this Blob")│
│                                                       │
│ Dataset = Pointer to specific data                  │
│ (e.g., "the Sales table" or "*.csv in /data/")     │
│                                                       │
│ Activity = A single action                          │
│ (e.g., "copy data" or "run stored procedure")      │
│                                                       │
│ Pipeline = Sequence of activities                   │
│ (e.g., "copy → transform → load → notify")       │
│                                                       │
│ Trigger = When to run the pipeline                  │
│ (e.g., "every day at 2 AM" or "when file arrives")│
│                                                       │
│ Integration Runtime = Compute to run activities     │
│ ├── Azure IR: Cloud (default)                      │
│ ├── Self-hosted IR: On-premises data access        │
│ └── Azure-SSIS IR: Run SSIS packages              │
│                                                       │
└─────────────────────────────────────────────────────┘

Pricing:
├── Pipeline orchestration: ~$1 per 1,000 activity runs
├── Data movement: ~$0.25 per DIU-hour
├── Data flows: ~$0.27 per vCore-hour
└── 💡 Inactive pipelines cost nothing
```

---

## Part 2: Creating a Data Factory (Portal Walkthrough)

```
Console → Data factories → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE DATA FACTORY                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-data ▼]                                        │
│ Name: [adf-mycompany-prod]                                         │
│ Region: [Central India ▼]                                          │
│ Version: [V2]                                                       │
│                                                                       │
│ Git configuration:                                                   │
│ ● Configure Git later                                              │
│ ○ Azure DevOps Git                                                │
│ ○ GitHub                                                           │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
│ After creation → [Launch Studio]                                  │
│ Opens: https://adf.azure.com                                      │
│                                                                       │
│ ADF Studio tabs:                                                    │
│ ├── Author: Create pipelines, datasets, data flows              │
│ ├── Monitor: View pipeline runs, activity runs                  │
│ ├── Manage: Linked services, integration runtimes, triggers     │
│ └── Home: Templates, recent, getting started                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Linked Services & Datasets

```
Linked Services (Manage → Linked services → New):

90+ connectors:
├── Azure: Blob, ADLS, SQL, Cosmos, Synapse, Event Hub
├── Databases: SQL Server, PostgreSQL, MySQL, Oracle, MongoDB
├── Files: FTP, SFTP, Amazon S3, Google Cloud Storage
├── SaaS: Salesforce, Dynamics 365, SAP, ServiceNow
├── APIs: REST, OData, HTTP
└── On-premises: Via Self-hosted Integration Runtime

Example: Create Blob Storage linked service
├── Name: [ls_blob_source]
├── Type: [Azure Blob Storage]
├── Authentication: [Managed Identity] (recommended)
├── Storage account: [stmyapp ▼]
└── [Test connection] → [Create]

Datasets (Author → Datasets → New):
├── Linked service: [ls_blob_source]
├── Format: [CSV / Parquet / JSON / Excel / etc.]
├── File path: [data/sales/2024/*.csv]
├── First row as header: ☑
└── Schema: [Import schema] or define manually
```

---

## Part 4: Pipelines & Activities

```
Author → Pipelines → New pipeline

Activity types:
├── Move & transform:
│   ├── Copy data (most used! Source → Sink)
│   ├── Data flow (visual transformation)
│   └── Execute SSIS package
├── Azure:
│   ├── Databricks Notebook
│   ├── Synapse Notebook / SQL script
│   ├── Azure Function
│   ├── Azure ML Execute Pipeline
│   └── HDInsight (Hive/Pig/Spark)
├── Control flow:
│   ├── If Condition
│   ├── ForEach (loop over items)
│   ├── Until (loop until condition)
│   ├── Switch
│   ├── Execute Pipeline (call another pipeline)
│   ├── Wait
│   └── Set Variable / Append Variable
├── General:
│   ├── Web Activity (call REST API)
│   ├── Stored Procedure
│   ├── Lookup (query and return data)
│   ├── Get Metadata (file size, exists, etc.)
│   └── Delete
└── Notifications:
    └── Web Activity → Send to Teams/Slack webhook

Example pipeline (visual):
  [Lookup: Get file list]
       │
  [ForEach: Process each file]
       │
       ├── [Copy: CSV → SQL staging table]
       │        │
       │   [Stored Procedure: Merge to production]
       │
  [Web Activity: Notify Teams "Pipeline complete"]

Parameters:
├── Pipeline parameters: Input values at runtime
├── Variables: Store intermediate values
├── System variables: @pipeline().RunId, @utcnow()
├── Expressions: @concat(), @if(), @formatDateTime()
└── Dynamic content: Use expressions anywhere
```

---

## Part 5: Data Flows

```
Data Flows = Visual data transformation (no code)

Author → Data flows → New data flow

Transformation steps (visual canvas):
├── Source → Read from dataset
├── Filter → WHERE condition
├── Derived Column → Calculate new columns
├── Aggregate → GROUP BY + SUM/AVG/COUNT
├── Join → Combine two streams
├── Union → Stack rows from multiple sources
├── Pivot / Unpivot → Reshape data
├── Sort → ORDER BY
├── Lookup → Reference data enrichment
├── Conditional Split → Route rows by condition
├── Window → Running totals, rankings
├── Exists → Semi-join (filter by existence)
├── Alter Row → Insert/Update/Delete/Upsert policy
└── Sink → Write to destination

Example: Clean and aggregate sales data
  [Source: CSV files]
       │
  [Filter: amount > 0]
       │
  [Derived Column: year = year(orderDate)]
       │
  [Aggregate: group by year, category → sum(amount)]
       │
  [Sink: SQL table "SalesSummary"]

⚡ Data flows run on Spark clusters (auto-provisioned)
⚡ Debug mode: Test transformations with data preview
⚡ TTL: Keep Spark cluster warm to avoid cold-start (8 min)
```

---

## Part 6: Triggers & Monitoring

```
Triggers (Manage → Triggers → New):
├── Schedule: Run at specific times
│   └── Every day at 02:00 UTC, every Monday, etc.
├── Tumbling window: Fixed-size, non-overlapping time windows
│   └── Process data in 1-hour windows, with retry/dependency
├── Event: React to Storage events
│   └── When blob created in container "input/"
└── Custom event: React to Event Grid custom events

Monitoring (Monitor tab):
├── Pipeline runs: Status, duration, errors
├── Activity runs: Each step's details
├── Trigger runs: Which triggers fired
├── Data flow debug: Execution details
└── Integration runtime: Health and status

Alerts:
├── ADF → Alerts → New alert rule
├── Metrics: Pipeline failed runs, activity failed runs
├── Action: Email, SMS, webhook
└── Example: Alert when any pipeline fails
```

---

## Part 7: Terraform & az CLI Reference

### Terraform

```hcl
resource "azurerm_data_factory" "main" {
  name                = "adf-mycompany-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  identity {
    type = "SystemAssigned"
  }
}

resource "azurerm_data_factory_linked_service_azure_blob_storage" "source" {
  name              = "ls_blob_source"
  data_factory_id   = azurerm_data_factory.main.id
  connection_string = azurerm_storage_account.main.primary_connection_string
}

resource "azurerm_data_factory_pipeline" "etl" {
  name            = "pl_daily_etl"
  data_factory_id = azurerm_data_factory.main.id

  activities_json = jsonencode([{
    name = "CopyData"
    type = "Copy"
    # ... activity configuration
  }])
}
```

### Bicep

```bicep
// Data Factory
resource dataFactory 'Microsoft.DataFactory/factories@2018-06-01' = {
  name: 'adf-mycompany-prod'
  location: resourceGroup().location
  identity: { type: 'SystemAssigned' }
  properties: {
    publicNetworkAccess: 'Enabled'
  }
}

// Linked Service (Azure SQL)
resource linkedService 'Microsoft.DataFactory/factories/linkedservices@2018-06-01' = {
  parent: dataFactory
  name: 'ls-azure-sql'
  properties: {
    type: 'AzureSqlDatabase'
    typeProperties: {
      connectionString: 'Server=tcp:sql-myapp.database.windows.net;Database=mydb;'
    }
  }
}

// Pipeline
resource pipeline 'Microsoft.DataFactory/factories/pipelines@2018-06-01' = {
  parent: dataFactory
  name: 'pl-copy-data'
  properties: {
    activities: [
      {
        name: 'CopyData'
        type: 'Copy'
        inputs: [{ referenceName: 'src-dataset', type: 'DatasetReference' }]
        outputs: [{ referenceName: 'dest-dataset', type: 'DatasetReference' }]
        typeProperties: {
          source: { type: 'AzureSqlSource' }
          sink: { type: 'AzureSqlSink' }
        }
      }
    ]
  }
}
```
az datafactory create \
  --name adf-mycompany-prod \
  --resource-group rg-data \
  --location centralindia

# List pipelines
az datafactory pipeline list \
  --factory-name adf-mycompany-prod \
  --resource-group rg-data -o table

# Trigger pipeline run
az datafactory pipeline create-run \
  --factory-name adf-mycompany-prod \
  --resource-group rg-data \
  --name pl_daily_etl

# Monitor pipeline runs
az datafactory pipeline-run query-by-factory \
  --factory-name adf-mycompany-prod \
  --resource-group rg-data \
  --last-updated-after "2024-01-01T00:00:00Z" \
  --last-updated-before "2024-12-31T23:59:59Z"

# Delete
az datafactory delete --name adf-mycompany-prod --resource-group rg-data --yes
```

---

## Real-World Patterns

### Pattern 1: ETL from Multiple Sources to Data Lake

```
┌─────────────────────────────────────────────────┐
│  Daily ETL Pipeline                             │
├─────────────────────────────────────────────────┤
│                                                 │
│  Schedule Trigger (daily 2 AM)                  │
│       │                                         │
│       ▼                                         │
│  Copy Activity: SQL Server → Data Lake (raw)   │
│  Copy Activity: Salesforce → Data Lake (raw)   │
│  Copy Activity: REST API → Data Lake (raw)     │
│       │                                         │
│       ▼                                         │
│  Data Flow: Clean + Transform + Join           │
│       │                                         │
│       ▼                                         │
│  Write to Data Lake (curated zone)              │
│       │                                         │
│       ▼                                         │
│  Stored Proc: Load to SQL DW                   │
│                                                 │
│  Self-Hosted IR: for on-prem SQL Server        │
│  Managed VNet IR: for private endpoints         │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Data Factory = Cloud ETL/ELT for data integration
ADF Studio: https://adf.azure.com

Linked Service → Dataset → Activity → Pipeline → Trigger

Activities: Copy, Data Flow, Stored Proc, Databricks, Function, etc.
Data Flows: Visual Spark-based transformations (no code)
Triggers: Schedule | Tumbling Window | Event (blob) | Custom Event

Integration Runtime:
├── Azure IR: Cloud compute (default)
├── Self-hosted IR: On-premises access
└── Azure-SSIS IR: Run SSIS packages

90+ connectors: SQL, Blob, Cosmos, S3, Salesforce, SAP, REST, etc.

Pricing: ~$1/1000 runs + $0.25/DIU-hr (data movement)
⚡ Inactive = free
```

---

## What's Next?

Next chapter: [Chapter 58: Azure Databricks](58-databricks.md) — Unified analytics platform for big data and ML with Apache Spark.
