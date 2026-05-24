# Chapter 22: Azure Storage Accounts

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Storage Account Fundamentals](#part-1-storage-account-fundamentals)
- [Part 2: Creating a Storage Account (Full Portal Walkthrough)](#part-2-creating-a-storage-account-full-portal-walkthrough)
- [Part 3: Replication Options](#part-3-replication-options)
- [Part 4: Access Tiers](#part-4-access-tiers)
- [Part 5: Security & Access Control](#part-5-security--access-control)
- [Part 6: Networking](#part-6-networking)
- [Part 7: Terraform & Bicep](#part-7-terraform--bicep)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

An Azure Storage Account is a container that groups together all your Azure Storage services — Blobs, Files, Queues, and Tables. Think of it like a "folder" that holds all your storage data. Every time you want to store files, messages, or data in Azure, you first need a Storage Account.

```
What you'll learn:
├── Storage Account Fundamentals
│   ├── What is a Storage Account
│   ├── Services inside (Blob, Files, Queue, Table)
│   ├── Account types (Standard vs Premium)
│   └── Performance tiers & replication
├── Creating a Storage Account (Full Portal Walkthrough)
│   ├── Basics (name, region, performance, redundancy)
│   ├── Advanced (security, data lake, access tier)
│   ├── Networking (public/private, firewall, private endpoints)
│   ├── Data protection (soft delete, versioning, backups)
│   └── Encryption (Microsoft-managed vs customer-managed keys)
├── Replication (LRS, ZRS, GRS, RA-GRS, GZRS, RA-GZRS)
├── Access Tiers (Hot, Cool, Cold, Archive)
├── Security (Shared Key, SAS, Azure AD, encryption)
├── Networking (firewall, VNet, private endpoints)
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: Storage Account Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           STORAGE ACCOUNT CONCEPT                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is a Storage Account?                                          │
│ ├── A top-level container for Azure Storage services              │
│ ├── Provides a unique namespace for your data                    │
│ ├── Every object has a URL: https://<account>.blob.core.windows.net│
│ ├── Settings like replication, access tier apply to entire account│
│ └── You need at least one to use ANY Azure storage service       │
│                                                                       │
│ Services inside a Storage Account:                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ Storage Account: stmyappprod                                │  │
│ │ ├── Blob Storage (containers + blobs)                       │  │
│ │ │   ├── Container: images/                                 │  │
│ │ │   │   ├── photo1.jpg                                    │  │
│ │ │   │   └── photo2.png                                    │  │
│ │ │   ├── Container: backups/                                │  │
│ │ │   │   └── db-backup-2024.sql                           │  │
│ │ │   └── Container: logs/                                   │  │
│ │ │       └── app-2024-01-15.log                           │  │
│ │ │                                                           │  │
│ │ ├── Azure Files (file shares)                               │  │
│ │ │   └── Share: shared-files/                               │  │
│ │ │       ├── documents/                                     │  │
│ │ │       └── config/                                        │  │
│ │ │                                                           │  │
│ │ ├── Queue Storage (message queues)                          │  │
│ │ │   ├── Queue: order-processing                           │  │
│ │ │   └── Queue: notifications                              │  │
│ │ │                                                           │  │
│ │ └── Table Storage (NoSQL key-value store)                  │  │
│ │     ├── Table: Users                                       │  │
│ │     └── Table: Logs                                        │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Account types:                                                       │
│ ┌────────────────────┬────────────────────────────────────────┐   │
│ │ Type               │ Description                            │   │
│ ├────────────────────┼────────────────────────────────────────┤   │
│ │ Standard general   │ HDD-based. Blobs, Files, Queues, Tables│   │
│ │ purpose v2 (GPv2)  │ Most common. Cheapest for storage.    │   │
│ │                    │ Supports all tiers (Hot/Cool/Archive). │   │
│ │                    │                                        │   │
│ │ Premium block blob │ SSD-based. For high-transaction blob  │   │
│ │                    │ workloads (low latency). No Files/Queue│   │
│ │                    │                                        │   │
│ │ Premium file share │ SSD-based. For Azure Files only.      │   │
│ │                    │ High IOPS for file shares.             │   │
│ │                    │                                        │   │
│ │ Premium page blob  │ SSD-based. For page blobs (VM disks). │   │
│ │                    │ Used by unmanaged disks (legacy).      │   │
│ └────────────────────┴────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ 99% of the time, you'll use Standard general-purpose v2 (GPv2)│
│                                                                       │
│ ⚡ AWS equivalent: S3 (for blobs), EFS (for files), SQS (for queues)│
│ ⚡ GCP equivalent: Cloud Storage, Filestore                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Storage Account (Full Portal Walkthrough)

```
Console → Storage accounts → Create

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 1: BASICS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-storage-prod ▼]                                │
│                                                                       │
│ Storage account name: [stmyappprod]                                │
│ ⚡ Rules: 3-24 chars, lowercase letters and numbers ONLY.          │
│   No hyphens, underscores, or uppercase!                          │
│   Globally unique across ALL of Azure.                            │
│   Convention: st{purpose}{env} → stmyappprod                     │
│                                                                       │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Performance:                                                         │
│ ● Standard (HDD) — recommended for most workloads                │
│   ├── Lower cost per GB                                           │
│   ├── Supports all storage services (Blob, File, Queue, Table)  │
│   └── Supports all access tiers (Hot, Cool, Cold, Archive)      │
│                                                                       │
│ ○ Premium (SSD) — for high-performance workloads                 │
│   ├── Much lower latency, higher IOPS                            │
│   ├── Choose sub-type:                                            │
│   │   ○ Block blobs (high transaction rate blobs)               │
│   │   ○ File shares (high-performance file shares)              │
│   │   ○ Page blobs (VM disks — legacy)                          │
│   └── No access tiers (always "premium" tier)                   │
│                                                                       │
│ Redundancy: [Geo-redundant storage (GRS) ▼]                      │
│   ○ Locally redundant storage (LRS) — 3 copies, 1 datacenter   │
│   ○ Zone-redundant storage (ZRS) — 3 copies, 3 AZs             │
│   ● Geo-redundant storage (GRS) — 6 copies, 2 regions          │
│   ○ Geo-zone-redundant storage (GZRS) — best durability        │
│                                                                       │
│ [Next: Advanced >]                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 2: ADVANCED                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Security ──                                                      │
│                                                                       │
│ Require secure transfer (HTTPS): ☑ (always enable!)              │
│ ⚡ Forces all requests to use HTTPS. Never disable in production. │
│                                                                       │
│ Allow enabling anonymous access on individual containers: ☐      │
│ ⚡ Unchecked = no container can be made public. Recommended.      │
│                                                                       │
│ Enable storage account key access: ☑                              │
│ ⚡ If disabled, only Azure AD/RBAC can access storage.           │
│   More secure but requires identity-based auth everywhere.      │
│                                                                       │
│ Default to Microsoft Entra authorization in the Azure portal: ☐  │
│                                                                       │
│ Minimum TLS version: [Version 1.2 ▼]                             │
│ ⚡ Always use TLS 1.2 (older versions have vulnerabilities).     │
│                                                                       │
│ Permitted scope for copy operations: [From any storage account ▼]│
│                                                                       │
│ ── Data Lake Storage Gen2 ──                                        │
│                                                                       │
│ Enable hierarchical namespace: ☐                                  │
│ ⚡ Enables Data Lake Storage Gen2 (real folders, ACLs).           │
│   Enable ONLY if you need analytics/big data workloads.          │
│   Cannot be disabled once enabled!                               │
│                                                                       │
│ ── Blob storage ──                                                  │
│                                                                       │
│ Enable SFTP: ☐ (for SFTP access to blob storage)                │
│ Enable network file system v3: ☐ (Linux NFS access)             │
│                                                                       │
│ Access tier (default):                                              │
│ ● Hot (frequently accessed data — higher storage cost, lower access)│
│ ○ Cool (infrequently accessed — lower storage, higher access)   │
│ ⚡ This is the DEFAULT tier for new blobs. You can override per blob.│
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 3: NETWORKING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Network access:                                                      │
│ ○ Enable public access from all networks                         │
│ ● Enable public access from selected virtual networks and IPs   │
│ ○ Disable public access and use private access                   │
│                                                                       │
│ (If "selected networks"):                                           │
│ Virtual networks:                                                    │
│ [+ Add virtual network]                                            │
│   VNet: vnet-prod, Subnet: subnet-app                            │
│                                                                       │
│ Firewall:                                                            │
│ IP ranges: [203.0.113.0/24] (your office IP)                     │
│ ☑ Allow Azure services on the trusted services list              │
│ ⚡ If you restrict access, Azure services (Monitor, Backup)      │
│   still need access. Check this box.                             │
│                                                                       │
│ Private endpoints:                                                   │
│ [+ Add private endpoint]                                           │
│ Name: pe-stmyappprod-blob                                         │
│ Sub-resource: blob                                                 │
│ VNet: vnet-prod, Subnet: subnet-endpoints                        │
│ ⚡ Creates private IP for storage in your VNet.                   │
│   Apps access storage via private network (no public internet). │
│                                                                       │
│ [Next: Data protection >]                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 4: DATA PROTECTION                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Recovery ──                                                      │
│                                                                       │
│ ☑ Enable point-in-time restore for containers                    │
│   Retention: [7] days                                             │
│   ⚡ Restore blobs to any point in time (like a time machine!). │
│                                                                       │
│ ☑ Enable soft delete for blobs                                   │
│   Retention: [7] days                                             │
│   ⚡ Deleted blobs are kept for 7 days (can undelete).          │
│                                                                       │
│ ☑ Enable soft delete for containers                              │
│   Retention: [7] days                                             │
│   ⚡ Deleted containers recoverable for 7 days.                 │
│                                                                       │
│ ☑ Enable soft delete for file shares                             │
│   Retention: [7] days                                             │
│                                                                       │
│ ── Tracking ──                                                      │
│                                                                       │
│ ☑ Enable versioning for blobs                                    │
│   ⚡ Every update creates a new version. Access previous versions.│
│                                                                       │
│ ☑ Enable blob change feed                                        │
│   ⚡ Log of all changes (create/update/delete) to blobs.        │
│   Useful for auditing and event-driven processing.              │
│                                                                       │
│ ── Access control ──                                                │
│                                                                       │
│ ☑ Enable version-level immutability support                      │
│   ⚡ WORM (Write Once, Read Many) — for compliance.             │
│   Blobs cannot be modified or deleted during retention period.   │
│                                                                       │
│ [Next: Encryption >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 5: ENCRYPTION                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Encryption type:                                                     │
│ ● Microsoft-managed keys (MMK) — default, easiest               │
│   Azure manages the encryption keys automatically.               │
│ ○ Customer-managed keys (CMK) — for compliance/control          │
│   You provide keys from Key Vault. You manage rotation.         │
│                                                                       │
│ Enable support for customer-managed keys:                           │
│ ● Blobs and files only (default)                                  │
│ ○ All service types (includes queues and tables)                 │
│                                                                       │
│ Enable infrastructure encryption: ☐                               │
│ ⚡ Double encryption — data encrypted at both service level      │
│   AND infrastructure level. For high-security requirements.     │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Replication Options

```
┌─────────────────────────────────────────────────────────────────────┐
│           REPLICATION / REDUNDANCY                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────┬──────┬──────────┬──────────┬───────────────────────┐ │
│ │ Option   │Copies│ Scope    │Durability│ Use case              │ │
│ ├──────────┼──────┼──────────┼──────────┼───────────────────────┤ │
│ │ LRS      │ 3    │ 1 DC     │ 11 nines │ Dev/test, non-critical│ │
│ │ ZRS      │ 3    │ 3 AZs    │ 12 nines │ Prod, high availability│ │
│ │ GRS      │ 6    │ 2 regions│ 16 nines │ DR, geo-redundancy    │ │
│ │ RA-GRS   │ 6    │ 2 regions│ 16 nines │ Read from secondary   │ │
│ │ GZRS     │ 6    │ 2 regions│ 16 nines │ Best durability       │ │
│ │ RA-GZRS  │ 6    │ 2 regions│ 16 nines │ Best (read + geo + AZ)│ │
│ └──────────┴──────┴──────────┴──────────┴───────────────────────┘ │
│                                                                       │
│ Visual:                                                              │
│                                                                       │
│ LRS (Locally Redundant):                                            │
│ ┌─── Datacenter ────────────────┐                                  │
│ │ Copy 1  │  Copy 2  │  Copy 3 │    3 copies in 1 datacenter    │
│ └──────────────────────────────┘                                   │
│ ⚡ Cheapest. Risk: datacenter failure loses ALL data.             │
│                                                                       │
│ ZRS (Zone Redundant):                                                │
│ ┌── AZ 1 ──┐ ┌── AZ 2 ──┐ ┌── AZ 3 ──┐                        │
│ │  Copy 1  │ │  Copy 2  │ │  Copy 3  │   1 copy per AZ         │
│ └──────────┘ └──────────┘ └──────────┘                           │
│ ⚡ Survives datacenter failure. Best for production.              │
│                                                                       │
│ GRS (Geo Redundant):                                                 │
│ ┌─── Primary Region ──────┐  ┌─── Secondary Region ───────┐     │
│ │ Copy 1 │ Copy 2 │ Copy 3│  │ Copy 4 │ Copy 5 │ Copy 6  │     │
│ └─────────────────────────┘  └────────────────────────────┘      │
│ ⚡ 6 copies across 2 regions. Secondary is READ-ONLY (RA-GRS). │
│   Replication is asynchronous (few minutes lag).                │
│                                                                       │
│ Decision guide:                                                      │
│ ├── Dev/test → LRS (cheapest)                                   │
│ ├── Production → ZRS (zone redundant)                           │
│ ├── Disaster recovery → GRS or GZRS                            │
│ └── Need read from secondary → RA-GRS or RA-GZRS              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Access Tiers

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACCESS TIERS (for Blob Storage)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────┬──────────────┬──────────────┬────────────────────┐   │
│ │ Tier     │ Storage cost │ Access cost  │ Best for            │   │
│ ├──────────┼──────────────┼──────────────┼────────────────────┤   │
│ │ Hot      │ Highest      │ Lowest       │ Frequent access    │   │
│ │ Cool     │ Lower        │ Higher       │ Infrequent (30+ d) │   │
│ │ Cold     │ Even lower   │ Even higher  │ Rarely (90+ days)  │   │
│ │ Archive  │ Cheapest     │ Highest      │ Compliance (180+ d)│   │
│ └──────────┴──────────────┴──────────────┴────────────────────┘   │
│                                                                       │
│ ⚡ Hot: For data you access often (app data, images, current logs)│
│ ⚡ Cool: Data kept for 30+ days, rarely accessed (old logs)      │
│ ⚡ Cold: Data kept for 90+ days, very rarely accessed            │
│ ⚡ Archive: Data kept for 180+ days (compliance, legal holds)    │
│   Archive blobs are OFFLINE — takes hours to rehydrate!         │
│                                                                       │
│ Cost example (per GB/month, Central India):                         │
│ ├── Hot:     $0.018/GB                                            │
│ ├── Cool:    $0.010/GB (44% cheaper storage)                    │
│ ├── Cold:    $0.0045/GB (75% cheaper storage)                   │
│ ├── Archive: $0.002/GB  (89% cheaper storage!)                  │
│ │                                                                │
│ │ But: Reading is much more expensive in cool/cold/archive!   │
│ │ Hot read:     $0.004/10K operations                          │
│ │ Cool read:    $0.01/10K operations                           │
│ │ Archive read: $5.00/10K operations!!                         │
│                                                                       │
│ Set tier per blob or use Lifecycle Management for automation.    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Security & Access Control

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY & ACCESS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Authentication methods (how to access storage):                      │
│                                                                       │
│ 1. Storage Account Keys (Shared Key)                                │
│    ├── Two keys (key1, key2) per account                          │
│    ├── Full access to EVERYTHING in the account                  │
│    ├── Like a master password — very powerful                    │
│    ├── Rotate regularly (Portal → Access keys → Regenerate)    │
│    └── ⚠️ Avoid! Use Azure AD/managed identity instead.         │
│                                                                       │
│ 2. Shared Access Signature (SAS)                                    │
│    ├── Limited-time, limited-permission access token              │
│    ├── Embed in URL: https://st.blob.core.windows.net/cont/file?sv=...│
│    ├── Types:                                                     │
│    │   Account SAS: All services (blob, file, queue, table)    │
│    │   Service SAS: One service (e.g., blob only)              │
│    │   User Delegation SAS: Signed by Azure AD (most secure)  │
│    ├── You set: expiry, permissions (read/write), IP range     │
│    └── ⚡ Use User Delegation SAS when possible (no key needed)│
│                                                                       │
│ 3. Azure AD / RBAC (recommended!)                                   │
│    ├── Use managed identities or Azure AD users                  │
│    ├── Assign RBAC roles:                                        │
│    │   Storage Blob Data Reader                                 │
│    │   Storage Blob Data Contributor                            │
│    │   Storage Blob Data Owner                                  │
│    │   Storage Queue Data Contributor                           │
│    │   Storage Table Data Contributor                           │
│    ├── No keys or tokens needed                                  │
│    └── ⚡ Best practice: Use managed identities + RBAC          │
│                                                                       │
│ 4. Anonymous access (public blobs)                                  │
│    ├── Container access level: Blob or Container                 │
│    ├── Anyone with URL can read                                  │
│    └── ⚠️ Only for truly public content (public images, etc.)  │
│                                                                       │
│ Encryption:                                                          │
│ ├── All data encrypted at rest (AES-256) — always, by default  │
│ ├── All data encrypted in transit (HTTPS) — enforced            │
│ ├── Microsoft-managed keys (default) or customer-managed (CMK) │
│ └── Infrastructure encryption (optional double encryption)      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Networking

```
┌─────────────────────────────────────────────────────────────────────┐
│           NETWORKING                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Portal → Storage Account → Networking                             │
│                                                                       │
│ Three access levels:                                                 │
│                                                                       │
│ 1. Public access from all networks (default — least secure)       │
│    Anyone with the key/SAS/RBAC can access from anywhere.        │
│                                                                       │
│ 2. Selected networks (recommended for production)                  │
│    ├── Allow from specific VNets/subnets                         │
│    ├── Allow from specific IP addresses                          │
│    ├── Allow trusted Azure services                              │
│    └── Everything else is denied                                 │
│                                                                       │
│ 3. Disabled (private access only)                                   │
│    ├── No public access at all                                   │
│    ├── Access ONLY through private endpoints                    │
│    └── Most secure option                                        │
│                                                                       │
│ Private endpoints:                                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Storage Account: stmyappprod                                 │  │
│ │ ├── Private Endpoint (blob): pe-st-blob → 10.0.5.4         │  │
│ │ ├── Private Endpoint (file): pe-st-file → 10.0.5.5         │  │
│ │ └── Private Endpoint (table): pe-st-table → 10.0.5.6       │  │
│ │                                                              │  │
│ │ Each sub-resource (blob, file, queue, table) needs its own  │  │
│ │ private endpoint if you want private access to that service.│  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_storage_account" "main" {
  name                     = "stmyappprod"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "ZRS"
  account_kind             = "StorageV2"
  min_tls_version          = "TLS1_2"
  
  allow_nested_items_to_be_public = false
  
  blob_properties {
    versioning_enabled  = true
    change_feed_enabled = true
    
    delete_retention_policy {
      days = 7
    }
    
    container_delete_retention_policy {
      days = 7
    }
  }

  network_rules {
    default_action             = "Deny"
    virtual_network_subnet_ids = [azurerm_subnet.app.id]
    bypass                     = ["AzureServices"]
  }
}
```

### Bicep

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'stmyappprod'
  location: location
  sku: {
    name: 'Standard_ZRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    }
  }
}
```

---

## Part 8: az CLI Reference

```bash
# Create storage account
az storage account create \
  --name stmyappprod \
  --resource-group rg-storage-prod \
  --location centralindia \
  --sku Standard_ZRS \
  --kind StorageV2 \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

# List storage accounts
az storage account list --resource-group rg-storage-prod --output table

# Show storage account details
az storage account show --name stmyappprod --resource-group rg-storage-prod

# Get access keys
az storage account keys list --name stmyappprod --resource-group rg-storage-prod

# Regenerate key
az storage account keys renew --name stmyappprod --resource-group rg-storage-prod --key key1

# Update replication
az storage account update --name stmyappprod --resource-group rg-storage-prod --sku Standard_GRS

# Enable soft delete
az storage blob service-properties delete-policy update \
  --account-name stmyappprod \
  --enable true \
  --days-retained 7

# Add network rule (allow VNet)
az storage account network-rule add \
  --account-name stmyappprod \
  --resource-group rg-storage-prod \
  --vnet-name vnet-prod \
  --subnet subnet-app

# Delete storage account
az storage account delete --name stmyappprod --resource-group rg-storage-prod --yes
```

---

## Part 9: Real-World Patterns

```
Pattern 1: Secure Production Storage
├── Standard GPv2 with ZRS replication
├── Networking: Private endpoints only (no public access)
├── Auth: Azure AD RBAC (no shared keys)
├── Soft delete: 7 days for blobs and containers
├── Versioning: Enabled for data protection
├── Lifecycle management: Hot → Cool after 30 days → Archive after 90
├── Encryption: Customer-managed keys from Key Vault
└── Monitoring: Diagnostic logs → Log Analytics

Pattern 2: Static Website Hosting
├── Standard GPv2 with LRS (cost-effective)
├── Enable static website (Portal → Static website → Enable)
├── Index document: index.html
├── Error document: 404.html
├── Put Azure CDN or Front Door in front for caching + custom domain
└── Enable CORS if needed for API calls
```

---

## Quick Reference

```
Storage Account = Container for all Azure storage services
Naming = 3-24 chars, lowercase + numbers only, globally unique
Standard GPv2 = Default choice (supports everything)
Premium = SSD-based (for low-latency workloads)

Replication: LRS (cheapest) → ZRS (prod) → GRS (DR) → GZRS (best)
Access Tiers: Hot (frequent) → Cool (30d) → Cold (90d) → Archive (180d)

Auth: Azure AD/RBAC (best) > SAS tokens > Storage keys (avoid)

Always enable: HTTPS required, TLS 1.2, soft delete, versioning
Always restrict: Network access (VNet/private endpoints)
```

---

## What's Next?

Next chapter: [Chapter 23: Blob Storage](23-blob-storage.md) — Deep dive into containers, blobs, lifecycle management, and real-world patterns for storing files in Azure.
