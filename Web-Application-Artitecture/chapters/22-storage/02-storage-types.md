# Block Storage vs File Storage vs Object Storage

> **What you'll learn**: The three fundamental ways to store data in computing, how they differ architecturally, their performance characteristics, and how to choose the right type for each use case.

---

## Real-Life Analogy

Think of three different ways to store books:

1. **Block Storage** = A **blank notebook** with numbered pages. You decide exactly what goes on each page. You can open page 47, erase it, and rewrite it instantly. Maximum control, maximum speed — but YOU manage everything.

2. **File Storage** = A **library with shelves and sections**. Books are organized in aisles → shelves → sections. Multiple people can browse the same library. You find a book by its location: "Aisle 3, Shelf 2, Position 5." There's a librarian (file system) managing it.

3. **Object Storage** = A **massive warehouse with a barcode system**. You hand your book to the warehouse with a unique barcode. To get it back, you give the barcode. You don't know or care WHERE in the warehouse it sits. It can store millions of books and never runs out of space.

```
BLOCK STORAGE              FILE STORAGE              OBJECT STORAGE
(Raw pages)                (Organized library)       (Warehouse + barcodes)

┌───┬───┬───┬───┐         /home/                    Bucket: "data"
│ 1 │ 2 │ 3 │ 4 │           /docs/                   key: "report.pdf"
├───┼───┼───┼───┤             report.pdf              key: "photo.jpg"
│ 5 │ 6 │ 7 │ 8 │           /photos/                  key: "video.mp4"
├───┼───┼───┼───┤             vacation.jpg
│ 9 │10 │11 │12 │
└───┴───┴───┴───┘         Tree hierarchy             Flat key-value

Fastest, lowest level      Shared, organized          Massive scale, HTTP API
```

---

## Core Concept Explained Step-by-Step

### The Three Storage Paradigms

```
┌─────────────────────────────────────────────────────────────────────┐
│                    STORAGE ABSTRACTION LEVELS                        │
│                                                                     │
│   High-Level    ┌──────────────────┐                               │
│   (Easy)        │  OBJECT STORAGE  │  HTTP API, metadata-rich      │
│                 │  (S3, GCS, Blob) │  infinite scale, eventual     │
│                 └────────┬─────────┘                               │
│                          │                                          │
│   Mid-Level     ┌────────▼─────────┐                               │
│   (Familiar)    │   FILE STORAGE   │  POSIX, hierarchy, shared     │
│                 │  (NFS, EFS, SMB) │  access, locks                │
│                 └────────┬─────────┘                               │
│                          │                                          │
│   Low-Level     ┌────────▼─────────┐                               │
│   (Raw)         │  BLOCK STORAGE   │  Raw blocks, OS formats it,  │
│                 │  (EBS, SAN, iSCSI)│  fastest, lowest latency     │
│                 └──────────────────┘                               │
│                                                                     │
│   Hardware      ┌──────────────────┐                               │
│                 │  PHYSICAL DISK   │  HDD / SSD / NVMe            │
│                 └──────────────────┘                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Block Storage — The Raw Building Blocks

### What Is It?

Block storage divides data into fixed-size **blocks** (typically 512 bytes or 4 KB). Each block has an address. The operating system's file system (ext4, NTFS, XFS) sits ON TOP of block storage and organizes blocks into files.

```
┌─────────────────────────────────────────────┐
│          YOUR APPLICATION                   │
├─────────────────────────────────────────────┤
│          FILE SYSTEM (ext4/XFS)             │
│   Translates files → block addresses        │
├─────────────────────────────────────────────┤
│          BLOCK STORAGE                      │
│                                             │
│  Block 0  │ Block 1  │ Block 2  │ Block 3  │
│  [header] │ [data  ] │ [data  ] │ [data  ] │
│           │          │          │          │
│  Block 4  │ Block 5  │ Block 6  │ Block 7  │
│  [data  ] │ [free  ] │ [free  ] │ [data  ] │
│                                             │
│  Each block = fixed size (4 KB typically)   │
└─────────────────────────────────────────────┘
```

### Key Characteristics

- **Lowest latency** — Direct read/write to specific blocks
- **Attached to ONE server** — Like an external hard drive (unless using shared SAN)
- **You format it** — Put any file system you want (ext4, XFS, NTFS)
- **Fixed capacity** — Must provision size upfront (e.g., 500 GB EBS volume)
- **Supports in-place edits** — Modify byte 1000 of a file without rewriting the whole file

### Cloud Examples

| Provider | Service | Description |
|----------|---------|-------------|
| AWS | EBS (Elastic Block Store) | Block volumes for EC2 instances |
| GCP | Persistent Disk | Block storage for Compute Engine |
| Azure | Managed Disks | Block storage for Azure VMs |

---

## File Storage — The Shared Filing Cabinet

### What Is It?

File storage presents data as a **hierarchy of files and directories**, accessed through a file system protocol (NFS, SMB/CIFS). Multiple clients can mount the same file system simultaneously.

```
┌─────────────────────────────────────────────┐
│            NFS/SMB FILE SERVER               │
│                                             │
│  /shared/                                   │
│    ├── projects/                            │
│    │     ├── project-a/                     │
│    │     │     ├── design.fig              │
│    │     │     └── specs.docx              │
│    │     └── project-b/                     │
│    │           └── report.pdf              │
│    └── team/                               │
│          ├── alice/                         │
│          └── bob/                           │
│                                             │
│  PROTOCOL: NFS v4 / SMB 3.0               │
│  ACCESS: Multiple clients simultaneously   │
│  LOCKING: File-level locks supported       │
└─────────────────────────────────────────────┘

     ┌────────┐  ┌────────┐  ┌────────┐
     │Client 1│  │Client 2│  │Client 3│
     │(mount) │  │(mount) │  │(mount) │
     └────┬───┘  └────┬───┘  └────┬───┘
          │           │           │
          └───────────┼───────────┘
                      │
                      ▼
             ┌────────────────┐
             │  NFS/SMB Share │
             │  (shared file  │
             │   system)      │
             └────────────────┘
