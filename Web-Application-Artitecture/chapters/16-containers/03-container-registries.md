# Container Registries (Docker Hub, ECR, GCR)

> **What you'll learn**: How container images are stored, shared, and distributed through registries — the "app stores" for Docker images — including security scanning, access control, and production best practices.

---

## Real-Life Analogy: A Library for Container Images

Think of a **container registry** like a **library**:

- **Books** = Container images (the things you want to use)
- **Library catalog** = Registry index (searchable list of available images)
- **ISBN** = Image tag/digest (unique identifier for a specific version)
- **Library card** = Authentication credentials (who can check out books)
- **Public library** = Docker Hub (anyone can browse and borrow)
- **Private library** = ECR/GCR (only your organization has access)

```
Developer builds image ──push──▶  Registry  ◀──pull──  Production Server
                                    │
                        ┌───────────┴───────────┐
                        │  Stores layers         │
                        │  Manages tags          │
                        │  Controls access       │
                        │  Scans vulnerabilities │
                        └───────────────────────┘
```

---

## What is a Container Registry?

A container registry is a **storage and distribution system** for container images. It's where you:

1. **Push** images after building them
2. **Pull** images when deploying to servers
3. **Tag** images with version numbers
4. **Scan** images for security vulnerabilities
5. **Control** who can access which images

```
┌─────────────────────── CI/CD Pipeline ───────────────────────────┐
│                                                                    │
│  Code Push ──▶ Build ──▶ Test ──▶ Push to Registry ──▶ Deploy    │
│                                        │                          │
│                              ┌─────────┴─────────┐               │
│                              │                    │               │
│                              ▼                    ▼               │
│                    ┌──────────────┐      ┌──────────────┐        │
│                    │  Docker Hub  │      │   AWS ECR    │        │
│                    │  (public)    │      │  (private)   │        │
│                    └──────────────┘      └──────────────┘        │
│                              │                    │               │
│                              ▼                    ▼               │
│                    ┌──────────────┐      ┌──────────────┐        │
│                    │ Open Source  │      │  Production  │        │
│                    │ Consumers    │      │  Kubernetes  │        │
│                    └──────────────┘      └──────────────┘        │
└───────────────────────────────────────────────────────────────────┘
```

---

## Image Naming Convention

```
[registry-url/] [namespace/] repository : tag @ sha256:digest

Examples:
──────────────────────────────────────────────────────────────────────
docker.io/library/nginx:1.25              ← Docker Hub official image
docker.io/mycompany/webapp:v2.1.0         ← Docker Hub user image
123456789.dkr.ecr.us-east-1.amazonaws.com/webapp:latest  ← AWS ECR
gcr.io/my-project/api-server:abc123       ← Google Container Registry
ghcr.io/myorg/service:v1.0.0             ← GitHub Container Registry
registry.example.com/team/app:v3.2        ← Self-hosted registry
```

**Breakdown:**
```
┌──────────────────────────────────────────────────────┐
│  123456789.dkr.ecr.us-east-1.amazonaws.com/webapp:v2 │
│  └──────────── registry ──────────────────┘└─repo─┘└tag┘
│                                                       │
│  If no registry specified → defaults to docker.io     │
│  If no tag specified → defaults to :latest            │
└──────────────────────────────────────────────────────┘
```

---

## Major Container Registries Compared

