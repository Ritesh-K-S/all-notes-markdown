# Chapter 21: S3 Advanced Features

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Bucket Policies](#part-1-bucket-policies)
- [Part 2: ACLs (Legacy)](#part-2-acls-legacy)
- [Part 3: Presigned URLs](#part-3-presigned-urls)
- [Part 4: Event Notifications](#part-4-event-notifications)
- [Part 5: S3 Select & Glacier Select](#part-5-s3-select--glacier-select)
- [Part 6: Access Points](#part-6-access-points)
- [Part 7: S3 Batch Operations](#part-7-s3-batch-operations)
- [Part 8: S3 Object Lambda](#part-8-s3-object-lambda)
- [Part 9: S3 Inventory & Analytics](#part-9-s3-inventory--analytics)
- [Part 10: Transfer Acceleration](#part-10-transfer-acceleration)
- [Part 11: Terraform & CLI Examples](#part-11-terraform--cli-examples)
- [Part 12: Real-World Patterns](#part-12-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### When Will You Need These Features?

In Chapter 20, you learned how to create buckets, upload files, and manage storage classes. But in real production environments, you'll quickly need more:

- **"I need to let users upload files directly from a browser"** → Presigned URLs
- **"I want to process images automatically when uploaded"** → Event Notifications
- **"Multiple teams share one bucket, each needs different access"** → Access Points
- **"I need to enforce HTTPS-only access to my bucket"** → Bucket Policies
- **"My uploads are slow because users are far from the AWS region"** → Transfer Acceleration

This chapter covers advanced S3 features for access control, event-driven architectures, data processing, and performance optimization. These are essential for production-grade S3 usage.

```
What you'll learn:
├── Bucket Policies (JSON-based access control)
├── ACLs (legacy access control — when you still see them)
├── Presigned URLs (temporary secure access)
├── Event Notifications (S3 → Lambda/SQS/SNS/EventBridge)
├── S3 Select (query inside objects with SQL)
├── Access Points (simplified bucket access management)
├── S3 Batch Operations (bulk operations on millions of objects)
├── S3 Object Lambda (transform objects on retrieval)
├── S3 Inventory & Storage Analytics
├── Transfer Acceleration (faster uploads via edge locations)
└── CORS configuration
```

---

## Part 1: Bucket Policies

```
Console → S3 → Bucket → Permissions → Bucket policy → Edit

┌─────────────────────────────────────────────────────────────────┐
│           BUCKET POLICIES                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: JSON-based policies attached to a bucket that define     │
│ who can do what on the bucket and its objects.                  │
│                                                                   │
│ Structure:                                                      │
│ {                                                                │
│   "Version": "2012-10-17",                                     │
│   "Statement": [                                                │
│     {                                                            │
│       "Sid": "AllowPublicRead",                                │
│       "Effect": "Allow | Deny",                                │
│       "Principal": "who (*, account, role, user)",             │
│       "Action": "what (s3:GetObject, s3:PutObject...)",        │
│       "Resource": "which bucket/objects",                      │
│       "Condition": { optional conditions }                     │
│     }                                                            │
│   ]                                                              │
│ }                                                                │
│                                                                   │
│ Key fields:                                                     │
│ ├── Sid: Statement ID (human-readable name, optional)         │
│ ├── Effect: Allow or Deny                                     │
│ ├── Principal: Who the policy applies to                      │
│ │   ├── "*" → everyone (public)                               │
│ │   ├── "arn:aws:iam::123456789012:root" → specific account  │
│ │   ├── "arn:aws:iam::123456789012:role/MyRole" → IAM role   │
│ │   └── {"Service": "cloudfront.amazonaws.com"} → AWS service│
│ ├── Action: S3 API actions                                    │
│ │   ├── "s3:GetObject" → read objects                        │
│ │   ├── "s3:PutObject" → upload objects                      │
│ │   ├── "s3:DeleteObject" → delete objects                   │
│ │   ├── "s3:ListBucket" → list objects in bucket             │
│ │   └── "s3:*" → all S3 actions                             │
│ ├── Resource: Which bucket/objects                             │
│ │   ├── "arn:aws:s3:::my-bucket" → the bucket itself         │
│ │   ├── "arn:aws:s3:::my-bucket/*" → all objects in bucket   │
│ │   └── "arn:aws:s3:::my-bucket/public/*" → specific prefix  │
│ └── Condition: Additional constraints                          │
│     ├── IpAddress: Restrict by source IP                      │
│     ├── StringEquals aws:PrincipalOrgID: Restrict to AWS Org  │
│     ├── SecureTransport: Require HTTPS                        │
│     └── And many more...                                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Common Bucket Policy Examples:

1. Force HTTPS only:
{
  "Statement": [{
    "Sid": "ForceHTTPS",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ],
    "Condition": {
      "Bool": {"aws:SecureTransport": "false"}
    }
  }]
}

2. Allow CloudFront OAC access only:
{
  "Statement": [{
    "Sid": "AllowCloudFrontOAC",
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456:distribution/EDFDVBD6"
      }
    }
  }]
}

3. Cross-account access:
{
  "Statement": [{
    "Sid": "CrossAccountAccess",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::987654321098:root"},
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ]
  }]
}

4. Restrict to VPC endpoint:
{
  "Statement": [{
    "Sid": "RestrictToVPCEndpoint",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ],
    "Condition": {
      "StringNotEquals": {
        "aws:sourceVpce": "vpce-1234567890abcdef0"
      }
    }
  }]
}
```

---

## Part 2: ACLs (Legacy)

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACLs (ACCESS CONTROL LISTS) — LEGACY                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️ AWS recommends DISABLING ACLs for new buckets.                   │
│ Use bucket policies + IAM policies instead.                         │
│                                                                       │
│ When ACLs are still relevant:                                       │
│ ├── Legacy applications that depend on ACLs                       │
│ ├── S3 access logging (log delivery group needs ACL)             │
│ └── Some cross-account scenarios (prefer bucket policies)         │
│                                                                       │
│ Canned ACLs:                                                        │
│ ├── private: Owner gets FULL_CONTROL. Default.                   │
│ ├── public-read: Owner FULL_CONTROL + public READ               │
│ ├── public-read-write: ⚠️ Dangerous! Public read+write          │
│ ├── authenticated-read: Any authenticated AWS user can read     │
│ ├── bucket-owner-full-control: Used for cross-account uploads   │
│ └── log-delivery-write: Used for S3 access logging              │
│                                                                       │
│ Best practice: Disable ACLs (Object Ownership = Bucket Owner     │
│ Enforced) and use bucket policies exclusively.                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Presigned URLs

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRESIGNED URLs                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Temporary URLs that grant time-limited access to a private  │
│ S3 object. No bucket policy or ACL changes needed.                 │
│                                                                       │
│ Use cases:                                                           │
│ ├── Allow users to download a private file (e.g., invoice PDF)  │
│ ├── Allow users to upload directly to S3 (presigned PUT)        │
│ ├── Share files temporarily without making them public           │
│ └── Frontend direct uploads (bypass your server)                 │
│                                                                       │
│ How it works:                                                        │
│ 1. Your server generates a presigned URL using AWS credentials   │
│ 2. URL contains: bucket, key, expiration, signature             │
│ 3. Anyone with the URL can access the object until expiration   │
│ 4. After expiration → 403 Forbidden                              │
│                                                                       │
│ Generate presigned URL:                                             │
│                                                                       │
│ Console:                                                             │
│ S3 → Bucket → Object → Actions → Share with presigned URL       │
│   Time interval: [Minutes ▼] [15]                                 │
│   → Creates URL valid for specified duration                     │
│                                                                       │
│ CLI:                                                                 │
│ aws s3 presign s3://my-bucket/file.pdf --expires-in 3600        │
│ → Returns URL valid for 1 hour (3600 seconds)                   │
│ → Max: 7 days (604800 seconds) with IAM user credentials        │
│ → Max: 12 hours with STS/role temporary credentials             │
│                                                                       │
│ SDK (Python):                                                        │
│ url = s3_client.generate_presigned_url(                           │
│     'get_object',                                                   │
│     Params={'Bucket': 'my-bucket', 'Key': 'file.pdf'},          │
│     ExpiresIn=3600                                                  │
│ )                                                                    │
│                                                                       │
│ Presigned URL for uploads (PUT):                                    │
│ url = s3_client.generate_presigned_url(                           │
│     'put_object',                                                   │
│     Params={                                                        │
│         'Bucket': 'my-bucket',                                     │
│         'Key': 'uploads/user123/photo.jpg',                       │
│         'ContentType': 'image/jpeg'                                │
│     },                                                              │
│     ExpiresIn=3600                                                  │
│ )                                                                    │
│ # Frontend: PUT request to this URL with file in body            │
│                                                                       │
│ ⚡ Security:                                                          │
│ ├── URL inherits permissions of the IAM entity that created it  │
│ ├── If creator's credentials are revoked, URL stops working     │
│ ├── Keep expiration as short as possible                         │
│ └── Never log or expose presigned URLs in public                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Event Notifications

```
Console → S3 → Bucket → Properties → Event notifications → Create event notification

┌─────────────────────────────────────────────────────────────────┐
│           CREATE EVENT NOTIFICATION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Event name: [image-uploaded]                                   │
│                                                                   │
│ ── Prefix and suffix filters ──                                │
│ Prefix: [uploads/]                                             │
│ Suffix: [.jpg]                                                 │
│ → Filter events to specific objects                            │
│ → Prefix: "uploads/" only triggers for that prefix            │
│ → Suffix: ".jpg" only triggers for JPEG files                 │
│                                                                   │
│ ── Event types ──                                               │
│ All object create events:                                      │
│   ☑ s3:ObjectCreated:*                                        │
│   ☐ s3:ObjectCreated:Put                                      │
│   ☐ s3:ObjectCreated:Post                                     │
│   ☐ s3:ObjectCreated:Copy                                     │
│   ☐ s3:ObjectCreated:CompleteMultipartUpload                  │
│                                                                   │
│ All object removal events:                                     │
│   ☐ s3:ObjectRemoved:*                                        │
│   ☐ s3:ObjectRemoved:Delete                                   │
│   ☐ s3:ObjectRemoved:DeleteMarkerCreated                      │
│                                                                   │
│ Other:                                                          │
│   ☐ s3:ObjectRestore:Post (Glacier restore initiated)         │
│   ☐ s3:ObjectRestore:Completed                                │
│   ☐ s3:ObjectTagging:Put                                      │
│   ☐ s3:ObjectTagging:Delete                                   │
│                                                                   │
│ ── Destination ──                                               │
│ ● Lambda function                                               │
│ ○ SNS topic                                                     │
│ ○ SQS queue                                                     │
│                                                                   │
│ Lambda function: [image-resizer]  ▼                            │
│ → S3 invokes this Lambda when event occurs                    │
│ → Lambda resource policy must allow s3.amazonaws.com          │
│                                                                   │
│                    [Save changes]                               │
│                                                                   │
│ Alternative: Amazon EventBridge                                │
│ S3 → Properties → Amazon EventBridge → Edit → Enable          │
│ → Sends ALL S3 events to EventBridge                          │
│ → More powerful: filtering, multiple targets, replay          │
│ → ⚡ Recommended for new architectures                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Event notification targets:
├── Lambda: Process the object (resize image, parse CSV, etc.)
├── SQS: Queue events for async processing
├── SNS: Fan-out to multiple subscribers
└── EventBridge: Advanced routing, filtering, and replay
```

---

## Part 5: S3 Select & Glacier Select

```
┌─────────────────────────────────────────────────────────────────────┐
│           S3 SELECT                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Query data inside S3 objects using SQL without downloading  │
│ the entire object. Reduces data transfer and processing time.      │
│                                                                       │
│ Supported formats: CSV, JSON, Parquet                               │
│                                                                       │
│ Console:                                                             │
│ S3 → Bucket → Object → Actions → Query with S3 Select            │
│   Input settings:                                                   │
│     Format: [CSV ▼]                                                 │
│     Delimiter: [Comma ▼]                                            │
│     Header: ☑ First line is header                                 │
│     Compression: [None ▼] (or GZIP, BZIP2)                        │
│                                                                       │
│   SQL expression:                                                   │
│   ┌────────────────────────────────────────────────────────────┐   │
│   │ SELECT s.name, s.age                                       │   │
│   │ FROM s3object s                                            │   │
│   │ WHERE s.age > 25                                           │   │
│   │ LIMIT 100                                                  │   │
│   └────────────────────────────────────────────────────────────┘   │
│                                                                       │
│   Output settings:                                                  │
│     Format: [CSV ▼]                                                 │
│                                                                       │
│ Benefits:                                                            │
│ ├── Transfer only needed data (save bandwidth & time)            │
│ ├── Up to 400% faster than downloading full object               │
│ ├── Up to 80% less data transferred                              │
│ └── Works with GZIP/BZIP2 compressed files                      │
│                                                                       │
│ ⚠️ S3 Select is being superseded by S3 Object Lambda and Athena   │
│ for more complex queries.                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Access Points

```
Console → S3 → Access Points → Create access point

┌─────────────────────────────────────────────────────────────────┐
│           CREATE ACCESS POINT                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Access point name: [finance-team-access]                       │
│ → Creates a unique hostname for accessing the bucket          │
│ → Each access point has its own policy and network settings   │
│                                                                   │
│ Bucket: [my-company-data-lake]                                │
│                                                                   │
│ Network origin:                                                │
│   ● Internet                                                    │
│   ○ Virtual private cloud (VPC)                                │
│                                                                   │
│ → Internet: Accessible from anywhere                          │
│ → VPC: Only accessible from specified VPC                     │
│   VPC ID: [vpc-0abc123def456]                                 │
│                                                                   │
│ Block Public Access settings:                                  │
│   ☑ Block all public access                                   │
│                                                                   │
│ Access point policy:                                           │
│ {                                                                │
│   "Statement": [{                                               │
│     "Effect": "Allow",                                         │
│     "Principal": {                                              │
│       "AWS": "arn:aws:iam::123456789012:role/FinanceRole"     │
│     },                                                          │
│     "Action": ["s3:GetObject", "s3:ListBucket"],              │
│     "Resource": [                                               │
│       "arn:aws:s3:us-east-1:123456789012:accesspoint/         │
│        finance-team-access/object/finance/*"                   │
│     ]                                                           │
│   }]                                                            │
│ }                                                                │
│                                                                   │
│ → Why access points?                                           │
│   ├── Simplify managing access for shared data lakes         │
│   ├── Each team gets their own access point + policy         │
│   ├── Own network controls (Internet vs VPC)                 │
│   ├── Avoid massive, complex bucket policies                 │
│   └── Up to 10,000 access points per region per account      │
│                                                                   │
│                    [Create access point]                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 7: S3 Batch Operations

```
Console → S3 → Batch Operations → Create job

┌─────────────────────────────────────────────────────────────────┐
│           S3 BATCH OPERATIONS                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Perform bulk operations on billions of objects at once.  │
│                                                                   │
│ ── Manifest ──                                                  │
│ Manifest format:                                               │
│   ● S3 Inventory report                                        │
│   ○ CSV manifest                                                │
│ Manifest object: s3://my-bucket/inventory/manifest.json       │
│ → List of objects to process                                   │
│                                                                   │
│ ── Operation ──                                                │
│ ○ Copy                                                          │
│ ○ Invoke AWS Lambda function                                   │
│ ● Replace all object tags                                      │
│ ○ Delete all object tags                                       │
│ ○ Replace access control list (ACL)                            │
│ ○ Restore from Glacier                                         │
│ ○ Object Lock retention                                        │
│ ○ Object Lock legal hold                                       │
│                                                                   │
│ Use cases:                                                      │
│ ├── Copy millions of objects to another bucket               │
│ ├── Change storage class for all objects                     │
│ ├── Re-encrypt objects with new KMS key                      │
│ ├── Apply tags to millions of objects                         │
│ ├── Invoke Lambda on every object                            │
│ └── Restore all Glacier objects at once                       │
│                                                                   │
│ ── Completion report ──                                        │
│ Report bucket: [s3://my-bucket/batch-reports/]               │
│ Report scope: ● All tasks ○ Failed tasks only                 │
│                                                                   │
│ ── Permissions ──                                               │
│ IAM role: [s3-batch-role]                                     │
│ → Needs permissions for both source and destination           │
│                                                                   │
│                    [Create job] → Requires confirmation       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 8: S3 Object Lambda

```
┌─────────────────────────────────────────────────────────────────────┐
│           S3 OBJECT LAMBDA                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Transform objects on-the-fly when retrieved.                 │
│ Your Lambda function modifies data before returning to caller.     │
│                                                                       │
│ Flow:                                                                │
│ ┌──────┐    ┌───────────────┐    ┌────────┐    ┌──────────────┐   │
│ │ App  │ → │ Object Lambda │ → │ Lambda │ → │ S3 (original)│   │
│ │      │ ← │ Access Point  │ ← │ (xform)│ ← │              │   │
│ └──────┘    └───────────────┘    └────────┘    └──────────────┘   │
│                                                                       │
│ Use cases:                                                           │
│ ├── Redact PII from documents before serving                     │
│ ├── Resize images on retrieval                                    │
│ ├── Convert data formats (CSV → JSON)                            │
│ ├── Decompress/decrypt custom formats                             │
│ └── Add watermarks to images                                      │
│                                                                       │
│ Console → S3 → Object Lambda Access Points → Create                │
│ ├── Access point name: [redacted-data]                           │
│ ├── Supporting access point: [arn:aws:s3:...:my-bucket-ap]      │
│ ├── Lambda function: [redact-pii-function]                       │
│ ├── S3 APIs: ☑ GetObject ☐ HeadObject ☐ ListObjects            │
│ └── Payload: Optional JSON config passed to Lambda               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: S3 Inventory & Analytics

```
Console → S3 → Bucket → Management → Inventory configurations

┌─────────────────────────────────────────────────────────────────┐
│           S3 INVENTORY                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Scheduled report of all objects and metadata in bucket.  │
│ Much faster than LIST API for large buckets.                   │
│                                                                   │
│ Inventory name: [weekly-full-inventory]                        │
│                                                                   │
│ Destination:                                                    │
│   Bucket: [s3://my-bucket-inventory/]                         │
│   Prefix: [inventory-reports/]                                │
│   Format: ● CSV ○ ORC ○ Parquet                               │
│   Encryption: ○ None ● SSE-S3 ○ SSE-KMS                      │
│                                                                   │
│ Frequency: ○ Daily ● Weekly                                    │
│                                                                   │
│ Object versions: ● Current version only ○ All versions         │
│                                                                   │
│ Additional fields:                                              │
│   ☑ Size                                                       │
│   ☑ Last modified date                                         │
│   ☑ Storage class                                              │
│   ☑ Encryption status                                          │
│   ☑ ETag                                                       │
│   ☐ Replication status                                         │
│   ☐ Object lock                                                │
│                                                                   │
│ Use cases:                                                      │
│ ├── Audit all objects in bucket                               │
│ ├── Identify unencrypted objects                              │
│ ├── Find objects by storage class                             │
│ ├── Input for S3 Batch Operations (manifest)                 │
│ └── Compliance reporting                                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Storage Class Analysis:
Console → S3 → Bucket → Metrics → Storage Class Analysis
→ Analyzes access patterns to recommend optimal storage class
→ Takes 24-48 hours to start generating recommendations
→ Helps decide: Should objects move to IA or Glacier?
```

---

## Part 10: Transfer Acceleration

```
Console → S3 → Bucket → Properties → Transfer acceleration → Edit

┌─────────────────────────────────────────────────────────────────┐
│           TRANSFER ACCELERATION                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Transfer acceleration: ● Enabled ○ Suspended                   │
│                                                                   │
│ How it works:                                                   │
│ ┌─────────┐    ┌─────────────────┐    ┌──────────────────┐    │
│ │ Client  │ →  │ CloudFront Edge │ →  │ S3 Bucket        │    │
│ │ (far    │    │ Location (near  │    │ (us-east-1)      │    │
│ │ away)   │    │ client)         │    │                   │    │
│ └─────────┘    └─────────────────┘    └──────────────────┘    │
│     ↑               ↑                        ↑                │
│  Public         AWS backbone          Normal S3 endpoint     │
│  internet       (fast, optimized)                             │
│                                                                   │
│ → Uses AWS backbone network instead of public internet       │
│ → 50-500% faster for distant uploads                         │
│ → Extra cost: $0.04-$0.08/GB                                 │
│                                                                   │
│ Endpoint changes:                                              │
│ Normal:   my-bucket.s3.amazonaws.com                          │
│ Accel:    my-bucket.s3-accelerate.amazonaws.com               │
│                                                                   │
│ ⚡ Speed test: https://s3-accelerate-speedtest.s3-accelerate.a│
│   mazonaws.com/en/accelerate-speed-comparsion.html            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 10.5: CORS (Cross-Origin Resource Sharing)

### What is CORS? Why Do I Need It?

**CORS** is a browser security feature. If your website at `https://myapp.com` tries to load an image or file from `https://my-bucket.s3.amazonaws.com`, the browser **blocks it** by default because it's a different domain ("cross-origin").

```
Without CORS configured:
Browser at myapp.com → fetch("s3.amazonaws.com/file.jpg") → ❌ BLOCKED

With CORS configured on the S3 bucket:
Browser at myapp.com → fetch("s3.amazonaws.com/file.jpg") → ✅ Allowed
```

### When Do You Need CORS?

```
├── Static website hosted on S3 + API on different domain     ✅ Need CORS
├── JavaScript uploading files directly to S3 (presigned URL) ✅ Need CORS
├── Single-page app (React/Vue) loading assets from S3        ✅ Need CORS
├── Server-side app (Node.js/Python) accessing S3 via SDK     ❌ No CORS needed
└── CloudFront in front of S3 (same domain)                   ❌ No CORS needed
```

> 💡 CORS is a **browser** restriction only. Server-to-server calls (Lambda → S3, EC2 → S3) are never blocked by CORS.

### Console Configuration

```
Console → S3 → Bucket → Permissions tab → CORS configuration → Edit

Paste this JSON:
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": ["https://myapp.com"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3600
  }
]
```

**Field explanations:**

| Field | What It Means | Example |
|-------|---------------|---------|
| AllowedOrigins | Which websites can access this bucket | `["https://myapp.com"]` or `["*"]` for any |
| AllowedMethods | Which HTTP methods are allowed | `["GET"]` for read-only, add `PUT` for uploads |
| AllowedHeaders | Which request headers are allowed | `["*"]` allows all |
| ExposeHeaders | Which response headers the browser can read | `["ETag"]` for upload verification |
| MaxAgeSeconds | How long to cache the CORS check | `3600` = 1 hour |

### Terraform Example

```hcl
resource "aws_s3_bucket_cors_configuration" "example" {
  bucket = aws_s3_bucket.main.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = ["https://myapp.com"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3600
  }
}
```

> ⚠️ **Common mistake:** Forgetting to add CORS when using presigned URLs for direct browser uploads. The upload will fail with a CORS error even though the URL is valid.

---

## Part 11: Terraform & CLI Examples

```hcl
# Bucket policy — force HTTPS
resource "aws_s3_bucket_policy" "force_https" {
  bucket = aws_s3_bucket.main.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "ForceHTTPS"
      Effect    = "Deny"
      Principal = "*"
      Action    = "s3:*"
      Resource = [
        aws_s3_bucket.main.arn,
        "${aws_s3_bucket.main.arn}/*"
      ]
      Condition = {
        Bool = { "aws:SecureTransport" = "false" }
      }
    }]
  })
}

# CORS configuration
resource "aws_s3_bucket_cors_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = ["https://myapp.com"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3600
  }
}

# Event notification to Lambda
resource "aws_s3_bucket_notification" "lambda" {
  bucket = aws_s3_bucket.main.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.image_resize.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "uploads/"
    filter_suffix       = ".jpg"
  }
}

# Event notification to EventBridge
resource "aws_s3_bucket_notification" "eventbridge" {
  bucket      = aws_s3_bucket.main.id
  eventbridge = true
}
```

```bash
# Generate presigned URL (GET — download)
aws s3 presign s3://my-bucket/reports/q4-2024.pdf --expires-in 3600

# Generate presigned URL (PUT — upload) 
aws s3 presign s3://my-bucket/uploads/photo.jpg \
  --expires-in 3600 \
  --region us-east-1

# CORS configuration
aws s3api put-bucket-cors --bucket my-bucket \
  --cors-configuration '{
    "CORSRules": [{
      "AllowedOrigins": ["https://myapp.com"],
      "AllowedMethods": ["GET", "PUT"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3600
    }]
  }'

# Enable Transfer Acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket my-bucket \
  --accelerate-configuration Status=Enabled

# Enable EventBridge notifications
aws s3api put-bucket-notification-configuration \
  --bucket my-bucket \
  --notification-configuration '{"EventBridgeConfiguration": {}}'
```

---

## Part 12: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           ADVANCED S3 PATTERNS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Secure Direct Upload from Frontend                       │
│ ┌──────┐  1.request  ┌────────┐  2.presigned  ┌──────┐            │
│ │ App  │ ──────────► │ Server │ ────────────► │ App  │             │
│ │      │             │ (API)  │    URL        │      │             │
│ └──────┘             └────────┘               └──┬───┘             │
│                                                   │ 3.upload       │
│                                              ┌────▼────┐           │
│                                              │ S3      │           │
│                                              └─────────┘           │
│ → Server generates presigned PUT URL → frontend uploads directly │
│ → No file passes through your server                              │
│                                                                       │
│ Pattern 2: Data Lake with Access Points                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ S3 Data Lake Bucket                                          │   │
│ │ ├── /raw/         ← ingestion access point                 │   │
│ │ ├── /processed/   ← analytics access point                 │   │
│ │ ├── /finance/     ← finance-team access point (VPC only)   │   │
│ │ └── /public/      ← public access point (read-only)        │   │
│ └──────────────────────────────────────────────────────────────┘   │
│ → Each team has their own access point with specific policies    │
│                                                                       │
│ Pattern 3: Event-Driven Image Processing                            │
│ S3 upload → EventBridge → Step Functions                           │
│   ├── Lambda 1: Validate image                                    │
│   ├── Lambda 2: Resize (thumbnail, medium, large)                │
│   ├── Lambda 3: Extract metadata (EXIF)                           │
│   └── Lambda 4: Update database with URLs                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
S3 Advanced Quick Reference:
├── Bucket policies: JSON-based, attached to bucket
├── Presigned URLs: Temp access, max 7 days (IAM user), 12hrs (role)
├── Event notifications: S3 → Lambda/SQS/SNS/EventBridge
├── S3 Select: SQL queries on CSV/JSON/Parquet objects
├── Access Points: Simplified access management for shared buckets
├── Batch Operations: Bulk operations on millions of objects
├── Object Lambda: Transform objects on retrieval
├── Inventory: Scheduled reports of all objects
├── Transfer Acceleration: Faster uploads via edge locations
└── CORS: Required for browser-based uploads
```

---

## What's Next?

In **Chapter 22: EBS - Elastic Block Store**, we'll cover block storage volumes for EC2 instances — volume types, snapshots, encryption, and performance optimization.