```

### Key Characteristics

- **Hierarchical** — Directories within directories (tree structure)
- **Shared access** — Multiple servers/clients mount simultaneously
- **POSIX compliant** — Supports locks, permissions, symlinks, appends
- **Familiar interface** — Apps use normal file I/O (open, read, write, close)
- **Moderate latency** — Network hop adds some latency vs. block storage

### Cloud Examples

| Provider | Service | Description |
|----------|---------|-------------|
| AWS | EFS (Elastic File System) | Managed NFS for EC2 |
| GCP | Filestore | Managed NFS for GCE |
| Azure | Azure Files | Managed SMB/NFS shares |

---

## Object Storage — The Infinite Warehouse

### What Is It?

Object storage manages data as **objects** in a flat namespace. Each object = data + metadata + unique key. Accessed via HTTP REST APIs. No file system, no hierarchy (the "/" in keys is just a convention).

> Covered in depth in Chapter 22.1: [Object Storage (S3, GCS, Azure Blob)](./01-object-storage.md)

---

## Head-to-Head Comparison

### Architecture Comparison

```
┌───────────────────┬───────────────────┬───────────────────┐
│   BLOCK STORAGE   │   FILE STORAGE    │  OBJECT STORAGE   │
├───────────────────┼───────────────────┼───────────────────┤
│                   │                   │                   │
│  App              │  App              │  App              │
│   │               │   │               │   │               │
│   ▼               │   ▼               │   ▼               │
│  File System      │  NFS/SMB Client   │  HTTP Client      │
│   │               │   │               │   │               │
│   ▼               │   ▼               │   ▼               │
│  Block Device     │  Network          │  REST API         │
│   │               │   │               │   │               │
│   ▼               │   ▼               │   ▼               │
│  [Physical Disk]  │  [File Server]    │  [Object Store]   │
│                   │                   │                   │
│  Attached to ONE  │  Shared by MANY   │  Accessed by ALL  │
│  server           │  servers          │  via HTTP         │
└───────────────────┴───────────────────┴───────────────────┘
```

### Detailed Feature Comparison

| Feature | Block Storage | File Storage | Object Storage |
|---------|-------------|-------------|---------------|
| **Data unit** | Fixed-size blocks (4KB) | Files in directories | Objects (data + metadata) |
| **Namespace** | Flat (block addresses) | Hierarchical (tree) | Flat (bucket + key) |
| **Protocol** | iSCSI, FC, NVMe | NFS, SMB/CIFS | HTTP/HTTPS REST |
| **Latency** | < 1 ms (SSD) | 1–10 ms | 10–100 ms (first byte) |
| **Throughput** | Very high (GB/s) | High (hundreds MB/s) | Very high (parallel GETs) |
| **Max size** | TB range (per volume) | PB (managed services) | Unlimited |
| **Scalability** | Limited (resize manually) | Moderate | Infinite |
| **Shared access** | No (1 server) | Yes (many clients) | Yes (HTTP from anywhere) |
| **Modify in place** | Yes (byte-level) | Yes (file-level) | No (replace entire object) |
| **Cost** | $$$ (highest) | $$ (medium) | $ (lowest at scale) |
| **Durability** | Depends on RAID/replication | Depends on setup | 99.999999999% (built-in) |
| **Use case** | Databases, OS boot | Shared documents, CMS | Media, backups, data lakes |

### Performance Characteristics

```
LATENCY (lower is better)
──────────────────────────────────────────────────
Block (NVMe SSD)  │████ < 0.1 ms
Block (EBS gp3)   │████████ ~ 1 ms
File (NFS)        │████████████████ ~ 2-10 ms
Object (S3)       │████████████████████████████████████ ~ 50-200 ms (first byte)


