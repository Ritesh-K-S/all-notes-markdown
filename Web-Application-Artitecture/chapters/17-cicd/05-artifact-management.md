# Artifact Management & Container Image Pipelines

> **What you'll learn**: How to build, version, store, scan, and promote container images and build artifacts through environments — the critical link between CI (building) and CD (deploying).

---

## Real-Life Analogy

Think of a **pharmaceutical company** manufacturing medicine.

You don't just mix chemicals and hand pills directly to patients. There's a strict process:

1. **Manufacturing** → Create the batch (build the artifact)
2. **Labeling** → Stamp it with batch number, date, ingredients (versioning/tagging)
3. **Quality Testing** → Lab tests for safety and efficacy (security scanning)
4. **Warehouse Storage** → Store in a controlled facility with proper temperature (registry)
5. **Distribution** → Ship to pharmacies in proper sequence — first a small batch to test, then mass distribution (promotion through environments)
6. **Recall mechanism** → If something's wrong, trace back to the exact batch (immutable artifacts with provenance)

**Artifact management is your software's pharmaceutical supply chain.** You build it once, label it, test it, store it safely, and ship it through controlled stages to production.

---

## Core Concept Explained Step-by-Step

### Step 1: What is an Artifact?

An **artifact** is the packaged, deployable output of your build process. It's what actually runs in production.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      TYPES OF ARTIFACTS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  SOURCE CODE ────build────▶ ARTIFACT                                  │
│                                                                       │
│  ┌────────────────────┬───────────────────────────────────────────┐  │
│  │ Language/Platform   │ Artifact Type                             │  │
│  ├────────────────────┼───────────────────────────────────────────┤  │
│  │ Java               │ JAR, WAR, EAR                             │  │
│  │ Python             │ Wheel (.whl), Docker Image                │  │
│  │ Node.js            │ npm package (.tgz), Docker Image          │  │
│  │ Go                 │ Static binary, Docker Image               │  │
│  │ .NET               │ NuGet package, Docker Image               │  │
│  │ Frontend (React)   │ Static bundle (HTML/CSS/JS .zip)          │  │
│  │ Any language       │ Docker/OCI Image (most common today)      │  │
│  │ Infrastructure     │ VM Image (AMI, VMDK)                      │  │
│  │ Helm               │ Helm Chart (.tgz)                         │  │
│  │ Terraform          │ Module archive                            │  │
│  └────────────────────┴───────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Step 2: The Container Image Pipeline

Since Docker/OCI images are the most common artifact type today, let's focus on the **container image pipeline**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  CONTAINER IMAGE PIPELINE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐  │
│  │  Build  │──▶│  Tag    │──▶│  Scan   │──▶│  Push   │──▶│ Promote │  │
│  │  Image  │   │  Image  │   │  Image  │   │  Image  │   │  Image  │  │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘  │
│                                                                           │
│  Build:    docker build -t myapp .                                       │
│  Tag:      myapp:v1.2.3, myapp:abc123f, myapp:latest                    │
│  Scan:     trivy image myapp:v1.2.3 (find vulnerabilities)              │
│  Push:     docker push registry.com/myapp:v1.2.3                        │
│  Promote:  Copy from dev-registry → prod-registry                        │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Step 3: Container Registries — Where Images Live

A **container registry** is a storage service for container images (like GitHub for code, but for Docker images).

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CONTAINER REGISTRIES                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  PUBLIC REGISTRIES (open to everyone):                                │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Docker Hub         hub.docker.com/r/nginx                    │    │
│  │  GitHub Container   ghcr.io/company/myapp                     │    │
│  │  Quay.io           quay.io/company/myapp                      │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  CLOUD PROVIDER REGISTRIES (private, integrated):                    │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  AWS ECR           123456789.dkr.ecr.us-east-1.amazonaws.com  │    │
│  │  Google Artifact   us-docker.pkg.dev/project/repo/image       │    │
│  │  Azure ACR         myregistry.azurecr.io/myapp                │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  SELF-HOSTED REGISTRIES:                                             │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Harbor             (open-source, enterprise features)        │    │
│  │  JFrog Artifactory  (universal artifact manager)              │    │
│  │  Nexus Repository   (Java-focused, supports Docker)           │    │
│  │  GitLab Registry    (built into GitLab)                       │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Step 4: Image Tagging Strategies

