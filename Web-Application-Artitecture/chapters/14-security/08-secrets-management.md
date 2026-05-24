# Secrets Management (Vault, AWS Secrets Manager)

> **What you'll learn**: How to securely store, access, rotate, and audit sensitive credentials like database passwords, API keys, and encryption keys — without ever hardcoding them in your code or config files.

---

## Real-Life Analogy

Imagine you run a company with 100 employees. Each one needs keys to different rooms.

**Bad approach (no secrets management):**
- You write all the door codes on sticky notes and tape them to everyone's desk.
- If someone leaves the company, you have no idea which codes they know.
- If a code needs to change, you have to find and replace every sticky note.

**Good approach (secrets management = a key cabinet):**
- All codes are in a **locked cabinet** with a guard (Vault).
- Employees **request** a code, and the guard checks if they're authorized.
- The guard **logs** every access — you know exactly who used what, when.
- You can **rotate** a code in ONE place — everyone automatically gets the new one.
- If someone leaves, you just **revoke their access** to the cabinet.

```
WITHOUT Secrets Management:              WITH Secrets Management:
┌──────────────────────────┐            ┌──────────────────────────┐
│                          │            │                          │
│  config.py:              │            │  config.py:              │
│  DB_PASS = "hunter2"     │            │  DB_PASS = vault.get(    │
│  API_KEY = "sk_live_..." │            │    "database/password")  │
│  AWS_SECRET = "AKIA..."  │            │                          │
│                          │            │  ┌─────────┐             │
│  Committed to Git! 😱    │            │  │  VAULT  │ → encrypted │
│  Everyone can see! 😱    │            │  │         │ → audited   │
│  Hard to rotate! 😱      │            │  │         │ → rotatable │
│                          │            │  └─────────┘             │
└──────────────────────────┘            └──────────────────────────┘
```

---

## Core Concept Explained Step-by-Step

### Step 1: What Are Secrets?

**Secrets** are any sensitive data that your application needs but shouldn't be visible to humans or stored in code:

```
┌─────────────────────────────────────────────────────────────┐
│                    TYPES OF SECRETS                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Database credentials    │ postgres://user:PASSWORD@host/db  │
│  API keys               │ sk_live_4eC39HqLyjWDarjtT1zdp7dc │
│  Encryption keys        │ AES-256 symmetric keys             │
│  TLS certificates       │ Private key material               │
│  OAuth client secrets   │ Client secret for OAuth flows      │
│  SSH keys               │ Private keys for server access     │
│  Cloud credentials      │ AWS_SECRET_ACCESS_KEY              │
│  Webhook secrets        │ HMAC signing keys                  │
│  JWT signing keys       │ Keys used to sign tokens           │
│  Service account tokens │ Machine-to-machine auth            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Why You Can't Just Use Environment Variables

Environment variables are better than hardcoding, but still have problems:

```
┌─────────────────────────────────────────────────────────────┐
│         PROBLEMS WITH ENVIRONMENT VARIABLES                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. No access control — any process can read all env vars    │
│  2. No audit trail — who accessed what, when?                │
│  3. No rotation — changing requires redeployment             │
│  4. Visible in process listings (ps aux, /proc/<pid>/environ)│
│  5. Leaked in error logs and crash dumps                     │
│  6. No encryption at rest                                    │
│  7. Shared across all containers in a pod (Kubernetes)       │
│                                                              │
│  Environment variables are OK for:                           │
│  ✓ Non-sensitive config (port numbers, log levels)           │
│  ✓ Referencing WHERE to find secrets (vault address)         │
│  ✗ NOT for the secrets themselves (in production)            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Step 3: How Secrets Management Works

