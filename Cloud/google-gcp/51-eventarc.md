# Chapter 51 — Eventarc

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Eventarc Fundamentals](#part-1--eventarc-fundamentals)
- [Part 2: Event Providers & Types](#part-2--event-providers--types)
- [Part 3: Triggers](#part-3--triggers)
- [Part 4: Cloud Storage Events](#part-4--cloud-storage-events)
- [Part 5: BigQuery Events](#part-5--bigquery-events)
- [Part 6: Pub/Sub Events](#part-6--pubsub-events)
- [Part 7: Cloud Audit Log Events](#part-7--cloud-audit-log-events)
- [Part 8: Custom Events & Channels](#part-8--custom-events--channels)
- [Part 9: Destinations — Cloud Run](#part-9--destinations--cloud-run)
- [Part 10: Destinations — Cloud Functions](#part-10--destinations--cloud-functions)
- [Part 11: Destinations — Workflows](#part-11--destinations--workflows)
- [Part 12: Destinations — GKE](#part-12--destinations--gke)
- [Part 13: IAM & Security](#part-13--iam--security)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Eventarc is Google Cloud's eventing platform that enables you to build event-driven architectures by routing events from Google Cloud services, SaaS applications, and custom sources to supported destinations. Eventarc uses CloudEvents standard format, providing a unified way to react to changes across 130+ Google Cloud services — from a file uploaded to Cloud Storage, to a row inserted in BigQuery, to an infrastructure change captured in Audit Logs.

---

## Part 1 — Eventarc Fundamentals

### What Is Eventarc?

```
┌─────────────────────────────────────────────────────────────────┐
│              EVENTARC OVERVIEW                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Eventarc connects event sources to event consumers:            │
│                                                                   │
│  ┌─────────────────┐   ┌───────────┐   ┌──────────────────┐   │
│  │ Event Sources    │   │ Eventarc  │   │ Destinations      │   │
│  │                 │   │           │   │                  │   │
│  │ Cloud Storage   │──►│ Trigger:  │──►│ Cloud Run        │   │
│  │ BigQuery        │   │ matches   │   │ Cloud Functions  │   │
│  │ Pub/Sub         │   │ events to │   │ Workflows        │   │
│  │ Audit Logs      │   │ targets   │   │ GKE              │   │
│  │ Firestore       │   │           │   │                  │   │
│  │ Custom (yours)  │   │ CloudEvent│   │                  │   │
│  │ 130+ services   │   │ format    │   │                  │   │
│  └─────────────────┘   └───────────┘   └──────────────────┘   │
│                                                                   │
│  Key concepts:                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Trigger    → matches events and routes to destination    │   │
│  │ Provider   → source of events (GCS, BQ, Audit Logs)     │   │
│  │ Event type → specific event (object.finalized, etc.)     │   │
│  │ Channel    → named event bus for custom events           │   │
│  │ CloudEvent → standard event format (CNCF spec)           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Transport mechanisms:                                           │
│  • Direct events: Cloud Storage, Firestore (via Pub/Sub)       │
│  • Audit Log events: any service with Cloud Audit Logs         │
│  • Pub/Sub events: from Pub/Sub topics                         │
│  • Custom events: from your applications via channels          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Eventarc | AWS EventBridge | Azure Event Grid |
|---------|-------------|----------------|-----------------|
| Service | Eventarc | EventBridge | Event Grid |
| Event format | CloudEvents | EventBridge format | CloudEvents (v2) |
| GCP/Cloud sources | 130+ Google services | 90+ AWS services | 40+ Azure services |
| Custom events | Yes (channels) | Yes (custom bus) | Yes (custom topics) |
| SaaS sources | Limited | 30+ SaaS partners | 15+ SaaS |
| Filtering | Event type + resource | Pattern matching (JSON) | Advanced filters |
| Destinations | Cloud Run, Functions, Workflows, GKE | Lambda, SQS, Step Functions, etc. | Functions, Logic Apps, etc. |
| Schema registry | No | Yes | Yes |
| Archive & replay | No (use Pub/Sub) | Yes | No |
| Pricing | Free (pay for underlying Pub/Sub) | $1/million events | $0.60/million operations |

### Pricing

| Component | Cost |
|-----------|------|
| Eventarc triggers | Free |
| Event delivery | Pay for underlying transport (Pub/Sub, Audit Logs) |
| Pub/Sub transport | Standard Pub/Sub pricing ($40/TiB) |
| Audit Log events | Cloud Logging pricing (free for admin activity) |

---

## Part 2 — Event Providers & Types

### Event Categories

```
┌──────────────────────────────────────────────────────────────┐
│         EVENT CATEGORIES                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Direct Events (native integrations):                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Source               │ Event Types                   │    │
│  │  Cloud Storage        │ object.finalize               │    │
│  │                       │ object.delete                  │    │
│  │                       │ object.archive                 │    │
│  │                       │ object.metadataUpdate          │    │
│  │  Firestore            │ document.create                │    │
│  │                       │ document.update                │    │
│  │                       │ document.delete                │    │
│  │                       │ document.write                 │    │
│  │  Firebase Auth        │ user.create                    │    │
│  │                       │ user.delete                    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  2. Audit Log Events (130+ services):                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Any Google Cloud service that generates audit logs:  │    │
│  │  • Compute Engine (VM created/deleted)               │    │
│  │  • Cloud SQL (instance modified)                     │    │
│  │  • IAM (role granted/revoked)                        │    │
│  │  • BigQuery (job completed, table created)           │    │
│  │  • Cloud Run (service deployed)                      │    │
│  │  • Kubernetes Engine (cluster modified)              │    │
│  │  • Secret Manager (secret accessed)                  │    │
│  │  Event type: google.cloud.audit.log.v1.written      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  3. Pub/Sub Events:                                           │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Any message published to a Pub/Sub topic            │    │
│  │  Event type: google.cloud.pubsub.topic.v1.messagePublished│
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  4. Custom Events (your applications):                       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Publish custom CloudEvents to Eventarc channels     │    │
│  │  Event type: any custom type you define              │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 3 — Triggers

### Trigger Configuration

```
┌──────────────────────────────────────────────────────────────┐
│         TRIGGER ANATOMY                                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  A trigger defines:                                            │
│  ├── WHAT events to match (matching criteria)                │
│  ├── WHERE to send them (destination)                        │
│  └── HOW to authenticate (service account)                   │
│                                                                │
│  Matching criteria (filters):                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  attribute     │ operator │ value                    │    │
│  ├─────────────────┼──────────┼─────────────────────────┤    │
│  │ type           │ =        │ event type string        │    │
│  │ source         │ =        │ event source             │    │
│  │ subject        │ =        │ resource path (optional) │    │
│  │ bucket         │ =        │ GCS bucket name          │    │
│  │ database       │ =        │ Firestore database       │    │
│  │ serviceName    │ =        │ audit log service name   │    │
│  │ methodName     │ =        │ audit log method name    │    │
│  └─────────────────┴──────────┴─────────────────────────┘    │
│                                                                │
│  Each trigger can have multiple criteria (AND logic).        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# List available event types
gcloud eventarc providers list --location=us-central1

# List event types for a specific provider
gcloud eventarc providers describe storage.googleapis.com \
    --location=us-central1
```

---

## Part 4 — Cloud Storage Events

### Trigger on Object Upload

```bash
# Trigger Cloud Run on file upload to GCS
gcloud eventarc triggers create gcs-upload-trigger \
    --location=us-central1 \
    --destination-run-service=file-processor \
    --destination-run-region=us-central1 \
    --event-filters="type=google.cloud.storage.object.v1.finalized" \
    --event-filters="bucket=my-uploads-bucket" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com

# Trigger on object deletion
gcloud eventarc triggers create gcs-delete-trigger \
    --location=us-central1 \
    --destination-run-service=cleanup-service \
    --destination-run-region=us-central1 \
    --event-filters="type=google.cloud.storage.object.v1.deleted" \
    --event-filters="bucket=my-data-bucket" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

### CloudEvent Payload (GCS)

```json
{
  "specversion": "1.0",
  "type": "google.cloud.storage.object.v1.finalized",
  "source": "//storage.googleapis.com/projects/_/buckets/my-uploads-bucket",
  "subject": "objects/photos/image.jpg",
  "id": "1234567890",
  "time": "2024-01-15T10:30:00.000Z",
  "datacontenttype": "application/json",
  "data": {
    "name": "photos/image.jpg",
    "bucket": "my-uploads-bucket",
    "contentType": "image/jpeg",
    "size": "2048576",
    "metageneration": "1",
    "timeCreated": "2024-01-15T10:30:00.000Z",
    "updated": "2024-01-15T10:30:00.000Z"
  }
}
```

---

## Part 5 — BigQuery Events

### BigQuery Audit Log Events

```bash
# Trigger when a BigQuery job completes
gcloud eventarc triggers create bq-job-trigger \
    --location=us-central1 \
    --destination-run-service=bq-handler \
    --destination-run-region=us-central1 \
    --event-filters="type=google.cloud.audit.log.v1.written" \
    --event-filters="serviceName=bigquery.googleapis.com" \
    --event-filters="methodName=google.cloud.bigquery.v2.JobService.InsertJob" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com

# Trigger when a BigQuery table is created
gcloud eventarc triggers create bq-table-trigger \
    --location=us-central1 \
    --destination-run-service=table-handler \
    --destination-run-region=us-central1 \
    --event-filters="type=google.cloud.audit.log.v1.written" \
    --event-filters="serviceName=bigquery.googleapis.com" \
    --event-filters="methodName=google.cloud.bigquery.v2.TableService.InsertTable" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

---

## Part 6 — Pub/Sub Events

### Trigger from Pub/Sub Message

```bash
# Trigger Cloud Run when message published to topic
gcloud eventarc triggers create pubsub-trigger \
    --location=us-central1 \
    --destination-run-service=message-handler \
    --destination-run-region=us-central1 \
    --event-filters="type=google.cloud.pubsub.topic.v1.messagePublished" \
    --transport-topic=projects/my-project/topics/my-topic \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com

# This creates a Pub/Sub push subscription under the hood
# that delivers to the Cloud Run service
```

---

## Part 7 — Cloud Audit Log Events

### Audit Log Triggers

```
┌──────────────────────────────────────────────────────────────┐
│         AUDIT LOG EVENTS                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  React to ANY Google Cloud infrastructure change:            │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Service              │ Example Method               │    │
│  ├───────────────────────┼──────────────────────────────┤    │
│  │ compute.googleapis.com│ v1.compute.instances.insert  │    │
│  │                       │ v1.compute.instances.delete  │    │
│  │ run.googleapis.com    │ v2.Services.CreateService    │    │
│  │ sqladmin.googleapis   │ instances.create             │    │
│  │ iam.googleapis.com    │ SetIamPolicy                 │    │
│  │ secretmanager.goog    │ versions.access              │    │
│  │ container.goog        │ v1.projects.zones.clusters   │    │
│  │                       │   .create                    │    │
│  └───────────────────────┴──────────────────────────────┘    │
│                                                                │
│  Audit Log event types:                                       │
│  • google.cloud.audit.log.v1.written (general)              │
│  • Filter by serviceName + methodName                        │
│                                                                │
│  Note: Data Access logs must be ENABLED for the service     │
│  Admin Activity logs are always on                           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# React to VM creation
gcloud eventarc triggers create vm-created-trigger \
    --location=us-central1 \
    --destination-run-service=security-scanner \
    --destination-run-region=us-central1 \
    --event-filters="type=google.cloud.audit.log.v1.written" \
    --event-filters="serviceName=compute.googleapis.com" \
    --event-filters="methodName=v1.compute.instances.insert" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com

# React to IAM policy changes
gcloud eventarc triggers create iam-change-trigger \
    --location=us-central1 \
    --destination-run-service=compliance-checker \
    --destination-run-region=us-central1 \
    --event-filters="type=google.cloud.audit.log.v1.written" \
    --event-filters="serviceName=iam.googleapis.com" \
    --event-filters="methodName=google.iam.admin.v1.SetIamPolicy" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com

# React to secret access
gcloud eventarc triggers create secret-access-trigger \
    --location=us-central1 \
    --destination-run-service=audit-logger \
    --destination-run-region=us-central1 \
    --event-filters="type=google.cloud.audit.log.v1.written" \
    --event-filters="serviceName=secretmanager.googleapis.com" \
    --event-filters="methodName=google.cloud.secretmanager.v1.SecretManagerService.AccessSecretVersion" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

---

## Part 8 — Custom Events & Channels

### Publishing Custom Events

```
┌──────────────────────────────────────────────────────────────┐
│         CUSTOM EVENTS                                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Publish your own events from your applications:             │
│                                                                │
│  Your App ──publish──► Eventarc Channel ──trigger──► Target  │
│                                                                │
│  1. Create a channel (event bus)                              │
│  2. Create a trigger that matches your custom event type     │
│  3. Publish CloudEvents to the channel from your code        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create a channel
gcloud eventarc channels create my-app-events \
    --location=us-central1

# Create trigger for custom events
gcloud eventarc triggers create custom-order-trigger \
    --location=us-central1 \
    --destination-run-service=order-handler \
    --destination-run-region=us-central1 \
    --channel=my-app-events \
    --event-filters="type=com.mycompany.orders.created" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

```python
# Python — publish custom event to Eventarc channel
from google.cloud import eventarc_publishing_v1
from google.protobuf import any_pb2, timestamp_pb2
import json
import uuid

client = eventarc_publishing_v1.PublisherClient()
channel = f"projects/my-project/locations/us-central1/channels/my-app-events"

# Create CloudEvent
event = eventarc_publishing_v1.CloudEvent(
    id=str(uuid.uuid4()),
    source="//my-app/order-service",
    type="com.mycompany.orders.created",
    spec_version="1.0",
    text_data=json.dumps({
        "orderId": "order-12345",
        "customerId": "cust-678",
        "total": 99.99,
    }),
)

# Publish
client.publish_events(
    channel=channel,
    events=[event],
)
```

---

## Part 9 — Destinations — Cloud Run

### Cloud Run as Event Destination

```bash
# Create trigger → Cloud Run
gcloud eventarc triggers create file-trigger \
    --location=us-central1 \
    --destination-run-service=file-processor \
    --destination-run-region=us-central1 \
    --destination-run-path="/events/storage" \
    --event-filters="type=google.cloud.storage.object.v1.finalized" \
    --event-filters="bucket=uploads-bucket" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

```python
# Cloud Run handler (Python/Flask)
from flask import Flask, request
import json

app = Flask(__name__)

@app.route("/events/storage", methods=["POST"])
def handle_storage_event():
    # CloudEvent headers
    event_type = request.headers.get("ce-type")
    event_source = request.headers.get("ce-source")
    event_subject = request.headers.get("ce-subject")

    # Event data
    data = request.get_json()
    bucket = data.get("bucket")
    name = data.get("name")
    content_type = data.get("contentType")

    print(f"File uploaded: gs://{bucket}/{name} ({content_type})")

    # Process the file...
    return "OK", 200
```

---

## Part 10 — Destinations — Cloud Functions

### Cloud Functions (2nd Gen) as Event Destination

```bash
# Create trigger → Cloud Functions (2nd gen)
gcloud eventarc triggers create func-trigger \
    --location=us-central1 \
    --destination-function=image-processor \
    --destination-function-region=us-central1 \
    --event-filters="type=google.cloud.storage.object.v1.finalized" \
    --event-filters="bucket=images-bucket" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

```python
# Cloud Functions 2nd gen handler
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def process_image(cloud_event: CloudEvent):
    data = cloud_event.data
    bucket = data["bucket"]
    name = data["name"]

    print(f"Processing image: gs://{bucket}/{name}")
    # Process image...
```

---

## Part 11 — Destinations — Workflows

### Workflows as Event Destination

```bash
# Trigger a Workflow on file upload
gcloud eventarc triggers create workflow-trigger \
    --location=us-central1 \
    --destination-workflow=data-pipeline \
    --destination-workflow-location=us-central1 \
    --event-filters="type=google.cloud.storage.object.v1.finalized" \
    --event-filters="bucket=data-lake-bucket" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com

# The entire CloudEvent is passed as the workflow's input argument
```

```yaml
# Workflow receives the CloudEvent as input
main:
  params: [event]
  steps:
    - extract:
        assign:
          - bucket: ${event.data.bucket}
          - fileName: ${event.data.name}
          - contentType: ${event.data.contentType}

    - process:
        call: http.post
        args:
          url: https://processor.run.app/process
          body:
            source: ${"gs://" + bucket + "/" + fileName}
          auth:
            type: OIDC
        result: processResult

    - done:
        return: ${processResult.body}
```

---

## Part 12 — Destinations — GKE

### GKE as Event Destination

```bash
# Trigger GKE service on events
gcloud eventarc triggers create gke-trigger \
    --location=us-central1 \
    --destination-gke-cluster=my-cluster \
    --destination-gke-location=us-central1 \
    --destination-gke-namespace=default \
    --destination-gke-service=event-handler \
    --destination-gke-path="/events" \
    --event-filters="type=google.cloud.storage.object.v1.finalized" \
    --event-filters="bucket=gke-data-bucket" \
    --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

---

## Part 13 — IAM & Security

### Required Permissions

```
┌──────────────────────────────────────────────────────────────┐
│         IAM & SECURITY                                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  IAM Roles:                                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ roles/eventarc.admin         │ Full control          │    │
│  │ roles/eventarc.developer     │ Create/manage triggers│    │
│  │ roles/eventarc.eventReceiver │ Receive events        │    │
│  │ roles/eventarc.publisher     │ Publish custom events │    │
│  │ roles/eventarc.viewer        │ View triggers         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Service account for triggers:                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ The trigger SA needs:                                │    │
│  │ • roles/eventarc.eventReceiver (on project)         │    │
│  │ • roles/run.invoker (if destination is Cloud Run)   │    │
│  │ • roles/workflows.invoker (if dest is Workflows)    │    │
│  │                                                      │    │
│  │ For Pub/Sub transport:                               │    │
│  │ • Pub/Sub SA needs roles/iam.serviceAccountTokenCreator │
│  │   (on the trigger's service account)                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  For Audit Log events:                                        │
│  • Enable Data Access audit logs for the relevant service   │
│  • Admin Activity logs are always enabled                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Grant required roles to trigger service account
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:eventarc-sa@my-project.iam.gserviceaccount.com" \
    --role="roles/eventarc.eventReceiver"

gcloud run services add-iam-policy-binding file-processor \
    --region=us-central1 \
    --member="serviceAccount:eventarc-sa@my-project.iam.gserviceaccount.com" \
    --role="roles/run.invoker"

# Pub/Sub SA needs token creator role
PROJECT_NUM=$(gcloud projects describe my-project --format="value(projectNumber)")
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:service-${PROJECT_NUM}@gcp-sa-pubsub.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountTokenCreator"
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Service Account ─────────────────────────────────────────
resource "google_service_account" "eventarc" {
  account_id   = "eventarc-sa"
  display_name = "Eventarc Trigger SA"
  project      = var.project_id
}

resource "google_project_iam_member" "eventarc_receiver" {
  project = var.project_id
  role    = "roles/eventarc.eventReceiver"
  member  = "serviceAccount:${google_service_account.eventarc.email}"
}

resource "google_cloud_run_service_iam_member" "eventarc_invoker" {
  project  = var.project_id
  location = var.region
  service  = google_cloud_run_v2_service.processor.name
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.eventarc.email}"
}

# ─── Cloud Storage Trigger → Cloud Run ───────────────────────
resource "google_eventarc_trigger" "gcs_upload" {
  name     = "gcs-upload-trigger"
  location = var.region
  project  = var.project_id

  matching_criteria {
    attribute = "type"
    value     = "google.cloud.storage.object.v1.finalized"
  }
  matching_criteria {
    attribute = "bucket"
    value     = google_storage_bucket.uploads.name
  }

  destination {
    cloud_run_service {
      service = google_cloud_run_v2_service.processor.name
      region  = var.region
      path    = "/events/storage"
    }
  }

  service_account = google_service_account.eventarc.email
}

# ─── Audit Log Trigger → Cloud Run ───────────────────────────
resource "google_eventarc_trigger" "vm_created" {
  name     = "vm-created-trigger"
  location = var.region
  project  = var.project_id

  matching_criteria {
    attribute = "type"
    value     = "google.cloud.audit.log.v1.written"
  }
  matching_criteria {
    attribute = "serviceName"
    value     = "compute.googleapis.com"
  }
  matching_criteria {
    attribute = "methodName"
    value     = "v1.compute.instances.insert"
  }

  destination {
    cloud_run_service {
      service = google_cloud_run_v2_service.security_scanner.name
      region  = var.region
    }
  }

  service_account = google_service_account.eventarc.email
}

# ─── Pub/Sub Trigger → Cloud Run ─────────────────────────────
resource "google_eventarc_trigger" "pubsub" {
  name     = "pubsub-trigger"
  location = var.region
  project  = var.project_id

  matching_criteria {
    attribute = "type"
    value     = "google.cloud.pubsub.topic.v1.messagePublished"
  }

  transport {
    pubsub {
      topic = google_pubsub_topic.orders.id
    }
  }

  destination {
    cloud_run_service {
      service = google_cloud_run_v2_service.handler.name
      region  = var.region
    }
  }

  service_account = google_service_account.eventarc.email
}

# ─── Event → Workflow ─────────────────────────────────────────
resource "google_eventarc_trigger" "to_workflow" {
  name     = "gcs-to-workflow"
  location = var.region
  project  = var.project_id

  matching_criteria {
    attribute = "type"
    value     = "google.cloud.storage.object.v1.finalized"
  }
  matching_criteria {
    attribute = "bucket"
    value     = google_storage_bucket.data.name
  }

  destination {
    workflow = google_workflows_workflow.pipeline.id
  }

  service_account = google_service_account.eventarc.email
}

# ─── Custom Event Channel ────────────────────────────────────
resource "google_eventarc_channel" "custom" {
  name     = "app-events"
  location = var.region
  project  = var.project_id
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# TRIGGERS
# ═══════════════════════════════════════════════════════════════
gcloud eventarc triggers create NAME \
    --location=LOC \
    --event-filters="type=EVENT_TYPE" \
    --event-filters="ATTR=VALUE" \
    --destination-run-service=SVC \
    --destination-run-region=REG \
    --destination-run-path=/path \
    --service-account=SA

gcloud eventarc triggers list --location=LOC
gcloud eventarc triggers describe NAME --location=LOC
gcloud eventarc triggers update NAME --location=LOC [options]
gcloud eventarc triggers delete NAME --location=LOC

# ═══════════════════════════════════════════════════════════════
# PROVIDERS
# ═══════════════════════════════════════════════════════════════
gcloud eventarc providers list --location=LOC
gcloud eventarc providers describe PROVIDER --location=LOC

# ═══════════════════════════════════════════════════════════════
# CHANNELS
# ═══════════════════════════════════════════════════════════════
gcloud eventarc channels create NAME --location=LOC
gcloud eventarc channels list --location=LOC
gcloud eventarc channels describe NAME --location=LOC
gcloud eventarc channels delete NAME --location=LOC
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Image Processing Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: AUTOMATED IMAGE PROCESSING                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  User uploads image → automatic processing pipeline:                 │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  User uploads to: gs://uploads-bucket/photos/img.jpg     │        │
│  │       │                                                   │        │
│  │       ▼ Eventarc trigger (object.finalized)              │        │
│  │  ┌──────────────────────────────────────────────┐        │        │
│  │  │ Cloud Run: image-processor                    │        │        │
│  │  │ 1. Download image from GCS                    │        │        │
│  │  │ 2. Generate thumbnail (256x256)               │        │        │
│  │  │ 3. Upload to gs://thumbnails-bucket/          │        │        │
│  │  │ 4. Call Vision API for labels/moderation       │        │        │
│  │  │ 5. Store metadata in Firestore                │        │        │
│  │  │ 6. Publish "image.processed" to Pub/Sub       │        │        │
│  │  └──────────────────────────────────────────────┘        │        │
│  │       │                                                   │        │
│  │       ▼ Eventarc trigger (pubsub)                        │        │
│  │  ┌──────────────────────────────────────────────┐        │        │
│  │  │ Cloud Function: notification-sender           │        │        │
│  │  │ → Email user that image is ready              │        │        │
│  │  └──────────────────────────────────────────────┘        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Security Auto-Remediation

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: AUTO-REMEDIATION FOR SECURITY                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Detect and auto-fix security misconfigurations:                     │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Trigger 1: Public VM detected                           │        │
│  │  Event: compute.instances.insert                         │        │
│  │  → Cloud Run checks if VM has external IP               │        │
│  │  → If yes + not in allowed list → remove external IP    │        │
│  │  → Alert in Slack                                         │        │
│  │                                                           │        │
│  │  Trigger 2: Public GCS bucket detected                   │        │
│  │  Event: storage.setIamPermissions                        │        │
│  │  → Cloud Run checks if allUsers / allAuthenticatedUsers  │        │
│  │  → If yes → remove public access                        │        │
│  │  → Alert in Slack + create Jira ticket                   │        │
│  │                                                           │        │
│  │  Trigger 3: Overly permissive IAM role                   │        │
│  │  Event: iam.SetIamPolicy                                 │        │
│  │  → Cloud Run checks for roles/owner or roles/editor     │        │
│  │    granted to non-approved principals                    │        │
│  │  → Alert security team (don't auto-remediate IAM)       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  All triggers: Audit Log events → Cloud Run auto-remediation         │
│  Security team reviews alerts and permanent fixes in Terraform       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Data Lake Ingestion

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: EVENT-DRIVEN DATA LAKE                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Files arrive in GCS → automatic processing into data lake:         │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  gs://raw-data/                                           │        │
│  │  ├── csv/ → Eventarc → Workflow:                         │        │
│  │  │         1. Validate CSV schema                        │        │
│  │  │         2. Load into BQ staging table                 │        │
│  │  │         3. Run dbt transform                          │        │
│  │  │         4. Move file to gs://processed/               │        │
│  │  │                                                        │        │
│  │  ├── json/ → Eventarc → Cloud Function:                  │        │
│  │  │          1. Parse and validate JSON                   │        │
│  │  │          2. Stream to BigQuery via streaming insert   │        │
│  │  │          3. Archive to gs://archive/                  │        │
│  │  │                                                        │        │
│  │  └── parquet/ → Eventarc → Cloud Run:                    │        │
│  │               1. Register in BigQuery as external table  │        │
│  │               2. Update data catalog                     │        │
│  │               3. Notify data team via Pub/Sub            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Eventarc routes by file path pattern (bucket filter)               │
│  Each format has its own optimized processing pipeline              │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create GCS trigger | `gcloud eventarc triggers create T --event-filters="type=google.cloud.storage.object.v1.finalized" --event-filters="bucket=B"` |
| Create Audit Log trigger | `--event-filters="type=google.cloud.audit.log.v1.written" --event-filters="serviceName=S" --event-filters="methodName=M"` |
| Create Pub/Sub trigger | `--event-filters="type=google.cloud.pubsub.topic.v1.messagePublished" --transport-topic=T` |
| Destination: Cloud Run | `--destination-run-service=SVC --destination-run-region=R` |
| Destination: Workflow | `--destination-workflow=W --destination-workflow-location=L` |
| Destination: GKE | `--destination-gke-cluster=C --destination-gke-service=S` |
| List providers | `gcloud eventarc providers list --location=L` |
| List triggers | `gcloud eventarc triggers list --location=L` |
| Create channel | `gcloud eventarc channels create NAME --location=L` |
| Event format | CloudEvents (CNCF standard) |
| Pricing | Free (pay for underlying Pub/Sub transport) |

---

## What is Eventarc? (Beginner Explanation)

### Simple Analogy

```
┌──────────────────────────────────────────────────────────────┐
│         NOTIFICATION SYSTEM ANALOGY                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Eventarc is like a NOTIFICATION SYSTEM:                     │
│                                                                │
│  Think of your phone's notification center:                  │
│  • When you get an email → notification pops up              │
│  • When someone likes your photo → notification pops up      │
│  • When a package is delivered → notification pops up        │
│                                                                │
│  Eventarc works the same way for GCP:                        │
│  • When a file is uploaded to Cloud Storage → your code runs │
│  • When a database record changes → your code runs           │
│  • When a VM is created → your code runs                     │
│  • When someone accesses a secret → your code runs           │
│                                                                │
│  ┌───────────────────┐   ┌──────────┐   ┌──────────────┐    │
│  │ Something happens │   │ Eventarc │   │ Your code    │    │
│  │ in GCP            │──►│ "Hey,    │──►│ reacts to    │    │
│  │ (file uploaded,   │   │ something│   │ the event    │    │
│  │  VM created,      │   │ happened!│   │ (process     │    │
│  │  secret accessed) │   │ Here are │   │  file, scan  │    │
│  │                   │   │ details" │   │  VM, alert)  │    │
│  └───────────────────┘   └──────────┘   └──────────────┘    │
│                                                                │
│  You DON'T need to constantly check "did anything happen?"  │
│  (that would be polling — wasteful and slow)                 │
│                                                                │
│  Instead, Eventarc TELLS you when something happens          │
│  (event-driven — efficient and instant)                      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Event-Driven = Reacting to Things Happening

```
┌──────────────────────────────────────────────────────────────┐
│         EVENT-DRIVEN ARCHITECTURE                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Traditional (polling):                                       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Your code: "Any new files?" → No                    │    │
│  │  (wait 5 seconds)                                     │    │
│  │  Your code: "Any new files?" → No                    │    │
│  │  (wait 5 seconds)                                     │    │
│  │  Your code: "Any new files?" → No                    │    │
│  │  (wait 5 seconds)                                     │    │
│  │  Your code: "Any new files?" → Yes! Process it       │    │
│  │                                                      │    │
│  │  Problem: wasting resources asking repeatedly        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Event-driven (Eventarc):                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Your code: (sleeping, using zero resources)         │    │
│  │  ...                                                  │    │
│  │  File uploaded!                                       │    │
│  │  Eventarc: "Wake up! A file was uploaded. Here are  │    │
│  │             the details."                             │    │
│  │  Your code: processes the file → goes back to sleep  │    │
│  │                                                      │    │
│  │  Benefit: pay only when events actually happen       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Eventarc vs Pub/Sub — When to Use Which

```
┌──────────────────────────────────────────────────────────────┐
│         EVENTARC vs PUB/SUB                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Both deal with events, but they serve different purposes:   │
│                                                                │
│  ┌────────────────────────┬─────────────────────────────┐    │
│  │ Eventarc               │ Pub/Sub                      │    │
│  ├────────────────────────┼─────────────────────────────┤    │
│  │ Event ROUTING           │ Message TRANSPORT            │    │
│  │ "When X happens in     │ "Send message A to           │    │
│  │  GCP, run my code"     │  whoever is listening"       │    │
│  ├────────────────────────┼─────────────────────────────┤    │
│  │ Auto-detects events    │ You publish messages         │    │
│  │ from 130+ GCP services │ manually from your code     │    │
│  ├────────────────────────┼─────────────────────────────┤    │
│  │ CloudEvents format     │ Any message format           │    │
│  ├────────────────────────┼─────────────────────────────┤    │
│  │ Built-in GCP triggers  │ General-purpose messaging    │    │
│  │ (GCS, BQ, Audit Logs)  │ (app-to-app, any data)      │    │
│  ├────────────────────────┼─────────────────────────────┤    │
│  │ Uses Pub/Sub under     │ Standalone service           │    │
│  │ the hood for transport │                              │    │
│  └────────────────────────┴─────────────────────────────┘    │
│                                                                │
│  Use EVENTARC when:                                            │
│  ├── Reacting to GCP infrastructure changes                  │
│  │   "When a file is uploaded to GCS, process it"            │
│  ├── Reacting to audit log events                             │
│  │   "When a VM is created, scan it for compliance"          │
│  └── You want automatic event detection (no code to publish) │
│                                                                │
│  Use PUB/SUB when:                                             │
│  ├── Your application generates custom messages               │
│  │   "Order placed" → notify inventory + shipping services  │
│  ├── Decoupling microservices                                 │
│  │   Service A publishes → Services B, C, D subscribe       │
│  ├── Need message replay, dead-letter queues, ordering       │
│  └── High-throughput messaging (millions of messages/sec)    │
│                                                                │
│  They work TOGETHER:                                           │
│  Eventarc uses Pub/Sub under the hood to transport events.   │
│  You can also create Eventarc triggers on Pub/Sub topics.    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Creating & Managing Triggers

### Step 1 — Create a Trigger from Console

```
┌──────────────────────────────────────────────────────────────┐
│  CONSOLE: Create an Eventarc Trigger                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Go to Console → search "Eventarc" → click on it         │
│     (or navigate: Eventarc → Triggers)                       │
│                                                                │
│  2. Click "CREATE TRIGGER" button at the top                 │
│                                                                │
│  3. Fill in the trigger details:                              │
│     ┌──────────────────────────────────────────────────┐     │
│     │ Trigger name:    [ gcs-upload-trigger     ]       │     │
│     │ Region:          [ us-central1            ▼ ]     │     │
│     │                                                    │     │
│     │ ── Event Provider ──                               │     │
│     │ Event provider:  [ Cloud Storage          ▼ ]     │     │
│     │ Event type:      [ google.cloud.storage.         │     │
│     │                    object.v1.finalized    ▼ ]     │     │
│     │ Bucket:          [ my-uploads-bucket      ▼ ]     │     │
│     │                                                    │     │
│     │ ── Destination ──                                  │     │
│     │ Destination type:[ Cloud Run              ▼ ]     │     │
│     │ Cloud Run        [ file-processor         ▼ ]     │     │
│     │ service:                                           │     │
│     │ Service URL path:[ /events/storage         ]       │     │
│     │ Region:          [ us-central1            ▼ ]     │     │
│     │                                                    │     │
│     │ ── Service Account ──                              │     │
│     │ Service account: [ eventarc-sa@project     ▼ ]    │     │
│     │                   .iam.gserviceaccount.com         │     │
│     └──────────────────────────────────────────────────┘     │
│                                                                │
│  4. Click "CREATE"                                            │
│                                                                │
│  5. Wait for trigger to become Active (usually 1-2 minutes)  │
│                                                                │
│  Note: For Audit Log events, choose:                          │
│  • Event provider: select the GCP service (e.g., Compute)   │
│  • Event type: google.cloud.audit.log.v1.written             │
│  • Method name: e.g., v1.compute.instances.insert            │
│                                                                │
│  Equivalent gcloud command:                                   │
│  gcloud eventarc triggers create gcs-upload-trigger \        │
│      --location=us-central1 \                                 │
│      --destination-run-service=file-processor \               │
│      --destination-run-region=us-central1 \                   │
│      --destination-run-path="/events/storage" \               │
│      --event-filters="type=google.cloud.storage.object.      │
│        v1.finalized" \                                        │
│      --event-filters="bucket=my-uploads-bucket" \             │
│      --service-account=eventarc-sa@project.iam...            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 2 — View Trigger Details

```
┌──────────────────────────────────────────────────────────────┐
│  CONSOLE: View Trigger Details                                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Console → Eventarc → Triggers                             │
│                                                                │
│  2. You'll see a list of all triggers:                        │
│     ┌──────────────────────────────────────────────────┐     │
│     │ Name              │ Status │ Provider │ Region   │     │
│     ├───────────────────┼────────┼──────────┼──────────┤     │
│     │ gcs-upload-trigger│ Active │ Storage  │ us-cntrl │     │
│     │ vm-created-trigger│ Active │ Audit Log│ us-cntrl │     │
│     │ pubsub-trigger    │ Active │ Pub/Sub  │ us-cntrl │     │
│     └──────────────────────────────────────────────────┘     │
│                                                                │
│  3. Click on a trigger name to see details:                  │
│     • Trigger name and ID                                     │
│     • Event matching criteria (provider, type, filters)      │
│     • Destination (Cloud Run service, path, region)          │
│     • Service account used for authentication                │
│     • Transport (Pub/Sub subscription created automatically) │
│     • Creation time and last modified time                   │
│                                                                │
│  4. Check the "LOGS" tab to see:                              │
│     • Events received                                         │
│     • Events delivered successfully                           │
│     • Failed deliveries with error details                   │
│                                                                │
│  Equivalent gcloud command:                                   │
│  gcloud eventarc triggers describe gcs-upload-trigger \      │
│      --location=us-central1                                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 3 — Delete a Trigger from Console

```
┌──────────────────────────────────────────────────────────────┐
│  CONSOLE: Delete a Trigger                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Console → Eventarc → Triggers                             │
│                                                                │
│  2. Find the trigger you want to delete                      │
│                                                                │
│  3. Either:                                                    │
│     a) Check the box next to the trigger → click "DELETE"    │
│        at the top bar                                         │
│     b) Click the trigger name → click "DELETE" button        │
│        on the details page                                    │
│                                                                │
│  4. Confirm the deletion in the dialog                       │
│                                                                │
│  5. The trigger and its underlying Pub/Sub subscription      │
│     are deleted automatically                                 │
│                                                                │
│  Note: Unlike API Gateway, there's no dependency order —    │
│  you can delete a trigger directly without deleting other    │
│  resources first.                                              │
│                                                                │
│  Equivalent gcloud command:                                   │
│  gcloud eventarc triggers delete gcs-upload-trigger \        │
│      --location=us-central1                                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Common gcloud Commands for Triggers

```bash
# ═══════════════════════════════════════════════════════════════
# LIST & MANAGE TRIGGERS
# ═══════════════════════════════════════════════════════════════

# List all triggers in a region
gcloud eventarc triggers list --location=us-central1

# List all triggers across all regions
gcloud eventarc triggers list --location=-

# Describe a specific trigger (see full details)
gcloud eventarc triggers describe gcs-upload-trigger \
    --location=us-central1

# Delete a trigger
gcloud eventarc triggers delete gcs-upload-trigger \
    --location=us-central1

# Delete with auto-confirm (skip prompt)
gcloud eventarc triggers delete gcs-upload-trigger \
    --location=us-central1 --quiet

# ═══════════════════════════════════════════════════════════════
# USEFUL DISCOVERY COMMANDS
# ═══════════════════════════════════════════════════════════════

# List all available event providers
gcloud eventarc providers list --location=us-central1

# See what event types a provider supports
gcloud eventarc providers describe storage.googleapis.com \
    --location=us-central1

# List all channels (for custom events)
gcloud eventarc channels list --location=us-central1

# Delete a channel
gcloud eventarc channels delete my-app-events \
    --location=us-central1
```

---

## What's Next?

Continue to **Chapter 52: GKE Deep Dive** → `52-gke-deep-dive.md`