```
┌──────────────────────────────────────────────────────────────────────┐
│                    REGISTRY LANDSCAPE                                  │
│                                                                       │
│  ┌─── Public ──────────┐    ┌─── Cloud Provider ────────────────┐   │
│  │                      │    │                                    │   │
│  │  Docker Hub          │    │  AWS ECR (Elastic Container Reg.) │   │
│  │  GitHub GHCR         │    │  Google GCR / Artifact Registry   │   │
│  │  Quay.io (RedHat)    │    │  Azure ACR (Container Registry)   │   │
│  │                      │    │  Alibaba CR                        │   │
│  └──────────────────────┘    └────────────────────────────────────┘   │
│                                                                       │
│  ┌─── Self-Hosted ─────┐    ┌─── Enterprise ────────────────────┐   │
│  │                      │    │                                    │   │
│  │  Harbor (CNCF)       │    │  JFrog Artifactory                │   │
│  │  Docker Registry     │    │  Sonatype Nexus                   │   │
│  │  GitLab Registry     │    │  VMware Harbor                    │   │
│  │                      │    │                                    │   │
│  └──────────────────────┘    └────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

| Registry | Best For | Free Tier | Key Feature |
|----------|----------|-----------|-------------|
| **Docker Hub** | Open source, personal projects | 1 private repo | Largest public library |
| **AWS ECR** | AWS-based workloads | 500 MB/month | IAM integration, replication |
| **Google AR** | GCP workloads | 500 MB/month | Vulnerability scanning |
| **Azure ACR** | Azure workloads | - | Geo-replication, tasks |
| **GitHub GHCR** | GitHub-hosted projects | Unlimited public | GitHub Actions integration |
| **Harbor** | Self-hosted, air-gapped | Open source | RBAC, scanning, replication |
| **JFrog** | Enterprise, multi-format | Community edition | Universal artifact management |

---

## How It Works Internally

### Registry Protocol (Docker Registry HTTP API v2)

```
Client                          Registry
  │                                │
  │──── GET /v2/ ─────────────────▶│  Check if registry is available
  │◀─── 200 OK ───────────────────│
  │                                │
  │──── HEAD /v2/repo/manifests/tag▶│  Check if image exists
  │◀─── 200 + Content-Length ──────│
  │                                │
  │──── GET /v2/repo/manifests/tag─▶│  Download image manifest
  │◀─── Manifest JSON ─────────────│  (list of layers)
  │                                │
  │  For each layer:               │
  │──── HEAD /v2/repo/blobs/sha256 ▶│  Check if layer exists
  │◀─── 200 (exists) or 404 ───────│
  │                                │
  │──── GET /v2/repo/blobs/sha256 ─▶│  Download layer
  │◀─── Layer binary data ─────────│
  │                                │
```

### Image Manifest Structure

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 7023,
    "digest": "sha256:abc123..."
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 32654,
      "digest": "sha256:layer1hash..."
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 16724,
      "digest": "sha256:layer2hash..."
    }
  ]
}
```

### Content-Addressable Storage

```
Registry Storage Backend (S3, GCS, etc.)
├── blobs/
│   └── sha256/
│       ├── abc123...  (layer 1 - Ubuntu base - 77 MB)
│       ├── def456...  (layer 2 - Python install - 200 MB)
│       └── ghi789...  (layer 3 - App code - 5 KB)
├── manifests/
│   └── myapp/
│       ├── v1.0.0     (points to layers abc + def + ghi)
│       ├── v1.0.1     (points to layers abc + def + NEW)  ← shares 2 layers!
│       └── latest     (pointer to v1.0.1)
```

> **Key Insight**: Layers are stored by their **content hash** (SHA256). If two images share the same base layer, it's stored **only once**. This saves enormous storage and bandwidth.

---

## Code Examples

### Python — Interacting with Registries Programmatically

```python
import docker
import json

# Connect to Docker daemon
client = docker.from_env()

# Pull an image from Docker Hub
print("Pulling nginx:alpine...")
image = client.images.pull("nginx", tag="alpine")
print(f"Image ID: {image.id}")
print(f"Tags: {image.tags}")
print(f"Size: {image.attrs['Size'] / 1024 / 1024:.1f} MB")

# Build an image from Dockerfile
print("\nBuilding custom image...")
image, build_logs = client.images.build(
    path="./app",
    tag="mycompany/webapp:v1.0.0",
    rm=True,           # Remove intermediate containers
    pull=True,         # Pull latest base image
)

# Tag for a private registry (AWS ECR)
ecr_url = "123456789.dkr.ecr.us-east-1.amazonaws.com"
image.tag(f"{ecr_url}/webapp", tag="v1.0.0")
image.tag(f"{ecr_url}/webapp", tag="latest")

# Push to registry
print("\nPushing to ECR...")
for line in client.images.push(f"{ecr_url}/webapp", tag="v1.0.0", stream=True, decode=True):
    if 'status' in line:
        print(f"  {line['status']}")

# List local images
print("\nLocal images:")
for img in client.images.list():
    for tag in img.tags:
        print(f"  {tag} ({img.short_id})")
```

### Java — Using Jib for Registry-Native Builds (No Docker Required!)