THROUGHPUT (higher is better)
──────────────────────────────────────────────────
Block (io2)       │████████████████████████████████ 4 GB/s per volume
File (EFS perf)   │████████████████████ ~3 GB/s (burst)
Object (S3)       │██████████████████████████████████████ No limit (parallel)


MAX IOPS
──────────────────────────────────────────────────
Block (io2 BX)    │████████████████████████████████ 256,000 IOPS
File (EFS)        │████████████████ ~55,000 IOPS
Object (S3)       │████████ 5,500 GET / 3,500 PUT per prefix/sec
```

---

## How It Works Internally

### Block Storage Internals

```
APPLICATION: "Write 'Hello' to file at offset 1000"
                │
                ▼
FILE SYSTEM (ext4):
  1. Look up file's inode (metadata: block map)
  2. Find which block contains offset 1000
     offset 1000 ÷ 4096 (block size) = block index 0
     offset within block = 1000
  3. Issue write to block device
                │
                ▼
BLOCK DEVICE DRIVER:
  4. Translate logical block → physical sector
  5. Issue I/O command to disk controller
                │
                ▼
PHYSICAL DISK:
  6. Write data to physical medium
     (HDD: move head → SSD: write to NAND cell)
```

### File Storage Internals (NFS)

```
CLIENT APP: open("/shared/report.pdf")
                │
                ▼
NFS CLIENT (kernel):
  1. Send LOOKUP RPC to NFS server
  2. Server checks permissions, returns file handle
                │
                ▼
CLIENT APP: read(fd, buffer, 4096)
                │
                ▼
NFS CLIENT:
  3. Send READ RPC (file_handle, offset=0, count=4096)
                │
                ▼
NFS SERVER:
  4. Find file on local block storage
  5. Read data from disk
  6. Return data over network
                │
                ▼
CLIENT APP: receives data in buffer

NETWORK OVERHEAD:
  Each operation = network round-trip
  Caching (client-side) reduces latency
  Lock manager (NLM) handles concurrent access
```

### Object Storage Internals

```
CLIENT APP: PUT object

HTTP REQUEST:
  PUT /bucket/photos/cat.jpg HTTP/1.1
  Host: s3.amazonaws.com
  Content-Length: 5242880
  Content-Type: image/jpeg
  x-amz-meta-photographer: alice
  Authorization: AWS4-HMAC-SHA256 ...
  
  [5 MB of image data]

OBJECT STORE INTERNALS:
  1. Authenticate & authorize request
  2. Compute content hash (integrity check)
  3. Split into chunks (if large)
  4. Write chunks to multiple storage nodes
  5. Store metadata in distributed index
  6. Replicate across availability zones
  7. Return HTTP 200 with ETag (MD5 hash)

RESPONSE:
  HTTP/1.1 200 OK
  ETag: "d41d8cd98f00b204e9800998ecf8427e"
  x-amz-request-id: ABC123
