# Chapter 40: KMS (Key Management Service)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: KMS Fundamentals](#part-1-kms-fundamentals)
- [Part 2: Creating a KMS Key (Full Portal Walkthrough)](#part-2-creating-a-kms-key-full-portal-walkthrough)
- [Part 3: Envelope Encryption](#part-3-envelope-encryption)
- [Part 4: Key Policies & Grants](#part-4-key-policies--grants)
- [Part 5: Key Rotation & Multi-Region Keys](#part-5-key-rotation--multi-region-keys)
- [Part 6: KMS with AWS Services](#part-6-kms-with-aws-services)
- [Part 7: Terraform & CLI Examples](#part-7-terraform--cli-examples)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Encryption? Why Do We Need KMS?

**Encryption** is like putting your data in a locked safe. Even if someone steals the safe, they can't read the data without the key. In AWS, your data is stored on shared infrastructure — encryption ensures that only **you** (and services you authorize) can read it.

**KMS (Key Management Service)** manages the "keys" to your safes. Instead of you worrying about where to store encryption keys securely (a chicken-and-egg problem!), KMS stores and protects them for you in tamper-resistant hardware.

**Simple examples:**
- 🗄️ Your S3 bucket stores customer data → KMS encrypts every object so even AWS engineers can't read it
- 💾 Your RDS database stores passwords → KMS encrypts the storage so a stolen disk is useless
- 📝 Your app encrypts sensitive fields before storing them → KMS provides the key

**Key concept for beginners:** Most of the time, you just select "Enable encryption" and pick a KMS key — AWS handles the rest. You don't need to understand cryptography to use KMS.

AWS KMS is a managed service for creating and controlling encryption keys used to protect your data. It integrates with nearly every AWS service that supports encryption.

```
What you'll learn:
├── KMS Fundamentals (key types, hierarchy)
├── Creating a KMS key (full portal walkthrough)
├── Envelope encryption (data keys)
├── Key policies & grants (access control)
├── Key rotation & multi-region keys
├── KMS integration with AWS services
└── Terraform & CLI examples
```

---

## Part 1: KMS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           KMS KEY TYPES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Key ownership:                                                       │
│ ┌─────────────────┬─────────────────┬──────────────────────────┐  │
│ │ Type            │ Managed by      │ Use case                 │  │
│ ├─────────────────┼─────────────────┼──────────────────────────┤  │
│ │ AWS owned keys  │ AWS (hidden)    │ Default encryption       │  │
│ │                 │                 │ (S3-SSE, DynamoDB)       │  │
│ │ AWS managed     │ AWS (visible,   │ Service-default with     │  │
│ │ keys            │ auto-rotate)    │ audit trail (aws/s3)     │  │
│ │ Customer        │ You (full       │ Full control: policy,    │  │
│ │ managed keys    │ control)        │ rotation, cross-account  │  │
│ └─────────────────┴─────────────────┴──────────────────────────┘  │
│                                                                       │
│ Key spec (algorithms):                                              │
│ ├── SYMMETRIC_DEFAULT: AES-256-GCM (⚡ most common)             │
│ │   → Single key for encrypt + decrypt                          │
│ │   → Used by AWS services automatically                       │
│ ├── RSA_2048/3072/4096: Asymmetric (public/private pair)       │
│ │   → Encrypt/decrypt OR sign/verify                           │
│ │   → Public key downloadable (external use)                   │
│ └── ECC: Elliptic curve (sign/verify only)                     │
│     → NIST P-256, P-384, P-521, secp256k1                    │
│                                                                       │
│ Key material origin:                                                │
│ ├── KMS (default): AWS generates key material in HSMs          │
│ ├── External: You import key material (BYOK)                   │
│ ├── Custom key store: CloudHSM cluster backing                 │
│ └── External key store: Keys outside AWS (XKS)                 │
│                                                                       │
│ Pricing:                                                             │
│ ├── Customer managed key: $1/month/key                          │
│ ├── AWS managed key: Free                                        │
│ ├── API calls: $0.03/10,000 requests                           │
│ └── Free tier: 20,000 requests/month                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a KMS Key (Full Portal Walkthrough)

```
Console → KMS → Customer managed keys → Create key

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: CONFIGURE KEY                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Key type:                                                       │
│ ● Symmetric  ○ Asymmetric                                     │
│ → ⚡ Symmetric for AWS service encryption                     │
│ → Asymmetric for external apps, code signing                 │
│                                                                   │
│ Key usage:                                                      │
│ ● Encrypt and decrypt                                          │
│ ○ Sign and verify                                              │
│ ○ Generate and verify MAC                                     │
│                                                                   │
│ Advanced options:                                               │
│ Key material origin:                                           │
│ ● KMS (⚡ recommended)                                        │
│ ○ External (import your own key material)                     │
│ ○ Custom key store (CloudHSM)                                 │
│ ○ External key store (XKS)                                    │
│                                                                   │
│ Regionality:                                                    │
│ ● Single-Region key                                            │
│ ○ Multi-Region key                                             │
│ → Multi-Region: Replicated to other regions                  │
│ → Use for: Cross-region encryption, disaster recovery        │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: ADD LABELS                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Alias: [alias/prod-data-key]                                  │
│ → Human-readable name (alias/ prefix auto-added)             │
│ → Can point alias to different key (rotation)                │
│                                                                   │
│ Description: [Production data encryption key]                 │
│                                                                   │
│ Tags:                                                           │
│ Key: [environment]  Value: [production]                       │
│ Key: [team]  Value: [security]                                │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: KEY ADMINISTRATORS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Key administrators (manage key, NOT use it):                   │
│ ☑ arn:aws:iam::123456:role/SecurityAdmin                     │
│ ☑ arn:aws:iam::123456:user/key-admin                         │
│                                                                   │
│ → Can: Enable/disable, set policy, schedule deletion          │
│ → Cannot: Use key to encrypt/decrypt data                    │
│ → ⚡ Principle of least privilege: Admin ≠ User               │
│                                                                   │
│ ☐ Allow key administrators to delete this key                 │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 4: KEY USERS                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Key users (can use key to encrypt/decrypt):                    │
│ ☑ arn:aws:iam::123456:role/AppServerRole                     │
│ ☑ arn:aws:iam::123456:role/LambdaExecutionRole               │
│                                                                   │
│ Other AWS accounts that can use this key:                     │
│ [111222333444] [Add]                                          │
│ → Cross-account access (target account also needs IAM perm)  │
│                                                                   │
│                    [Next → Review → Finish]                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Envelope Encryption

```
┌─────────────────────────────────────────────────────────────────────┐
│           ENVELOPE ENCRYPTION                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: KMS can only encrypt up to 4 KB directly.                 │
│ Solution: Envelope encryption (data key wrapping).                 │
│                                                                       │
│ How it works:                                                        │
│                                                                       │
│ 1. GenerateDataKey API call to KMS:                                │
│    KMS returns:                                                      │
│    ├── Plaintext data key (use to encrypt your data)            │
│    └── Encrypted data key (store alongside encrypted data)      │
│                                                                       │
│ 2. Encrypt your data:                                               │
│    ┌──────────────┐                                                │
│    │ Your Data    │ + Plaintext key = Encrypted Data             │
│    │ (any size)   │                                                │
│    └──────────────┘                                                │
│                                                                       │
│ 3. Delete plaintext key from memory immediately                   │
│                                                                       │
│ 4. Store:                                                            │
│    ┌────────────────────────────────────────────────────────┐     │
│    │ Encrypted Data  +  Encrypted Data Key (envelope)      │     │
│    └────────────────────────────────────────────────────────┘     │
│                                                                       │
│ To decrypt:                                                          │
│ 1. Send encrypted data key to KMS → Decrypt API                  │
│ 2. KMS returns plaintext data key                                  │
│ 3. Use plaintext data key to decrypt your data                    │
│ 4. Delete plaintext key from memory                                │
│                                                                       │
│ Why envelope encryption:                                            │
│ ├── Data never leaves your app (KMS only sees the data key)    │
│ ├── Fast (symmetric encryption is fast, KMS call is small)     │
│ ├── Scales (encrypt GBs/TBs without sending to KMS)           │
│ └── AWS services use this automatically (S3, EBS, RDS, etc.)  │
│                                                                       │
│ ⚡ AWS SDK Encryption Client handles this automatically          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Key Policies & Grants

```
┌─────────────────────────────────────────────────────────────────────┐
│           KEY POLICIES                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Every KMS key MUST have a key policy (resource-based policy).     │
│ Without it, NO ONE can use the key (not even root).               │
│                                                                       │
│ Default key policy (enables IAM policies to grant access):        │
│ {                                                                    │
│   "Statement": [{                                                   │
│     "Sid": "Enable IAM policies",                                  │
│     "Effect": "Allow",                                              │
│     "Principal": {"AWS": "arn:aws:iam::123456:root"},             │
│     "Action": "kms:*",                                              │
│     "Resource": "*"                                                  │
│   }]                                                                 │
│ }                                                                    │
│ → ⚠️ This doesn't grant access — it enables IAM policies to     │
│   grant access. Without this, IAM policies are ignored.           │
│                                                                       │
│ Custom key policy example:                                          │
│ {                                                                    │
│   "Statement": [                                                    │
│     {                                                                │
│       "Sid": "Admin",                                               │
│       "Effect": "Allow",                                            │
│       "Principal": {"AWS": "arn:aws:iam::123456:role/Admin"},     │
│       "Action": ["kms:Create*", "kms:Describe*", "kms:Enable*", │
│                   "kms:Put*", "kms:Update*", "kms:Revoke*",      │
│                   "kms:Disable*", "kms:Delete*", "kms:Schedule*"],│
│       "Resource": "*"                                               │
│     },                                                               │
│     {                                                                │
│       "Sid": "Users",                                               │
│       "Effect": "Allow",                                            │
│       "Principal": {"AWS": "arn:aws:iam::123456:role/AppRole"},  │
│       "Action": ["kms:Encrypt", "kms:Decrypt",                    │
│                   "kms:GenerateDataKey*", "kms:DescribeKey"],     │
│       "Resource": "*"                                               │
│     }                                                                │
│   ]                                                                  │
│ }                                                                    │
│                                                                       │
│ Grants:                                                              │
│ ├── Temporary, programmatic access to KMS keys                  │
│ ├── Alternative to key policy modification                       │
│ ├── Created by API call (not in key policy document)            │
│ ├── Used by AWS services (e.g., EBS volume attach)              │
│ └── Can be revoked without changing key policy                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Key Rotation & Multi-Region Keys

```
┌─────────────────────────────────────────────────────────────────────┐
│           KEY ROTATION                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Automatic rotation:                                                  │
│ Console → KMS → [Key] → Key rotation → Enable                    │
│ Rotation period: [365] days (90-2560 days)                         │
│                                                                       │
│ ├── New key material generated on schedule                       │
│ ├── Key ID and ARN stay the same                                 │
│ ├── Old key material kept (for decrypting old data)             │
│ ├── New encryptions use new key material                        │
│ ├── Transparent: No application changes needed                  │
│ └── ⚡ Always enable for customer managed keys                   │
│                                                                       │
│ Manual rotation:                                                     │
│ ├── Create new KMS key                                            │
│ ├── Update alias to point to new key                            │
│ ├── Keep old key for decrypting old data                        │
│ └── Use for: Asymmetric keys, imported key material             │
│                                                                       │
│ ── MULTI-REGION KEYS ──                                             │
│ Console → KMS → [Key] → Regionality → Create replica key        │
│                                                                       │
│ ├── Primary key in one region, replica keys in others           │
│ ├── Same key ID across regions (interchangeable)                │
│ ├── Encrypt in us-east-1, decrypt in eu-west-1                 │
│ ├── Independent key policies per region                         │
│ ├── Use for: Cross-region S3 replication encryption,           │
│ │   DynamoDB global tables, disaster recovery                  │
│ └── ⚡ Cost: $1/month per key per region                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: KMS with AWS Services

```
┌─────────────────────────────────────────────────────────────────────┐
│           KMS INTEGRATION                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Services using KMS (envelope encryption):                           │
│ ├── S3: SSE-KMS (per-object or bucket default)                  │
│ ├── EBS: Volume encryption (transparent to EC2)                 │
│ ├── RDS: Storage encryption, snapshot encryption                │
│ ├── DynamoDB: Table encryption                                   │
│ ├── Lambda: Environment variable encryption                    │
│ ├── Secrets Manager: Secret value encryption                   │
│ ├── SSM Parameter Store: SecureString parameters               │
│ ├── CloudTrail: Log file encryption                            │
│ ├── CloudWatch Logs: Log group encryption                      │
│ ├── SQS: Message encryption                                     │
│ ├── SNS: Message encryption                                     │
│ ├── Kinesis: Stream data encryption                             │
│ ├── Redshift: Cluster encryption                                │
│ └── ElastiCache: At-rest encryption                             │
│                                                                       │
│ ⚡ Almost every AWS service supports KMS encryption              │
│ ⚡ Always use customer managed keys for production              │
│   (audit trail + cross-account + custom rotation)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & CLI Examples

```hcl
resource "aws_kms_key" "data" {
  description             = "Production data encryption key"
  enable_key_rotation     = true
  rotation_period_in_days = 365
  deletion_window_in_days = 30
  multi_region            = false

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "EnableRootAccount"
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::123456789012:root" }
        Action    = "kms:*"
        Resource  = "*"
      },
      {
        Sid       = "AllowAppRole"
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::123456789012:role/AppRole" }
        Action    = ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey*"]
        Resource  = "*"
      }
    ]
  })

  tags = { Environment = "prod" }
}

resource "aws_kms_alias" "data" {
  name          = "alias/prod-data-key"
  target_key_id = aws_kms_key.data.key_id
}
```

```bash
# Create key
aws kms create-key --description "My encryption key"

# Create alias
aws kms create-alias --alias-name alias/my-key --target-key-id key-id

# Encrypt data (up to 4 KB)
aws kms encrypt \
  --key-id alias/my-key \
  --plaintext fileb://secret.txt \
  --output text --query CiphertextBlob | base64 -d > secret.enc

# Decrypt data
aws kms decrypt \
  --ciphertext-blob fileb://secret.enc \
  --output text --query Plaintext | base64 -d > secret.txt

# Generate data key (envelope encryption)
aws kms generate-data-key \
  --key-id alias/my-key \
  --key-spec AES_256

# Enable key rotation
aws kms enable-key-rotation --key-id key-id

# Schedule key deletion (7-30 day waiting period)
aws kms schedule-key-deletion --key-id key-id --pending-window-in-days 30
```

---

## Troubleshooting: Common KMS Issues

### "AccessDeniedException" When Encrypting/Decrypting

KMS permissions require BOTH IAM policy AND key policy to grant access. This trips up many users.

```
Debugging:
1. ☐ Does the IAM user/role have kms:Encrypt/kms:Decrypt?
   Check IAM → User/Role → Permissions

2. ☐ Does the KMS key policy allow the IAM principal?
   Console → KMS → Key → Key policy tab
   The default key policy delegates to IAM policies.
   Custom key policies may restrict access explicitly.

3. ☐ Cross-account? Both sides need permissions:
   Source account: IAM policy allows kms:Decrypt
   Key account: Key policy allows the source account's principal

4. ☐ Is the key in the right region?
   KMS keys are region-specific. A key in us-east-1 can't be used
   by a service in ap-south-1.
```

### "Key is pending deletion"

```
KMS keys have a 7-30 day waiting period before actual deletion.

Fix: Cancel the deletion if it was accidental:
  Console → KMS → Key → Key actions → Cancel deletion
  CLI: aws kms cancel-key-deletion --key-id KEY_ID

⚠️ Once a key is actually deleted, ALL DATA encrypted with it
is PERMANENTLY UNRECOVERABLE. There is no backup.
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using AWS managed keys for cross-account | Can't share AWS managed keys | Create CMK (customer managed key) |
| Accidentally scheduling key deletion | 7-day countdown to data loss | Cancel immediately, set longer waiting period |
| Not enabling key rotation | Compliance violation | Enable automatic rotation (every year) |
| Granting kms:* to everyone | Over-permissive, audit failure | Use specific actions (Encrypt, Decrypt, GenerateDataKey) |

---

## Quick Reference

```
KMS Quick Reference:
├── What: Managed encryption key service
├── Key types: Symmetric (AES-256), Asymmetric (RSA, ECC)
├── Key ownership: AWS owned, AWS managed, Customer managed
├── Envelope encryption: Data key wraps data, KMS wraps data key
├── Key policy: Required (resource-based policy on every key)
├── Key rotation: Automatic (365 days default), enable always
├── Multi-region: Same key replicated across regions
├── Pricing: $1/key/month + $0.03/10K API calls
├── Integrates with: Almost every AWS service
├── ⚡ Customer managed keys for production (audit + control)
└── ⚡ Never disable or delete keys without checking dependencies
```

---

## What's Next?

In **Chapter 41: Secrets Manager**, we'll cover secure storage and automatic rotation of secrets like database passwords, API keys, and credentials.
