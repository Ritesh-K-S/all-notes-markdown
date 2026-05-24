# Chapter 60: Azure Machine Learning

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure ML Fundamentals](#part-1-azure-ml-fundamentals)
- [Part 2: Creating a Workspace (Portal Walkthrough)](#part-2-creating-a-workspace-portal-walkthrough)
- [Part 3: Compute & Data](#part-3-compute--data)
- [Part 4: Training Models](#part-4-training-models)
- [Part 5: Deploying Models (Endpoints)](#part-5-deploying-models-endpoints)
- [Part 6: MLOps & Pipelines](#part-6-mlops--pipelines)
- [Part 7: Terraform & az CLI Reference](#part-7-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Machine Learning is an end-to-end platform for building, training, deploying, and managing machine learning models. It supports both code-first (SDK/CLI) and low-code (Designer, AutoML) approaches.

```
What you'll learn:
├── Azure ML Fundamentals
│   ├── What is ML (simple explanation)
│   ├── Workspace architecture
│   └── Code-first vs low-code
├── Creating a Workspace (Portal)
├── Compute & Data
├── Training Models (Notebooks, AutoML, Designer)
├── Deploying Models (Endpoints)
├── MLOps & Pipelines
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Azure ML Fundamentals

```
Machine Learning = Teaching computers to learn from data

Simple example:
├── Data: 1000 house prices with features (size, bedrooms, location)
├── Training: Algorithm learns relationship between features → price
├── Model: The learned relationship
├── Prediction: Give it a new house → predicts the price

Azure ML Workspace architecture:
┌─────────────────────────────────────────────────────────────┐
│ Azure ML Workspace                                            │
│                                                               │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│ │ Compute  │ │ Data     │ │ Models   │ │Endpoints │      │
│ │          │ │          │ │          │ │          │      │
│ │ Instance │ │ Datasets │ │ Registry │ │ Online   │      │
│ │ Cluster  │ │ Datastores│ │ Versions │ │ Batch    │      │
│ │ Serverless│ │          │ │          │ │          │      │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
│                                                               │
│ Connected resources (auto-created):                          │
│ ├── Storage Account (datasets, models, logs)                │
│ ├── Key Vault (secrets, connection strings)                  │
│ ├── Application Insights (monitoring)                        │
│ └── Container Registry (Docker images for deployment)       │
│                                                               │
│ Ways to work:                                                │
│ ├── ML Studio (web UI) → https://ml.azure.com              │
│ ├── Python SDK v2 (code-first)                              │
│ ├── Azure CLI ml extension (automation)                     │
│ └── VS Code extension (local development)                   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Workspace (Portal Walkthrough)

```
Console → Machine Learning → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE AZURE ML WORKSPACE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-ml ▼]                                          │
│ Workspace name: [mlw-mycompany]                                    │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Storage account: [stmlmycompany] (auto-created)                   │
│ Key vault: [kv-mlmycompany] (auto-created)                        │
│ Application insights: [ai-mlmycompany] (auto-created)             │
│ Container registry: [None] (created on first deployment)          │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
│ After creation → [Launch studio]                                  │
│ URL: https://ml.azure.com                                          │
│                                                                       │
│ ML Studio sections:                                                  │
│ ├── Authoring                                                      │
│ │   ├── Notebooks: Jupyter notebooks                             │
│ │   ├── Automated ML: No-code model training                    │
│ │   └── Designer: Drag-and-drop pipeline builder                │
│ ├── Assets                                                         │
│ │   ├── Data: Datasets and datastores                            │
│ │   ├── Jobs: Training runs                                      │
│ │   ├── Models: Registered models                                │
│ │   ├── Endpoints: Deployed models                               │
│ │   └── Components: Reusable pipeline steps                     │
│ └── Manage                                                         │
│     └── Compute: Instances, clusters, serverless                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Compute & Data

```
Compute types:
├── Compute Instance: Your personal dev VM (Jupyter, VS Code, terminal)
│   Size: [Standard_DS3_v2 (4 cores, 14 GB)]
│   ⚡ Auto-shutdown schedule to save cost!
│
├── Compute Cluster: Multi-node for training jobs
│   Min nodes: [0] (scale to zero when idle!)
│   Max nodes: [4]
│   Size: [Standard_NC6] (GPU for deep learning)
│
├── Serverless Compute: No cluster management
│   Automatically provisions compute for each job
│   Pay only while job runs
│
└── Attached Compute: Use existing resources
    Databricks, HDInsight, VMs, AKS

Data:
├── Datastores: Connection to storage (Blob, ADLS, SQL, etc.)
│   Default: workspace Storage Account (auto-connected)
│
├── Data Assets: Pointer to specific data
│   Types:
│   ├── URI File: Single file
│   ├── URI Folder: Folder of files
│   └── MLTable: Tabular data with schema
│
│ ML Studio → Data → [+ Create]
│ Name: [house-prices]
│ Type: [MLTable]
│ Source: [From Azure storage / local files / web URL]
```

---

## Part 4: Training Models

```
Three ways to train:

1. Notebooks (code-first):
   ML Studio → Notebooks → Create
   Attach to compute instance, write Python:

   from azure.ai.ml import MLClient, command
   from azure.identity import DefaultAzureCredential

   ml_client = MLClient(DefaultAzureCredential(), subscription_id, rg, ws)

   job = command(
       code="./src",
       command="python train.py --data ${{inputs.data}}",
       inputs={"data": Input(type="uri_folder", path="azureml://datastores/...")},
       environment="AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest",
       compute="cpu-cluster",
   )
   ml_client.jobs.create_or_update(job)

2. AutoML (no-code):
   ML Studio → Automated ML → [+ New]
   ├── Select dataset: [house-prices]
   ├── Task type: Regression (predict a number)
   ├── Target column: [price]
   ├── Compute: [cpu-cluster]
   └── Run → AutoML tries many algorithms, picks the best!
   Results: Best model, metrics (R², RMSE), feature importance

3. Designer (drag-and-drop):
   ML Studio → Designer → [+ New]
   Drag components:
   [Dataset] → [Split Data] → [Train Model] → [Score] → [Evaluate]
   ⚡ Good for learning, but code-first is preferred for production
```

---

## Part 5: Deploying Models (Endpoints)

```
After training → Deploy model for predictions

Online Endpoints (real-time):
├── ML Studio → Models → [model-name] → Deploy → Online endpoint
├── Endpoint name: [house-price-predictor]
├── Deployment name: [blue]
├── Instance type: [Standard_DS3_v2]
├── Instance count: [1]
│
├── Call the endpoint:
│   POST https://house-price-predictor.centralindia.inference.ml.azure.com/score
│   Authorization: Bearer <key>
│   Body: {"data": [{"size": 1500, "bedrooms": 3, "location": "Mumbai"}]}
│   Response: {"predictions": [4500000]}
│
├── Blue/Green deployments:
│   Deploy v2 as "green" → Test → Shift traffic → Remove "blue"
│
└── Managed vs Kubernetes endpoints:
    Managed: Azure handles infra (recommended)
    Kubernetes: Deploy to your AKS cluster

Batch Endpoints (large-scale scoring):
├── Process millions of records (no real-time needed)
├── Input: File in storage → Output: Predictions file
├── Runs as a job on compute cluster
└── Use for: Monthly batch scoring, data processing
```

---

## Part 6: MLOps & Pipelines

```
ML Pipelines = Reproducible, automated ML workflows

Pipeline example:
  [Prepare Data] → [Train Model] → [Evaluate] → [Register Model] → [Deploy]

Define in code (Python SDK v2):
  from azure.ai.ml.dsl import pipeline

  @pipeline()
  def training_pipeline(data_input):
      prepare = prepare_component(data=data_input)
      train = train_component(data=prepare.outputs.processed_data)
      evaluate = evaluate_component(model=train.outputs.model)
      return {"model": train.outputs.model}

  pipeline_job = training_pipeline(data_input=data_asset)
  ml_client.jobs.create_or_update(pipeline_job)

MLOps with Azure DevOps / GitHub Actions:
├── Code change → CI pipeline runs tests
├── Training pipeline runs → New model trained
├── Compare metrics → If better, register new model
├── CD pipeline → Deploy to staging endpoint
├── Approval → Deploy to production
└── Monitor → Detect data drift → Retrain

Model Registry:
├── ML Studio → Models
├── Version models (v1, v2, v3)
├── Track which experiment produced each model
├── Tags and metadata for organization
└── Deploy any version to endpoint
```

---

## Part 7: Terraform & az CLI Reference

### Terraform

```hcl
resource "azurerm_machine_learning_workspace" "main" {
  name                    = "mlw-mycompany"
  resource_group_name     = azurerm_resource_group.main.name
  location                = azurerm_resource_group.main.location
  application_insights_id = azurerm_application_insights.main.id
  key_vault_id            = azurerm_key_vault.main.id
  storage_account_id      = azurerm_storage_account.main.id

  identity {
    type = "SystemAssigned"
  }
}

resource "azurerm_machine_learning_compute_cluster" "cluster" {
  name                          = "cpu-cluster"
  machine_learning_workspace_id = azurerm_machine_learning_workspace.main.id
  vm_size                       = "Standard_DS3_v2"
  vm_priority                   = "Dedicated"

  scale_settings {
    min_node_count                       = 0
    max_node_count                       = 4
    scale_down_nodes_after_idle_duration  = "PT120S"
  }
}
```

### Bicep

```bicep
// Azure ML Workspace
resource mlWorkspace 'Microsoft.MachineLearningServices/workspaces@2023-10-01' = {
  name: 'mlw-mycompany-prod'
  location: resourceGroup().location
  identity: { type: 'SystemAssigned' }
  properties: {
    storageAccount: storageAccount.id
    keyVault: keyVault.id
    applicationInsights: appInsights.id
    containerRegistry: acr.id
  }
}

// Compute Cluster
resource computeCluster 'Microsoft.MachineLearningServices/workspaces/computes@2023-10-01' = {
  parent: mlWorkspace
  name: 'gpu-cluster'
  location: resourceGroup().location
  properties: {
    computeType: 'AmlCompute'
    properties: {
      vmSize: 'Standard_NC6s_v3'
      vmPriority: 'Dedicated'
      scaleSettings: {
        minNodeCount: 0
        maxNodeCount: 4
        nodeIdleTimeBeforeScaleDown: 'PT120S'
      }
    }
  }
}
```
az extension add --name ml

# Create workspace
az ml workspace create \
  --name mlw-mycompany \
  --resource-group rg-ml \
  --location centralindia

# Create compute cluster
az ml compute create \
  --name cpu-cluster \
  --type AmlCompute \
  --size Standard_DS3_v2 \
  --min-instances 0 --max-instances 4 \
  --workspace-name mlw-mycompany \
  --resource-group rg-ml

# Submit training job
az ml job create --file job.yml --workspace-name mlw-mycompany --resource-group rg-ml

# List models
az ml model list --workspace-name mlw-mycompany --resource-group rg-ml -o table

# Deploy model
az ml online-endpoint create --name house-price-predictor \
  --workspace-name mlw-mycompany --resource-group rg-ml

# Delete workspace
az ml workspace delete --name mlw-mycompany --resource-group rg-ml --yes
```

---

## Real-World Patterns

### Pattern 1: MLOps Pipeline

```
┌─────────────────────────────────────────────────┐
│  ML Model Lifecycle                             │
├─────────────────────────────────────────────────┤
│                                                 │
│  Data Prep ─→ Train ─→ Evaluate ─→ Register     │
│  (Dataset)   (Compute   (Metrics   (Model      │
│               Cluster)   compare)   Registry)  │
│                                      │          │
│                                      ▼          │
│                              Deploy to Endpoint │
│                              (Managed Online)   │
│                                      │          │
│                                      ▼          │
│                              Monitor (drift,    │
│                              accuracy, latency) │
│                                      │          │
│                              Retrain if needed   │
│                                                 │
│  Pipeline triggered by: schedule / data change  │
│  A/B testing: blue/green endpoint deployments   │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Azure ML = End-to-end ML platform (train, deploy, manage)
Studio: https://ml.azure.com

Compute: Instance (dev) | Cluster (training) | Serverless
Training: Notebooks (code) | AutoML (no-code) | Designer (drag-drop)
Endpoints: Online (real-time) | Batch (large-scale)

Connected resources: Storage + Key Vault + App Insights + ACR

MLOps: CI/CD + Model Registry + Pipelines + Monitoring
Data: Datastores (connections) → Data Assets (pointers to data)

AutoML: Give it data + target column → Best model automatically!
Blue/Green: Deploy v2 alongside v1, shift traffic gradually

⚡ Compute cluster: Set min nodes to 0 (scale to zero!)
⚡ Compute instance: Set auto-shutdown schedule!
```

---

## What's Next?

Next chapter: [Chapter 61: Azure AI Services](61-ai-services.md) — Pre-built AI APIs: OpenAI, Vision, Speech, Language, and more.