```java
// build.gradle (Gradle plugin — builds container WITHOUT Docker daemon!)
plugins {
    id 'com.google.cloud.tools.jib' version '3.4.0'
}

jib {
    from {
        image = 'eclipse-temurin:17-jre-alpine'
    }
    to {
        image = '123456789.dkr.ecr.us-east-1.amazonaws.com/my-service'
        tags = ['v1.0.0', 'latest']
        credHelper = 'ecr-login'  // AWS credential helper
    }
    container {
        mainClass = 'com.example.Application'
        jvmFlags = ['-Xms256m', '-Xmx512m', '-XX:+UseG1GC']
        ports = ['8080']
        environment = ['SPRING_PROFILES_ACTIVE': 'prod']
        user = '1000:1000'  // Non-root
        creationTime = 'USE_CURRENT_TIMESTAMP'
    }
}
```

```bash
# Build and push directly to registry (no Docker daemon needed!)
./gradlew jib

# Or build to local Docker daemon
./gradlew jibDockerBuild
```

---

## Infrastructure Examples

### AWS ECR Setup

```bash
# Create ECR repository
aws ecr create-repository \
  --repository-name myapp/webapp \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS

# Login to ECR (token valid for 12 hours)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Push image
docker tag webapp:v1.0 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp/webapp:v1.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp/webapp:v1.0

# Set lifecycle policy (auto-delete old images)
aws ecr put-lifecycle-policy \
  --repository-name myapp/webapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 10 images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v"],
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": { "type": "expire" }
      },
      {
        "rulePriority": 2,
        "description": "Delete untagged after 7 days",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 7
        },
        "action": { "type": "expire" }
      }
    ]
  }'
```

### Google Artifact Registry Setup

```bash
# Create repository
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production images"

# Configure Docker to use gcloud as credential helper
gcloud auth configure-docker us-central1-docker.pkg.dev

# Tag and push
docker tag webapp:v1.0 us-central1-docker.pkg.dev/my-project/my-repo/webapp:v1.0
docker push us-central1-docker.pkg.dev/my-project/my-repo/webapp:v1.0

# Scan for vulnerabilities
gcloud artifacts docker images scan \
  us-central1-docker.pkg.dev/my-project/my-repo/webapp:v1.0
```

### Self-Hosted Harbor Registry

```yaml
# harbor-values.yaml for Helm deployment
expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: harbor-tls
  ingress:
    hosts:
      core: registry.mycompany.com

persistence:
  persistentVolumeClaim:
    registry:
      size: 500Gi
    database:
      size: 10Gi

# Security scanning
trivy:
  enabled: true

# Replication to backup registry
replication:
  enabled: true
```

```bash
# Deploy Harbor with Helm
helm install harbor harbor/harbor -f harbor-values.yaml -n harbor
```

---

## Security: Image Scanning & Signing

```
┌──────────────────── Secure Image Pipeline ──────────────────────┐
│                                                                   │
│  Build ──▶ Scan ──▶ Sign ──▶ Push ──▶ Verify ──▶ Deploy        │
│              │        │                   │                       │
│              ▼        ▼                   ▼                       │
│         ┌────────┐ ┌────────┐      ┌──────────┐                 │
│         │ Trivy  │ │ Cosign │      │ Admission│                 │
│         │ Grype  │ │ Notary │      │ Controller│                │
│         │ Snyk   │ │        │      │ (verify  │                 │
│         └────────┘ └────────┘      │  sigs)   │                 │
│                                    └──────────┘                 │
└──────────────────────────────────────────────────────────────────┘
```

### Vulnerability Scanning with Trivy

```bash
# Scan an image for vulnerabilities
trivy image myapp:v1.0

# Output:
# myapp:v1.0 (alpine 3.18)
# ═══════════════════════════════════════
# Total: 3 (HIGH: 1, MEDIUM: 2)
#
# ┌────────────┬──────────┬──────────┬───────────────────┐
# │  Library   │  Vuln ID │ Severity │    Fixed Version  │
# ├────────────┼──────────┼──────────┼───────────────────┤
# │  libcrypto │ CVE-2024 │   HIGH   │    3.1.4-r1       │
# │  libssl    │ CVE-2024 │  MEDIUM  │    3.1.4-r1       │
# └────────────┴──────────┴──────────┴───────────────────┘

# Scan and fail CI if HIGH vulnerabilities found
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:v1.0
```

### Image Signing with Cosign