```

---

## Code Examples

### Python — Demonstrating All Three Storage Types

```python
# === BLOCK STORAGE (simulated as raw device I/O) ===
# In practice, you'd format it with a filesystem first.
# This shows direct block-level access (rare in application code):

import os

# Write directly to a block device (Linux, requires root)
# fd = os.open('/dev/sdb', os.O_WRONLY | os.O_DIRECT)
# os.lseek(fd, 4096 * 10, os.SEEK_SET)  # Seek to block 10
# os.write(fd, data_block)
# os.close(fd)

# More realistically, block storage is mounted and used as a filesystem:
with open('/mnt/ebs-volume/database.db', 'rb+') as f:
    f.seek(1000)                # Random access at byte level
    f.write(b'updated data')   # In-place modification
    f.seek(0)
    header = f.read(100)       # Read specific bytes


# === FILE STORAGE (NFS mount — used like local files) ===
import shutil

# NFS share mounted at /mnt/shared (via: mount -t nfs server:/share /mnt/shared)
shared_path = '/mnt/shared/team-docs/quarterly-report.docx'

# Multiple servers can read/write the same file
with open(shared_path, 'a') as f:   # Append mode
    f.write("New section added by Server B\n")

# Copy file within shared filesystem
shutil.copy(shared_path, '/mnt/shared/backups/report-backup.docx')


# === OBJECT STORAGE (HTTP API via boto3) ===
import boto3

s3 = boto3.client('s3')

# Upload — whole object at once (no partial write)
s3.put_object(
    Bucket='my-bucket',
    Key='reports/2024/q1-report.pdf',
    Body=open('report.pdf', 'rb'),
    Metadata={'author': 'alice', 'department': 'engineering'}
)

# Download — whole object at once (or range GET for partial)
response = s3.get_object(Bucket='my-bucket', Key='reports/2024/q1-report.pdf')
data = response['Body'].read()
```

### Java — Comparing Access Patterns

```java
import java.io.*;
import java.nio.*;
import java.nio.file.*;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;

public class StorageComparison {

    // === BLOCK STORAGE (via mounted filesystem) ===
    static void blockStorageExample() throws IOException {
        // EBS volume mounted at /data — used like local disk
        RandomAccessFile raf = new RandomAccessFile("/data/db/table.dat", "rw");
        
        // RANDOM ACCESS: Jump to any byte position instantly
        raf.seek(4096 * 100);  // Go to block 100
        byte[] block = new byte[4096];
        raf.readFully(block);   // Read exactly one block
        
        // IN-PLACE MODIFICATION: Change specific bytes
        raf.seek(4096 * 100);
        block[0] = 0x01;       // Modify first byte
        raf.write(block);      // Write back same block
        raf.close();
    }

    // === FILE STORAGE (NFS/SMB share) ===
    static void fileStorageExample() throws IOException {
        // Shared NFS mount — multiple servers access same files
        Path sharedFile = Paths.get("/mnt/nfs-share/config/app.properties");
        
        // Read from shared location
        String content = Files.readString(sharedFile);
        
        // Write — NFS handles locking between clients
        Files.writeString(sharedFile, content + "\nnew.setting=true",
            StandardOpenOption.APPEND);
        
        // File operations work normally (rename, delete, list)
        Files.list(Paths.get("/mnt/nfs-share/config/"))
            .forEach(p -> System.out.println(p.getFileName()));
    }

