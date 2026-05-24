# Chapter 58 — Vertex AI

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Vertex AI Fundamentals](#part-1--vertex-ai-fundamentals)
- [Part 2: Datasets](#part-2--datasets)
- [Part 3: Feature Store](#part-3--feature-store)
- [Part 4: Training — Custom Jobs](#part-4--training--custom-jobs)
- [Part 5: Training — AutoML](#part-5--training--automl)
- [Part 6: Models & Model Registry](#part-6--models--model-registry)
- [Part 7: Endpoints & Prediction](#part-7--endpoints--prediction)
- [Part 8: Pipelines (Kubeflow)](#part-8--pipelines-kubeflow)
- [Part 9: Experiments & Metadata](#part-9--experiments--metadata)
- [Part 10: Model Monitoring](#part-10--model-monitoring)
- [Part 11: Vertex AI Workbench](#part-11--vertex-ai-workbench)
- [Part 12: Vector Search](#part-12--vector-search)
- [Part 13: Generative AI (Gemini)](#part-13--generative-ai-gemini)
- [Part 14: Console Walkthrough — Vertex AI Setup & Training](#part-14-console-walkthrough--vertex-ai-setup--training)
- [Part 15: Terraform & gcloud CLI Reference](#part-15--terraform--gcloud-cli-reference)
- [Part 16: Real-World Patterns](#part-16--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Vertex AI is Google Cloud's unified machine learning platform that brings together all ML services under one roof — from data preparation and feature engineering, through training (AutoML or custom), to model deployment, monitoring, and management. It supports the full MLOps lifecycle and integrates Google's latest generative AI models (Gemini) for building AI-powered applications.

---

## Part 1 — Vertex AI Fundamentals

### Platform Overview

```
┌────────────────────────────────────────────────────────────────────┐
│         VERTEX AI PLATFORM                                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    VERTEX AI                               │     │
│  │                                                            │     │
│  │  DATA                    TRAIN                             │     │
│  │  ┌──────────────┐       ┌──────────────┐                 │     │
│  │  │ Datasets     │       │ Custom       │                 │     │
│  │  │ Feature Store│       │ AutoML       │                 │     │
│  │  │ Labeling     │       │ Pre-trained  │                 │     │
│  │  └──────────────┘       └──────────────┘                 │     │
│  │                                                            │     │
│  │  DEPLOY                  MANAGE                           │     │
│  │  ┌──────────────┐       ┌──────────────┐                 │     │
│  │  │ Endpoints    │       │ Model Registry│                │     │
│  │  │ Batch Predict│       │ Experiments   │                │     │
│  │  │ Online Predict│      │ Pipelines     │                │     │
│  │  └──────────────┘       │ Monitoring    │                │     │
│  │                          │ Metadata     │                │     │
│  │                          └──────────────┘                 │     │
│  │                                                            │     │
│  │  GENERATIVE AI                                            │     │
│  │  ┌──────────────────────────────────────────────┐        │     │
│  │  │ Gemini (text, vision, code)                   │        │     │
│  │  │ Model Garden (100+ foundation models)         │        │     │
│  │  │ Tuning (fine-tuning, RLHF, distillation)     │        │     │
│  │  │ Grounding (RAG, Search, custom)               │        │     │
│  │  │ Agent Builder                                  │        │     │
│  │  │ Vector Search                                  │        │     │
│  │  └──────────────────────────────────────────────┘        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Vertex AI | AWS SageMaker | Azure ML |
|---------|-------------|--------------|---------|
| Unified ML platform | Vertex AI | SageMaker | Azure ML Studio |
| AutoML | Vertex AutoML | SageMaker Autopilot | AutoML |
| Custom training | Custom jobs (any framework) | Training jobs | Compute instances |
| Model serving | Endpoints (online/batch) | Endpoints (real-time/batch) | Managed endpoints |
| MLOps pipelines | Kubeflow Pipelines | SageMaker Pipelines | Azure ML Pipelines |
| Feature store | Vertex Feature Store | SageMaker Feature Store | Feature Store (preview) |
| Experiment tracking | Vertex Experiments | SageMaker Experiments | MLflow |
| Gen AI | Gemini, Model Garden | Bedrock | Azure OpenAI |
| Notebooks | Workbench | Studio notebooks | Compute instances |
| Vector DB | Vector Search | OpenSearch | AI Search |

### Pricing (Key Components)

| Component | Cost |
|-----------|------|
| Training (custom) | Per compute hour (e.g., n1-standard-4: ~$0.19/hr) |
| Training (GPU) | T4: ~$0.35/hr, A100: ~$3.67/hr |
| AutoML training | $3.15/node-hour (image), varies by type |
| Prediction (online) | Per node-hour (e.g., n1-standard-2: ~$0.12/hr) |
| Prediction (batch) | Per node-hour (same as training rates) |
| Gemini 1.5 Pro | $1.25/M input tokens, $5.00/M output tokens |
| Gemini 1.5 Flash | $0.075/M input tokens, $0.30/M output tokens |

---

## Part 2 — Datasets

### Managing Datasets

```python
from google.cloud import aiplatform

aiplatform.init(project='my-project', location='us-central1')

# Create tabular dataset from BigQuery
dataset = aiplatform.TabularDataset.create(
    display_name='customer-churn',
    bq_source='bq://my-project.ml_data.customers',
)

# Create tabular dataset from GCS (CSV)
dataset = aiplatform.TabularDataset.create(
    display_name='sales-data',
    gcs_source='gs://my-bucket/data/sales.csv',
)

# Create image dataset
dataset = aiplatform.ImageDataset.create(
    display_name='product-images',
    gcs_source='gs://my-bucket/images/import.jsonl',
    import_schema_uri=aiplatform.schema.dataset.ioformat.image
        .single_label_classification,
)

# Create text dataset
dataset = aiplatform.TextDataset.create(
    display_name='support-tickets',
    gcs_source='gs://my-bucket/text/tickets.jsonl',
    import_schema_uri=aiplatform.schema.dataset.ioformat.text
        .single_label_classification,
)
```

---

## Part 3 — Feature Store

### Managed Feature Storage

```
┌────────────────────────────────────────────────────────────────────┐
│         FEATURE STORE                                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Central repository for ML features:                               │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Feature Store                                             │     │
│  │  └── Feature Group: customer_features                     │     │
│  │       ├── Feature: avg_order_value  (DOUBLE)              │     │
│  │       ├── Feature: total_orders     (INT64)               │     │
│  │       ├── Feature: days_since_last  (INT64)               │     │
│  │       └── Feature: lifetime_value   (DOUBLE)              │     │
│  │                                                            │     │
│  │  Benefits:                                                │     │
│  │  • Share features across models                          │     │
│  │  • Point-in-time correctness (avoid data leakage)       │     │
│  │  • Online serving (low latency) + batch serving          │     │
│  │  • Feature freshness monitoring                          │     │
│  │  • BigQuery as offline store                             │     │
│  │  • Bigtable as online store                              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```python
# Create Feature Store (2.0 — BigQuery-based)
from google.cloud import aiplatform

# Feature Group backed by BigQuery table
feature_group = aiplatform.FeatureGroup.create(
    name='customer_features',
    source=aiplatform.FeatureGroup.BigQuery(
        uri='bq://my-project.ml_features.customer_features',
        entity_id_columns=['customer_id'],
    ),
)

# Create Feature View for online serving
feature_view = aiplatform.FeatureOnlineStore.create_feature_view(
    name='customer_serving',
    source=feature_group,
    sync_config=aiplatform.FeatureView.SyncConfig(cron='0 * * * *'),  # hourly
)

# Online serving — fetch features for prediction
response = feature_view.read(key=['customer-123'])
```

---

## Part 4 — Training — Custom Jobs

### Custom Training with Any Framework

```python
from google.cloud import aiplatform

# Custom training job (Python script on GCS)
job = aiplatform.CustomTrainingJob(
    display_name='churn-model-training',
    script_path='train.py',
    container_uri='us-docker.pkg.dev/vertex-ai/training/tf-gpu.2-14.py310:latest',
    requirements=['pandas', 'scikit-learn'],
    model_serving_container_image_uri=(
        'us-docker.pkg.dev/vertex-ai/prediction/tf2-gpu.2-14:latest'
    ),
)

# Run training
model = job.run(
    dataset=dataset,
    model_display_name='churn-model-v1',
    machine_type='n1-standard-8',
    accelerator_type='NVIDIA_TESLA_T4',
    accelerator_count=1,
    replica_count=1,
    args=['--epochs=50', '--batch-size=128'],
)

# Custom container training (bring your own Docker)
job = aiplatform.CustomContainerTrainingJob(
    display_name='custom-training',
    container_uri='gcr.io/my-project/training:latest',
    model_serving_container_image_uri='gcr.io/my-project/serving:latest',
)

model = job.run(
    model_display_name='my-model',
    machine_type='a2-highgpu-1g',       # A100 GPU
    replica_count=1,
)

# Distributed training (multi-worker)
job = aiplatform.CustomJob(
    display_name='distributed-training',
    worker_pool_specs=[
        {   # Chief worker
            'machine_spec': {'machine_type': 'n1-standard-16', 'accelerator_type': 'NVIDIA_TESLA_V100', 'accelerator_count': 2},
            'replica_count': 1,
            'container_spec': {'image_uri': 'gcr.io/my-project/train:latest'},
        },
        {   # Additional workers
            'machine_spec': {'machine_type': 'n1-standard-16', 'accelerator_type': 'NVIDIA_TESLA_V100', 'accelerator_count': 2},
            'replica_count': 3,
            'container_spec': {'image_uri': 'gcr.io/my-project/train:latest'},
        },
    ],
)
job.run()
```

### Pre-Built Training Containers

| Framework | CPU Container | GPU Container |
|-----------|-------------|-------------|
| TensorFlow 2.14 | `tf-cpu.2-14.py310` | `tf-gpu.2-14.py310` |
| PyTorch 2.1 | `pytorch-cpu.2-1.py310` | `pytorch-gpu.2-1.py310` |
| Scikit-learn 1.3 | `sklearn-cpu.1-3.py310` | — |
| XGBoost 1.7 | `xgboost-cpu.1-7.py310` | — |

---

## Part 5 — Training — AutoML

### AutoML (No-Code ML)

```python
# AutoML Tabular (classification)
job = aiplatform.AutoMLTabularTrainingJob(
    display_name='churn-automl',
    optimization_prediction_type='classification',
    optimization_objective='maximize-au-roc',
)

model = job.run(
    dataset=dataset,
    target_column='churned',
    training_fraction_split=0.8,
    validation_fraction_split=0.1,
    test_fraction_split=0.1,
    budget_milli_node_hours=1000,       # 1 node-hour
    model_display_name='churn-automl-v1',
)

# AutoML Image Classification
job = aiplatform.AutoMLImageTrainingJob(
    display_name='product-classifier',
    prediction_type='classification',
    multi_label=False,
)

model = job.run(
    dataset=image_dataset,
    budget_milli_node_hours=8000,
    model_display_name='product-classifier-v1',
)

# AutoML Text Classification
job = aiplatform.AutoMLTextTrainingJob(
    display_name='ticket-classifier',
    prediction_type='classification',
    multi_label=True,
)

# AutoML Forecasting
job = aiplatform.AutoMLForecastingTrainingJob(
    display_name='sales-forecast',
    optimization_objective='minimize-rmse',
)

model = job.run(
    dataset=dataset,
    target_column='revenue',
    time_column='date',
    time_series_identifier_column='store_id',
    forecast_horizon=30,
    model_display_name='sales-forecast-v1',
)
```

### AutoML Capabilities

| Type | Tasks | Input |
|------|-------|-------|
| Tabular | Classification, regression, forecasting | CSV, BigQuery |
| Image | Classification, object detection, segmentation | Images + labels |
| Text | Classification, entity extraction, sentiment | Text + labels |
| Video | Classification, object tracking, action recognition | Videos + labels |

---

## Part 6 — Models & Model Registry

### Model Management

```python
# Upload a pre-trained model
model = aiplatform.Model.upload(
    display_name='my-custom-model',
    artifact_uri='gs://my-bucket/models/v1/',
    serving_container_image_uri=(
        'us-docker.pkg.dev/vertex-ai/prediction/tf2-cpu.2-14:latest'
    ),
    labels={'team': 'ml', 'version': '1'},
)

# Model versioning (upload as new version of existing model)
model = aiplatform.Model.upload(
    display_name='my-custom-model',
    parent_model=existing_model.resource_name,
    artifact_uri='gs://my-bucket/models/v2/',
    serving_container_image_uri=(
        'us-docker.pkg.dev/vertex-ai/prediction/tf2-cpu.2-14:latest'
    ),
    is_default_version=True,
)

# List models
models = aiplatform.Model.list()
for m in models:
    print(f"{m.display_name}: {m.resource_name}")

# Model evaluation
model = aiplatform.Model(model.resource_name)
evaluations = model.list_model_evaluations()
for e in evaluations:
    print(f"AUC: {e.metrics.get('auRoc', 'N/A')}")
```

---

## Part 7 — Endpoints & Prediction

### Online & Batch Prediction

```
┌────────────────────────────────────────────────────────────────────┐
│         PREDICTION TYPES                                            │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Online Prediction (real-time):                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Client ──REST/gRPC──► Endpoint ──► Model ──► Response   │     │
│  │  Latency: milliseconds                                    │     │
│  │  Use for: user-facing features, real-time decisions      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Batch Prediction (large-scale):                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  GCS/BQ input ──► Batch Job ──► Model ──► GCS/BQ output │     │
│  │  Throughput: millions of predictions                      │     │
│  │  Use for: scoring entire datasets, nightly predictions   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```python
# Deploy model to endpoint
endpoint = aiplatform.Endpoint.create(display_name='churn-endpoint')

endpoint.deploy(
    model=model,
    deployed_model_display_name='churn-v1',
    machine_type='n1-standard-4',
    min_replica_count=1,
    max_replica_count=5,
    traffic_percentage=100,
    accelerator_type='NVIDIA_TESLA_T4',     # optional GPU
    accelerator_count=1,
)

# Online prediction
response = endpoint.predict(
    instances=[
        {'tenure_months': 12, 'monthly_charges': 79.99, 'contract_type': 'month'},
        {'tenure_months': 48, 'monthly_charges': 29.99, 'contract_type': 'two_year'},
    ]
)
print(response.predictions)

# Traffic splitting (A/B testing / canary)
endpoint.deploy(
    model=model_v2,
    deployed_model_display_name='churn-v2',
    machine_type='n1-standard-4',
    min_replica_count=1,
    traffic_percentage=10,              # 10% to new model
)

# Batch prediction
batch_job = model.batch_predict(
    job_display_name='monthly-scoring',
    gcs_source='gs://my-bucket/input/customers.jsonl',
    gcs_destination_prefix='gs://my-bucket/predictions/',
    machine_type='n1-standard-4',
    max_replica_count=10,
    starting_replica_count=5,
)
batch_job.wait()
```

---

## Part 8 — Pipelines (Kubeflow)

### ML Pipelines

```python
from kfp.v2 import dsl, compiler
from google_cloud_pipeline_components.v1.custom_job import CustomTrainingJobOp
from google_cloud_pipeline_components.v1.endpoint import (
    EndpointCreateOp, ModelDeployOp
)

@dsl.pipeline(
    name='churn-prediction-pipeline',
    description='End-to-end churn prediction pipeline',
)
def churn_pipeline(
    project: str = 'my-project',
    location: str = 'us-central1',
):
    # Step 1: Data preparation
    prep_op = dsl.ContainerOp(
        name='prepare-data',
        image='gcr.io/my-project/data-prep:latest',
        arguments=['--date', '{{ $.pipeline_job.create_time }}'],
        file_outputs={'dataset': '/output/dataset_uri.txt'},
    )

    # Step 2: Train model
    train_op = CustomTrainingJobOp(
        display_name='train-churn',
        container_uri='gcr.io/my-project/train:latest',
        model_serving_container_image_uri=(
            'us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-3:latest'
        ),
        project=project,
        location=location,
    ).after(prep_op)

    # Step 3: Deploy model
    endpoint_op = EndpointCreateOp(
        display_name='churn-endpoint',
        project=project,
        location=location,
    )

    deploy_op = ModelDeployOp(
        model=train_op.outputs['model'],
        endpoint=endpoint_op.outputs['endpoint'],
        machine_type='n1-standard-4',
        min_replica_count=1,
        max_replica_count=3,
    )

# Compile pipeline
compiler.Compiler().compile(
    pipeline_func=churn_pipeline,
    package_path='pipeline.yaml',
)

# Run pipeline
from google.cloud import aiplatform
aiplatform.init(project='my-project', location='us-central1')

job = aiplatform.PipelineJob(
    display_name='churn-pipeline-run',
    template_path='pipeline.yaml',
    pipeline_root='gs://my-bucket/pipeline-root/',
)
job.run()
```

---

## Part 9 — Experiments & Metadata

### Tracking Experiments

```python
from google.cloud import aiplatform

aiplatform.init(
    project='my-project',
    location='us-central1',
    experiment='churn-experiment',
)

# Start a run
with aiplatform.start_run('run-001') as run:
    # Log parameters
    run.log_params({
        'learning_rate': 0.01,
        'epochs': 50,
        'batch_size': 128,
        'model_type': 'xgboost',
    })

    # Train model...
    model, metrics = train_model(lr=0.01, epochs=50)

    # Log metrics
    run.log_metrics({
        'accuracy': metrics['accuracy'],
        'auc_roc': metrics['auc_roc'],
        'f1_score': metrics['f1_score'],
        'loss': metrics['loss'],
    })

    # Log time-series metrics
    for epoch in range(50):
        run.log_time_series_metrics({
            'train_loss': train_losses[epoch],
            'val_loss': val_losses[epoch],
        }, step=epoch)

# Compare experiments
experiment_df = aiplatform.get_experiment_df('churn-experiment')
print(experiment_df[['run_name', 'param.learning_rate', 'metric.auc_roc']])
```

---

## Part 10 — Model Monitoring

### Drift & Skew Detection

```python
# Create model monitoring job
from google.cloud.aiplatform import model_monitoring

# Monitoring for deployed model
monitoring_job = aiplatform.ModelDeploymentMonitoringJob.create(
    display_name='churn-monitoring',
    endpoint=endpoint,
    logging_sampling_strategy=(
        model_monitoring.RandomSampleConfig(sample_rate=0.1)
    ),
    schedule_config=model_monitoring.ScheduleConfig(
        monitor_interval=3600       # check every hour
    ),
    alert_config=model_monitoring.EmailAlertConfig(
        user_emails=['ml-team@company.com']
    ),
    objective_configs={
        'churn-v1': model_monitoring.ObjectiveConfig(
            training_dataset=model_monitoring.TrainingDataset(
                dataset=dataset,
                target_field='churned',
            ),
            training_prediction_skew_detection_config=(
                model_monitoring.SkewDetectionConfig(
                    data_source='training',
                    skew_thresholds={'tenure_months': 0.3, 'monthly_charges': 0.3},
                )
            ),
            prediction_drift_detection_config=(
                model_monitoring.DriftDetectionConfig(
                    drift_thresholds={'tenure_months': 0.3, 'monthly_charges': 0.3},
                )
            ),
        )
    },
)
```

---

## Part 11 — Vertex AI Workbench

### Managed Notebooks

```bash
# Create managed notebook instance
gcloud workbench instances create ml-notebook \
    --location=us-central1-a \
    --machine-type=n1-standard-8 \
    --accelerator-type=NVIDIA_TESLA_T4 \
    --accelerator-core-count=1 \
    --install-gpu-driver \
    --disable-public-ip

# List instances
gcloud workbench instances list --location=us-central1-a

# Start/stop
gcloud workbench instances start ml-notebook --location=us-central1-a
gcloud workbench instances stop ml-notebook --location=us-central1-a
```

---

## Part 12 — Vector Search

### Similarity Search at Scale

```python
# Create Vector Search index
from google.cloud import aiplatform

# Prepare embeddings (JSONL format in GCS)
# Each line: {"id": "item1", "embedding": [0.1, 0.2, ...]}

index = aiplatform.MatchingEngineIndex.create_tree_ah_index(
    display_name='product-embeddings',
    contents_delta_uri='gs://my-bucket/embeddings/',
    dimensions=768,                     # embedding dimensions
    approximate_neighbors_count=150,
    distance_measure_type='DOT_PRODUCT_DISTANCE',
)

# Create index endpoint
index_endpoint = aiplatform.MatchingEngineIndexEndpoint.create(
    display_name='product-search-endpoint',
    public_endpoint_enabled=True,
)

# Deploy index to endpoint
index_endpoint.deploy_index(
    index=index,
    deployed_index_id='product-idx-v1',
    machine_type='e2-standard-16',
    min_replica_count=1,
    max_replica_count=5,
)

# Query nearest neighbors
response = index_endpoint.find_neighbors(
    deployed_index_id='product-idx-v1',
    queries=[[0.1, 0.2, 0.3, ...]],     # query embedding
    num_neighbors=10,
)
```

---

## Part 13 — Generative AI (Gemini)

### Using Gemini Models

```python
import vertexai
from vertexai.generative_models import GenerativeModel, Part

vertexai.init(project='my-project', location='us-central1')

# Gemini text generation
model = GenerativeModel('gemini-1.5-pro')
response = model.generate_content('Explain quantum computing in simple terms.')
print(response.text)

# Multi-turn conversation (chat)
chat = model.start_chat()
response = chat.send_message('What is machine learning?')
print(response.text)

response = chat.send_message('How does it differ from deep learning?')
print(response.text)

# Multimodal (text + image)
model = GenerativeModel('gemini-1.5-pro')
image = Part.from_uri('gs://my-bucket/product.jpg', mime_type='image/jpeg')
response = model.generate_content([
    'Describe this product and suggest a marketing tagline.',
    image,
])

# System instructions
model = GenerativeModel(
    'gemini-1.5-pro',
    system_instruction='You are a helpful customer support agent for an e-commerce company. Be concise and friendly.',
)

# Grounding with Google Search
from vertexai.generative_models import Tool, grounding
model = GenerativeModel('gemini-1.5-pro')
tool = Tool.from_google_search_retrieval(grounding.GoogleSearchRetrieval())
response = model.generate_content(
    'What are the latest GCP announcements from Google Cloud Next?',
    tools=[tool],
)

# Tuned model
from vertexai.tuning import sft
sft_job = sft.train(
    source_model='gemini-1.5-flash-002',
    train_dataset='gs://my-bucket/training_data.jsonl',
    tuned_model_display_name='custom-support-bot',
    epochs=3,
)
```

### Model Garden

```
┌────────────────────────────────────────────────────────────────────┐
│         MODEL GARDEN                                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  100+ foundation models available:                                 │
│                                                                      │
│  Google models:                                                     │
│  • Gemini 1.5 Pro / Flash (text, code, vision, audio)             │
│  • PaLM 2 (text, chat, code)                                     │
│  • Imagen (image generation)                                      │
│  • Chirp (speech-to-text)                                         │
│  • Codey (code generation)                                        │
│                                                                      │
│  Open-source models:                                               │
│  • Llama 3 (Meta)                                                  │
│  • Gemma (Google open-source)                                     │
│  • Mistral / Mixtral                                              │
│  • Stable Diffusion (Stability AI)                                │
│  • Claude (Anthropic — via partner)                               │
│                                                                      │
│  Deploy: one-click to Vertex AI endpoint                          │
│  Tune: fine-tune with your data                                   │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 14: Console Walkthrough — Vertex AI Setup & Training

### Creating a Dataset

```
Console → Vertex AI → Datasets → CREATE DATASET

┌─────────────────────────────────────────────────────────────────┐
│           CREATE DATASET                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Dataset name ──                                              │
│ Name: [customer-churn-data]                                     │
│                                                                   │
│ ── Data type and objective ──                                   │
│ Data type:                                                      │
│ ● Tabular                                                      │
│ ○ Image                                                        │
│ ○ Text                                                         │
│ ○ Video                                                        │
│                                                                   │
│ Objective (for Tabular):                                       │
│ ● Classification                                               │
│ ○ Regression                                                    │
│ ○ Forecasting                                                   │
│                                                                   │
│ ── Region ──                                                     │
│ Region: [us-central1 ▼]                                         │
│                                                                   │
│                             [CREATE]                            │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│           IMPORT DATA (after dataset creation)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Data source:                                                    │
│ ● Upload CSV files from your computer                          │
│ ○ Select CSV files from Cloud Storage                          │
│ ○ Select a table from BigQuery                                  │
│                                                                   │
│ GCS path: [gs://my-ml-data/churn-training.csv]                 │
│                                                                   │
│                           [CONTINUE]                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Training a Model (AutoML)

```
Console → Vertex AI → Training → CREATE

┌─────────────────────────────────────────────────────────────────┐
│           TRAIN NEW MODEL (Step 1: Training method)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Dataset: [customer-churn-data ▼]                               │
│                                                                   │
│ Training method:                                                │
│ ● AutoML                                                       │
│   ⚡ Google selects best model architecture automatically.      │
│   No coding required. Best for tabular/image classification.   │
│                                                                   │
│ ○ Custom training                                              │
│   ⚡ Bring your own training code (TensorFlow, PyTorch, etc.)  │
│   Full control over model architecture and hyperparameters.    │
│                                                                   │
│                              [CONTINUE]                        │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│           Step 2: Model details                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Model name: [churn-prediction-v1]                               │
│ Target column: [churned ▼]                                      │
│ ⚡ Select the column to predict (label).                         │
│                                                                   │
│ ── Feature selection ──                                         │
│ ☑ account_age                                                   │
│ ☑ monthly_charges                                               │
│ ☑ total_charges                                                 │
│ ☑ contract_type                                                 │
│ ☐ customer_id (exclude — not predictive)                       │
│                                                                   │
│ Data split:                                                     │
│ ● Automatic (80% train / 10% validation / 10% test)            │
│ ○ Manual (specify data split column)                           │
│                                                                   │
│                              [CONTINUE]                        │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│           Step 3: Training options                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Budget: [8] node hours maximum                                 │
│ ⚡ AutoML trains multiple models and selects the best.          │
│   More hours = potentially better model but higher cost.       │
│   Minimum 1 hour. Google recommends 1-72 hours.                │
│                                                                   │
│ ☑ Enable early stopping                                        │
│ ⚡ Stop training early if model isn't improving.                 │
│   Saves cost without sacrificing quality.                       │
│                                                                   │
│ Encryption: [● Google-managed | ○ CMEK]                        │
│                                                                   │
│                        [START TRAINING]                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Deploying a Model to an Endpoint

```
Console → Vertex AI → Model Registry → Click model → DEPLOY TO ENDPOINT

┌─────────────────────────────────────────────────────────────────┐
│           DEPLOY TO ENDPOINT                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Define your endpoint ──                                     │
│ ● Create new endpoint                                          │
│ ○ Use existing endpoint                                        │
│                                                                   │
│ Endpoint name: [churn-prediction-endpoint]                     │
│ Region: [us-central1 ▼]                                         │
│                                                                   │
│ ── Model settings ──                                            │
│ Traffic split: [100]% to this model                            │
│ ⚡ If endpoint already has models, set traffic split            │
│   (e.g., 90% existing / 10% new for canary deployment).       │
│                                                                   │
│ Minimum compute nodes: [1]                                     │
│ Maximum compute nodes: [5]                                     │
│ ⚡ Min 1 for always-on. Set to 0 for scale-to-zero.            │
│                                                                   │
│ Machine type: [n1-standard-4 ▼]                                 │
│ Accelerator: [None ▼] (or NVIDIA T4, V100, A100)               │
│                                                                   │
│ ── Model monitoring (optional) ──                               │
│ ☑ Enable prediction monitoring                                 │
│   Sampling rate: [10]%                                         │
│   Alert emails: [ml-team@techcorp.com]                         │
│                                                                   │
│ ── Logging ──                                                    │
│ ☑ Enable access logging (to Cloud Logging)                    │
│ ☑ Enable request-response logging (to BigQuery)               │
│                                                                   │
│                             [DEPLOY]                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 15 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Dataset ──────────────────────────────────────────────────
resource "google_vertex_ai_dataset" "tabular" {
  display_name        = "customer-churn"
  metadata_schema_uri = "gs://google-cloud-aiplatform/schema/dataset/metadata/tabular_1.0.0.yaml"
  region              = var.region
  project             = var.project_id
}

# ─── Custom Training Job ─────────────────────────────────────
resource "google_vertex_ai_custom_job" "training" {
  display_name = "churn-training"
  region       = var.region
  project      = var.project_id

  job_spec {
    worker_pool_specs {
      machine_spec {
        machine_type      = "n1-standard-8"
        accelerator_type  = "NVIDIA_TESLA_T4"
        accelerator_count = 1
      }
      replica_count = 1
      container_spec {
        image_uri = "gcr.io/${var.project_id}/train:latest"
        args      = ["--epochs=50", "--lr=0.01"]
      }
    }
  }
}

# ─── Endpoint ─────────────────────────────────────────────────
resource "google_vertex_ai_endpoint" "prediction" {
  name         = "churn-endpoint"
  display_name = "Churn Prediction Endpoint"
  location     = var.region
  project      = var.project_id
}

# ─── Feature Store ────────────────────────────────────────────
resource "google_vertex_ai_feature_online_store" "store" {
  name    = "customer-features-store"
  region  = var.region
  project = var.project_id

  bigtable {
    auto_scaling {
      min_node_count = 1
      max_node_count = 3
      cpu_utilization_target = 70
    }
  }
}

# ─── Workbench Instance ──────────────────────────────────────
resource "google_workbench_instance" "notebook" {
  name     = "ml-notebook"
  location = "${var.region}-a"
  project  = var.project_id

  gce_setup {
    machine_type = "n1-standard-8"
    accelerator_configs {
      type       = "NVIDIA_TESLA_T4"
      core_count = 1
    }
    disable_public_ip = true
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# CUSTOM TRAINING
# ═══════════════════════════════════════════════════════════════
gcloud ai custom-jobs create --region=R \
    --display-name=NAME \
    --worker-pool-spec=machine-type=TYPE,replica-count=N,\
container-image-uri=IMAGE

gcloud ai custom-jobs list --region=R
gcloud ai custom-jobs describe JOB_ID --region=R

# ═══════════════════════════════════════════════════════════════
# MODELS
# ═══════════════════════════════════════════════════════════════
gcloud ai models upload --region=R \
    --display-name=NAME \
    --artifact-uri=gs://bucket/model/ \
    --container-image-uri=IMAGE

gcloud ai models list --region=R
gcloud ai models describe MODEL_ID --region=R
gcloud ai models delete MODEL_ID --region=R

# ═══════════════════════════════════════════════════════════════
# ENDPOINTS
# ═══════════════════════════════════════════════════════════════
gcloud ai endpoints create --region=R --display-name=NAME
gcloud ai endpoints deploy-model ENDPOINT_ID --region=R \
    --model=MODEL_ID \
    --display-name=NAME \
    --machine-type=TYPE \
    --min-replica-count=1 \
    --max-replica-count=5

gcloud ai endpoints predict ENDPOINT_ID --region=R \
    --json-request=request.json

gcloud ai endpoints list --region=R

# ═══════════════════════════════════════════════════════════════
# PIPELINES
# ═══════════════════════════════════════════════════════════════
gcloud ai pipelines create --region=R \
    --display-name=NAME \
    --template-path=pipeline.yaml

gcloud ai pipelines list --region=R

# ═══════════════════════════════════════════════════════════════
# WORKBENCH
# ═══════════════════════════════════════════════════════════════
gcloud workbench instances create NAME --location=ZONE [opts]
gcloud workbench instances list --location=ZONE
gcloud workbench instances start NAME --location=ZONE
gcloud workbench instances stop NAME --location=ZONE
gcloud workbench instances delete NAME --location=ZONE
```

---

## Part 15 — Real-World Patterns

### Pattern 1: End-to-End MLOps Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: MLOPS PIPELINE                                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Vertex AI Pipeline (scheduled weekly)                     │        │
│  │                                                            │        │
│  │  1. Data Validation                                       │        │
│  │     → Check schema, distributions, completeness          │        │
│  │     → Alert if data drift detected                       │        │
│  │                                                            │        │
│  │  2. Feature Engineering                                   │        │
│  │     → Compute features from BigQuery tables              │        │
│  │     → Update Feature Store                               │        │
│  │                                                            │        │
│  │  3. Train Model                                           │        │
│  │     → Custom training on GPU (T4)                        │        │
│  │     → Log metrics to Experiments                         │        │
│  │                                                            │        │
│  │  4. Evaluate                                              │        │
│  │     → Compare with champion model                        │        │
│  │     → If better → continue, else → stop                 │        │
│  │                                                            │        │
│  │  5. Register Model                                        │        │
│  │     → Upload to Model Registry with version              │        │
│  │                                                            │        │
│  │  6. Deploy (Canary)                                       │        │
│  │     → Deploy to endpoint with 10% traffic               │        │
│  │     → Monitor for 24 hours                               │        │
│  │     → If OK → shift to 100%                             │        │
│  │                                                            │        │
│  │  7. Monitor                                               │        │
│  │     → Continuous model monitoring (skew + drift)         │        │
│  │     → Alert → retrigger pipeline if degradation          │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: RAG-Powered Chatbot

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: RAG CHATBOT                                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Indexing pipeline (offline):                              │        │
│  │  Documents → Chunk → Embed → Vector Search index         │        │
│  │                                                            │        │
│  │  ┌──────────┐  ┌──────────┐  ┌─────────────────┐       │        │
│  │  │ GCS docs │→ │ Cloud Run│→ │ Vertex AI        │       │        │
│  │  │ (PDFs,   │  │ chunker  │  │ Vector Search    │       │        │
│  │  │  HTML)   │  │ embedder │  │ (index)          │       │        │
│  │  └──────────┘  └──────────┘  └─────────────────┘       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Query pipeline (online):                                  │        │
│  │                                                            │        │
│  │  User question                                            │        │
│  │       │                                                    │        │
│  │       ▼ embed query                                       │        │
│  │  ┌──────────────┐                                        │        │
│  │  │ Vector Search│ → retrieve top-10 relevant chunks      │        │
│  │  └──────┬───────┘                                        │        │
│  │         ▼                                                  │        │
│  │  ┌──────────────┐                                        │        │
│  │  │ Gemini Pro   │ → generate answer with context         │        │
│  │  │ (grounded)   │   "Based on our documentation: ..."    │        │
│  │  └──────────────┘                                        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Stack: Gemini + Vector Search + Cloud Run + Eventarc              │
│  Eventarc triggers re-indexing when new docs uploaded              │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Real-Time Fraud Detection

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: REAL-TIME ML INFERENCE                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Transaction comes in                                      │        │
│  │       │                                                    │        │
│  │       ▼                                                    │        │
│  │  Cloud Run (API gateway)                                  │        │
│  │       │                                                    │        │
│  │       ├──► Feature Store (online) → get customer features │        │
│  │       │    latency: ~5 ms                                  │        │
│  │       │                                                    │        │
│  │       ├──► Vertex AI Endpoint → fraud prediction          │        │
│  │       │    model: XGBoost (custom trained)                │        │
│  │       │    latency: ~20 ms                                 │        │
│  │       │                                                    │        │
│  │       ├──► Decision: approve / decline / review           │        │
│  │       │                                                    │        │
│  │       └──► Pub/Sub → log prediction + feedback            │        │
│  │                  │                                          │        │
│  │                  ▼                                          │        │
│  │            BigQuery (predictions table)                    │        │
│  │            Model monitoring (detect drift)                │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Total latency: < 50 ms end-to-end                                  │
│  Model retrained weekly via Vertex AI Pipeline                      │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command / Code |
|--------|---------------|
| Init SDK | `aiplatform.init(project=P, location=L)` |
| Upload model | `aiplatform.Model.upload(artifact_uri=U, container_image_uri=I)` |
| Create endpoint | `aiplatform.Endpoint.create(display_name=N)` |
| Deploy model | `endpoint.deploy(model=M, machine_type=T)` |
| Online predict | `endpoint.predict(instances=[...])` |
| AutoML tabular | `aiplatform.AutoMLTabularTrainingJob(...)` |
| Custom training | `aiplatform.CustomTrainingJob(script_path=S, container_uri=C)` |
| Run pipeline | `aiplatform.PipelineJob(template_path=T).run()` |
| Gemini text | `GenerativeModel('gemini-1.5-pro').generate_content(prompt)` |
| Create notebook | `gcloud workbench instances create NAME --location=Z` |
| List models | `gcloud ai models list --region=R` |

---

## What is Machine Learning? (Beginner Explanation)

### ML in Simple Terms

**Machine Learning is like teaching a computer to recognize patterns.** Show it 1,000 photos of cats and 1,000 photos of dogs, and it learns to tell the difference — without you writing explicit rules like "cats have pointy ears" or "dogs have longer snouts." The computer figures out the patterns on its own.

```
┌────────────────────────────────────────────────────────────────────┐
│         TRADITIONAL PROGRAMMING vs MACHINE LEARNING                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Traditional Programming:                                          │
│  ┌───────────┐   ┌──────────┐                                     │
│  │ Data      │ + │ Rules    │  →  Answers                         │
│  │ (photo)   │   │ (if ears │                                     │
│  │           │   │  pointy  │                                     │
│  │           │   │  = cat)  │                                     │
│  └───────────┘   └──────────┘                                     │
│                                                                      │
│  Machine Learning:                                                 │
│  ┌───────────┐   ┌──────────┐                                     │
│  │ Data      │ + │ Answers  │  →  Rules (model)                   │
│  │ (1000     │   │ (this is │                                     │
│  │  photos)  │   │  a cat)  │                                     │
│  └───────────┘   └──────────┘                                     │
│                                                                      │
│  The computer LEARNS the rules from examples!                     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Why Vertex AI?

Setting up ML from scratch is hard — you'd need to provision servers, install GPU drivers, manage Python dependencies, build training infrastructure, set up model serving, and handle scaling. **Vertex AI is a fully managed platform** that handles all of this for you:

| Without Vertex AI | With Vertex AI |
|-------------------|----------------|
| Provision GPU servers manually | Google manages the infrastructure |
| Install CUDA, cuDNN, TensorFlow | Pre-built containers ready to go |
| Build your own training pipeline | Managed training jobs |
| Set up model serving (Flask/FastAPI) | One-click endpoint deployment |
| Handle autoscaling yourself | Built-in autoscaling |
| Build monitoring from scratch | Integrated model monitoring |
| Manage experiment tracking | Built-in experiment tracking |

> **Think of it this way:** Vertex AI is to ML what Cloud Run is to web apps — you focus on the code/model, Google handles the servers.

### AutoML vs Custom Training — When to Use Which

```
┌────────────────────────────────────────────────────────────────────┐
│         CHOOSING YOUR TRAINING APPROACH                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐ │
│  │       AutoML                 │  │     Custom Training          │ │
│  │                              │  │                              │ │
│  │  ✅ No ML expertise needed   │  │  ✅ Full control over model  │ │
│  │  ✅ Upload data → get model  │  │  ✅ Use any framework         │ │
│  │  ✅ Great for tabular data   │  │  ✅ Advanced architectures   │ │
│  │  ✅ Quick prototyping        │  │  ✅ Custom preprocessing     │ │
│  │                              │  │                              │ │
│  │  Best when:                  │  │  Best when:                  │ │
│  │  • You have clean data       │  │  • You have ML expertise     │ │
│  │  • Standard problem type     │  │  • Unique model requirements │ │
│  │  • Need results fast         │  │  • Need to optimize deeply   │ │
│  │  • Limited ML team           │  │  • Specific framework needed │ │
│  │                              │  │                              │ │
│  │  "I have a CSV with labels,  │  │  "I need a custom transformer│ │
│  │   predict the label"         │  │   with a special loss fn"    │ │
│  └─────────────────────────────┘  └─────────────────────────────┘ │
│                                                                      │
│  💡 Start with AutoML to get a baseline, then go custom if needed │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

| Question | AutoML | Custom Training |
|----------|--------|----------------|
| Do I need ML knowledge? | No | Yes |
| Time to first model? | Hours | Days to weeks |
| Can I use PyTorch/TensorFlow? | No (automatic) | Yes (any framework) |
| Can I customize the architecture? | No | Yes |
| Typical use case | "Will this customer churn?" | "Custom image segmentation model" |
| Cost | Higher per training hour | Lower per hour, but more hours |

---

## Console Walkthrough: Cleanup & Deletion

> **⚠️ Important:** Endpoints cost money even when idle! A deployed model on an endpoint charges per node-hour (e.g., ~$0.12/hr for n1-standard-2), which adds up to **~$87/month** even with zero predictions. Always clean up after experiments.

### Step 1: Undeploy Model from Endpoint

```
Console → Vertex AI → Online prediction → Endpoints → Click endpoint

┌─────────────────────────────────────────────────────────────────┐
│           ENDPOINT DETAILS                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Endpoint: churn-prediction-endpoint                              │
│ Status: Active                                                   │
│                                                                   │
│ ── Deployed models ──                                           │
│ ┌───────────────────────────────────────────────────────┐      │
│ │ Model: churn-v1          Traffic: 100%                │      │
│ │ Machine: n1-standard-4   Replicas: 1-5               │      │
│ │                                                       │      │
│ │ ⚡ This model is running and costing you money!        │      │
│ │                                                       │      │
│ │ [⋮] → UNDEPLOY MODEL FROM ENDPOINT                   │      │
│ └───────────────────────────────────────────────────────┘      │
│                                                                   │
│ Confirm: "Undeploy churn-v1 from this endpoint?"               │
│                                     [CANCEL] [UNDEPLOY]         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# CLI: Undeploy model from endpoint
# First, find the deployed model ID
gcloud ai endpoints describe ENDPOINT_ID --region=us-central1 \
    --format="value(deployedModels.id)"

# Then undeploy
gcloud ai endpoints undeploy-model ENDPOINT_ID \
    --region=us-central1 \
    --deployed-model-id=DEPLOYED_MODEL_ID
```

### Step 2: Delete the Endpoint

```
Console → Vertex AI → Online prediction → Endpoints
→ Select endpoint → [⋮] → DELETE

┌─────────────────────────────────────────────────────────────────┐
│           DELETE ENDPOINT                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ⚠️  Are you sure you want to delete "churn-prediction-endpoint"?│
│                                                                   │
│ ⚡ You must undeploy all models BEFORE deleting the endpoint.    │
│   If models are still deployed, the delete will fail.            │
│                                                                   │
│                                     [CANCEL] [DELETE]            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# CLI: Delete endpoint (must have no deployed models)
gcloud ai endpoints delete ENDPOINT_ID --region=us-central1
```

### Step 3: Delete the Model

```
Console → Vertex AI → Model Registry → Select model → [⋮] → DELETE

┌─────────────────────────────────────────────────────────────────┐
│           DELETE MODEL                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ⚠️  Are you sure you want to delete "churn-model-v1"?           │
│                                                                   │
│ ⚡ The model must NOT be deployed to any endpoint.               │
│   All model versions will be deleted.                            │
│   Model artifacts in GCS are NOT deleted automatically.         │
│                                                                   │
│                                     [CANCEL] [DELETE]            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# CLI: Delete model
gcloud ai models delete MODEL_ID --region=us-central1
```

### Step 4: Delete the Dataset

```
Console → Vertex AI → Datasets → Select dataset → [⋮] → DELETE

┌─────────────────────────────────────────────────────────────────┐
│           DELETE DATASET                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ⚠️  Are you sure you want to delete "customer-churn-data"?      │
│                                                                   │
│ ⚡ This removes the dataset metadata from Vertex AI.             │
│   Your actual data in GCS/BigQuery is NOT deleted.              │
│   Trained models are NOT affected.                              │
│                                                                   │
│                                     [CANCEL] [DELETE]            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# CLI: Delete dataset
gcloud ai datasets delete DATASET_ID --region=us-central1
```

### Cleanup Order (Important!)

```
┌────────────────────────────────────────────────────────────────────┐
│         CLEANUP ORDER — FOLLOW THIS SEQUENCE                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Undeploy model from endpoint     ← MUST do first             │
│       │                                                             │
│  2. Delete endpoint                  ← Stops billing!            │
│       │                                                             │
│  3. Delete model from registry       ← Free, but tidy up         │
│       │                                                             │
│  4. Delete dataset                   ← Free, but tidy up         │
│       │                                                             │
│  5. (Optional) Delete GCS artifacts  ← If you don't need them   │
│       │                                                             │
│  6. (Optional) Stop Workbench        ← Also costs money!         │
│                                                                      │
│  ⚠️  You CANNOT delete an endpoint with deployed models.          │
│  ⚠️  You CANNOT delete a model that is deployed to an endpoint.   │
│                                                                      │
│  💡 TIP: Set a budget alert in Billing to catch forgotten         │
│     endpoints! Even 1 idle endpoint = ~$87/month.                 │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 59: AI/ML APIs** → `59-ai-ml-apis.md`