How you tag images determines how you track and deploy them:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IMAGE TAGGING STRATEGIES                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Strategy 1: Git SHA (RECOMMENDED for CI/CD)                         │
│  ─────────────────────────────────────────                           │
│  registry.com/myapp:a1b2c3d                                          │
│  ✅ Unique, immutable, traceable to exact commit                     │
│  ✅ Easy to correlate: "what code is running?"                       │
│                                                                       │
│  Strategy 2: Semantic Versioning (for releases)                      │
│  ─────────────────────────────────────────────                       │
│  registry.com/myapp:1.2.3                                            │
│  registry.com/myapp:1.2 (points to latest patch)                     │
│  registry.com/myapp:1   (points to latest minor)                     │
│  ✅ Human-readable, clear upgrade path                               │
│  ⚠️  Requires release process to bump versions                       │
│                                                                       │
│  Strategy 3: Branch + Build Number                                   │
│  ────────────────────────────────                                    │
│  registry.com/myapp:main-142                                         │
│  registry.com/myapp:feature-auth-38                                  │
│  ✅ Good for development/preview environments                        │
│                                                                       │
│  Strategy 4: Timestamp                                               │
│  ─────────────────────                                               │
│  registry.com/myapp:20260523-143022                                  │
│  ✅ Sortable, unique                                                 │
│  ❌ Not traceable to code without extra metadata                     │
│                                                                       │
│  ⚠️  NEVER USE:                                                      │
│  registry.com/myapp:latest                                           │
│  ❌ Mutable! You don't know WHAT version "latest" is                 │
│  ❌ K8s won't re-pull if tag hasn't changed                          │
│  ❌ Impossible to rollback to "the previous latest"                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Step 5: Image Scanning & Security

Before any image reaches production, it should be scanned for vulnerabilities:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IMAGE SECURITY PIPELINE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────┐  │
│  │  Build  │──▶│ Vulnerability│──▶│  License     │──▶│  Sign    │  │
│  │  Image  │   │  Scan (CVEs) │   │  Compliance  │   │  Image   │  │
│  └─────────┘   └──────────────┘   └──────────────┘   └──────────┘  │
│                                                                       │
│  Vulnerability Scanning:                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Tool: Trivy, Grype, Snyk Container, AWS Inspector          │    │
│  │                                                             │    │
│  │  What it checks:                                            │    │
│  │  • OS packages (apt/yum) for known CVEs                     │    │
│  │  • Application dependencies (pip, npm, maven)               │    │
│  │  • Base image vulnerabilities                               │    │
│  │  • Misconfigurations (running as root, no healthcheck)      │    │
│  │                                                             │    │
│  │  Output:                                                    │    │
│  │  ┌──────────┬────────────┬─────────┬────────────────────┐  │    │
│  │  │ Severity │ Package    │ Version │ Fixed In            │  │    │
│  │  ├──────────┼────────────┼─────────┼────────────────────┤  │    │
│  │  │ CRITICAL │ openssl    │ 3.0.1   │ 3.0.7              │  │    │
│  │  │ HIGH     │ libcurl    │ 7.68.0  │ 7.68.0-1ubuntu2.18 │  │    │
│  │  │ MEDIUM   │ expat      │ 2.4.1   │ 2.4.9              │  │    │
│  │  └──────────┴────────────┴─────────┴────────────────────┘  │    │
│  │                                                             │    │
│  │  Policy: Block if CRITICAL or HIGH > 0 (configurable)      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  Image Signing (Supply Chain Security):                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Tool: Cosign (Sigstore), Notary v2                          │    │
│  │                                                             │    │
│  │  Purpose: Prove the image was built by YOUR CI pipeline     │    │
│  │  Nobody tampered with it between build and deploy           │    │
│  │                                                             │    │
│  │  Build ──▶ Sign with private key ──▶ Push signature         │    │
│  │  Deploy: Verify signature before pulling image              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Step 6: Artifact Promotion Through Environments

