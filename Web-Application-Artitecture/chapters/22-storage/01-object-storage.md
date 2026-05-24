# Object Storage (S3, GCS, Azure Blob)

> **What you'll learn**: How object storage works, why it's the backbone of modern cloud applications, and how to use services like AWS S3, Google Cloud Storage, and Azure Blob Storage to store unlimited amounts of unstructured data.

---

## Real-Life Analogy

Imagine you have a **massive warehouse with infinite shelves**. Each item you store gets:
- A **unique label** (like a barcode) so you can find it later
- A **description tag** attached to it (metadata: when you stored it, how heavy it is, what category)
- It goes into a **specific section** of the warehouse (bucket/container)

You don't care *which exact shelf* it's on — you just hand the warehouse your item with a label, and when you need it back, you give them the label and they find it instantly. You never reorganize the shelves, never worry about running out of space — the warehouse magically expands.

That's object storage. Unlike your computer's file system (where you think in folders and subfolders), object storage is a **flat structure** — everything is just an object with a key (name) in a bucket (container).

---

## Core Concept Explained Step-by-Step

### What is Object Storage?

**Object storage** is a data storage architecture that manages data as discrete units called **objects**. Each object contains three things:

```
┌─────────────────────────────────────────────┐
│              OBJECT                          │
├─────────────────────────────────────────────┤
│  1. DATA        → The actual file content   │
│                   (image, video, log, etc.)  │
│                                             │
│  2. METADATA    → Key-value pairs about     │
│                   the object                │
│                   (size, type, created_at,  │
│                    custom tags)             │
│                                             │
│  3. UNIQUE KEY  → Globally unique identifier│
│                   (like a file path)        │
│                   e.g., "photos/cat.jpg"    │
└─────────────────────────────────────────────┘
```

### How is it Different from a File System?

```
FILE SYSTEM (Your Computer)                  OBJECT STORAGE (Cloud)
─────────────────────────────                ─────────────────────────────
/home/                                       Bucket: "my-app-photos"
  /user/                                       ├── photos/vacation/beach.jpg
    /photos/                                   ├── photos/vacation/sunset.jpg
      /vacation/                               ├── thumbnails/beach_thumb.jpg
        beach.jpg                              └── raw/DSC_0001.RAW
        sunset.jpg
    /thumbnails/                             (Flat structure! The "/" in keys
      beach_thumb.jpg                         is just a naming convention,
                                              NOT actual folders)
Hierarchical tree structure                  Flat key-value structure
```

### The Building Blocks

```
┌────────────────────────────────────────────────────────────┐
│                     OBJECT STORAGE SERVICE                  │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Bucket A   │  │   Bucket B   │  │   Bucket C   │    │
│  │  "user-imgs" │  │  "app-logs"  │  │  "backups"   │    │
│  │              │  │              │  │              │    │
│  │ ┌──┐ ┌──┐   │  │ ┌──┐ ┌──┐   │  │ ┌──┐ ┌──┐   │    │
│  │ │OB│ │OB│   │  │ │OB│ │OB│   │  │ │OB│ │OB│   │    │
│  │ └──┘ └──┘   │  │ └──┘ └──┘   │  │ └──┘ └──┘   │    │
│  │ ┌──┐ ┌──┐   │  │ ┌──┐        │  │ ┌──┐ ┌──┐   │    │
│  │ │OB│ │OB│   │  │ │OB│        │  │ │OB│ │OB│   │    │
│  │ └──┘ └──┘   │  │ └──┘        │  │ └──┘ └──┘   │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                            │
│  OB = Object (data + metadata + key)                       │
└────────────────────────────────────────────────────────────┘
```

### Key Properties of Object Storage

| Property | Description |
|----------|-------------|
| **Flat namespace** | No directory hierarchy — just buckets and keys |
| **Unlimited scale** | Store petabytes without provisioning |
| **HTTP access** | Read/write via REST APIs (GET, PUT, DELETE) |
| **Rich metadata** | Custom key-value pairs on each object |
| **Durability** | 99.999999999% (11 nines) — data is replicated |
| **Eventual consistency** | Some operations may take a moment to propagate |
| **Immutable** | Objects are replaced whole, not modified in-place |

---

## How It Works Internally

### The Write Path (Uploading an Object)

