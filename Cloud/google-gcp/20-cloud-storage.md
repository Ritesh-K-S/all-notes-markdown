# Chapter 20: Cloud Storage (GCS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Storage Fundamentals](#part-1-cloud-storage-fundamentals)
- [Part 2: Creating a Bucket (Full Portal Walkthrough)](#part-2-creating-a-bucket-full-portal-walkthrough)
- [Part 3: Uploading & Managing Objects](#part-3-uploading--managing-objects)
- [Part 4: Storage Classes Deep Dive](#part-4-storage-classes-deep-dive)
- [Part 5: Object Lifecycle Management](#part-5-object-lifecycle-management)
- [Part 6: Object Versioning](#part-6-object-versioning)
- [Part 7: Retention Policies & Object Holds](#part-7-retention-policies--object-holds)
- [Part 8: Public Access & Website Hosting](#part-8-public-access--website-hosting)
- [Part 9: Terraform & CLI](#part-9-terraform--cli)
- [Part 10: Real-World Patterns](#part-10-real-world-patterns)
- [Part 11: Console Walkthrough: Managing & Deleting Buckets](#part-11-console-walkthrough-managing--deleting-buckets)
- [Part 12: CORS Configuration](#part-12-cors-configuration)
- [Part 13: Cost Calculation Example](#part-13-cost-calculation-example)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud Storage (GCS) is a globally unified, scalable object storage service for any amount of data. You store files (objects) in buckets, choose storage classes for cost optimization, and configure lifecycle rules to automatically manage data over time. It serves everything from website assets and backups to data lake storage for analytics.

```
What you'll learn:
├── Cloud Storage Fundamentals
│   ├── What & why (object storage, globally unified)
│   ├── Buckets, objects, folders (flat namespace)
│   ├── Storage classes & pricing
│   └── Consistency model
├── Creating a Bucket (Full Portal Walkthrough)
│   ├── Name & location
│   ├── Storage class
│   ├── Access control (Uniform vs Fine-grained)
│   ├── Protection (Versioning, Retention, Soft delete)
│   ├── Encryption (Google-managed vs CMEK)
│   └── Labels
├── Uploading & Managing Objects
│   ├── Upload (Console, gsutil, client libraries)
│   ├── Object metadata & custom metadata
│   ├── Content-Type, Cache-Control, Content-Encoding
│   └── Folders (prefixes, not real directories)
├── Storage Classes Deep Dive
│   ├── Standard (hot, frequent access)
│   ├── Nearline (30-day minimum)
│   ├── Coldline (90-day minimum)
│   ├── Archive (365-day minimum)
│   └── Autoclass (automatic tier management)
├── Object Lifecycle Management
│   ├── Rules (delete, set storage class, abort multipart)
│   ├── Conditions (age, created before, matches prefix, etc.)
│   └── Common patterns
├── Object Versioning
├── Retention Policies & Object Holds
├── Bucket Lock (compliance)
├── Requester Pays
├── Public Access & Website Hosting
├── Terraform & gsutil / gcloud CLI
├── Real-world patterns
├── Console Walkthrough: Managing & Deleting Buckets
├── CORS Configuration
└── Cost Calculation Example
```

---

## Part 1: Cloud Storage Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD STORAGE CONCEPT                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Cloud Storage?                                              │
│ ├── Object storage service (store files/blobs, not file systems) │
│ ├── Globally unified (single namespace, worldwide access)        │
│ ├── 99.999999999% (11 nines) durability                          │
│ ├── 99.95% availability (Standard, multi-region)                │
│ ├── Auto-replication (multi-region: geo-redundant)              │
│ ├── No provisioning — pay for what you store + access           │
│ ├── Unlimited storage (no bucket size limits!)                   │
│ ├── Max object size: 5 TiB (single object!)                    │
│ └── Flat namespace (no real directories, "/" in name = prefix)  │
│                                                                       │
│ Key concepts:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Bucket: Container for objects                                │  │
│ │ ├── Globally unique name (across ALL GCP accounts!)         │  │
│ │ ├── Has a location (region, dual-region, multi-region)     │  │
│ │ ├── Has a default storage class                            │  │
│ │ ├── Has access control settings (Uniform or Fine-grained) │  │
│ │ └── Has protection settings (versioning, retention)        │  │
│ │                                                              │  │
│ │ Object: A file stored in a bucket                           │  │
│ │ ├── Name (key): "photos/2024/sunset.jpg"                  │  │
│ │ ├── Data: The actual file bytes                            │  │
│ │ ├── Metadata: Content-Type, custom metadata, etc.          │  │
│ │ └── Generation: Version number (if versioning enabled)     │  │
│ │                                                              │  │
│ │ URL:                                                         │  │
│ │ ├── gs://my-bucket/photos/sunset.jpg (gsutil format)      │  │
│ │ ├── https://storage.googleapis.com/my-bucket/photos/...   │  │
│ │ └── https://storage.cloud.google.com/my-bucket/photos/... │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Consistency:                                                        │
│ ├── Strong global consistency for all operations ✅               │
│ ├── Read-after-write: Upload → immediately readable             │
│ ├── Read-after-delete: Delete → immediately gone                │
│ ├── List-after-write: Upload → immediately listed               │
│ └── ⚡ Unlike S3 (which had eventual consistency until 2020),     │
│     GCS has always been strongly consistent.                    │
│                                                                       │
│ Storage classes:                                                    │
│ ┌────────────────┬───────────┬──────────────┬───────────────────┐│
│ │ Class           │ Min hold  │ Storage/GB/mo│ Best for           ││
│ ├────────────────┼───────────┼──────────────┼───────────────────┤│
│ │ Standard        │ None      │ $0.020-0.026│ Hot, frequent      ││
│ │ Nearline        │ 30 days   │ $0.010-0.016│ Monthly access     ││
│ │ Coldline        │ 90 days   │ $0.004-0.006│ Quarterly access   ││
│ │ Archive         │ 365 days  │ $0.0012-0.002│ Yearly/never     ││
│ └────────────────┴───────────┴──────────────┴───────────────────┘│
│                                                                       │
│ Retrieval costs (per GB read):                                     │
│ ├── Standard: Free                                                │
│ ├── Nearline: $0.01                                               │
│ ├── Coldline: $0.02                                               │
│ └── Archive: $0.05                                                │
│                                                                       │
│ Operations costs (per 10K):                                        │
│ ├── Standard: $0.05 (Class A), $0.004 (Class B)                │
│ ├── Nearline: $0.10, $0.01                                       │
│ ├── Coldline: $0.10, $0.05                                       │
│ └── Archive: $0.50, $0.50                                        │
│                                                                       │
│ ⚡ Class A operations: Write/list (insert, list, update metadata) │
│   Class B operations: Read (get object, get metadata)           │
│                                                                       │
│ Location types:                                                     │
│ ├── Region: Single region (asia-south1). Cheapest.              │
│ │   Use for: Compute co-location, latency-sensitive.           │
│ ├── Dual-region: Two specific regions (asia-south1 + asia-south2)│
│ │   Use for: HA with controlled region pair.                   │
│ ├── Multi-region: Continent-wide (ASIA, US, EU).               │
│ │   Use for: Global access, highest availability.             │
│ └── ⚡ Multi-region = auto-replicated across regions in that area.│
│     Most expensive but most available.                         │
│                                                                       │
│ ⚡ AWS equivalent: Amazon S3                                        │
│ ⚡ Azure equivalent: Azure Blob Storage                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Bucket (Full Portal Walkthrough)

```
Console → Cloud Storage → Buckets → Create

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 1: NAME YOUR BUCKET                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Bucket name: [my-company-prod-assets]                               │
│ ⚡ GLOBALLY unique across all GCP accounts!                         │
│   Naming rules:                                                     │
│   ├── 3-63 characters                                              │
│   ├── Lowercase letters, numbers, hyphens, underscores, dots     │
│   ├── Must start and end with letter or number                   │
│   ├── Cannot be an IP address (e.g., 192.168.0.1)              │
│   ├── Cannot start with "goog" or contain "google"              │
│   └── ⚡ Best practice: Include project/env in name               │
│       my-company-prod-assets, my-company-dev-uploads            │
│                                                                       │
│ ⚠️ Bucket names are PERMANENT. Cannot rename a bucket!              │
│   Must delete and recreate with new name.                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 2: CHOOSE WHERE TO STORE YOUR DATA                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Location type:                                                       │
│                                                                       │
│ ○ Region (lowest latency, single region)                           │
│   Region: [asia-south1 (Mumbai) ▼]                                │
│   ⚡ Data stored in ONE region only.                                 │
│     Cheapest. Best when compute is in same region.              │
│     ⚠️ If region goes down → data unavailable (but not lost).     │
│                                                                       │
│ ○ Dual-region (high availability, two specific regions)           │
│   Regions: [asia-south1 + asia-south2 ▼]                         │
│   ⚡ Data replicated across exactly 2 regions you choose.          │
│     Turbo replication: RPO < 15 minutes (optional, extra cost).  │
│     ~1.8x cost of single region.                                │
│                                                                       │
│ ● Multi-region (highest availability, continent-level) ✅          │
│   Multi-region: [asia ▼]                                           │
│   Options: US, EU, ASIA                                            │
│   ⚡ Data replicated across multiple regions in the continent.     │
│     Best for globally accessed content (website assets, CDN).   │
│     ~2x cost of single region.                                  │
│                                                                       │
│ ⚡ Cannot change location after bucket creation!                    │
│   Must delete and recreate.                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 3: CHOOSE A DEFAULT STORAGE CLASS                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ☐ Set a default class                                              │
│                                                                       │
│ ○ Standard (best for frequently accessed data) ✅                  │
│   No minimum storage duration. No retrieval cost.               │
│   Use for: Website assets, streaming media, data in use.       │
│                                                                       │
│ ○ Nearline (best for infrequently accessed data)                 │
│   30-day minimum. $0.01/GB retrieval.                           │
│   Use for: Backups accessed monthly, long-tail content.        │
│                                                                       │
│ ○ Coldline (best for rarely accessed data)                       │
│   90-day minimum. $0.02/GB retrieval.                           │
│   Use for: Disaster recovery, quarterly reports.               │
│                                                                       │
│ ○ Archive (best for archiving and online backup)                 │
│   365-day minimum. $0.05/GB retrieval.                          │
│   Use for: Regulatory archives, long-term backup.              │
│                                                                       │
│ ☑ Autoclass                                                        │
│   ⚡ Autoclass automatically transitions objects between classes!  │
│     Frequently accessed → Standard                              │
│     30 days no access → Nearline                                │
│     90 days no access → Coldline                                │
│     365 days no access → Archive                                │
│     Accessed again → back to Standard                           │
│     No lifecycle rules needed! GCS does it for you.            │
│     Extra: $0.0025/1000 objects/month management fee.          │
│     ⚡ Best when you don't know access patterns!                  │
│                                                                       │
│ ⚡ Default class applies to new objects unless overridden.         │
│   Each object can have its own class (mix in one bucket).      │
│                                                                       │
│ ⚡ Early deletion charge:                                           │
│   Delete Nearline object at day 15 → charged for 30 days.      │
│   Delete Coldline object at day 30 → charged for 90 days.      │
│   Delete Archive object at day 100 → charged for 365 days.    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 4: CHOOSE HOW TO CONTROL ACCESS                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ☑ Enforce public access prevention                                 │
│ ⚡ Prevents making any object public. Good for sensitive data!     │
│   If enabled, you can NOT make objects public via IAM or ACLs.  │
│                                                                       │
│ Access control:                                                      │
│ ● Uniform (recommended) ✅                                         │
│   All objects in bucket controlled by bucket-level IAM only.    │
│   No per-object ACLs. Simpler, more secure.                    │
│   ⚡ Use this for all new buckets!                                  │
│                                                                       │
│ ○ Fine-grained                                                     │
│   Bucket-level IAM + per-object ACLs.                          │
│   Legacy mode. More complex. Use only if you need per-object  │
│   permissions (rare).                                           │
│                                                                       │
│ ⚡ Uniform → Fine-grained: Possible (but not recommended).       │
│   Fine-grained → Uniform: Possible after 90 days of Uniform.  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 5: CHOOSE HOW TO PROTECT OBJECT DATA                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Soft delete ──                                                   │
│ Soft delete policy:                                                 │
│ ● Enable (default) ✅                                               │
│ ○ Disable                                                          │
│ Retention duration: [7] days (default 7, max 90)                  │
│ ⚡ Deleted objects are RECOVERABLE for this many days.              │
│   Acts like a recycle bin. Protects against accidental deletes. │
│   Soft-deleted data IS charged at storage rate.                 │
│                                                                       │
│ ── Object versioning ──                                            │
│ ☐ Enable                                                           │
│ ⚡ When enabled, overwriting/deleting creates a new version.       │
│   Old versions preserved. You can restore any previous version. │
│   ⚠️ Each version costs storage! Use lifecycle rules to clean up.  │
│   Detailed in Part 6 below.                                     │
│                                                                       │
│ ── Retention policy ──                                             │
│ ☐ Set a retention policy                                          │
│ Duration: [___] days                                               │
│ ⚡ Objects CANNOT be deleted or overwritten until retention expires.│
│   For compliance: HIPAA, PCI, regulatory data retention.        │
│   ⚠️ Cannot shorten or remove once LOCKED!                         │
│   Detailed in Part 7 below.                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 6: CHOOSE HOW TO ENCRYPT OBJECT DATA                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ● Google-managed encryption key (default) ✅                       │
│   Google manages keys. AES-256. No cost. No config.             │
│   ⚡ All GCS data is ALWAYS encrypted at rest. You can't disable.  │
│                                                                       │
│ ○ Customer-managed encryption key (CMEK)                          │
│   Cloud KMS key: [projects/my-project/locations/global/         │
│                    keyRings/my-ring/cryptoKeys/my-key ▼]       │
│   ⚡ You control the key. You can disable/destroy it to render   │
│     data unreadable. Required for some compliance scenarios.   │
│   ⚠️ KMS key costs: $0.06/month per key version + $0.03/10K ops. │
│                                                                       │
│ ○ Customer-supplied encryption key (CSEK)                         │
│   You provide the key with each API call.                      │
│   Google never stores your key. Maximum control.               │
│   ⚠️ If you lose the key, data is PERMANENTLY unrecoverable!      │
│   ⚡ Rarely used. Most use Google-managed or CMEK.                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 7: ADDITIONAL (Labels)                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Labels:                                                              │
│ [+ Add label]                                                        │
│ ┌─────────────┬──────────────┐                                     │
│ │ Key          │ Value         │                                     │
│ ├─────────────┼──────────────┤                                     │
│ │ environment  │ production    │                                     │
│ │ team         │ backend       │                                     │
│ │ cost-center  │ engineering   │                                     │
│ └─────────────┴──────────────┘                                     │
│ ⚡ Labels for organizing, filtering, and cost allocation.           │
│   Up to 64 labels per bucket.                                    │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Uploading & Managing Objects

```
┌─────────────────────────────────────────────────────────────────────┐
│           OBJECT MANAGEMENT                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Upload methods:                                                      │
│                                                                       │
│ 1. Console: Bucket → Upload files / Upload folder                 │
│    Simple drag-and-drop. Good for small files.                   │
│                                                                       │
│ 2. gsutil (CLI):                                                    │
│    # Upload single file                                            │
│    gsutil cp photo.jpg gs://my-bucket/photos/                    │
│                                                                       │
│    # Upload directory (recursive)                                  │
│    gsutil -m cp -r ./uploads/ gs://my-bucket/uploads/            │
│    ⚡ -m = multi-threaded (much faster for many files!)             │
│                                                                       │
│    # Sync directory (like rsync, only uploads changes)           │
│    gsutil -m rsync -r ./local-dir/ gs://my-bucket/remote-dir/   │
│    ⚡ rsync = Only uploads new/changed files. Saves bandwidth.     │
│                                                                       │
│ 3. gcloud storage (newer CLI, replacing gsutil):                  │
│    gcloud storage cp photo.jpg gs://my-bucket/photos/            │
│    gcloud storage cp -r ./uploads/ gs://my-bucket/uploads/      │
│    gcloud storage rsync ./local/ gs://my-bucket/remote/          │
│                                                                       │
│ 4. Client libraries (Node.js, Python, Go, Java, etc.):          │
│    // Node.js example                                              │
│    const {Storage} = require('@google-cloud/storage');            │
│    const storage = new Storage();                                  │
│    await storage.bucket('my-bucket')                              │
│      .upload('photo.jpg', { destination: 'photos/photo.jpg' }); │
│                                                                       │
│ 5. Resumable uploads (large files):                               │
│    ⚡ Files > 5 MB should use resumable uploads.                    │
│      If upload fails, resume from where it stopped.             │
│      gsutil handles this automatically for files > 8 MiB.      │
│                                                                       │
│ 6. Parallel composite uploads (very large files):                │
│    gsutil -o GSUtil:parallel_composite_upload_threshold=150M \   │
│      cp huge-file.tar.gz gs://my-bucket/                         │
│    ⚡ Splits file into chunks → uploads in parallel → composes.   │
│                                                                       │
│ ── Object metadata ──                                              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Fixed metadata (set by GCS):                                 │  │
│ │ ├── name: photos/sunset.jpg                                 │  │
│ │ ├── bucket: my-bucket                                       │  │
│ │ ├── generation: 1234567890 (version number)                │  │
│ │ ├── metageneration: 1 (metadata version)                   │  │
│ │ ├── size: 2048576 (bytes)                                   │  │
│ │ ├── md5Hash: abc123...                                      │  │
│ │ ├── crc32c: def456...                                       │  │
│ │ ├── storageClass: STANDARD                                  │  │
│ │ ├── timeCreated: 2026-05-16T10:30:00Z                     │  │
│ │ └── updated: 2026-05-16T10:30:00Z                          │  │
│ │                                                              │  │
│ │ Editable metadata:                                           │  │
│ │ ├── Content-Type: image/jpeg                               │  │
│ │ │   ⚡ Auto-detected on upload. Override if wrong.           │  │
│ │ ├── Content-Encoding: gzip                                  │  │
│ │ │   ⚡ Set when uploading pre-compressed files.              │  │
│ │ ├── Content-Disposition: attachment; filename="report.pdf" │  │
│ │ │   ⚡ Forces download instead of inline display.            │  │
│ │ ├── Cache-Control: public, max-age=3600                    │  │
│ │ │   ⚡ Controls CDN and browser caching.                     │  │
│ │ │   "no-cache": Always revalidate.                        │  │
│ │ │   "max-age=3600": Cache 1 hour.                         │  │
│ │ └── Custom metadata: x-goog-meta-*                        │  │
│ │     # Set custom metadata                                   │  │
│ │     gsutil setmeta -h "x-goog-meta-uploaded-by:ritesh" \  │  │
│ │       gs://my-bucket/photo.jpg                             │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Folders (they're not real!):                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ GCS is a flat namespace. "/" in object names = prefix.      │  │
│ │                                                              │  │
│ │ Object name: "photos/2024/vacation/sunset.jpg"             │  │
│ │ ├── This is ONE object with a long name                    │  │
│ │ ├── Console SHOWS it as: photos/ → 2024/ → vacation/     │  │
│ │ └── But there's no "photos" directory — just a prefix     │  │
│ │                                                              │  │
│ │ Console creates "folder placeholder" objects:               │  │
│ │ "photos/" (0-byte object) when you "Create folder"        │  │
│ │ But objects don't need folder placeholders to exist.       │  │
│ │                                                              │  │
│ │ ⚡ Listing with prefix:                                      │  │
│ │   gsutil ls gs://my-bucket/photos/2024/                    │  │
│ │   → Lists all objects with prefix "photos/2024/"          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Storage Classes Deep Dive

```
┌─────────────────────────────────────────────────────────────────────┐
│           STORAGE CLASSES                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Standard:                                                            │
│ ├── No minimum storage duration                                   │
│ ├── No retrieval cost                                              │
│ ├── Highest storage cost ($0.020-0.026/GB/month)                │
│ ├── SLA: 99.95% (multi-region), 99.9% (region)                │
│ └── Use for: Frequently accessed data, website content,         │
│     app assets, streaming, data in active use.                │
│                                                                       │
│ Nearline:                                                            │
│ ├── 30-day minimum (early deletion charged)                     │
│ ├── $0.01/GB retrieval cost                                      │
│ ├── ~50% cheaper storage ($0.010-0.016/GB/month)               │
│ ├── SLA: 99.95% (multi-region), 99.9% (region)                │
│ └── Use for: Monthly backups, long-tail content (old blog posts),│
│     infrequently used data that still needs quick access.     │
│                                                                       │
│ Coldline:                                                            │
│ ├── 90-day minimum (early deletion charged)                     │
│ ├── $0.02/GB retrieval cost                                      │
│ ├── ~80% cheaper storage ($0.004-0.006/GB/month)               │
│ ├── SLA: 99.95% (multi-region), 99.9% (region)                │
│ └── Use for: Disaster recovery data, quarterly accessed data,  │
│     compliance archives that may need occasional access.      │
│                                                                       │
│ Archive:                                                             │
│ ├── 365-day minimum (early deletion charged)                    │
│ ├── $0.05/GB retrieval cost                                      │
│ ├── ~95% cheaper storage ($0.0012-0.002/GB/month)              │
│ ├── SLA: No SLA (same availability, but no SLA guarantee)     │
│ └── Use for: Regulatory archives (7-year retention),           │
│     cold backups, rarely or never accessed data.              │
│                                                                       │
│ ⚡ Key difference from S3 Glacier:                                   │
│   GCS Coldline/Archive = INSTANT access (milliseconds!)         │
│   S3 Glacier = Hours to retrieve (unless Instant Retrieval)     │
│   All GCS classes have the same access latency!                │
│                                                                       │
│ Cost comparison (1 TB, asia-south1, region):                      │
│ ┌────────────────┬───────────┬──────────────┬──────────────────┐ │
│ │ Class           │ Storage/mo│ 1 GB retrieve│ Total (1 read/mo)│ │
│ ├────────────────┼───────────┼──────────────┼──────────────────┤ │
│ │ Standard        │ $23.00    │ $0.00        │ $23.00           │ │
│ │ Nearline        │ $13.00    │ $0.01        │ $13.01           │ │
│ │ Coldline        │ $6.00     │ $0.02        │ $6.02            │ │
│ │ Archive         │ $2.50     │ $0.05        │ $2.55            │ │
│ └────────────────┴───────────┴──────────────┴──────────────────┘ │
│                                                                       │
│ Autoclass:                                                          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Bucket → Settings → Autoclass: Enable                      │  │
│ │                                                              │  │
│ │ How it works:                                                │  │
│ │ Object uploaded → Standard                                  │  │
│ │     ↓ 30 days no access                                     │  │
│ │ Moved to → Nearline                                         │  │
│ │     ↓ 90 days no access                                     │  │
│ │ Moved to → Coldline                                         │  │
│ │     ↓ 365 days no access                                    │  │
│ │ Moved to → Archive                                          │  │
│ │     ↓ accessed again                                        │  │
│ │ Moved back to → Standard                                    │  │
│ │                                                              │  │
│ │ ⚡ No lifecycle rules needed! GCS manages transitions.       │  │
│ │   Small management fee: $0.0025/1000 objects/month.        │  │
│ │   No early deletion charges with Autoclass!                │  │
│ │   Best when access patterns are unpredictable.             │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Change storage class:                                               │
│ # Per object                                                       │
│ gsutil rewrite -s nearline gs://my-bucket/old-logs/*              │
│                                                                       │
│ # Or via lifecycle rule (automatic, recommended):                 │
│ See Part 5 below.                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Object Lifecycle Management

```
┌─────────────────────────────────────────────────────────────────────┐
│           LIFECYCLE MANAGEMENT                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Bucket → Lifecycle → Add a rule                         │
│                                                                       │
│ ── Action ──                                                        │
│ ● Delete object                                                    │
│ ○ Set storage class to: [Nearline/Coldline/Archive ▼]            │
│ ○ Abort incomplete multipart upload                               │
│                                                                       │
│ ── Conditions (combine multiple!) ──                               │
│                                                                       │
│ ☑ Age: [30] days                                                   │
│   ⚡ Days since object was created (or last updated if versioned). │
│                                                                       │
│ ☐ Created before: [2025-01-01]                                    │
│   ⚡ Absolute date. Delete everything created before this date.    │
│                                                                       │
│ ☐ Storage class matches: [Standard ▼]                             │
│   ⚡ Only apply to objects of this storage class.                  │
│                                                                       │
│ ☐ Number of newer versions: [3]                                   │
│   ⚡ For versioned buckets. Delete versions after 3 newer exist.  │
│                                                                       │
│ ☐ Is live: ● Yes ○ No                                            │
│   ⚡ "Is live" = current (non-archived) version.                   │
│     "Not live" = older versions (noncurrent).                  │
│                                                                       │
│ ☐ Days since became noncurrent: [7]                               │
│   ⚡ For versioned buckets: delete old versions after 7 days.     │
│                                                                       │
│ ☐ Matches prefix: [logs/, temp/]                                  │
│   ⚡ Only apply to objects matching this prefix.                    │
│                                                                       │
│ ☐ Matches suffix: [.tmp, .log]                                    │
│   ⚡ Only apply to objects matching this suffix.                    │
│                                                                       │
│ ☐ Days since custom time: [90]                                    │
│   ⚡ Custom time = user-set metadata timestamp.                    │
│     Useful for: "delete 90 days after processing complete".    │
│                                                                       │
│ Common lifecycle configurations:                                   │
│                                                                       │
│ Pattern 1: Tiered storage (cost optimization)                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Rule 1: Age 30 days + Class Standard → Set to Nearline     │  │
│ │ Rule 2: Age 90 days + Class Nearline → Set to Coldline    │  │
│ │ Rule 3: Age 365 days + Class Coldline → Set to Archive    │  │
│ │ Rule 4: Age 2555 days (7 years) → Delete                  │  │
│ │                                                              │  │
│ │ Flow: Standard → 30d → Nearline → 90d → Coldline         │  │
│ │       → 365d → Archive → 7 years → Delete                 │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pattern 2: Cleanup temp files                                      │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Rule: Age 1 day + Prefix "temp/" → Delete                  │  │
│ │ Rule: Age 7 days + Suffix ".tmp" → Delete                 │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pattern 3: Version cleanup                                         │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Rule: Is live=No + Number newer versions > 3 → Delete      │  │
│ │   (Keep only 3 versions of each object)                    │  │
│ │ Rule: Is live=No + Days since noncurrent 30 → Delete      │  │
│ │   (Delete all old versions after 30 days)                  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pattern 4: Abort incomplete uploads                                │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Rule: Abort incomplete multipart uploads after 7 days      │  │
│ │ ⚡ Prevents abandoned uploads from costing money!            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ JSON lifecycle configuration:                                      │
│ {                                                                    │
│   "lifecycle": {                                                    │
│     "rule": [                                                       │
│       {                                                              │
│         "action": {"type": "SetStorageClass",                     │
│                    "storageClass": "NEARLINE"},                   │
│         "condition": {"age": 30,                                   │
│                       "matchesStorageClass": ["STANDARD"]}       │
│       },                                                             │
│       {                                                              │
│         "action": {"type": "Delete"},                              │
│         "condition": {"age": 2555}                                 │
│       }                                                              │
│     ]                                                                │
│   }                                                                  │
│ }                                                                    │
│                                                                       │
│ ⚡ Rules evaluated daily (not real-time). May take up to 24 hours. │
│   Multiple rules can apply to same object — first match wins.   │
│   Transition order: Standard → Nearline → Coldline → Archive   │
│   (can only move "colder", never "warmer" via lifecycle).       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Object Versioning

```
┌─────────────────────────────────────────────────────────────────────┐
│           OBJECT VERSIONING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Enable: Bucket → Settings → Object versioning → Enable            │
│                                                                       │
│ How it works:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Upload report.pdf (version 1) — generation: 1001           │  │
│ │ Upload report.pdf (version 2) — generation: 1002           │  │
│ │                                                              │  │
│ │ Current state:                                               │  │
│ │ ├── report.pdf (live, gen 1002) ← current version         │  │
│ │ └── report.pdf (noncurrent, gen 1001) ← old version       │  │
│ │                                                              │  │
│ │ Delete report.pdf:                                           │  │
│ │ ├── Creates a "delete marker" (0-byte noncurrent object)  │  │
│ │ ├── report.pdf appears deleted to normal reads             │  │
│ │ ├── Old versions still exist!                               │  │
│ │ └── Restore by deleting the delete marker                  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ List versions:                                                       │
│ gsutil ls -la gs://my-bucket/report.pdf                           │
│ # Shows all versions with generation numbers                     │
│                                                                       │
│ Access specific version:                                            │
│ gsutil cp gs://my-bucket/report.pdf#1001 ./old-report.pdf        │
│                                                                       │
│ Restore old version (copy old version as live):                   │
│ gsutil cp gs://my-bucket/report.pdf#1001 gs://my-bucket/report.pdf│
│                                                                       │
│ Delete specific version (permanently):                             │
│ gsutil rm gs://my-bucket/report.pdf#1001                          │
│                                                                       │
│ ⚡ Best practices with versioning:                                   │
│ ├── Always set lifecycle rules to clean old versions!           │
│ │   Otherwise old versions accumulate → storage costs grow!   │
│ ├── Example: Keep max 3 versions, delete after 30 days         │
│ ├── Use for: Documents, configs, anything that changes         │
│ └── Don't use for: Append-only logs, immutable data            │
│                                                                       │
│ ⚡ Versioning vs Soft delete:                                        │
│ ├── Soft delete: Recovers deleted objects (7-90 days)           │
│ ├── Versioning: Recovers overwritten AND deleted objects       │
│ └── Both can be enabled simultaneously for maximum protection  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Retention Policies & Object Holds

```
┌─────────────────────────────────────────────────────────────────────┐
│           RETENTION & HOLDS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Retention Policy (bucket level):                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Bucket → Settings → Retention policy → Set                 │  │
│ │ Duration: [365] days                                        │  │
│ │                                                              │  │
│ │ Effect:                                                      │  │
│ │ ├── Objects CANNOT be deleted until retention period expires│  │
│ │ ├── Objects CANNOT be overwritten (new version = separate) │  │
│ │ ├── Retention period starts when object is created         │  │
│ │ └── Each object has its own "retain until" time            │  │
│ │                                                              │  │
│ │ Unlocked retention policy:                                  │  │
│ │ ├── Can increase duration (e.g., 365 → 730 days)          │  │
│ │ ├── Can remove policy entirely                             │  │
│ │ └── ⚡ Flexible — can change your mind                       │  │
│ │                                                              │  │
│ │ Lock retention policy:                                      │  │
│ │ ├── Bucket → Lock retention policy                         │  │
│ │ ├── ⚠️ IRREVERSIBLE! Cannot unlock, shorten, or remove!     │  │
│ │ ├── Can only INCREASE duration after locking               │  │
│ │ ├── Cannot delete the bucket until ALL objects expire      │  │
│ │ └── Required for: SEC Rule 17a-4, FINRA, CFTC compliance  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Object Holds (per-object):                                         │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Temporary hold:                                              │  │
│ │ ├── Prevents deletion of specific objects                  │  │
│ │ ├── Can be placed and removed freely                       │  │
│ │ └── Use for: Legal hold, investigation hold                │  │
│ │                                                              │  │
│ │ gsutil retention temp set gs://my-bucket/evidence.pdf      │  │
│ │ gsutil retention temp release gs://my-bucket/evidence.pdf  │  │
│ │                                                              │  │
│ │ Event-based hold:                                            │  │
│ │ ├── Automatically placed when object is created            │  │
│ │ ├── Retention timer starts when hold is released           │  │
│ │ ├── Enable on bucket: Default event-based hold = ON       │  │
│ │ └── Use for: "Retain for X days after processing complete"│  │
│ │                                                              │  │
│ │ Example flow:                                                │  │
│ │ 1. Upload data.csv → event-based hold auto-placed         │  │
│ │ 2. Process data.csv → release hold                        │  │
│ │ 3. Retention timer (365 days) starts NOW                  │  │
│ │ 4. After 365 days → can be deleted                        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ Retention vs Versioning vs Soft delete:                           │
│ ┌──────────────────┬──────────────────┬──────────────────────┐    │
│ │ Feature           │ Protects against  │ Reversible?           │    │
│ ├──────────────────┼──────────────────┼──────────────────────┤    │
│ │ Soft delete       │ Accidental delete │ Yes (7-90 day window)│    │
│ │ Versioning        │ Overwrite+delete  │ Yes (unlimited hist) │    │
│ │ Retention policy  │ ALL deletion      │ Only if not locked   │    │
│ │ Object hold       │ Specific obj del  │ Yes (release hold)  │    │
│ │ Bucket lock       │ Policy removal    │ ❌ NEVER!            │    │
│ └──────────────────┴──────────────────┴──────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Public Access & Website Hosting

```
┌─────────────────────────────────────────────────────────────────────┐
│           PUBLIC ACCESS & WEBSITE HOSTING                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Make bucket public:                                                 │
│ 1. Remove public access prevention on bucket                     │
│ 2. Bucket → Permissions → Grant access                          │
│    Member: allUsers                                                │
│    Role: Storage Object Viewer                                    │
│ ⚡ All objects now publicly readable via:                            │
│   https://storage.googleapis.com/BUCKET/OBJECT                   │
│                                                                       │
│ Make single object public (fine-grained ACL mode):                │
│ gsutil acl ch -u AllUsers:R gs://my-bucket/public/logo.png       │
│ URL: https://storage.googleapis.com/my-bucket/public/logo.png    │
│                                                                       │
│ Static website hosting:                                             │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Bucket → Settings → Website configuration                  │  │
│ │ Main page suffix: [index.html]                              │  │
│ │ 404 page: [404.html]                                        │  │
│ │                                                              │  │
│ │ Upload website files:                                        │  │
│ │ gsutil -m cp -r ./build/* gs://my-website-bucket/           │  │
│ │                                                              │  │
│ │ Set Cache-Control:                                           │  │
│ │ gsutil -m setmeta -h "Cache-Control:public, max-age=3600" │  │
│ │   gs://my-website-bucket/**                                 │  │
│ │                                                              │  │
│ │ Access:                                                      │  │
│ │ http://storage.googleapis.com/my-website-bucket/index.html │  │
│ │                                                              │  │
│ │ ⚡ For custom domain (www.example.com):                       │  │
│ │   1. Bucket name MUST match domain: www.example.com        │  │
│ │   2. Verify domain ownership in Search Console             │  │
│ │   3. CNAME: www.example.com → c.storage.googleapis.com    │  │
│ │   4. ⚠️ HTTP only! For HTTPS, put Cloud CDN + LB in front.  │  │
│ │                                                              │  │
│ │ ⚡ Better alternative for static sites:                       │  │
│ │   Use Cloud CDN + Cloud Load Balancing in front of GCS.   │  │
│ │   Gives HTTPS, custom domains, caching, Cloud Armor.      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Requester Pays:                                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Bucket → Settings → Requester pays → Enable                │  │
│ │                                                              │  │
│ │ ⚡ Network + operation costs charged to the REQUESTER,       │  │
│ │   not the bucket owner.                                    │  │
│ │   Storage costs still charged to bucket owner.             │  │
│ │   Requester must specify their billing project.            │  │
│ │                                                              │  │
│ │ gsutil -u their-project-id ls gs://requester-pays-bucket/  │  │
│ │                                                              │  │
│ │ Use for: Sharing large datasets (genomics, satellite data) │  │
│ │   where you don't want to pay for everyone's downloads.   │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform & CLI

### Terraform

```hcl
# Standard bucket (multi-region, versioning, lifecycle)
resource "google_storage_bucket" "prod_assets" {
  name          = "my-company-prod-assets"
  location      = "ASIA"
  project       = var.project_id
  storage_class = "STANDARD"

  # Uniform access control (recommended)
  uniform_bucket_level_access = true

  # Public access prevention
  public_access_prevention = "enforced"

  # Versioning
  versioning {
    enabled = true
  }

  # Soft delete (7-day recoverability)
  soft_delete_policy {
    retention_duration_seconds = 604800  # 7 days
  }

  # Lifecycle rules
  lifecycle_rule {
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
    condition {
      age                   = 30
      matches_storage_class = ["STANDARD"]
    }
  }

  lifecycle_rule {
    action {
      type          = "SetStorageClass"
      storage_class = "COLDLINE"
    }
    condition {
      age                   = 90
      matches_storage_class = ["NEARLINE"]
    }
  }

  lifecycle_rule {
    action {
      type          = "SetStorageClass"
      storage_class = "ARCHIVE"
    }
    condition {
      age                   = 365
      matches_storage_class = ["COLDLINE"]
    }
  }

  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      age = 2555  # 7 years
    }
  }

  # Delete old versions after 30 days
  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      num_newer_versions = 3
      with_state         = "ARCHIVED"
    }
  }

  # Abort incomplete multipart uploads
  lifecycle_rule {
    action {
      type = "AbortIncompleteMultipartUpload"
    }
    condition {
      age = 7
    }
  }

  # CORS (for web app access)
  cors {
    origin          = ["https://app.example.com"]
    method          = ["GET", "HEAD", "PUT", "POST"]
    response_header = ["Content-Type", "Authorization"]
    max_age_seconds = 3600
  }

  labels = {
    environment = "production"
    team        = "backend"
  }
}

# Bucket with Autoclass
resource "google_storage_bucket" "data_lake" {
  name          = "my-company-data-lake"
  location      = "asia-south1"
  project       = var.project_id

  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"

  autoclass {
    enabled                = true
    terminal_storage_class = "ARCHIVE"
  }

  labels = {
    environment = "production"
    purpose     = "data-lake"
  }
}

# Bucket with retention policy (compliance)
resource "google_storage_bucket" "compliance" {
  name          = "my-company-compliance-records"
  location      = "asia-south1"
  project       = var.project_id
  storage_class = "COLDLINE"

  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"

  retention_policy {
    retention_period = 220752000  # 7 years in seconds
    is_locked        = false     # Set true for immutable (IRREVERSIBLE!)
  }

  versioning {
    enabled = true
  }
}

# Bucket for static website
resource "google_storage_bucket" "website" {
  name          = "www-example-com"
  location      = "ASIA"
  project       = var.project_id
  storage_class = "STANDARD"

  uniform_bucket_level_access = true

  website {
    main_page_suffix = "index.html"
    not_found_page   = "404.html"
  }

  cors {
    origin          = ["*"]
    method          = ["GET"]
    response_header = ["Content-Type"]
    max_age_seconds = 86400
  }
}

# Make website bucket public
resource "google_storage_bucket_iam_member" "public_read" {
  bucket = google_storage_bucket.website.name
  role   = "roles/storage.objectViewer"
  member = "allUsers"
}

# Upload object
resource "google_storage_bucket_object" "index" {
  name         = "index.html"
  bucket       = google_storage_bucket.website.name
  source       = "./build/index.html"
  content_type = "text/html"

  cache_control = "public, max-age=3600"
}

# Bucket with CMEK encryption
resource "google_storage_bucket" "encrypted" {
  name          = "my-company-encrypted-data"
  location      = "asia-south1"
  project       = var.project_id

  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"

  encryption {
    default_kms_key_name = google_kms_crypto_key.storage_key.id
  }

  depends_on = [google_kms_crypto_key_iam_member.storage_sa]
}

# Grant KMS access to storage service account
resource "google_kms_crypto_key_iam_member" "storage_sa" {
  crypto_key_id = google_kms_crypto_key.storage_key.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  member        = "serviceAccount:service-${var.project_number}@gs-project-accounts.iam.gserviceaccount.com"
}

# IAM — grant access to a service account
resource "google_storage_bucket_iam_member" "app_access" {
  bucket = google_storage_bucket.prod_assets.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.app_sa.email}"
}

# IAM — read-only access for a team
resource "google_storage_bucket_iam_member" "team_read" {
  bucket = google_storage_bucket.prod_assets.name
  role   = "roles/storage.objectViewer"
  member = "group:data-team@company.com"
}
```

### gsutil / gcloud storage CLI

```bash
# ═══ Bucket operations ═══

# Create bucket
gsutil mb -p my-project -l asia-south1 -c standard \
  -b on gs://my-company-prod-assets
# -p: project, -l: location, -c: storage class
# -b on: uniform bucket-level access

# Or with gcloud:
gcloud storage buckets create gs://my-company-prod-assets \
  --project=my-project \
  --location=asia-south1 \
  --default-storage-class=standard \
  --uniform-bucket-level-access \
  --public-access-prevention

# List buckets
gsutil ls
gcloud storage buckets list --project=my-project

# Get bucket info
gsutil ls -L -b gs://my-company-prod-assets
gcloud storage buckets describe gs://my-company-prod-assets

# Delete bucket (must be empty)
gsutil rm -r gs://my-company-prod-assets
gcloud storage rm -r gs://my-company-prod-assets

# ═══ Object operations ═══

# Upload file
gsutil cp report.pdf gs://my-bucket/reports/
gcloud storage cp report.pdf gs://my-bucket/reports/

# Upload directory (recursive, multi-threaded)
gsutil -m cp -r ./uploads/ gs://my-bucket/uploads/
gcloud storage cp -r ./uploads/ gs://my-bucket/uploads/

# Download file
gsutil cp gs://my-bucket/reports/report.pdf ./
gcloud storage cp gs://my-bucket/reports/report.pdf ./

# Download entire bucket
gsutil -m cp -r gs://my-bucket/ ./local-copy/

# Sync (only upload changes — like rsync)
gsutil -m rsync -r ./local-dir/ gs://my-bucket/remote-dir/
gcloud storage rsync ./local-dir/ gs://my-bucket/remote-dir/
# Add -d to delete remote files not in local (DANGEROUS!)

# List objects
gsutil ls gs://my-bucket/
gsutil ls -l gs://my-bucket/reports/     # With sizes
gsutil ls -la gs://my-bucket/report.pdf  # All versions

# Delete object
gsutil rm gs://my-bucket/old-file.txt
gcloud storage rm gs://my-bucket/old-file.txt

# Delete with prefix (all logs)
gsutil -m rm gs://my-bucket/logs/**

# Move/rename object
gsutil mv gs://my-bucket/old-name.pdf gs://my-bucket/new-name.pdf

# Copy between buckets
gsutil -m cp -r gs://source-bucket/ gs://dest-bucket/

# ═══ Metadata ═══

# Set Content-Type
gsutil setmeta -h "Content-Type:application/pdf" \
  gs://my-bucket/report.pdf

# Set Cache-Control
gsutil setmeta -h "Cache-Control:public, max-age=86400" \
  gs://my-bucket/static/**

# Set custom metadata
gsutil setmeta -h "x-goog-meta-uploaded-by:ritesh" \
  gs://my-bucket/report.pdf

# View metadata
gsutil stat gs://my-bucket/report.pdf

# ═══ Versioning ═══

# Enable versioning
gsutil versioning set on gs://my-bucket/
gcloud storage buckets update gs://my-bucket/ --versioning

# List versions
gsutil ls -la gs://my-bucket/report.pdf

# Restore old version
gsutil cp "gs://my-bucket/report.pdf#1234567890" \
  gs://my-bucket/report.pdf

# ═══ Lifecycle ═══

# Get current lifecycle
gsutil lifecycle get gs://my-bucket/

# Set lifecycle from JSON file
gsutil lifecycle set lifecycle.json gs://my-bucket/

# ═══ Retention ═══

# Set retention policy (365 days)
gsutil retention set 365d gs://my-bucket/

# Lock retention (⚠️ IRREVERSIBLE!)
gsutil retention lock gs://my-bucket/

# Set temporary hold on object
gsutil retention temp set gs://my-bucket/evidence.pdf
gsutil retention temp release gs://my-bucket/evidence.pdf

# ═══ Access / IAM ═══

# View bucket IAM
gsutil iam get gs://my-bucket/

# Grant access
gsutil iam ch user:dev@company.com:objectViewer gs://my-bucket/

# Make public
gsutil iam ch allUsers:objectViewer gs://my-bucket/

# Remove access
gsutil iam ch -d user:dev@company.com:objectViewer gs://my-bucket/

# ═══ CORS ═══

# Set CORS
gsutil cors set cors.json gs://my-bucket/

# cors.json:
# [{"origin": ["https://app.example.com"],
#   "method": ["GET", "PUT"],
#   "responseHeader": ["Content-Type"],
#   "maxAgeSeconds": 3600}]

# ═══ Useful flags ═══

# -m: Multi-threaded (parallel operations, much faster!)
gsutil -m cp -r ./big-directory/ gs://my-bucket/

# -o: Override config
gsutil -o GSUtil:parallel_composite_upload_threshold=150M \
  cp huge-file.tar.gz gs://my-bucket/

# du: Disk usage
gsutil du -s gs://my-bucket/
gsutil du -sh gs://my-bucket/  # Human-readable

# hash: Verify file integrity
gsutil hash local-file.txt
gsutil stat gs://my-bucket/local-file.txt  # Compare hashes
```

---

## Part 10: Real-World Patterns

### Startup

```
Simple storage setup:

Buckets:
├── my-startup-prod-uploads (asia-south1, Standard)
│   Purpose: User-uploaded files (photos, documents)
│   Access: App SA has objectAdmin, public access prevented
│   Versioning: Enabled (protect user uploads)
│   Lifecycle: Old versions deleted after 30 days
│   CORS: Allow app.startup.com
│   Size: ~50 GB, Cost: ~$1.15/month
│
├── my-startup-prod-assets (ASIA, Standard)
│   Purpose: Static website assets (JS, CSS, images)
│   Access: Public read (allUsers:objectViewer)
│   CDN: Cloud CDN in front for caching
│   Cache-Control: public, max-age=86400
│   Lifecycle: None (always current)
│   Size: ~5 GB, Cost: ~$0.13/month
│
├── my-startup-backups (asia-south1, Nearline)
│   Purpose: Database backups (daily)
│   Access: Only backup SA, public access prevented
│   Lifecycle:
│   ├── Age 90 days → Coldline
│   ├── Age 365 days → Archive
│   └── Age 1825 days (5 years) → Delete
│   Size: ~200 GB, Cost: ~$2.60/month (Nearline)
│
└── my-startup-dev-uploads (asia-south1, Standard)
    Purpose: Dev/staging uploads
    Lifecycle: Delete after 30 days (auto-cleanup)
    Size: ~10 GB, Cost: ~$0.23/month

Total: ~$4/month storage
```

### Mid-Size

```
Multi-bucket architecture:

Buckets:
├── app-prod-user-uploads (asia-south1, Standard)
│   50 teams, 100K users, ~2 TB uploads
│   Organized by: uploads/{userId}/{year}/{filename}
│   Versioning: ON, lifecycle: 3 versions max, 30d noncurrent
│   Signed URLs for direct upload from browser (no server proxy!)
│   Virus scanning: Cloud Functions trigger on upload → scan
│   Cost: ~$46/month
│
├── app-prod-static (ASIA, Standard + CDN)
│   Website assets, API responses cache
│   Cloud CDN in front, Cache-Control headers
│   CI/CD: Build → gsutil rsync → invalidate CDN
│   Cost: ~$3/month
│
├── app-prod-exports (asia-south1, Standard + Autoclass)
│   Reports, CSVs, generated PDFs
│   Autoclass: Hot data auto-tiers to cheaper classes
│   Notifications: Pub/Sub on upload → notify user
│   Cost: ~$15/month
│
├── data-lake-raw (asia-south1, Standard)
│   Raw data from APIs, ETL sources
│   Prefix structure: raw/{source}/{date}/
│   BigQuery external tables read directly from here
│   Lifecycle: 90d → Nearline (raw data ages quickly)
│   Size: ~5 TB, Cost: ~$115/month
│
├── data-lake-processed (asia-south1, Standard)
│   Transformed data (Parquet, Avro files)
│   Read by BigQuery, Dataflow, Dataproc
│   Lifecycle: 365d → Coldline
│   Size: ~3 TB, Cost: ~$69/month
│
├── backups-db (asia-south1 + asia-south2, Nearline)
│   Dual-region for DR
│   Cloud SQL automated exports, Firestore exports
│   Retention policy: 365 days (unlocked)
│   Lifecycle: 90d Nearline → Coldline, 365d → Archive, 7y delete
│   Size: ~1 TB, Cost: ~$16/month
│
├── backups-config (asia-south1, Coldline)
│   Terraform state (remote backend), config snapshots
│   Versioning: ON, retention: 30 days
│   Size: ~10 GB, Cost: ~$0.06/month
│
└── logs-archive (asia-south1, Standard + Autoclass)
    Log sink from Cloud Logging
    Autoclass: Quickly moves to Archive as logs age
    Retention policy: 1 year (compliance)
    Size: ~2 TB, Cost: starts ~$46, drops quickly with Autoclass

Security:
├── All buckets: Uniform access, public access prevention
├── IAM: Least privilege per service account per bucket
├── CMEK: Sensitive data buckets use Cloud KMS keys
├── VPC Service Controls: Prevent data exfiltration
└── Audit: Data access logs enabled for sensitive buckets

Total: ~$310/month
```

### Enterprise

```
Enterprise storage platform:

Buckets (30+):
├── Per-service data buckets (10+):
│   ├── svc-orders-uploads (user order documents)
│   ├── svc-payments-receipts (payment confirmations)
│   ├── svc-users-avatars (profile images)
│   ├── svc-reports-output (generated reports)
│   └── ... per microservice isolation
│
├── Data platform (5+):
│   ├── data-lake-raw-{source} (per source raw data)
│   ├── data-lake-processed (transformed data)
│   ├── data-lake-curated (business-ready datasets)
│   ├── ml-training-data (Vertex AI training sets)
│   └── ml-models (exported model artifacts)
│
├── Backups & DR (5+):
│   ├── backups-db-primary (dual-region, hourly)
│   ├── backups-db-dr (different multi-region, daily)
│   ├── backups-config (Terraform state, K8s manifests)
│   ├── backups-application (app data exports)
│   └── dr-replication (cross-region, turbo replication)
│
├── Compliance (3+):
│   ├── compliance-audit-logs (LOCKED retention 7 years)
│   ├── compliance-financial-records (LOCKED retention 10 years)
│   └── compliance-pii-exports (CMEK, VPC SC, retention)
│
├── Static & CDN (3+):
│   ├── cdn-static-assets (multi-region, Cloud CDN)
│   ├── cdn-media (video/image delivery)
│   └── cdn-downloads (software releases)
│
└── Operations (3+):
    ├── logs-archive (Cloud Logging sink)
    ├── build-artifacts (Cloud Build output)
    └── terraform-state (versioned, retention policy)

Patterns:
├── Signed URLs: All user uploads via time-limited signed URLs
│   Client → GET signed URL from API → PUT directly to GCS
│   No data touches your servers (saves bandwidth + compute)
├── Notifications: Pub/Sub on all upload buckets
│   Upload → Pub/Sub → Cloud Functions (process/scan/index)
├── Transfer Service: Daily sync from AWS S3 (multi-cloud)
├── Compose objects: Merge multipart uploads into single object
├── Parallel composite: Large file uploads split automatically
└── Cross-project: Shared buckets with org-level IAM policies

Security:
├── Org policy: Uniform bucket access enforced org-wide
├── Org policy: Public access prevention enforced org-wide
├── VPC Service Controls: All data buckets in perimeter
├── CMEK: All production + compliance buckets
├── Access Transparency: See when Google accesses your data
├── Data Loss Prevention: DLP API scans sensitive uploads
├── Object holds: Legal hold on investigation-related data
├── Retention locks: Compliance buckets (FINRA, HIPAA)
└── Audit: Data access logs → BigQuery → SIEM

Cost: $3,000-15,000/month (50-200 TB total)
```

---

## Part 11: Console Walkthrough: Managing & Deleting Buckets

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING BUCKETS FROM CONSOLE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Cloud Storage → Buckets → Click bucket name             │
│                                                                       │
│ ── Edit Bucket Settings ──                                         │
│ Bucket → Settings tab                                              │
│                                                                       │
│ Change default storage class:                                      │
│ ├── Settings → Default storage class → Edit                     │
│ ├── Select new class (Standard/Nearline/Coldline/Archive)       │
│ ├── Click Save                                                    │
│ ├── ⚡ Only affects NEW objects uploaded after the change.          │
│ │   Existing objects keep their current storage class.          │
│ └── To change existing objects: use lifecycle rules or            │
│     gsutil rewrite -s NEARLINE gs://my-bucket/**                │
│                                                                       │
│ Enable/disable versioning:                                         │
│ ├── Settings → Object versioning → Edit                          │
│ ├── Toggle Enable/Disable                                         │
│ ├── ⚡ Disabling does NOT delete existing versions.                 │
│ │   It just stops creating new versions on overwrite.          │
│ └── Clean up old versions with lifecycle rules.                  │
│                                                                       │
│ Add/edit labels:                                                    │
│ ├── Settings → Labels → Edit                                     │
│ ├── Add key-value pairs (e.g., team=backend, env=prod)          │
│ ├── Click Save                                                    │
│ └── ⚡ Labels help filter buckets in Console and track costs        │
│     in Billing → Reports (group by label).                     │
│                                                                       │
│ ── Download Objects ──                                              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Console → Bucket → Navigate to the object                  │  │
│ │                                                              │  │
│ │ Single file:                                                 │  │
│ │ ├── Click the object name to open its detail page          │  │
│ │ └── Click [Download] button at the top                     │  │
│ │                                                              │  │
│ │ Multiple files:                                              │  │
│ │ ├── ☑ Check the boxes next to each file you want           │  │
│ │ ├── Click [Download] in the top action bar                 │  │
│ │ └── ⚡ Files download individually (no auto-zip in Console). │  │
│ │     For bulk download, use CLI:                            │  │
│ │     gsutil -m cp -r gs://my-bucket/folder/ ./local/        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Delete Objects ──                                                │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Console → Bucket → Navigate to the object(s)               │  │
│ │                                                              │  │
│ │ Single object:                                               │  │
│ │ ├── ☑ Check the box next to the object                     │  │
│ │ ├── Click [Delete] in the top action bar                   │  │
│ │ └── Confirm deletion in the popup                          │  │
│ │                                                              │  │
│ │ Multiple objects:                                            │  │
│ │ ├── ☑ Check boxes for all objects to delete                │  │
│ │ ├── Click [Delete] → Confirm                               │  │
│ │                                                              │  │
│ │ Delete all objects in a "folder":                           │  │
│ │ ├── ☑ Check the folder prefix checkbox                     │  │
│ │ ├── Click [Delete] → Confirm                               │  │
│ │ └── ⚡ Deletes all objects with that prefix.                  │  │
│ │                                                              │  │
│ │ ⚡ If soft delete is enabled (default), deleted objects are   │  │
│ │   recoverable for 7 days. Check: Bucket → Show soft-deleted│  │
│ │   objects (toggle at top).                                  │  │
│ │                                                              │  │
│ │ ⚡ If versioning is enabled, deleting creates a "delete      │  │
│ │   marker" — old versions still exist! To permanently       │  │
│ │   delete, delete the specific version from the versions tab.│  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Delete a Bucket ──                                               │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Console → Cloud Storage → Buckets                           │  │
│ │                                                              │  │
│ │ Option 1: Delete empty bucket                               │  │
│ │ ├── First delete all objects inside (or use lifecycle rules)│  │
│ │ ├── ☑ Check the box next to the bucket name                │  │
│ │ ├── Click [Delete] in the top action bar                   │  │
│ │ └── Type the bucket name to confirm → Delete               │  │
│ │                                                              │  │
│ │ Option 2: Delete bucket with contents (one step)           │  │
│ │ ├── ☑ Check the box next to the bucket name                │  │
│ │ ├── Click [Delete]                                          │  │
│ │ ├── Console warns: "This bucket contains objects"          │  │
│ │ ├── Type the bucket name to confirm                        │  │
│ │ └── GCS deletes ALL objects first, then the bucket         │  │
│ │                                                              │  │
│ │ ⚠️ Bucket deletion is PERMANENT!                              │  │
│ │   ├── Cannot undo. Bucket name becomes available again     │  │
│ │   │   (someone else could take it!).                       │  │
│ │   ├── If retention policy is locked, cannot delete bucket  │  │
│ │   │   until ALL objects pass their retention period.       │  │
│ │   └── Soft-deleted objects must expire first (or be purged)│  │
│ │       before bucket can be deleted.                        │  │
│ │                                                              │  │
│ │ CLI equivalent:                                              │  │
│ │ # Delete bucket and all contents                            │  │
│ │ gsutil rm -r gs://my-bucket/                                │  │
│ │ gcloud storage rm -r gs://my-bucket/                       │  │
│ │                                                              │  │
│ │ # Delete only contents, keep bucket                         │  │
│ │ gsutil rm gs://my-bucket/**                                 │  │
│ │ gcloud storage rm gs://my-bucket/**                        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: CORS Configuration

```
┌─────────────────────────────────────────────────────────────────────┐
│           CORS (Cross-Origin Resource Sharing)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is CORS?                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Browsers block requests to a different domain for security. │  │
│ │ This is called the "Same-Origin Policy."                   │  │
│ │                                                              │  │
│ │ Example:                                                     │  │
│ │ Your web app: https://app.example.com                      │  │
│ │ Your GCS bucket: https://storage.googleapis.com/my-bucket  │  │
│ │                                                              │  │
│ │ These are DIFFERENT domains (different origins).            │  │
│ │ Browser says: "Blocked! You can't fetch from a different   │  │
│ │ domain unless that domain explicitly allows it."           │  │
│ │                                                              │  │
│ │ CORS = the mechanism where the server (GCS) tells the      │  │
│ │ browser: "It's OK, I allow requests from app.example.com."│  │
│ │                                                              │  │
│ │ Without CORS: Browser blocks the request. You see errors   │  │
│ │   like "Access-Control-Allow-Origin" in browser DevTools.  │  │
│ │ With CORS: Browser allows the request. Everything works.   │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ When do you need CORS on a GCS bucket?                             │
│ ├── Web app uploads files directly to GCS (signed URLs)         │
│ ├── Web app downloads/displays files from GCS                   │
│ ├── JavaScript fetch() calls to GCS from the browser            │
│ ├── Fonts/CSS loaded from GCS on a different domain             │
│ └── ⚡ If your app server fetches from GCS (server-to-server),    │
│     CORS is NOT needed. CORS is a BROWSER-ONLY restriction.    │
│                                                                       │
│ ── Setting CORS on a Bucket ──                                     │
│                                                                       │
│ Step 1: Create a CORS JSON config file (cors.json):              │
│                                                                       │
│ [                                                                    │
│   {                                                                  │
│     "origin": ["https://app.example.com"],                        │
│     "method": ["GET", "HEAD", "PUT", "POST", "DELETE"],          │
│     "responseHeader": ["Content-Type", "Authorization",          │
│                         "Content-Length", "x-goog-resumable"], │
│     "maxAgeSeconds": 3600                                        │
│   }                                                                  │
│ ]                                                                    │
│                                                                       │
│ Fields explained:                                                   │
│ ├── origin: Which domains can make requests.                     │
│ │   ["https://app.example.com"] = only this domain.             │
│ │   ["*"] = any domain (⚠️ use for public/dev only!).            │
│ ├── method: Which HTTP methods are allowed.                     │
│ │   GET/HEAD for downloads. PUT/POST for uploads.              │
│ ├── responseHeader: Which headers the browser can read.         │
│ │   Include "x-goog-resumable" for resumable uploads.         │
│ └── maxAgeSeconds: How long the browser caches the CORS policy. │
│     3600 = 1 hour. Reduces preflight requests.                │
│                                                                       │
│ Step 2: Apply CORS config to bucket:                              │
│                                                                       │
│ # Using gsutil                                                     │
│ gsutil cors set cors.json gs://my-bucket/                         │
│                                                                       │
│ # Using gcloud                                                     │
│ gcloud storage buckets update gs://my-bucket/ \                  │
│   --cors-file=cors.json                                           │
│                                                                       │
│ Step 3: Verify CORS is set:                                       │
│ gsutil cors get gs://my-bucket/                                   │
│                                                                       │
│ Remove CORS (set empty config):                                   │
│ echo '[]' > empty-cors.json                                       │
│ gsutil cors set empty-cors.json gs://my-bucket/                  │
│                                                                       │
│ ── Example: CORS for Web Uploads (Signed URL) ──                  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Scenario: Users upload profile photos from your web app    │  │
│ │ directly to GCS using signed URLs.                         │  │
│ │                                                              │  │
│ │ cors-upload.json:                                            │  │
│ │ [                                                            │  │
│ │   {                                                          │  │
│ │     "origin": [                                             │  │
│ │       "https://app.example.com",                           │  │
│ │       "https://staging.example.com"                        │  │
│ │     ],                                                       │  │
│ │     "method": ["GET", "HEAD", "PUT"],                     │  │
│ │     "responseHeader": [                                     │  │
│ │       "Content-Type",                                      │  │
│ │       "Content-Length",                                     │  │
│ │       "x-goog-resumable"                                   │  │
│ │     ],                                                       │  │
│ │     "maxAgeSeconds": 3600                                  │  │
│ │   }                                                          │  │
│ │ ]                                                            │  │
│ │                                                              │  │
│ │ Flow:                                                        │  │
│ │ 1. User selects a file in your web app                     │  │
│ │ 2. App calls your backend API: POST /get-upload-url       │  │
│ │ 3. Backend generates a signed URL (PUT, 15 min expiry)    │  │
│ │ 4. App uses JavaScript fetch() to PUT file to signed URL  │  │
│ │ 5. GCS checks CORS → origin matches → upload succeeds    │  │
│ │ 6. ⚡ File goes directly to GCS, never touches your server!  │  │
│ │                                                              │  │
│ │ ⚡ Without this CORS config, step 5 would fail with:         │  │
│ │   "No 'Access-Control-Allow-Origin' header is present"     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Common CORS configs ──                                          │
│                                                                       │
│ Public bucket (read-only, any origin):                            │
│ [{"origin": ["*"], "method": ["GET", "HEAD"],                   │
│   "responseHeader": ["Content-Type"], "maxAgeSeconds": 86400}] │
│                                                                       │
│ Dev/local testing (allow localhost):                              │
│ [{"origin": ["http://localhost:3000",                             │
│               "http://localhost:5173"],                           │
│   "method": ["GET", "HEAD", "PUT", "POST", "DELETE"],           │
│   "responseHeader": ["*"], "maxAgeSeconds": 60}]               │
│ ⚡ maxAgeSeconds: 60 in dev (changes apply quickly).               │
│   3600+ in production (reduces preflight requests).             │
│                                                                       │
│ ⚡ CORS is set at the bucket level, not per-object.                 │
│   All objects in the bucket share the same CORS policy.         │
│                                                                       │
│ ⚡ Terraform CORS config (see Part 9 above for full example):      │
│   cors {                                                           │
│     origin          = ["https://app.example.com"]               │
│     method          = ["GET", "HEAD", "PUT"]                    │
│     response_header = ["Content-Type", "x-goog-resumable"]     │
│     max_age_seconds = 3600                                       │
│   }                                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Cost Calculation Example

```
┌─────────────────────────────────────────────────────────────────────┐
│           COST CALCULATION — WORKED EXAMPLE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: A startup stores user uploads in Cloud Storage.          │
│ ├── 500 GB in Standard storage (asia-south1, single region)     │
│ ├── 100,000 read operations per day (Class B: GET object)       │
│ ├── 10,000 write operations per day (Class A: PUT object)       │
│ ├── 50 GB egress (data downloaded to internet) per month        │
│ └── Let's calculate the monthly cost!                            │
│                                                                       │
│ ── 1. Storage Cost ──                                               │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Standard storage (asia-south1): $0.023/GB/month             │  │
│ │ 500 GB × $0.023 = $11.50/month                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── 2. Operations Cost ──                                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Class B operations (reads/GETs):                             │  │
│ │ 100,000 ops/day × 30 days = 3,000,000 ops/month            │  │
│ │ Standard Class B: $0.004 per 10,000 ops                    │  │
│ │ 3,000,000 ÷ 10,000 × $0.004 = $1.20/month                 │  │
│ │                                                              │  │
│ │ Class A operations (writes/PUTs):                            │  │
│ │ 10,000 ops/day × 30 days = 300,000 ops/month               │  │
│ │ Standard Class A: $0.05 per 10,000 ops                     │  │
│ │ 300,000 ÷ 10,000 × $0.05 = $1.50/month                    │  │
│ │                                                              │  │
│ │ Total operations: $1.20 + $1.50 = $2.70/month              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── 3. Network Egress Cost ──                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Data transfer to internet (egress):                         │  │
│ │ First 0-1 TB/month: $0.12/GB (Asia)                        │  │
│ │ 50 GB × $0.12 = $6.00/month                                │  │
│ │                                                              │  │
│ │ ⚡ Egress within same region (GCS → Compute Engine):         │  │
│ │   FREE! No charge for intra-region traffic.                │  │
│ │ ⚡ Egress to same multi-region location: FREE!                │  │
│ │ ⚡ Ingress (upload to GCS): Always FREE!                      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Total Monthly Cost ──                                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ ┌───────────────────────┬────────────────────┐              │  │
│ │ │ Component              │ Monthly Cost        │              │  │
│ │ ├───────────────────────┼────────────────────┤              │  │
│ │ │ Storage (500 GB)       │ $11.50              │              │  │
│ │ │ Read operations (3M)   │ $1.20               │              │  │
│ │ │ Write operations (300K)│ $1.50               │              │  │
│ │ │ Egress (50 GB)         │ $6.00               │              │  │
│ │ ├───────────────────────┼────────────────────┤              │  │
│ │ │ TOTAL                  │ ~$20.20/month       │              │  │
│ │ └───────────────────────┴────────────────────┘              │  │
│ │                                                              │  │
│ │ ⚡ That's about $0.67/day for 500 GB with heavy read usage! │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── What if they used Nearline instead? ──                          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Storage: 500 GB × $0.013 = $6.50/month (saved $5.00!)      │  │
│ │ Retrieval: 50 GB reads × $0.01/GB = $0.50/month (extra!)  │  │
│ │ Class B ops: $0.01/10K × 300 = $3.00/month (more costly!) │  │
│ │ Class A ops: $0.10/10K × 30 = $3.00/month (more costly!)  │  │
│ │ Egress: $6.00 (same)                                        │  │
│ │ Total: ~$19.00/month                                        │  │
│ │                                                              │  │
│ │ ⚡ Slightly cheaper, BUT Nearline has 30-day minimum hold!   │  │
│ │   With 100K reads/day, this data is HOT — Standard is      │  │
│ │   the right choice. Nearline saves money only when you     │  │
│ │   access data infrequently (once a month or less).         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Cost Optimization Tips ──                                       │
│ ├── Use lifecycle rules to auto-tier old data to cheaper classes │
│ ├── Enable Autoclass if access patterns are unpredictable       │
│ ├── Use regional buckets (cheaper than multi-region) when        │
│ │   compute and storage are in the same region                 │
│ ├── Put Cloud CDN in front for frequently read public content   │
│ │   (reduces egress costs + faster delivery)                   │
│ ├── Delete temp files and abort incomplete uploads (lifecycle)  │
│ ├── Use the GCP Pricing Calculator for exact estimates:         │
│ │   https://cloud.google.com/products/calculator               │
│ └── Monitor costs: Billing → Reports → filter by Cloud Storage │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Object storage (files/blobs) — globally unified |
| Durability | 99.999999999% (11 nines) |
| Availability | 99.95% (multi-region Standard) |
| Max object size | 5 TiB |
| Bucket names | Globally unique, permanent |
| Location types | Region, Dual-region, Multi-region |
| Storage classes | Standard, Nearline (30d), Coldline (90d), Archive (365d) |
| Autoclass | Auto-tier objects based on access patterns |
| Access control | Uniform (recommended) or Fine-grained (ACLs) |
| Versioning | Keep old versions on overwrite/delete |
| Soft delete | Recover deleted objects (7-90 day window) |
| Retention policy | Prevent deletion until period expires |
| Bucket lock | Immutable retention (IRREVERSIBLE) |
| Object holds | Temporary / Event-based per-object lock |
| Lifecycle rules | Auto-transition classes, auto-delete, cleanup |
| Encryption | Always encrypted (Google-managed, CMEK, or CSEK) |
| Consistency | Strong global consistency (read-after-write) |
| Website hosting | Static site hosting (index + 404 pages) |
| Requester pays | Requester pays for access costs |
| AWS equivalent | Amazon S3 |
| Azure equivalent | Azure Blob Storage |

---

## What's Next?

In the next chapter, we'll cover advanced Cloud Storage topics — IAM vs ACLs, signed URLs, notifications, transfer service, and compose objects.

→ Next: [Chapter 21: Cloud Storage Advanced](21-cloud-storage-advanced.md)

---

*Last Updated: May 2026*
