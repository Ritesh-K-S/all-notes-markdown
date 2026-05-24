# Chapter 23: Azure Blob Storage

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Blob Storage Fundamentals](#part-1-blob-storage-fundamentals)
- [Part 2: Working with Containers & Blobs (Portal Walkthrough)](#part-2-working-with-containers--blobs-portal-walkthrough)
- [Part 3: Blob Types](#part-3-blob-types)
- [Part 4: Lifecycle Management](#part-4-lifecycle-management)
- [Part 5: Versioning & Soft Delete](#part-5-versioning--soft-delete)
- [Part 6: Static Website Hosting](#part-6-static-website-hosting)
- [Part 7: Blob Index Tags & Metadata](#part-7-blob-index-tags--metadata)
- [Part 8: Immutable Storage (WORM)](#part-8-immutable-storage-worm)
- [Part 9: Terraform & Bicep](#part-9-terraform--bicep)
- [Part 10: az CLI & AzCopy Reference](#part-10-az-cli--azcopy-reference)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Blob Storage is Azure's object storage service for storing massive amounts of unstructured data — files, images, videos, logs, backups, documents, etc. "Blob" stands for Binary Large Object. Think of it as a giant file system in the cloud where you can store any type of file.

```
What you'll learn:
├── Blob Storage Fundamentals
│   ├── Containers (like folders)
│   ├── Blobs (the actual files)
│   ├── Blob types (Block, Append, Page)
│   └── URL structure & access
├── Creating Containers & Uploading Blobs (Portal)
├── Blob Types (when to use which)
├── Lifecycle Management (auto-tier, auto-delete)
├── Versioning & Soft Delete (data protection)
├── Static Website Hosting
├── Blob Index Tags & Metadata
├── Immutable Storage (WORM compliance)
├── Terraform, Bicep, az CLI, AzCopy
└── Real-world patterns
```

---

## Part 1: Blob Storage Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           BLOB STORAGE CONCEPT                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Hierarchy:                                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Storage Account: stmyappprod                                │  │
│ │ URL: https://stmyappprod.blob.core.windows.net              │  │
│ │                                                              │  │
│ │ ├── Container: images (like a folder/bucket)               │  │
│ │ │   ├── profile/user1.jpg (blob)                          │  │
│ │ │   ├── profile/user2.jpg (blob)                          │  │
│ │ │   └── banners/hero.png (blob)                           │  │
│ │ │                                                           │  │
│ │ │   URL: https://stmyappprod.blob.core.windows.net/images/│  │
│ │ │        profile/user1.jpg                                 │  │
│ │ │                                                           │  │
│ │ ├── Container: backups                                     │  │
│ │ │   ├── db-2024-01-15.bak                                │  │
│ │ │   └── db-2024-01-16.bak                                │  │
│ │ │                                                           │  │
│ │ └── Container: logs                                        │  │
│ │     ├── 2024/01/15/app.log                               │  │
│ │     └── 2024/01/16/app.log                               │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Container = Like a folder or S3 bucket                       │
│ │   Cannot nest containers (use virtual folders with / in name)│
│ │   Name: 3-63 chars, lowercase, hyphens allowed               │
│ ├── Blob = The actual file (any type, any size up to 190 TB)   │
│ │   Can have virtual folder structure using / in the name      │
│ │   Example: "images/profile/user1.jpg"                        │
│ └── Flat namespace = No real folders, just blob names with /   │
│     "images/profile/user1.jpg" is ONE blob name (not a folder)│
│                                                                       │
│ Container access levels:                                             │
│ ┌────────────────┬────────────────────────────────────────────┐   │
│ │ Level          │ Description                                │   │
│ ├────────────────┼────────────────────────────────────────────┤   │
│ │ Private        │ No anonymous access (default, recommended) │   │
│ │ Blob           │ Anonymous read for blobs only              │   │
│ │ Container      │ Anonymous read for blobs + list blobs     │   │
│ └────────────────┴────────────────────────────────────────────┘   │
│ ⚡ Always use Private unless you specifically need public access.│
│                                                                       │
│ ⚡ AWS equivalent: S3 Buckets                                       │
│ ⚡ GCP equivalent: Cloud Storage Buckets                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Working with Containers & Blobs (Portal Walkthrough)

```
Console → Storage Account → Containers

┌─────────────────────────────────────────────────────────────────────┐
│           CREATING A CONTAINER                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ [+ Container]                                                       │
│                                                                       │
│ Name: [images]                                                      │
│ ⚡ Rules: 3-63 chars, lowercase, numbers, hyphens                  │
│   No uppercase, no underscores, no consecutive hyphens            │
│                                                                       │
│ Anonymous access level:                                              │
│ ● Private (no anonymous access) ← ALWAYS use this              │
│ ○ Blob (anonymous read access for blobs only)                   │
│ ○ Container (anonymous read access for container and blobs)     │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           UPLOADING BLOBS                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Click on container → [Upload]                                      │
│                                                                       │
│ Files: [Browse...] (select files from your computer)               │
│                                                                       │
│ ▶ Advanced:                                                        │
│                                                                       │
│ Upload to folder: [profile/]                                       │
│ ⚡ Creates virtual folder path: images/profile/photo.jpg          │
│   Not a real folder — just a prefix in the blob name.            │
│                                                                       │
│ Blob type:                                                          │
│ ● Block blob (default — for most files)                           │
│ ○ Page blob (for random read/write — VHD disks)                 │
│ ○ Append blob (for log files — append-only)                     │
│                                                                       │
│ Block size: [auto]                                                  │
│                                                                       │
│ Access tier:                                                         │
│ ○ Hot (frequently accessed)                                       │
│ ○ Cool (infrequently accessed, 30+ days)                         │
│ ○ Cold (rarely accessed, 90+ days)                                │
│ ○ Archive (long-term, 180+ days, offline)                        │
│ ⚡ Overrides the account default tier for this specific blob.     │
│                                                                       │
│ [Upload]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING BLOBS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Click on a blob to see details:                                     │
│                                                                       │
│ URL: https://stmyappprod.blob.core.windows.net/images/photo.jpg  │
│ Type: Block blob                                                    │
│ Size: 2.4 MB                                                        │
│ Access tier: Hot                                                     │
│ Last modified: 2024-01-15T10:30:00Z                                │
│ Content type: image/jpeg                                            │
│ ETag: "0x8DBXXXXXXXXXXXX"                                          │
│                                                                       │
│ Actions:                                                             │
│ ├── [Download] — Download the blob to your computer              │
│ ├── [Delete] — Delete the blob (soft delete if enabled)          │
│ ├── [Change tier] — Move between Hot/Cool/Cold/Archive          │
│ ├── [Generate SAS] — Create a temporary access URL              │
│ ├── [Snapshots] — Create/view snapshots                         │
│ ├── [Versions] — View previous versions                         │
│ ├── [Edit] — Edit blob content (for text files)                 │
│ └── [Properties] — View/edit metadata, content type             │
│                                                                       │
│ Generate SAS (temporary access link):                               │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Signing method: Account key / User delegation key           │  │
│ │ Permissions: ☑ Read  ☐ Write  ☐ Delete  ☐ List           │  │
│ │ Start: 2024-01-15 10:00                                     │  │
│ │ Expiry: 2024-01-15 18:00                                    │  │
│ │ Allowed IP: 203.0.113.0/24 (optional)                      │  │
│ │ Protocol: HTTPS only                                         │  │
│ │                                                              │  │
│ │ [Generate SAS token and URL]                                │  │
│ │                                                              │  │
│ │ SAS URL: https://stmyappprod.blob.core.windows.net/images/ │  │
│ │   photo.jpg?sv=2023-01-01&se=2024-01-15&sr=b&sp=r&sig=... │  │
│ │                                                              │  │
│ │ ⚡ Share this URL — anyone with it can read the blob       │  │
│ │   until the expiry time. No login needed!                  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Blob Types

```
┌─────────────────────────────────────────────────────────────────────┐
│           BLOB TYPES                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Block Blob (99% of use cases)                                    │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ├── Optimized for uploading large amounts of data           │  │
│ │ ├── Max size: 190.7 TB (using 4000 blocks × 4000 MB each) │  │
│ │ ├── Upload in blocks (parallel upload for speed)           │  │
│ │ ├── Use for: images, videos, documents, backups, logs      │  │
│ │ ├── Supports all access tiers (Hot/Cool/Cold/Archive)     │  │
│ │ └── Most features: versioning, snapshots, lifecycle mgmt  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 2. Append Blob (for logs)                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ├── Optimized for APPEND operations                         │  │
│ │ ├── You can only ADD data to the end (no modify/delete)    │  │
│ │ ├── Max size: 195 GB                                        │  │
│ │ ├── Use for: application logs, audit logs, event data      │  │
│ │ └── Cannot change tier (always Hot)                        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 3. Page Blob (for VHDs)                                             │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ├── Optimized for random read/write operations              │  │
│ │ ├── Max size: 8 TB                                          │  │
│ │ ├── Stored as 512-byte pages                                │  │
│ │ ├── Use for: virtual machine disks (VHD files)             │  │
│ │ ├── Azure Managed Disks use page blobs internally          │  │
│ │ └── Rarely used directly (use Managed Disks instead)      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Decision: Use Block Blob unless you have a specific reason not to.│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Lifecycle Management

```
┌─────────────────────────────────────────────────────────────────────┐
│           LIFECYCLE MANAGEMENT                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Automatically move or delete blobs based on age.                    │
│ Saves money by moving old data to cheaper tiers!                    │
│                                                                       │
│ Portal → Storage Account → Lifecycle management → Add a rule     │
│                                                                       │
│ Rule name: [move-to-cool-after-30-days]                            │
│ Rule scope: ● Apply to all blobs  ○ Limit with filters           │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ If: Base blob was last modified more than [30] days ago     │  │
│ │ Then: Move to Cool storage                                  │  │
│ │                                                              │  │
│ │ If: Base blob was last modified more than [90] days ago     │  │
│ │ Then: Move to Cold storage                                  │  │
│ │                                                              │  │
│ │ If: Base blob was last modified more than [180] days ago    │  │
│ │ Then: Move to Archive storage                               │  │
│ │                                                              │  │
│ │ If: Base blob was last modified more than [365] days ago    │  │
│ │ Then: Delete the blob                                       │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Filter by container or blob prefix:                                  │
│ Container: [logs]                                                   │
│ Blob prefix: [2023/] (only apply to blobs starting with "2023/") │
│                                                                       │
│ Also manage:                                                         │
│ ├── Snapshots: Delete snapshots after N days                     │
│ ├── Versions: Delete old versions after N days                   │
│ └── Previous versions: Move to cool/archive, then delete        │
│                                                                       │
│ Example lifecycle flow:                                              │
│ Day 0 ──→ Day 30 ──→ Day 90 ──→ Day 180 ──→ Day 365            │
│ HOT       COOL       COLD       ARCHIVE      DELETE              │
│ $0.018/GB $0.010/GB  $0.0045/GB $0.002/GB    $0/GB              │
│                                                                       │
│ ⚡ Runs once per day. Changes are not immediate.                    │
│ ⚡ Moving FROM Archive requires "rehydration" (takes hours).       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Versioning & Soft Delete

```
┌─────────────────────────────────────────────────────────────────────┐
│           DATA PROTECTION                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Blob Versioning:                                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Every time you update a blob, the old version is saved.     │  │
│ │                                                              │  │
│ │ photo.jpg                                                    │  │
│ │ ├── Current version: v3 (latest upload)                    │  │
│ │ ├── Version: v2 (previous upload)                          │  │
│ │ └── Version: v1 (original upload)                          │  │
│ │                                                              │  │
│ │ Access previous version:                                     │  │
│ │ https://st.blob.core.windows.net/images/photo.jpg?versionid=v2│
│ │                                                              │  │
│ │ Restore: Promote any version to be the current version.    │  │
│ │                                                              │  │
│ │ ⚡ Each version is billed for its storage.                  │  │
│ │   Use lifecycle management to delete old versions.         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Soft Delete (blob):                                                  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ When you delete a blob, it's NOT immediately gone.          │  │
│ │ It's kept in a "soft deleted" state for N days.            │  │
│ │                                                              │  │
│ │ Day 0: Delete photo.jpg                                     │  │
│ │ Day 1-7: photo.jpg still exists (soft deleted, hidden)     │  │
│ │   → You can UNDELETE it! (Portal → Show deleted → Undelete)│
│ │ Day 8: Permanently deleted (if retention = 7 days)         │  │
│ │                                                              │  │
│ │ ⚡ Enable this! It saves you from accidental deletes.      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Soft Delete (container):                                             │
│ ├── Same concept but for entire containers                       │
│ ├── Deleted containers recoverable for retention period          │
│ └── Portal → Containers → Show deleted containers → Restore    │
│                                                                       │
│ Point-in-Time Restore:                                               │
│ ├── Restore all blobs in a container to a specific timestamp    │
│ ├── "Undo all changes since yesterday 3 PM"                     │
│ ├── Requires versioning + change feed to be enabled              │
│ └── Great for recovering from ransomware or accidental bulk ops  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Static Website Hosting

```
┌─────────────────────────────────────────────────────────────────────┐
│           STATIC WEBSITE HOSTING                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Host static websites (HTML, CSS, JS) directly from Blob Storage!   │
│                                                                       │
│ Portal → Storage Account → Static website → Enable               │
│                                                                       │
│ Index document name: [index.html]                                  │
│ Error document path: [404.html]                                    │
│                                                                       │
│ [Save]                                                               │
│                                                                       │
│ This creates:                                                        │
│ ├── A special container named "$web"                              │
│ ├── Public endpoint: https://stmyappprod.z13.web.core.windows.net│
│ ├── Upload your HTML/CSS/JS files to $web container              │
│ └── Files are served publicly (no auth needed)                   │
│                                                                       │
│ For custom domain + CDN:                                             │
│ ├── Create Azure CDN or Front Door profile                       │
│ ├── Point origin to the static website endpoint                  │
│ ├── Map custom domain (www.mysite.com)                           │
│ ├── Enable HTTPS with managed certificate                        │
│ └── Result: Fast, global, cheap static website!                  │
│                                                                       │
│ ⚡ Cost: ~$1-5/month for a typical static site                     │
│ ⚡ AWS equivalent: S3 Static Website Hosting                       │
│ ⚡ GCP equivalent: Cloud Storage static hosting                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Blob Index Tags & Metadata

```
┌─────────────────────────────────────────────────────────────────────┐
│           TAGS & METADATA                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Blob Metadata (key-value pairs attached to blob):                   │
│ ├── Custom info stored WITH the blob                              │
│ ├── Not searchable (you need to know the blob to read them)     │
│ ├── Example: {"author": "john", "department": "marketing"}      │
│ └── Set via Portal → Blob → Properties → Metadata              │
│                                                                       │
│ Blob Index Tags (searchable tags):                                   │
│ ├── Key-value pairs that are SEARCHABLE across the account       │
│ ├── Find blobs by tag: "department = marketing"                  │
│ ├── Up to 10 tags per blob                                       │
│ ├── Tag name: 1-128 chars, Tag value: 0-256 chars               │
│ └── Example:                                                      │
│     department=marketing                                          │
│     project=website-redesign                                      │
│     status=approved                                               │
│                                                                       │
│ ⚡ Use Blob Index Tags for finding blobs across containers.       │
│   Use Metadata for blob-specific info you read with the blob.   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Immutable Storage (WORM)

```
┌─────────────────────────────────────────────────────────────────────┐
│           IMMUTABLE STORAGE                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ WORM = Write Once, Read Many                                        │
│ Once written, blobs CANNOT be modified or deleted for a set period.│
│ Required for regulatory compliance (SEC, FINRA, HIPAA).            │
│                                                                       │
│ Two types:                                                           │
│                                                                       │
│ 1. Time-based retention policy:                                      │
│    ├── Blobs cannot be modified/deleted for N days                 │
│    ├── Example: Retain financial records for 7 years              │
│    ├── Can be locked (cannot reduce retention after locking)     │
│    └── Portal → Container → Access policy → Add policy          │
│                                                                       │
│ 2. Legal hold:                                                       │
│    ├── Blobs cannot be modified/deleted until hold is removed    │
│    ├── No time limit — stays until you explicitly remove it     │
│    ├── Used for: legal cases, investigations                     │
│    └── Portal → Container → Access policy → Add legal hold      │
│                                                                       │
│ ⚡ Works at container level or blob version level.                 │
│ ⚡ Even Azure admins cannot delete immutable blobs!                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform & Bicep

### Terraform

```hcl
# Create a container
resource "azurerm_storage_container" "images" {
  name                  = "images"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# Upload a blob
resource "azurerm_storage_blob" "logo" {
  name                   = "logo.png"
  storage_account_name   = azurerm_storage_account.main.name
  storage_container_name = azurerm_storage_container.images.name
  type                   = "Block"
  source                 = "files/logo.png"
}

# Lifecycle management
resource "azurerm_storage_management_policy" "lifecycle" {
  storage_account_id = azurerm_storage_account.main.id

  rule {
    name    = "move-to-cool"
    enabled = true
    filters {
      blob_types = ["blockBlob"]
    }
    actions {
      base_blob {
        tier_to_cool_after_days_since_modification_greater_than    = 30
        tier_to_cold_after_days_since_modification_greater_than    = 90
        tier_to_archive_after_days_since_modification_greater_than = 180
        delete_after_days_since_modification_greater_than          = 365
      }
    }
  }
}
```

### Bicep

```bicep
// Blob container
resource container 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  name: '${storageAccount.name}/default/images'
  properties: {
    publicAccess: 'None'
  }
}

// Lifecycle management policy
resource lifecyclePolicy 'Microsoft.Storage/storageAccounts/managementPolicies@2023-01-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    policy: {
      rules: [
        {
          name: 'move-to-cool'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
            }
            actions: {
              baseBlob: {
                tierToCool: { daysAfterModificationGreaterThan: 30 }
                tierToCold: { daysAfterModificationGreaterThan: 90 }
                tierToArchive: { daysAfterModificationGreaterThan: 180 }
                delete: { daysAfterModificationGreaterThan: 365 }
              }
            }
          }
        }
      ]
    }
  }
}
```

---

## Part 10: az CLI & AzCopy Reference

```bash
# Create container
az storage container create \
  --name images \
  --account-name stmyappprod \
  --auth-mode login

# List containers
az storage container list --account-name stmyappprod --auth-mode login --output table

# Upload blob
az storage blob upload \
  --account-name stmyappprod \
  --container-name images \
  --name profile/user1.jpg \
  --file ./user1.jpg \
  --auth-mode login

# Upload folder (recursive)
az storage blob upload-batch \
  --account-name stmyappprod \
  --destination images \
  --source ./local-images/ \
  --auth-mode login

# List blobs
az storage blob list --account-name stmyappprod --container-name images --output table

# Download blob
az storage blob download \
  --account-name stmyappprod \
  --container-name images \
  --name profile/user1.jpg \
  --file ./downloaded-user1.jpg

# Change blob tier
az storage blob set-tier \
  --account-name stmyappprod \
  --container-name images \
  --name old-photo.jpg \
  --tier Cool

# Delete blob
az storage blob delete \
  --account-name stmyappprod \
  --container-name images \
  --name old-photo.jpg

# Delete container
az storage container delete --name old-container --account-name stmyappprod

# Generate SAS token
az storage blob generate-sas \
  --account-name stmyappprod \
  --container-name images \
  --name photo.jpg \
  --permissions r \
  --expiry 2024-12-31T23:59:00Z \
  --auth-mode login \
  --as-user

# ── AzCopy (high-performance bulk transfer) ──

# Copy local folder to blob storage
azcopy copy "./local-folder/*" "https://stmyappprod.blob.core.windows.net/images?<SAS>" --recursive

# Copy between storage accounts
azcopy copy "https://source.blob.core.windows.net/data?<SAS>" \
            "https://dest.blob.core.windows.net/data?<SAS>" --recursive

# Sync (like rsync — only copies changes)
azcopy sync "./local-folder" "https://st.blob.core.windows.net/images?<SAS>"
```

---

## Part 11: Real-World Patterns

```
Pattern 1: Application File Storage
├── Container: user-uploads (private)
├── App uploads files via SDK (Azure.Storage.Blobs)
├── Generate SAS URLs for user downloads (short expiry)
├── Lifecycle: Move to cool after 30 days
├── Soft delete: 7 days, versioning enabled
└── Access: Managed identity (no storage keys in code)

Pattern 2: Log Archival Pipeline
├── Container: logs (private)
├── App writes logs as append blobs (append-only)
├── Lifecycle: Hot → Cool (30d) → Archive (90d) → Delete (365d)
├── Blob Index Tags: app=api, env=prod, month=2024-01
└── Query archived logs: Rehydrate from archive (takes hours)

Pattern 3: CDN-Backed Media Storage
├── Container: media (blob-level public access)
├── Azure CDN profile → origin: blob endpoint
├── Custom domain: media.mysite.com (HTTPS)
├── Cache rules: 30 days for images, 7 days for CSS/JS
└── Cost: ~$0.08/GB transfer + minimal storage
```

---

## Quick Reference

```
Container = Folder-like grouping for blobs (no nesting)
Block Blob = Default type for files (99% use case)
Append Blob = For log files (append-only)
Page Blob = For VM disks (random read/write)

Access tiers (per blob): Hot → Cool → Cold → Archive
Lifecycle management = Auto-move blobs between tiers

Versioning = Keep previous versions of blobs
Soft delete = Recover deleted blobs within retention period
Point-in-time restore = Undo changes to a specific timestamp

Static website = Host HTML/CSS/JS directly from $web container
Immutable = WORM (cannot delete/modify for compliance)

AzCopy = High-performance bulk transfer tool
```

---

## What's Next?

Next chapter: [Chapter 24: Azure Files & File Sync](24-azure-files.md) — Create cloud file shares accessible via SMB/NFS, and sync with on-premises servers.
