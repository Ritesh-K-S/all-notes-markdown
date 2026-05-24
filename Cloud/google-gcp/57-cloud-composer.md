# Chapter 57 — Cloud Composer (Airflow)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Composer Fundamentals](#part-1--cloud-composer-fundamentals)
- [Part 2: Environment Creation & Configuration](#part-2--environment-creation--configuration)
- [Part 3: DAGs — Directed Acyclic Graphs](#part-3--dags--directed-acyclic-graphs)
- [Part 4: Operators](#part-4--operators)
- [Part 5: GCP-Specific Operators](#part-5--gcp-specific-operators)
- [Part 6: Sensors & Triggers](#part-6--sensors--triggers)
- [Part 7: Variables, Connections & Secrets](#part-7--variables-connections--secrets)
- [Part 8: XComs & Task Communication](#part-8--xcoms--task-communication)
- [Part 9: Branching & Conditional Logic](#part-9--branching--conditional-logic)
- [Part 10: Dynamic DAGs & Task Groups](#part-10--dynamic-dags--task-groups)
- [Part 11: Scheduling & Catchup](#part-11--scheduling--catchup)
- [Part 12: Monitoring & Alerting](#part-12--monitoring--alerting)
- [Part 13: Security & Networking](#part-13--security--networking)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Cloud Composer is Google Cloud's fully managed workflow orchestration service built on Apache Airflow. It lets you author, schedule, and monitor data pipelines that span across clouds and on-premises data centers. Composer manages the Airflow infrastructure (web server, scheduler, workers, database, GCS bucket) so you focus on writing DAGs — the Python files that define your pipeline logic.

---

## Part 1 — Cloud Composer Fundamentals

### Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│         CLOUD COMPOSER ARCHITECTURE                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Cloud Composer Environment                                │     │
│  │                                                            │     │
│  │  ┌──────────────────────────────────────────────┐        │     │
│  │  │ GKE Cluster (managed)                         │        │     │
│  │  │ ┌──────────┐ ┌──────────┐ ┌──────────┐     │        │     │
│  │  │ │ Airflow  │ │ Airflow  │ │ Airflow  │     │        │     │
│  │  │ │ Scheduler│ │ Workers  │ │ Triggerer│     │        │     │
│  │  │ │          │ │ (Celery) │ │ (async)  │     │        │     │
│  │  │ └──────────┘ └──────────┘ └──────────┘     │        │     │
│  │  └──────────────────────────────────────────────┘        │     │
│  │                                                            │     │
│  │  ┌──────────────┐  ┌──────────────┐                      │     │
│  │  │ Airflow      │  │ Cloud SQL    │                      │     │
│  │  │ Web Server   │  │ (metadata DB)│                      │     │
│  │  │ (App Engine) │  │              │                      │     │
│  │  └──────────────┘  └──────────────┘                      │     │
│  │                                                            │     │
│  │  ┌──────────────────────────────────────────────┐        │     │
│  │  │ GCS Bucket (DAGs, plugins, data, logs)        │        │     │
│  │  │ gs://REGION-COMPOSER-ENV-BUCKET/              │        │     │
│  │  │ ├── dags/        ← upload DAG files here     │        │     │
│  │  │ ├── plugins/     ← custom operators/hooks    │        │     │
│  │  │ ├── data/        ← data files for DAGs       │        │     │
│  │  │ └── logs/        ← task execution logs       │        │     │
│  │  └──────────────────────────────────────────────┘        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Composer versions:                                                 │
│  • Composer 1: Airflow 1.x (legacy)                               │
│  • Composer 2: Airflow 2.x (recommended, autoscaling)             │
│  • Composer 3: latest, improved UI and features                   │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Composer | AWS MWAA | Azure Data Factory |
|---------|-------------|---------|-------------------|
| Engine | Apache Airflow | Apache Airflow | Proprietary + Airflow |
| Managed | Yes | Yes | Yes |
| DAG authoring | Python | Python | GUI + Python |
| GKE-based | Yes | ECS-based | VMSS-based |
| Scaling | Auto (Composer 2+) | Auto | Auto |
| Pricing | Per environment + GKE | Per environment | Per activity run |

### Pricing (Composer 2)

| Component | Cost |
|-----------|------|
| Environment (small) | ~$350/month |
| Environment (medium) | ~$600/month |
| Environment (large) | ~$1,200/month |
| Compute (workers) | GKE node pricing |
| Database (metadata) | Cloud SQL pricing |
| Storage (DAGs, logs) | GCS pricing |

---

## Part 2 — Environment Creation & Configuration

### Create an Environment

```bash
# Create Composer 2 environment
gcloud composer environments create my-composer \
    --location=us-central1 \
    --image-version=composer-2.9.7-airflow-2.9.3 \
    --environment-size=small \
    --service-account=composer-sa@my-project.iam.gserviceaccount.com

# With networking
gcloud composer environments create my-composer \
    --location=us-central1 \
    --image-version=composer-2.9.7-airflow-2.9.3 \
    --network=my-vpc \
    --subnetwork=composer-subnet \
    --enable-private-environment \
    --enable-privately-used-public-ips

# Get environment info
gcloud composer environments describe my-composer \
    --location=us-central1

# Get DAGs bucket
gcloud composer environments describe my-composer \
    --location=us-central1 \
    --format="get(config.dagGcsPrefix)"
```

### Install Python Packages

```bash
# Install PyPI packages
gcloud composer environments update my-composer \
    --location=us-central1 \
    --update-pypi-packages-from-file=requirements.txt

# Or individual packages
gcloud composer environments update my-composer \
    --location=us-central1 \
    --update-pypi-package="pandas>=1.5.0"

# Set Airflow configuration
gcloud composer environments update my-composer \
    --location=us-central1 \
    --update-airflow-configs=\
core-dags_are_paused_at_creation=true,\
webserver-dag_default_view=graph

# Set environment variables
gcloud composer environments update my-composer \
    --location=us-central1 \
    --update-env-variables=\
PROJECT_ID=my-project,\
ENVIRONMENT=production
```

---

## Part 3 — DAGs — Directed Acyclic Graphs

### Basic DAG Structure

```python
# dags/hello_world.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator

# DAG default arguments
default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'email': ['alerts@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'start_date': datetime(2024, 1, 1),
}

# DAG definition
with DAG(
    dag_id='hello_world',
    default_args=default_args,
    description='A simple hello world DAG',
    schedule='0 6 * * *',              # daily at 6 AM
    catchup=False,                      # don't backfill
    tags=['example'],
    max_active_runs=1,
) as dag:

    def greet(**kwargs):
        execution_date = kwargs['ds']
        print(f"Hello! Running for date: {execution_date}")

    start = BashOperator(
        task_id='start',
        bash_command='echo "Starting pipeline"',
    )

    process = PythonOperator(
        task_id='process',
        python_callable=greet,
    )

    end = BashOperator(
        task_id='end',
        bash_command='echo "Pipeline complete"',
    )

    # Task dependencies
    start >> process >> end
```

### Deploy DAGs

```bash
# Upload DAG to Composer bucket
BUCKET=$(gcloud composer environments describe my-composer \
    --location=us-central1 \
    --format="get(config.dagGcsPrefix)")

gsutil cp dags/hello_world.py $BUCKET/

# Sync all DAGs
gsutil -m rsync -r dags/ $BUCKET/
```

---

## Part 4 — Operators

### Common Operators

```python
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.bash import BashOperator
from airflow.operators.empty import EmptyOperator
from airflow.operators.email import EmailOperator
from airflow.utils.trigger_rule import TriggerRule

# PythonOperator — run Python function
process = PythonOperator(
    task_id='process_data',
    python_callable=my_function,
    op_args=['arg1', 'arg2'],
    op_kwargs={'key': 'value'},
)

# BashOperator — run shell command
extract = BashOperator(
    task_id='extract_data',
    bash_command='gsutil cp gs://bucket/data.csv /tmp/data.csv',
)

# EmptyOperator — placeholder/join point
join = EmptyOperator(
    task_id='join',
    trigger_rule=TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS,
)

# EmailOperator — send email
notify = EmailOperator(
    task_id='send_report',
    to='team@company.com',
    subject='Pipeline {{ ds }} Complete',
    html_content='<p>Report for {{ ds }} is ready.</p>',
)
```

---

## Part 5 — GCP-Specific Operators

### Google Cloud Operators

```python
# BigQuery
from airflow.providers.google.cloud.operators.bigquery import (
    BigQueryInsertJobOperator,
    BigQueryCheckOperator,
)

run_query = BigQueryInsertJobOperator(
    task_id='run_bq_query',
    configuration={
        'query': {
            'query': """
                SELECT DATE(created_at) AS date, COUNT(*) AS orders
                FROM `my-project.sales.orders`
                WHERE DATE(created_at) = '{{ ds }}'
                GROUP BY date
            """,
            'destinationTable': {
                'projectId': 'my-project',
                'datasetId': 'analytics',
                'tableId': 'daily_orders',
            },
            'writeDisposition': 'WRITE_TRUNCATE',
            'useLegacySql': False,
        }
    },
)

check_data = BigQueryCheckOperator(
    task_id='check_data_quality',
    sql="""
        SELECT COUNT(*) FROM `my-project.analytics.daily_orders`
        WHERE date = '{{ ds }}'
    """,
    use_legacy_sql=False,
)

# Dataproc
from airflow.providers.google.cloud.operators.dataproc import (
    DataprocCreateClusterOperator,
    DataprocSubmitPySparkJobOperator,
    DataprocDeleteClusterOperator,
)

create_cluster = DataprocCreateClusterOperator(
    task_id='create_cluster',
    project_id='my-project',
    cluster_name='ephemeral-{{ ds_nodash }}',
    region='us-central1',
    cluster_config={
        'master_config': {'num_instances': 1, 'machine_type_uri': 'n2-standard-4'},
        'worker_config': {'num_instances': 3, 'machine_type_uri': 'n2-standard-8'},
    },
)

submit_job = DataprocSubmitPySparkJobOperator(
    task_id='run_spark_etl',
    main='gs://scripts-bucket/etl.py',
    cluster_name='ephemeral-{{ ds_nodash }}',
    region='us-central1',
    arguments=['--date={{ ds }}'],
)

delete_cluster = DataprocDeleteClusterOperator(
    task_id='delete_cluster',
    project_id='my-project',
    cluster_name='ephemeral-{{ ds_nodash }}',
    region='us-central1',
    trigger_rule=TriggerRule.ALL_DONE,    # always delete
)

# GCS
from airflow.providers.google.cloud.operators.gcs import (
    GCSCreateBucketOperator,
    GCSDeleteObjectsOperator,
)
from airflow.providers.google.cloud.transfers.gcs_to_bigquery import (
    GCSToBigQueryOperator,
)

load_to_bq = GCSToBigQueryOperator(
    task_id='load_csv_to_bq',
    bucket='data-bucket',
    source_objects=['exports/{{ ds }}/*.csv'],
    destination_project_dataset_table='my-project.raw.imports',
    source_format='CSV',
    skip_leading_rows=1,
    autodetect=True,
    write_disposition='WRITE_TRUNCATE',
)

# Dataflow
from airflow.providers.google.cloud.operators.dataflow import (
    DataflowTemplatedJobStartOperator,
)

run_dataflow = DataflowTemplatedJobStartOperator(
    task_id='run_dataflow',
    template='gs://dataflow-templates/latest/GCS_Text_to_BigQuery',
    parameters={
        'inputFilePattern': 'gs://data-bucket/input/{{ ds }}/*.csv',
        'outputTable': 'my-project:dataset.table',
        'JSONPath': 'gs://data-bucket/schema.json',
        'bigQueryLoadingTemporaryDirectory': 'gs://temp-bucket/bq/',
    },
    location='us-central1',
)
```

---

## Part 6 — Sensors & Triggers

### Waiting for External Events

```python
from airflow.providers.google.cloud.sensors.gcs import (
    GCSObjectExistenceSensor,
    GCSObjectsWithPrefixExistenceSensor,
)
from airflow.providers.google.cloud.sensors.bigquery import (
    BigQueryTableExistenceSensor,
)

# Wait for file in GCS
wait_for_file = GCSObjectExistenceSensor(
    task_id='wait_for_file',
    bucket='data-bucket',
    object='exports/{{ ds }}/complete.flag',
    timeout=3600,                      # 1 hour timeout
    poke_interval=60,                  # check every 60 seconds
    mode='reschedule',                 # free up worker while waiting
)

# Wait for files with prefix
wait_for_data = GCSObjectsWithPrefixExistenceSensor(
    task_id='wait_for_data',
    bucket='data-bucket',
    prefix='exports/{{ ds }}/',
    timeout=7200,
    mode='reschedule',
)

# Wait for BigQuery table partition
wait_for_partition = BigQueryTableExistenceSensor(
    task_id='wait_for_partition',
    project_id='upstream-project',
    dataset_id='analytics',
    table_id='events${{ ds_nodash }}',
    timeout=7200,
    mode='reschedule',
)

# Sensor modes:
# 'poke'       — worker keeps checking (holds slot)
# 'reschedule' — frees worker between checks (recommended)
```

### Deferrable Operators (Async)

```python
from airflow.providers.google.cloud.operators.bigquery import (
    BigQueryInsertJobOperator,
)

# Deferrable: pauses task, fires trigger, resumes when done
# Frees up worker while waiting for long-running BQ query
async_query = BigQueryInsertJobOperator(
    task_id='async_bq_query',
    configuration={'query': {'query': 'SELECT ...', 'useLegacySql': False}},
    deferrable=True,                   # uses Triggerer (async)
)
```

---

## Part 7 — Variables, Connections & Secrets

### Variables

```python
from airflow.models import Variable

# Set via CLI
# gcloud composer environments run my-composer --location=us-central1 \
#     variables -- set project_id my-project

# Use in DAG
project_id = Variable.get("project_id")
config = Variable.get("pipeline_config", deserialize_json=True)

# Use in templates (Jinja)
task = BashOperator(
    task_id='use_var',
    bash_command='echo {{ var.value.project_id }}',
)
```

### Connections

```bash
# Add connection via CLI
gcloud composer environments run my-composer --location=us-central1 \
    connections -- add 'my_postgres' \
    --conn-type 'postgres' \
    --conn-host 'db.example.com' \
    --conn-port '5432' \
    --conn-login 'user' \
    --conn-password 'pass' \
    --conn-schema 'mydb'
```

### Secret Backend (Secret Manager)

```bash
# Configure Secret Manager as backend
gcloud composer environments update my-composer \
    --location=us-central1 \
    --update-airflow-configs=\
secrets-backend=airflow.providers.google.cloud.secrets.secret_manager.CloudSecretManagerBackend,\
secrets-backend_kwargs='{"project_id": "my-project", "connections_prefix": "airflow-connections", "variables_prefix": "airflow-variables"}'

# Store a variable in Secret Manager
echo -n "my-value" | gcloud secrets create airflow-variables-my_var \
    --data-file=- --project=my-project

# Airflow auto-reads from Secret Manager
# Variable.get("my_var") → reads from secret "airflow-variables-my_var"
```

---

## Part 8 — XComs & Task Communication

### Passing Data Between Tasks

```python
# Push XCom
def extract(**kwargs):
    data = {'records': 1500, 'file': 'gs://bucket/output.csv'}
    return data  # auto-pushed as XCom with key='return_value'

# Pull XCom
def transform(**kwargs):
    ti = kwargs['ti']
    data = ti.xcom_pull(task_ids='extract_task')
    print(f"Processing {data['records']} records from {data['file']}")

extract_task = PythonOperator(
    task_id='extract_task',
    python_callable=extract,
)

transform_task = PythonOperator(
    task_id='transform_task',
    python_callable=transform,
)

extract_task >> transform_task

# Use XCom in templates
notify = BashOperator(
    task_id='notify',
    bash_command='echo "Processed {{ ti.xcom_pull(task_ids="extract_task")["records"] }} records"',
)
```

---

## Part 9 — Branching & Conditional Logic

### Branch Operator

```python
from airflow.operators.python import BranchPythonOperator

def choose_path(**kwargs):
    ds = kwargs['ds']
    day = datetime.strptime(ds, '%Y-%m-%d').weekday()
    if day < 5:  # weekday
        return 'weekday_processing'
    else:
        return 'weekend_processing'

branch = BranchPythonOperator(
    task_id='branch',
    python_callable=choose_path,
)

weekday = PythonOperator(
    task_id='weekday_processing',
    python_callable=process_weekday,
)

weekend = PythonOperator(
    task_id='weekend_processing',
    python_callable=process_weekend,
)

join = EmptyOperator(
    task_id='join',
    trigger_rule=TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS,
)

branch >> [weekday, weekend] >> join
```

### Short-Circuit Operator

```python
from airflow.operators.python import ShortCircuitOperator

# Skip all downstream if condition is False
check = ShortCircuitOperator(
    task_id='check_if_data_exists',
    python_callable=lambda: check_gcs_file_exists('gs://bucket/data.csv'),
)

check >> process >> notify
# If check returns False, process and notify are skipped
```

---

## Part 10 — Dynamic DAGs & Task Groups

### TaskGroups

```python
from airflow.utils.task_group import TaskGroup

with DAG('etl_pipeline', ...) as dag:

    with TaskGroup('extract', tooltip='Extract data') as extract:
        extract_orders = PythonOperator(task_id='orders', ...)
        extract_users = PythonOperator(task_id='users', ...)
        extract_products = PythonOperator(task_id='products', ...)

    with TaskGroup('transform', tooltip='Transform data') as transform:
        clean = PythonOperator(task_id='clean', ...)
        enrich = PythonOperator(task_id='enrich', ...)
        clean >> enrich

    with TaskGroup('load', tooltip='Load to warehouse') as load:
        load_bq = BigQueryInsertJobOperator(task_id='bigquery', ...)
        load_gcs = BashOperator(task_id='gcs_archive', ...)

    extract >> transform >> load
```

### Dynamic Task Mapping (Airflow 2.3+)

```python
# Generate tasks dynamically based on data
@dag(schedule='@daily', start_date=datetime(2024, 1, 1), catchup=False)
def dynamic_etl():

    @task
    def get_regions():
        return ['us-east1', 'eu-west1', 'ap-south1']

    @task
    def process_region(region: str):
        print(f"Processing data for region: {region}")
        # region-specific ETL logic...

    @task
    def summarize(results):
        print(f"Processed {len(results)} regions")

    regions = get_regions()
    results = process_region.expand(region=regions)  # dynamic fan-out
    summarize(results)

dynamic_etl()
```

---

## Part 11 — Scheduling & Catchup

### Schedule Expressions

| Expression | Description |
|-----------|------------|
| `@once` | Run once |
| `@hourly` | Every hour |
| `@daily` | Every day at midnight |
| `@weekly` | Every Sunday at midnight |
| `@monthly` | First day of month |
| `@yearly` | January 1st |
| `0 6 * * *` | Daily at 6:00 AM |
| `0 */2 * * *` | Every 2 hours |
| `0 8 * * 1-5` | Weekdays at 8:00 AM |
| `None` | Manual trigger only |

### Catchup & Backfill

```python
# catchup=True: run for all missed intervals since start_date
# catchup=False: only run for the latest interval
with DAG(
    'my_dag',
    start_date=datetime(2024, 1, 1),
    schedule='@daily',
    catchup=False,              # recommended for most DAGs
    max_active_runs=1,          # prevent parallel backfills
) as dag:
    ...

# Manual backfill via CLI
# gcloud composer environments run my-composer --location=us-central1 \
#     dags -- backfill my_dag -s 2024-01-01 -e 2024-01-31
```

---

## Part 12 — Monitoring & Alerting

### Callbacks & SLAs

```python
def task_failure_callback(context):
    """Called when a task fails."""
    dag_id = context['dag'].dag_id
    task_id = context['task_instance'].task_id
    execution_date = context['ds']
    # Send Slack/PagerDuty alert
    send_alert(f"FAILED: {dag_id}.{task_id} on {execution_date}")

def dag_success_callback(context):
    """Called when entire DAG succeeds."""
    send_notification(f"DAG {context['dag'].dag_id} completed successfully")

with DAG(
    'monitored_pipeline',
    default_args={
        'on_failure_callback': task_failure_callback,
        'retries': 2,
        'retry_delay': timedelta(minutes=5),
        'execution_timeout': timedelta(hours=2),     # kill if too long
    },
    on_success_callback=dag_success_callback,
    dagrun_timeout=timedelta(hours=6),                # entire DAG timeout
    ...
) as dag:
    ...
```

### Cloud Monitoring Integration

```bash
# Key Composer metrics:
# composer.googleapis.com/environment/dagbag_size
# composer.googleapis.com/environment/dag_processing/total_parse_time
# composer.googleapis.com/environment/scheduler_heartbeat
# composer.googleapis.com/environment/worker/pod_eviction_count
# composer.googleapis.com/environment/database_health
# composer.googleapis.com/environment/task/scheduler_delay
```

---

## Part 13 — Security & Networking

### Private Environment

```bash
# Create private Composer environment
gcloud composer environments create private-composer \
    --location=us-central1 \
    --image-version=composer-2.9.7-airflow-2.9.3 \
    --enable-private-environment \
    --enable-privately-used-public-ips \
    --network=my-vpc \
    --subnetwork=composer-subnet \
    --web-server-allow-ip=10.0.0.0/8,description='Internal' \
    --service-account=composer-sa@my-project.iam.gserviceaccount.com
```

### Required IAM Roles

| Role | Purpose |
|------|---------|
| `roles/composer.admin` | Full Composer management |
| `roles/composer.environmentAndStorageObjectAdmin` | Manage environments + DAGs |
| `roles/composer.user` | Trigger DAGs, view environment |
| `roles/composer.worker` | Worker SA (task execution) |

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Service Account ─────────────────────────────────────────
resource "google_service_account" "composer" {
  account_id   = "composer-sa"
  display_name = "Cloud Composer SA"
  project      = var.project_id
}

resource "google_project_iam_member" "composer_worker" {
  project = var.project_id
  role    = "roles/composer.worker"
  member  = "serviceAccount:${google_service_account.composer.email}"
}

resource "google_project_iam_member" "composer_bq" {
  project = var.project_id
  role    = "roles/bigquery.admin"
  member  = "serviceAccount:${google_service_account.composer.email}"
}

resource "google_project_iam_member" "composer_dataproc" {
  project = var.project_id
  role    = "roles/dataproc.admin"
  member  = "serviceAccount:${google_service_account.composer.email}"
}

# ─── Composer Environment ────────────────────────────────────
resource "google_composer_environment" "production" {
  name    = "production-composer"
  region  = var.region
  project = var.project_id

  config {
    software_config {
      image_version = "composer-2.9.7-airflow-2.9.3"

      airflow_config_overrides = {
        core-dags_are_paused_at_creation = "true"
        webserver-dag_default_view       = "graph"
        secrets-backend = "airflow.providers.google.cloud.secrets.secret_manager.CloudSecretManagerBackend"
      }

      pypi_packages = {
        pandas         = ">=2.0.0"
        requests       = ">=2.31.0"
        slack-sdk      = ">=3.0"
      }

      env_variables = {
        PROJECT_ID  = var.project_id
        ENVIRONMENT = "production"
      }
    }

    workloads_config {
      scheduler {
        cpu        = 2
        memory_gb  = 4
        storage_gb = 5
        count      = 2
      }
      web_server {
        cpu        = 2
        memory_gb  = 4
        storage_gb = 5
      }
      worker {
        cpu        = 2
        memory_gb  = 8
        storage_gb = 10
        min_count  = 1
        max_count  = 6
      }
      triggerer {
        cpu       = 1
        memory_gb = 1
        count     = 1
      }
    }

    environment_size = "ENVIRONMENT_SIZE_MEDIUM"

    node_config {
      service_account = google_service_account.composer.email
      network         = google_compute_network.vpc.id
      subnetwork      = google_compute_subnetwork.composer.id
    }

    private_environment_config {
      enable_private_endpoint             = false
      enable_privately_used_public_ips    = true
      cloud_composer_network_ipv4_cidr_block = "172.31.245.0/24"
    }

    maintenance_window {
      start_time = "2024-01-01T02:00:00Z"
      end_time   = "2024-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA"
    }
  }
}

# Output DAGs bucket
output "dags_bucket" {
  value = google_composer_environment.production.config[0].dag_gcs_prefix
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# ENVIRONMENTS
# ═══════════════════════════════════════════════════════════════
gcloud composer environments create NAME --location=L [opts]
gcloud composer environments update NAME --location=L [opts]
gcloud composer environments describe NAME --location=L
gcloud composer environments list --locations=L
gcloud composer environments delete NAME --location=L

# ═══════════════════════════════════════════════════════════════
# DAG MANAGEMENT
# ═══════════════════════════════════════════════════════════════
# Upload DAG
gsutil cp dag.py $(gcloud composer environments describe ENV \
    --location=L --format="get(config.dagGcsPrefix)")/

# List DAGs
gcloud composer environments run ENV --location=L dags -- list

# Trigger DAG
gcloud composer environments run ENV --location=L \
    dags -- trigger DAG_ID

# Pause/unpause DAG
gcloud composer environments run ENV --location=L \
    dags -- pause DAG_ID
gcloud composer environments run ENV --location=L \
    dags -- unpause DAG_ID

# ═══════════════════════════════════════════════════════════════
# VARIABLES & CONNECTIONS
# ═══════════════════════════════════════════════════════════════
gcloud composer environments run ENV --location=L \
    variables -- set KEY VALUE

gcloud composer environments run ENV --location=L \
    variables -- get KEY

gcloud composer environments run ENV --location=L \
    connections -- add CONN_ID --conn-type TYPE [opts]

# ═══════════════════════════════════════════════════════════════
# CONFIGURATION
# ═══════════════════════════════════════════════════════════════
gcloud composer environments update ENV --location=L \
    --update-pypi-packages-from-file=requirements.txt

gcloud composer environments update ENV --location=L \
    --update-airflow-configs=KEY=VALUE
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Daily Data Warehouse ETL

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: DATA WAREHOUSE ETL                                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  DAG: daily_dwh_etl (schedule: @daily, 6 AM)             │        │
│  │                                                            │        │
│  │  ┌─────────────────────────────────────────────┐         │        │
│  │  │ TaskGroup: extract                           │         │        │
│  │  │ ├── sensor: wait_for_upstream_data (GCS)    │         │        │
│  │  │ ├── gcs_to_bq: load_orders                 │         │        │
│  │  │ ├── gcs_to_bq: load_customers              │         │        │
│  │  │ └── gcs_to_bq: load_products               │         │        │
│  │  └──────────────────┬──────────────────────────┘         │        │
│  │                     ▼                                      │        │
│  │  ┌─────────────────────────────────────────────┐         │        │
│  │  │ TaskGroup: transform                         │         │        │
│  │  │ ├── bq_query: clean_and_deduplicate         │         │        │
│  │  │ ├── bq_query: build_dim_tables              │         │        │
│  │  │ └── bq_query: build_fact_tables             │         │        │
│  │  └──────────────────┬──────────────────────────┘         │        │
│  │                     ▼                                      │        │
│  │  ┌─────────────────────────────────────────────┐         │        │
│  │  │ TaskGroup: quality_checks                    │         │        │
│  │  │ ├── bq_check: row_count > 0                │         │        │
│  │  │ ├── bq_check: no_null_keys                 │         │        │
│  │  │ └── bq_check: referential_integrity         │         │        │
│  │  └──────────────────┬──────────────────────────┘         │        │
│  │                     ▼                                      │        │
│  │  ┌─────────────────────────────────────────────┐         │        │
│  │  │ notify: send_slack_success                   │         │        │
│  │  └─────────────────────────────────────────────┘         │        │
│  │                                                            │        │
│  │  on_failure_callback → PagerDuty alert                   │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Ephemeral Dataproc Orchestration

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: DATAPROC ORCHESTRATION                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  DAG: spark_ml_pipeline (schedule: @weekly)               │        │
│  │                                                            │        │
│  │  1. create_cluster                                        │        │
│  │     → DataprocCreateClusterOperator                      │        │
│  │     → 2 primary + 10 Spot secondary workers              │        │
│  │                                                            │        │
│  │  2. feature_engineering                                   │        │
│  │     → DataprocSubmitPySparkJobOperator                   │        │
│  │     → Read from BigQuery, write features to GCS          │        │
│  │                                                            │        │
│  │  3. train_model                                           │        │
│  │     → DataprocSubmitPySparkJobOperator                   │        │
│  │     → Spark MLlib training, save model to GCS            │        │
│  │                                                            │        │
│  │  4. evaluate_model                                        │        │
│  │     → PythonOperator                                     │        │
│  │     → Load model, compute metrics, log to Vertex AI     │        │
│  │                                                            │        │
│  │  5. deploy_if_better (BranchPythonOperator)              │        │
│  │     ├── deploy_model → Vertex AI endpoint               │        │
│  │     └── skip_deploy → log "model not improved"          │        │
│  │                                                            │        │
│  │  6. delete_cluster (trigger_rule: ALL_DONE)              │        │
│  │     → Always runs, even if jobs fail                     │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Multi-System Data Sync

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: CROSS-SYSTEM SYNC                                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  DAG: cross_system_sync (schedule: every 2 hours)         │        │
│  │                                                            │        │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │        │
│  │  │ Extract │  │ Extract │  │ Extract │                 │        │
│  │  │ CRM API │  │ Cloud   │  │ On-Prem │                 │        │
│  │  │ (REST)  │  │ SQL     │  │ DB      │                 │        │
│  │  └────┬────┘  └────┬────┘  └────┬────┘                 │        │
│  │       └─────────────┼───────────┘                        │        │
│  │                     ▼                                      │        │
│  │              ┌──────────────┐                             │        │
│  │              │ Stage in GCS │                             │        │
│  │              │ (JSON/CSV)   │                             │        │
│  │              └──────┬───────┘                             │        │
│  │                     ▼                                      │        │
│  │              ┌──────────────┐                             │        │
│  │              │ Load to      │                             │        │
│  │              │ BigQuery     │                             │        │
│  │              │ (staging)    │                             │        │
│  │              └──────┬───────┘                             │        │
│  │                     ▼                                      │        │
│  │              ┌──────────────┐                             │        │
│  │              │ MERGE into   │                             │        │
│  │              │ production   │                             │        │
│  │              │ tables       │                             │        │
│  │              └──────┬───────┘                             │        │
│  │                     ▼                                      │        │
│  │              ┌──────────────┐                             │        │
│  │              │ Notify +     │                             │        │
│  │              │ audit log    │                             │        │
│  │              └──────────────┘                             │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create environment | `gcloud composer environments create E --location=L` |
| Upload DAG | `gsutil cp dag.py $DAGS_BUCKET/` |
| Trigger DAG | `gcloud composer environments run E --location=L dags -- trigger DAG` |
| List DAGs | `gcloud composer environments run E --location=L dags -- list` |
| Pause DAG | `gcloud composer environments run E --location=L dags -- pause DAG` |
| Set variable | `gcloud composer environments run E --location=L variables -- set K V` |
| Install packages | `--update-pypi-packages-from-file=requirements.txt` |
| Get DAGs bucket | `--format="get(config.dagGcsPrefix)"` |
| Sensor mode | `mode='reschedule'` (frees worker while waiting) |
| Async operator | `deferrable=True` (uses Triggerer) |

---

## What is Workflow Orchestration? (Beginner Explanation)

If you've never worked with workflow orchestration, here's the simplest way to think about it:

```
┌────────────────────────────────────────────────────────────────────┐
│         WORKFLOW ORCHESTRATION — THE BIG IDEA                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Cloud Composer is like a conductor in an orchestra.               │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  An orchestra has many instruments (violin, drums, flute) │     │
│  │  playing different parts. The conductor doesn't play any  │     │
│  │  instrument — instead, they coordinate:                   │     │
│  │                                                            │     │
│  │  • WHEN each instrument plays (scheduling)                │     │
│  │  • In what ORDER (dependencies)                           │     │
│  │  • What to do if someone hits a wrong note (error handling)│    │
│  │  • When to start over or skip ahead (retries, branching)  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  In data pipelines, the "instruments" are:                        │
│  • BigQuery queries                                                │
│  • Dataflow jobs                                                   │
│  • Dataproc Spark jobs                                             │
│  • API calls, file transfers, notifications                       │
│                                                                      │
│  Cloud Composer (the conductor) coordinates all of them:          │
│                                                                      │
│  "First, wait for the data file to arrive."                       │
│  "Then, run the BigQuery ETL."                                    │
│  "Then, run the Spark ML job on Dataproc."                        │
│  "If all succeed, send a Slack notification."                     │
│  "If anything fails, alert the on-call engineer."                 │
│                                                                      │
│  Without orchestration, you'd have to manually run each step     │
│  and check if the previous one finished. Composer automates      │
│  all of that.                                                      │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### What is Apache Airflow?

Apache Airflow is the **open-source engine** that powers Cloud Composer:

- **DAG (Directed Acyclic Graph):** You define your pipeline as a Python file that describes tasks and their dependencies. "Run task A, then B and C in parallel, then D after both finish."
- **Scheduler:** Checks your DAGs and triggers them on schedule (e.g., every day at 6 AM)
- **Workers:** Actually execute the tasks (run queries, call APIs, etc.)
- **Web UI:** A dashboard where you can see all your DAGs, monitor runs, view logs, and manually trigger pipelines

> **Cloud Composer = Google-managed Airflow.** You don't have to install, configure, or maintain the Airflow infrastructure. Google handles the web server, scheduler, database, and worker scaling. You just upload your DAG Python files to a GCS bucket and Composer runs them.

---

## Console Walkthrough: Managing Environments

### Viewing DAGs in Console

```
Console → Composer → Click on your environment name → OPEN AIRFLOW UI

┌─────────────────────────────────────────────────────────────────┐
│           VIEWING DAGs                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Navigate to Composer                                    │
│ ────────────────────────                                         │
│ Console → search "Composer" → Click "Cloud Composer"            │
│ You'll see a list of your environments.                         │
│                                                                   │
│ Step 2: Open the Airflow UI                                     │
│ ────────────────────────                                         │
│ Click on your environment name → Click "OPEN AIRFLOW UI"       │
│ (opens in a new tab — this is the Airflow web dashboard)        │
│                                                                   │
│ Step 3: Browse DAGs                                              │
│ ───────────────                                                  │
│ The Airflow UI shows a table of all your DAGs:                  │
│                                                                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ DAG          │ Owner │ Schedule │ Recent Tasks │ Actions  │   │
│ ├──────────────┼───────┼──────────┼──────────────┼──────────┤   │
│ │ daily_etl    │ team  │ @daily   │ ●●●●●●●●●●  │ ▶ ⏸ 🔄   │   │
│ │ hourly_sync  │ team  │ @hourly  │ ●●●●●○●●●●  │ ▶ ⏸ 🔄   │   │
│ │ ml_pipeline  │ ml    │ @weekly  │ ●●●●●●●●●●  │ ▶ ⏸ 🔄   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ • Green circles (●) = successful task runs                      │
│ • Red circles = failed runs                                     │
│ • Yellow circles = currently running                            │
│ • Click any DAG name to see its graph view and run history     │
│                                                                   │
│ 💡 You can also see DAGs from the Console directly:              │
│    Composer → Environment → "DAGs" tab                          │
│    (lists DAGs with links to the Airflow UI)                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Triggering a DAG Run from Console

```
Airflow UI → DAGs list → Find your DAG

┌─────────────────────────────────────────────────────────────────┐
│           TRIGGERING A DAG RUN                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Option A: Quick Trigger (from DAGs list)                        │
│ ────────────────────────────────────                             │
│ 1. In the Airflow UI DAGs list, find your DAG                  │
│ 2. Click the ▶ (play) button on the right side                 │
│ 3. The DAG triggers immediately with default config             │
│                                                                   │
│ Option B: Trigger with Config (from DAG detail)                 │
│ ───────────────────────────────────────────                      │
│ 1. Click on the DAG name to open its detail page               │
│ 2. Click "Trigger DAG" (play button) at top right              │
│ 3. Optionally add a JSON config:                                │
│    {"date": "2024-06-15", "full_refresh": true}                │
│ 4. Click "Trigger"                                              │
│                                                                   │
│ Option C: From gcloud CLI                                       │
│ ─────────────────────                                            │
│ gcloud composer environments run my-composer \                  │
│     --location=us-central1 \                                    │
│     dags -- trigger my_dag_id                                   │
│                                                                   │
│ ── After Triggering ──                                          │
│ The run appears in the DAG's "Grid" or "Graph" view.           │
│ Tasks execute in dependency order. You can watch them turn      │
│ from white (queued) → green (running) → dark green (success).  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Viewing Task Logs

```
Airflow UI → Click DAG name → Click on a task instance

┌─────────────────────────────────────────────────────────────────┐
│           VIEWING TASK LOGS                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Find the Task                                            │
│ ─────────────────                                                │
│ • Open your DAG in the Airflow UI                               │
│ • Switch to "Graph" view to see the visual pipeline             │
│ • Click on any task box (e.g., "run_bq_query")                 │
│                                                                   │
│ Step 2: Open Task Details                                        │
│ ─────────────────────                                            │
│ A popup appears with options:                                    │
│ • [Log]           — view stdout/stderr for this task run        │
│ • [Instance Details] — metadata (start time, duration, state)  │
│ • [Rendered Template] — see the actual values after Jinja       │
│ • [Mark Success]  — manually mark as success (skip re-run)     │
│ • [Clear]         — reset task to re-run it                     │
│                                                                   │
│ Step 3: Read the Log                                             │
│ ────────────────                                                 │
│ Click "Log" to see the full task execution log:                 │
│ ┌─────────────────────────────────────────────────────────┐    │
│ │ [2024-01-15 06:01:23] INFO - Running BQ query...         │    │
│ │ [2024-01-15 06:01:24] INFO - Query job started: bq_abc12 │    │
│ │ [2024-01-15 06:02:10] INFO - Query complete. 15,234 rows │    │
│ │ [2024-01-15 06:02:10] INFO - Task completed successfully │    │
│ └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│ If a task failed, the log shows the error traceback.            │
│ Use "Attempt" dropdown to see logs from retry attempts.        │
│                                                                   │
│ 💡 Tip: Logs are also stored in GCS at:                          │
│    gs://REGION-COMPOSER-ENV-BUCKET/logs/                        │
│    and in Cloud Logging (Logs Explorer).                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Scaling Workers from Console

```
Console → Composer → Click environment → ENVIRONMENT CONFIGURATION

┌─────────────────────────────────────────────────────────────────┐
│           SCALING WORKERS                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Navigate                                                 │
│ ─────────────────                                                │
│ Console → Composer → Click your environment name                │
│ → Click "ENVIRONMENT CONFIGURATION" tab                         │
│                                                                   │
│ Step 2: Edit Workloads Configuration                            │
│ ────────────────────────────────                                 │
│ Scroll to "Workloads configuration" section → Click "EDIT"     │
│                                                                   │
│ ── Scheduler ──                                                  │
│ • CPU: [2 ▼]     Memory: [4 GB ▼]     Count: [2]              │
│                                                                   │
│ ── Worker ──                                                     │
│ • CPU: [2 ▼]     Memory: [8 GB ▼]     Storage: [10 GB ▼]     │
│ • Min workers: [1]     Max workers: [6]                         │
│   (Composer 2 auto-scales workers between min and max)         │
│                                                                   │
│ ── Web Server ──                                                 │
│ • CPU: [2 ▼]     Memory: [4 GB ▼]                              │
│                                                                   │
│ ── Triggerer ──                                                  │
│ • CPU: [1 ▼]     Memory: [1 GB ▼]     Count: [1]              │
│                                                                   │
│ Step 3: Click [SAVE]                                            │
│ → Composer applies the changes (may take a few minutes)        │
│ → Workers auto-scale between min and max based on task load    │
│                                                                   │
│ 💡 Tip: If tasks are queuing up (waiting for a free worker),    │
│    increase the max worker count. If workers are idle most of  │
│    the time, decrease the min worker count to save costs.       │
│                                                                   │
│ You can also change the environment size:                       │
│ • Small — up to ~50 DAGs                                       │
│ • Medium — up to ~200 DAGs                                     │
│ • Large — 200+ DAGs                                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Deleting an Environment from Console

```
Console → Composer → Environments list

┌─────────────────────────────────────────────────────────────────┐
│           DELETING A COMPOSER ENVIRONMENT                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Navigate                                                 │
│ ─────────────────                                                │
│ Console → Composer → You see the list of environments           │
│                                                                   │
│ Step 2: Select and Delete                                       │
│ ─────────────────────                                            │
│ • Check the box ☑ next to the environment name                 │
│ • Click "DELETE" at the top                                     │
│ • Type the environment name to confirm                         │
│ • Click "DELETE" again                                          │
│                                                                   │
│ ── What Gets Deleted ──                                         │
│ • The GKE cluster (scheduler, workers, triggerer)              │
│ • The Airflow web server                                        │
│ • The Cloud SQL metadata database                              │
│ • Running DAG executions are terminated                        │
│                                                                   │
│ ── What is NOT Deleted ──                                       │
│ • The GCS bucket (DAGs, logs, plugins) — you must delete       │
│   this manually if you no longer need it                       │
│ • BigQuery tables or other resources created by your DAGs      │
│ • Secret Manager secrets                                        │
│                                                                   │
│ ── Deletion Takes Time ──                                       │
│ Deleting a Composer environment can take 15-30 minutes         │
│ because it needs to tear down the GKE cluster and associated   │
│ resources.                                                       │
│                                                                   │
│ ⚠️  Warning: This action is irreversible. All DAG run          │
│    history and metadata are permanently deleted. Back up       │
│    your DAGs from the GCS bucket before deleting.              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 58: Vertex AI** → `58-vertex-ai.md`