```
Client                    API Gateway              Storage Nodes
  │                          │                         │
  │  PUT /bucket/key         │                         │
  │  + data + metadata       │                         │
  │─────────────────────────▶│                         │
  │                          │                         │
  │                          │  1. Validate request    │
  │                          │  2. Generate object ID  │
  │                          │  3. Split into chunks   │
  │                          │                         │
  │                          │   Write chunk 1 ───────▶│ Node A
  │                          │   Write chunk 1 ───────▶│ Node B (replica)
  │                          │   Write chunk 1 ───────▶│ Node C (replica)
  │                          │                         │
  │                          │   Write chunk 2 ───────▶│ Node D
  │                          │   Write chunk 2 ───────▶│ Node E (replica)
  │                          │   Write chunk 2 ───────▶│ Node F (replica)
  │                          │                         │
  │                          │  4. Update metadata     │
  │                          │     index               │
  │                          │                         │
  │  HTTP 200 OK             │                         │
  │◀─────────────────────────│                         │
```

### The Read Path (Downloading an Object)

```
Client                    API Gateway              Metadata Index     Storage Nodes
  │                          │                         │                    │
  │  GET /bucket/key         │                         │                    │
  │─────────────────────────▶│                         │                    │
  │                          │  Lookup key ───────────▶│                    │
  │                          │                         │                    │
  │                          │  ◀── chunk locations ───│                    │
  │                          │      (Node A: chunk 1)  │                    │
  │                          │      (Node D: chunk 2)  │                    │
  │                          │                         │                    │
  │                          │  Fetch chunk 1 ─────────────────────────────▶│ Node A
  │                          │  Fetch chunk 2 ─────────────────────────────▶│ Node D
  │                          │                         │                    │
  │  HTTP 200 + data         │                         │                    │
  │◀─────────────────────────│                         │                    │
```

### How Durability is Achieved

AWS S3, for example, provides **99.999999999% durability** (11 nines). This means if you store 10 million objects, you can expect to lose 1 object every 10,000 years. How?

```
┌─────────────────────────────────────────────────┐
│          DURABILITY MECHANISMS                   │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. REPLICATION                                 │
│     Object is copied to 3+ Availability Zones  │
│                                                 │
│     AZ-1        AZ-2        AZ-3              │
│    ┌─────┐    ┌─────┐    ┌─────┐             │
│    │Copy1│    │Copy2│    │Copy3│             │
│    └─────┘    └─────┘    └─────┘             │
│                                                 │
│  2. CHECKSUMS                                   │
│     Every chunk has MD5/SHA-256 hash verified  │
│     on read and during background scrubbing    │
│                                                 │
│  3. ERASURE CODING (for cost efficiency)        │
│     Split data into N fragments, need K to     │
│     reconstruct (e.g., 6 of 9 fragments)       │
│                                                 │
│  4. BACKGROUND REPAIR                           │
│     Continuously scan and fix corrupted copies │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Storage Classes (Cost Optimization)

```
                        ACCESS FREQUENCY
    ─────────────────────────────────────────────────▶
    HOT                                          COLD

┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐
│ STANDARD │  │ INFREQ.  │  │  GLACIER  │  │ DEEP     │
│          │  │ ACCESS   │  │ FLEXIBLE  │  │ ARCHIVE  │
│          │  │          │  │           │  │          │
│ Cost: $$$│  │ Cost: $$ │  │ Cost: $   │  │ Cost: ¢  │
│ Access:  │  │ Access:  │  │ Access:   │  │ Access:  │
│ instant  │  │ instant  │  │ minutes   │  │ 12 hours │
│          │  │ +retriev.│  │ to hours  │  │          │
│          │  │ fee      │  │           │  │          │
└──────────┘  └──────────┘  └───────────┘  └──────────┘

 Use for:      Use for:      Use for:      Use for:
 Active app    Monthly       Quarterly     Compliance
 data, user    reports,      backups,      archives,
 uploads       old logs      audit data    legal holds
```

---

## Code Examples

### Python — Using AWS S3 with boto3

```python
import boto3
from botocore.exceptions import ClientError

# Initialize the S3 client
s3 = boto3.client('s3', region_name='us-east-1')

# --- CREATE A BUCKET ---
s3.create_bucket(Bucket='my-app-photos-2024')

# --- UPLOAD AN OBJECT ---
s3.upload_file(
    Filename='local_photo.jpg',          # Local file path
    Bucket='my-app-photos-2024',         # Target bucket
    Key='users/123/profile.jpg',         # Object key (like a path)
    ExtraArgs={
        'ContentType': 'image/jpeg',     # MIME type
        'Metadata': {                    # Custom metadata
            'uploaded-by': 'user-123',
            'original-name': 'selfie.jpg'
        }
    }
)

# --- DOWNLOAD AN OBJECT ---
s3.download_file(
    Bucket='my-app-photos-2024',
    Key='users/123/profile.jpg',
    Filename='downloaded_photo.jpg'
)

# --- GENERATE A PRE-SIGNED URL (temporary access) ---
# Gives a time-limited URL for direct download without credentials
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-app-photos-2024', 'Key': 'users/123/profile.jpg'},
    ExpiresIn=3600  # URL valid for 1 hour
)
print(f"Shareable URL: {url}")