```
┌────────────────────────────────────────────────────────────────┐
│                SECRET MANAGEMENT ARCHITECTURE                    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐      ┌──────────────────┐      ┌─────────────┐  │
│  │ App Pod  │─────▶│  Secrets Manager  │─────▶│  Encrypted  │  │
│  │          │      │  (Vault/AWS SM)   │      │  Storage    │  │
│  │ 1. Auth  │      │                   │      │  (Backend)  │  │
│  │ 2. Request│      │  - Authenticates  │      │             │  │
│  │ 3. Use   │      │  - Authorizes     │      │ AES-256-GCM │  │
│  │ 4. Forget│      │  - Audits         │      │  encrypted  │  │
│  └──────────┘      │  - Rate limits    │      └─────────────┘  │
│                    │  - Rotates        │                        │
│                    └──────────────────┘                        │
│                                                                 │
│  Flow:                                                          │
│  1. App authenticates to Vault (using token, IAM role, K8s SA) │
│  2. App requests secret: "Give me the database password"        │
│  3. Vault checks policy: "Is this app allowed to read this?"    │
│  4. If allowed → return secret + log the access                 │
│  5. App uses secret, NEVER stores it on disk                    │
│  6. Secret can have TTL — app must re-fetch after expiry        │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Step 4: Secret Rotation

```
┌────────────────────────────────────────────────────────────────┐
│                    SECRET ROTATION FLOW                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Manual Rotation (bad):                                         │
│  1. Generate new password                                       │
│  2. Update database                                             │
│  3. Update all applications                                     │
│  4. Restart everything                                          │
│  5. Hope nothing breaks                                         │
│  Time: Hours. Risk: High.                                       │
│                                                                 │
│  Automatic Rotation (good):                                     │
│  ┌──────┐      ┌───────┐      ┌────────┐                      │
│  │Vault │─────▶│ DB    │─────▶│ Apps   │                      │
│  │      │      │       │      │        │                      │
│  │ 1. Generate new password                                    │
│  │ 2. Update DB credentials                                    │
│  │ 3. Apps automatically get new password on next request      │
│  │ 4. Old password expires after grace period                  │
│  └──────┘      └───────┘      └────────┘                      │
│  Time: Seconds. Risk: Zero downtime.                            │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### HashiCorp Vault Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    VAULT ARCHITECTURE                            │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐       │
│  │                    API Layer                          │       │
│  │  (HTTP/gRPC — all interactions via API)              │       │
│  └─────────────────────────────────────────────────────┘       │
│                          │                                      │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────────┐       │
│  │ Auth     │  │ Secret       │  │ Audit              │       │
│  │ Methods  │  │ Engines      │  │ Devices            │       │
│  │          │  │              │  │                    │       │
│  │ - Token  │  │ - KV (v2)   │  │ - File             │       │
│  │ - AWS IAM│  │ - Database  │  │ - Syslog           │       │
│  │ - K8s SA │  │ - PKI       │  │ - Socket           │       │
│  │ - AppRole│  │ - Transit   │  │                    │       │
│  │ - LDAP   │  │ - SSH       │  │ Every operation    │       │
│  └──────────┘  └──────────────┘  │ is logged!        │       │
│                          │        └────────────────────┘       │
│  ┌─────────────────────────────────────────────────────┐       │
│  │              Barrier (Encryption Layer)               │       │
│  │         AES-256-GCM — everything encrypted           │       │
│  └─────────────────────────────────────────────────────┘       │
│                          │                                      │
│  ┌─────────────────────────────────────────────────────┐       │
│  │              Storage Backend                          │       │
│  │    Consul / DynamoDB / Integrated Raft Storage       │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  Seal/Unseal:                                                   │
│  - Vault starts "sealed" — encrypted, can't serve requests     │
│  - Must be "unsealed" with master key (Shamir's Secret Sharing)│
│  - Auto-unseal with cloud KMS (AWS/GCP/Azure KMS)              │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Dynamic Secrets — Short-Lived Credentials

```
Traditional (Static):                    Vault (Dynamic):
┌────────────────────┐                  ┌────────────────────┐
│ App connects with  │                  │ App asks Vault for │
│ shared password    │                  │ database access    │
│ "production_pass"  │                  │                    │
│                    │                  │ Vault creates:     │
│ • Never rotated   │                  │ username: v-app-xyz│
│ • Everyone knows it│                  │ password: random32 │
│ • Hard to revoke  │                  │ TTL: 1 hour        │
│                    │                  │                    │
│                    │                  │ After 1 hour:      │
│                    │                  │ Vault DELETES the  │
│                    │                  │ user from database!│
└────────────────────┘                  └────────────────────┘

Compromise of dynamic secret? → It expires in 1 hour anyway!
```

---

## Code Examples

### Python — Using HashiCorp Vault

```python
# vault_client.py — Fetch secrets from HashiCorp Vault
import hvac  # pip install hvac

class VaultClient:
    """Secure secret access via HashiCorp Vault"""
    
    def __init__(self):
        self.client = hvac.Client(
            url='https://vault.internal:8200',
            # Auth using Kubernetes service account (in K8s)
            # Or AppRole for non-K8s environments
        )
        # Authenticate using Kubernetes auth method
        self.client.auth.kubernetes.login(
            role='my-app-role',
            jwt=open('/var/run/secrets/kubernetes.io/serviceaccount/token').read()
        )
    
    def get_secret(self, path: str, key: str) -> str:
        """Fetch a secret from Vault KV v2 engine"""
        response = self.client.secrets.kv.v2.read_secret_version(
            path=path,
            mount_point='secret'
        )
        return response['data']['data'][key]
    
    def get_database_credentials(self) -> dict:
        """Get dynamic database credentials (auto-expire!)"""
        response = self.client.secrets.database.generate_credentials(
            name='my-app-db-role'
        )
        return {
            'username': response['data']['username'],
            'password': response['data']['password'],
            'ttl': response['lease_duration'],  # Seconds until expiry
            'lease_id': response['lease_id'],   # For renewal/revocation
        }

# Usage in application
vault = VaultClient()

# Static secret (API key)
stripe_key = vault.get_secret("payments/stripe", "api_key")

# Dynamic secret (database credentials that auto-expire)
db_creds = vault.get_database_credentials()
connection_string = f"postgresql://{db_creds['username']}:{db_creds['password']}@db:5432/myapp"
```

### Python — Using AWS Secrets Manager

```python
# aws_secrets.py — Fetch secrets from AWS Secrets Manager
import boto3
import json
from functools import lru_cache
from datetime import datetime, timedelta

class AWSSecretsClient:
    """Access secrets from AWS Secrets Manager with caching"""
    
    def __init__(self, region: str = 'us-east-1'):
        self.client = boto3.client('secretsmanager', region_name=region)
        self._cache = {}
        self._cache_ttl = timedelta(minutes=5)
    
    def get_secret(self, secret_name: str) -> dict:
        """Fetch secret with local caching to reduce API calls"""
        # Check cache first
        cached = self._cache.get(secret_name)
        if cached and cached['expires_at'] > datetime.utcnow():
            return cached['value']
        
        # Fetch from AWS
        response = self.client.get_secret_value(SecretId=secret_name)
        secret_value = json.loads(response['SecretString'])
        
        # Cache it
        self._cache[secret_name] = {
            'value': secret_value,
            'expires_at': datetime.utcnow() + self._cache_ttl
        }
        
        return secret_value

# Usage
secrets = AWSSecretsClient()

# Get database credentials
db_secret = secrets.get_secret("prod/myapp/database")
# Returns: {"username": "admin", "password": "...", "host": "..."}

# Get API keys
api_keys = secrets.get_secret("prod/myapp/api-keys")
stripe_key = api_keys["stripe_secret_key"]
```

### Java — Using Spring Cloud Vault

```java
// application.yml — Spring Boot + Vault integration
// spring:
//   cloud:
//     vault:
//       uri: https://vault.internal:8200
//       authentication: KUBERNETES
//       kubernetes:
//         role: my-app-role
//         service-account-token-file: /var/run/secrets/.../token
//       kv:
//         enabled: true
//         backend: secret
//         default-context: myapp

// DatabaseConfig.java — Auto-injected from Vault
@Configuration
public class DatabaseConfig {
    
    // These values are automatically fetched from Vault!
    @Value("${database.username}")
    private String username;
    
    @Value("${database.password}")
    private String password;
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://db:5432/myapp");
        config.setUsername(username);  // From Vault
        config.setPassword(password);  // From Vault
        return new HikariDataSource(config);
    }
}
```

### Java — Direct Vault API Usage

```java
import org.springframework.vault.core.VaultTemplate;
import org.springframework.vault.support.VaultResponse;

@Service
public class SecretService {
    
    private final VaultTemplate vaultTemplate;
    
    public SecretService(VaultTemplate vaultTemplate) {
        this.vaultTemplate = vaultTemplate;
    }
    
    public String getApiKey(String service) {
        VaultResponse response = vaultTemplate.read("secret/data/api-keys/" + service);
        Map<String, Object> data = (Map<String, Object>) response.getData().get("data");
        return (String) data.get("api_key");
    }
    
    public DatabaseCredentials getDynamicDbCreds() {
        VaultResponse response = vaultTemplate.read("database/creds/my-app-role");
        return new DatabaseCredentials(
            (String) response.getData().get("username"),
            (String) response.getData().get("password"),
            (Integer) response.getData().get("ttl")
        );
    }
}
```

---

## Infrastructure Examples

### HashiCorp Vault — Docker Setup

```yaml
# docker-compose.yml — Vault for development
version: '3.8'
services:
  vault:
    image: hashicorp/vault:latest
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: "dev-root-token"
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    ports:
      - "8200:8200"
    volumes:
      - vault-data:/vault/data
      
volumes:
  vault-data:
```

### Vault Configuration for Production

```hcl
# vault-policy.hcl — Define who can access what secrets
# App can read database secrets, nothing else
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

path "database/creds/myapp-role" {
  capabilities = ["read"]
}

# Admin can manage secrets
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

```bash
# Enable database secret engine with auto-rotation
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/mydb \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@db:5432/myapp" \
    allowed_roles="myapp-role" \
    username="vault_admin" \
    password="vault_admin_pass"

# Create role that generates dynamic credentials
vault write database/roles/myapp-role \
    db_name=mydb \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

### Kubernetes — External Secrets Operator

```yaml
# Sync secrets from Vault/AWS to Kubernetes
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1h  # Re-sync every hour
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: myapp-secrets  # K8s Secret name
    creationPolicy: Owner
  data:
    - secretKey: db-password
      remoteRef:
        key: secret/data/myapp/database
        property: password
    - secretKey: api-key
      remoteRef:
        key: secret/data/myapp/api-keys
        property: stripe_key
---
# Use in deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          envFrom:
            - secretRef:
                name: myapp-secrets
```

---

## Real-World Example

### How Netflix Manages Secrets at Scale

```
┌────────────────────────────────────────────────────────────────┐
│              NETFLIX SECRET MANAGEMENT                           │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Tool: "Titus" (their container platform) + custom secrets      │
│                                                                 │
│  Architecture:                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │  App (Titus  │────▶│  Secret      │────▶│  Backend     │   │
│  │  Container)  │     │  Proxy       │     │  (KMS +      │   │
│  └──────────────┘     │  (Sidecar)   │     │  DynamoDB)   │   │
│                       └──────────────┘     └──────────────┘   │
│                                                                 │
│  Key principles:                                                │
│  • Every service has unique credentials                         │
│  • Secrets are short-lived (hours, not months)                  │
│  • All access is audited and alerted on                        │
│  • Rotation is fully automated                                  │
│  • Compromised secrets are revoked in seconds                   │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### How AWS Manages Internal Secrets

- Uses **AWS KMS** (Key Management Service) as the root of trust
- Envelope encryption: Data keys encrypted by master keys
- Master keys stored in **Hardware Security Modules (HSMs)**
- Keys can never be exported from HSMs
- All access logged to CloudTrail

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| Secrets in Git (even in history!) | Permanent exposure | Use git-secrets, pre-commit hooks |
| Secrets in Docker images | Anyone with image access reads them | Inject at runtime, not build |
| `.env` files in production | Visible, no rotation | Use proper secrets manager |
| Shared credentials across environments | Dev breach = prod breach | Separate secrets per env |
| No secret rotation | Compromised secrets work forever | Auto-rotate with Vault/AWS |
| Logging secrets | Appear in log aggregation systems | Redact before logging |
| Long-lived tokens | Extended exposure window | Short TTL + refresh |
| Secrets in ConfigMaps (K8s) | Base64 is NOT encryption | Use K8s Secrets + external-secrets |

### What To Do If a Secret Is Leaked

```
INCIDENT RESPONSE:
┌──────────────────────────────────────────────────────────────┐
│  1. IMMEDIATELY rotate the compromised secret                 │
│  2. Revoke all sessions/tokens using the old secret          │
│  3. Check audit logs: was the secret used maliciously?       │
│  4. If committed to Git: the secret is PERMANENTLY exposed!  │
│     → Rotate even if you force-push to remove the commit     │
│     → Assume it was scraped by bots within minutes            │
│  5. Document the incident and fix the root cause             │
└──────────────────────────────────────────────────────────────┘
```

---

## When to Use / When NOT to Use

### Secrets Manager Selection:

| Scenario | Tool |
|----------|------|
| AWS-native, simple needs | AWS Secrets Manager |
| Multi-cloud / on-premise | HashiCorp Vault |
| Kubernetes-native | Sealed Secrets + External Secrets |
| Azure ecosystem | Azure Key Vault |
| GCP ecosystem | Google Secret Manager |
| Small team, just starting | Environment variables (temporary!) |
| Enterprise with compliance | Vault + HSM integration |

### When You MUST Use a Secrets Manager:

- Production environments (always)
- When multiple services share credentials
- When you need audit trails (compliance)
- When secrets must rotate automatically
- When you have multiple environments (dev/staging/prod)

### When Environment Variables Are Acceptable:

- Local development only
- Single-developer side projects
- CI/CD secrets (GitHub Secrets, GitLab CI Variables) — as a bridge
- Combined with container runtime secrets injection

---

## Key Takeaways

- **Never hardcode secrets** in source code, config files, or Docker images
- **Use a dedicated secrets manager** (Vault, AWS Secrets Manager) in production
- **Dynamic secrets** (short-lived, auto-generated) are more secure than static passwords
- **Automate rotation** — secrets should change regularly without human intervention
- **Audit all access** — know WHO accessed WHAT secret, WHEN
- **Principle of least privilege** — each service gets access only to secrets it needs
- If a secret is ever exposed (even briefly), treat it as **compromised and rotate immediately**

---

## What's Next?

Next, we'll explore **DDoS Protection & WAF** — how to protect your application from distributed denial-of-service attacks and common web exploits using Web Application Firewalls. That's Chapter 14.9.
