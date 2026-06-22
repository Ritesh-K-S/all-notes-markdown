# Cloud ML Platforms

## Table of Contents
- [What Are Cloud ML Platforms?](#what-are-cloud-ml-platforms)
- [Why They Matter](#why-they-matter)
- [AWS SageMaker](#aws-sagemaker)
- [GCP Vertex AI](#gcp-vertex-ai)
- [Azure Machine Learning](#azure-machine-learning)
- [Platform Comparison](#platform-comparison)
- [How to Choose](#how-to-choose)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What Are Cloud ML Platforms?

### Simple Explanation

Imagine you're a chef who needs a kitchen to cook. You have three options:
1. **Build your own kitchen** (on-premise) — Expensive, full control, takes months
2. **Rent kitchen equipment** (IaaS - EC2/VMs) — You manage the cooking setup
3. **Use a professional kitchen service** (Cloud ML Platforms) — They provide ovens, prep tables, dishwashers, and a menu system. You just focus on your recipes.

Cloud ML Platforms are like **professional kitchen services for machine learning**. They provide pre-built infrastructure for every step: data storage, training compute, experiment tracking, model serving, and monitoring — all integrated.

### Formal Definition

Cloud ML Platforms are **managed services** that provide end-to-end machine learning infrastructure, including:
- Scalable compute for training (including GPUs/TPUs)
- Managed notebooks for experimentation
- Automated ML (AutoML) capabilities
- Model registry and versioning
- One-click deployment with autoscaling
- Built-in monitoring and retraining pipelines

---

## Why They Matter

### The Build vs Buy Decision

| Aspect | Self-Managed (K8s + Custom) | Cloud ML Platform |
|--------|---------------------------|-------------------|
| **Setup time** | Weeks to months | Hours to days |
| **GPU management** | Complex (drivers, scheduling) | Automatic |
| **Cost at small scale** | High (min infrastructure) | Pay-per-use |
| **Cost at large scale** | Can be cheaper | Can be expensive |
| **Team expertise needed** | ML + DevOps + Infra | ML only |
| **Vendor lock-in** | None | Moderate to High |
| **Customization** | Unlimited | Platform constraints |
| **Compliance** | You handle everything | Built-in certifications |

### When to Use Cloud ML Platforms

- **Startups** — Ship fast without infrastructure team
- **Enterprise** — Compliance, governance, audit trails built-in
- **GPU-heavy workloads** — Don't want to manage GPU clusters
- **Teams without DevOps** — Data scientists who want to deploy
- **Variable workloads** — Scale to zero when not training

### When NOT to Use Them

- **Very simple models** — A scikit-learn model on a cron job doesn't need SageMaker
- **Extremely cost-sensitive** — At high volumes, self-managed can be 3-5x cheaper
- **Strict data residency** — Some regions may not have all services
- **Edge/on-device ML** — Cloud platforms are for cloud deployment

---

## AWS SageMaker

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS SAGEMAKER ECOSYSTEM                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │ SageMaker│   │ SageMaker│   │ SageMaker│   │SageMaker│ │
│  │ Studio   │   │ Training │   │ Endpoints│   │Pipelines│ │
│  │(Notebook)│   │  Jobs    │   │ (Serving)│   │  (MLOps)│ │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬────┘ │
│       │               │               │              │       │
│       ▼               ▼               ▼              ▼       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              S3 (Data & Model Storage)                 │   │
│  └──────────────────────────────────────────────────────┘   │
│       │               │               │              │       │
│       ▼               ▼               ▼              ▼       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │ Feature  │   │  Model   │   │ Ground   │   │  Model  │ │
│  │  Store   │   │ Registry │   │  Truth   │   │ Monitor │ │
│  └──────────┘   └──────────┘   └──────────┘   └─────────┘ │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose | Use When |
|-----------|---------|----------|
| **Studio** | IDE for ML development | Interactive experimentation |
| **Training Jobs** | Managed training compute | Need GPUs/distributed training |
| **Endpoints** | Real-time model serving | Low-latency predictions |
| **Batch Transform** | Batch predictions | Process large datasets offline |
| **Pipelines** | ML workflow orchestration | Automated training/deploy |
| **Feature Store** | Feature management | Shared features across teams |
| **Model Monitor** | Drift detection | Production model health |
| **Autopilot** | AutoML | Quick baseline models |

### Code Example: Full SageMaker Workflow

```python
# sagemaker_training.py
"""
Complete AWS SageMaker workflow:
Data prep → Training → Evaluation → Deployment
"""
import sagemaker
import boto3
from sagemaker import get_execution_role
from sagemaker.sklearn import SKLearn
from sagemaker.model_monitor import DefaultModelMonitor
import pandas as pd
import json

# ============================================
# SETUP
# ============================================

# Get SageMaker session and role
session = sagemaker.Session()
role = get_execution_role()  # IAM role with SageMaker permissions
bucket = session.default_bucket()  # Default S3 bucket
prefix = 'loan-approval-model'  # S3 prefix for this project

print(f"Role: {role}")
print(f"Bucket: {bucket}")
print(f"Region: {session.boto_region_name}")


# ============================================
# STEP 1: Upload Data to S3
# ============================================

# Upload training data to S3
train_path = session.upload_data(
    path='data/train.csv',          # Local file
    bucket=bucket,
    key_prefix=f'{prefix}/data/train'
)
test_path = session.upload_data(
    path='data/test.csv',
    bucket=bucket,
    key_prefix=f'{prefix}/data/test'
)

print(f"Training data: {train_path}")
print(f"Test data: {test_path}")


# ============================================
# STEP 2: Create Training Script
# ============================================

# This script runs ON the training instance (not locally)
training_script = """
# train.py - Runs on SageMaker training instance
import argparse
import os
import pandas as pd
import numpy as np
import joblib
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import accuracy_score, f1_score, classification_report

def main():
    # SageMaker passes hyperparameters as command-line arguments
    parser = argparse.ArgumentParser()
    parser.add_argument('--n-estimators', type=int, default=100)
    parser.add_argument('--max-depth', type=int, default=5)
    parser.add_argument('--learning-rate', type=float, default=0.1)
    
    # SageMaker-specific environment variables
    parser.add_argument('--model-dir', type=str, default=os.environ.get('SM_MODEL_DIR'))
    parser.add_argument('--train', type=str, default=os.environ.get('SM_CHANNEL_TRAIN'))
    parser.add_argument('--test', type=str, default=os.environ.get('SM_CHANNEL_TEST'))
    
    args = parser.parse_args()
    
    # Load data from the SageMaker-provided paths
    train_df = pd.read_csv(os.path.join(args.train, 'train.csv'))
    test_df = pd.read_csv(os.path.join(args.test, 'test.csv'))
    
    # Split features/target (first column is target by SageMaker convention)
    X_train = train_df.iloc[:, 1:]
    y_train = train_df.iloc[:, 0]
    X_test = test_df.iloc[:, 1:]
    y_test = test_df.iloc[:, 0]
    
    # Train model
    model = GradientBoostingClassifier(
        n_estimators=args.n_estimators,
        max_depth=args.max_depth,
        learning_rate=args.learning_rate,
        random_state=42
    )
    model.fit(X_train, y_train)
    
    # Evaluate
    y_pred = model.predict(X_test)
    accuracy = accuracy_score(y_test, y_pred)
    f1 = f1_score(y_test, y_pred, average='weighted')
    
    # Print metrics (SageMaker captures these from stdout)
    print(f"accuracy: {accuracy:.4f}")
    print(f"f1_score: {f1:.4f}")
    print(classification_report(y_test, y_pred))
    
    # Save model to the model directory
    # SageMaker automatically uploads this to S3
    joblib.dump(model, os.path.join(args.model_dir, 'model.pkl'))

if __name__ == '__main__':
    main()
"""


# ============================================
# STEP 3: Configure and Launch Training Job
# ============================================

# Use SageMaker's SKLearn estimator
sklearn_estimator = SKLearn(
    entry_point='train.py',          # Training script
    source_dir='src/',               # Directory containing train.py
    role=role,
    instance_count=1,                # Number of training instances
    instance_type='ml.m5.xlarge',    # Instance type (4 vCPU, 16 GB RAM)
    framework_version='1.2-1',       # Scikit-learn version
    py_version='py3',
    
    # Hyperparameters passed to training script
    hyperparameters={
        'n-estimators': 200,
        'max-depth': 7,
        'learning-rate': 0.1
    },
    
    # Metric definitions for tracking
    metric_definitions=[
        {'Name': 'accuracy', 'Regex': 'accuracy: ([0-9\\.]+)'},
        {'Name': 'f1_score', 'Regex': 'f1_score: ([0-9\\.]+)'}
    ],
    
    # Tags for cost tracking
    tags=[
        {'Key': 'Project', 'Value': 'LoanApproval'},
        {'Key': 'Environment', 'Value': 'Development'}
    ]
)

# Launch training (this creates EC2 instances, trains, then shuts down)
sklearn_estimator.fit({
    'train': train_path,    # S3 path to training data
    'test': test_path       # S3 path to test data
})

print(f"Training job: {sklearn_estimator.latest_training_job.name}")
print(f"Model artifact: {sklearn_estimator.model_data}")


# ============================================
# STEP 4: Deploy Model as Real-time Endpoint
# ============================================

# Deploy with a single line
predictor = sklearn_estimator.deploy(
    initial_instance_count=1,        # Start with 1 instance
    instance_type='ml.t2.medium',    # Cheap instance for serving
    endpoint_name='loan-approval-endpoint'
)

# Test the endpoint
import numpy as np
test_input = np.array([[35, 75000, 720, 5, 2]])  # age, income, credit, years, dependents
prediction = predictor.predict(test_input)
print(f"Prediction: {prediction}")


# ============================================
# STEP 5: Set Up Autoscaling
# ============================================

# Configure autoscaling for the endpoint
client = boto3.client('application-autoscaling')

# Register the endpoint as a scalable target
client.register_scalable_target(
    ServiceNamespace='sagemaker',
    ResourceId=f'endpoint/loan-approval-endpoint/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    MinCapacity=1,
    MaxCapacity=10
)

# Create scaling policy based on invocations per instance
client.put_scaling_policy(
    PolicyName='LoanApprovalScalingPolicy',
    ServiceNamespace='sagemaker',
    ResourceId=f'endpoint/loan-approval-endpoint/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    PolicyType='TargetTrackingScaling',
    TargetTrackingScalingPolicyConfiguration={
        'TargetValue': 1000,  # Target 1000 invocations per instance per minute
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'SageMakerVariantInvocationsPerInstance'
        },
        'ScaleInCooldown': 300,   # Wait 5 min before scaling in
        'ScaleOutCooldown': 60    # Wait 1 min before scaling out
    }
)


# ============================================
# STEP 6: Set Up Model Monitoring
# ============================================

from sagemaker.model_monitor import DataCaptureConfig

# Enable data capture on the endpoint
data_capture_config = DataCaptureConfig(
    enable_capture=True,
    sampling_percentage=100,          # Capture all requests
    destination_s3_uri=f's3://{bucket}/{prefix}/data-capture'
)

# Create a monitoring schedule
from sagemaker.model_monitor.dataset_format import DatasetFormat

monitor = DefaultModelMonitor(
    role=role,
    instance_count=1,
    instance_type='ml.m5.xlarge'
)

# Create baseline from training data
monitor.suggest_baseline(
    baseline_dataset=train_path,
    dataset_format=DatasetFormat.csv(header=True)
)

# Schedule monitoring (runs hourly)
from sagemaker.model_monitor import CronExpressionGenerator

monitor.create_monitoring_schedule(
    monitor_schedule_name='loan-approval-monitor',
    endpoint_input='loan-approval-endpoint',
    output_s3_uri=f's3://{bucket}/{prefix}/monitoring-results',
    statistics=monitor.baseline_statistics(),
    constraints=monitor.suggested_constraints(),
    schedule_cron_expression=CronExpressionGenerator.hourly()
)


# ============================================
# STEP 7: SageMaker Pipelines (MLOps)
# ============================================

from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import TrainingStep, ProcessingStep
from sagemaker.workflow.parameters import ParameterString, ParameterFloat
from sagemaker.processing import SKLearnProcessor

# Define pipeline parameters
input_data = ParameterString(name="InputData", default_value=train_path)
model_approval_status = ParameterString(name="ModelApprovalStatus", default_value="PendingManualApproval")

# Processing step
sklearn_processor = SKLearnProcessor(
    framework_version='1.2-1',
    role=role,
    instance_type='ml.m5.xlarge',
    instance_count=1
)

# Training step
training_step = TrainingStep(
    name="TrainModel",
    estimator=sklearn_estimator,
    inputs={
        'train': sagemaker.inputs.TrainingInput(s3_data=input_data),
        'test': sagemaker.inputs.TrainingInput(s3_data=test_path)
    }
)

# Create pipeline
pipeline = Pipeline(
    name="LoanApprovalPipeline",
    parameters=[input_data, model_approval_status],
    steps=[training_step]
)

# Submit pipeline
pipeline.upsert(role_arn=role)
execution = pipeline.start()
print(f"Pipeline execution: {execution.arn}")


# ============================================
# CLEANUP (Important for cost management!)
# ============================================

def cleanup():
    """Delete all resources to avoid ongoing charges."""
    # Delete endpoint (stops billing immediately)
    predictor.delete_endpoint()
    
    # Delete model
    predictor.delete_model()
    
    # Stop monitoring
    monitor.delete_monitoring_schedule()
    
    print("All resources cleaned up!")

# Uncomment when done:
# cleanup()
```

### SageMaker Cost Optimization Tips

```python
# Use Spot Instances for training (up to 90% savings)
from sagemaker.estimator import Estimator

estimator = Estimator(
    image_uri='...',
    role=role,
    instance_count=1,
    instance_type='ml.p3.2xlarge',  # GPU instance
    use_spot_instances=True,         # 🔑 Use spot instances
    max_wait=7200,                   # Max wait time (2 hours)
    max_run=3600,                    # Max training time (1 hour)
    checkpoint_s3_uri=f's3://{bucket}/checkpoints'  # Resume if interrupted
)

# Use SageMaker Serverless Inference for low-traffic endpoints
from sagemaker.serverless import ServerlessInferenceConfig

serverless_config = ServerlessInferenceConfig(
    memory_size_in_mb=2048,    # 2 GB memory
    max_concurrency=5          # Max 5 concurrent invocations
)

predictor = model.deploy(
    serverless_inference_config=serverless_config  # No instance management!
)
# Cost: Pay only when invoked (~$0.0001 per request)
```

---

## GCP Vertex AI

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    GCP VERTEX AI ECOSYSTEM                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │ Workbench│   │ Training │   │Prediction │   │Pipelines│ │
│  │(Notebook)│   │  (Custom │   │(Endpoints)│   │ (KFP)   │ │
│  │          │   │  + AutoML)│   │           │   │         │ │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬────┘ │
│       │               │               │              │       │
│       ▼               ▼               ▼              ▼       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         GCS (Google Cloud Storage)                     │   │
│  └──────────────────────────────────────────────────────┘   │
│       │               │               │              │       │
│       ▼               ▼               ▼              ▼       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │ Feature  │   │  Model   │   │ Metadata │   │  Model  │ │
│  │  Store   │   │ Registry │   │  Store   │   │ Monitor │ │
│  └──────────┘   └──────────┘   └──────────┘   └─────────┘ │
│                                                               │
│  UNIQUE: TPU access, BigQuery ML, Gemini integration         │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Code Example: Vertex AI Workflow

```python
# vertex_ai_workflow.py
"""
Complete GCP Vertex AI workflow:
Custom training → Model upload → Endpoint deployment
"""
from google.cloud import aiplatform
from google.cloud import storage
import json

# ============================================
# SETUP
# ============================================

# Initialize Vertex AI
PROJECT_ID = "your-gcp-project-id"
REGION = "us-central1"
BUCKET_URI = f"gs://{PROJECT_ID}-ml-assets"

aiplatform.init(
    project=PROJECT_ID,
    location=REGION,
    staging_bucket=BUCKET_URI
)


# ============================================
# STEP 1: Custom Training Job
# ============================================

# Define custom training job
custom_job = aiplatform.CustomTrainingJob(
    display_name="loan-approval-training",
    script_path="src/train.py",              # Local training script
    container_uri="us-docker.pkg.dev/vertex-ai/training/sklearn-cpu.1-2:latest",
    requirements=["pandas", "scikit-learn", "joblib"],
    model_serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-2:latest"
)

# Run training
model = custom_job.run(
    dataset=None,  # Or pass a Vertex AI managed dataset
    model_display_name="loan-approval-model",
    args=[
        "--n-estimators", "200",
        "--max-depth", "7",
        "--learning-rate", "0.1",
        "--data-path", f"{BUCKET_URI}/data/"
    ],
    replica_count=1,
    machine_type="n1-standard-4",      # 4 vCPU, 15 GB RAM
    # accelerator_type="NVIDIA_TESLA_T4",  # Uncomment for GPU
    # accelerator_count=1,
)

print(f"Model resource name: {model.resource_name}")


# ============================================
# STEP 2: Upload Model to Registry
# ============================================

# If you have a pre-trained model, upload directly
model = aiplatform.Model.upload(
    display_name="loan-approval-v2",
    artifact_uri=f"{BUCKET_URI}/models/latest/",  # GCS path to model files
    serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-2:latest",
    labels={"team": "ml-platform", "version": "2"},
    description="Loan approval model v2 - GBM with 200 estimators"
)


# ============================================
# STEP 3: Deploy to Endpoint
# ============================================

# Create an endpoint
endpoint = aiplatform.Endpoint.create(
    display_name="loan-approval-endpoint",
    labels={"env": "production"}
)

# Deploy model to the endpoint
model.deploy(
    endpoint=endpoint,
    deployed_model_display_name="loan-approval-v2",
    machine_type="n1-standard-2",
    min_replica_count=1,         # Minimum instances
    max_replica_count=5,         # Maximum for autoscaling
    traffic_percentage=100,      # 100% traffic to this model
    # For A/B testing, deploy multiple models with different traffic %
)

print(f"Endpoint: {endpoint.resource_name}")


# ============================================
# STEP 4: Make Predictions
# ============================================

# Online prediction
instances = [
    {"age": 30, "income": 65000, "credit_score": 720, 
     "employment_years": 5, "dependents": 2}
]

prediction = endpoint.predict(instances=instances)
print(f"Prediction: {prediction.predictions}")
print(f"Deployed model ID: {prediction.deployed_model_id}")


# ============================================
# STEP 5: Batch Prediction
# ============================================

batch_prediction_job = model.batch_predict(
    job_display_name="loan-approval-batch",
    gcs_source=f"{BUCKET_URI}/data/batch_input.jsonl",
    gcs_destination_prefix=f"{BUCKET_URI}/predictions/",
    machine_type="n1-standard-4",
    starting_replica_count=2,
    max_replica_count=10
)

batch_prediction_job.wait()
print(f"Batch output: {batch_prediction_job.output_info}")


# ============================================
# STEP 6: Vertex AI Pipelines (KFP v2)
# ============================================

from kfp.v2 import dsl, compiler
from kfp.v2.dsl import component, Input, Output, Dataset, Model, Metrics
from google.cloud import aiplatform as aip

@component(
    base_image="python:3.10",
    packages_to_install=["pandas", "scikit-learn", "google-cloud-storage"]
)
def load_data(
    data_uri: str,
    dataset: Output[Dataset]
):
    """Load data from GCS."""
    import pandas as pd
    df = pd.read_csv(data_uri)
    df.to_csv(dataset.path, index=False)

@component(
    base_image="python:3.10",
    packages_to_install=["pandas", "scikit-learn", "joblib"]
)
def train_model(
    dataset: Input[Dataset],
    n_estimators: int,
    model_artifact: Output[Model],
    metrics: Output[Metrics]
):
    """Train and evaluate model."""
    import pandas as pd
    from sklearn.ensemble import GradientBoostingClassifier
    from sklearn.model_selection import train_test_split
    from sklearn.metrics import accuracy_score, f1_score
    import joblib
    
    df = pd.read_csv(dataset.path)
    X = df.iloc[:, 1:]
    y = df.iloc[:, 0]
    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
    
    model = GradientBoostingClassifier(n_estimators=n_estimators, random_state=42)
    model.fit(X_train, y_train)
    
    y_pred = model.predict(X_test)
    accuracy = accuracy_score(y_test, y_pred)
    f1 = f1_score(y_test, y_pred, average='weighted')
    
    # Log metrics
    metrics.log_metric("accuracy", accuracy)
    metrics.log_metric("f1_score", f1)
    
    # Save model
    joblib.dump(model, model_artifact.path)

@dsl.pipeline(
    name="loan-approval-pipeline",
    description="Training pipeline for loan approval model"
)
def ml_pipeline(
    data_uri: str = f"{BUCKET_URI}/data/train.csv",
    n_estimators: int = 200
):
    """Define the ML pipeline DAG."""
    # Load data
    load_task = load_data(data_uri=data_uri)
    
    # Train model
    train_task = train_model(
        dataset=load_task.outputs["dataset"],
        n_estimators=n_estimators
    )

# Compile and run pipeline
compiler.Compiler().compile(
    pipeline_func=ml_pipeline,
    package_path="pipeline.json"
)

# Submit pipeline run
job = aip.PipelineJob(
    display_name="loan-approval-pipeline-run",
    template_path="pipeline.json",
    pipeline_root=f"{BUCKET_URI}/pipeline-root",
    parameter_values={
        "data_uri": f"{BUCKET_URI}/data/train.csv",
        "n_estimators": 200
    }
)

job.run(sync=True)


# ============================================
# STEP 7: Model Monitoring
# ============================================

from google.cloud.aiplatform import model_monitoring

# Create monitoring job
monitoring_job = aiplatform.ModelDeploymentMonitoringJob.create(
    display_name="loan-approval-monitoring",
    endpoint=endpoint,
    logging_sampling_strategy=model_monitoring.RandomSampleConfig(sample_rate=0.8),
    schedule_config=model_monitoring.ScheduleConfig(monitor_interval=3600),  # hourly
    
    # Drift detection config
    drift_detection_config=model_monitoring.DriftDetectionConfig(
        drift_thresholds={
            "age": model_monitoring.ThresholdConfig(value=0.3),
            "income": model_monitoring.ThresholdConfig(value=0.3),
        }
    ),
    
    # Skew detection (training vs serving)
    skew_detection_config=model_monitoring.SkewDetectionConfig(
        data_source=f"{BUCKET_URI}/data/train.csv",
        skew_thresholds={
            "age": model_monitoring.ThresholdConfig(value=0.3),
        }
    ),
    
    alert_config=model_monitoring.EmailAlertConfig(
        user_emails=["ml-team@company.com"]
    )
)


# ============================================
# CLEANUP
# ============================================

def cleanup():
    """Clean up Vertex AI resources."""
    endpoint.undeploy_all()
    endpoint.delete()
    model.delete()
    print("Resources cleaned up!")

# Uncomment when done:
# cleanup()
```

### BigQuery ML (Unique to GCP)

```sql
-- Train a model directly in BigQuery (no Python needed!)
-- Great for structured/tabular data at massive scale

-- Step 1: Create model
CREATE OR REPLACE MODEL `project.dataset.loan_model`
OPTIONS(
    model_type='BOOSTED_TREE_CLASSIFIER',
    input_label_cols=['loan_approved'],
    max_iterations=200,
    learn_rate=0.1,
    data_split_method='AUTO_SPLIT'
) AS
SELECT
    age,
    income,
    credit_score,
    employment_years,
    loan_approved
FROM `project.dataset.loan_applications`
WHERE application_date > '2024-01-01';

-- Step 2: Evaluate model
SELECT *
FROM ML.EVALUATE(MODEL `project.dataset.loan_model`);

-- Step 3: Make predictions
SELECT *
FROM ML.PREDICT(MODEL `project.dataset.loan_model`,
    (SELECT age, income, credit_score, employment_years
     FROM `project.dataset.new_applications`)
);

-- Step 4: Export model for deployment
EXPORT MODEL `project.dataset.loan_model`
OPTIONS(URI = 'gs://bucket/exported_model/');
```

---

## Azure Machine Learning

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   AZURE ML ECOSYSTEM                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │ Compute  │   │ Designer │   │ Managed  │   │   ML    │ │
│  │Instances │   │ (Low-Code│   │Endpoints │   │Pipelines│ │
│  │(Notebook)│   │  ML)     │   │(Serving) │   │  (SDK)  │ │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬────┘ │
│       │               │               │              │       │
│       ▼               ▼               ▼              ▼       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │       Azure Blob Storage / Data Lake                   │   │
│  └──────────────────────────────────────────────────────┘   │
│       │               │               │              │       │
│       ▼               ▼               ▼              ▼       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │Registered│   │   MLflow  │   │Responsible│   │   Data  │ │
│  │  Models  │   │ Tracking │   │    AI     │   │  Assets │ │
│  └──────────┘   └──────────┘   └──────────┘   └─────────┘ │
│                                                               │
│  UNIQUE: Responsible AI dashboard, Designer (drag-and-drop)  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Code Example: Azure ML Workflow

```python
# azure_ml_workflow.py
"""
Complete Azure ML workflow using SDK v2.
"""
from azure.ai.ml import MLClient, command, Input, Output
from azure.ai.ml.entities import (
    ManagedOnlineEndpoint,
    ManagedOnlineDeployment,
    Model,
    Environment,
    BuildContext
)
from azure.identity import DefaultAzureCredential
from azure.ai.ml.constants import AssetTypes

# ============================================
# SETUP
# ============================================

# Authenticate and create ML client
credential = DefaultAzureCredential()
ml_client = MLClient(
    credential=credential,
    subscription_id="your-subscription-id",
    resource_group_name="your-resource-group",
    workspace_name="your-ml-workspace"
)

print(f"Connected to workspace: {ml_client.workspace_name}")


# ============================================
# STEP 1: Register Data Assets
# ============================================

from azure.ai.ml.entities import Data

# Register training data
train_data = Data(
    name="loan-training-data",
    path="azureml://datastores/workspaceblobstore/paths/data/train.csv",
    type=AssetTypes.URI_FILE,
    description="Loan approval training dataset",
    tags={"source": "data-warehouse", "version": "2024-Q4"}
)
ml_client.data.create_or_update(train_data)


# ============================================
# STEP 2: Create Training Environment
# ============================================

# Option A: From conda specification
env = Environment(
    name="sklearn-training-env",
    description="Environment for sklearn training",
    conda_file="environments/conda.yml",
    image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04:latest"
)
ml_client.environments.create_or_update(env)

# Option B: From Dockerfile
env_docker = Environment(
    build=BuildContext(path="docker/"),
    name="custom-training-env"
)


# ============================================
# STEP 3: Submit Training Job
# ============================================

# Define the training command
training_job = command(
    code="./src",                    # Local source code directory
    command="python train.py --data ${{inputs.training_data}} --n-estimators 200 --output ${{outputs.model}}",
    inputs={
        "training_data": Input(type="uri_file", path="azureml:loan-training-data@latest")
    },
    outputs={
        "model": Output(type="uri_folder", mode="rw_mount")
    },
    environment="sklearn-training-env@latest",
    compute="gpu-cluster",           # Or "cpu-cluster" for CPU
    instance_count=1,
    display_name="loan-approval-training",
    experiment_name="loan-approval",
    tags={"model_type": "gbm", "version": "2.0"}
)

# Submit job
returned_job = ml_client.jobs.create_or_update(training_job)
print(f"Job submitted: {returned_job.name}")
print(f"Studio URL: {returned_job.studio_url}")

# Wait for completion
ml_client.jobs.stream(returned_job.name)


# ============================================
# STEP 4: Register Model
# ============================================

model = Model(
    name="loan-approval-model",
    path=f"azureml://jobs/{returned_job.name}/outputs/model",
    type=AssetTypes.CUSTOM_MODEL,
    description="GBM model for loan approval prediction",
    tags={"accuracy": "0.92", "f1": "0.89"}
)
registered_model = ml_client.models.create_or_update(model)
print(f"Model registered: {registered_model.name} v{registered_model.version}")


# ============================================
# STEP 5: Deploy to Managed Online Endpoint
# ============================================

# Create endpoint
endpoint = ManagedOnlineEndpoint(
    name="loan-approval-endpoint",
    description="Loan approval prediction endpoint",
    auth_mode="key",               # or "aml_token" for AAD auth
    tags={"env": "production"}
)
ml_client.online_endpoints.begin_create_or_update(endpoint).result()

# Create deployment
deployment = ManagedOnlineDeployment(
    name="blue",                   # Blue-green deployment naming
    endpoint_name="loan-approval-endpoint",
    model=registered_model.id,
    environment="sklearn-training-env@latest",
    code_configuration={
        "code": "./src",
        "scoring_script": "score.py"
    },
    instance_type="Standard_DS2_v2",
    instance_count=1,
    # Autoscaling
    scale_settings={
        "type": "default"          # Azure handles autoscaling
    }
)
ml_client.online_deployments.begin_create_or_update(deployment).result()

# Route 100% traffic to this deployment
endpoint.traffic = {"blue": 100}
ml_client.online_endpoints.begin_create_or_update(endpoint).result()

print("Endpoint deployed successfully!")


# ============================================
# STEP 6: Test the Endpoint
# ============================================

import json

# Prepare test data
test_data = {
    "input_data": {
        "columns": ["age", "income", "credit_score", "employment_years", "dependents"],
        "data": [[30, 65000, 720, 5, 2], [45, 120000, 800, 15, 3]]
    }
}

# Invoke endpoint
response = ml_client.online_endpoints.invoke(
    endpoint_name="loan-approval-endpoint",
    request_file=None,
    deployment_name="blue",
    input=json.dumps(test_data)
)

print(f"Predictions: {response}")


# ============================================
# STEP 7: Blue-Green Deployment (A/B Testing)
# ============================================

# Deploy new model version as "green"
green_deployment = ManagedOnlineDeployment(
    name="green",
    endpoint_name="loan-approval-endpoint",
    model=f"azureml:loan-approval-model:2",  # New version
    environment="sklearn-training-env@latest",
    code_configuration={
        "code": "./src",
        "scoring_script": "score.py"
    },
    instance_type="Standard_DS2_v2",
    instance_count=1
)
ml_client.online_deployments.begin_create_or_update(green_deployment).result()

# Send 10% traffic to green (canary)
endpoint.traffic = {"blue": 90, "green": 10}
ml_client.online_endpoints.begin_create_or_update(endpoint).result()

# After validation, shift all traffic to green
endpoint.traffic = {"blue": 0, "green": 100}
ml_client.online_endpoints.begin_create_or_update(endpoint).result()

# Delete old deployment
ml_client.online_deployments.begin_delete(
    name="blue", 
    endpoint_name="loan-approval-endpoint"
).result()


# ============================================
# CLEANUP
# ============================================

def cleanup():
    """Clean up Azure ML resources."""
    ml_client.online_endpoints.begin_delete(name="loan-approval-endpoint").result()
    print("Endpoint deleted!")

# Uncomment when done:
# cleanup()
```

### Azure ML Scoring Script

```python
# src/score.py - Scoring script for Azure ML endpoint
import os
import json
import joblib
import numpy as np

def init():
    """
    Called once when the deployment starts.
    Load model into memory.
    """
    global model
    
    # AZUREML_MODEL_DIR is set by Azure ML
    model_path = os.path.join(os.environ["AZUREML_MODEL_DIR"], "model.pkl")
    model = joblib.load(model_path)
    print(f"Model loaded from {model_path}")

def run(raw_data):
    """
    Called for each prediction request.
    
    Args:
        raw_data: JSON string with input data
        
    Returns:
        JSON string with predictions
    """
    try:
        data = json.loads(raw_data)
        input_data = np.array(data["input_data"]["data"])
        
        predictions = model.predict(input_data).tolist()
        probabilities = model.predict_proba(input_data).tolist()
        
        return json.dumps({
            "predictions": predictions,
            "probabilities": probabilities
        })
    except Exception as e:
        return json.dumps({"error": str(e)})
```

---

## Platform Comparison

### Feature-by-Feature Comparison

| Feature | AWS SageMaker | GCP Vertex AI | Azure ML |
|---------|--------------|---------------|----------|
| **Notebooks** | Studio (JupyterLab) | Workbench (JupyterLab) | Compute Instances |
| **AutoML** | Autopilot | AutoML (Vision, Tabular, NLP) | AutoML |
| **Custom Training** | Training Jobs | Custom Jobs | Command Jobs |
| **Distributed Training** | Built-in (data/model parallel) | Built-in | Built-in |
| **GPU/TPU** | GPU (A100, V100, T4) | GPU + TPU (unique!) | GPU (A100, V100, T4) |
| **Model Registry** | Model Registry | Model Registry | Model Registry + MLflow |
| **Serving** | Endpoints + Serverless | Endpoints | Managed Online Endpoints |
| **Batch Prediction** | Batch Transform | Batch Prediction | Batch Endpoints |
| **Pipelines** | SageMaker Pipelines | Vertex Pipelines (KFP) | Azure ML Pipelines |
| **Feature Store** | Feature Store | Feature Store | Managed Feature Store |
| **Monitoring** | Model Monitor | Model Monitoring | Data Drift Monitor |
| **Explainability** | Clarify | Explainable AI | Responsible AI |
| **MLflow Support** | Partial | Yes | Native (first-class) |
| **Low-Code** | Canvas | AutoML Console | Designer (drag-drop) |
| **Edge Deployment** | Neo + IoT Greengrass | Edge Manager | IoT Edge |

### Pricing Comparison (Approximate, 2024-2025)

| Resource | AWS SageMaker | GCP Vertex AI | Azure ML |
|----------|--------------|---------------|----------|
| **Notebook (ml.m5.xlarge)** | ~$0.23/hr | ~$0.19/hr | ~$0.21/hr |
| **Training (ml.p3.2xlarge/equiv)** | ~$3.83/hr | ~$3.06/hr | ~$3.06/hr |
| **Serving (ml.m5.large/equiv)** | ~$0.12/hr | ~$0.10/hr | ~$0.11/hr |
| **Serverless Inference** | Yes ($0.0001/req) | Yes | Yes |
| **Spot/Preemptible** | Up to 90% off | Up to 80% off | Up to 80% off |
| **Free Tier** | Limited | $300 credit | $200 credit |

> **Important:** Prices vary by region and change frequently. Always check the latest pricing pages.

### Ecosystem & Integration

| Aspect | AWS | GCP | Azure |
|--------|-----|-----|-------|
| **Data Warehouse** | Redshift | BigQuery (excellent!) | Synapse |
| **Stream Processing** | Kinesis | Pub/Sub + Dataflow | Event Hubs + Stream Analytics |
| **Container Orchestration** | EKS | GKE | AKS |
| **CI/CD** | CodePipeline | Cloud Build | Azure DevOps |
| **Identity** | IAM | IAM | Azure AD (enterprise-friendly) |
| **Marketplace** | AWS Marketplace | GCP Marketplace | Azure Marketplace |
| **Enterprise Integration** | Good | Growing | Excellent (Office 365, Teams) |

---

## How to Choose

### Decision Matrix

```
Choose AWS SageMaker when:
├── Already using AWS ecosystem
├── Need widest variety of instance types
├── Want serverless inference
├── Need SageMaker-specific features (Clarify, JumpStart)
└── Team has AWS expertise

Choose GCP Vertex AI when:
├── Need TPUs for large model training
├── Heavy BigQuery usage (BigQuery ML is amazing)
├── Working with Google's pre-trained models
├── Want best price-performance for training
└── Prefer Kubeflow Pipelines (KFP)

Choose Azure ML when:
├── Enterprise with Microsoft ecosystem (Office 365, Teams, AD)
├── Need Responsible AI features (fairness, interpretability)
├── Want native MLflow integration
├── Prefer drag-and-drop Designer for quick prototyping
├── Need strong compliance/governance (Azure Policy)
└── Using Azure DevOps for CI/CD
```

### Migration Strategy

```python
# Multi-cloud abstraction pattern using MLflow
# This code works on ANY platform

import mlflow
import mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier

# Set tracking URI based on environment
import os
PLATFORM = os.environ.get("ML_PLATFORM", "local")

if PLATFORM == "aws":
    mlflow.set_tracking_uri("databricks://aws-workspace")
elif PLATFORM == "gcp":
    mlflow.set_tracking_uri("databricks://gcp-workspace")
elif PLATFORM == "azure":
    mlflow.set_tracking_uri("azureml://your-workspace")
else:
    mlflow.set_tracking_uri("mlruns")  # Local

# Platform-agnostic training code
with mlflow.start_run():
    model = GradientBoostingClassifier(n_estimators=200)
    model.fit(X_train, y_train)
    
    accuracy = accuracy_score(y_test, model.predict(X_test))
    
    mlflow.log_metric("accuracy", accuracy)
    mlflow.sklearn.log_model(model, "model")
    
    # This model can be deployed on ANY platform
    print(f"Model URI: {mlflow.active_run().info.artifact_uri}")
```

---

## Common Mistakes

### 1. Not Accounting for Cold Start Latency
```python
# ❌ WRONG: Deploy with default settings, then wonder why first request takes 30s
endpoint = model.deploy(instance_type="ml.m5.xlarge")

# ✅ RIGHT: Understand cold start implications
# Option A: Keep minimum instances warm
endpoint = model.deploy(
    instance_type="ml.m5.xlarge",
    min_instances=1  # Always keep 1 instance warm
)

# Option B: Use serverless with provisioned concurrency
# (AWS Lambda / SageMaker Serverless)
```

### 2. Ignoring Data Transfer Costs
```python
# ❌ WRONG: Training data in US-East, training in EU-West
# Cross-region data transfer = $0.02/GB * 100GB = $2 per training run

# ✅ RIGHT: Co-locate data and compute
# Keep training data in the SAME region as your compute
training_job = command(
    compute="us-east-cluster",  # Same region as data
    inputs={"data": "s3://us-east-bucket/data/"}  # Same region!
)
```

### 3. Not Setting Cost Alerts
```python
# ✅ ALWAYS set up cost alerts
# AWS example with boto3
import boto3

cloudwatch = boto3.client('cloudwatch')
cloudwatch.put_metric_alarm(
    AlarmName='SageMaker-Monthly-Spend',
    MetricName='EstimatedCharges',
    Namespace='AWS/Billing',
    Statistic='Maximum',
    Period=86400,           # Check daily
    Threshold=500.0,       # Alert at $500
    ComparisonOperator='GreaterThanThreshold',
    AlarmActions=['arn:aws:sns:us-east-1:123456:billing-alert']
)
```

### 4. Vendor Lock-in Without Abstraction
```python
# ❌ WRONG: Deeply coupled to one platform
from sagemaker.sklearn import SKLearn  # Hard to migrate away

# ✅ RIGHT: Use abstraction layers
# Option 1: MLflow for experiment tracking + model packaging
# Option 2: Docker containers (portable across all clouds)
# Option 3: ONNX model format (platform-agnostic serving)
# Option 4: Kubeflow (runs on any Kubernetes)
```

### 5. Over-Provisioning Compute
```python
# ❌ WRONG: Using ml.p3.8xlarge for a logistic regression
estimator = SKLearn(instance_type='ml.p3.8xlarge')  # $14/hr for sklearn!

# ✅ RIGHT: Match compute to workload
# Sklearn models: ml.m5.large ($0.12/hr) is usually enough
# Deep learning small: ml.g4dn.xlarge ($0.53/hr)
# Deep learning large: ml.p3.2xlarge ($3.83/hr)
# LLM fine-tuning: ml.p4d.24xlarge ($32/hr)
```

---

## Interview Questions

### Conceptual

**Q1: Compare AWS SageMaker, GCP Vertex AI, and Azure ML. When would you choose each?**
> **A:** SageMaker for AWS-native teams needing variety of instance types and serverless inference. Vertex AI for teams using BigQuery, needing TPUs, or wanting Kubeflow pipelines natively. Azure ML for enterprises with Microsoft stack (AD, DevOps, Office 365) needing strong governance and Responsible AI. Key differentiators: GCP has TPUs and BigQuery ML, Azure has Designer and Responsible AI dashboard, AWS has broadest marketplace and instance selection.

**Q2: How would you handle model deployment across multiple cloud providers?**
> **A:** Use abstraction layers: (1) MLflow for experiment tracking and model packaging (works everywhere), (2) Docker containers for portable serving (any K8s cluster), (3) ONNX format for model interoperability, (4) Terraform/Pulumi for infrastructure-as-code across clouds, (5) Kubeflow Pipelines for portable ML workflows. Keep core training code platform-agnostic; only deployment configs are platform-specific.

**Q3: What's the total cost of running an ML model in production on cloud platforms?**
> **A:** Total cost includes: (1) Compute for training (GPU hours × iterations), (2) Serving infrastructure (instances × uptime), (3) Storage (model artifacts, data, logs), (4) Data transfer (cross-region, ingress/egress), (5) Monitoring and logging, (6) Feature store (if used), (7) Pipeline orchestration. Often serving cost dominates because it's 24/7, while training is periodic. Optimize with: spot instances for training, autoscaling for serving, serverless for low-traffic endpoints.

**Q4: How do you ensure reproducibility when training on cloud platforms?**
> **A:** (1) Pin all library versions in requirements.txt/conda.yml, (2) Use fixed Docker images (not :latest tags), (3) Version training data with DVC or platform data versioning, (4) Set random seeds in code, (5) Log all hyperparameters and configs to experiment tracker, (6) Use deterministic GPU operations where possible, (7) Tag compute instance types and configurations, (8) Store git commit SHA with each training run.

### Scenario-Based

**Q5: Your SageMaker endpoint costs $2000/month but only gets 100 requests/day. How do you optimize?**
> **A:** Switch to SageMaker Serverless Inference or Async Inference. For 100 requests/day, serverless would cost ~$0.01/day ($0.30/month) vs $2000/month. Alternatively, use Lambda + S3 model for simple sklearn models. Consider batch processing if real-time isn't truly needed. Key: match infrastructure to actual traffic patterns.

**Q6: You need to train a model on 500GB of data. Walk through your cloud architecture.**
> **A:** (1) Store data in native storage (S3/GCS/Blob) in same region as compute, (2) Use distributed training across multiple instances if single-instance is too slow, (3) Use SageMaker Data Parallelism / Vertex distributed training, (4) Consider data sampling for experimentation, full data for final training, (5) Use spot/preemptible instances with checkpointing to save costs, (6) Set up proper data pipeline (not copying data for each run — use mounting/streaming), (7) Monitor training with real-time metrics to detect early failures.

---

## Quick Reference

### One-Line Deployment Cheat Sheet

| Platform | Deploy Command |
|----------|---------------|
| **SageMaker** | `estimator.deploy(instance_type='ml.m5.large', initial_instance_count=1)` |
| **Vertex AI** | `model.deploy(endpoint=endpoint, machine_type='n1-standard-2')` |
| **Azure ML** | `ml_client.online_deployments.begin_create_or_update(deployment)` |

### Instance Type Quick Reference

| Workload | AWS | GCP | Azure |
|----------|-----|-----|-------|
| **Light (sklearn)** | ml.m5.large | n1-standard-2 | Standard_DS2_v2 |
| **Medium (small DL)** | ml.g4dn.xlarge | n1-standard-4 + T4 | Standard_NC4as_T4 |
| **Heavy (large DL)** | ml.p3.2xlarge | a2-highgpu-1g | Standard_NC6s_v3 |
| **LLM Training** | ml.p4d.24xlarge | a2-megagpu-16g | Standard_ND96amsr_A100 |

### Cost Optimization Strategies

| Strategy | Savings | Applies To |
|----------|---------|-----------|
| Spot/Preemptible instances | 60-90% | Training |
| Serverless inference | 95%+ (low traffic) | Serving |
| Autoscaling | 30-50% | Serving |
| Reserved instances | 30-60% | Steady-state serving |
| Right-sizing | 20-50% | All |
| Multi-region optimization | 10-20% | Data transfer |
| Scheduled scaling | 30-40% | Predictable traffic |

---

> **Pro Tip:** Start with the simplest (cheapest) option that works. You can always scale up. A SageMaker Serverless endpoint + GitHub Actions CI/CD is a production-ready setup that costs nearly nothing at low scale.
