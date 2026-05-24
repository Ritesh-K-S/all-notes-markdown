# Chapter 55: Amazon SageMaker

---

## Table of Contents

- [Overview](#overview)
- [Part 1: SageMaker Fundamentals](#part-1-sagemaker-fundamentals)
- [Part 2: SageMaker Studio & Notebooks (Full Portal Walkthrough)](#part-2-sagemaker-studio--notebooks-full-portal-walkthrough)
- [Part 3: Training & Deploying Models](#part-3-training--deploying-models)
- [Part 4: SageMaker Features & MLOps](#part-4-sagemaker-features--mlops)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Machine Learning? Why SageMaker?

**Machine Learning (ML)** is teaching computers to learn from data and make predictions without being explicitly programmed. For example:
- 🛒 Predicting which customers will cancel their subscription (churn prediction)
- 📷 Automatically tagging photos ("this image contains a dog")
- 📬 Detecting spam emails
- 💰 Recommending products ("customers who bought X also bought Y")

Building ML models typically requires:
1. Collecting and cleaning data
2. Choosing and training a model
3. Evaluating if the model is good enough
4. Deploying the model so your app can use it
5. Monitoring and retraining over time

**SageMaker handles the entire lifecycle** — from data preparation to model deployment. Think of it as a **complete ML workshop** where you have all the tools in one place.

**When to use SageMaker vs pre-built AI services (Chapter 56):**
- **SageMaker**: You have custom data and need a custom model (e.g., predicting YOUR customers' behavior)
- **Pre-built AI services** (Rekognition, Comprehend): Standard ML tasks that work out of the box (e.g., detecting faces in images)

Amazon SageMaker is a fully managed ML platform to build, train, and deploy machine learning models at scale. It covers the entire ML lifecycle from data preparation to production inference.

```
What you'll learn:
├── SageMaker fundamentals (ML lifecycle)
├── SageMaker Studio & notebooks (portal walkthrough)
├── Training models (built-in algorithms, custom training)
├── Deploying models (endpoints, serverless, batch)
├── MLOps (Pipelines, Model Registry, Feature Store)
└── Terraform & CLI examples
```

---

## Part 1: SageMaker Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           SAGEMAKER ML LIFECYCLE                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Prepare → Build → Train → Tune → Deploy → Monitor                │
│                                                                       │
│ 1. Prepare Data                                                      │
│    ├── SageMaker Data Wrangler (visual data prep)               │
│    ├── SageMaker Processing (run preprocessing scripts)        │
│    ├── SageMaker Feature Store (curated features)              │
│    └── Ground Truth (data labeling)                              │
│                                                                       │
│ 2. Build Model                                                       │
│    ├── SageMaker Studio (IDE for ML)                             │
│    ├── Built-in algorithms (XGBoost, Linear Learner, etc.)     │
│    ├── Pre-built containers (TensorFlow, PyTorch, etc.)        │
│    ├── Custom containers (bring your own)                       │
│    └── SageMaker JumpStart (pre-trained models, foundation)    │
│                                                                       │
│ 3. Train Model                                                       │
│    ├── Managed training infrastructure (auto-provision/stop)   │
│    ├── Distributed training (data parallel, model parallel)    │
│    ├── Spot training (up to 90% cost savings)                  │
│    └── SageMaker Experiments (track experiments)                │
│                                                                       │
│ 4. Tune Model                                                        │
│    └── Hyperparameter Tuning (automatic optimization)          │
│                                                                       │
│ 5. Deploy Model                                                      │
│    ├── Real-time endpoints (persistent, low latency)           │
│    ├── Serverless inference (scale to zero)                    │
│    ├── Batch transform (process large datasets)                │
│    └── Async inference (queue-based, large payloads)          │
│                                                                       │
│ 6. Monitor                                                           │
│    ├── Model Monitor (data drift, quality monitoring)          │
│    └── SageMaker Clarify (bias detection, explainability)     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: SageMaker Studio & Notebooks (Full Portal Walkthrough)

```
Console → SageMaker → Domains → Create domain

┌─────────────────────────────────────────────────────────────────┐
│           CREATE SAGEMAKER DOMAIN                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Setup:                                                          │
│ ● Quick setup (⚡ fastest, recommended for getting started)   │
│ ○ Standard setup (customize VPC, IAM, storage)               │
│                                                                   │
│ Domain name: [ml-workspace]                                   │
│                                                                   │
│ User profile:                                                  │
│ Name: [data-scientist-1]                                      │
│ Execution role: ● Create a new role                          │
│ → S3 bucket access:                                          │
│   ● Any S3 bucket                                            │
│   ○ Specific S3 buckets: [ml-data-bucket]                   │
│   ○ None                                                      │
│                                                                   │
│ VPC: [ml-vpc ▼] (Standard setup only)                        │
│ Subnets: [private subnets ▼]                                 │
│                                                                   │
│                    [Submit]                                    │
└─────────────────────────────────────────────────────────────────┘

SageMaker Studio IDE:

┌─────────────────────────────────────────────────────────────────┐
│           SAGEMAKER STUDIO                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Left sidebar:                                                  │
│ ├── 🏠 Home (launcher)                                       │
│ ├── 📁 File browser                                          │
│ ├── 🔧 SageMaker resources                                  │
│ │   ├── Experiments and trials                               │
│ │   ├── Model registry                                       │
│ │   ├── Endpoints                                            │
│ │   ├── Feature Store                                        │
│ │   └── Pipelines                                            │
│ └── 🚀 JumpStart (pre-trained models)                       │
│                                                                   │
│ Create notebook:                                               │
│ File → New → Notebook                                        │
│                                                                   │
│ Select kernel:                                                 │
│ Image: [SageMaker Distribution v1 ▼]                        │
│ → Includes: PyTorch, TensorFlow, Pandas, Scikit-learn       │
│ Kernel: [Python 3 ▼]                                        │
│ Instance type: [ml.t3.medium ▼]                              │
│ ├── ml.t3.medium (2 vCPU, 4 GB) → exploration              │
│ ├── ml.m5.xlarge (4 vCPU, 16 GB) → standard                │
│ ├── ml.g4dn.xlarge (GPU) → deep learning                   │
│ └── ml.p3.2xlarge (V100 GPU) → heavy training              │
│                                                                   │
│ ⚡ Use smallest instance for exploration                      │
│ ⚡ Scale up only for training/processing                     │
│ ⚠️ Stop instances when not in use (you pay per hour)        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           SAGEMAKER JUMPSTART                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Pre-trained models & solutions:                                │
│ ├── Foundation Models                                         │
│ │   ├── Meta Llama 3 (text generation)                      │
│ │   ├── Stability AI SDXL (image generation)               │
│ │   └── Hugging Face models                                 │
│ ├── Vision (image classification, object detection)          │
│ ├── NLP (text classification, NER, translation)             │
│ ├── Tabular (classification, regression)                    │
│ └── Solution templates (fraud detection, demand forecast)  │
│                                                                   │
│ Deploy: [Deploy] → Creates endpoint in minutes              │
│ Fine-tune: [Fine-tune] → Train on your data                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Training & Deploying Models

```
┌─────────────────────────────────────────────────────────────────────┐
│           TRAINING                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → SageMaker → Training → Training jobs → Create           │
│                                                                       │
│ Job name: [xgboost-customer-churn-v1]                              │
│                                                                       │
│ Algorithm source:                                                    │
│ ● Built-in algorithm                                                │
│   → XGBoost, Linear Learner, K-Means, Image Classification, etc.│
│ ○ Your own algorithm in ECR container                              │
│ ○ Pre-built framework container (TensorFlow, PyTorch)             │
│                                                                       │
│ Instance type: [ml.m5.xlarge ▼]                                    │
│ Instance count: [1]                                                  │
│ ☑ Use managed spot training                                        │
│ → Up to 90% cost savings                                          │
│ → ⚡ Always use for training (with checkpointing)                 │
│ Max wait time: [3600] seconds                                      │
│ Max run time: [3600] seconds                                       │
│                                                                       │
│ Input data:                                                          │
│ Channel name: [train]                                               │
│ S3 data location: [s3://ml-data/train/]                            │
│ Content type: [text/csv]                                            │
│ Input mode: ● File ○ Pipe ○ FastFile                              │
│                                                                       │
│ Channel name: [validation]                                          │
│ S3 data location: [s3://ml-data/validation/]                       │
│                                                                       │
│ Hyperparameters:                                                     │
│ max_depth: [5]                                                       │
│ eta: [0.2]                                                           │
│ num_round: [100]                                                     │
│ objective: [binary:logistic]                                        │
│                                                                       │
│ Output data:                                                         │
│ S3 output path: [s3://ml-models/output/]                           │
│ → Model artifacts saved here (model.tar.gz)                      │
│                                                                       │
│ IAM role: [SageMakerExecutionRole ▼]                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           DEPLOYING MODELS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → SageMaker → Inference → Endpoints → Create              │
│                                                                       │
│ Deployment options:                                                  │
│                                                                       │
│ 1. Real-time Endpoint (persistent):                                 │
│    Instance type: [ml.m5.xlarge ▼]                                │
│    Instance count: [2] (multi-AZ for HA)                          │
│    → Low latency (~100ms), always running                        │
│    → ⚡ Use for production APIs                                   │
│    → ⚠️ Pay per hour even when idle                              │
│                                                                       │
│ 2. Serverless Inference:                                             │
│    Memory: [2048] MB                                                │
│    Max concurrency: [5]                                              │
│    → Auto-scales to zero                                           │
│    → Cold start ~1-2 seconds                                      │
│    → ⚡ Use for infrequent/unpredictable traffic                 │
│                                                                       │
│ 3. Batch Transform:                                                  │
│    S3 input: [s3://batch-input/]                                   │
│    S3 output: [s3://batch-output/]                                 │
│    Instance type: [ml.m5.4xlarge]                                  │
│    → Process large datasets offline                               │
│    → ⚡ Use for nightly scoring/predictions                      │
│                                                                       │
│ 4. Async Inference:                                                  │
│    → Queue-based, handles large payloads                         │
│    → Auto-scales (including to 0)                                 │
│    → Results delivered to S3 + SNS notification                  │
│    → ⚡ Use for long-running inference (video, images)           │
│                                                                       │
│ Auto-scaling:                                                        │
│ Metric: InvocationsPerInstance                                      │
│ Target value: [100]                                                  │
│ Min instances: [1]  Max instances: [10]                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: SageMaker Features & MLOps

```
┌─────────────────────────────────────────────────────────────────────┐
│           MLOPS FEATURES                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ SageMaker Pipelines (ML CI/CD):                                    │
│ ├── Define ML workflows as DAGs                                  │
│ ├── Steps: Processing, Training, Tuning, Transform, Register   │
│ ├── Condition steps (if model accuracy > threshold)            │
│ ├── Versioned, repeatable, auditable                            │
│ └── Integration with CI/CD (CodePipeline, GitHub Actions)     │
│                                                                       │
│ Model Registry:                                                      │
│ ├── Version and catalog trained models                          │
│ ├── Approval workflow (Pending → Approved → Deployed)         │
│ ├── Model lineage (track data + code + artifacts)             │
│ └── Deploy approved models to endpoints                        │
│                                                                       │
│ Feature Store:                                                       │
│ ├── Centralized feature repository                               │
│ ├── Online store (low-latency, for real-time inference)        │
│ ├── Offline store (S3, for training)                            │
│ ├── Feature groups with schema definitions                     │
│ └── ⚡ Consistent features between training and inference     │
│                                                                       │
│ Model Monitor:                                                       │
│ ├── Monitor deployed models for quality degradation            │
│ ├── Data quality monitoring (schema, statistics drift)        │
│ ├── Model quality monitoring (accuracy, precision, recall)    │
│ ├── Bias drift monitoring                                       │
│ └── Schedule: Hourly or daily monitoring jobs                  │
│                                                                       │
│ SageMaker Clarify:                                                  │
│ ├── Bias detection (pre-training and post-training)           │
│ ├── Feature importance (SHAP values)                            │
│ └── Model explainability reports                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
# SageMaker domain
resource "aws_sagemaker_domain" "ml" {
  domain_name = "ml-workspace"
  auth_mode   = "IAM"
  vpc_id      = aws_vpc.ml.id
  subnet_ids  = aws_subnet.private[*].id

  default_user_settings {
    execution_role = aws_iam_role.sagemaker.arn
  }
}

# SageMaker model
resource "aws_sagemaker_model" "xgboost" {
  name               = "customer-churn-model"
  execution_role_arn = aws_iam_role.sagemaker.arn

  primary_container {
    image          = "683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.7-1"
    model_data_url = "s3://ml-models/output/model.tar.gz"
  }
}

# Real-time endpoint
resource "aws_sagemaker_endpoint_configuration" "prod" {
  name = "churn-endpoint-config"

  production_variants {
    variant_name           = "primary"
    model_name             = aws_sagemaker_model.xgboost.name
    instance_type          = "ml.m5.xlarge"
    initial_instance_count = 2
  }
}

resource "aws_sagemaker_endpoint" "prod" {
  name                 = "churn-prediction-endpoint"
  endpoint_config_name = aws_sagemaker_endpoint_configuration.prod.name
}
```

```bash
# Create training job
aws sagemaker create-training-job \
  --training-job-name xgboost-churn-v1 \
  --algorithm-specification TrainingImage=683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.7-1,TrainingInputMode=File \
  --role-arn arn:aws:iam::123456:role/SageMakerRole \
  --input-data-config '[{"ChannelName":"train","DataSource":{"S3DataSource":{"S3DataType":"S3Prefix","S3Uri":"s3://ml-data/train/"}}}]' \
  --output-data-config S3OutputPath=s3://ml-models/output/ \
  --resource-config InstanceType=ml.m5.xlarge,InstanceCount=1,VolumeSizeInGB=10 \
  --stopping-condition MaxRuntimeInSeconds=3600

# Invoke endpoint
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name churn-prediction-endpoint \
  --content-type text/csv \
  --body "1,35,1,45000,2,1,0,0,1" \
  output.json

# List endpoints
aws sagemaker list-endpoints --sort-by CreationTime --sort-order Descending
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Real-Time Prediction API                                 │
│ Client → API Gateway → Lambda → SageMaker Endpoint                │
│ ├── SageMaker endpoint with auto-scaling                        │
│ ├── Multi-model endpoint (multiple models on one instance)    │
│ ├── Model Monitor for drift detection                           │
│ └── A/B testing with production variants                       │
│                                                                       │
│ Pattern 2: ML Pipeline (MLOps)                                      │
│ S3 Data → SageMaker Pipeline:                                      │
│ 1. Processing (data prep)                                         │
│ 2. Training (model training)                                      │
│ 3. Evaluation (metrics check)                                     │
│ 4. Condition (accuracy > 0.85?)                                  │
│ 5. Register (Model Registry)                                     │
│ 6. Deploy (endpoint update)                                      │
│ → Triggered by: Data arrival, schedule, or code commit         │
│                                                                       │
│ Pattern 3: Batch Scoring                                             │
│ S3 (input data) → Batch Transform → S3 (predictions)            │
│ ├── Nightly scoring of customer base                            │
│ ├── No persistent endpoint needed                                │
│ ├── Cost-effective for large datasets                           │
│ └── Results loaded into data warehouse                          │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Use Spot training for cost savings (with checkpointing)     │
│ 2. Serverless inference for unpredictable traffic               │
│ 3. Model Registry for versioning and approval workflow         │
│ 4. Feature Store for consistent features                        │
│ 5. Monitor models for data and prediction drift                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
SageMaker Quick Reference:
├── Studio: ML IDE (notebooks, experiments, model registry)
├── JumpStart: Pre-trained models + foundation models
├── Training: Managed infrastructure, Spot training (90% off)
├── Built-in algorithms: XGBoost, Linear Learner, K-Means, etc.
├── Frameworks: TensorFlow, PyTorch, MXNet, Scikit-learn
├── Deployment:
│   ├── Real-time endpoint (low latency, always on)
│   ├── Serverless inference (scale to zero)
│   ├── Batch transform (offline scoring)
│   └── Async inference (large payloads, queue-based)
├── MLOps:
│   ├── Pipelines (ML CI/CD workflows)
│   ├── Model Registry (versioning, approval)
│   ├── Feature Store (online + offline)
│   └── Model Monitor (drift detection)
├── ⚡ Use Spot training + checkpointing for cost savings
├── ⚡ JumpStart for quick model deployment
└── ⚡ Serverless inference for infrequent predictions
```

---

## What's Next?

In **Chapter 56: AI/ML Services**, we'll cover pre-built AI services like Rekognition, Comprehend, Translate, Polly, Textract, Bedrock, and more for adding intelligence without building models.