# --- LIST OBJECTS WITH PREFIX ---
response = s3.list_objects_v2(
    Bucket='my-app-photos-2024',
    Prefix='users/123/'  # Only objects starting with this prefix
)
for obj in response.get('Contents', []):
    print(f"  {obj['Key']} - {obj['Size']} bytes - {obj['LastModified']}")

# --- DELETE AN OBJECT ---
s3.delete_object(Bucket='my-app-photos-2024', Key='users/123/profile.jpg')
```

### Java — Using AWS S3 with AWS SDK v2

```java
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.GetObjectPresignRequest;

import java.nio.file.Path;
import java.time.Duration;

public class ObjectStorageExample {
    public static void main(String[] args) {
        // Initialize the S3 client
        S3Client s3 = S3Client.builder()
            .region(Region.US_EAST_1)
            .build();

        String bucket = "my-app-photos-2024";
        String key = "users/123/profile.jpg";

        // --- UPLOAD AN OBJECT ---
        s3.putObject(
            PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType("image/jpeg")
                .metadata(Map.of(
                    "uploaded-by", "user-123",
                    "original-name", "selfie.jpg"
                ))
                .build(),
            RequestBody.fromFile(Path.of("local_photo.jpg"))
        );

        // --- DOWNLOAD AN OBJECT ---
        s3.getObject(
            GetObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .build(),
            Path.of("downloaded_photo.jpg")
        );

        // --- GENERATE PRE-SIGNED URL ---
        S3Presigner presigner = S3Presigner.create();
        var presignedUrl = presigner.presignGetObject(
            GetObjectPresignRequest.builder()
                .signatureDuration(Duration.ofHours(1))
                .getObjectRequest(GetObjectRequest.builder()
                    .bucket(bucket)
                    .key(key)
                    .build())
                .build()
        );
        System.out.println("URL: " + presignedUrl.url());

        // --- LIST OBJECTS ---
        ListObjectsV2Response listing = s3.listObjectsV2(
            ListObjectsV2Request.builder()
                .bucket(bucket)
                .prefix("users/123/")
                .build()
        );
        listing.contents().forEach(obj ->
            System.out.printf("  %s - %d bytes%n", obj.key(), obj.size())
        );
    }
}
```

---

## Infrastructure Examples

### S3 Bucket with Lifecycle Policy (Terraform)

```hcl
resource "aws_s3_bucket" "app_data" {
  bucket = "my-app-data-prod"
}

# Automatically move old data to cheaper storage
resource "aws_s3_bucket_lifecycle_configuration" "app_data_lifecycle" {
  bucket = aws_s3_bucket.app_data.id

  rule {
    id     = "archive-old-data"
    status = "Enabled"

    # After 30 days → move to Infrequent Access (cheaper)
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    # After 90 days → move to Glacier (much cheaper)
    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    # After 365 days → delete
    expiration {
      days = 365
    }
  }
}