    // === OBJECT STORAGE (S3 via HTTP) ===
    static void objectStorageExample() {
        S3Client s3 = S3Client.create();
        
        // PUT: Store object (atomic, whole-object write)
        s3.putObject(
            PutObjectRequest.builder()
                .bucket("my-bucket")
                .key("data/report.pdf")
                .contentType("application/pdf")
                .build(),
            Path.of("report.pdf")
        );
        
        // GET: Retrieve object (or use Range header for partial)
        s3.getObject(
            GetObjectRequest.builder()
                .bucket("my-bucket")
                .key("data/report.pdf")
                .build(),
            Path.of("downloaded-report.pdf")
        );
        
        // No rename! Must copy + delete
        s3.copyObject(CopyObjectRequest.builder()
            .sourceBucket("my-bucket").sourceKey("data/report.pdf")
            .destinationBucket("my-bucket").destinationKey("data/report-v2.pdf")
            .build());
        s3.deleteObject(DeleteObjectRequest.builder()
            .bucket("my-bucket").key("data/report.pdf")
            .build());
    }
}
```

---

## Infrastructure Examples

### AWS Architecture Using All Three Storage Types

```
┌─────────────────────────────────────────────────────────────────┐
│                   TYPICAL WEB APP ARCHITECTURE                   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐      │
│  │                    EC2 Instance                       │      │
│  │                                                      │      │
│  │  ┌─────────────┐    ┌──────────────────────────┐    │      │
│  │  │  App Server │    │  PostgreSQL Database      │    │      │
│  │  │  (Django)   │    │                          │    │      │
│  │  └──────┬──────┘    └────────────┬─────────────┘    │      │
│  │         │                        │                   │      │
│  └─────────┼────────────────────────┼───────────────────┘      │
│            │                        │                           │
│            │                        ▼                           │
│            │              ┌──────────────────┐                  │
│            │              │  EBS Volume      │  ◀── BLOCK       │
│            │              │  (gp3, 500 GB)   │      STORAGE     │
│            │              │  /data/pgdata/   │                  │
│            │              │  Low latency     │                  │
│            │              │  High IOPS       │                  │
│            │              └──────────────────┘                  │
│            │                                                    │
│            ▼                                                    │
│  ┌──────────────────┐                                          │
│  │  EFS Mount       │  ◀── FILE STORAGE                        │
│  │  /mnt/shared/    │      Shared config, templates           │
│  │  Multiple EC2s   │      Multiple instances mount it        │
│  │  mount this      │                                          │
│  └──────────────────┘                                          │
│                                                                 │
│            │                                                    │
│            ▼                                                    │
│  ┌──────────────────┐                                          │
│  │  S3 Bucket       │  ◀── OBJECT STORAGE                      │
│  │  User uploads    │      Photos, videos, documents          │
│  │  Static assets   │      Unlimited, cheap, durable          │
│  │  Backups         │                                          │
│  └──────────────────┘                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes Storage Classes

```yaml
# Block Storage — for databases requiring fast I/O
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-block-storage
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "50"         # High IOPS for DB workloads
  fsType: xfs
volumeBindingMode: WaitForFirstConsumer

---
# File Storage — for shared access across pods
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shared-file-storage
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345abc
  directoryPerms: "700"

---
# Usage in a Pod (Block — exclusive access)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  storageClassName: fast-block-storage
  accessModes: [ReadWriteOnce]      # Only ONE pod can mount
  resources:
    requests:
      storage: 100Gi

---
# Usage in a Pod (File — shared across pods)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-config
spec:
  storageClassName: shared-file-storage
  accessModes: [ReadWriteMany]      # MANY pods can mount
  resources:
    requests:
      storage: 10Gi
```

---

## Real-World Example

### How Companies Choose Storage Types

```
┌──────────────────────────────────────────────────────────────┐
│                STORAGE DECISIONS AT SCALE                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  UBER                                                        │
│  ├── Block:   MySQL/PostgreSQL data files (EBS io2)         │
│  ├── File:    Shared ML model files across training pods    │
│  └── Object:  Trip receipts, driver documents (S3)          │
│                                                              │
│  NETFLIX                                                     │
│  ├── Block:   Cassandra SSTables (local NVMe SSDs)          │
│  ├── File:    Shared encoding job configs                   │
│  └── Object:  ALL video content + backups (S3)              │
│                                                              │
│  SLACK                                                       │
│  ├── Block:   MySQL databases (EBS)                         │
│  ├── File:    Not heavily used                              │
│  └── Object:  All file uploads from users (S3)             │
│                                                              │
│  PINTEREST                                                   │
│  ├── Block:   HBase/MySQL (high IOPS)                       │
│  ├── File:    ML pipeline shared data                       │
│  └── Object:  Billions of pins/images (S3)                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Using object storage for databases** | Too high latency; no random byte access | Use block storage (EBS/Persistent Disk) for DBs |
| **Using block storage for media files** | Expensive, limited size, not shareable | Use object storage — cheaper at scale |
| **Using NFS for high-IOPS workloads** | Network latency kills performance | Use local NVMe or provisioned IOPS EBS |
| **Not matching storage to workload** | Overpaying or underperforming | Analyze access pattern (random vs. sequential, hot vs. cold) |
| **Assuming file storage = POSIX everywhere** | Cloud file storage has limits (no hard links, different lock semantics) | Test your app's I/O patterns against the specific service |
| **Ignoring the cost model** | Block: pay per GB provisioned; Object: pay per GB stored + requests | Right-size block volumes; use object lifecycle policies |

### Cost Comparison (AWS, approximate)

```
MONTHLY COST PER TB:
──────────────────────────────────────────────────
Block (EBS gp3)           │████████████████████████████ $80/TB/month
Block (EBS io2)           │████████████████████████████████████████████ $125/TB/month
File (EFS Standard)       │████████████████████████████████████████████████████ $300/TB/month
Object (S3 Standard)      │████████ $23/TB/month
Object (S3 Glacier)       │██ $4/TB/month
Object (S3 Deep Archive)  │█ $1/TB/month