The **promotion model** ensures the same artifact tested in staging is what runs in production:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ARTIFACT PROMOTION MODEL                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│              BUILD ONCE → PROMOTE THROUGH ENVIRONMENTS                    │
│                                                                           │
│  CI builds image:  registry.com/myapp:abc123f                            │
│         │                                                                 │
│         ▼                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐             │
│  │     DEV      │────▶│   STAGING    │────▶│  PRODUCTION  │             │
│  │              │     │              │     │              │             │
│  │ Auto-deploy  │     │ Auto-deploy  │     │ Manual gate  │             │
│  │ Run smoke    │     │ Run full E2E │     │ or auto after│             │
│  │ tests        │     │ Perf tests   │     │ staging pass │             │
│  │              │     │ Security scan│     │              │             │
│  └──────────────┘     └──────────────┘     └──────────────┘             │
│                                                                           │
│  SAME IMAGE (abc123f) at every stage!                                    │
│  Only CONFIGURATION changes (env vars, secrets, replicas)                │
│                                                                           │
│  ──────────────────────────────────────────────────────────────────      │
│                                                                           │
│  ANTI-PATTERN (Don't do this!):                                          │
│  ❌ Build image-dev, Build image-staging, Build image-prod               │
│     (Each rebuild might produce different results!)                       │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

**Registry-based promotion** (copying images between registries):

```
┌──────────────────────┐   promote   ┌──────────────────────┐
│  dev-registry.com/   │ ──────────▶ │  prod-registry.com/  │
│  myapp:abc123f       │  (copy tag) │  myapp:abc123f       │
└──────────────────────┘             └──────────────────────┘

Or using tag-based promotion:
┌──────────────────────────────────────────────────┐
│  registry.com/myapp:abc123f       (built by CI)  │
│  registry.com/myapp:staging       (promoted)     │
│  registry.com/myapp:production    (promoted)     │
│                                                  │
│  All three tags point to THE SAME image layers!  │
└──────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Docker Image Layers & Storage

Understanding how images are stored helps you optimize build times:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DOCKER IMAGE INTERNALS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  An image is a STACK OF LAYERS (each layer is a filesystem diff):    │
│                                                                       │
│  ┌───────────────────────────────────────────────┐                   │
│  │  Layer 5: COPY . /app  (your code, ~10MB)     │ ← Changes often  │
│  ├───────────────────────────────────────────────┤                   │
│  │  Layer 4: RUN pip install  (deps, ~200MB)     │ ← Changes weekly │
│  ├───────────────────────────────────────────────┤                   │
│  │  Layer 3: COPY requirements.txt               │ ← Changes weekly │
│  ├───────────────────────────────────────────────┤                   │
│  │  Layer 2: RUN apt-get install  (OS, ~50MB)    │ ← Changes rarely │
│  ├───────────────────────────────────────────────┤                   │
│  │  Layer 1: FROM python:3.11-slim  (base, 150MB)│ ← Changes rarely │
│  └───────────────────────────────────────────────┘                   │
│                                                                       │
│  CACHING: If a layer hasn't changed, Docker reuses the cached layer. │
│  ORDER MATTERS: Put rarely-changing layers FIRST, code LAST.         │
│                                                                       │
│  Registry stores each layer ONCE (content-addressable by SHA256).    │
│  Multiple tags sharing layers = no extra storage cost.                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### OCI Image Specification

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OCI IMAGE MANIFEST                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  {                                                                    │
│    "schemaVersion": 2,                                               │
│    "mediaType": "application/vnd.oci.image.manifest.v1+json",        │
│    "config": {                                                        │
│      "digest": "sha256:abc123...",  ← Image config (env, cmd, etc.)  │
│      "size": 7023                                                    │
│    },                                                                 │
│    "layers": [                                                        │
│      {                                                                │
│        "digest": "sha256:def456...",  ← Layer 1 content hash        │
│        "size": 32654                                                 │
│      },                                                               │
│      {                                                                │
│        "digest": "sha256:789ghi...",  ← Layer 2 content hash        │
│        "size": 16724                                                 │
│      }                                                                │
│    ]                                                                  │
│  }                                                                    │
│                                                                       │
│  A TAG (e.g., "v1.2.3") simply POINTS to a manifest digest.         │
│  Manifests are immutable — same content = same digest forever.        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Multi-Architecture Images

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MULTI-ARCH IMAGE (Manifest List)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  registry.com/myapp:v1.2.3  ← single tag, multiple architectures    │
│         │                                                             │
│         ▼                                                             │
│  ┌─────────────────────────────────────────┐                         │
│  │         MANIFEST LIST (INDEX)           │                         │
│  ├─────────────────────────────────────────┤                         │
│  │  Platform: linux/amd64 → manifest A     │ ← Intel/AMD servers    │
│  │  Platform: linux/arm64 → manifest B     │ ← AWS Graviton, M1 Mac│
│  │  Platform: linux/arm/v7 → manifest C    │ ← Raspberry Pi         │
│  └─────────────────────────────────────────┘                         │
│                                                                       │
│  Docker/K8s automatically picks the right image for the platform!    │
│                                                                       │
│  Build command:                                                       │
│  docker buildx build --platform linux/amd64,linux/arm64 \           │
│    -t registry.com/myapp:v1.2.3 --push .                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: Optimized Dockerfile with Multi-Stage Build

```dockerfile
# Dockerfile — Multi-stage build for minimal production image
# Stage 1: Build dependencies (large image with build tools)
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies (C compilers for native packages)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && rm -rf /var/lib/apt/lists/*

# Install Python dependencies (cached if requirements.txt hasn't changed)
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: Production image (minimal, no build tools)
FROM python:3.11-slim AS production

WORKDIR /app

# Copy only the installed packages from builder
COPY --from=builder /install /usr/local

# Install only runtime dependencies (no gcc!)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 && rm -rf /var/lib/apt/lists/*

# Copy application code (changes most often — last layer!)
COPY . .

# Security: run as non-root user
RUN useradd -r -s /bin/false appuser
USER appuser

# Metadata
LABEL org.opencontainers.image.source="https://github.com/company/myapp"
LABEL org.opencontainers.image.version="1.2.3"

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')"

CMD ["gunicorn", "app.main:app", "-w", "4", "-b", "0.0.0.0:8080"]
```

```python
# scripts/build_and_push.py
# Complete image build pipeline script
import subprocess
import sys
import json
from datetime import datetime

def build_image(registry: str, app_name: str, git_sha: str, version: str):
    """Build, scan, sign, and push a container image."""
    
    full_tag = f"{registry}/{app_name}:{git_sha}"
    version_tag = f"{registry}/{app_name}:{version}"
    
    # Step 1: Build the image
    print(f"🔨 Building image: {full_tag}")
    subprocess.run([
        "docker", "build",
        "--build-arg", f"BUILD_DATE={datetime.utcnow().isoformat()}",
        "--build-arg", f"GIT_SHA={git_sha}",
        "--label", f"org.opencontainers.image.revision={git_sha}",
        "-t", full_tag,
        "-t", version_tag,
        "."
    ], check=True)
    
    # Step 2: Scan for vulnerabilities
    print("🔍 Scanning for vulnerabilities...")
    result = subprocess.run([
        "trivy", "image",
        "--severity", "CRITICAL,HIGH",
        "--exit-code", "1",  # Fail if critical/high found
        "--format", "json",
        "--output", "scan-results.json",
        full_tag
    ])
    
    if result.returncode != 0:
        print("❌ Critical/High vulnerabilities found! Blocking push.")
        with open("scan-results.json") as f:
            findings = json.load(f)
            for vuln in findings.get("Results", []):
                for v in vuln.get("Vulnerabilities", []):
                    print(f"  {v['Severity']}: {v['PkgName']} - {v['Title']}")
        sys.exit(1)
    
    print("✅ No critical vulnerabilities found")
    
    # Step 3: Push to registry
    print(f"📤 Pushing {full_tag}")
    subprocess.run(["docker", "push", full_tag], check=True)
    subprocess.run(["docker", "push", version_tag], check=True)
    
    # Step 4: Sign the image (supply chain security)
    print("🔏 Signing image with cosign...")
    subprocess.run([
        "cosign", "sign",
        "--key", "cosign.key",
        full_tag
    ], check=True)
    
    print(f"✅ Image built, scanned, pushed, and signed: {full_tag}")
    return full_tag

if __name__ == "__main__":
    build_image(
        registry="ghcr.io/company",
        app_name="my-app",
        git_sha=sys.argv[1],
        version=sys.argv[2]
    )
```

### Java: Multi-Stage Dockerfile with Maven

```dockerfile
# Dockerfile for Java application
# Stage 1: Build with Maven
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# Copy POM first (dependency layer caching)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src/ src/
RUN mvn package -DskipTests -B

# Stage 2: Minimal runtime image
FROM eclipse-temurin:17-jre-alpine AS production

WORKDIR /app

# Copy only the built JAR from builder stage
COPY --from=builder /app/target/*.jar app.jar

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

```java
// src/main/java/com/company/BuildInfo.java
// Embed build information in the application for traceability
package com.company;

import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;
import java.util.Map;

/**
 * Exposes build metadata at /actuator/info endpoint.
 * Useful for: "Which version is running in production right now?"
 */
@Component
public class BuildInfo implements InfoContributor {
    
    @Override
    public void contribute(Info.Builder builder) {
        // These are set via Spring Boot's build-info plugin
        // or passed as environment variables from the container
        builder.withDetail("build", Map.of(
            "artifact", getEnvOrDefault("BUILD_ARTIFACT", "my-app"),
            "version", getEnvOrDefault("BUILD_VERSION", "unknown"),
            "gitSha", getEnvOrDefault("GIT_SHA", "unknown"),
            "buildTime", getEnvOrDefault("BUILD_TIME", "unknown"),
            "javaVersion", System.getProperty("java.version")
        ));
    }
    
    private String getEnvOrDefault(String key, String defaultValue) {
        String value = System.getenv(key);
        return value != null ? value : defaultValue;
    }
}
```

---

## Infrastructure Examples

### GitHub Actions: Complete Image Pipeline

```yaml
# .github/workflows/image-pipeline.yml
name: Container Image Pipeline

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-scan-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write  # For uploading scan results
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      # Generate image tags based on Git context
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            # Git SHA (always)
            type=sha,prefix=
            # Semantic version from Git tag
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            # Branch name (for non-main branches)
            type=ref,event=branch
      
      - uses: docker/setup-buildx-action@v3
      
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      # Build and push image
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64  # Multi-arch!
      
      # Scan for vulnerabilities
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      # Upload scan results to GitHub Security tab
      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
      
      # Sign the image with Cosign (keyless)
      - name: Sign image
        uses: sigstore/cosign-installer@v3
      - run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}

  # Gate: Only promote to production if scan passed
  promote-to-production:
    needs: build-scan-push
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval
    steps:
      - name: Promote image
        run: |
          echo "Image ${{ needs.build-scan-push.outputs.image-tag }} promoted to production"
          # In practice: update GitOps repo or trigger deployment