# Enable versioning (keep history of changes)
resource "aws_s3_bucket_versioning" "app_data_versioning" {
  bucket = aws_s3_bucket.app_data.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Block all public access (security!)
resource "aws_s3_bucket_public_access_block" "app_data_block" {
  bucket = aws_s3_bucket.app_data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Nginx Config for S3 Proxy (Private Bucket Access)

```nginx
# Proxy requests through Nginx to S3 (users never see S3 directly)
server {
    listen 443 ssl;
    server_name media.myapp.com;

    location /images/ {
        # Proxy to S3 bucket
        proxy_pass https://my-app-photos-2024.s3.amazonaws.com/;
        
        # Cache responses locally for 1 hour
        proxy_cache_valid 200 1h;
        proxy_cache images_cache;
        
        # Don't pass auth headers to S3
        proxy_set_header Authorization "";
        
        # Add cache headers for downstream CDN
        add_header Cache-Control "public, max-age=86400";
    }
}
```

---

## Real-World Example

### How Netflix Uses Object Storage

Netflix stores **petabytes** of video content in object storage:

```
┌─────────────────────────────────────────────────────────────┐
│                   NETFLIX VIDEO PIPELINE                     │
│                                                             │
│  Original Video    Transcoding Farm        Object Storage   │
│  (uploaded by      (convert to             (S3 / GCS)       │
│   studio)          multiple formats)                        │
│                                                             │
│  ┌──────────┐     ┌──────────────┐     ┌────────────────┐ │
│  │  4K RAW  │────▶│  Encode to:  │────▶│ Bucket:        │ │
│  │  Master  │     │  - 4K HDR    │     │ netflix-videos │ │
│  │  File    │     │  - 1080p     │     │                │ │
│  │ (100 GB) │     │  - 720p      │     │ /title-123/    │ │
│  └──────────┘     │  - 480p      │     │   /4k-hdr/     │ │
│                   │  - 360p      │     │     chunk-001  │ │
│                   │  - Audio:    │     │     chunk-002  │ │
│                   │    EN, ES,   │     │     ...        │ │
│                   │    FR, etc.  │     │   /1080p/      │ │
│                   └──────────────┘     │     chunk-001  │ │
│                                        │     ...        │ │
│                                        └────────────────┘ │
│                                              │             │
│                                              ▼             │
│                                        ┌────────────┐     │
│                                        │    CDN     │     │
│                                        │  (Open     │     │
│                                        │  Connect)  │     │
│                                        └────────────┘     │
│                                              │             │
│                                              ▼             │
│                                        ┌────────────┐     │
│                                        │  Your TV   │     │
│                                        └────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

**Key patterns:**
- Each video is stored in **multiple resolutions and codecs** (~1,200 files per title)
- Videos are split into **small chunks** (2-4 seconds each) for adaptive streaming
- Object keys are structured: `/{title-id}/{resolution}/{segment-number}`
- Lifecycle rules move old/unpopular content to cheaper storage tiers
- Pre-signed URLs allow CDN edge servers to fetch directly from S3

### How Instagram Stores Photos

- **500+ million photos uploaded per day**
- Each photo stored in multiple sizes (thumbnail, medium, full, original)
- Object keys encode user ID + timestamp for even distribution
- Metadata includes EXIF data, filters applied, geolocation

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Public buckets** | Data breaches (Capital One 2019) | Block all public access by default; use pre-signed URLs |
| **No lifecycle rules** | Storage costs grow forever | Set transition/expiration rules from day one |
| **Storing hot data in cold tiers** | Retrieval costs explode | Analyze access patterns; use S3 Analytics |
| **Flat key patterns** | Hot partitions (e.g., all keys start with "2024/") | Add random prefix or hash to distribute load |
| **No versioning** | Accidental deletes are permanent | Enable versioning + MFA delete for production |
| **Large single uploads** | Timeouts, no resume | Use multipart upload for files > 100 MB |
| **Ignoring consistency model** | Read-after-write bugs | S3 now provides strong consistency (since Dec 2020) |

### The Hot Partition Problem

```
BAD key pattern:                    GOOD key pattern:
─────────────────                   ──────────────────
2024/01/01/file1.jpg               a3f2/2024/01/01/file1.jpg
2024/01/01/file2.jpg               7b1c/2024/01/01/file2.jpg
2024/01/01/file3.jpg               e9d4/2024/01/01/file3.jpg
2024/01/01/file4.jpg               2a8f/2024/01/01/file4.jpg

All keys route to SAME partition!   Hash prefix distributes across
                                    many partitions
```

> **Note**: AWS S3 addressed this in 2018 by supporting 3,500+ PUT and 5,500+ GET requests per prefix per second. But for extreme throughput, distributing keys is still best practice.

---

## When to Use / When NOT to Use

### ✅ Use Object Storage When:

- Storing **unstructured data** (images, videos, documents, logs, backups)
- You need **unlimited scalability** without managing disks
- Data is mostly **write-once, read-many**
- You want built-in **durability and replication**
- Serving static content via **CDN integration**
- Building **data lakes** for analytics

### ❌ Do NOT Use Object Storage When:

- You need **low-latency random access** (< 1ms) → Use block storage / local SSD
- Data needs **frequent in-place updates** → Use a database
- You need a **POSIX file system** (file locking, appends) → Use EFS/NFS
- You need **transactional guarantees** → Use a relational database
- Real-time **streaming writes** (like a log that appends every ms) → Use Kafka or block storage

---

## Key Takeaways

- **Object storage** stores data as objects (data + metadata + key) in a flat namespace — no folders, no hierarchy.
- **Buckets** are top-level containers; **keys** identify objects within a bucket.
- Objects are accessed via **HTTP REST APIs** (PUT, GET, DELETE) — no mount required.
- Provides **extreme durability** (11 nines on S3) through replication and erasure coding across availability zones.
- **Storage classes** (Standard, IA, Glacier, Deep Archive) let you optimize cost based on access frequency.
- Use **pre-signed URLs** for secure, time-limited access without exposing credentials.
- **Lifecycle policies** automatically transition data to cheaper storage or delete it after a set period.

---

## What's Next?

Next, we'll explore the fundamental differences between the three storage paradigms: [Block Storage vs File Storage vs Object Storage](./02-storage-types.md) — understanding when to use each type and how they compare in terms of performance, use cases, and architecture.