```bash
# Generate signing key
cosign generate-key-pair

# Sign the image after push
cosign sign --key cosign.key myregistry.com/myapp:v1.0

# Verify signature before deploying
cosign verify --key cosign.pub myregistry.com/myapp:v1.0
```

---

## Real-World Example: How GitHub Uses Container Registries

GitHub Container Registry (GHCR) is used internally and externally:

```
┌─────────────── GitHub Actions CI/CD ─────────────────────┐
│                                                           │
│  on: push to main                                        │
│                                                           │
│  jobs:                                                    │
│    build-and-push:                                       │
│      ┌───────────────────────────────────────────┐       │
│      │ 1. Checkout code                          │       │
│      │ 2. Set up Docker Buildx                   │       │
│      │ 3. Login to GHCR                          │       │
│      │ 4. Build multi-arch image (amd64 + arm64) │       │
│      │ 5. Scan with Trivy                        │       │
│      │ 6. Push to ghcr.io/org/app:sha-abc123     │       │
│      │ 7. Sign with Cosign (keyless!)            │       │
│      └───────────────────────────────────────────┘       │
│                          │                                │
│                          ▼                                │
│      ┌───────────────────────────────────────────┐       │
│      │  ghcr.io/org/app:sha-abc123               │       │
│      │  ghcr.io/org/app:v2.1.0                   │       │
│      │  ghcr.io/org/app:latest                   │       │
│      └───────────────────────────────────────────┘       │
└───────────────────────────────────────────────────────────┘
```

### GitHub Actions Workflow for Registry Push

```yaml
name: Build and Push

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
```

---

## Common Mistakes / Pitfalls

| Mistake | Problem | Solution |
|---------|---------|----------|
| Using `:latest` in production | Can't reproduce deployments | Use immutable tags (`:v1.2.3` or `:sha-abc123`) |
| No lifecycle policies | Registry fills up, costs spike | Auto-delete untagged/old images |
| Pulling from public in prod | Rate limits, supply chain risk | Mirror/cache in private registry |
| No vulnerability scanning | Deploying images with known CVEs | Scan on push + admission control |
| Shared credentials | Hard to audit, revoke access | Per-service/per-team credentials |
| No geo-replication | Slow pulls in other regions | Replicate to deployment regions |
| Large images | Slow CI/CD, slow scaling | Multi-stage builds, Alpine bases |
| Not using digest pinning | Tag can be overwritten (mutable) | Reference by `@sha256:...` for critical deployments |

---

## When to Use / When NOT to Use

### Registry Choice Decision Matrix

```
┌─────────────────────────────────────────────────────────┐
│                  Which Registry?                          │
│                                                          │
│  ┌─── Open Source / Side Project ───┐                   │
│  │  → Docker Hub (free public repos)│                   │
│  │  → GHCR (GitHub-integrated)      │                   │
│  └──────────────────────────────────┘                   │
│                                                          │
│  ┌─── Company on AWS ──────────────┐                    │
│  │  → AWS ECR (IAM, CloudTrail)    │                    │
│  └─────────────────────────────────┘                    │
│                                                          │
│  ┌─── Company on GCP ──────────────┐                    │
│  │  → Artifact Registry             │                    │
│  └──────────────────────────────────┘                    │
│                                                          │
│  ┌─── Multi-Cloud / Air-Gapped ───┐                    │
│  │  → Harbor (self-hosted, CNCF)   │                    │
│  └─────────────────────────────────┘                    │
│                                                          │
│  ┌─── Enterprise (many artifact types) ─┐              │
│  │  → JFrog Artifactory                  │              │
│  └────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **A registry is the "warehouse" for container images** — it stores, versions, and distributes them securely
2. **Images are stored as content-addressable layers** — shared layers save storage and speed up pulls
3. **Never use `:latest` in production** — use immutable tags like `v1.2.3` or Git SHA-based tags
4. **Always scan images for vulnerabilities** before deploying (Trivy, Grype, Snyk)
5. **Use lifecycle policies** to auto-delete old/untagged images and control costs
6. **Geo-replicate** your registry to regions where you deploy for faster pulls
7. **Sign images with Cosign/Notary** and verify signatures before deployment for supply chain security

---

## What's Next?

Now that you know how to build, store, and distribute container images, let's learn how to orchestrate hundreds or thousands of containers with Kubernetes in [Chapter 16.4: Kubernetes Architecture](./04-kubernetes-architecture.md).