```

### Harbor Registry with Vulnerability Policy

```yaml
# Harbor project configuration (via API/UI)
# Harbor automatically scans images on push and enforces policies

# harbor-project-policy.yaml (conceptual)
project:
  name: production-apps
  policies:
    vulnerability:
      # Block deployment of images with critical vulnerabilities
      prevent_vulnerable_images_from_running: true
      severity_threshold: high  # Block HIGH and CRITICAL
    
    image_signing:
      # Only allow signed images
      content_trust: true
    
    retention:
      # Keep last 10 tags, delete untagged after 7 days
      rules:
        - tag_match: "v*"
          retain_count: 10
        - tag_match: "sha-*"
          retain_days: 30
        - untagged: true
          retain_days: 7
    
    replication:
      # Replicate production images to DR registry
      destination: dr-registry.company.com
      trigger: on_push
      filter:
        - tag: "v*"
```

### JFrog Artifactory: Universal Artifact Management

```
┌──────────────────────────────────────────────────────────────────────┐
│                  JFROG ARTIFACTORY SETUP                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Artifactory manages ALL artifact types in one place:                  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │                      ARTIFACTORY                              │     │
│  │                                                              │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │     │
│  │  │ Docker Repo  │  │  Maven Repo  │  │  npm Repo    │      │     │
│  │  │ (OCI images) │  │  (JAR/WAR)   │  │  (packages)  │      │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │     │
│  │  │  PyPI Repo   │  │  Helm Repo   │  │ Generic Repo │      │     │
│  │  │  (wheels)    │  │  (charts)    │  │ (binaries)   │      │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │     │
│  │                                                              │     │
│  │  Features:                                                    │     │
│  │  • Immutable artifacts (can't overwrite published versions)  │     │
│  │  • Build info (link artifacts to CI build)                    │     │
│  │  • Xray security scanning (integrated)                        │     │
│  │  • Replication to edge nodes globally                         │     │
│  │  • Retention policies (auto-cleanup old artifacts)            │     │
│  └──────────────────────────────────────────────────────────────┘     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Netflix's Image Pipeline (Titus)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NETFLIX IMAGE PIPELINE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Netflix builds thousands of container images daily for Titus (their     │
│  container platform):                                                     │
│                                                                           │
│  1. Developer pushes code                                                 │
│         │                                                                 │
│         ▼                                                                 │
│  2. Jenkins CI builds the app + creates Docker image                     │
│         │                                                                 │
│         ▼                                                                 │
│  3. Image is "baked" — dependencies frozen, metadata embedded            │
│         │                                                                 │
│         ▼                                                                 │
│  4. Vulnerability scan (block critical CVEs)                             │
│         │                                                                 │
│         ▼                                                                 │
│  5. Push to internal registry                                             │
│         │                                                                 │
│         ▼                                                                 │
│  6. Spinnaker picks up the image                                         │
│         │                                                                 │
│         ▼                                                                 │
│  7. Canary deployment (1% traffic)                                       │
│     - Compare metrics vs baseline                                        │
│     - Automated analysis (Kayenta)                                       │
│         │                                                                 │
│    Pass? ──No──▶ Automatic Rollback                                      │
│         │                                                                 │
│        Yes                                                                │
│         │                                                                 │
│         ▼                                                                 │
│  8. Progressive rollout (10% → 50% → 100%)                              │
│                                                                           │
│  Key practices:                                                           │
│  • Images are IMMUTABLE once built                                       │
│  • Every image has full provenance (who built, what commit, when)        │
│  • Base images are maintained by a platform team                          │
│  • Automated base image updates when security patches are released       │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Google's Approach: Binary Authorization

