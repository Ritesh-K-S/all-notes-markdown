# Chapter 21: Cloud Storage Advanced (GCS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Access Control — IAM vs ACLs](#part-1-access-control--iam-vs-acls)
- [Part 2: Signed URLs & Signed Policy Documents](#part-2-signed-urls--signed-policy-documents)
- [Part 3: Pub/Sub Notifications & Eventarc](#part-3-pubsub-notifications--eventarc)
- [Part 4: Object Composition](#part-4-object-composition)
- [Part 5: Storage Transfer Service](#part-5-storage-transfer-service)
- [Part 6: Requester Pays](#part-6-requester-pays)
- [Part 7: Customer-Managed Encryption Keys (CMEK & CSEK)](#part-7-customer-managed-encryption-keys-cmek--csek)
- [Part 8: Object Holds & Bucket Lock](#part-8-object-holds--bucket-lock)
- [Part 9: Batch Operations](#part-9-batch-operations)
- [Part 10: Console Walkthrough — Advanced GCS Features](#part-10-console-walkthrough--advanced-gcs-features)
- [Part 11: Terraform & CLI](#part-11-terraform--cli)
- [Part 12: Real-World Patterns](#part-12-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Building on Cloud Storage fundamentals (Chapter 20), this chapter covers advanced features: fine-grained access control, signed URLs for temporary access, event notifications, object composition, cross-cloud data transfer, encryption options, and compliance features.

```
What you'll learn:
├── Access Control
│   ├── Uniform vs Fine-grained access
│   ├── IAM policies (recommended)
│   ├── ACLs (legacy, per-object control)
│   └── When to use each
├── Signed URLs & Signed Policy Documents
│   ├── Temporary access without Google account
│   ├── V4 signing (recommended)
│   ├── Upload via signed URL
│   └── Signed policy documents (HTML form uploads)
├── Pub/Sub Notifications & Eventarc
│   ├── Object change notifications → Pub/Sub
│   ├── Eventarc triggers (Cloud Run, Cloud Functions)
│   └── Notification configuration
├── Object Composition
│   ├── Combine up to 32 objects into one
│   ├── Parallel uploads + compose pattern
│   └── Use cases (large file assembly)
├── Storage Transfer Service
│   ├── AWS S3 → GCS
│   ├── Azure Blob → GCS
│   ├── HTTP/HTTPS source → GCS
│   ├── Between GCS buckets
│   └── Scheduled & recurring transfers
├── Requester Pays
├── Encryption (CMEK & CSEK)
├── Object Holds & Bucket Lock (compliance)
├── Batch Operations
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: Access Control — IAM vs ACLs

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACCESS CONTROL MODELS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GCS has TWO access control systems:                                 │
│                                                                       │
│ 1. UNIFORM ACCESS (recommended ✅)                                  │
│    ├── Uses IAM ONLY for all access decisions                     │
│    ├── Bucket-level permissions apply to all objects              │
│    ├── No per-object ACLs                                         │
│    ├── Simpler, easier to audit                                   │
│    └── Cannot be reversed once enabled for 90 days!              │
│                                                                       │
│ 2. FINE-GRAINED ACCESS (legacy)                                     │
│    ├── Uses IAM + ACLs together                                   │
│    ├── ACLs give per-object control                               │
│    ├── More complex, harder to audit                              │
│    └── Default for new buckets (but Google recommends Uniform)   │
│                                                                       │
│ ⚡ Google's recommendation: Use UNIFORM for all new buckets.       │
│   ACLs are legacy and make it hard to audit "who has access to   │
│   what" because permissions can come from IAM OR ACLs.           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### IAM Roles for Cloud Storage

```
┌─────────────────────────────────────────────────────────────────────┐
│           PREDEFINED IAM ROLES                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────────────────┬────────────────────────────┐  │
│ │ Role                             │ What it allows              │  │
│ ├──────────────────────────────────┼────────────────────────────┤  │
│ │ roles/storage.objectViewer       │ Read objects only           │  │
│ │                                  │ (list + get objects)        │  │
│ │                                  │                             │  │
│ │ roles/storage.objectCreator      │ Create objects only         │  │
│ │                                  │ (upload, no read/delete)    │  │
│ │                                  │                             │  │
│ │ roles/storage.objectUser         │ Read + write + delete       │  │
│ │                                  │ objects (no bucket mgmt)    │  │
│ │                                  │                             │  │
│ │ roles/storage.objectAdmin        │ Full object control         │  │
│ │                                  │ (CRUD + ACLs on objects)    │  │
│ │                                  │                             │  │
│ │ roles/storage.admin              │ Full control (bucket +      │  │
│ │                                  │ objects + IAM)              │  │
│ │                                  │                             │  │
│ │ roles/storage.hmacKeyAdmin       │ Manage HMAC keys            │  │
│ │                                  │ (S3 interop)                │  │
│ │                                  │                             │  │
│ │ roles/storage.legacyBucketOwner  │ Legacy: ACL-based control   │  │
│ │ roles/storage.legacyBucketReader │ Legacy: Read bucket + list  │  │
│ │ roles/storage.legacyBucketWriter │ Legacy: Write to bucket     │  │
│ │ roles/storage.legacyObjectOwner  │ Legacy: Per-object ACL      │  │
│ │ roles/storage.legacyObjectReader │ Legacy: Read objects (ACL)  │  │
│ └──────────────────────────────────┴────────────────────────────┘  │
│                                                                       │
│ Granting access at different levels:                                │
│ ├── Project level: User can access ALL buckets in the project    │
│ ├── Bucket level: User can access only THAT bucket               │
│ ├── Object level: ❌ Not possible with IAM (use ACLs or Signed) │
│ │   ⚡ This is WHY ACLs exist — per-object permissions           │
│ │   But prefer: Organize objects into separate buckets instead   │
│ └── Folder level: Use IAM Conditions with resource.name prefix   │
│                                                                       │
│ IAM Condition example (folder-level access):                       │
│ Role: roles/storage.objectViewer                                   │
│ Condition:                                                          │
│   resource.name.startsWith("projects/_/buckets/my-bucket/         │
│   objects/public/")                                                │
│ ⚡ User can ONLY read objects under the "public/" prefix.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### ACLs (Access Control Lists) — Legacy

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACLs (LEGACY — USE ONLY WHEN NEEDED)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ACLs grant per-OBJECT or per-BUCKET permissions.                   │
│ Only available in Fine-grained access mode.                        │
│                                                                       │
│ ACL entry = (scope, permission)                                    │
│                                                                       │
│ Scopes (who):                                                       │
│ ├── User email: user-john@company.com                             │
│ ├── Group email: group-team@company.com                           │
│ ├── Domain: domain-company.com                                     │
│ ├── Project: project-owners-123456                                │
│ ├── allUsers (public — anyone on internet)                        │
│ └── allAuthenticatedUsers (any Google account)                    │
│                                                                       │
│ Permissions (what):                                                  │
│ ├── READER: View object / list bucket                             │
│ ├── WRITER: Upload to bucket / overwrite object                  │
│ └── OWNER: Full control + manage ACLs                             │
│                                                                       │
│ Predefined ACLs (shortcuts):                                       │
│ ├── private: Owner-only access (default)                          │
│ ├── publicRead: Public readable ⚠️                                │
│ ├── publicReadWrite: Public read + write ⚠️⚠️                    │
│ ├── authenticatedRead: Any Google account can read               │
│ ├── bucketOwnerRead: Bucket owner can read object                │
│ ├── bucketOwnerFullControl: Bucket owner gets full control       │
│ └── projectPrivate: Project team access based on roles           │
│                                                                       │
│ Setting ACLs:                                                       │
│ gsutil acl set public-read gs://my-bucket/file.txt               │
│ gsutil acl ch -u john@company.com:READER gs://my-bucket/file.txt │
│ gsutil acl get gs://my-bucket/file.txt                            │
│                                                                       │
│ ⚠️ ACL ISSUES:                                                      │
│ ├── Hard to audit (permissions split across IAM + ACLs)          │
│ ├── Max 100 ACL entries per object                                │
│ ├── Easy to accidentally make objects public                     │
│ └── Cannot use IAM Conditions with ACLs                          │
│                                                                       │
│ AWS equivalent: S3 ACLs (also deprecated in favor of policies)   │
│ Azure equivalent: Shared Access Signatures (different concept)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Uniform vs Fine-Grained — Decision Guide

```
┌──────────────────────────┬───────────────────┬───────────────────────┐
│ Feature                  │ Uniform ✅         │ Fine-grained          │
├──────────────────────────┼───────────────────┼───────────────────────┤
│ Access control           │ IAM only          │ IAM + ACLs            │
│ Per-object permissions   │ ❌ (use buckets)  │ ✅ via ACLs           │
│ Auditability             │ Simple ✅          │ Complex (2 systems)   │
│ Public access prevention │ Org policy works  │ Harder to enforce     │
│ IAM Conditions           │ ✅ Supported      │ ✅ Supported          │
│ Signed URLs              │ ✅ Work fine      │ ✅ Work fine          │
│ Recommended              │ ✅ YES            │ Only if per-object    │
│                          │                   │ permissions needed    │
│ Can switch to Uniform    │ N/A               │ ✅ (90-day lock)      │
│ Can switch to Fine-grain │ ❌ After 90 days  │ N/A                   │
└──────────────────────────┴───────────────────┴───────────────────────┘

⚡ Rule: Start with Uniform. Only use Fine-grained if you MUST
   have per-object permissions AND separate buckets won't work.
```

---

## Part 2: Signed URLs & Signed Policy Documents

```
┌─────────────────────────────────────────────────────────────────────┐
│           SIGNED URLs                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A URL that gives temporary access to a specific object      │
│       WITHOUT requiring the user to have a Google account          │
│       or IAM permissions.                                          │
│                                                                       │
│ How it works:                                                       │
│ 1. Your server signs a URL using a service account's private key │
│ 2. The signed URL contains: bucket, object, expiration, signature│
│ 3. Anyone with the URL can access the object until it expires    │
│                                                                       │
│ Use cases:                                                          │
│ ├── Let users download private files (invoices, reports)         │
│ ├── Let users upload files directly to GCS (skip your server)   │
│ ├── Share files with external partners (no Google account)       │
│ └── Pre-authenticated URLs in emails or APIs                     │
│                                                                       │
│ Supported operations:                                               │
│ ├── GET (download)                                                │
│ ├── PUT (upload)                                                  │
│ ├── DELETE                                                        │
│ └── HEAD (metadata only)                                          │
│                                                                       │
│ Signing versions:                                                   │
│ ├── V4 signing (recommended ✅) — max 7 days expiration         │
│ └── V2 signing (legacy) — no max expiration                      │
│                                                                       │
│ URL format (V4):                                                   │
│ https://storage.googleapis.com/BUCKET/OBJECT                     │
│   ?X-Goog-Algorithm=GOOG4-RSA-SHA256                             │
│   &X-Goog-Credential=SA_EMAIL/DATE/auto/storage/goog4_request   │
│   &X-Goog-Date=20260517T120000Z                                 │
│   &X-Goog-Expires=3600                                           │
│   &X-Goog-SignedHeaders=host                                     │
│   &X-Goog-Signature=LONG_HEX_SIGNATURE                          │
│                                                                       │
│ AWS equivalent: S3 Pre-signed URLs                                 │
│ Azure equivalent: Shared Access Signatures (SAS tokens)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Generating Signed URLs

```
┌─────────────────────────────────────────────────────────────────────┐
│           GENERATING SIGNED URLs                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Method 1: gsutil (for testing)                                     │
│ ─────────────────────────────────────                               │
│ # Download URL (expires in 1 hour)                                 │
│ gsutil signurl -d 1h \                                             │
│   /path/to/service-account-key.json \                             │
│   gs://my-bucket/private/report.pdf                               │
│                                                                       │
│ # Upload URL (PUT method, expires in 30 minutes)                  │
│ gsutil signurl -m PUT -d 30m \                                    │
│   /path/to/service-account-key.json \                             │
│   gs://my-bucket/uploads/user-file.jpg                            │
│                                                                       │
│ Method 2: gcloud storage (newer CLI)                               │
│ ─────────────────────────────────────                               │
│ gcloud storage sign-url \                                          │
│   gs://my-bucket/private/report.pdf \                             │
│   --private-key-file=/path/to/key.json \                          │
│   --duration=1h                                                    │
│                                                                       │
│ Method 3: Client library (production — recommended ✅)             │
│ ─────────────────────────────────────                               │
│ # Python example                                                   │
│ from google.cloud import storage                                   │
│ from datetime import timedelta                                     │
│                                                                       │
│ client = storage.Client()                                          │
│ bucket = client.bucket("my-bucket")                                │
│ blob = bucket.blob("private/report.pdf")                          │
│                                                                       │
│ # Generate download URL                                            │
│ url = blob.generate_signed_url(                                    │
│     version="v4",                                                  │
│     expiration=timedelta(hours=1),                                │
│     method="GET",                                                  │
│ )                                                                   │
│ print(url)                                                          │
│                                                                       │
│ # Generate upload URL                                              │
│ upload_url = blob.generate_signed_url(                             │
│     version="v4",                                                  │
│     expiration=timedelta(minutes=30),                              │
│     method="PUT",                                                  │
│     content_type="application/octet-stream",                      │
│ )                                                                   │
│                                                                       │
│ ⚡ For production: Use IAM-signed URLs (no key file needed):       │
│   blob.generate_signed_url(                                       │
│       version="v4",                                                │
│       expiration=timedelta(hours=1),                              │
│       method="GET",                                                │
│       service_account_email="sa@project.iam.gserviceaccount.com",│
│   )                                                                │
│   Requires: iam.serviceAccounts.signBlob permission              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Signed Policy Documents (HTML Form Uploads)

```
┌─────────────────────────────────────────────────────────────────────┐
│           SIGNED POLICY DOCUMENTS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Allow users to upload directly from browser via HTML form   │
│       with restrictions on what they can upload.                  │
│                                                                       │
│ Policy conditions (restrictions):                                   │
│ ├── Bucket: Must upload to specific bucket                       │
│ ├── Key prefix: Must start with "uploads/user123/"              │
│ ├── Content-Type: Must be "image/jpeg" or "image/png"           │
│ ├── Size limit: Must be < 10 MB                                  │
│ └── Expiration: Policy expires at specific time                  │
│                                                                       │
│ Use case: User avatar upload, file submission forms              │
│                                                                       │
│ AWS equivalent: S3 POST Policy                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Pub/Sub Notifications & Eventarc

```
┌─────────────────────────────────────────────────────────────────────┐
│           OBJECT CHANGE NOTIFICATIONS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GCS can send notifications when objects are created, deleted,      │
│ archived, or have metadata updated. This enables event-driven     │
│ architectures.                                                     │
│                                                                       │
│ Two notification systems:                                           │
│                                                                       │
│ 1. Pub/Sub Notifications (recommended ✅)                          │
│    ├── Object changes → Pub/Sub topic → subscribers             │
│    ├── Subscribers: Cloud Functions, Cloud Run, Dataflow, etc.  │
│    ├── JSON message with object metadata                         │
│    └── Can filter by prefix/suffix (at subscriber level)        │
│                                                                       │
│ 2. Eventarc (newer, higher-level ✅)                               │
│    ├── Object changes → Eventarc → Cloud Run / Cloud Functions  │
│    ├── CloudEvents format                                        │
│    ├── Filter by event type, bucket, object prefix               │
│    └── ⚡ Recommended for new projects (built on Pub/Sub +       │
│          Cloud Audit Logs under the hood)                        │
│                                                                       │
│ 3. Object Change Notification (legacy — DO NOT USE ❌)             │
│    └── Webhook-based, replaced by Pub/Sub notifications         │
│                                                                       │
│ Event types:                                                        │
│ ├── OBJECT_FINALIZE: Object created or overwritten              │
│ ├── OBJECT_METADATA_UPDATE: Metadata changed                    │
│ ├── OBJECT_DELETE: Object permanently deleted                   │
│ └── OBJECT_ARCHIVE: Object version becomes non-current          │
│     (when versioning is enabled)                                │
│                                                                       │
│ AWS equivalent: S3 Event Notifications → SNS/SQS/Lambda         │
│                  + S3 → EventBridge                               │
│ Azure equivalent: Blob Storage → Event Grid                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Setting Up Pub/Sub Notifications

```
┌─────────────────────────────────────────────────────────────────────┐
│           PUB/SUB NOTIFICATION SETUP                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Create a Pub/Sub topic                                     │
│ gcloud pubsub topics create gcs-notifications                     │
│                                                                       │
│ Step 2: Create notification on the bucket                          │
│ gcloud storage buckets notifications create \                     │
│   gs://my-bucket \                                                 │
│   --topic=gcs-notifications \                                     │
│   --event-types=OBJECT_FINALIZE,OBJECT_DELETE \                  │
│   --object-prefix=uploads/                                        │
│                                                                       │
│ Step 3: Verify                                                     │
│ gcloud storage buckets notifications list gs://my-bucket          │
│                                                                       │
│ Message format (JSON in Pub/Sub):                                  │
│ {                                                                   │
│   "kind": "storage#object",                                       │
│   "id": "my-bucket/uploads/photo.jpg/1234567890",                │
│   "name": "uploads/photo.jpg",                                    │
│   "bucket": "my-bucket",                                          │
│   "contentType": "image/jpeg",                                    │
│   "size": "2048576",                                               │
│   "timeCreated": "2026-05-17T10:00:00.000Z",                    │
│   "metageneration": "1",                                          │
│   "eventType": "OBJECT_FINALIZE"                                 │
│ }                                                                   │
│                                                                       │
│ Common pattern: GCS → Pub/Sub → Cloud Function                   │
│ (e.g., Image uploaded → generate thumbnail automatically)        │
│                                                                       │
│ ⚡ The GCS service account needs Pub/Sub Publisher role            │
│   on the topic. GCS uses:                                          │
│   service-PROJECT_NUMBER@gs-project-accounts.iam.gserviceaccount │
│   Grant: roles/pubsub.publisher on the topic                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Eventarc Integration

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENTARC (NEWER APPROACH)                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Eventarc can trigger Cloud Run or Cloud Functions directly       │
│ from GCS events without manually managing Pub/Sub topics.        │
│                                                                       │
│ # Trigger Cloud Run on object upload                              │
│ gcloud eventarc triggers create gcs-upload-trigger \              │
│   --location=asia-south1 \                                        │
│   --destination-run-service=image-processor \                     │
│   --destination-run-region=asia-south1 \                          │
│   --event-filters="type=google.cloud.storage.object.v1.finalized"│
│   --event-filters="bucket=my-bucket" \                            │
│   --service-account=SA_EMAIL                                      │
│                                                                       │
│ # Trigger Cloud Function Gen 2 on object upload                  │
│ # (Cloud Functions Gen 2 uses Eventarc under the hood)           │
│ gcloud functions deploy process-upload \                           │
│   --gen2 \                                                         │
│   --runtime=python312 \                                            │
│   --trigger-event-filters="type=google.cloud.storage.object.v1.  │
│     finalized" \                                                  │
│   --trigger-event-filters="bucket=my-bucket"                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Object Composition

```
┌─────────────────────────────────────────────────────────────────────┐
│           OBJECT COMPOSITION                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Combine up to 32 existing objects into a single new object  │
│       without downloading/re-uploading data.                      │
│       Server-side operation — fast, no data transfer costs.      │
│                                                                       │
│ How it works:                                                       │
│ ├── Source objects must be in the SAME bucket                    │
│ ├── Up to 32 source objects per compose call                    │
│ ├── Can chain: compose 32 → compose result with 31 more → ...  │
│ ├── Result object can be up to 5 TiB                             │
│ ├── Source objects are NOT deleted (delete separately if needed) │
│ └── Atomic: The composed object appears fully or not at all     │
│                                                                       │
│ Use cases:                                                          │
│ ├── PARALLEL UPLOAD PATTERN:                                      │
│ │   Large file (10 GB) → split into 100 MB chunks              │
│ │   → upload all chunks in parallel (fast!)                     │
│ │   → compose into single object                                │
│ │                                                                 │
│ ├── LOG AGGREGATION:                                              │
│ │   Hourly log files → compose into daily log                   │
│ │                                                                 │
│ └── APPEND PATTERN:                                               │
│     Compose existing object + new data → updated object         │
│                                                                       │
│ CLI:                                                                │
│ gcloud storage objects compose \                                   │
│   gs://my-bucket/chunk-001 \                                      │
│   gs://my-bucket/chunk-002 \                                      │
│   gs://my-bucket/chunk-003 \                                      │
│   gs://my-bucket/complete-file.dat                                │
│                                                                       │
│ gsutil compose \                                                    │
│   gs://my-bucket/chunk-* \                                        │
│   gs://my-bucket/complete-file.dat                                │
│                                                                       │
│ Python:                                                             │
│ bucket = client.bucket("my-bucket")                                │
│ sources = [bucket.blob(f"chunk-{i:03d}") for i in range(1, 33)] │
│ composed = bucket.blob("complete-file.dat")                       │
│ composed.compose(sources)                                          │
│                                                                       │
│ AWS equivalent: S3 Multipart Upload (different — parts are       │
│   uploaded directly, not composed from existing objects)          │
│ Azure equivalent: Put Block List (similar concept)                │
│                                                                       │
│ ⚡ Key difference from S3: GCS compose works on EXISTING objects.│
│   S3 multipart upload is for uploading NEW data in parts.       │
│   GCS also supports multipart upload (resumable uploads) but    │
│   compose is unique — it merges existing objects.               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Storage Transfer Service

```
┌─────────────────────────────────────────────────────────────────────┐
│           STORAGE TRANSFER SERVICE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed service for bulk data transfer from various          │
│       sources into Cloud Storage (or between buckets).            │
│                                                                       │
│ Transfer sources:                                                   │
│ ├── Amazon S3 (or S3-compatible storage)                         │
│ ├── Azure Blob Storage                                            │
│ ├── Other GCS buckets (cross-project, cross-region)             │
│ ├── HTTP/HTTPS URLs (public file lists)                          │
│ ├── On-premises file systems (via Transfer Service Agent)       │
│ └── HDFS (Hadoop Distributed File System)                        │
│                                                                       │
│ Key features:                                                       │
│ ├── Scheduled & recurring transfers                              │
│ ├── Incremental: Only transfer new/changed objects               │
│ ├── Filter: Include/exclude by prefix, modification time         │
│ ├── Delete: Optionally delete source after transfer              │
│ ├── Overwrite: Control if existing objects are overwritten       │
│ ├── Bandwidth limit: Avoid saturating network                    │
│ ├── Logging: Detailed logs for each transfer                     │
│ └── Manifest: Transfer specific list of objects                  │
│                                                                       │
│ Pricing:                                                            │
│ ├── No charge for the transfer service itself!                   │
│ ├── You pay: Egress from source (AWS/Azure egress fees)         │
│ ├── You pay: GCS operations (write, Class A)                    │
│ └── You pay: GCS storage for the destination                    │
│                                                                       │
│ AWS equivalent: AWS DataSync, S3 Batch Operations                 │
│ Azure equivalent: Azure Data Box, AzCopy                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Transfer Job

```
Console → Storage Transfer Service → Create transfer job

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE TRANSFER JOB                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Source type: ○ Cloud Storage  ○ Amazon S3  ○ Azure Blob Storage  │
│              ○ HTTP/HTTPS URL list  ○ POSIX file system          │
│                                                                       │
│ === Example: S3 → GCS Migration ===                                │
│                                                                       │
│ Source:                                                              │
│   S3 bucket: [my-aws-bucket]                                      │
│   Access key ID: [AKIA...]                                         │
│   Secret access key: [...]                                         │
│   ⚡ Use an IAM user with read-only S3 access                      │
│                                                                       │
│ Destination:                                                        │
│   GCS bucket: [my-gcp-bucket]                                     │
│   Path prefix: [migrated/from-s3/]  (optional)                   │
│                                                                       │
│ Schedule:                                                           │
│   ○ Run now (one-time)                                             │
│   ○ Run daily at [02:00 AM]  starting [2026-05-17]               │
│   ○ Run weekly / custom schedule                                  │
│   ⚡ Scheduled = great for ongoing sync from S3 to GCS            │
│                                                                       │
│ Transfer options:                                                   │
│   Overwrite: ○ If different  ○ Always  ○ Never                   │
│   Delete source objects after transfer: ☐                         │
│   Delete destination objects not in source: ☐                    │
│   ⚡ "If different" = check checksum, only transfer if changed    │
│                                                                       │
│ Filter:                                                             │
│   Include prefixes: [logs/2026/]                                  │
│   Exclude prefixes: [logs/2026/debug/]                           │
│   Modified after: [2026-01-01]                                    │
│   Modified before: []                                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

CLI:
gcloud transfer jobs create \
  s3://my-aws-bucket \
  gs://my-gcp-bucket \
  --source-creds-file=aws-creds.json \
  --include-prefixes=logs/2026/ \
  --schedule-starts=2026-05-17 \
  --schedule-repeats-every=1d
```

---

## Part 6: Requester Pays

```
┌─────────────────────────────────────────────────────────────────────┐
│           REQUESTER PAYS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: The requester (data consumer) pays for data access and       │
│       transfer costs instead of the bucket owner.                 │
│                                                                       │
│ Why:                                                                │
│ ├── You share public datasets but don't want to pay egress      │
│ ├── Cross-project data sharing where consumer should pay        │
│ └── Large datasets (genomics, satellite imagery, etc.)          │
│                                                                       │
│ How:                                                                │
│ ├── Bucket owner enables Requester Pays on the bucket            │
│ ├── Requester must include their billing project in requests    │
│ ├── Requester pays: Network egress + operation costs            │
│ ├── Bucket owner pays: Storage costs only                       │
│ └── Anonymous requests are BLOCKED (must authenticate)          │
│                                                                       │
│ Enable:                                                             │
│ gcloud storage buckets update gs://my-bucket --requester-pays    │
│                                                                       │
│ Access (requester must specify billing project):                   │
│ gsutil -u BILLING_PROJECT_ID ls gs://requester-pays-bucket       │
│ gcloud storage ls gs://requester-pays-bucket \                    │
│   --billing-project=my-project                                    │
│                                                                       │
│ AWS equivalent: S3 Requester Pays                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Customer-Managed Encryption Keys (CMEK & CSEK)

```
┌─────────────────────────────────────────────────────────────────────┐
│           ENCRYPTION OPTIONS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ALL objects in GCS are encrypted at rest. Always. No exceptions.  │
│ The question is: WHO manages the encryption key?                  │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Option 1: Google-managed keys (default)                      │  │
│ │ ├── Google manages everything                               │  │
│ │ ├── AES-256 encryption                                      │  │
│ │ ├── Keys rotated automatically                              │  │
│ │ ├── No configuration needed                                 │  │
│ │ └── Free ✅                                                  │  │
│ │                                                              │  │
│ │ Option 2: CMEK (Customer-Managed Encryption Keys)           │  │
│ │ ├── You create and manage key in Cloud KMS                 │  │
│ │ ├── You control rotation, can disable/destroy key          │  │
│ │ ├── ⚡ If you destroy the key → data is UNRECOVERABLE      │  │
│ │ ├── Audit key usage via Cloud Audit Logs                   │  │
│ │ ├── Set as default on bucket or per-object                 │  │
│ │ └── Cost: KMS key cost ($0.06/month + $0.03/10K ops)      │  │
│ │                                                              │  │
│ │ Option 3: CSEK (Customer-Supplied Encryption Keys)          │  │
│ │ ├── YOU provide the raw AES-256 key with each request      │  │
│ │ ├── Google uses your key, does NOT store it                │  │
│ │ ├── ⚠️ If you lose the key → data is UNRECOVERABLE        │  │
│ │ ├── Key must be sent with every read/write request         │  │
│ │ ├── Cannot use Console (CLI/API only)                      │  │
│ │ ├── Cannot use: Pub/Sub notifications, Compose, etc.       │  │
│ │ └── Free (no KMS cost) but operationally complex          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Setting CMEK as bucket default:                                    │
│ gcloud storage buckets update gs://my-bucket \                    │
│   --default-encryption-key=projects/PROJECT/locations/LOCATION/  │
│   keyRings/KEY_RING/cryptoKeys/KEY_NAME                           │
│                                                                       │
│ ⚡ GCS service account needs roles/cloudkms.cryptoKeyEncrypter    │
│   Decrypter on the KMS key.                                      │
│                                                                       │
│ AWS equivalent: SSE-S3, SSE-KMS, SSE-C                            │
│ Azure equivalent: Microsoft-managed, Customer-managed, Customer- │
│                   provided keys                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Object Holds & Bucket Lock

```
┌─────────────────────────────────────────────────────────────────────┐
│           COMPLIANCE FEATURES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ For regulatory compliance (HIPAA, SOX, SEC Rule 17a-4, etc.)     │
│ you may need to prevent objects from being deleted or modified.   │
│                                                                       │
│ Three mechanisms:                                                   │
│                                                                       │
│ 1. RETENTION POLICY (covered in Ch20 — recap):                    │
│    ├── Minimum retention period on a bucket                      │
│    ├── Objects cannot be deleted/overwritten until period ends   │
│    └── Period: seconds to years                                  │
│                                                                       │
│ 2. BUCKET LOCK (permanent!):                                       │
│    ├── Locks the retention policy PERMANENTLY                    │
│    ├── ⚠️ CANNOT be unlocked — ever!                              │
│    ├── ⚠️ CANNOT shorten the retention period after locking       │
│    ├── Can lengthen retention period (make it stricter)         │
│    ├── Bucket cannot be deleted until ALL objects have met       │
│    │   their retention period                                    │
│    └── Required for: SEC Rule 17a-4, CFTC Rule 1.31            │
│                                                                       │
│ 3. OBJECT HOLDS:                                                    │
│    ├── Event-based hold:                                          │
│    │   ├── Set on bucket (applies to new objects automatically) │
│    │   ├── Object cannot be deleted while hold is active        │
│    │   ├── Release hold → retention period starts              │
│    │   └── Use: Legal hold, investigation hold                 │
│    │                                                              │
│    └── Temporary hold:                                            │
│        ├── Set on individual objects                             │
│        ├── Object cannot be deleted while hold is active        │
│        ├── No retention period — just prevents deletion         │
│        └── Use: Prevent accidental deletion during processing  │
│                                                                       │
│ CLI:                                                                │
│ # Lock bucket's retention policy (PERMANENT!)                    │
│ gcloud storage buckets update gs://my-bucket --lock-retention    │
│                                                                       │
│ # Set temporary hold on an object                                 │
│ gcloud storage objects update gs://my-bucket/file.txt \          │
│   --temporary-hold                                                │
│                                                                       │
│ # Release temporary hold                                          │
│ gcloud storage objects update gs://my-bucket/file.txt \          │
│   --no-temporary-hold                                             │
│                                                                       │
│ # Enable event-based hold on bucket (new objects get hold)       │
│ gcloud storage buckets update gs://my-bucket \                   │
│   --default-event-based-hold                                     │
│                                                                       │
│ AWS equivalent: S3 Object Lock (Governance/Compliance mode)      │
│ Azure equivalent: Immutable Blob Storage (time-based/legal hold) │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Batch Operations

```
┌─────────────────────────────────────────────────────────────────────┐
│           BATCH OPERATIONS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GCS supports batch requests to perform multiple operations         │
│ in a single HTTP request (API-level batching).                    │
│                                                                       │
│ Bulk operations with CLI:                                          │
│                                                                       │
│ # Move/rename all objects with a prefix                           │
│ gcloud storage mv gs://bucket/old-prefix/* gs://bucket/new-prefix/│
│                                                                       │
│ # Copy between buckets (parallel, resumable)                      │
│ gcloud storage cp -r gs://source-bucket/* gs://dest-bucket/      │
│                                                                       │
│ # Set metadata on all objects matching pattern                    │
│ gcloud storage objects update gs://bucket/**.csv \                │
│   --content-type=text/csv                                         │
│                                                                       │
│ # Delete all objects older than 30 days (use lifecycle instead)  │
│ # But for one-time cleanup:                                       │
│ gsutil -m rm gs://bucket/temp/**                                  │
│   (-m = multi-threaded, parallel operations)                     │
│                                                                       │
│ # Change storage class of all objects in a prefix                │
│ gcloud storage objects update gs://bucket/archive/** \           │
│   --storage-class=COLDLINE                                        │
│                                                                       │
│ ⚡ gsutil -m enables multi-threaded operations for bulk tasks.    │
│   gcloud storage is multi-threaded by default.                  │
│                                                                       │
│ AWS equivalent: S3 Batch Operations (managed batch job)          │
│ ⚡ GCS does NOT have an equivalent managed batch service like     │
│   S3 Batch Operations. Use CLI parallelism or client libraries. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Console Walkthrough — Advanced GCS Features

### Setting Up Object Lifecycle Rules

```
Console → Cloud Storage → Buckets → Click bucket name → LIFECYCLE tab → ADD A RULE

┌─────────────────────────────────────────────────────────────────┐
│           ADD A LIFECYCLE RULE                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Step 1: Select an action ──                                   │
│ ○ Set storage class to Nearline                                 │
│ ○ Set storage class to Coldline                                 │
│ ○ Set storage class to Archive                                  │
│ ● Delete object                                                 │
│ ○ Abort incomplete multipart upload                             │
│                                                                   │
│                                   [CONTINUE]                    │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│ ── Step 2: Select object conditions ──                           │
│                                                                   │
│ ☑ Age: [365] days                                               │
│ ☐ Created before: [date picker]                                 │
│ ☐ Custom time before: [date picker]                             │
│ ☐ Days since custom time: [number]                              │
│ ☐ Days since noncurrent time: [number]                          │
│ ☑ Storage class matches: [Standard ▼]                           │
│ ☐ Is live: ○ Yes ○ No (for versioned objects)                  │
│ ☐ Number of newer versions: [number]                            │
│ ☐ Object name matches prefix: [logs/]                          │
│                                                                   │
│ ⚡ Multiple conditions = AND logic (all must be true).           │
│ ⚡ Example: Delete Standard class objects older than 365 days.   │
│                                                                   │
│                              [CREATE]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Configuring Retention Policy & Bucket Lock

```
Console → Cloud Storage → Buckets → Click bucket → PROTECTION tab

┌─────────────────────────────────────────────────────────────────┐
│           PROTECTION                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Retention policy ──                                           │
│ Retention period: [365] days                                    │
│ ⚡ Objects cannot be deleted or overwritten before this period.  │
│                                                                   │
│ [SET RETENTION POLICY]                                           │
│                                                                   │
│ ── Lock retention policy ──                                     │
│ [LOCK]                                                           │
│ ⚠️ WARNING: Locking is PERMANENT and IRREVERSIBLE!              │
│    Once locked:                                                 │
│    ├── Retention period cannot be reduced                       │
│    ├── Bucket cannot be deleted until ALL objects expire        │
│    └── Used for regulatory compliance (SEC, FINRA)             │
│                                                                   │
│ ── Object holds ──                                              │
│ ☐ Default event-based hold (on new objects)                    │
│ ⚡ Individual objects can have holds set via object details.     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Storage Transfer Service Job

```
Console → Data Transfer → Storage Transfer Service → CREATE TRANSFER JOB

┌─────────────────────────────────────────────────────────────────┐
│           CREATE TRANSFER JOB                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Step 1: Source ──                                             │
│ Source type:                                                    │
│ ● Google Cloud Storage bucket                                   │
│ ○ Amazon S3 bucket                                              │
│ ○ Azure Blob Storage container                                  │
│ ○ URL list (HTTP/HTTPS source)                                  │
│ ○ HDFS (Hadoop Distributed File System)                         │
│ ○ POSIX file system (via agent)                                 │
│                                                                   │
│ Source bucket: [source-bucket-name ▼]                           │
│ Source path (optional): [data/exports/]                         │
│                                                                   │
│                                   [NEXT STEP]                   │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│ ── Step 2: Destination ──                                        │
│                                                                   │
│ Destination bucket: [destination-bucket-name ▼]                │
│ Destination path (optional): [imported/]                        │
│                                                                   │
│                                   [NEXT STEP]                   │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│ ── Step 3: Settings ──                                           │
│                                                                   │
│ Description: [Daily S3 to GCS sync]                             │
│                                                                   │
│ Schedule:                                                       │
│ ● Run now (one-time)                                           │
│ ○ Run daily at: [02:00 AM ▼] UTC                               │
│ ○ Run every [1 ▼] [hours ▼]                                   │
│                                                                   │
│ Start date: [today]     End date: [never / date]               │
│                                                                   │
│ ── Transfer options ──                                          │
│ ☑ Overwrite objects if different                                │
│ ☐ Delete objects from source after transfer                    │
│ ☑ Delete objects from destination if not in source             │
│ ☐ Preserve ACLs (for GCS-to-GCS only)                         │
│                                                                   │
│ ── Filter options (optional) ──                                 │
│ Include prefixes: [images/, documents/]                        │
│ Exclude prefixes: [temp/, staging/]                            │
│ Modified since: [2024-01-01]                                   │
│                                                                   │
│                              [CREATE]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Setting Bucket Permissions (Uniform vs Fine-Grained)

```
Console → Cloud Storage → Buckets → Click bucket → PERMISSIONS tab

┌─────────────────────────────────────────────────────────────────┐
│           PERMISSIONS                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Access control: Uniform  [SWITCH TO FINE-GRAINED]               │
│ ⚡ Uniform = IAM only (recommended)                              │
│   Fine-grained = IAM + ACLs (legacy)                            │
│ ⚠️ Once Uniform for 90 days → cannot switch back!               │
│                                                                   │
│ ── Grant access ──                                              │
│ [GRANT ACCESS]                                                  │
│                                                                   │
│ New principals: [allUsers]                                      │
│ Role: [Storage Object Viewer ▼]                                 │
│ ⚡ This makes bucket content PUBLIC (for static websites).      │
│                                                                   │
│ ── Current permissions ──                                       │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ Principal                    │ Role                        │  │
│ ├────────────────────────────────────────────────────────────┤  │
│ │ allUsers                     │ Storage Object Viewer       │  │
│ │ admin@techcorp.com           │ Storage Admin               │  │
│ │ backend-sa@proj.iam.gserv... │ Storage Object Creator     │  │
│ └────────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Terraform & CLI

### Terraform

```hcl
# Bucket with uniform access, CMEK, notifications
resource "google_storage_bucket" "advanced" {
  name     = "myco-prod-data-advanced"
  location = "ASIA"
  project  = var.project_id

  storage_class               = "STANDARD"
  uniform_bucket_level_access = true   # Uniform access (no ACLs)

  # Retention policy (7 years for compliance)
  retention_policy {
    retention_period = 220752000   # 7 years in seconds
    is_locked        = false       # Set true to PERMANENTLY lock
  }

  # Default event-based hold
  default_event_based_hold = false

  # CMEK encryption
  encryption {
    default_kms_key_name = google_kms_crypto_key.storage_key.id
  }

  # Versioning
  versioning {
    enabled = true
  }

  # Soft delete (recovery window)
  soft_delete_policy {
    retention_duration_seconds = 604800   # 7 days
  }

  labels = {
    environment = "prod"
    team        = "data"
    managed-by  = "terraform"
  }
}

# IAM binding — grant read access to a group
resource "google_storage_bucket_iam_member" "viewer" {
  bucket = google_storage_bucket.advanced.name
  role   = "roles/storage.objectViewer"
  member = "group:data-analysts@company.com"
}

# IAM binding with condition (folder-level access)
resource "google_storage_bucket_iam_member" "folder_access" {
  bucket = google_storage_bucket.advanced.name
  role   = "roles/storage.objectViewer"
  member = "group:marketing@company.com"

  condition {
    title      = "marketing-folder-only"
    expression = "resource.name.startsWith(\"projects/_/buckets/${google_storage_bucket.advanced.name}/objects/marketing/\")"
  }
}

# Pub/Sub notification
resource "google_storage_notification" "upload_notify" {
  bucket         = google_storage_bucket.advanced.name
  payload_format = "JSON_API_V1"
  topic          = google_pubsub_topic.gcs_notifications.id
  event_types    = ["OBJECT_FINALIZE", "OBJECT_DELETE"]

  custom_attributes = {
    source = "gcs-bucket"
  }

  depends_on = [google_pubsub_topic_iam_member.gcs_publisher]
}

# Allow GCS to publish to Pub/Sub
resource "google_pubsub_topic" "gcs_notifications" {
  name = "gcs-upload-notifications"
}

data "google_storage_project_service_account" "gcs" {}

resource "google_pubsub_topic_iam_member" "gcs_publisher" {
  topic  = google_pubsub_topic.gcs_notifications.id
  role   = "roles/pubsub.publisher"
  member = "serviceAccount:${data.google_storage_project_service_account.gcs.email_address}"
}
```

### gcloud CLI Reference

```bash
# === Access Control ===
# Check bucket access control mode
gcloud storage buckets describe gs://my-bucket --format="value(iamConfiguration.uniformBucketLevelAccess.enabled)"

# Enable uniform access
gcloud storage buckets update gs://my-bucket --uniform-bucket-level-access

# Grant IAM role on bucket
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=user:john@company.com \
  --role=roles/storage.objectViewer

# === Signed URLs ===
# Generate download URL (1 hour)
gcloud storage sign-url gs://my-bucket/file.pdf \
  --private-key-file=sa-key.json \
  --duration=1h

# === Notifications ===
# Create notification
gcloud storage buckets notifications create gs://my-bucket \
  --topic=projects/my-project/topics/my-topic \
  --event-types=OBJECT_FINALIZE

# List notifications
gcloud storage buckets notifications list gs://my-bucket

# Delete notification
gcloud storage buckets notifications delete gs://my-bucket \
  --notification-id=1

# === Compose ===
gcloud storage objects compose \
  gs://my-bucket/part-001 gs://my-bucket/part-002 gs://my-bucket/part-003 \
  gs://my-bucket/combined-file

# === Transfer Service ===
# Create transfer job (S3 → GCS)
gcloud transfer jobs create \
  s3://source-bucket gs://dest-bucket \
  --source-creds-file=aws-creds.json

# List transfer jobs
gcloud transfer jobs list

# Monitor transfer
gcloud transfer jobs monitor JOB_NAME

# === Requester Pays ===
gcloud storage buckets update gs://my-bucket --requester-pays

# === Holds ===
gcloud storage objects update gs://my-bucket/file.txt --temporary-hold
gcloud storage objects update gs://my-bucket/file.txt --no-temporary-hold

# === Encryption (CMEK) ===
gcloud storage buckets update gs://my-bucket \
  --default-encryption-key=projects/proj/locations/loc/keyRings/kr/cryptoKeys/key
```

---

## Part 11: Real-World Patterns

### Startup

```
┌─────────────────────────────────────────────────────────────────────┐
│ STARTUP (5-10 developers)                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Access:                                                              │
│ ├── Uniform bucket-level access on all buckets                   │
│ ├── IAM roles at project level (simple, team is small)           │
│ └── Signed URLs for user file downloads                          │
│                                                                       │
│ Notifications:                                                      │
│ ├── GCS → Pub/Sub → Cloud Function (image resize)              │
│ └── Simple event-driven processing                               │
│                                                                       │
│ Encryption: Google-managed (default, free)                        │
│                                                                       │
│ No Transfer Service needed (small data volumes)                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Mid-Size

```
┌─────────────────────────────────────────────────────────────────────┐
│ MID-SIZE (50-100 developers)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Access:                                                              │
│ ├── Uniform access on all buckets                                │
│ ├── IAM roles at bucket level (team-specific access)            │
│ ├── IAM Conditions for folder-level restrictions                │
│ ├── Signed URLs for external partner file sharing               │
│ └── Org policy: storage.publicAccessPrevention = enforced       │
│                                                                       │
│ Notifications:                                                      │
│ ├── Eventarc → Cloud Run (processing pipeline)                  │
│ ├── Multiple notification topics per bucket                      │
│ └── Dead-letter topics for failed processing                    │
│                                                                       │
│ Encryption: CMEK for sensitive data buckets                      │
│                                                                       │
│ Transfer: Storage Transfer Service for                            │
│ ├── S3 → GCS migration (one-time)                               │
│ └── Cross-region replication (scheduled)                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Enterprise

```
┌─────────────────────────────────────────────────────────────────────┐
│ ENTERPRISE (500+ developers)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Access:                                                              │
│ ├── Uniform access enforced via Org Policy                       │
│ ├── VPC Service Controls around storage buckets                  │
│ ├── IAM Conditions for multi-tenant folder isolation            │
│ ├── Signed URLs with short expiration (15 min)                  │
│ └── Data access audit logs enabled                               │
│                                                                       │
│ Notifications:                                                      │
│ ├── Eventarc → Cloud Run processing pipeline                    │
│ ├── Cross-project notification routing                           │
│ └── DLP scanning on upload (via Cloud Function)                 │
│                                                                       │
│ Encryption:                                                         │
│ ├── CMEK on all production buckets (mandatory)                  │
│ ├── Separate KMS keys per team/data classification             │
│ └── Key rotation every 90 days                                   │
│                                                                       │
│ Compliance:                                                         │
│ ├── Retention policies with Bucket Lock                          │
│ ├── Event-based holds for legal/regulatory                      │
│ └── Audit trail via Cloud Audit Logs                             │
│                                                                       │
│ Transfer:                                                           │
│ ├── Multi-cloud sync (S3 ↔ GCS daily)                          │
│ ├── On-prem → GCS via Transfer Service Agent                   │
│ └── Transfer Appliance for petabyte-scale migrations            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌──────────────────────────────────────────────────────────────────────┐
│ CLOUD STORAGE ADVANCED — QUICK REFERENCE                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│ Access Control:                                                       │
│ ├── Uniform (IAM only) ← recommended                               │
│ ├── Fine-grained (IAM + ACLs) ← legacy                             │
│ └── Org Policy: storage.uniformBucketLevelAccess = True             │
│                                                                        │
│ Signed URLs:                                                          │
│ ├── V4 signing, max 7 days                                           │
│ ├── GET (download), PUT (upload), DELETE, HEAD                       │
│ └── No Google account needed for access                              │
│                                                                        │
│ Notifications: GCS → Pub/Sub or Eventarc                              │
│ Compose: Up to 32 objects → 1 object (server-side)                   │
│ Transfer Service: S3/Azure/HTTP/on-prem → GCS                        │
│ Requester Pays: Consumer pays egress + operations                    │
│                                                                        │
│ Encryption:                                                           │
│ ├── Google-managed (default, free)                                   │
│ ├── CMEK (Cloud KMS key, you control rotation)                      │
│ └── CSEK (you supply raw key per request)                            │
│                                                                        │
│ Compliance:                                                           │
│ ├── Retention policy (min hold period)                               │
│ ├── Bucket Lock (permanent retention — IRREVERSIBLE)                │
│ ├── Temporary hold (prevent deletion)                                │
│ └── Event-based hold (hold until released)                           │
│                                                                        │
│ AWS ↔ GCP mapping:                                                     │
│ ├── S3 Bucket Policy → IAM policy on bucket                         │
│ ├── S3 ACLs → GCS ACLs (both deprecated)                            │
│ ├── S3 Pre-signed URL → Signed URL                                   │
│ ├── S3 Event Notifications → Pub/Sub Notifications                   │
│ ├── S3 Batch Operations → gcloud storage (CLI parallelism)          │
│ ├── SSE-KMS → CMEK                                                   │
│ ├── SSE-C → CSEK                                                     │
│ ├── S3 Object Lock → Retention + Bucket Lock                        │
│ └── AWS DataSync → Storage Transfer Service                          │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Cloud Storage FUSE

Cloud Storage FUSE lets you **mount a GCS bucket as a local filesystem** on a Linux VM or GKE pod. Instead of using `gsutil` or the storage API, you read and write files using normal file operations (`cat`, `cp`, `ls`, etc.).

### What Is It?

```
┌──────────────────────────────────────────────────────────────┐
│                    Cloud Storage FUSE                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Your VM / GKE Pod                                            │
│  ┌────────────────────────┐                                   │
│  │ Application reads/writes│                                  │
│  │ /mnt/gcs-bucket/        │ ← looks like a local directory  │
│  └────────┬───────────────┘                                   │
│           │ (FUSE)                                             │
│  ┌────────▼───────────────┐                                   │
│  │ gcsfuse daemon          │ ← translates file ops to API    │
│  └────────┬───────────────┘                                   │
│           │ (GCS JSON API)                                     │
│  ┌────────▼───────────────┐                                   │
│  │ Cloud Storage bucket    │                                  │
│  └────────────────────────┘                                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

**Key concept:** GCS is an object store, not a filesystem. Cloud Storage FUSE bridges the gap by translating file operations into GCS API calls. It's not a full POSIX filesystem — it's a convenience layer.

### Installation

```bash
# On Debian/Ubuntu VMs
export GCSFUSE_REPO=gcsfuse-$(lsb_release -c -s)
echo "deb [signed-by=/usr/share/keyrings/cloud.google.asc] https://packages.cloud.google.com/apt $GCSFUSE_REPO main" \
    | sudo tee /etc/apt/sources.list.d/gcsfuse.list
sudo apt-get update && sudo apt-get install -y gcsfuse

# Mount a bucket
mkdir -p /mnt/my-bucket
gcsfuse my-bucket-name /mnt/my-bucket

# Verify
ls /mnt/my-bucket/

# Unmount
fusermount -u /mnt/my-bucket
```

**On GKE:** Use the [GCS FUSE CSI driver](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/cloud-storage-fuse-csi-driver) — it's built into GKE and mounts buckets as volumes in your pods:

```yaml
# GKE Pod spec with Cloud Storage FUSE volume
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
  annotations:
    gke-gcsfuse/volumes: "true"  # Enable the CSI driver
spec:
  containers:
  - name: trainer
    image: my-ml-image
    volumeMounts:
    - name: training-data
      mountPath: /data
  volumes:
  - name: training-data
    csi:
      driver: gcsfuse.csi.storage.gke.io
      volumeAttributes:
        bucketName: my-training-data-bucket
```

### Use Cases

| Use Case | Why FUSE? |
|----------|----------|
| **ML training on GKE/VMs** | Training scripts expect local file paths — FUSE lets them read datasets from GCS without code changes |
| **Data science notebooks** | Mount datasets as local files in Jupyter — no SDK needed |
| **Legacy applications** | Apps that only read from local disk can access GCS without modification |
| **Shared datasets** | Multiple VMs/pods mount the same bucket — changes visible to all |
| **Log aggregation** | Write logs to a mounted bucket — automatic cloud storage without agents |

### Limitations

Cloud Storage FUSE is **not a replacement for a real filesystem**. Be aware of these limitations:

| Limitation | Details |
|------------|--------|
| **Not POSIX-compliant** | No file locking, no hard links, no atomic renames across directories |
| **Higher latency** | Each file operation is an API call — slower than local SSD or Persistent Disk |
| **No random writes** | Modifying a file rewrites the entire object (fine for read-heavy workloads) |
| **Eventual consistency (rare)** | List operations may not immediately reflect new objects |
| **Metadata operations are slow** | `ls` on large directories (100K+ files) can be very slow |
| **Not for databases** | Never mount a FUSE volume for database storage (use Persistent Disk instead) |

> **💡 When to use FUSE vs. other options:**
> - **Read-heavy ML workloads** → FUSE is great (sequential reads of large files)
> - **Frequent small writes** → Use Persistent Disk or Filestore instead
> - **Database storage** → Always use Persistent Disk
> - **High-throughput sequential access** → FUSE with file caching enabled works well

### Performance Tips

```bash
# Enable file caching for read-heavy workloads
gcsfuse --file-cache-capacity-mb=1024 \
        --stat-cache-ttl=60s \
        --type-cache-ttl=60s \
        my-bucket /mnt/my-bucket

# Enable parallel downloads for large files
gcsfuse --parallel-downloads-per-file=16 \
        my-bucket /mnt/my-bucket
```

---

## What's Next?

Continue to **Chapter 22: Persistent Disk & Local SSD** → `22-persistent-disk.md`
