# Chapter 41: Secrets Manager

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Secrets Manager Fundamentals](#part-1-secrets-manager-fundamentals)
- [Part 2: Storing a Secret (Full Portal Walkthrough)](#part-2-storing-a-secret-full-portal-walkthrough)
- [Part 3: Automatic Rotation](#part-3-automatic-rotation)
- [Part 4: Accessing Secrets in Applications](#part-4-accessing-secrets-in-applications)
- [Part 5: Secrets Manager vs SSM Parameter Store](#part-5-secrets-manager-vs-ssm-parameter-store)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Secrets Manager? Why Not Just Hardcode Passwords?

Every application needs credentials — database passwords, API keys, OAuth tokens. The worst thing you can do is **hardcode them** in your source code:

```python
# ❌ NEVER DO THIS
db_password = "SuperSecret123!"  # Anyone with code access sees this
```

Hardcoded secrets end up in Git repositories, CI/CD logs, and eventually get leaked. **Secrets Manager** is like a **secure vault** for your credentials. Your app asks Secrets Manager for the password at runtime, and Secrets Manager delivers it securely.

**Think of it as:** Instead of writing your ATM PIN on a sticky note on your monitor, you memorize it (or store it in a password manager like 1Password).

**Key benefits:**
- 🔐 Secrets stored encrypted, never in your code
- 🔄 **Automatic rotation**: KMS can rotate database passwords every 30 days without downtime
- 👁️ **Audit trail**: CloudTrail logs every time a secret is accessed
- 🔗 **Integration**: Lambda, ECS, EC2 all fetch secrets automatically

AWS Secrets Manager stores, retrieves, and automatically rotates secrets like database credentials, API keys, and tokens. It eliminates hardcoded secrets in your application code.

```
What you'll learn:
├── Secrets Manager Fundamentals
├── Storing a secret (full portal walkthrough)
├── Automatic rotation (Lambda-based)
├── Accessing secrets in applications (SDK, Lambda, ECS)
├── Secrets Manager vs SSM Parameter Store
└── Terraform & CLI examples
```

---

## Part 1: Secrets Manager Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW SECRETS MANAGER WORKS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐      │
│ │ Application  │───►│ Secrets Manager  │───►│ KMS          │      │
│ │              │    │                  │    │ (encryption)  │      │
│ │ GetSecret    │◄───│ Secret Value     │    └──────────────┘      │
│ │ Value()      │    │ (encrypted)      │                          │
│ └──────────────┘    └──────────────────┘                          │
│                           │                                        │
│                     ┌─────┴──────┐                                 │
│                     │ Lambda     │ (auto-rotation)                 │
│                     │ (rotates   │                                 │
│                     │  secret)   │                                 │
│                     └────────────┘                                 │
│                                                                       │
│ Key features:                                                        │
│ ├── Encrypted at rest with KMS (customer or AWS managed key)   │
│ ├── Automatic rotation with Lambda                              │
│ ├── Versioning (AWSCURRENT, AWSPREVIOUS, AWSPENDING)           │
│ ├── Cross-account access via resource policy                   │
│ ├── Cross-region replication                                    │
│ ├── Fine-grained IAM access control                            │
│ └── Audit trail via CloudTrail                                  │
│                                                                       │
│ Pricing:                                                             │
│ ├── $0.40/secret/month                                           │
│ ├── $0.05/10,000 API calls                                      │
│ └── No free tier                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Storing a Secret (Full Portal Walkthrough)

```
Console → Secrets Manager → Store a new secret

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: CHOOSE SECRET TYPE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Secret type:                                                    │
│ ● Credentials for Amazon RDS database                         │
│ ○ Credentials for Amazon Redshift cluster                     │
│ ○ Credentials for Amazon DocumentDB database                  │
│ ○ Credentials for other database                              │
│ ○ Other type of secret (API key, OAuth token, etc.)          │
│                                                                   │
│ ── For RDS credentials ──                                      │
│ Username: [admin]                                              │
│ Password: [****] or [Generate]                                │
│                                                                   │
│ Database: [prod-mysql-db ▼]                                   │
│ → Auto-discovers RDS instances in account                    │
│ → Enables managed rotation                                    │
│                                                                   │
│ ── For "Other type" ──                                         │
│ ● Key/value pairs                                              │
│   Key: [api_key]     Value: [sk-123abc...]                    │
│   Key: [api_secret]  Value: [****]                            │
│ ○ Plaintext                                                    │
│   { "api_key": "sk-123abc", "api_secret": "xyz" }            │
│                                                                   │
│ Encryption key:                                                │
│ ● aws/secretsmanager (AWS managed, default)                   │
│ ○ Customer managed key: [alias/prod-secrets ▼]               │
│ → ⚡ Use customer managed key for cross-account access       │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: CONFIGURE SECRET                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Secret name: [prod/myapp/database]                            │
│ → Use path-style naming: environment/app/purpose             │
│ → IAM policies can use wildcards: prod/*                     │
│                                                                   │
│ Description: [Production database credentials]                │
│                                                                   │
│ Tags:                                                           │
│ Key: [environment]  Value: [production]                       │
│ Key: [application]  Value: [my-app]                           │
│                                                                   │
│ Resource permissions (resource policy):                        │
│ [Edit permissions] → JSON policy for cross-account access    │
│                                                                   │
│ Replicate secret:                                              │
│ ☑ Replicate to other regions                                  │
│ Region: [EU (Ireland) eu-west-1 ▼]                           │
│ Encryption key: [aws/secretsmanager ▼]                       │
│ → ⚡ Replicate for DR and cross-region applications          │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: CONFIGURE ROTATION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Automatic rotation: ☑ Enable                                  │
│                                                                   │
│ Rotation schedule:                                             │
│ ● Schedule expression: rate(30 days)                          │
│ ○ Cron expression: cron(0 8 1 * ? *)                         │
│                                                                   │
│ Window duration: [3h] (rotation completes within this window) │
│                                                                   │
│ Rotation function:                                             │
│ ● Create a new Lambda function (⚡ for RDS)                   │
│ ○ Use an existing Lambda function                             │
│                                                                   │
│ → For RDS: Secrets Manager auto-creates Lambda that:         │
│   1. Creates new password                                     │
│   2. Updates password in RDS                                  │
│   3. Tests new credentials                                    │
│   4. Marks new version as AWSCURRENT                         │
│                                                                   │
│ Rotation strategy (for RDS):                                  │
│ ● Single user rotation                                        │
│   → Changes password of the same user                        │
│   → Brief connection interruption during rotation            │
│ ○ Alternating users rotation                                  │
│   → Creates 2 users, alternates between them                 │
│   → Zero downtime (old creds work until swap)               │
│   → ⚡ Recommended for production                             │
│                                                                   │
│                    [Next → Review → Store]                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Automatic Rotation

```
┌─────────────────────────────────────────────────────────────────────┐
│           ROTATION FLOW                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Lambda rotation function executes 4 steps:                         │
│                                                                       │
│ 1. createSecret:                                                    │
│    → Generate new credentials                                     │
│    → Store as AWSPENDING version                                  │
│                                                                       │
│ 2. setSecret:                                                       │
│    → Update the resource (RDS, etc.) with new credentials        │
│    → Change password in database                                  │
│                                                                       │
│ 3. testSecret:                                                      │
│    → Verify new credentials work                                  │
│    → Connect to database with AWSPENDING credentials             │
│                                                                       │
│ 4. finishSecret:                                                    │
│    → Move AWSCURRENT label to new version                        │
│    → Move old version to AWSPREVIOUS                              │
│    → Old credentials still work briefly (graceful transition)    │
│                                                                       │
│ Secret versions:                                                     │
│ ├── AWSCURRENT: Active credentials (apps use this)              │
│ ├── AWSPENDING: New credentials being tested                     │
│ └── AWSPREVIOUS: Previous working credentials (rollback)        │
│                                                                       │
│ ⚡ Supported auto-rotation targets:                                 │
│ ├── Amazon RDS (all engines)                                     │
│ ├── Amazon Aurora                                                 │
│ ├── Amazon Redshift                                               │
│ ├── Amazon DocumentDB                                             │
│ └── Custom (write your own Lambda function)                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Accessing Secrets in Applications

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACCESSING SECRETS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Python (boto3):                                                      │
│ import boto3, json                                                  │
│ client = boto3.client('secretsmanager')                            │
│ response = client.get_secret_value(SecretId='prod/myapp/database')│
│ secret = json.loads(response['SecretString'])                      │
│ db_host = secret['host']                                           │
│ db_password = secret['password']                                   │
│                                                                       │
│ Lambda environment:                                                  │
│ → Use AWS Parameters and Secrets Lambda Extension (layer)        │
│ → Caches secrets, reduces API calls                               │
│ → Fetch via localhost HTTP: GET http://localhost:2773/             │
│   secretsmanager/get?secretId=prod/myapp/database                │
│                                                                       │
│ ECS task definition:                                                │
│ "secrets": [                                                        │
│   {                                                                  │
│     "name": "DB_PASSWORD",                                         │
│     "valueFrom": "arn:aws:secretsmanager:us-east-1:123456:       │
│                   secret:prod/myapp/database:password::"           │
│   }                                                                  │
│ ]                                                                    │
│ → Injected as environment variable at task startup               │
│ → Format: ARN:json-key:version-stage:version-id                 │
│                                                                       │
│ CodeBuild buildspec.yml:                                            │
│ env:                                                                 │
│   secrets-manager:                                                  │
│     DB_PASSWORD: "prod/myapp/database:password"                   │
│                                                                       │
│ CloudFormation:                                                      │
│ {{resolve:secretsmanager:prod/myapp/database:SecretString:pass}} │
│                                                                       │
│ ⚡ Never log or print secret values                                 │
│ ⚡ Use caching (SDK caching, Lambda extension) to reduce costs    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Secrets Manager vs SSM Parameter Store

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECRETS MANAGER vs PARAMETER STORE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────────┬──────────────────┬────────────────────────┐│
│ │ Feature            │ Secrets Manager  │ SSM Parameter Store    ││
│ ├────────────────────┼──────────────────┼────────────────────────┤│
│ │ Auto rotation      │ ✅ Built-in      │ ❌ Manual only         ││
│ │ Cross-region       │ ✅ Replication   │ ❌ No replication      ││
│ │ Cross-account      │ ✅ Resource policy│ ❌ No (use IAM only)  ││
│ │ Pricing            │ $0.40/secret/mo  │ Free (standard)       ││
│ │                    │                  │ $0.05/advanced/mo     ││
│ │ Encryption         │ Always encrypted │ SecureString (opt)    ││
│ │ Versioning         │ ✅ Auto          │ Limited               ││
│ │ Max size           │ 64 KB            │ 4 KB std, 8 KB adv   ││
│ │ Hierarchy          │ Name-based       │ Path hierarchy (/a/b) ││
│ │ CloudFormation     │ Dynamic ref      │ Dynamic ref           ││
│ │ ECS integration    │ ✅ Native        │ ✅ Native             ││
│ └────────────────────┴──────────────────┴────────────────────────┘│
│                                                                       │
│ Decision:                                                            │
│ ├── DB passwords, API keys → Secrets Manager (rotation!)        │
│ ├── Config values, feature flags → Parameter Store (free!)      │
│ ├── Need rotation → Secrets Manager                              │
│ ├── Need cross-region → Secrets Manager                         │
│ ├── Need hierarchy + free → Parameter Store                     │
│ └── ⚡ Use both: SM for secrets, PS for configuration           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & CLI Examples

```hcl
resource "aws_secretsmanager_secret" "db" {
  name                    = "prod/myapp/database"
  description             = "Production database credentials"
  kms_key_id              = aws_kms_key.secrets.arn
  recovery_window_in_days = 30

  replica {
    region     = "eu-west-1"
    kms_key_id = aws_kms_key.secrets_eu.arn
  }

  tags = { Environment = "prod" }
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    username = "admin"
    password = random_password.db.result
    host     = aws_db_instance.main.address
    port     = 3306
    dbname   = "myapp"
  })
}

resource "aws_secretsmanager_secret_rotation" "db" {
  secret_id           = aws_secretsmanager_secret.db.id
  rotation_lambda_arn = aws_lambda_function.rotation.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

```bash
# Create secret
aws secretsmanager create-secret \
  --name prod/myapp/database \
  --secret-string '{"username":"admin","password":"MyP@ssword123"}'

# Get secret value
aws secretsmanager get-secret-value \
  --secret-id prod/myapp/database

# Update secret
aws secretsmanager update-secret \
  --secret-id prod/myapp/database \
  --secret-string '{"username":"admin","password":"NewP@ssword456"}'

# Rotate immediately
aws secretsmanager rotate-secret \
  --secret-id prod/myapp/database

# Delete secret (with recovery window)
aws secretsmanager delete-secret \
  --secret-id prod/myapp/database \
  --recovery-window-in-days 30

# Restore deleted secret
aws secretsmanager restore-secret \
  --secret-id prod/myapp/database
```

---

## Troubleshooting: Common Secrets Manager Issues

### "Rotation failed" (Lambda rotation function error)

```
Debugging:
1. ☐ Check CloudWatch Logs for the rotation Lambda:
   Log group: /aws/lambda/SecretsManager-ROTATION_FUNCTION

2. ☐ Can the Lambda reach the database?
   If in a VPC: Lambda needs NAT Gateway or VPC endpoints
   For Secrets Manager: VPC endpoint com.amazonaws.REGION.secretsmanager

3. ☐ Does the Lambda have correct IAM permissions?
   Needs: secretsmanager:GetSecretValue, secretsmanager:UpdateSecret,
   secretsmanager:PutSecretValue, plus database connection permissions

4. ☐ Is the database accepting the new password?
   Check password policy (length, special characters)
```

### "My app is using an old/stale secret value"

```
Causes:
1. App caches secret at startup and never refreshes
   Fix: Use the AWS SDK caching client with TTL (e.g., 1 hour)

2. After rotation, there are two versions:
   AWSCURRENT = new secret (apps should use this)
   AWSPREVIOUS = old secret (kept for rollback)
   Ensure your app requests AWSCURRENT (default if not specified)

3. Lambda/container not restarted after rotation
   Fix: Use environment variable refresh or SDK caching client
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Hardcoding secrets in code | Secrets leak via Git | Always use Secrets Manager or env vars |
| Not testing rotation | First rotation breaks the app | Test rotation in dev environment first |
| Rotation Lambda can't reach DB | Rotation fails silently | Check VPC, security groups, and endpoints |
| No caching in application | Too many API calls, throttled | Use SDK caching client (cache for 1 hour) |
| Missing cross-account policy | Other accounts can't access | Add resource policy on the secret |

---

## Quick Reference

```
Secrets Manager Quick Reference:
├── What: Managed secret storage with auto-rotation
├── Secrets: DB passwords, API keys, tokens, certificates
├── Encryption: KMS (always encrypted at rest)
├── Rotation: Lambda-based, 4-step process
├── Versions: AWSCURRENT, AWSPENDING, AWSPREVIOUS
├── Cross-region: Built-in replication
├── Cross-account: Resource-based policy
├── Access: SDK, Lambda extension, ECS secrets, buildspec
├── Pricing: $0.40/secret/month + $0.05/10K API calls
├── vs Parameter Store: SM for rotation, PS for free config
├── ⚡ Always enable rotation for database credentials
└── ⚡ Use path naming: env/app/purpose
```

---

## What's Next?

In **Chapter 42: WAF & Shield**, we'll cover web application firewall rules, DDoS protection, and bot mitigation for your applications.
