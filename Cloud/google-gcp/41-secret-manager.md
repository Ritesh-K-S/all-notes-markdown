# Chapter 41 — Secret Manager

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Secret Manager Fundamentals](#part-1--secret-manager-fundamentals)
- [Part 2: Secrets & Secret Versions](#part-2--secrets--secret-versions)
- [Part 3: Creating & Accessing Secrets](#part-3--creating--accessing-secrets)
- [Part 4: Replication Policies](#part-4--replication-policies)
- [Part 5: Secret Rotation](#part-5--secret-rotation)
- [Part 6: Expiration & TTL](#part-6--expiration--ttl)
- [Part 7: Labels, Annotations & Aliases](#part-7--labels-annotations--aliases)
- [Part 8: IAM & Access Control](#part-8--iam--access-control)
- [Part 9: CMEK Encryption](#part-9--cmek-encryption)
- [Part 10: Pub/Sub Notifications](#part-10--pubsub-notifications)
- [Part 11: Integration with GCP Services](#part-11--integration-with-gcp-services)
- [Part 12: Regional vs Global Secrets](#part-12--regional-vs-global-secrets)
- [Part 13: Monitoring & Audit Logging](#part-13--monitoring--audit-logging)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [Why Not Just Use Environment Variables? (Beginner Explanation)](#why-not-just-use-environment-variables-beginner-explanation)
- [Console Walkthrough: Creating & Managing Secrets](#console-walkthrough-creating--managing-secrets)
- [Console Walkthrough: Deleting Secrets](#console-walkthrough-deleting-secrets)
- [Secret Rotation with Cloud Functions](#secret-rotation-with-cloud-functions)
- [What's Next?](#whats-next)

---

## Overview

Secret Manager is Google Cloud's fully managed service for storing, managing, and accessing sensitive data such as API keys, passwords, certificates, and other secrets. It provides versioning, automatic replication, IAM-based access control, CMEK encryption, audit logging, and rotation support. Unlike Cloud KMS (which manages encryption keys), Secret Manager stores the actual secret values.

---

## Part 1 — Secret Manager Fundamentals

### What Is Secret Manager?

```
┌─────────────────────────────────────────────────────────────────┐
│                  SECRET MANAGER OVERVIEW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Secret Manager stores sensitive configuration data:             │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Secret: "db-password"                                    │   │
│  │  ├── Version 1: "oldpassword123"     (DISABLED)          │   │
│  │  ├── Version 2: "betterpass456"      (ENABLED)           │   │
│  │  └── Version 3: "str0ng!P@ss789"     (ENABLED, latest)   │   │
│  │                                                           │   │
│  │  Secret: "api-key-stripe"                                 │   │
│  │  └── Version 1: "sk_live_abc123..."  (ENABLED, latest)   │   │
│  │                                                           │   │
│  │  Secret: "tls-certificate"                                │   │
│  │  ├── Version 1: "-----BEGIN CERT..." (DESTROYED)         │   │
│  │  └── Version 2: "-----BEGIN CERT..." (ENABLED, latest)   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  What to store:                                                   │
│  ✓ Database passwords          ✓ API keys                       │
│  ✓ TLS certificates/keys       ✓ OAuth client secrets           │
│  ✓ SSH keys                    ✓ Encryption keys (for apps)     │
│  ✓ Service account keys        ✓ Connection strings              │
│                                                                   │
│  What NOT to store:                                               │
│  ✗ Large files (max 64 KiB per version)                         │
│  ✗ Encryption keys for GCP services (use Cloud KMS instead)     │
│  ✗ Frequently changing data (use a database)                    │
│  ✗ Environment configuration (use Runtime Configurator)         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Secret Manager | AWS Secrets Manager | Azure Key Vault (Secrets) |
|---------|-------------------|---------------------|--------------------------|
| Service | Secret Manager | Secrets Manager | Key Vault |
| Versioning | Yes (immutable versions) | Yes (staging labels) | Yes (versions) |
| Replication | Automatic / user-managed | Single region | Vault-level region |
| Rotation | Pub/Sub + Cloud Function | Built-in Lambda rotation | Event Grid + Function |
| Max secret size | 64 KiB | 64 KiB | 25 KiB |
| CMEK support | Yes | Yes (KMS) | Yes (Key Vault keys) |
| Notifications | Pub/Sub | EventBridge | Event Grid |
| IAM granularity | Per-secret IAM | Per-secret resource policy | Vault-level RBAC |
| Pricing | $0.06/version/month + $0.03/10K access | $0.40/secret/month + $0.05/10K access | $0.03/10K operations |
| Cross-region | Built-in replication | Manual (multi-region deploy) | Vault-level |

### Secret Manager vs Cloud KMS

```
┌──────────────────────────────────────────────────────────────┐
│         SECRET MANAGER vs CLOUD KMS                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌─────────────────────┬─────────────────────────┐          │
│  │ Secret Manager      │ Cloud KMS                │          │
│  ├─────────────────────┼─────────────────────────┤          │
│  │ Stores SECRET DATA  │ Stores ENCRYPTION KEYS  │          │
│  │ (passwords, tokens) │ (for encrypting data)   │          │
│  │                     │                         │          │
│  │ You retrieve the    │ You send data to KMS    │          │
│  │ secret value        │ for encryption/decrypt  │          │
│  │                     │                         │          │
│  │ "Give me the DB     │ "Encrypt this data      │          │
│  │  password"          │  with my key"           │          │
│  │                     │                         │          │
│  │ Versions = values   │ Versions = key material │          │
│  │ IAM controls READ   │ IAM controls USE        │          │
│  └─────────────────────┴─────────────────────────┘          │
│                                                                │
│  They complement each other:                                  │
│  • Secret Manager stores secrets (encrypted by KMS)          │
│  • CMEK: use YOUR KMS key to encrypt secrets                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Pricing

| Component | Cost |
|-----------|------|
| Active secret versions | $0.06/version/month |
| Access operations | $0.03 per 10,000 |
| Destroyed versions | Free |
| Replication | Included (no extra charge) |

---

## Part 2 — Secrets & Secret Versions

### Secret Resource Model

```
┌──────────────────────────────────────────────────────────────┐
│         SECRET MANAGER RESOURCE MODEL                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Project                                                       │
│  └── Secret (named container)                                 │
│      ├── Metadata                                             │
│      │   ├── Name: "db-password"                             │
│      │   ├── Replication: automatic / user-managed            │
│      │   ├── Labels: env=prod, team=backend                  │
│      │   ├── Expiration: 2026-12-31T00:00:00Z (optional)    │
│      │   ├── Rotation: every 90 days (optional)              │
│      │   ├── Pub/Sub topics: [...] (optional)                │
│      │   └── CMEK key: projects/.../keys/... (optional)      │
│      │                                                        │
│      └── Versions (immutable payloads)                       │
│          ├── Version 1: payload bytes  (state: DESTROYED)    │
│          ├── Version 2: payload bytes  (state: DISABLED)     │
│          └── Version 3: payload bytes  (state: ENABLED)      │
│                                                                │
│  Resource name:                                                │
│  projects/PROJECT_ID/secrets/SECRET_NAME                     │
│  projects/PROJECT_ID/secrets/SECRET_NAME/versions/VERSION    │
│                                                                │
│  Special version aliases:                                      │
│  • "latest" → most recently created enabled version          │
│  • Custom aliases: "prod", "staging" (user-defined)          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Version States

| State | Description | Can Access Payload? | Billed? |
|-------|-------------|---------------------|---------|
| `ENABLED` | Active, accessible | Yes | Yes |
| `DISABLED` | Temporarily inaccessible | No | Yes |
| `DESTROYED` | Permanently deleted | No | No |

### Version Lifecycle

```
┌──────┐     ┌──────────┐     ┌───────────┐
│ENABLED│────►│ DISABLED │────►│ DESTROYED │
│       │     │          │     │           │
└───┬───┘     └────┬─────┘     └───────────┘
    │              │ re-enable       ▲
    │              ▼                 │
    │         ┌──────────┐          │
    │         │ ENABLED  │──────────┘
    │         │ (again)  │  destroy
    │         └──────────┘
    │
    └──────────────────────────────►┐
                destroy             │
                                    ▼
                              ┌───────────┐
                              │ DESTROYED │
                              │(permanent)│
                              └───────────┘
```

---

## Part 3 — Creating & Accessing Secrets

### Creating Secrets

```bash
# Create a secret (empty — no version yet)
gcloud secrets create db-password \
    --replication-policy="automatic" \
    --labels="env=prod,team=backend"

# Add a secret version (the actual value)
echo -n "str0ng!P@ss789" | gcloud secrets versions add db-password \
    --data-file=-

# Create secret + version in one step
echo -n "sk_live_abc123" | gcloud secrets create api-key-stripe \
    --replication-policy="automatic" \
    --data-file=-

# Create from a file
gcloud secrets create tls-cert \
    --replication-policy="automatic" \
    --data-file=./server.crt
```

### Accessing Secrets

```bash
# Access the latest version
gcloud secrets versions access latest --secret=db-password

# Access a specific version
gcloud secrets versions access 2 --secret=db-password

# Access and save to file
gcloud secrets versions access latest --secret=tls-cert \
    --out-file=./server.crt

# List all versions
gcloud secrets versions list db-password

# Describe a secret (metadata only)
gcloud secrets describe db-password
```

### Programmatic Access

```python
# pip install google-cloud-secret-manager
from google.cloud import secretmanager

def access_secret(project_id, secret_id, version_id="latest"):
    """Access a secret version."""
    client = secretmanager.SecretManagerServiceClient()
    
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version_id}"
    
    response = client.access_secret_version(request={"name": name})
    
    # Verify checksum (integrity check)
    crc32c = response.payload.data_crc32c
    payload = response.payload.data.decode("UTF-8")
    
    return payload

# Usage
db_password = access_secret("my-project", "db-password")
```

```go
package main

import (
    "context"
    "fmt"
    secretmanager "cloud.google.com/go/secretmanager/apiv1"
    smpb "cloud.google.com/go/secretmanager/apiv1/secretmanagerpb"
)

func accessSecret(projectID, secretID, versionID string) (string, error) {
    ctx := context.Background()
    client, err := secretmanager.NewClient(ctx)
    if err != nil {
        return "", err
    }
    defer client.Close()

    name := fmt.Sprintf("projects/%s/secrets/%s/versions/%s",
        projectID, secretID, versionID)

    resp, err := client.AccessSecretVersion(ctx, &smpb.AccessSecretVersionRequest{
        Name: name,
    })
    if err != nil {
        return "", err
    }

    return string(resp.Payload.Data), nil
}
```

```javascript
// npm install @google-cloud/secret-manager
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');

async function accessSecret(projectId, secretId, versionId = 'latest') {
  const client = new SecretManagerServiceClient();

  const name = `projects/${projectId}/secrets/${secretId}/versions/${versionId}`;
  const [version] = await client.accessSecretVersion({ name });

  return version.payload.data.toString('utf8');
}

// Usage
const dbPassword = await accessSecret('my-project', 'db-password');
```

```java
import com.google.cloud.secretmanager.v1.*;

public class SecretAccess {
    public static String accessSecret(String projectId, String secretId) 
            throws Exception {
        try (SecretManagerServiceClient client = 
                SecretManagerServiceClient.create()) {
            
            SecretVersionName name = SecretVersionName.of(
                projectId, secretId, "latest");
            
            AccessSecretVersionResponse response = 
                client.accessSecretVersion(name);
            
            return response.getPayload().getData().toStringUtf8();
        }
    }
}
```

---

## Part 4 — Replication Policies

### Replication Options

```
┌──────────────────────────────────────────────────────────────┐
│         REPLICATION POLICIES                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  AUTOMATIC REPLICATION (recommended):                          │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Google manages replication across regions            │    │
│  │  • Secret data replicated to multiple locations      │    │
│  │  • High availability by default                      │    │
│  │  • No configuration needed                           │    │
│  │  • Can still use CMEK (single key protects all)      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  USER-MANAGED REPLICATION (for compliance):                   │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  You specify exactly which regions store the secret  │    │
│  │  • Useful for data residency (GDPR, etc.)           │    │
│  │  • Each replica can have its own CMEK key            │    │
│  │  • At least 1 region required                        │    │
│  │  • More regions = higher availability                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Example — EU-only secret:                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Secret: "eu-customer-db-password"                   │    │
│  │  Replication: USER_MANAGED                           │    │
│  │  ├── Replica: europe-west1 (Belgium)                │    │
│  │  │   └── CMEK: projects/.../keys/eu-secret-key-1   │    │
│  │  └── Replica: europe-west4 (Netherlands)            │    │
│  │      └── CMEK: projects/.../keys/eu-secret-key-2   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ⚠ CANNOT change replication policy after creation           │
│    (must create a new secret)                                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating Secrets with Replication

```bash
# Automatic replication (default)
gcloud secrets create my-secret --replication-policy="automatic"

# User-managed: single region
gcloud secrets create eu-secret \
    --replication-policy="user-managed" \
    --locations="europe-west1"

# User-managed: multiple regions
gcloud secrets create eu-secret-ha \
    --replication-policy="user-managed" \
    --locations="europe-west1,europe-west4"
```

---

## Part 5 — Secret Rotation

### Rotation Architecture

```
┌──────────────────────────────────────────────────────────────┐
│         SECRET ROTATION                                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Secret Manager provides rotation SCHEDULING, but YOU        │
│  implement the actual rotation logic:                        │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                                                      │    │
│  │  1. Rotation schedule triggers                       │    │
│  │     ▼                                                │    │
│  │  2. Pub/Sub notification sent                        │    │
│  │     ▼                                                │    │
│  │  3. Cloud Function receives notification             │    │
│  │     ▼                                                │    │
│  │  4. Function generates new secret value              │    │
│  │     (e.g., new DB password)                          │    │
│  │     ▼                                                │    │
│  │  5. Function updates the external system              │    │
│  │     (e.g., ALTER USER SET PASSWORD in Cloud SQL)     │    │
│  │     ▼                                                │    │
│  │  6. Function adds new version to Secret Manager      │    │
│  │     ▼                                                │    │
│  │  7. Function disables old version                    │    │
│  │                                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Timeline:                                            │    │
│  │  Day 0     Day 90    Day 180   Day 270               │    │
│  │  ──┼─────────┼─────────┼─────────┼──                │    │
│  │    v1        v2        v3        v4                   │    │
│  │  (rotate)  (rotate)  (rotate)  (rotate)              │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Configuring Rotation

```bash
# Set rotation schedule on a secret
gcloud secrets update db-password \
    --next-rotation-time="2026-06-01T00:00:00Z" \
    --rotation-period="7776000s"  # 90 days

# Set up Pub/Sub topic for rotation notifications
gcloud secrets update db-password \
    --add-topics="projects/my-project/topics/secret-rotation"
```

### Rotation Cloud Function Example

```python
# Cloud Function triggered by Pub/Sub rotation notification
import functions_framework
import json
import secrets
import string
from google.cloud import secretmanager, sql_v1beta4

@functions_framework.cloud_event
def rotate_db_password(cloud_event):
    """Rotate Cloud SQL database password."""
    
    # Parse the notification
    data = json.loads(cloud_event.data["message"]["data"])
    secret_name = data["name"]  # e.g., "projects/my-project/secrets/db-password"
    
    # 1. Generate a new password
    alphabet = string.ascii_letters + string.digits + "!@#$%"
    new_password = ''.join(secrets.choice(alphabet) for _ in range(32))
    
    # 2. Update Cloud SQL user password
    sql_client = sql_v1beta4.SqlUsersServiceClient()
    sql_client.update(
        project="my-project",
        instance="my-db-instance",
        name="app-user",
        body={"password": new_password},
    )
    
    # 3. Add new version to Secret Manager
    sm_client = secretmanager.SecretManagerServiceClient()
    sm_client.add_secret_version(
        request={
            "parent": secret_name,
            "payload": {"data": new_password.encode("UTF-8")},
        }
    )
    
    # 4. Disable old version (optional — keep for rollback window)
    versions = sm_client.list_secret_versions(
        request={"parent": secret_name}
    )
    for version in versions:
        if version.state == secretmanager.SecretVersion.State.ENABLED:
            if version.name != f"{secret_name}/versions/latest":
                sm_client.disable_secret_version(
                    request={"name": version.name}
                )
    
    print(f"Successfully rotated secret: {secret_name}")
```

---

## Part 6 — Expiration & TTL

### Secret Expiration

```bash
# Create a secret that expires on a specific date
gcloud secrets create temp-api-key \
    --replication-policy="automatic" \
    --expire-time="2026-12-31T23:59:59Z"

# Create a secret with TTL (time-to-live)
gcloud secrets create temp-token \
    --replication-policy="automatic" \
    --ttl="86400s"  # 24 hours from creation

# Update expiration
gcloud secrets update temp-api-key \
    --expire-time="2027-06-30T23:59:59Z"

# Remove expiration
gcloud secrets update temp-api-key \
    --remove-expiration
```

### Expiration Behavior

```
When a secret expires:
• The SECRET is deleted (not just versions)
• ALL versions are destroyed
• This is IRREVERSIBLE
• No notification is sent before expiration

Use cases:
• Temporary credentials for contractors
• Short-lived API keys
• Time-limited access tokens
• Secrets that must be rotated or deleted by policy
```

---

## Part 7 — Labels, Annotations & Aliases

### Labels

```bash
# Add labels when creating
gcloud secrets create db-password \
    --replication-policy="automatic" \
    --labels="env=prod,team=backend,compliance=pci"

# Update labels
gcloud secrets update db-password \
    --update-labels="env=prod,team=platform"

# Remove labels
gcloud secrets update db-password \
    --remove-labels="compliance"

# Filter secrets by label
gcloud secrets list --filter="labels.env=prod"
```

### Version Aliases

```bash
# Set an alias for a version
gcloud secrets versions access latest --secret=db-password
# Alias "latest" is always available and points to newest enabled version

# Custom aliases (via API / Terraform)
# Aliases let you refer to versions by name instead of number:
# "prod" → version 5
# "staging" → version 6
```

### Annotations

```bash
# Annotations store arbitrary metadata (not queryable)
gcloud secrets update db-password \
    --update-annotations="rotation-owner=security-team,jira-ticket=SEC-1234"
```

---

## Part 8 — IAM & Access Control

### IAM Roles

| Role | Description | Use For |
|------|-------------|---------|
| `roles/secretmanager.admin` | Full management | Secret administrators |
| `roles/secretmanager.secretAccessor` | Access secret payloads | Applications reading secrets |
| `roles/secretmanager.secretVersionAdder` | Add new versions | Rotation functions |
| `roles/secretmanager.secretVersionManager` | Enable/disable/destroy versions | Version lifecycle |
| `roles/secretmanager.viewer` | View metadata (not payloads) | Auditors |

### Granting Access

```bash
# Grant access to a specific secret (most common)
gcloud secrets add-iam-policy-binding db-password \
    --member="serviceAccount:app@my-project.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

# Grant access at project level (all secrets)
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:app@my-project.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

# Rotation function: needs accessor + version adder
gcloud secrets add-iam-policy-binding db-password \
    --member="serviceAccount:rotation-fn@my-project.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding db-password \
    --member="serviceAccount:rotation-fn@my-project.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretVersionAdder"
```

### Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│         IAM BEST PRACTICES                                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Grant per-secret, not per-project                        │
│     → Each app only accesses its own secrets                 │
│                                                                │
│  2. Use secretAccessor (not admin) for applications          │
│     → Least privilege: only read, can't modify               │
│                                                                │
│  3. Separate admin from accessor                              │
│     → Admin team creates secrets                             │
│     → App team reads secrets                                 │
│     → Neither can do the other's job                         │
│                                                                │
│  4. Use IAM Conditions for time-limited access               │
│     → Contractor access expires on date X                    │
│                                                                │
│  5. Use Workload Identity for GKE pods                       │
│     → Map KSA to GSA with secretAccessor role               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 9 — CMEK Encryption

### Default vs CMEK Encryption

```
┌──────────────────────────────────────────────────────────────┐
│         SECRET ENCRYPTION OPTIONS                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Default encryption:                                           │
│  • All secrets encrypted at rest with Google-managed keys    │
│  • AES-256                                                    │
│  • Automatic key rotation                                    │
│  • No configuration needed                                   │
│                                                                │
│  CMEK encryption:                                              │
│  • You specify a Cloud KMS key                               │
│  • Secret data encrypted with YOUR key                       │
│  • You control key lifecycle (rotate, disable, destroy)      │
│  • Disabling KMS key = secret inaccessible                   │
│  • Destroying KMS key = secret permanently lost              │
│                                                                │
│  For user-managed replication:                                │
│  • Each replica can have its own CMEK key                    │
│  • Keys MUST be in the same region as the replica            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Configuring CMEK

```bash
# Create secret with CMEK (automatic replication)
gcloud secrets create sensitive-data \
    --replication-policy="automatic" \
    --kms-key-name="projects/my-project/locations/global/keyRings/secret-keys/cryptoKeys/secret-cmek"

# Create secret with CMEK (user-managed, per-replica keys)
gcloud secrets create eu-data \
    --replication-policy="user-managed" \
    --locations="europe-west1,europe-west4" \
    --kms-key-name="projects/my-project/locations/europe-west1/keyRings/keys/cryptoKeys/eu-key-1,projects/my-project/locations/europe-west4/keyRings/keys/cryptoKeys/eu-key-2"
```

```bash
# Grant Secret Manager service agent access to KMS key
PROJECT_NUMBER=$(gcloud projects describe my-project --format='value(projectNumber)')

gcloud kms keys add-iam-policy-binding secret-cmek \
    --keyring=secret-keys \
    --location=global \
    --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-secretmanager.iam.gserviceaccount.com" \
    --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

---

## Part 10 — Pub/Sub Notifications

### Event Notifications

```
┌──────────────────────────────────────────────────────────────┐
│         PUB/SUB NOTIFICATIONS                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Secret Manager can publish events to Pub/Sub topics:         │
│                                                                │
│  Events:                                                       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  SECRET_ROTATE                                       │    │
│  │  → Rotation schedule triggered                       │    │
│  │  → Use to trigger rotation Cloud Function            │    │
│  │                                                      │    │
│  │  SECRET_VERSION_ADD                                  │    │
│  │  → New version added                                 │    │
│  │  → Use to notify teams / trigger deployments         │    │
│  │                                                      │    │
│  │  SECRET_VERSION_ENABLE                               │    │
│  │  → Version enabled                                   │    │
│  │                                                      │    │
│  │  SECRET_VERSION_DISABLE                              │    │
│  │  → Version disabled                                  │    │
│  │                                                      │    │
│  │  SECRET_VERSION_DESTROY                              │    │
│  │  → Version destroyed                                 │    │
│  │                                                      │    │
│  │  SECRET_DELETE                                       │    │
│  │  → Secret deleted                                    │    │
│  │  → Use to trigger cleanup / alerting                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Flow:                                                        │
│  Secret Manager → Pub/Sub → Cloud Function / Cloud Run       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Setting Up Notifications

```bash
# Create Pub/Sub topic
gcloud pubsub topics create secret-events

# Grant Secret Manager permission to publish
PROJECT_NUMBER=$(gcloud projects describe my-project --format='value(projectNumber)')
gcloud pubsub topics add-iam-policy-binding secret-events \
    --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-secretmanager.iam.gserviceaccount.com" \
    --role="roles/pubsub.publisher"

# Add topic to secret
gcloud secrets update db-password \
    --add-topics="projects/my-project/topics/secret-events"

# Remove topic
gcloud secrets update db-password \
    --remove-topics="projects/my-project/topics/secret-events"
```

---

## Part 11 — Integration with GCP Services

### Cloud Run

```yaml
# Cloud Run service.yaml — mount secret as env var
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-api
spec:
  template:
    spec:
      containers:
        - image: gcr.io/my-project/my-api:latest
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: latest           # version
                  name: db-password     # secret name
```

```bash
# CLI: deploy Cloud Run with secret as env var
gcloud run deploy my-api \
    --image=gcr.io/my-project/my-api:latest \
    --set-secrets="DB_PASSWORD=db-password:latest"

# Mount as file
gcloud run deploy my-api \
    --image=gcr.io/my-project/my-api:latest \
    --set-secrets="/secrets/db-password=db-password:latest"
```

### Cloud Functions

```bash
# Cloud Functions Gen2 with secret
gcloud functions deploy my-function \
    --gen2 \
    --runtime=python311 \
    --set-secrets="DB_PASSWORD=db-password:latest"

# Mount as volume
gcloud functions deploy my-function \
    --gen2 \
    --runtime=python311 \
    --set-secrets="/secrets/db-password=db-password:latest"
```

### GKE

```yaml
# GKE — using Workload Identity + application code
# Pod's KSA is mapped to a GSA with secretAccessor role

# Option 1: Application reads from Secret Manager API
# (recommended — always gets latest version)

# Option 2: Use Secret Store CSI Driver
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: gcp-secrets
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/my-project/secrets/db-password/versions/latest"
        path: "db-password"
---
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app-sa
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: secrets
          mountPath: "/secrets"
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: gcp-secrets
```

### Compute Engine

```bash
# On a Compute Engine VM, use the metadata server or client library
# The VM's service account needs roles/secretmanager.secretAccessor

# Using gcloud on the VM:
gcloud secrets versions access latest --secret=db-password

# Or programmatically (Python):
from google.cloud import secretmanager
client = secretmanager.SecretManagerServiceClient()
response = client.access_secret_version(
    name="projects/my-project/secrets/db-password/versions/latest"
)
password = response.payload.data.decode("UTF-8")
```

---

## Part 12 — Regional vs Global Secrets

### Regional Secret Manager

```
┌──────────────────────────────────────────────────────────────┐
│         REGIONAL vs GLOBAL SECRET MANAGER                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Global Secret Manager (default):                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  • API endpoint: secretmanager.googleapis.com        │    │
│  │  • Secrets replicated across regions (auto or manual)│    │
│  │  • Metadata stored globally                          │    │
│  │  • Most common usage                                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Regional Secret Manager:                                      │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  • API endpoint: REGION-secretmanager.googleapis.com │    │
│  │  • Secret data ONLY in the specified region          │    │
│  │  • Metadata ONLY in the specified region             │    │
│  │  • For strict data residency requirements            │    │
│  │  • Supports CMEK with regional keys                  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  When to use Regional:                                         │
│  • Data sovereignty laws require data in specific country    │
│  • Lower latency (secret in same region as app)              │
│  • Compliance: no metadata leaves the region                 │
│                                                                │
│  When to use Global:                                           │
│  • Multi-region applications                                  │
│  • High availability across regions                          │
│  • No data residency constraints                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 13 — Monitoring & Audit Logging

### Audit Logs

```
┌──────────────────────────────────────────────────────────────┐
│         AUDIT LOGGING                                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Admin Activity (always on, free):                            │
│  • CreateSecret, DeleteSecret, UpdateSecret                  │
│  • AddSecretVersion, DisableSecretVersion                    │
│  • DestroySecretVersion                                      │
│  • SetIamPolicy                                               │
│                                                                │
│  Data Access (must enable, chargeable):                       │
│  • AccessSecretVersion (reading the actual secret value)     │
│  • ListSecretVersions, GetSecret                             │
│                                                                │
│  ⚠ The actual secret value is NEVER logged                  │
│  Only the metadata (who, when, which secret) is recorded.    │
│                                                                │
│  Log entry example:                                            │
│  {                                                             │
│    "protoPayload": {                                          │
│      "methodName": "AccessSecretVersion",                    │
│      "resourceName": "projects/my-project/secrets/           │
│                       db-password/versions/3",               │
│      "authenticationInfo": {                                  │
│        "principalEmail": "app@my-project.iam..."             │
│      }                                                        │
│    }                                                          │
│  }                                                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Monitoring Alerts

```bash
# Alert on secret access from unexpected principals
gcloud logging metrics create secret-unexpected-access \
    --description="Secret access from unexpected principal" \
    --log-filter='resource.type="secretmanager.googleapis.com/Secret" AND protoPayload.methodName="AccessSecretVersion" AND NOT protoPayload.authenticationInfo.principalEmail=~".*@my-project.iam.gserviceaccount.com"'

# Alert on secret deletion
gcloud logging metrics create secret-deletion \
    --description="Secret deleted" \
    --log-filter='resource.type="secretmanager.googleapis.com/Secret" AND protoPayload.methodName="DeleteSecret"'

# Alert on version destruction
gcloud logging metrics create secret-version-destroyed \
    --description="Secret version destroyed" \
    --log-filter='resource.type="secretmanager.googleapis.com/Secret" AND protoPayload.methodName="DestroySecretVersion"'
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Complete Setup

```hcl
# ─── Secret ──────────────────────────────────────────────────
resource "google_secret_manager_secret" "db_password" {
  secret_id = "db-password"
  project   = var.project_id

  replication {
    auto {}
  }

  labels = {
    env  = "prod"
    team = "backend"
  }

  # Optional: rotation schedule
  rotation {
    rotation_period    = "7776000s"  # 90 days
    next_rotation_time = "2026-08-01T00:00:00Z"
  }

  # Optional: Pub/Sub topics for notifications
  topics {
    name = google_pubsub_topic.secret_events.id
  }
}

# ─── Secret Version (the actual value) ──────────────────────
resource "google_secret_manager_secret_version" "db_password_v1" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = var.db_password  # from Terraform variable (not in state!)

  # Consider using random_password for auto-generation
}

# Auto-generated password
resource "random_password" "db_password" {
  length  = 32
  special = true
}

resource "google_secret_manager_secret_version" "db_password_auto" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = random_password.db_password.result
}

# ─── User-Managed Replication with CMEK ──────────────────────
resource "google_secret_manager_secret" "eu_secret" {
  secret_id = "eu-customer-data"
  project   = var.project_id

  replication {
    user_managed {
      replicas {
        location = "europe-west1"
        customer_managed_encryption {
          kms_key_name = google_kms_crypto_key.eu_key_1.id
        }
      }
      replicas {
        location = "europe-west4"
        customer_managed_encryption {
          kms_key_name = google_kms_crypto_key.eu_key_2.id
        }
      }
    }
  }
}

# ─── IAM ─────────────────────────────────────────────────────
resource "google_secret_manager_secret_iam_member" "app_accessor" {
  secret_id = google_secret_manager_secret.db_password.id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.app.email}"
}

resource "google_secret_manager_secret_iam_member" "rotation_fn" {
  secret_id = google_secret_manager_secret.db_password.id
  role      = "roles/secretmanager.secretVersionAdder"
  member    = "serviceAccount:${google_service_account.rotation.email}"
}

# ─── Cloud Run with Secret ──────────────────────────────────
resource "google_cloud_run_v2_service" "api" {
  name     = "my-api"
  location = "us-central1"
  project  = var.project_id

  template {
    service_account = google_service_account.app.email

    containers {
      image = "gcr.io/${var.project_id}/my-api:latest"

      env {
        name = "DB_PASSWORD"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.db_password.secret_id
            version = "latest"
          }
        }
      }
    }
  }

  depends_on = [google_secret_manager_secret_iam_member.app_accessor]
}

# ─── Pub/Sub for Notifications ──────────────────────────────
resource "google_pubsub_topic" "secret_events" {
  name    = "secret-events"
  project = var.project_id
}

resource "google_pubsub_topic_iam_member" "sm_publisher" {
  topic  = google_pubsub_topic.secret_events.id
  role   = "roles/pubsub.publisher"
  member = "serviceAccount:service-${data.google_project.current.number}@gcp-sa-secretmanager.iam.gserviceaccount.com"
}

# ─── Secret with Expiration ─────────────────────────────────
resource "google_secret_manager_secret" "temp_key" {
  secret_id = "contractor-api-key"
  project   = var.project_id

  replication {
    auto {}
  }

  expire_time = "2026-12-31T23:59:59Z"
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# SECRETS
# ═══════════════════════════════════════════════════════════════

# Create secret
gcloud secrets create SECRET_NAME --replication-policy="automatic"

# Create with labels
gcloud secrets create SECRET_NAME \
    --replication-policy="automatic" \
    --labels="env=prod,team=backend"

# List secrets
gcloud secrets list
gcloud secrets list --filter="labels.env=prod"

# Describe secret
gcloud secrets describe SECRET_NAME

# Update secret
gcloud secrets update SECRET_NAME --update-labels="env=staging"

# Delete secret
gcloud secrets delete SECRET_NAME

# ═══════════════════════════════════════════════════════════════
# SECRET VERSIONS
# ═══════════════════════════════════════════════════════════════

# Add version (from stdin)
echo -n "secret-value" | gcloud secrets versions add SECRET_NAME --data-file=-

# Add version (from file)
gcloud secrets versions add SECRET_NAME --data-file=./secret.txt

# Access version
gcloud secrets versions access latest --secret=SECRET_NAME
gcloud secrets versions access VERSION_NUM --secret=SECRET_NAME

# List versions
gcloud secrets versions list SECRET_NAME

# Disable version
gcloud secrets versions disable VERSION_NUM --secret=SECRET_NAME

# Enable version
gcloud secrets versions enable VERSION_NUM --secret=SECRET_NAME

# Destroy version
gcloud secrets versions destroy VERSION_NUM --secret=SECRET_NAME

# ═══════════════════════════════════════════════════════════════
# IAM
# ═══════════════════════════════════════════════════════════════

# Grant access
gcloud secrets add-iam-policy-binding SECRET_NAME \
    --member=MEMBER --role=ROLE

# View policy
gcloud secrets get-iam-policy SECRET_NAME

# ═══════════════════════════════════════════════════════════════
# ROTATION & NOTIFICATIONS
# ═══════════════════════════════════════════════════════════════

# Set rotation schedule
gcloud secrets update SECRET_NAME \
    --next-rotation-time="TIMESTAMP" \
    --rotation-period="DURATION"

# Add Pub/Sub topic
gcloud secrets update SECRET_NAME \
    --add-topics="projects/PROJECT/topics/TOPIC"

# Remove Pub/Sub topic
gcloud secrets update SECRET_NAME \
    --remove-topics="projects/PROJECT/topics/TOPIC"
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Centralized Secret Management for Microservices

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: CENTRALIZED SECRET MANAGEMENT                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Central Secrets Project: "org-secrets-prod"              │        │
│  │                                                           │        │
│  │  Secrets:                                                 │        │
│  │  ├── checkout/db-password                                 │        │
│  │  ├── checkout/stripe-api-key                              │        │
│  │  ├── users/db-password                                    │        │
│  │  ├── users/auth0-client-secret                            │        │
│  │  ├── shared/datadog-api-key                               │        │
│  │  └── shared/slack-webhook-url                             │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  IAM (per-secret granularity):                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  checkout-sa → secretAccessor on checkout/* secrets     │        │
│  │  users-sa    → secretAccessor on users/* secrets        │        │
│  │  all-svc-sa  → secretAccessor on shared/* secrets       │        │
│  │  security-tm → secretmanager.admin (all secrets)        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Benefits:                                                             │
│  • Single source of truth for all secrets                             │
│  • Centralized audit logging                                          │
│  • Consistent rotation policies                                       │
│  • Security team manages lifecycle                                    │
│  • Service teams have read-only access to their secrets               │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Automated Database Credential Rotation

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: AUTOMATED DB CREDENTIAL ROTATION                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Secret: "checkout-db-password"                           │        │
│  │  Rotation: every 30 days                                  │        │
│  │  Pub/Sub topic: "secret-rotation"                         │        │
│  └──────────┬───────────────────────────────────────────────┘        │
│             │ SECRET_ROTATE event                                     │
│             ▼                                                         │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Cloud Function: "rotate-db-credentials"                  │        │
│  │                                                           │        │
│  │  Steps:                                                    │        │
│  │  1. Generate new random password (32 chars)               │        │
│  │  2. Connect to Cloud SQL Admin API                        │        │
│  │  3. Update user password in Cloud SQL                     │        │
│  │  4. Test connection with new password                     │        │
│  │  5. If test passes:                                       │        │
│  │     a. Add new version to Secret Manager                  │        │
│  │     b. Disable previous version                           │        │
│  │  6. If test fails:                                        │        │
│  │     a. Revert Cloud SQL password to old value             │        │
│  │     b. Send alert to security team                        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Application reads "latest" version:                                  │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Cloud Run "checkout-api"                                 │        │
│  │  env: DB_PASSWORD = db-password:latest                    │        │
│  │                                                           │        │
│  │  On rotation:                                              │        │
│  │  • New Cloud Run revision deployed with new secret       │        │
│  │  • Old revision drains connections gracefully             │        │
│  │  • Zero-downtime rotation                                 │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Multi-Environment Secret Promotion

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: SECRET PROMOTION ACROSS ENVIRONMENTS                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐           │
│  │ dev project  │     │ staging     │     │ prod        │           │
│  │              │────►│ project     │────►│ project     │           │
│  └─────────────┘     └─────────────┘     └─────────────┘           │
│                                                                        │
│  Each environment has its own secrets:                                 │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  dev-project:                                             │        │
│  │  ├── db-password  (weak password, no rotation)           │        │
│  │  ├── api-key      (test API key)                         │        │
│  │  └── tls-cert     (self-signed)                          │        │
│  │                                                           │        │
│  │  staging-project:                                         │        │
│  │  ├── db-password  (strong, 90-day rotation)              │        │
│  │  ├── api-key      (sandbox API key)                      │        │
│  │  └── tls-cert     (Let's Encrypt staging)                │        │
│  │                                                           │        │
│  │  prod-project:                                            │        │
│  │  ├── db-password  (strong, 30-day rotation, CMEK, HSM)  │        │
│  │  ├── api-key      (production API key)                   │        │
│  │  └── tls-cert     (production CA-signed)                 │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Same secret NAMES across environments.                               │
│  Application code uses the same secret name regardless               │
│  of environment. The project determines which secret is accessed.     │
│                                                                        │
│  CI/CD pipeline:                                                       │
│  • Terraform creates secrets per environment                          │
│  • Secret values are NEVER in source control                         │
│  • Pipeline sets values via gcloud or Terraform sensitive vars       │
│  • Rotation policies increase in strictness: dev < staging < prod    │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create secret | `gcloud secrets create NAME --replication-policy="automatic"` |
| Add version | `echo -n "value" \| gcloud secrets versions add NAME --data-file=-` |
| Access latest | `gcloud secrets versions access latest --secret=NAME` |
| List secrets | `gcloud secrets list` |
| List versions | `gcloud secrets versions list NAME` |
| Disable version | `gcloud secrets versions disable VER --secret=NAME` |
| Destroy version | `gcloud secrets versions destroy VER --secret=NAME` |
| Delete secret | `gcloud secrets delete NAME` |
| Grant access | `gcloud secrets add-iam-policy-binding NAME --member=M --role=R` |
| Set rotation | `gcloud secrets update NAME --rotation-period=90d` |
| Add Pub/Sub | `gcloud secrets update NAME --add-topics=TOPIC` |
| Max secret size | 64 KiB per version |
| Accessor role | `roles/secretmanager.secretAccessor` |
| Cloud Run mount | `--set-secrets="ENV_VAR=secret:version"` |

---

## Why Not Just Use Environment Variables? (Beginner Explanation)

If you're new to cloud security, you might wonder: "Why can't I just use regular environment variables for my passwords and API keys?"

### The Problem with Environment Variables

```
┌──────────────────────────────────────────────────────────────┐
│         ENV VARS vs SECRET MANAGER                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Environment Variables:                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ✗ Visible in deployment configs, Docker files,      │    │
│  │    CI/CD pipelines, and process listings              │    │
│  │  ✗ No versioning — if you change a value, the old   │    │
│  │    value is gone forever                              │    │
│  │  ✗ No audit trail — you can't see WHO read the      │    │
│  │    password or WHEN                                   │    │
│  │  ✗ No access control — anyone who can view the      │    │
│  │    deployment config can see the password             │    │
│  │  ✗ Not encrypted at rest — stored as plain text     │    │
│  │  ✗ Hard to rotate — must redeploy every service     │    │
│  │    that uses the changed value                        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Secret Manager:                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ✓ Encrypted at rest (AES-256, optional CMEK)       │    │
│  │  ✓ Versioned — every value change creates a new     │    │
│  │    version, old values are preserved                  │    │
│  │  ✓ Audited — Cloud Audit Logs record every access   │    │
│  │    (who, when, which secret)                          │    │
│  │  ✓ IAM access control — per-secret permissions      │    │
│  │  ✓ Automatic rotation support via Pub/Sub           │    │
│  │  ✓ Centralized — one place for all secrets          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### When to Use Each

```
┌──────────────────────────────────────────────────────────────┐
│  USE ENVIRONMENT VARIABLES for:                                │
│  • Non-sensitive configuration (APP_PORT, LOG_LEVEL, NODE_ENV)│
│  • Feature flags (ENABLE_CACHE=true)                          │
│  • Service URLs (API_BASE_URL=https://api.example.com)       │
│  • Runtime settings that are NOT secrets                      │
│                                                                │
│  USE SECRET MANAGER for:                                       │
│  • Database passwords                                          │
│  • API keys and tokens                                        │
│  • TLS certificates and private keys                          │
│  • OAuth client secrets                                       │
│  • SSH keys                                                    │
│  • Any value that would be a security risk if leaked          │
│                                                                │
│  💡 A common pattern: Store the secret in Secret Manager,    │
│     then inject it as an env var at runtime using             │
│     --set-secrets in Cloud Run / Cloud Functions.             │
│     The env var is populated FROM Secret Manager, so you      │
│     get the convenience of env vars with the security of      │
│     Secret Manager.                                            │
└──────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Creating & Managing Secrets

### Step 1 — Navigate to Secret Manager

```
Console → Navigation Menu → Security → Secret Manager
(or search "Secret Manager" in the top search bar)

If this is your first time, you may be prompted to enable the
Secret Manager API. Click "Enable" and wait a few seconds.
```

### Step 2 — Create a Secret

```
Click "+ CREATE SECRET" at the top of the page.

┌──────────────────────────────────────────────────────────────┐
│  CREATE SECRET FORM — All Fields Explained                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Name *                                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  db-password                                          │    │
│  └──────────────────────────────────────────────────────┘    │
│  • Must be unique within the project                         │
│  • Allowed: letters, numbers, hyphens, underscores           │
│  • Cannot be changed after creation                          │
│  • Tip: use a naming convention like "service-type"          │
│    (e.g., checkout-db-password, payments-stripe-key)         │
│                                                                │
│  ─────────────────────────────────────────────────────────   │
│                                                                │
│  Secret value *                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  (enter your secret value here, or upload a file)     │    │
│  └──────────────────────────────────────────────────────┘    │
│  • You can type/paste the value directly                     │
│  • Or click "Upload file" to upload from your machine        │
│  • Max size: 64 KiB per version                              │
│  • This becomes Version 1 of the secret                      │
│                                                                │
│  ─────────────────────────────────────────────────────────   │
│                                                                │
│  Replication policy                                            │
│  ○ Automatic (recommended)                                   │
│    → Google manages replication across regions               │
│    → Best for most use cases                                 │
│  ○ User-managed                                              │
│    → You pick the specific regions                           │
│    → Use for data residency / compliance                     │
│  ⚠ Cannot be changed after creation                        │
│                                                                │
│  ─────────────────────────────────────────────────────────   │
│                                                                │
│  Encryption                                                    │
│  ○ Google-managed encryption key (default)                   │
│  ○ Customer-managed encryption key (CMEK)                    │
│    → Select a Cloud KMS key                                  │
│                                                                │
│  ─────────────────────────────────────────────────────────   │
│                                                                │
│  Rotation (optional — expand "Rotation" section)              │
│  • Set a rotation period (e.g., 90 days)                     │
│  • Set the next rotation time                                │
│  • Requires a Pub/Sub topic for rotation notifications       │
│                                                                │
│  ─────────────────────────────────────────────────────────   │
│                                                                │
│  Expiration (optional — expand "Expiration" section)          │
│  • Set an expiration date or TTL                             │
│  • Secret (and all versions) auto-deleted on expiration      │
│  • ⚠ This is IRREVERSIBLE once the secret expires          │
│                                                                │
│  ─────────────────────────────────────────────────────────   │
│                                                                │
│  Notifications (optional — expand "Notifications" section)    │
│  • Add a Pub/Sub topic to receive event notifications        │
│  • Events: version add, enable, disable, destroy, delete     │
│                                                                │
│  ─────────────────────────────────────────────────────────   │
│                                                                │
│  Labels (optional)                                             │
│  • Key-value pairs for organizing secrets                    │
│  • Example: env=prod, team=backend, compliance=pci           │
│                                                                │
│  ─────────────────────────────────────────────────────────   │
│                                                                │
│  Click "CREATE SECRET"                                        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 3 — View Secret Versions

```
After creating a secret, you are taken to the secret detail page.

Console → Secret Manager → click on "db-password"

┌──────────────────────────────────────────────────────────────┐
│  SECRET DETAIL PAGE                                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Secret: db-password                                          │
│                                                                │
│  VERSIONS tab (default view):                                 │
│  ┌────────┬──────────┬──────────────┬──────────────────┐    │
│  │ Version│ State    │ Created      │ Actions          │    │
│  ├────────┼──────────┼──────────────┼──────────────────┤    │
│  │ 1      │ Enabled  │ May 22, 2026 │ ⋮ (three dots)  │    │
│  └────────┴──────────┴──────────────┴──────────────────┘    │
│                                                                │
│  Other tabs:                                                   │
│  • OVERVIEW — metadata, replication, labels                  │
│  • ROTATION — rotation schedule and Pub/Sub config           │
│  • PERMISSIONS — IAM bindings for this secret                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 4 — Add a New Version

```
On the secret detail page → VERSIONS tab:

1. Click "+ NEW VERSION" at the top
2. Enter the new secret value (or upload a file)
3. Click "ADD NEW VERSION"

The new version appears in the list:
┌────────┬──────────┬──────────────┐
│ Version│ State    │ Created      │
├────────┼──────────┼──────────────┤
│ 2      │ Enabled  │ May 22, 2026 │  ← new (this is now "latest")
│ 1      │ Enabled  │ May 22, 2026 │
└────────┴──────────┴──────────────┘

Both versions remain enabled. The "latest" alias now points to Version 2.
```

### Step 5 — Disable / Enable a Version

```
On the VERSIONS tab:

1. Click the ⋮ (three-dot menu) next to the version
2. Click "Disable"
3. Confirm in the dialog

The version state changes to DISABLED:
┌────────┬──────────┬──────────────┐
│ Version│ State    │ Created      │
├────────┼──────────┼──────────────┤
│ 2      │ Enabled  │ May 22, 2026 │
│ 1      │ Disabled │ May 22, 2026 │  ← disabled
└────────┴──────────┴──────────────┘

• Disabled versions CANNOT be accessed (API returns error)
• Disabled versions are still billed
• To re-enable: click ⋮ → "Enable"
• This is useful for temporarily revoking an old credential
  while keeping it available for rollback
```

### Step 6 — Access a Secret Version (View the Value)

```
On the VERSIONS tab:

1. Click the ⋮ (three-dot menu) next to an ENABLED version
2. Click "View secret value"
3. The value is displayed in a dialog box
   (you can copy it to clipboard)

⚠ This action is recorded in Cloud Audit Logs
  (Data Access log → AccessSecretVersion)

Note: You can only view ENABLED versions.
Disabled or destroyed versions cannot be viewed.
```

---

## Console Walkthrough: Deleting Secrets

### Delete a Secret Version

```
On the secret detail page → VERSIONS tab:

1. Click the ⋮ (three-dot menu) next to the version
2. Click "Destroy"
3. Type the version number to confirm
4. Click "DESTROY VERSION"

┌────────┬───────────┬──────────────┐
│ Version│ State     │ Created      │
├────────┼───────────┼──────────────┤
│ 2      │ Enabled   │ May 22, 2026 │
│ 1      │ Destroyed │ May 22, 2026 │  ← permanently gone
└────────┴───────────┴──────────────┘

⚠ DESTROYED is PERMANENT — the secret value is gone forever.
  The version entry remains in the list but the payload is deleted.
  Destroyed versions are NOT billed.

Tip: Disable first, wait a few days, then destroy.
     This gives you a rollback window in case something breaks.
```

### Delete an Entire Secret

```
From the Secret Manager list page:

Option A — Single secret:
1. Click the ⋮ (three-dot menu) next to the secret name
2. Click "Delete"
3. Type the secret name to confirm
4. Click "DELETE"

Option B — From secret detail page:
1. Open the secret
2. Click "DELETE" at the top of the page
3. Type the secret name to confirm
4. Click "DELETE"

⚠ Deleting a secret destroys ALL versions permanently.
  This action is IRREVERSIBLE.
```

### gcloud Commands for Version & Secret Management

```bash
# ─── Disable a version ──────────────────────────────────────
gcloud secrets versions disable VERSION_NUM --secret=SECRET_NAME

# Example: disable version 1 of db-password
gcloud secrets versions disable 1 --secret=db-password

# ─── Enable a previously disabled version ────────────────────
gcloud secrets versions enable VERSION_NUM --secret=SECRET_NAME

# Example: re-enable version 1 of db-password
gcloud secrets versions enable 1 --secret=db-password

# ─── Destroy a version (PERMANENT) ──────────────────────────
gcloud secrets versions destroy VERSION_NUM --secret=SECRET_NAME

# Example: destroy version 1 of db-password
gcloud secrets versions destroy 1 --secret=db-password
# You will be prompted to confirm. Add --quiet to skip the prompt.

# ─── Delete an entire secret (PERMANENT) ─────────────────────
gcloud secrets delete SECRET_NAME

# Example: delete the db-password secret and all its versions
gcloud secrets delete db-password
# You will be prompted to confirm. Add --quiet to skip the prompt.

# ─── Verify what's left ─────────────────────────────────────
gcloud secrets list
gcloud secrets versions list SECRET_NAME
```

---

## Secret Rotation with Cloud Functions

### Why Rotation Matters

```
┌──────────────────────────────────────────────────────────────┐
│  WHY ROTATE SECRETS?                                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  • Limits blast radius — if a secret leaks, the damage      │
│    window is limited to the rotation interval                │
│  • Compliance — many standards (PCI-DSS, SOC 2, HIPAA)      │
│    require periodic credential rotation                      │
│  • Stale credentials — old passwords may have been           │
│    shared, logged, or cached in places you don't know about  │
│  • Automated rotation eliminates human error and ensures     │
│    secrets are always fresh without manual intervention       │
│                                                                │
│  Without rotation:                                             │
│  ───────────────────────────────────────────────────────►    │
│  Password created ──── leaked on Day 50 ──── still valid     │
│                         on Day 500 (10 months of exposure!)  │
│                                                                │
│  With 30-day rotation:                                        │
│  ──┼──────┼──────┼──────┼──────┼──────┼──────┼──►           │
│   v1     v2     v3     v4     v5     v6     v7               │
│          ▲ leaked on Day 50                                   │
│          └─ but v2 was already replaced by v3 on Day 60      │
│             → max 10 days of exposure, not 450!              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### End-to-End Rotation Example

This example shows the complete flow: Secret Manager triggers a Pub/Sub notification on schedule, which invokes a Cloud Function that rotates the database password.

```
┌──────────────────────────────────────────────────────────────┐
│  ROTATION FLOW (End-to-End)                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────┐  rotation      ┌─────────────────┐      │
│  │ Secret Manager │  schedule      │ Pub/Sub Topic    │      │
│  │ "db-password"  │──triggers──►  │ "secret-rotate"  │      │
│  │ (every 30 days)│  SECRET_ROTATE │                  │      │
│  └────────────────┘               └────────┬─────────┘      │
│                                             │                 │
│                                             ▼                 │
│                                   ┌─────────────────┐        │
│                                   │ Cloud Function   │        │
│                                   │ "rotate-db-pw"   │        │
│                                   │                   │        │
│                                   │ 1. Generate new  │        │
│                                   │    password       │        │
│                                   │ 2. Update Cloud  │        │
│                                   │    SQL password   │        │
│                                   │ 3. Verify new    │        │
│                                   │    password works │        │
│                                   │ 4. Add new secret│        │
│                                   │    version        │        │
│                                   │ 5. Disable old   │        │
│                                   │    version        │        │
│                                   └─────────────────┘        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

#### Step 1 — Set Up the Pub/Sub Topic

```bash
# Create the Pub/Sub topic
gcloud pubsub topics create secret-rotate

# Grant Secret Manager permission to publish to the topic
PROJECT_NUMBER=$(gcloud projects describe my-project --format='value(projectNumber)')
gcloud pubsub topics add-iam-policy-binding secret-rotate \
    --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-secretmanager.iam.gserviceaccount.com" \
    --role="roles/pubsub.publisher"
```

#### Step 2 — Configure the Secret for Rotation

```bash
# Enable rotation on the secret (every 30 days)
gcloud secrets update db-password \
    --next-rotation-time="2026-07-01T00:00:00Z" \
    --rotation-period="2592000s" \
    --add-topics="projects/my-project/topics/secret-rotate"
```

#### Step 3 — Deploy the Rotation Cloud Function

```python
# main.py — Cloud Function to rotate a Cloud SQL password
import functions_framework
import json
import secrets
import string
from google.cloud import secretmanager
from google.cloud.sql.connector import Connector

@functions_framework.cloud_event
def rotate_db_password(cloud_event):
    """Triggered by SECRET_ROTATE Pub/Sub notification."""

    # Parse the notification to get the secret name
    data = json.loads(cloud_event.data["message"]["data"])
    secret_name = data["name"]
    # secret_name = "projects/my-project/secrets/db-password"

    # 1. Generate a strong new password
    alphabet = string.ascii_letters + string.digits + "!@#$%^&*"
    new_password = ''.join(secrets.choice(alphabet) for _ in range(32))

    # 2. Update the password in Cloud SQL
    connector = Connector()
    conn = connector.connect(
        "my-project:us-central1:my-db-instance",
        "pymysql",
        user="root",
        password=_get_current_password(secret_name),
        db="mysql",
    )
    with conn.cursor() as cursor:
        cursor.execute(
            "ALTER USER 'app-user'@'%%' IDENTIFIED BY %s", (new_password,)
        )
    conn.close()

    # 3. Verify the new password works
    test_conn = connector.connect(
        "my-project:us-central1:my-db-instance",
        "pymysql",
        user="app-user",
        password=new_password,
        db="myapp",
    )
    test_conn.close()
    connector.close()

    # 4. Add new version to Secret Manager
    sm_client = secretmanager.SecretManagerServiceClient()
    sm_client.add_secret_version(
        request={
            "parent": secret_name,
            "payload": {"data": new_password.encode("UTF-8")},
        }
    )

    print(f"Successfully rotated: {secret_name}")


def _get_current_password(secret_name):
    """Retrieve the current password from Secret Manager."""
    client = secretmanager.SecretManagerServiceClient()
    response = client.access_secret_version(
        request={"name": f"{secret_name}/versions/latest"}
    )
    return response.payload.data.decode("UTF-8")
```

```bash
# Deploy the Cloud Function
gcloud functions deploy rotate-db-pw \
    --gen2 \
    --runtime=python311 \
    --trigger-topic=secret-rotate \
    --source=./rotation-function/ \
    --entry-point=rotate_db_password \
    --service-account=rotation-fn@my-project.iam.gserviceaccount.com

# Grant the function's service account the necessary roles
gcloud secrets add-iam-policy-binding db-password \
    --member="serviceAccount:rotation-fn@my-project.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding db-password \
    --member="serviceAccount:rotation-fn@my-project.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretVersionAdder"
```

#### What Happens Every 30 Days (Automatically)

```
Day 0:   Secret Manager triggers SECRET_ROTATE → Pub/Sub
         → Cloud Function generates new password
         → Updates Cloud SQL user password
         → Adds Version N+1 to Secret Manager
         → Applications using "latest" pick up new password

Day 30:  Same cycle repeats → Version N+2
Day 60:  Same cycle repeats → Version N+3
...

You never touch a password manually again.
```

---

## What's Next?

Continue to **Chapter 42: Security Command Center** → `42-security-command-center.md`