```
┌─────────────────────────────────────────────────────────────────────┐
│              GOOGLE BINARY AUTHORIZATION                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Policy: "Only images that passed ALL checks can run in production"  │
│                                                                       │
│  ┌─────────────┐                                                     │
│  │ Build image │                                                     │
│  └──────┬──────┘                                                     │
│         │                                                             │
│         ▼                                                             │
│  ┌─────────────────────────────────────────────────┐                 │
│  │            ATTESTATION CHAIN                     │                 │
│  │                                                 │                 │
│  │  ✅ Attestation 1: Built by authorized CI       │                 │
│  │  ✅ Attestation 2: Vulnerability scan passed    │                 │
│  │  ✅ Attestation 3: Code review approved         │                 │
│  │  ✅ Attestation 4: Compliance check passed      │                 │
│  └─────────────────────────────────────────────────┘                 │
│         │                                                             │
│         ▼                                                             │
│  ┌─────────────────────────────────────────────────┐                 │
│  │  GKE Admission Controller                        │                 │
│  │  "Does this image have ALL required attestations?"│                 │
│  │                                                 │                 │
│  │  YES → Allow pod to run                          │                 │
│  │  NO  → REJECT (pod fails to start)              │                 │
│  └─────────────────────────────────────────────────┘                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Solution |
|---------|--------|----------|
| **Using `:latest` tag** | Don't know what's running, can't rollback | Use immutable tags (SHA, semver) |
| **Rebuilding for each environment** | Different binary in staging vs prod | Build once, promote the same artifact |
| **No vulnerability scanning** | Known CVEs in production | Scan in CI pipeline; block critical vulns |
| **Large images (2GB+)** | Slow pulls, slow deploys, wasted storage | Multi-stage builds, alpine/slim bases |
| **Running as root in container** | Security risk if container is compromised | Always add `USER nonroot` in Dockerfile |
| **No image retention policy** | Registry grows to terabytes, costs spiral | Auto-delete old tags; keep last N versions |
| **Secrets baked into image** | Anyone who pulls the image gets your secrets | Use env vars or mounted secrets at runtime |
| **No provenance/signing** | Supply chain attacks possible | Sign images with Cosign; verify before deploy |
| **No base image update process** | Running year-old OS with known CVEs | Automate base image rebuilds weekly |

---

## When to Use / When NOT to Use

### Container Images (Docker/OCI):
- ✅ Deploying to Kubernetes, ECS, Cloud Run, or any container platform
- ✅ Need consistent environment between dev/staging/prod
- ✅ Microservices architecture (each service has its own image)
- ❌ Deploying to serverless (AWS Lambda) — use zip archives
- ❌ Frontend-only apps — use CDN + static files

### Artifact Repository (JFrog/Nexus):
- ✅ Multiple artifact types (JARs + Docker + npm + Helm)
- ✅ Need artifact promotion workflows
- ✅ Enterprise compliance requirements (audit trail)
- ❌ Small team with one language — Docker Hub/GHCR is enough

### Image Signing:
- ✅ Regulated industries (finance, healthcare)
- ✅ Multi-team organizations (ensure only approved images deploy)
- ✅ Public-facing infrastructure (prevent supply chain attacks)
- 🟡 Small teams with trusted CI — lower priority but good practice

---

## Key Takeaways

- 🔑 **Build once, deploy everywhere** — the same immutable artifact goes through all environments
- 🔑 **Never use `:latest`** — always tag with commit SHA or semantic version for traceability
- 🔑 **Scan every image** for vulnerabilities before it reaches production (Trivy, Grype, Snyk)
- 🔑 **Multi-stage builds** dramatically reduce image size (don't ship compilers to production!)
- 🔑 **Sign your images** (Cosign) to ensure supply chain integrity — prove YOUR CI built it
- 🔑 **Retention policies** prevent registries from growing unbounded — auto-delete old artifacts
- 🔑 The artifact pipeline is the **bridge between CI and CD** — without reliable, versioned artifacts, deployments are unpredictable

---

## What's Next?

You've now completed the entire CI/CD journey — from understanding what CI/CD is, through pipeline design, tool selection, GitOps workflows, and artifact management. Next, we move to **Part 18: Monitoring, Logging & Observability** — because once you deploy frequently, you need to see exactly what's happening in production.

**Next: [Part 18 — Why Monitoring Matters](../18-observability/01-why-monitoring.md)**
