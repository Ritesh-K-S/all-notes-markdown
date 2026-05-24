# Chapter 40 — Cloud KMS (Key Management Service)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud KMS Fundamentals](#part-1--cloud-kms-fundamentals)
- [Part 2: Resource Hierarchy — Key Rings, Keys, Key Versions](#part-2--resource-hierarchy--key-rings-keys-key-versions)
- [Part 3: Key Purposes & Algorithms](#part-3--key-purposes--algorithms)
- [Part 4: Encryption & Decryption](#part-4--encryption--decryption)
- [Part 5: Key Rotation](#part-5--key-rotation)
- [Part 6: CMEK — Customer-Managed Encryption Keys](#part-6--cmek--customer-managed-encryption-keys)
- [Part 7: CSEK — Customer-Supplied Encryption Keys](#part-7--csek--customer-supplied-encryption-keys)
- [Part 8: Envelope Encryption](#part-8--envelope-encryption)
- [Part 9: Asymmetric Keys — Signing & Encryption](#part-9--asymmetric-keys--signing--encryption)
- [Part 10: Key Import & External Key Manager (EKM)](#part-10--key-import--external-key-manager-ekm)
- [Part 11: Key Access Justifications & Assured Workloads](#part-11--key-access-justifications--assured-workloads)
- [Part 12: IAM & Security](#part-12--iam--security)
- [Part 13: Monitoring & Audit Logging](#part-13--monitoring--audit-logging)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Cloud KMS is Google Cloud's managed service for creating, managing, and using cryptographic keys. It provides a central place to manage encryption keys for your cloud services, supporting symmetric encryption, asymmetric encryption, asymmetric signing, and MAC signing. Cloud KMS integrates with nearly every GCP service via CMEK (Customer-Managed Encryption Keys) and supports hardware-backed keys via Cloud HSM.

---

## Part 1 — Cloud KMS Fundamentals

### What Is Cloud KMS?

```
┌─────────────────────────────────────────────────────────────────┐
│                  CLOUD KMS OVERVIEW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Google Cloud encrypts ALL data at rest by default using         │
│  Google-managed keys. Cloud KMS lets YOU control the keys.      │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                Encryption Options                         │   │
│  │                                                           │   │
│  │  Level 1: Google Default Encryption (GDEF)               │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │ • Google manages everything                       │   │   │
│  │  │ • AES-256 for data at rest                       │   │   │
│  │  │ • Automatic key rotation                         │   │   │
│  │  │ • No configuration needed                        │   │   │
│  │  │ • Free                                            │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  │                                                           │   │
│  │  Level 2: CMEK (Customer-Managed Encryption Keys)        │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │ • You create & manage keys in Cloud KMS          │   │   │
│  │  │ • You control rotation, access, lifecycle        │   │   │
│  │  │ • Keys stay in Google Cloud                      │   │   │
│  │  │ • GCP services use YOUR key to encrypt data      │   │   │
│  │  │ • Disable key = data inaccessible                │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  │                                                           │   │
│  │  Level 3: CSEK (Customer-Supplied Encryption Keys)       │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │ • You provide the raw key material               │   │   │
│  │  │ • Google uses it to encrypt, doesn't store it    │   │   │
│  │  │ • Only for Compute Engine disks & Cloud Storage   │   │   │
│  │  │ • You must supply key with every request         │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  │                                                           │   │
│  │  Level 4: EKM (External Key Manager)                     │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │ • Keys stored outside Google Cloud entirely      │   │   │
│  │  │ • Managed by third-party HSM (e.g., Thales)     │   │   │
│  │  │ • Google never sees the key material             │   │   │
│  │  │ • Highest control / sovereignty                  │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Cloud KMS | AWS KMS | Azure Key Vault |
|---------|--------------|---------|-----------------|
| Service | Cloud KMS | AWS KMS | Key Vault |
| Key hierarchy | Key Ring → Key → Key Version | Key (flat) | Vault → Key |
| HSM-backed | Cloud HSM (FIPS 140-2 L3) | Custom Key Store (CloudHSM) | Managed HSM (FIPS 140-2 L3) |
| External keys | EKM (External Key Manager) | External Key Store (XKS) | Managed HSM with BYOK |
| Default encryption | Google-managed (free) | AWS-managed (free) | Microsoft-managed (free) |
| CMEK support | ~100+ services | ~70+ services | ~50+ services |
| Rotation | Automatic (configurable period) | Automatic (annual) | Automatic (configurable) |
| Asymmetric keys | Yes (RSA, EC) | Yes (RSA, EC) | Yes (RSA, EC) |
| Pricing (symmetric) | $0.06/key version/month | $1.00/key/month | $0.03/key operation (10K) |
| Free API calls | 10,000/month | 20,000/month | N/A |

### Pricing

| Component | Cost |
|-----------|------|
| Software key versions (active) | $0.06/month |
| HSM key versions (active) | $1.00–$2.50/month |
| EKM key versions (active) | $3.00/month |
| Key operations (encrypt/decrypt) | $0.03 per 10,000 operations |
| Key operations (HSM) | $0.03–$0.15 per 10,000 operations |
| Destroyed key versions | Free |

---

## Part 2 — Resource Hierarchy — Key Rings, Keys, Key Versions

### KMS Resource Model

```
┌──────────────────────────────────────────────────────────────┐
│         CLOUD KMS RESOURCE HIERARCHY                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Project                                                       │
│  └── Location (region or "global")                            │
│      └── Key Ring (logical grouping)                          │
│          ├── Crypto Key 1 (e.g., "app-data-key")             │
│          │   ├── Key Version 1 (ENABLED — primary)           │
│          │   ├── Key Version 2 (ENABLED)                     │
│          │   └── Key Version 3 (DESTROYED)                   │
│          ├── Crypto Key 2 (e.g., "backup-key")               │
│          │   └── Key Version 1 (ENABLED — primary)           │
│          └── Crypto Key 3 (e.g., "signing-key")              │
│              └── Key Version 1 (ENABLED — primary)           │
│                                                                │
│  Resource name format:                                         │
│  projects/PROJECT/locations/LOCATION/keyRings/RING/           │
│    cryptoKeys/KEY/cryptoKeyVersions/VERSION                   │
│                                                                │
│  Example:                                                      │
│  projects/my-project/locations/us-central1/keyRings/          │
│    prod-keys/cryptoKeys/app-data/cryptoKeyVersions/1         │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ IMPORTANT CONSTRAINTS:                                  │  │
│  │ • Key Rings CANNOT be deleted (only keys inside them)  │  │
│  │ • Key Rings CANNOT be renamed or moved                 │  │
│  │ • Key Versions CANNOT be deleted immediately           │  │
│  │   (scheduled destruction with 24h minimum wait)        │  │
│  │ • Choose locations carefully — they're permanent       │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Key Version States

```
┌──────────────────────────────────────────────────────────────┐
│         KEY VERSION LIFECYCLE                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────────┐                                      │
│  │ PENDING_GENERATION │  (generating key material)           │
│  └─────────┬──────────┘                                      │
│            ▼                                                  │
│  ┌────────────────────┐                                      │
│  │     ENABLED        │  ← can encrypt & decrypt             │
│  └─────────┬──────────┘                                      │
│            │                                                  │
│       ┌────┴────┐                                            │
│       ▼         ▼                                            │
│  ┌─────────┐  ┌──────────────────────┐                      │
│  │DISABLED │  │DESTROY_SCHEDULED     │                      │
│  │         │  │(24h–120 day wait)    │                      │
│  └────┬────┘  └──────────┬───────────┘                      │
│       │                  │                                    │
│       │ re-enable        │ wait period expires                │
│       ▼                  ▼                                    │
│  ┌─────────┐  ┌──────────────────────┐                      │
│  │ENABLED  │  │     DESTROYED        │                      │
│  │(again)  │  │ (key material gone)  │                      │
│  └─────────┘  │ (irreversible!)      │                      │
│               └──────────────────────┘                      │
│                                                                │
│  Notes:                                                        │
│  • DISABLED: cannot encrypt/decrypt but material exists      │
│  • DESTROY_SCHEDULED: can be restored before window expires  │
│  • DESTROYED: permanent — encrypted data is UNRECOVERABLE   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating Key Rings and Keys

```bash
# Create a key ring
gcloud kms keyrings create prod-keys \
    --location=us-central1 \
    --project=my-project

# Create a symmetric encryption key
gcloud kms keys create app-data-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --purpose=encryption \
    --rotation-period=90d \
    --next-rotation-time="2026-08-01T00:00:00Z"

# List keys in a key ring
gcloud kms keys list \
    --keyring=prod-keys \
    --location=us-central1
```

---

## Part 3 — Key Purposes & Algorithms

### Key Purposes

| Purpose | Use Case | Algorithms |
|---------|----------|------------|
| `ENCRYPT_DECRYPT` | Symmetric encryption/decryption | AES-256-GCM |
| `ASYMMETRIC_SIGN` | Digital signing | RSA (2048–4096), EC (P256, P384) |
| `ASYMMETRIC_DECRYPT` | Asymmetric encryption | RSA (2048–4096) with OAEP |
| `MAC` | Message authentication codes | HMAC-SHA256 |
| `RAW_ENCRYPT_DECRYPT` | Raw symmetric encryption (no envelope) | AES-256-GCM, AES-256-CBC, AES-128-CTR |

### Algorithm Details

```
┌──────────────────────────────────────────────────────────────┐
│         KEY ALGORITHMS BY PURPOSE                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  SYMMETRIC ENCRYPTION (ENCRYPT_DECRYPT):                      │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Algorithm                   │ Key Size  │ Mode         │ │
│  │ GOOGLE_SYMMETRIC_ENCRYPTION │ 256-bit   │ AES-GCM     │ │
│  └─────────────────────────────────────────────────────────┘ │
│  → Only 1 algorithm for symmetric keys                      │
│  → Google handles IV/nonce generation                       │
│  → Max plaintext size: 64 KiB per request                  │
│                                                                │
│  ASYMMETRIC SIGNING (ASYMMETRIC_SIGN):                       │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ RSA_SIGN_PSS_2048_SHA256                                │ │
│  │ RSA_SIGN_PSS_3072_SHA256                                │ │
│  │ RSA_SIGN_PSS_4096_SHA256 / SHA512                      │ │
│  │ RSA_SIGN_PKCS1_2048_SHA256                              │ │
│  │ RSA_SIGN_PKCS1_3072_SHA256                              │ │
│  │ RSA_SIGN_PKCS1_4096_SHA256 / SHA512                    │ │
│  │ EC_SIGN_P256_SHA256                                     │ │
│  │ EC_SIGN_P384_SHA384                                     │ │
│  │ EC_SIGN_SECP256K1_SHA256  (blockchain)                 │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                                │
│  ASYMMETRIC ENCRYPTION (ASYMMETRIC_DECRYPT):                 │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ RSA_DECRYPT_OAEP_2048_SHA256                            │ │
│  │ RSA_DECRYPT_OAEP_3072_SHA256                            │ │
│  │ RSA_DECRYPT_OAEP_4096_SHA256 / SHA512                  │ │
│  │ RSA_DECRYPT_OAEP_2048_SHA1 (legacy)                    │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                                │
│  MAC (HMAC):                                                  │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ HMAC_SHA256                                              │ │
│  │ HMAC_SHA1 (legacy)                                      │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Protection Levels

| Level | Description | FIPS 140-2 | Cost |
|-------|-------------|------------|------|
| `SOFTWARE` | Keys processed in software | Level 1 | $0.06/version/month |
| `HSM` | Keys in Cloud HSM | Level 3 | $1.00–$2.50/version/month |
| `EXTERNAL` | Keys in external KMS (EKM) | Depends on provider | $3.00/version/month |
| `EXTERNAL_VPC` | EKM via VPC network | Depends on provider | $3.00/version/month |

---

## Part 4 — Encryption & Decryption

### Symmetric Encryption

```bash
# Encrypt a file
gcloud kms encrypt \
    --key=app-data-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --plaintext-file=secret.txt \
    --ciphertext-file=secret.txt.enc

# Decrypt a file
gcloud kms decrypt \
    --key=app-data-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --ciphertext-file=secret.txt.enc \
    --plaintext-file=secret.txt
```

### Programmatic Encryption

```python
# pip install google-cloud-kms
from google.cloud import kms

def encrypt_symmetric(project_id, location_id, key_ring_id, key_id, plaintext):
    """Encrypt data using a symmetric key."""
    client = kms.KeyManagementServiceClient()
    
    key_name = client.crypto_key_path(
        project_id, location_id, key_ring_id, key_id
    )
    
    # Encrypt — uses the PRIMARY key version automatically
    response = client.encrypt(
        request={
            "name": key_name,
            "plaintext": plaintext.encode("utf-8"),
            # Optional: additional authenticated data (AAD)
            # "additional_authenticated_data": b"context-info",
        }
    )
    
    return response.ciphertext  # bytes

def decrypt_symmetric(project_id, location_id, key_ring_id, key_id, ciphertext):
    """Decrypt data using a symmetric key."""
    client = kms.KeyManagementServiceClient()
    
    key_name = client.crypto_key_path(
        project_id, location_id, key_ring_id, key_id
    )
    
    # Decrypt — automatically finds the right key version
    response = client.decrypt(
        request={
            "name": key_name,
            "ciphertext": ciphertext,
        }
    )
    
    return response.plaintext.decode("utf-8")
```

```go
// Go encryption example
package main

import (
    "context"
    "fmt"
    kms "cloud.google.com/go/kms/apiv1"
    kmspb "cloud.google.com/go/kms/apiv1/kmspb"
)

func encryptSymmetric(projectID, locationID, keyRingID, keyID string, plaintext []byte) ([]byte, error) {
    ctx := context.Background()
    client, err := kms.NewKeyManagementClient(ctx)
    if err != nil {
        return nil, err
    }
    defer client.Close()

    keyName := fmt.Sprintf(
        "projects/%s/locations/%s/keyRings/%s/cryptoKeys/%s",
        projectID, locationID, keyRingID, keyID,
    )

    resp, err := client.Encrypt(ctx, &kmspb.EncryptRequest{
        Name:      keyName,
        Plaintext: plaintext,
    })
    if err != nil {
        return nil, err
    }

    return resp.Ciphertext, nil
}
```

### Batch Encryption Considerations

```
┌──────────────────────────────────────────────────────────────┐
│         ENCRYPTION LIMITS & BEST PRACTICES                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Limits:                                                       │
│  • Max plaintext per request: 64 KiB                         │
│  • Max ciphertext per request: 64 KiB                        │
│  • API quota: 60,000 requests/min (adjustable)               │
│                                                                │
│  For large data:                                               │
│  → Use ENVELOPE ENCRYPTION (Part 8)                          │
│  → KMS encrypts a DEK (data encryption key)                 │
│  → DEK encrypts the actual data locally                      │
│  → No size limit on data, only 1 KMS call                   │
│                                                                │
│  For many small items:                                        │
│  → Batch encrypt in parallel (use async API)                 │
│  → Cache DEKs to reduce KMS API calls                        │
│  → Consider Tink library for envelope encryption              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 5 — Key Rotation

### Automatic Rotation

```
┌──────────────────────────────────────────────────────────────┐
│         KEY ROTATION                                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Before rotation:                                              │
│  Key "app-data"                                               │
│  ├── Version 1 (PRIMARY) ← used for encrypt & decrypt       │
│                                                                │
│  After rotation:                                               │
│  Key "app-data"                                               │
│  ├── Version 2 (PRIMARY) ← used for NEW encryptions         │
│  └── Version 1 (ENABLED) ← still used for decryption        │
│                                                                │
│  How it works:                                                 │
│  1. New key version is generated                              │
│  2. New version becomes PRIMARY                               │
│  3. New encrypt() calls use the new version                  │
│  4. Old version stays ENABLED for decrypt()                  │
│  5. Ciphertext includes version info → auto-selects key      │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ KEY POINT: You NEVER need to re-encrypt existing data  │  │
│  │ after rotation. The ciphertext stores which version    │  │
│  │ was used, and decrypt() automatically picks the right  │  │
│  │ key version.                                           │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
│  Recommended rotation periods:                                 │
│  • General purpose: 90 days                                   │
│  • Compliance (PCI-DSS): 365 days max                        │
│  • High security: 30 days                                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Configuring Rotation

```bash
# Set automatic rotation on a key (90-day rotation)
gcloud kms keys update app-data-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --rotation-period=90d \
    --next-rotation-time="2026-08-01T00:00:00Z"

# Manually rotate a key (create new primary version immediately)
gcloud kms keys versions create \
    --key=app-data-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --primary

# Check key version list
gcloud kms keys versions list \
    --key=app-data-key \
    --keyring=prod-keys \
    --location=us-central1

# Disable old key version (after verifying no data needs it)
gcloud kms keys versions disable 1 \
    --key=app-data-key \
    --keyring=prod-keys \
    --location=us-central1

# Schedule key version destruction (24h minimum wait)
gcloud kms keys versions destroy 1 \
    --key=app-data-key \
    --keyring=prod-keys \
    --location=us-central1
```

---

## Part 6 — CMEK — Customer-Managed Encryption Keys

### What Is CMEK?

```
┌──────────────────────────────────────────────────────────────┐
│         CMEK — CUSTOMER-MANAGED ENCRYPTION KEYS               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Default Google Encryption:                                    │
│  ┌───────────────┐     ┌───────────────┐                    │
│  │ Your Data     │────►│ GCP Service   │                    │
│  └───────────────┘     │ encrypts with │                    │
│                        │ GOOGLE key    │                    │
│                        └───────────────┘                    │
│  → You have NO control over the key                        │
│                                                                │
│  CMEK:                                                         │
│  ┌───────────────┐     ┌───────────────┐     ┌──────────┐  │
│  │ Your Data     │────►│ GCP Service   │────►│ Cloud KMS│  │
│  └───────────────┘     │ encrypts with │     │ YOUR key │  │
│                        │ YOUR key      │     └──────────┘  │
│                        └───────────────┘                    │
│  → You control the key: rotation, disable, destroy          │
│  → Disable key = data is INACCESSIBLE                      │
│  → Destroy key = data is PERMANENTLY LOST                  │
│                                                                │
│  CMEK-supported services (100+):                              │
│  • Cloud Storage         • BigQuery                          │
│  • Cloud SQL             • Pub/Sub                           │
│  • GKE                   • Cloud Spanner                     │
│  • Compute Engine disks  • Dataflow                          │
│  • Filestore             • Artifact Registry                 │
│  • Cloud Functions       • Cloud Run                         │
│  • Bigtable              • Secret Manager                    │
│  • Dataproc              • and many more...                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Setting Up CMEK

```bash
# Step 1: Create KMS key (if not already done)
gcloud kms keys create storage-cmek-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --purpose=encryption

# Step 2: Grant the GCP service's service agent access to the key
# Each service has a service agent that needs roles/cloudkms.cryptoKeyEncrypterDecrypter

# For Cloud Storage:
SERVICE_AGENT="service-PROJECT_NUMBER@gs-project-accounts.iam.gserviceaccount.com"
gcloud kms keys add-iam-policy-binding storage-cmek-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --member="serviceAccount:${SERVICE_AGENT}" \
    --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Step 3: Create the resource with CMEK
# Cloud Storage bucket with CMEK
gcloud storage buckets create gs://my-cmek-bucket \
    --location=us-central1 \
    --default-encryption-key=projects/my-project/locations/us-central1/keyRings/prod-keys/cryptoKeys/storage-cmek-key
```

### CMEK for Common Services

```bash
# ─── Cloud SQL with CMEK ─────────────────────────────────────
gcloud sql instances create my-db \
    --database-version=POSTGRES_15 \
    --tier=db-custom-2-8192 \
    --region=us-central1 \
    --disk-encryption-key=projects/my-project/locations/us-central1/keyRings/prod-keys/cryptoKeys/sql-cmek-key

# ─── BigQuery dataset with CMEK ──────────────────────────────
bq mk --dataset \
    --default_kms_key=projects/my-project/locations/us-central1/keyRings/prod-keys/cryptoKeys/bq-cmek-key \
    my-project:my_dataset

# ─── Compute Engine disk with CMEK ───────────────────────────
gcloud compute disks create my-disk \
    --zone=us-central1-a \
    --size=100GB \
    --kms-key=projects/my-project/locations/us-central1/keyRings/prod-keys/cryptoKeys/disk-cmek-key

# ─── GKE cluster with CMEK (boot disks + etcd) ──────────────
gcloud container clusters create my-cluster \
    --zone=us-central1-a \
    --boot-disk-kms-key=projects/my-project/locations/us-central1/keyRings/prod-keys/cryptoKeys/gke-cmek-key \
    --database-encryption-key=projects/my-project/locations/us-central1/keyRings/prod-keys/cryptoKeys/gke-cmek-key
```

---

## Part 7 — CSEK — Customer-Supplied Encryption Keys

### What Is CSEK?

```
┌──────────────────────────────────────────────────────────────┐
│         CSEK — CUSTOMER-SUPPLIED ENCRYPTION KEYS              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  With CSEK, YOU provide the raw key material with each        │
│  API request. Google uses it to encrypt/decrypt but           │
│  NEVER stores your key.                                       │
│                                                                │
│  ┌───────────────┐                                           │
│  │ Your App      │                                           │
│  │               │── key + data ──►┌────────────────┐       │
│  │               │                 │ GCP Service    │       │
│  │               │                 │ (encrypts)     │       │
│  │               │◄── encrypted ───│ (discards key) │       │
│  └───────────────┘    data         └────────────────┘       │
│                                                                │
│  Supported services (limited):                                │
│  • Compute Engine persistent disks                           │
│  • Cloud Storage objects                                     │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ WARNING:                                                │  │
│  │ • If you lose the key, your data is GONE forever       │  │
│  │ • Google cannot recover CSEK-encrypted data            │  │
│  │ • You must supply the key with EVERY request           │  │
│  │ • No automatic rotation                                 │  │
│  │ • Must be exactly 256 bits, base64-encoded             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
│  CMEK vs CSEK:                                                │
│  ┌──────────────────┬──────────────────┐                    │
│  │ CMEK             │ CSEK             │                    │
│  ├──────────────────┼──────────────────┤                    │
│  │ Key in Cloud KMS │ Key in your app  │                    │
│  │ Google stores it │ Google discards  │                    │
│  │ Rotation support │ No rotation      │                    │
│  │ 100+ services    │ 2 services       │                    │
│  │ Easier to manage │ Risky to manage  │                    │
│  └──────────────────┴──────────────────┘                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Using CSEK

```bash
# Generate a 256-bit key (base64-encoded)
KEY=$(python3 -c "import os, base64; print(base64.b64encode(os.urandom(32)).decode())")
echo "Your key (SAVE THIS): $KEY"

# Create a Compute Engine disk with CSEK
gcloud compute disks create csek-disk \
    --zone=us-central1-a \
    --size=100GB \
    --csek-key-file=- <<< "[{\"uri\": \"https://compute.googleapis.com/compute/v1/projects/my-project/zones/us-central1-a/disks/csek-disk\", \"key\": \"${KEY}\"}]"

# Upload to Cloud Storage with CSEK
curl -X POST \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    -H "x-goog-encryption-algorithm: AES256" \
    -H "x-goog-encryption-key: ${KEY}" \
    -H "x-goog-encryption-key-sha256: $(echo -n "${KEY}" | base64 -d | sha256sum | cut -d' ' -f1 | xxd -r -p | base64)" \
    --data-binary @myfile.txt \
    "https://storage.googleapis.com/upload/storage/v1/b/my-bucket/o?uploadType=media&name=myfile.txt"
```

---

## Part 8 — Envelope Encryption

### How Envelope Encryption Works

```
┌──────────────────────────────────────────────────────────────┐
│         ENVELOPE ENCRYPTION                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Problem: Cloud KMS limits plaintext to 64 KiB per call.     │
│  Solution: Envelope encryption — two layers of keys.          │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ENCRYPTION FLOW:                                     │    │
│  │                                                      │    │
│  │  1. Generate a random DEK (data encryption key)      │    │
│  │     locally (AES-256)                                │    │
│  │                                                      │    │
│  │  2. Encrypt your data with the DEK locally           │    │
│  │     → data can be ANY size (GB, TB)                  │    │
│  │                                                      │    │
│  │  3. Call Cloud KMS to encrypt the DEK                │    │
│  │     → KMS wraps the DEK with the KEK (key           │    │
│  │       encryption key = your Cloud KMS key)           │    │
│  │                                                      │    │
│  │  4. Store: encrypted_data + wrapped_DEK              │    │
│  │     → Discard the plaintext DEK                      │    │
│  │                                                      │    │
│  │  ┌──────┐ local    ┌──────────────────────┐         │    │
│  │  │ DEK  │────────►│ Encrypt large data   │         │    │
│  │  └──┬───┘         └──────────┬─────────────┘         │    │
│  │     │                        │                        │    │
│  │     │ KMS API               │ Store                  │    │
│  │     ▼                        ▼                        │    │
│  │  ┌──────┐          ┌──────────────────────┐         │    │
│  │  │ KEK  │          │ encrypted_data       │         │    │
│  │  │(KMS) │          │ + wrapped_DEK        │         │    │
│  │  └──────┘          └──────────────────────┘         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  DECRYPTION FLOW:                                     │    │
│  │                                                      │    │
│  │  1. Read wrapped_DEK + encrypted_data                │    │
│  │  2. Call Cloud KMS to decrypt (unwrap) the DEK       │    │
│  │  3. Use plaintext DEK to decrypt data locally        │    │
│  │  4. Discard the plaintext DEK                        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Benefits:                                                     │
│  • No size limit on data                                      │
│  • Only 1 KMS call per encrypt/decrypt (fast + cheap)        │
│  • DEK never leaves your application in plaintext            │
│  • This is how GCP services implement CMEK internally        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Implementation with Google Tink

```python
# pip install tink google-cloud-kms
import tink
from tink import aead
from tink.integration import gcpkms

# One-time setup
aead.register()

# KEK URI (your Cloud KMS key)
kek_uri = "gcp-kms://projects/my-project/locations/us-central1/keyRings/prod-keys/cryptoKeys/app-data-key"

# Create KMS client
gcp_client = gcpkms.GcpKmsClient(kek_uri, "")
aead.register_kms_client(gcp_client)

# Create envelope AEAD
# This automatically handles envelope encryption:
# DEK generation, data encryption, DEK wrapping
keyset_handle = tink.new_keyset_handle(
    aead.aead_key_templates.create_kms_envelope_aead_key_template(
        kek_uri=kek_uri,
        dek_template=aead.aead_key_templates.AES256_GCM,
    )
)
envelope_aead = keyset_handle.primitive(aead.Aead)

# Encrypt (any size data)
plaintext = b"Large data content..." * 100000  # ~2 MB
associated_data = b"order-12345"
ciphertext = envelope_aead.encrypt(plaintext, associated_data)

# Decrypt
decrypted = envelope_aead.decrypt(ciphertext, associated_data)
assert decrypted == plaintext
```

---

## Part 9 — Asymmetric Keys — Signing & Encryption

### Asymmetric Signing

```bash
# Create an asymmetric signing key
gcloud kms keys create signing-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --purpose=asymmetric-signing \
    --default-algorithm=ec-sign-p256-sha256

# Get the public key (for verification)
gcloud kms keys versions get-public-key 1 \
    --key=signing-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --output-file=public-key.pem

# Sign data
echo -n "data to sign" | \
    gcloud kms asymmetric-sign \
        --key=signing-key \
        --keyring=prod-keys \
        --location=us-central1 \
        --version=1 \
        --digest-algorithm=sha256 \
        --input-file=- \
        --signature-file=signature.sig

# Verify signature (using OpenSSL with the public key)
echo -n "data to sign" | \
    openssl dgst -sha256 -verify public-key.pem \
        -signature signature.sig
```

### Asymmetric Encryption

```bash
# Create an asymmetric encryption key
gcloud kms keys create asymmetric-enc-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --purpose=asymmetric-encryption \
    --default-algorithm=rsa-decrypt-oaep-2048-sha256

# Get public key (for encryption — can be shared)
gcloud kms keys versions get-public-key 1 \
    --key=asymmetric-enc-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --output-file=public-key.pem

# Encrypt with public key (can be done ANYWHERE — no KMS call needed)
echo -n "secret message" | \
    openssl pkeyutl -encrypt \
        -pubin -inkey public-key.pem \
        -pkeyopt rsa_padding_mode:oaep \
        -pkeyopt rsa_oaep_md:sha256 \
        -out encrypted.bin

# Decrypt with private key (requires KMS call)
gcloud kms asymmetric-decrypt \
    --key=asymmetric-enc-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --version=1 \
    --ciphertext-file=encrypted.bin \
    --plaintext-file=decrypted.txt
```

### Use Cases for Asymmetric Keys

```
┌──────────────────────────────────────────────────────────────┐
│         ASYMMETRIC KEY USE CASES                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Signing:                                                      │
│  • Sign container images (Binary Authorization)              │
│  • Sign JWTs for service-to-service auth                     │
│  • Sign software artifacts (code signing)                    │
│  • Sign blockchain transactions (secp256k1)                  │
│  • Sign API responses for integrity verification             │
│                                                                │
│  Encryption:                                                   │
│  • Encrypt data from external clients (no GCP access)        │
│  • End-to-end encryption (client encrypts with pubkey)       │
│  • Key exchange protocols                                     │
│  • Encrypt data in transit between systems                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 10 — Key Import & External Key Manager (EKM)

### Key Import

```
┌──────────────────────────────────────────────────────────────┐
│         KEY IMPORT                                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Import your existing keys INTO Cloud KMS:                    │
│                                                                │
│  ┌──────────┐     ┌─────────────┐     ┌──────────────┐     │
│  │ Your HSM │────►│ Import Job  │────►│ Cloud KMS    │     │
│  │ on-prem  │     │ (wrapping)  │     │ Key Version  │     │
│  └──────────┘     └─────────────┘     └──────────────┘     │
│                                                                │
│  How it works:                                                 │
│  1. Create an import job in Cloud KMS                        │
│  2. KMS provides a wrapping key (RSA public key)             │
│  3. Wrap your key material with the wrapping key             │
│  4. Upload the wrapped key material                          │
│  5. KMS unwraps and stores the key                           │
│                                                                │
│  Supported import methods:                                     │
│  • RSA-OAEP-3072-SHA256                                      │
│  • RSA-OAEP-4096-SHA256                                      │
│  • RSA-OAEP-3072-SHA1-AES-256 (two-step wrapping)          │
│  • RSA-OAEP-4096-SHA1-AES-256 (two-step wrapping)          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create an import job
gcloud kms import-jobs create my-import-job \
    --keyring=prod-keys \
    --location=us-central1 \
    --import-method=rsa-oaep-3072-sha256 \
    --protection-level=hsm

# Get wrapping key
gcloud kms import-jobs describe my-import-job \
    --keyring=prod-keys \
    --location=us-central1 \
    --format="value(publicKey.pem)" > wrapping-key.pem

# Import key material (after wrapping locally)
gcloud kms keys versions import \
    --key=imported-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --import-job=my-import-job \
    --algorithm=google-symmetric-encryption \
    --wrapped-key-file=wrapped-key.bin
```

### External Key Manager (EKM)

```
┌──────────────────────────────────────────────────────────────┐
│         EXTERNAL KEY MANAGER (EKM)                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Keys stored OUTSIDE Google Cloud entirely:                   │
│                                                                │
│  ┌────────────────┐          ┌──────────────────┐           │
│  │ GCP Service    │          │ External KMS     │           │
│  │ (BigQuery,     │◄────────►│ (Thales, Fortanix│           │
│  │  GCS, etc.)    │  EKM API │  Equinix, etc.)  │           │
│  └────────────────┘          └──────────────────┘           │
│                                                                │
│  Two modes:                                                    │
│  • EKM over Internet: external KMS via public endpoint       │
│  • EKM over VPC: external KMS via private VPC connection    │
│                                                                │
│  Supported EKM partners:                                       │
│  • Thales CipherTrust Manager                                │
│  • Fortanix SDKMS                                            │
│  • Equinix SmartKey                                          │
│  • Ionic Security                                            │
│                                                                │
│  Benefits:                                                     │
│  • Google NEVER sees your key material                       │
│  • Full data sovereignty                                     │
│  • You can revoke access instantly                           │
│  • Audit trail on your side                                  │
│                                                                │
│  Trade-offs:                                                   │
│  • Higher latency (network call to external KMS)             │
│  • Availability depends on external KMS uptime               │
│  • More complex setup                                         │
│  • Higher cost ($3/key version/month)                        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 11 — Key Access Justifications & Assured Workloads

### Key Access Justifications

```
┌──────────────────────────────────────────────────────────────┐
│         KEY ACCESS JUSTIFICATIONS                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Available with Assured Workloads + EKM:                      │
│                                                                │
│  Every time Google accesses your EKM key, a justification    │
│  reason is provided. Your EKM can approve or deny based      │
│  on the reason.                                               │
│                                                                │
│  Justification reasons:                                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ CUSTOMER_INITIATED_SUPPORT   → customer opened ticket │  │
│  │ GOOGLE_INITIATED_SERVICE     → system maintenance     │  │
│  │ THIRD_PARTY_DATA_REQUEST     → legal/law enforcement  │  │
│  │ CUSTOMER_INITIATED_ACCESS    → normal API call        │  │
│  │ GOOGLE_INITIATED_REVIEW      → security review        │  │
│  │ REASON_NOT_EXPECTED          → unexpected access      │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                                │
│  Your EKM policy example:                                      │
│  • ALLOW: CUSTOMER_INITIATED_ACCESS                          │
│  • ALLOW: CUSTOMER_INITIATED_SUPPORT                         │
│  • DENY: all others                                           │
│  → Google cannot access your data for any other reason       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 12 — IAM & Security

### IAM Roles

| Role | Description | Use For |
|------|-------------|---------|
| `roles/cloudkms.admin` | Full KMS management | KMS administrators |
| `roles/cloudkms.cryptoKeyEncrypterDecrypter` | Encrypt + decrypt | Application service accounts |
| `roles/cloudkms.cryptoKeyEncrypter` | Encrypt only | Write-only encryption |
| `roles/cloudkms.cryptoKeyDecrypter` | Decrypt only | Read-only decryption |
| `roles/cloudkms.signer` | Sign data | Signing applications |
| `roles/cloudkms.signerVerifier` | Sign + verify | Full signing workflow |
| `roles/cloudkms.viewer` | View keys (not use) | Auditors |
| `roles/cloudkms.publicKeyViewer` | View public keys | External verification |

### Separation of Duties

```
┌──────────────────────────────────────────────────────────────┐
│         SEPARATION OF DUTIES                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Best practice: separate key management from key usage.       │
│                                                                │
│  ┌────────────────────────┐  ┌────────────────────────┐     │
│  │ Key Administrator      │  │ Key User               │     │
│  │ (security team)        │  │ (application team)     │     │
│  │                        │  │                        │     │
│  │ cloudkms.admin         │  │ cryptoKeyEncrypter     │     │
│  │ Can: create, rotate,   │  │ Decrypter              │     │
│  │ disable, destroy keys  │  │ Can: encrypt/decrypt   │     │
│  │ Cannot: encrypt/decrypt│  │ Cannot: manage keys    │     │
│  └────────────────────────┘  └────────────────────────┘     │
│                                                                │
│  This ensures:                                                 │
│  • Security team controls key lifecycle                      │
│  • App team can only USE keys, not modify them               │
│  • No single person can both manage and use keys             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Granting Access

```bash
# Grant encrypt/decrypt to application SA
gcloud kms keys add-iam-policy-binding app-data-key \
    --keyring=prod-keys \
    --location=us-central1 \
    --member="serviceAccount:app@my-project.iam.gserviceaccount.com" \
    --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Grant admin to security team (at key ring level)
gcloud kms keyrings add-iam-policy-binding prod-keys \
    --location=us-central1 \
    --member="group:security-team@example.com" \
    --role="roles/cloudkms.admin"

# Grant viewer to auditors
gcloud kms keyrings add-iam-policy-binding prod-keys \
    --location=us-central1 \
    --member="group:auditors@example.com" \
    --role="roles/cloudkms.viewer"
```

---

## Part 13 — Monitoring & Audit Logging

### Audit Logs

```
┌──────────────────────────────────────────────────────────────┐
│         KMS AUDIT LOGGING                                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Cloud KMS generates audit logs for ALL key operations:       │
│                                                                │
│  Admin Activity (always on, free):                            │
│  • CreateKeyRing, CreateCryptoKey                             │
│  • UpdateCryptoKey, DestroyCryptoKeyVersion                  │
│  • SetIamPolicy                                               │
│                                                                │
│  Data Access (must enable, chargeable):                       │
│  • Encrypt, Decrypt                                           │
│  • AsymmetricSign, AsymmetricDecrypt                         │
│  • GetPublicKey                                               │
│                                                                │
│  Log entry includes:                                           │
│  • Who (principal email)                                      │
│  • What (method name)                                         │
│  • When (timestamp)                                           │
│  • Which key (full resource name)                             │
│  • From where (caller IP)                                     │
│  • Result (success/failure)                                   │
│                                                                │
│  ⚠ NOTE: The plaintext/ciphertext is NEVER logged            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Monitoring Alerts

```bash
# Alert on key destruction scheduled
# Cloud Monitoring → Alerting → Create Policy
# Metric: logging.googleapis.com/user/kms-key-destruction
# Condition: any occurrence

# Log-based metric for key destructions
gcloud logging metrics create kms-key-destruction \
    --description="KMS key version destruction scheduled" \
    --log-filter='resource.type="cloudkms_cryptokeyversion" AND protoPayload.methodName="DestroyCryptoKeyVersion"'

# Alert on unexpected encrypt/decrypt from unknown principals
gcloud logging metrics create kms-unexpected-access \
    --description="KMS access from unexpected principal" \
    --log-filter='resource.type="cloudkms_cryptokey" AND protoPayload.methodName=("Encrypt" OR "Decrypt") AND NOT protoPayload.authenticationInfo.principalEmail=~".*@my-project.iam.gserviceaccount.com"'
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Complete KMS Setup

```hcl
# ─── Key Ring ─────────────────────────────────────────────────
resource "google_kms_key_ring" "prod" {
  name     = "prod-keys"
  location = "us-central1"
  project  = var.project_id
}

# ─── Symmetric Encryption Key ────────────────────────────────
resource "google_kms_crypto_key" "app_data" {
  name            = "app-data-key"
  key_ring        = google_kms_key_ring.prod.id
  purpose         = "ENCRYPT_DECRYPT"
  rotation_period = "7776000s"  # 90 days

  version_template {
    algorithm        = "GOOGLE_SYMMETRIC_ENCRYPTION"
    protection_level = "SOFTWARE"
  }

  # Prevent accidental destruction
  lifecycle {
    prevent_destroy = true
  }
}

# ─── HSM-Backed Key ──────────────────────────────────────────
resource "google_kms_crypto_key" "hsm_key" {
  name     = "hsm-data-key"
  key_ring = google_kms_key_ring.prod.id
  purpose  = "ENCRYPT_DECRYPT"

  version_template {
    algorithm        = "GOOGLE_SYMMETRIC_ENCRYPTION"
    protection_level = "HSM"
  }

  lifecycle {
    prevent_destroy = true
  }
}

# ─── Asymmetric Signing Key ──────────────────────────────────
resource "google_kms_crypto_key" "signing" {
  name     = "signing-key"
  key_ring = google_kms_key_ring.prod.id
  purpose  = "ASYMMETRIC_SIGN"

  version_template {
    algorithm        = "EC_SIGN_P256_SHA256"
    protection_level = "SOFTWARE"
  }
}

# ─── IAM — Application encrypt/decrypt ───────────────────────
resource "google_kms_crypto_key_iam_member" "app_encrypter" {
  crypto_key_id = google_kms_crypto_key.app_data.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  member        = "serviceAccount:${google_service_account.app.email}"
}

# ─── IAM — GCS service agent for CMEK ────────────────────────
resource "google_kms_crypto_key_iam_member" "gcs_cmek" {
  crypto_key_id = google_kms_crypto_key.app_data.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  member        = "serviceAccount:service-${data.google_project.current.number}@gs-project-accounts.iam.gserviceaccount.com"
}

# ─── Cloud Storage with CMEK ─────────────────────────────────
resource "google_storage_bucket" "encrypted" {
  name     = "${var.project_id}-encrypted-data"
  location = "us-central1"
  project  = var.project_id

  encryption {
    default_kms_key_name = google_kms_crypto_key.app_data.id
  }

  depends_on = [google_kms_crypto_key_iam_member.gcs_cmek]
}

# ─── BigQuery Dataset with CMEK ──────────────────────────────
resource "google_bigquery_dataset" "encrypted" {
  dataset_id                = "encrypted_dataset"
  project                   = var.project_id
  location                  = "us-central1"
  default_encryption_configuration {
    kms_key_name = google_kms_crypto_key.app_data.id
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# KEY RINGS
# ═══════════════════════════════════════════════════════════════

# Create key ring
gcloud kms keyrings create RING_NAME --location=LOCATION

# List key rings
gcloud kms keyrings list --location=LOCATION

# ═══════════════════════════════════════════════════════════════
# KEYS
# ═══════════════════════════════════════════════════════════════

# Create symmetric key
gcloud kms keys create KEY_NAME \
    --keyring=RING --location=LOC \
    --purpose=encryption

# Create asymmetric signing key
gcloud kms keys create KEY_NAME \
    --keyring=RING --location=LOC \
    --purpose=asymmetric-signing \
    --default-algorithm=ec-sign-p256-sha256

# Create HSM key
gcloud kms keys create KEY_NAME \
    --keyring=RING --location=LOC \
    --purpose=encryption \
    --protection-level=hsm

# List keys
gcloud kms keys list --keyring=RING --location=LOC

# Update rotation
gcloud kms keys update KEY_NAME \
    --keyring=RING --location=LOC \
    --rotation-period=90d \
    --next-rotation-time=TIMESTAMP

# ═══════════════════════════════════════════════════════════════
# KEY VERSIONS
# ═══════════════════════════════════════════════════════════════

# Create new version (manual rotation)
gcloud kms keys versions create --key=KEY --keyring=RING --location=LOC --primary

# List versions
gcloud kms keys versions list --key=KEY --keyring=RING --location=LOC

# Disable version
gcloud kms keys versions disable VERSION --key=KEY --keyring=RING --location=LOC

# Enable version
gcloud kms keys versions enable VERSION --key=KEY --keyring=RING --location=LOC

# Schedule destruction
gcloud kms keys versions destroy VERSION --key=KEY --keyring=RING --location=LOC

# Restore scheduled destruction
gcloud kms keys versions restore VERSION --key=KEY --keyring=RING --location=LOC

# ═══════════════════════════════════════════════════════════════
# ENCRYPT / DECRYPT
# ═══════════════════════════════════════════════════════════════

# Encrypt
gcloud kms encrypt --key=KEY --keyring=RING --location=LOC \
    --plaintext-file=INPUT --ciphertext-file=OUTPUT

# Decrypt
gcloud kms decrypt --key=KEY --keyring=RING --location=LOC \
    --ciphertext-file=INPUT --plaintext-file=OUTPUT

# ═══════════════════════════════════════════════════════════════
# ASYMMETRIC OPERATIONS
# ═══════════════════════════════════════════════════════════════

# Get public key
gcloud kms keys versions get-public-key VERSION \
    --key=KEY --keyring=RING --location=LOC --output-file=pub.pem

# Sign
gcloud kms asymmetric-sign --key=KEY --keyring=RING --location=LOC \
    --version=VERSION --digest-algorithm=sha256 \
    --input-file=INPUT --signature-file=OUTPUT

# Asymmetric decrypt
gcloud kms asymmetric-decrypt --key=KEY --keyring=RING --location=LOC \
    --version=VERSION --ciphertext-file=INPUT --plaintext-file=OUTPUT

# ═══════════════════════════════════════════════════════════════
# IAM
# ═══════════════════════════════════════════════════════════════

# Grant role on key
gcloud kms keys add-iam-policy-binding KEY \
    --keyring=RING --location=LOC \
    --member=MEMBER --role=ROLE

# View IAM policy
gcloud kms keys get-iam-policy KEY --keyring=RING --location=LOC
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Multi-Tier Encryption for Healthcare Data

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: HIPAA-COMPLIANT MULTI-TIER ENCRYPTION                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Key Ring: "healthcare-prod" (us-central1)               │        │
│  │                                                           │        │
│  │  ├── phi-data-key (HSM, 30-day rotation)                 │        │
│  │  │   → Encrypts Cloud SQL with patient data              │        │
│  │  │   → CMEK for Cloud Storage (medical records)          │        │
│  │  │                                                        │        │
│  │  ├── audit-key (HSM, 90-day rotation)                    │        │
│  │  │   → CMEK for audit log sink bucket                    │        │
│  │  │   → Cannot be disabled without CISO approval          │        │
│  │  │                                                        │        │
│  │  ├── app-signing-key (HSM, no rotation)                  │        │
│  │  │   → Signs API responses for integrity                 │        │
│  │  │   → Public key shared with partners                   │        │
│  │  │                                                        │        │
│  │  └── backup-key (HSM, 365-day rotation)                  │        │
│  │      → Encrypts backup exports                           │        │
│  │      → Key material also imported to on-prem HSM         │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  IAM setup:                                                            │
│  • Security team: cloudkms.admin (key ring level)                    │
│  • App SA: cryptoKeyEncrypterDecrypter (phi-data-key only)           │
│  • Audit SA: cryptoKeyEncrypter (audit-key only — write only)        │
│  • Backup SA: cryptoKeyEncrypterDecrypter (backup-key only)          │
│  • NO human has decrypt access to phi-data-key                       │
│                                                                        │
│  Monitoring:                                                           │
│  • Alert on key destruction attempts                                  │
│  • Alert on IAM policy changes to any key                            │
│  • Alert on decrypt from unexpected IP ranges                        │
│  • Monthly rotation compliance report                                 │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: CMEK Across All Services (Enterprise Policy)

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: ORG-WIDE CMEK ENFORCEMENT                             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Organization Policy:                                                  │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  constraints/gcp.restrictNonCmekServices                  │        │
│  │  → Deny creation of resources WITHOUT CMEK               │        │
│  │  → Applied at org level                                   │        │
│  │                                                           │        │
│  │  constraints/gcp.restrictCmekCryptoKeyProjects            │        │
│  │  → Only allow CMEK keys from "security-keys-project"     │        │
│  │  → Prevents teams from using their own keys              │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Centralized key project:                                              │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Project: security-keys-prod                              │        │
│  │                                                           │        │
│  │  Key Ring: us-central1-keys                               │        │
│  │  ├── storage-cmek          (for all GCS buckets)         │        │
│  │  ├── compute-cmek          (for all disks)               │        │
│  │  ├── sql-cmek              (for all Cloud SQL)           │        │
│  │  ├── bigquery-cmek         (for all BigQuery)            │        │
│  │  └── gke-cmek              (for all GKE clusters)       │        │
│  │                                                           │        │
│  │  Key Ring: europe-west1-keys                              │        │
│  │  ├── storage-cmek-eu       (EU data residency)           │        │
│  │  └── sql-cmek-eu           (EU data residency)           │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Per-project setup:                                                    │
│  1. Security team creates key in central project                      │
│  2. Grants service agent of target project encrypt/decrypt            │
│  3. App team references key when creating resources                   │
│  4. Org policy enforces — no CMEK = resource creation blocked         │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Crypto Shredding (Secure Data Deletion)

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: CRYPTO SHREDDING                                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Use case: GDPR "right to be forgotten" — delete user data.          │
│                                                                        │
│  Traditional deletion:                                                 │
│  • Delete records from database                                       │
│  • Delete files from storage                                          │
│  • Hope backup systems also delete                                     │
│  • Hope replicas also delete                                          │
│  • Hard to prove ALL copies are gone                                  │
│                                                                        │
│  Crypto shredding:                                                     │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  1. Each customer/tenant gets their own KMS key           │        │
│  │     Key: "customer-{customer_id}-key"                     │        │
│  │                                                           │        │
│  │  2. All customer data encrypted with their key            │        │
│  │     via CMEK or envelope encryption                       │        │
│  │                                                           │        │
│  │  3. To "delete" a customer:                               │        │
│  │     → Destroy ALL versions of their KMS key               │        │
│  │     → Data still exists but is UNREADABLE                 │        │
│  │     → Applies to all copies, backups, replicas            │        │
│  │     → Provably destroyed (KMS audit log)                 │        │
│  │                                                           │        │
│  │  4. Clean up encrypted data at leisure                    │        │
│  │     → No urgency — it's cryptographically useless        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Warning: Don't share keys across customers!                          │
│  Destroying a shared key would affect ALL customers.                  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command / Location |
|--------|-------------------|
| Create key ring | `gcloud kms keyrings create RING --location=LOC` |
| Create symmetric key | `gcloud kms keys create KEY --keyring=RING --location=LOC --purpose=encryption` |
| Create HSM key | Add `--protection-level=hsm` |
| Encrypt file | `gcloud kms encrypt --key=KEY ... --plaintext-file=IN --ciphertext-file=OUT` |
| Decrypt file | `gcloud kms decrypt --key=KEY ... --ciphertext-file=IN --plaintext-file=OUT` |
| Rotate key | `gcloud kms keys versions create --key=KEY ... --primary` |
| Set auto-rotation | `gcloud kms keys update KEY ... --rotation-period=90d` |
| Disable version | `gcloud kms keys versions disable VER --key=KEY ...` |
| Destroy version | `gcloud kms keys versions destroy VER --key=KEY ...` |
| Grant encrypt/decrypt | `--role="roles/cloudkms.cryptoKeyEncrypterDecrypter"` |
| CMEK for GCS | `--default-encryption-key=KEY_RESOURCE_NAME` |
| Max plaintext size | 64 KiB per KMS call (use envelope for larger) |
| Protection levels | SOFTWARE, HSM, EXTERNAL, EXTERNAL_VPC |

---

## What Is Encryption? (Beginner Explanation)

### Simple Analogy

```
┌──────────────────────────────────────────────────────────────┐
│         ENCRYPTION — THE LOCKED SAFE ANALOGY                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Think of encryption as putting your data in a locked safe.  │
│  Cloud KMS manages the KEYS to that safe.                    │
│                                                                │
│  Without encryption:                                           │
│  ┌──────────────┐                                            │
│  │ "My password  │  ← Anyone who finds this can read it     │
│  │  is abc123"   │                                           │
│  └──────────────┘                                            │
│                                                                │
│  With encryption:                                              │
│  ┌──────────────┐     🔑 Key      ┌────────────────────┐    │
│  │ "My password  │ ──────────────► │ "xK9#mQ2$vL7!nR"  │    │
│  │  is abc123"   │   Encrypt       │ (unreadable)       │    │
│  └──────────────┘                  └────────────────────┘    │
│                                                                │
│  To read it again:                                             │
│  ┌────────────────────┐     🔑 Same Key   ┌──────────────┐  │
│  │ "xK9#mQ2$vL7!nR"  │ ────────────────► │ "My password  │  │
│  │ (unreadable)       │    Decrypt        │  is abc123"   │  │
│  └────────────────────┘                   └──────────────┘  │
│                                                                │
│  Key takeaway:                                                 │
│  • The DATA without the KEY is useless gibberish             │
│  • The KEY without the DATA is useless too                   │
│  • You need BOTH to read the original data                   │
│  • Cloud KMS = the secure key management service             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Encryption at Rest vs In Transit

```
┌──────────────────────────────────────────────────────────────┐
│         TWO TYPES OF ENCRYPTION                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ENCRYPTION AT REST (data sitting in storage):                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                                                       │    │
│  │  Your data stored on disk, in a database, or in      │    │
│  │  Cloud Storage → it's encrypted so that even if      │    │
│  │  someone steals the physical disk, they can't        │    │
│  │  read your data.                                      │    │
│  │                                                       │    │
│  │  Example: A Cloud Storage object sitting in a bucket │    │
│  │  is encrypted. The disk it sits on has scrambled     │    │
│  │  bytes, not your actual file.                         │    │
│  │                                                       │    │
│  │  → This is what Cloud KMS manages                    │    │
│  │  → Google encrypts ALL data at rest by default       │    │
│  │  → Cloud KMS lets you control the keys               │    │
│  │                                                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ENCRYPTION IN TRANSIT (data moving over the network):       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                                                       │    │
│  │  Your data traveling from your browser to Google's   │    │
│  │  servers → it's encrypted so that anyone              │    │
│  │  eavesdropping on the network can't read it.         │    │
│  │                                                       │    │
│  │  Example: When you upload a file to Cloud Storage,   │    │
│  │  it travels over HTTPS (TLS) — encrypted in transit. │    │
│  │                                                       │    │
│  │  → Handled by TLS/HTTPS (not Cloud KMS)              │    │
│  │  → Google encrypts all traffic between its data      │    │
│  │    centers automatically                              │    │
│  │                                                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Google Default vs CMEK vs CSEK — When to Use Which

```
┌──────────────────────────────────────────────────────────────────────┐
│     CHOOSING THE RIGHT ENCRYPTION OPTION                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Google Default Encryption — USE WHEN:                          │  │
│  │                                                                │  │
│  │ ✓ You're getting started and don't have compliance needs      │  │
│  │ ✓ You trust Google to manage keys properly                    │  │
│  │ ✓ You don't need to control key rotation or access            │  │
│  │ ✓ You want zero configuration                                 │  │
│  │                                                                │  │
│  │ → Best for: personal projects, dev/test, non-sensitive data   │  │
│  │ → Cost: FREE                                                  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ CMEK — USE WHEN:                                               │  │
│  │                                                                │  │
│  │ ✓ Compliance requires you to control encryption keys          │  │
│  │   (HIPAA, PCI-DSS, SOC 2, GDPR)                              │  │
│  │ ✓ You need to disable a key to make data inaccessible         │  │
│  │ ✓ You need crypto shredding (destroy key = destroy data)      │  │
│  │ ✓ You need custom rotation schedules                          │  │
│  │ ✓ You need audit logs showing who used which key              │  │
│  │                                                                │  │
│  │ → Best for: production workloads, regulated industries        │  │
│  │ → Cost: $0.06/key version/month + API call costs              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ CSEK — USE WHEN:                                               │  │
│  │                                                                │  │
│  │ ✓ You want Google to NEVER store your key material            │  │
│  │ ✓ You have your own key management infrastructure             │  │
│  │ ✓ You only need encryption for Compute Engine or GCS          │  │
│  │ ✓ You accept the risk of losing the key = losing data         │  │
│  │                                                                │  │
│  │ → Best for: very specific high-security use cases             │  │
│  │ → Cost: FREE (you manage the keys yourself)                   │  │
│  │ → ⚠ WARNING: Most teams should use CMEK instead              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  Simple decision tree:                                                 │
│  1. Do you need to control your encryption keys?                      │
│     → No  → Use Google Default Encryption                            │
│     → Yes → Continue to #2                                           │
│  2. Do you want Google to store/manage the key material?              │
│     → Yes → Use CMEK (recommended)                                   │
│     → No  → Continue to #3                                           │
│  3. Do you want the key OUTSIDE of Google entirely?                   │
│     → No  → Use CSEK (you supply key each time)                     │
│     → Yes → Use EKM (External Key Manager)                          │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Creating & Managing Keys

### Step 1 — Create a Key Ring

```
Console → Navigation Menu → Security → Key Management

┌──────────────────────────────────────────────────────────────┐
│ STEP 1: CREATE A KEY RING                                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Open Google Cloud Console → go to Security section        │
│  2. Click "Key Management" (or search "KMS" in search bar)   │
│  3. Click "+ CREATE KEY RING"                                 │
│                                                                │
│  Fill in the form:                                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Key ring name:  [prod-keys]                           │    │
│  │                                                       │    │
│  │   → Must be unique within the location                │    │
│  │   → Use a descriptive name (e.g., "prod-keys",       │    │
│  │     "healthcare-keys", "team-backend-keys")           │    │
│  │   → ⚠ CANNOT be renamed or deleted later!            │    │
│  │                                                       │    │
│  │ Location type:                                        │    │
│  │   ○ Region      (e.g., us-central1)                  │    │
│  │   ○ Multi-region (e.g., us, europe)                  │    │
│  │   ○ Global      (available everywhere)               │    │
│  │                                                       │    │
│  │   → ⚠ CANNOT be changed later!                       │    │
│  │   → For CMEK: must match the resource's location     │    │
│  │   → For general use: "global" is convenient           │    │
│  │                                                       │    │
│  │ Location:      [us-central1]                          │    │
│  │                                                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  4. Click "CREATE"                                            │
│                                                                │
│  The console will now open the key ring and prompt you to    │
│  create a key inside it.                                      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 2 — Create a Key

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 2: CREATE A KEY                                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  After creating the key ring (or inside an existing ring):    │
│  Click "+ CREATE KEY"                                         │
│                                                                │
│  Fill in the form:                                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                                                       │    │
│  │ Key name:  [app-data-key]                             │    │
│  │   → Descriptive name for what this key protects       │    │
│  │                                                       │    │
│  │ Protection level:                                     │    │
│  │   ○ Software    (cheapest — $0.06/version/month)     │    │
│  │   ○ HSM         (hardware-backed — $1-2.50/mo)       │    │
│  │   ○ External    (key stored outside Google)           │    │
│  │                                                       │    │
│  │ Purpose:                                              │    │
│  │   ○ Symmetric encrypt/decrypt  (most common)         │    │
│  │   ○ Asymmetric sign                                   │    │
│  │   ○ Asymmetric decrypt                                │    │
│  │   ○ MAC signing                                       │    │
│  │   → ⚠ Purpose CANNOT be changed after creation!      │    │
│  │                                                       │    │
│  │ [If Symmetric encrypt/decrypt selected:]              │    │
│  │ Rotation period:                                      │    │
│  │   ○ 90 days     (recommended)                        │    │
│  │   ○ 180 days                                          │    │
│  │   ○ 365 days                                          │    │
│  │   ○ Custom                                            │    │
│  │   ○ Never (manual rotation only)                     │    │
│  │                                                       │    │
│  │ Starting on:  [date picker for first rotation]        │    │
│  │                                                       │    │
│  │ [If Asymmetric selected:]                             │    │
│  │ Algorithm:                                            │    │
│  │   • EC_SIGN_P256_SHA256  (recommended for signing)   │    │
│  │   • RSA_SIGN_PSS_2048_SHA256                         │    │
│  │   • RSA_DECRYPT_OAEP_2048_SHA256                     │    │
│  │   • etc.                                              │    │
│  │                                                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  5. Click "CREATE"                                            │
│                                                                │
│  The key is created with Version 1 as the PRIMARY version.   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 3 — Rotate a Key

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 3: ROTATE A KEY (MANUAL)                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Go to Key Management → select your key ring              │
│  2. Click on the key name (e.g., "app-data-key")             │
│  3. Click "ROTATE KEY" button at the top                     │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Rotate key?                                           │    │
│  │                                                       │    │
│  │ A new key version will be generated and set as the   │    │
│  │ PRIMARY version. New encryption requests will use    │    │
│  │ this version. Previous versions remain available     │    │
│  │ for decryption.                                       │    │
│  │                                                       │    │
│  │           [CANCEL]    [ROTATE]                        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  4. Click "ROTATE"                                            │
│                                                                │
│  What happens:                                                 │
│  • New version (e.g., Version 2) is created                  │
│  • New version becomes the PRIMARY version                   │
│  • Version 1 stays ENABLED (still usable for decryption)     │
│  • All NEW encrypt() calls use Version 2                     │
│  • Existing data encrypted with Version 1 can still be       │
│    decrypted (KMS auto-selects the correct version)          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 4 — Disable / Enable a Key Version

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 4: DISABLE / ENABLE A KEY VERSION                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  To DISABLE a key version:                                     │
│  1. Go to Key Management → key ring → click on the key      │
│  2. You'll see a list of key versions (Version 1, 2, etc.)  │
│  3. Click the ⋮ (three dots) menu next to the version        │
│  4. Click "Disable"                                           │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Version  │ State    │ Primary │ Actions              │    │
│  │──────────│──────────│─────────│──────────────────────│    │
│  │ 2        │ ENABLED  │ ★       │ ⋮ Disable / Destroy │    │
│  │ 1        │ DISABLED │         │ ⋮ Enable / Destroy  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ⚠ What happens when you disable a version:                  │
│  • Data encrypted with this version CANNOT be decrypted      │
│  • The key material still exists (can re-enable later)       │
│  • This is REVERSIBLE — just click "Enable"                  │
│                                                                │
│  To ENABLE a disabled version:                                │
│  1. Click the ⋮ menu next to the disabled version            │
│  2. Click "Enable"                                            │
│  3. The version goes back to ENABLED state                   │
│  4. Decryption works again immediately                       │
│                                                                │
│  Use case: Quickly make data inaccessible during a           │
│  security incident, then re-enable after investigation.      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 5 — Schedule Destruction of a Key Version

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 5: SCHEDULE DESTRUCTION                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Go to Key Management → key ring → key → versions        │
│  2. Click the ⋮ menu next to the version                     │
│  3. Click "Schedule destruction"                              │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Schedule destruction of key version?                  │    │
│  │                                                       │    │
│  │ The key version will be scheduled for destruction    │    │
│  │ and will be DESTROYED after the waiting period.      │    │
│  │                                                       │    │
│  │ Scheduled destruction duration: 24 hours             │    │
│  │ (configurable up to 120 days per key setting)        │    │
│  │                                                       │    │
│  │ ⚠ After the waiting period, the key material is     │    │
│  │   PERMANENTLY deleted. Data encrypted with this      │    │
│  │   version will be UNRECOVERABLE.                     │    │
│  │                                                       │    │
│  │ You can RESTORE the version before the waiting       │    │
│  │ period expires.                                       │    │
│  │                                                       │    │
│  │           [CANCEL]    [SCHEDULE DESTRUCTION]          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  4. Click "SCHEDULE DESTRUCTION"                              │
│                                                                │
│  The version state changes to DESTROY_SCHEDULED with a       │
│  timestamp showing when destruction will occur.               │
│                                                                │
│  ⚠ CRITICAL: Once the waiting period expires and the state   │
│  changes to DESTROYED, the key material is gone FOREVER.     │
│  Google cannot recover it. Your encrypted data is lost.      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 6 — Restore a Key Version from DESTROY_SCHEDULED

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 6: RESTORE A KEY VERSION                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  You can ONLY restore a version in DESTROY_SCHEDULED state.  │
│  Once it's DESTROYED, it's too late.                         │
│                                                                │
│  In the Console:                                               │
│  1. Go to Key Management → key ring → key → versions        │
│  2. Find the version with state "Scheduled for destruction"  │
│  3. Click the ⋮ menu next to it                              │
│  4. Click "Restore"                                           │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Version  │ State               │ Actions             │    │
│  │──────────│─────────────────────│─────────────────────│    │
│  │ 2        │ ENABLED (PRIMARY)   │ ⋮ Disable / Destroy│    │
│  │ 1        │ DESTROY_SCHEDULED   │ ⋮ Restore ✓        │    │
│  │          │ (destroys in 23h)   │                     │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  5. Confirm the restore                                       │
│  6. The version goes back to DISABLED state                  │
│  7. You can then "Enable" it if you need to use it again     │
│                                                                │
│  With gcloud:                                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ gcloud kms keys versions restore 1 \                  │    │
│  │     --key=app-data-key \                              │    │
│  │     --keyring=prod-keys \                             │    │
│  │     --location=us-central1                            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  💡 Tip: Set up a Cloud Monitoring alert for key              │
│  destruction events so you catch accidental destructions     │
│  within the waiting period.                                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Using CMEK

### How to Enable CMEK on Cloud Storage Bucket

```
┌──────────────────────────────────────────────────────────────────────┐
│ CMEK ON CLOUD STORAGE                                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Prerequisites:                                                        │
│  • A KMS key in the SAME location as the bucket                      │
│  • Cloud Storage service agent must have encrypt/decrypt role        │
│                                                                        │
│  Step 1: Grant the Cloud Storage service agent access to the key     │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Console → Key Management → click on your key                   │  │
│  │ → "PERMISSIONS" tab → "ADD PRINCIPAL"                          │  │
│  │                                                                │  │
│  │ New principal:                                                 │  │
│  │   service-PROJECT_NUMBER@gs-project-accounts.iam.               │  │
│  │   gserviceaccount.com                                          │  │
│  │                                                                │  │
│  │ Role: Cloud KMS CryptoKey Encrypter/Decrypter                 │  │
│  │                                                                │  │
│  │ → Click SAVE                                                  │  │
│  │                                                                │  │
│  │ 💡 Find your PROJECT_NUMBER at:                               │  │
│  │    Console → IAM & Admin → Settings                           │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  Step 2: Create a bucket with CMEK                                    │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Console → Cloud Storage → "CREATE BUCKET"                      │  │
│  │                                                                │  │
│  │ 1. Name your bucket: [my-cmek-bucket]                         │  │
│  │ 2. Choose location: must match the KMS key location           │  │
│  │ 3. Continue through storage class and access control          │  │
│  │ 4. On "Data protection" step:                                 │  │
│  │    → Under "Data encryption"                                  │  │
│  │    → Select "Customer-managed encryption key (CMEK)"          │  │
│  │    → Choose your key ring and key from the dropdown           │  │
│  │ 5. Click "CREATE"                                             │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  For an EXISTING bucket:                                               │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Console → Cloud Storage → click bucket name                    │  │
│  │ → "CONFIGURATION" tab → click ✏️ edit next to Encryption      │  │
│  │ → Select "Customer-managed key" → choose your key             │  │
│  │ → Click SAVE                                                  │  │
│  │                                                                │  │
│  │ ⚠ This only affects NEW objects. Existing objects keep       │  │
│  │   their current encryption. Re-upload to re-encrypt.          │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### How to Enable CMEK on Cloud SQL

```
┌──────────────────────────────────────────────────────────────────────┐
│ CMEK ON CLOUD SQL                                                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ⚠ IMPORTANT: CMEK must be set at instance CREATION time.            │
│  You CANNOT add CMEK to an existing Cloud SQL instance.              │
│                                                                        │
│  Step 1: Grant the Cloud SQL service agent access to the key         │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Console → Key Management → click on your key                   │  │
│  │ → "PERMISSIONS" tab → "ADD PRINCIPAL"                          │  │
│  │                                                                │  │
│  │ New principal:                                                 │  │
│  │   service-PROJECT_NUMBER@gcp-sa-cloud-sql.iam.                  │  │
│  │   gserviceaccount.com                                          │  │
│  │                                                                │  │
│  │ Role: Cloud KMS CryptoKey Encrypter/Decrypter                 │  │
│  │                                                                │  │
│  │ → Click SAVE                                                  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  Step 2: Create a Cloud SQL instance with CMEK                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Console → SQL → "CREATE INSTANCE"                              │  │
│  │                                                                │  │
│  │ 1. Choose database engine (PostgreSQL, MySQL, SQL Server)     │  │
│  │ 2. Fill in Instance ID, password, region, etc.                │  │
│  │ 3. Expand "SHOW CONFIGURATION OPTIONS"                        │  │
│  │ 4. Under "Data Protection" section:                           │  │
│  │    → Find "Customer-managed encryption key (CMEK)"            │  │
│  │    → Toggle it ON                                             │  │
│  │    → Select your key ring and key from the dropdown           │  │
│  │    → The key must be in the SAME region as the instance       │  │
│  │ 5. Click "CREATE INSTANCE"                                    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  After creation:                                                       │
│  • The instance details page shows "Customer-managed" under          │
│    Encryption                                                         │
│  • All data, backups, and replicas use your CMEK key                 │
│  • If you disable/destroy the key → the instance becomes             │
│    INACCESSIBLE                                                       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### How to Enable CMEK on BigQuery

```
┌──────────────────────────────────────────────────────────────────────┐
│ CMEK ON BIGQUERY                                                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  BigQuery supports CMEK at the DATASET level (default key) or        │
│  at the individual TABLE level.                                       │
│                                                                        │
│  Step 1: Grant the BigQuery service agent access to the key          │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Console → Key Management → click on your key                   │  │
│  │ → "PERMISSIONS" tab → "ADD PRINCIPAL"                          │  │
│  │                                                                │  │
│  │ New principal:                                                 │  │
│  │   bq-PROJECT_NUMBER@bigquery-encryption.iam.                    │  │
│  │   gserviceaccount.com                                          │  │
│  │                                                                │  │
│  │ Role: Cloud KMS CryptoKey Encrypter/Decrypter                 │  │
│  │                                                                │  │
│  │ → Click SAVE                                                  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  Step 2: Create a dataset with CMEK                                   │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Console → BigQuery → click your project in Explorer panel      │  │
│  │ → Click ⋮ (three dots) → "Create dataset"                     │  │
│  │                                                                │  │
│  │ 1. Dataset ID: [my_encrypted_dataset]                         │  │
│  │ 2. Data location: must match the KMS key location             │  │
│  │ 3. Under "Advanced options":                                  │  │
│  │    → Encryption: select "Customer-managed encryption key"     │  │
│  │    → Choose your key from the dropdown                        │  │
│  │ 4. Click "CREATE DATASET"                                     │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  For an EXISTING dataset:                                              │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ You can update the default CMEK key for an existing dataset:   │  │
│  │                                                                │  │
│  │ bq update --default_kms_key \                                 │  │
│  │   projects/PROJECT/locations/LOC/keyRings/RING/               │  │
│  │   cryptoKeys/KEY \                                            │  │
│  │   PROJECT:DATASET                                             │  │
│  │                                                                │  │
│  │ ⚠ Existing tables keep their current encryption.             │  │
│  │   New tables will use the updated CMEK key.                   │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Deleting KMS Resources

### Important: Key Rings and Keys CANNOT Be Deleted

```
┌──────────────────────────────────────────────────────────────────────┐
│ ⚠ CRITICAL: KMS DELETION CONSTRAINTS                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                                                                │  │
│  │  Key Rings   → CANNOT be deleted. Ever. Period.               │  │
│  │  Keys        → CANNOT be deleted. Ever. Period.               │  │
│  │  Key Versions → CAN be destroyed (with a waiting period).     │  │
│  │                                                                │  │
│  │  Why? Because deleting a key ring or key name would allow     │  │
│  │  someone to create a NEW key with the same name, which        │  │
│  │  could be a security risk (key reuse / confusion attacks).    │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  What you CAN do:                                                      │
│  • Destroy ALL key versions within a key                              │
│    → The key still exists but has no usable key material              │
│    → The key name is still "taken"                                    │
│  • Disable key versions (reversible)                                  │
│  • Remove IAM permissions from the key (no one can use it)            │
│                                                                        │
│  Best practice to "clean up" unused keys:                              │
│  1. Verify no data depends on the key versions                        │
│  2. Disable all key versions first (wait and verify)                  │
│  3. Schedule destruction of all key versions                          │
│  4. Remove all IAM bindings from the key                              │
│  5. The key ring and key remain as empty shells                       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### How to Destroy Key Versions

```
┌──────────────────────────────────────────────────────────────────────┐
│ DESTROYING KEY VERSIONS                                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  In the Console:                                                       │
│  1. Go to Security → Key Management                                  │
│  2. Click on the key ring → click on the key                         │
│  3. Click the ⋮ menu next to the key version                        │
│  4. Click "Schedule destruction"                                      │
│  5. Confirm the action                                                │
│                                                                        │
│  The version enters DESTROY_SCHEDULED state with a waiting period:   │
│  • Default: 24 hours (minimum)                                       │
│  • Configurable up to 120 days when creating the key                 │
│  • During this period, you can RESTORE the version                   │
│  • After the period expires → DESTROYED (irreversible)               │
│                                                                        │
│  With gcloud CLI:                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ # Schedule destruction of key version 1                        │  │
│  │ gcloud kms keys versions destroy 1 \                           │  │
│  │     --key=app-data-key \                                       │  │
│  │     --keyring=prod-keys \                                      │  │
│  │     --location=us-central1                                     │  │
│  │                                                                │  │
│  │ # Check the state (should show DESTROY_SCHEDULED)              │  │
│  │ gcloud kms keys versions describe 1 \                          │  │
│  │     --key=app-data-key \                                       │  │
│  │     --keyring=prod-keys \                                      │  │
│  │     --location=us-central1 \                                   │  │
│  │     --format="value(state)"                                    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  To RESTORE before the waiting period expires:                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ # Restore key version 1 from DESTROY_SCHEDULED                 │  │
│  │ gcloud kms keys versions restore 1 \                           │  │
│  │     --key=app-data-key \                                       │  │
│  │     --keyring=prod-keys \                                      │  │
│  │     --location=us-central1                                     │  │
│  │                                                                │  │
│  │ # The version goes back to DISABLED state.                     │  │
│  │ # Re-enable it if needed:                                      │  │
│  │ gcloud kms keys versions enable 1 \                            │  │
│  │     --key=app-data-key \                                       │  │
│  │     --keyring=prod-keys \                                      │  │
│  │     --location=us-central1                                     │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ⚠ REMEMBER:                                                         │
│  • DESTROYED is PERMANENT — Google cannot recover the key material   │
│  • Any data encrypted with a destroyed version is UNRECOVERABLE      │
│  • Always disable first, wait, verify, THEN destroy                  │
│  • Set up monitoring alerts for destruction events                    │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 41: Secret Manager** → `41-secret-manager.md`