Note: File storage is expensive because it provides shared access + POSIX semantics.
Object storage is cheapest because it has simplest API and highest economies of scale.
```

---

## When to Use / When NOT to Use

### Decision Matrix

```
"What am I storing and how will I access it?"

                    ┌─────────────────────────────┐
                    │  Need low-latency random    │
                    │  access? (< 1ms)            │
                    └──────────────┬──────────────┘
                           YES     │      NO
                    ┌──────────────┘      │
                    ▼                     ▼
            ┌──────────────┐    ┌─────────────────────┐
            │ BLOCK STORAGE│    │ Need shared access  │
            │ (Database,   │    │ by multiple servers?│
            │  boot disk)  │    └──────────┬──────────┘
            └──────────────┘        YES    │     NO
                                ┌──────────┘     │
                                ▼                ▼
                        ┌──────────────┐  ┌──────────────┐
                        │FILE STORAGE  │  │Need POSIX?   │
                        │(Shared CMS,  │  │(file locks,  │
                        │ config files)│  │ appends)     │
                        └──────────────┘  └──────┬───────┘
                                            YES  │   NO
                                          ┌──────┘   │
                                          ▼          ▼
                                   ┌───────────┐ ┌──────────────┐
                                   │   FILE    │ │OBJECT STORAGE│
                                   │  STORAGE  │ │(Media, logs, │
                                   └───────────┘ │ backups,     │
                                                 │ data lake)   │
                                                 └──────────────┘
```

### Quick Reference

| Use Case | Best Storage | Why |
|----------|-------------|-----|
| PostgreSQL / MySQL data | **Block** | Needs random I/O, IOPS, low latency |
| Boot volumes / OS disk | **Block** | OS needs raw block device |
| User-uploaded images | **Object** | Write-once, read-many, unlimited scale |
| Log files (archived) | **Object** | Cheap, durable, lifecycle policies |
| Shared config across servers | **File** | Multiple nodes need same files |
| ML training data (shared) | **File** | Multiple GPUs/workers read same dataset |
| Video streaming content | **Object** | Massive, CDN-friendly, cost-effective |
| Application binaries/artifacts | **Object** | Versioned, widely accessible |
| Real-time analytics DB | **Block** | Needs consistent low-latency IOPS |

---

## Key Takeaways

- **Block storage** is the raw, lowest-level storage. Think hard drives. Fastest, lowest latency, used for databases and OS disks. Attached to ONE server.
- **File storage** adds a hierarchy (folders/files) and shared access. Think network drives. Used when multiple servers need the same files (shared config, CMS content).
- **Object storage** uses a flat key-value model with HTTP APIs. Think infinite warehouse. Used for unstructured data at massive scale (media, logs, backups).
- **Choose based on access pattern**: random byte-level → block, shared files → file, write-once/read-many at scale → object.
- **Cost decreases** as you move from block → file → object, but **latency increases**.
- Most real applications use **all three types together** — databases on block, shared configs on file, user uploads on object.
- Never use object storage where you need **in-place byte modifications** or **sub-millisecond latency**.

---

## What's Next?

Next, we'll go deep into how planet-scale companies store petabytes of data across thousands of machines with [Distributed File Systems (HDFS, GFS)](./03-distributed-file-systems.md) — the systems that power Google Search, Hadoop, and big data processing.
