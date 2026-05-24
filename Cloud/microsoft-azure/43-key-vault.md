# Chapter 43: Azure Key Vault

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Key Vault Fundamentals](#part-1-key-vault-fundamentals)
- [Part 2: Creating a Key Vault (Portal Walkthrough)](#part-2-creating-a-key-vault-portal-walkthrough)
- [Part 3: Secrets](#part-3-secrets)
- [Part 4: Keys](#part-4-keys)
- [Part 5: Certificates](#part-5-certificates)
- [Part 6: Access Policies vs RBAC](#part-6-access-policies-vs-rbac)
- [Part 7: Key Vault in Applications](#part-7-key-vault-in-applications)
- [Part 8: Terraform & Bicep](#part-8-terraform--bicep)
- [Part 9: az CLI Reference](#part-9-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Key Vault is a secure cloud safe for storing secrets (passwords, API keys, connection strings), encryption keys, and SSL/TLS certificates. Instead of putting passwords in your code or config files, you store them in Key Vault and your application fetches them at runtime.

```
What you'll learn:
├── Key Vault Fundamentals
│   ├── Why use Key Vault (never store secrets in code!)
│   ├── Standard vs Premium tier
│   └── Soft delete & purge protection
├── Creating a Key Vault (Portal)
├── Secrets (passwords, API keys, connection strings)
├── Keys (encryption keys for data protection)
├── Certificates (SSL/TLS certificates)
├── Access Policies vs RBAC (who can access what)
├── Key Vault in Applications (code integration)
├── Terraform, Bicep, az CLI
└── Quick reference
```

---

## Part 1: Key Vault Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           KEY VAULT OVERVIEW                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Why Key Vault?                                                       │
│ ❌ Bad: connectionString = "Server=mydb;Password=abc123" in code│
│ ✅ Good: connectionString = KeyVault.GetSecret("db-conn-string")│
│                                                                       │
│ What it stores:                                                      │
│ ├── Secrets: Passwords, API keys, connection strings            │
│ │   (any string value you want to keep secure)                  │
│ ├── Keys: Encryption keys (RSA, EC)                             │
│ │   (used for encrypting data, signing tokens)                  │
│ └── Certificates: SSL/TLS certificates                          │
│     (auto-renewal, integration with App Service)                │
│                                                                       │
│ Tiers:                                                               │
│ ├── Standard: Software-protected keys ($0.03/10k operations)   │
│ └── Premium: HSM-protected keys (hardware security module)     │
│     ⚡ HSM = Keys never leave tamper-proof hardware!            │
│                                                                       │
│ Safety features:                                                     │
│ ├── Soft delete (default: enabled, 7-90 days retention)        │
│ │   Deleted items can be recovered within retention period      │
│ ├── Purge protection (optional, recommended for production)     │
│ │   Prevents permanent deletion even by admins                  │
│ ├── Audit logging (every access is logged)                      │
│ └── Network access (firewall, private endpoint, VNet)          │
│                                                                       │
│ URL format: https://myvault.vault.azure.net/                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Key Vault (Portal Walkthrough)

```
Console → Key vaults → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE KEY VAULT                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Basics ──                                                        │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-security ▼]                                    │
│                                                                       │
│ Key vault name: [kv-myapp-prod]                                    │
│ ⚡ Globally unique. Creates: kv-myapp-prod.vault.azure.net      │
│                                                                       │
│ Region: [Central India ▼]                                          │
│ Pricing tier: [Standard ▼]                                        │
│                                                                       │
│ Days to retain deleted vaults: [90]                                │
│ Purge protection: ● Enable  ○ Disable                            │
│ ⚡ Recommended for production! Prevents accidental permanent    │
│   deletion of secrets/keys/certificates.                          │
│                                                                       │
│ ── Access Configuration ──                                         │
│ Permission model:                                                   │
│ ● Azure role-based access control (RBAC) ← Recommended         │
│ ○ Vault access policy (legacy)                                  │
│                                                                       │
│ ── Networking ──                                                    │
│ ● Public endpoint (all networks)                                  │
│ ○ Public endpoint (selected networks)                           │
│ ○ Private endpoint                                               │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Secrets

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECRETS                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Key vault → Secrets → [+ Generate/Import]                        │
│                                                                       │
│ Upload options: ● Manual  ○ Certificate                          │
│ Name: [db-connection-string]                                       │
│ Value: [Server=mydb.database.windows.net;Password=***]           │
│ Content type: [text/plain] (optional, for documentation)         │
│ Set activation date: ☐                                            │
│ Set expiration date: ☑ [2025-12-31]                              │
│ Enabled: ● Yes  ○ No                                            │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ Versioning:                                                          │
│ ├── Every update creates a NEW version                           │
│ ├── Previous versions are retained                                │
│ ├── Current version: https://kv-myapp-prod.vault.azure.net/     │
│ │   secrets/db-connection-string (latest)                       │
│ ├── Specific version: .../secrets/db-connection-string/<version>│
│ └── Can enable/disable individual versions                      │
│                                                                       │
│ Common secrets to store:                                            │
│ ├── Database connection strings                                   │
│ ├── API keys (third-party services)                              │
│ ├── Storage account keys                                         │
│ ├── OAuth client secrets                                         │
│ ├── Encryption passwords                                         │
│ └── Any sensitive configuration value                            │
│                                                                       │
│ Secret rotation:                                                     │
│ ├── Manual: Update the secret value periodically                │
│ └── Automatic: Key Vault can rotate some secrets via Event Grid│
│     Key Vault → Events → Subscribe to "SecretNearExpiry"      │
│     → Trigger Azure Function → Rotate secret                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Keys

```
Key vault → Keys → [+ Generate/Import]

Name: [encryption-key]
Key type: ● RSA  ○ EC  ○ oct (symmetric)
RSA key size: [2048 ▼] (2048/3072/4096)
Key operations: ☑ Encrypt ☑ Decrypt ☑ Sign ☑ Verify ☑ Wrap ☑ Unwrap

Use cases:
├── Encrypt/decrypt data (client-side encryption)
├── Azure Disk Encryption (customer-managed key)
├── Storage Service Encryption (CMK)
├── SQL Transparent Data Encryption (TDE) with BYOK
├── Sign and verify tokens (JWT signing)
└── Wrap/unwrap data encryption keys (envelope encryption)

⚡ Premium tier: Keys stored in HSM (hardware, FIPS 140-2 Level 2)
⚡ Standard tier: Keys protected by software
```

---

## Part 5: Certificates

```
Key vault → Certificates → [+ Generate/Import]

┌─────────────────────────────────────────────────────────────────────┐
│           CERTIFICATES                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Method: ● Generate  ○ Import                                      │
│                                                                       │
│ Certificate name: [myapp-ssl-cert]                                 │
│ Type of CA: ● Self-signed  ○ Integrated CA (DigiCert/GlobalSign)│
│ Subject: [CN=myapp.com]                                            │
│ DNS names: [myapp.com, www.myapp.com]                             │
│ Validity: [12] months                                               │
│ Content type: [PKCS #12 ▼]                                       │
│ Lifetime action: Auto-renew at 80% of lifetime                  │
│                                                                       │
│ For production SSL:                                                  │
│ ├── Integrated CA: DigiCert or GlobalSign                        │
│ │   Key Vault automatically renews before expiry!               │
│ ├── Non-integrated CA: Generate CSR, submit to your CA          │
│ └── Import: Upload existing .pfx or .pem certificate            │
│                                                                       │
│ Integration with App Service:                                       │
│ App Service → TLS/SSL → Import Key Vault Certificate           │
│ ⚡ App Service automatically picks up renewed certificates!    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Access Policies vs RBAC

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACCESS CONTROL                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Option 1: RBAC (recommended)                                       │
│ Key vault → Access control (IAM) → [+ Add role assignment]      │
│                                                                       │
│ Built-in roles:                                                      │
│ ├── Key Vault Administrator → Full access to all data          │
│ ├── Key Vault Secrets User → Read secrets only                 │
│ ├── Key Vault Secrets Officer → CRUD secrets                   │
│ ├── Key Vault Crypto User → Use keys for crypto operations    │
│ ├── Key Vault Crypto Officer → CRUD keys                      │
│ ├── Key Vault Certificates Officer → CRUD certificates        │
│ └── Key Vault Reader → Read metadata only (no secret values) │
│                                                                       │
│ Example: Give App Service access to read secrets:                  │
│ Role: Key Vault Secrets User                                      │
│ Assign to: App Service managed identity                           │
│                                                                       │
│ Option 2: Vault Access Policy (legacy)                             │
│ Key vault → Access policies → [+ Add Access Policy]            │
│ ├── Secret permissions: Get, List                                │
│ ├── Key permissions: (none)                                      │
│ ├── Certificate permissions: (none)                              │
│ └── Select principal: (user, group, or service principal)      │
│                                                                       │
│ ⚡ Use RBAC for new vaults! More consistent with rest of Azure. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Key Vault in Applications

### App Service + Key Vault (no code!)

```
App Service → Configuration → Application settings

Name: DB_CONNECTION_STRING
Value: @Microsoft.KeyVault(SecretUri=https://kv-myapp-prod.vault.azure.net/secrets/db-connection-string/)

⚡ App Service resolves the Key Vault reference automatically!
⚡ Requires: App Service managed identity + Key Vault Secrets User role
```

### Node.js SDK

```javascript
const { DefaultAzureCredential } = require("@azure/identity");
const { SecretClient } = require("@azure/keyvault-secrets");

const credential = new DefaultAzureCredential();
const client = new SecretClient("https://kv-myapp-prod.vault.azure.net", credential);

async function getSecret() {
  const secret = await client.getSecret("db-connection-string");
  console.log(secret.value); // The secret value
}
```

### .NET SDK

```csharp
// In Program.cs
builder.Configuration.AddAzureKeyVault(
    new Uri("https://kv-myapp-prod.vault.azure.net"),
    new DefaultAzureCredential());

// Access like any config value
var connectionString = builder.Configuration["db-connection-string"];
```

---

## Part 8: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_key_vault" "main" {
  name                        = "kv-myapp-prod"
  resource_group_name         = azurerm_resource_group.main.name
  location                    = azurerm_resource_group.main.location
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  sku_name                    = "standard"
  soft_delete_retention_days  = 90
  purge_protection_enabled    = true
  enable_rbac_authorization   = true
}

resource "azurerm_key_vault_secret" "db_conn" {
  name         = "db-connection-string"
  value        = var.db_connection_string
  key_vault_id = azurerm_key_vault.main.id
}
```

### Bicep

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-myapp-prod'
  location: resourceGroup().location
  properties: {
    tenantId: subscription().tenantId
    sku: { family: 'A', name: 'standard' }
    enableRbacAuthorization: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true
  }
}
```

---

## Part 9: az CLI Reference

```bash
# Create Key Vault
az keyvault create \
  --name kv-myapp-prod \
  --resource-group rg-security \
  --location centralindia \
  --enable-rbac-authorization true \
  --enable-purge-protection true

# ── Secrets ──
# Set a secret
az keyvault secret set \
  --vault-name kv-myapp-prod \
  --name db-connection-string \
  --value "Server=mydb;Password=MyP@ss!"

# Get a secret value
az keyvault secret show \
  --vault-name kv-myapp-prod \
  --name db-connection-string \
  --query value -o tsv

# List secrets
az keyvault secret list --vault-name kv-myapp-prod --output table

# Delete a secret (soft delete)
az keyvault secret delete --vault-name kv-myapp-prod --name db-connection-string

# Recover deleted secret
az keyvault secret recover --vault-name kv-myapp-prod --name db-connection-string

# ── Keys ──
# Create a key
az keyvault key create --vault-name kv-myapp-prod --name encryption-key --kty RSA --size 2048

# List keys
az keyvault key list --vault-name kv-myapp-prod --output table

# ── Certificates ──
# Create self-signed certificate
az keyvault certificate create \
  --vault-name kv-myapp-prod \
  --name myapp-cert \
  --policy "$(az keyvault certificate get-default-policy)"

# Delete Key Vault
az keyvault delete --name kv-myapp-prod --resource-group rg-security
```

---

## Real-World Patterns

### Pattern 1: Centralized Secrets for Microservices

```
┌─────────────────────────────────────────────────┐
│     Centralized Secrets Management              │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Service A│  │ Service B│  │ Service C│      │
│  │ (AKS)   │  │ (App Svc)│  │ (Func)   │      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
│       │              │              │           │
│       │    Managed Identity         │           │
│       └──────────┬──────────────────┘           │
│                  ▼                              │
│         ┌────────────────┐                      │
│         │   Key Vault    │                      │
│         ├────────────────┤                      │
│         │ DB Connection  │                      │
│         │ API Keys       │                      │
│         │ TLS Certs      │                      │
│         │ Encryption Keys│                      │
│         └────────────────┘                      │
│                                                 │
│  Benefits:                                      │
│  - No secrets in code or config files           │
│  - Automatic secret rotation                    │
│  - Audit trail of all access                    │
│  - RBAC controls who can read what              │
└─────────────────────────────────────────────────┘
```

### Pattern 2: TLS Certificate Automation

```
┌─────────────────────────────────────────────────┐
│     Certificate Lifecycle Automation            │
├─────────────────────────────────────────────────┤
│                                                 │
│  Key Vault ──→ Auto-Renew Certificate           │
│       │              │                          │
│       │         Certificate CA                  │
│       │         (DigiCert/GlobalSign)            │
│       ▼                                         │
│  ┌──────────────────────────────┐               │
│  │    Consumers (auto-sync)    │               │
│  ├──────────────────────────────┤               │
│  │ App Gateway  → HTTPS        │               │
│  │ App Service  → Custom Domain│               │
│  │ Front Door   → SSL          │               │
│  │ CDN          → HTTPS        │               │
│  └──────────────────────────────┘               │
│                                                 │
│  Flow: Purchase → Store → Auto-Renew → Deploy   │
│  No manual certificate management needed!       │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Key Vault URL: https://{name}.vault.azure.net

Stores: Secrets (passwords), Keys (encryption), Certificates (SSL)
Tiers: Standard (software keys) | Premium (HSM-protected keys)

Access control: RBAC (recommended) or Vault Access Policies
Key roles: Secrets User (read), Secrets Officer (CRUD), Administrator

Safety: Soft delete (recover deleted items) + Purge protection
Versioning: Every update creates a new version

App integration:
  App Service: @Microsoft.KeyVault(SecretUri=...) in app settings
  Code: DefaultAzureCredential + SecretClient SDK
  Managed Identity: No passwords needed!

Best practice: Store ALL secrets in Key Vault, never in code/config
```

---

## What's Next?

Next chapter: [Chapter 44: Azure RBAC & Custom Roles](44-rbac.md) — Role-based access control, built-in roles, custom roles, and scope levels.
