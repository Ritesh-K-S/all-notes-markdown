# Chapter 05: Containerization for ML

## Table of Contents
- [What is Containerization?](#what-is-containerization)
- [Why Containerization Matters for ML](#why-containerization-matters-for-ml)
- [How Docker Works](#how-docker-works)
- [Docker Fundamentals for ML](#docker-fundamentals-for-ml)
- [Building ML Docker Images](#building-ml-docker-images)
- [GPU Containers (NVIDIA Docker)](#gpu-containers-nvidia-docker)
- [Docker Compose for ML Stacks](#docker-compose-for-ml-stacks)
- [Kubernetes Basics for ML](#kubernetes-basics-for-ml)
- [Container Registries and CI/CD](#container-registries-and-cicd)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Containerization?

**Simple Explanation:** Imagine you're shipping a fish tank. Instead of telling someone "buy a 20-gallon tank, fill it with water at exactly 76°F, add these specific fish, use this filter..." — you ship the ENTIRE fish tank, water, fish, filter, and all. That's what a container does for software.

A container packages your ML model + code + Python version + all libraries + system dependencies into a single portable unit that runs identically everywhere — your laptop, your colleague's machine, a cloud server, or a Kubernetes cluster.

**Formal Definition:** A container is a lightweight, standalone, executable package that includes everything needed to run a piece of software: code, runtime, system tools, libraries, and settings. Unlike virtual machines, containers share the host OS kernel, making them much lighter and faster to start.

### Container vs Virtual Machine

```
Virtual Machines:                    Containers:
┌──────┐ ┌──────┐ ┌──────┐         ┌──────┐ ┌──────┐ ┌──────┐
│ App1 │ │ App2 │ │ App3 │         │ App1 │ │ App2 │ │ App3 │
├──────┤ ├──────┤ ├──────┤         ├──────┤ ├──────┤ ├──────┤
│Libs  │ │Libs  │ │Libs  │         │Libs  │ │Libs  │ │Libs  │
├──────┤ ├──────┤ ├──────┤         └──────┴─┴──────┴─┴──────┘
│Guest │ │Guest │ │Guest │         ┌─────────────────────────┐
│  OS  │ │  OS  │ │  OS  │         │    Container Runtime    │
├──────┴─┴──────┴─┴──────┤         │       (Docker)          │
│      Hypervisor         │         ├─────────────────────────┤
├─────────────────────────┤         │       Host OS           │
│       Host OS           │         ├─────────────────────────┤
├─────────────────────────┤         │      Hardware           │
│      Hardware           │         └─────────────────────────┘
└─────────────────────────┘

Size: 1-20 GB each                  Size: 50-500 MB each
Boot: Minutes                       Boot: Seconds
Overhead: High                      Overhead: Minimal
```

---

## Why Containerization Matters for ML

### The "Works on My Machine" Problem

```
Data Scientist's Laptop:          Production Server:
- Python 3.9.7                    - Python 3.8.10
- numpy 1.21.0                    - numpy 1.19.5
- scikit-learn 1.0                - scikit-learn 0.24
- Ubuntu 20.04                    - CentOS 7
- CUDA 11.3                       - CUDA 10.2

Result: "It works on my machine!" → Crashes in production
```

### Key Benefits for ML

| Benefit | Without Containers | With Containers |
|---------|-------------------|-----------------|
| **Reproducibility** | "Which Python version was that again?" | Exact same environment every time |
| **Dependency Hell** | Model A needs TF 2.10, Model B needs TF 1.15 | Each model has its own isolated env |
| **Scaling** | Manually set up each server | Spin up 100 identical copies in seconds |
| **GPU Compatibility** | CUDA version mismatches | Matched CUDA + driver in container |
| **Collaboration** | "Run these 47 setup commands" | `docker run my-model` |
| **Testing** | "It passed locally..." | Test in production-identical container |

### Real-World Scenarios

- **Model A** needs PyTorch 2.0 + CUDA 12.0
- **Model B** needs TensorFlow 1.15 + CUDA 10.0
- **Model C** needs Python 3.7 + specific system libraries

Without containers: impossible to run on the same machine.
With containers: each model runs in its own isolated environment, even on the same GPU.

---

## How Docker Works

### Architecture

```
┌─────────────────────────────────────────────┐
│                Docker Client                 │
│         (docker build, docker run)           │
└─────────────────────┬───────────────────────┘
                      │ REST API
┌─────────────────────▼───────────────────────┐
│              Docker Daemon (dockerd)          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Images   │  │Containers│  │ Networks  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────┐
│          Container Registry                  │
│    (Docker Hub, ECR, GCR, ACR)              │
└─────────────────────────────────────────────┘
```

### Core Concepts

| Concept | Analogy | Description |
|---------|---------|-------------|
| **Dockerfile** | Recipe | Instructions to build an image |
| **Image** | Blueprint/Template | Read-only template with everything needed |
| **Container** | Running Instance | A live, running copy of an image |
| **Registry** | App Store | Where images are stored and shared |
| **Volume** | External Hard Drive | Persistent storage that survives container restart |
| **Layer** | Transparency Film | Each instruction creates a cacheable layer |

### Layer System (Critical for ML)

```
Dockerfile Instructions:          Resulting Layers:
                                  
FROM python:3.10                  ─── Layer 1: Base Python (900MB)
                                       │ (cached, shared across images)
COPY requirements.txt .           ─── Layer 2: requirements file (1KB)
                                       │
RUN pip install -r requirements   ─── Layer 3: Python packages (2GB)
                                       │ (cached until requirements change)
COPY model.pkl .                  ─── Layer 4: Model file (500MB)
                                       │ (rebuilt when model changes)
COPY app.py .                     ─── Layer 5: Application code (5KB)
                                       │ (rebuilt when code changes)
```

> **Key Insight:** Docker caches layers. If you change `app.py`, only layers 4-5 rebuild. If you change `requirements.txt`, layers 2-5 rebuild. **Order matters!** Put things that change least at the top.

---

## Docker Fundamentals for ML

### Essential Docker Commands

```bash
# Build an image from a Dockerfile
docker build -t my-ml-model:v1.0 .

# Run a container from an image
docker run -p 8000:8000 my-ml-model:v1.0

# Run with GPU access
docker run --gpus all -p 8000:8000 my-ml-model:v1.0

# Run interactively (for debugging)
docker run -it my-ml-model:v1.0 /bin/bash

# Mount local directory (for development)
docker run -v $(pwd)/data:/app/data my-ml-model:v1.0

# List running containers
docker ps

# View container logs
docker logs <container_id>

# Stop a container
docker stop <container_id>

# Remove unused images (free disk space!)
docker system prune -a

# Check image size
docker images my-ml-model
```

### Your First ML Dockerfile

```dockerfile
# Dockerfile - Basic ML model server
# Always pin your base image version for reproducibility
FROM python:3.10-slim

# Set working directory inside the container
WORKDIR /app

# Install system dependencies first (changes rarely)
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies (changes occasionally)
# This is separate from code copy for layer caching!
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy model file (changes when model is retrained)
COPY model.joblib .

# Copy application code (changes frequently)
COPY app.py .

# Expose the port the app runs on
EXPOSE 8000

# Health check - Docker/K8s will restart if this fails
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run the application
# Use exec form (not shell form) for proper signal handling
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Requirements File

```txt
# requirements.txt - Pin ALL versions for reproducibility
fastapi==0.104.1
uvicorn==0.24.0
numpy==1.24.3
scikit-learn==1.3.2
joblib==1.3.2
pydantic==2.5.2
```

---

## Building ML Docker Images

### Production-Optimized Dockerfile

```dockerfile
# Dockerfile.production - Optimized for size and security

# ============ Stage 1: Builder ============
# Multi-stage build: first stage installs everything
FROM python:3.10-slim as builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y \
    gcc g++ \
    && rm -rf /var/lib/apt/lists/*

# Create virtual environment (isolates packages)
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install Python packages
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ============ Stage 2: Runtime ============
# Second stage: only runtime dependencies (much smaller!)
FROM python:3.10-slim as runtime

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install only runtime system dependencies
RUN apt-get update && apt-get install -y \
    libgomp1 \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && useradd --create-home appuser

# Copy application files
COPY model.joblib .
COPY app.py .

# Run as non-root user (security best practice)
USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### Image Size Comparison

| Base Image | Size | Use Case |
|-----------|------|----------|
| `python:3.10` | 920MB | Development, has everything |
| `python:3.10-slim` | 130MB | Production, most common |
| `python:3.10-alpine` | 50MB | Smallest, but compilation issues |
| `nvidia/cuda:12.0-runtime` | 2.5GB | GPU inference |
| `nvidia/cuda:12.0-devel` | 5.5GB | GPU training (has compilers) |

### Dockerfile for Deep Learning Models

```dockerfile
# Dockerfile.gpu - For PyTorch/TensorFlow with GPU
FROM nvidia/cuda:12.1.0-cudnn8-runtime-ubuntu22.04

# Prevent interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1

# Install Python and system dependencies
RUN apt-get update && apt-get install -y \
    python3.10 \
    python3-pip \
    python3.10-venv \
    curl \
    && ln -s /usr/bin/python3.10 /usr/bin/python \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install PyTorch with CUDA support
COPY requirements-gpu.txt .
RUN pip install --no-cache-dir -r requirements-gpu.txt

# Copy model and application
COPY models/ ./models/
COPY src/ ./src/

# Create non-root user
RUN useradd --create-home mluser
USER mluser

EXPOSE 8000

CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```txt
# requirements-gpu.txt
torch==2.1.0+cu121
torchvision==0.16.0+cu121
fastapi==0.104.1
uvicorn==0.24.0
numpy==1.24.3
Pillow==10.1.0
```

### .dockerignore (Critical for ML!)

```dockerignore
# .dockerignore - Exclude files from Docker build context
# Without this, Docker sends EVERYTHING to the daemon (slow!)

# Data files (huge, should be mounted or downloaded)
data/
*.csv
*.parquet
*.h5

# Training artifacts
checkpoints/
runs/
wandb/
mlruns/

# Virtual environments
venv/
.env/
__pycache__/
*.pyc

# Jupyter notebooks (not needed in production)
*.ipynb
.ipynb_checkpoints/

# Git
.git/
.gitignore

# IDE
.vscode/
.idea/

# Docker
Dockerfile*
docker-compose*.yml

# OS files
.DS_Store
Thumbs.db

# Documentation
docs/
*.md
```

---

## GPU Containers (NVIDIA Docker)

### How GPU Containers Work

```
┌─────────────────────────────────────────┐
│            Container                     │
│  ┌─────────────────────────────────┐   │
│  │  Application (PyTorch/TF)       │   │
│  │  CUDA Toolkit (Runtime)         │   │
│  │  cuDNN Libraries                │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
           │ (nvidia-container-toolkit)
┌──────────▼──────────────────────────────┐
│  Host: NVIDIA Driver (must match!)      │
│  Host: nvidia-container-toolkit         │
└──────────▼──────────────────────────────┘
           │
┌──────────▼──────────────────────────────┐
│          GPU Hardware                    │
└─────────────────────────────────────────┘
```

### NVIDIA Container Toolkit Setup

```bash
# Install NVIDIA Container Toolkit (Ubuntu/Debian)
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify GPU access inside container
docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi
```

### GPU Container Patterns

```python
# gpu_check.py - Verify GPU is accessible inside container
import torch
import subprocess

def check_gpu():
    """Comprehensive GPU check for containerized environments"""
    print("=" * 50)
    print("GPU Environment Check")
    print("=" * 50)
    
    # CUDA availability
    print(f"CUDA available: {torch.cuda.is_available()}")
    print(f"CUDA version: {torch.version.cuda}")
    print(f"cuDNN version: {torch.backends.cudnn.version()}")
    
    # GPU details
    if torch.cuda.is_available():
        print(f"GPU count: {torch.cuda.device_count()}")
        for i in range(torch.cuda.device_count()):
            print(f"  GPU {i}: {torch.cuda.get_device_name(i)}")
            print(f"    Memory: {torch.cuda.get_device_properties(i).total_mem / 1e9:.1f} GB")
    
    # nvidia-smi output
    try:
        result = subprocess.run(['nvidia-smi'], capture_output=True, text=True)
        print(f"\nnvidia-smi output:\n{result.stdout}")
    except FileNotFoundError:
        print("nvidia-smi not found in container")

if __name__ == "__main__":
    check_gpu()
```

### Running with Specific GPUs

```bash
# Use all GPUs
docker run --gpus all my-model:latest

# Use specific GPU(s)
docker run --gpus '"device=0"' my-model:latest
docker run --gpus '"device=0,1"' my-model:latest

# Limit GPU memory (useful for sharing)
docker run --gpus all \
    -e NVIDIA_VISIBLE_DEVICES=0 \
    -e CUDA_MEMORY_FRACTION=0.5 \
    my-model:latest
```

### CUDA Version Compatibility Matrix

| Host Driver | Max CUDA | Compatible Container CUDA |
|------------|----------|---------------------------|
| 525.x | 12.0 | 10.2, 11.x, 12.0 |
| 535.x | 12.2 | 10.2, 11.x, 12.0-12.2 |
| 545.x | 12.3 | 10.2, 11.x, 12.0-12.3 |

> **Key Rule:** Container CUDA version must be ≤ Host driver's supported CUDA version. The host driver is backward compatible with older CUDA versions.

---

## Docker Compose for ML Stacks

### What is Docker Compose?

Docker Compose lets you define and run multi-container applications. For ML, this means running your model server, database, cache, monitoring, and other services together with a single command.

### Complete ML Serving Stack

```yaml
# docker-compose.yml - Full ML serving infrastructure
version: '3.8'

services:
  # ============ Model Server ============
  model-server:
    build:
      context: .
      dockerfile: Dockerfile.production
    ports:
      - "8000:8000"
    environment:
      - MODEL_PATH=/app/models/model.joblib
      - LOG_LEVEL=info
      - WORKERS=4
    volumes:
      - ./models:/app/models:ro  # Read-only model mount
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # ============ Redis Cache ============
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

  # ============ Prometheus (Metrics) ============
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  # ============ Grafana (Dashboards) ============
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/dashboards:/etc/grafana/provisioning/dashboards
    depends_on:
      - prometheus

  # ============ Nginx (Load Balancer/Reverse Proxy) ============
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - model-server

volumes:
  redis_data:
  prometheus_data:
  grafana_data:
```

### Nginx Configuration for ML

```nginx
# nginx/nginx.conf - Load balancing and rate limiting
upstream model_backend {
    least_conn;  # Route to least busy server
    server model-server:8000;
    # Add more servers for horizontal scaling:
    # server model-server-2:8000;
    # server model-server-3:8000;
}

server {
    listen 80;
    
    # Rate limiting (protect from abuse)
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
    
    location /predict {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://model_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 30s;  # Model inference timeout
    }
    
    location /health {
        proxy_pass http://model_backend;
    }
    
    # Deny access to management endpoints from outside
    location /metrics {
        deny all;
        return 403;
    }
}
```

### Docker Compose Commands

```bash
# Start all services
docker-compose up -d

# Start with build
docker-compose up -d --build

# View logs
docker-compose logs -f model-server

# Scale model server (horizontal scaling)
docker-compose up -d --scale model-server=3

# Stop everything
docker-compose down

# Stop and remove volumes (CAUTION: deletes data)
docker-compose down -v

# Check status
docker-compose ps
```

### Development vs Production Compose

```yaml
# docker-compose.override.yml - Development overrides (auto-loaded)
version: '3.8'

services:
  model-server:
    build:
      context: .
      dockerfile: Dockerfile  # Use dev Dockerfile
    volumes:
      - .:/app  # Mount code for hot-reload
    environment:
      - LOG_LEVEL=debug
      - RELOAD=true
    command: uvicorn app:app --host 0.0.0.0 --port 8000 --reload
    # No GPU in dev
    deploy:
      resources: {}
```

```bash
# Development (uses docker-compose.yml + docker-compose.override.yml)
docker-compose up

# Production (explicit file, skips override)
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Kubernetes Basics for ML

### Why Kubernetes for ML?

Docker runs containers on a single machine. Kubernetes orchestrates containers across many machines. It handles:
- **Auto-scaling** — more replicas when traffic spikes
- **Self-healing** — restart crashed containers automatically
- **Rolling updates** — deploy new model versions with zero downtime
- **Resource management** — GPU scheduling, memory limits

### Kubernetes Architecture (Simplified)

```
┌──────────────────────────────────────────────────────┐
│                 Kubernetes Cluster                     │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │             Control Plane                        │ │
│  │  ┌──────┐ ┌──────────┐ ┌──────────┐           │ │
│  │  │ API  │ │Scheduler │ │Controller│           │ │
│  │  │Server│ │          │ │ Manager  │           │ │
│  │  └──────┘ └──────────┘ └──────────┘           │ │
│  └─────────────────────────────────────────────────┘ │
│                                                       │
│  ┌────────────────┐  ┌────────────────┐             │
│  │   Worker Node  │  │   Worker Node  │             │
│  │  ┌──────────┐  │  │  ┌──────────┐  │             │
│  │  │  Pod     │  │  │  │  Pod     │  │             │
│  │  │ ┌──────┐ │  │  │  │ ┌──────┐ │  │             │
│  │  │ │Model │ │  │  │  │ │Model │ │  │             │
│  │  │ │Server│ │  │  │  │ │Server│ │  │             │
│  │  │ └──────┘ │  │  │  │ └──────┘ │  │             │
│  │  └──────────┘  │  │  └──────────┘  │             │
│  │  GPU: A100     │  │  GPU: A100     │             │
│  └────────────────┘  └────────────────┘             │
└──────────────────────────────────────────────────────┘
```

### Key Kubernetes Resources for ML

```yaml
# 1. Deployment - Manages model server replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-model-server
  labels:
    app: ml-model
    version: v2.1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Create 1 extra pod during update
      maxUnavailable: 0  # Never have fewer than desired
  selector:
    matchLabels:
      app: ml-model
  template:
    metadata:
      labels:
        app: ml-model
        version: v2.1
    spec:
      containers:
      - name: model-server
        image: myregistry.com/ml-model:v2.1
        ports:
        - containerPort: 8000
        env:
        - name: MODEL_VERSION
          value: "2.1"
        - name: LOG_LEVEL
          value: "info"
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
            nvidia.com/gpu: 1
          limits:
            memory: "4Gi"
            cpu: "2000m"
            nvidia.com/gpu: 1
        readinessProbe:  # Is the pod ready to receive traffic?
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30  # Model loading time
          periodSeconds: 10
        livenessProbe:   # Should we restart this pod?
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 3
      # Pull model from shared storage
      initContainers:
      - name: model-downloader
        image: amazon/aws-cli
        command: ['aws', 's3', 'cp', 's3://models/v2.1/model.pt', '/models/']
        volumeMounts:
        - name: model-volume
          mountPath: /models
      volumes:
      - name: model-volume
        emptyDir: {}
```

```yaml
# 2. Service - Exposes the deployment internally
apiVersion: v1
kind: Service
metadata:
  name: ml-model-service
spec:
  selector:
    app: ml-model
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP  # Internal only; use LoadBalancer for external
```

```yaml
# 3. HorizontalPodAutoscaler - Auto-scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ml-model-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ml-model-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
```

### Canary Deployment for ML Models

```yaml
# canary-deployment.yaml - Route 10% traffic to new model
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-model-canary
spec:
  replicas: 1  # 1 out of 10 total = ~10% traffic
  selector:
    matchLabels:
      app: ml-model
      track: canary
  template:
    metadata:
      labels:
        app: ml-model
        track: canary
        version: v2.2-canary
    spec:
      containers:
      - name: model-server
        image: myregistry.com/ml-model:v2.2-canary
        ports:
        - containerPort: 8000
```

### GPU Scheduling in Kubernetes

```yaml
# gpu-node-selector.yaml - Schedule on specific GPU types
spec:
  containers:
  - name: model-server
    resources:
      limits:
        nvidia.com/gpu: 1
  nodeSelector:
    gpu-type: a100  # Custom label on GPU nodes
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

---

## Container Registries and CI/CD

### Automated Build Pipeline

```yaml
# .github/workflows/build-deploy.yml
name: Build and Deploy ML Model

on:
  push:
    branches: [main]
    paths:
      - 'models/**'
      - 'src/**'
      - 'Dockerfile'
      - 'requirements.txt'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/model-server:${{ github.sha }}
            ghcr.io/${{ github.repository }}/model-server:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/ml-model-server \
            model-server=ghcr.io/${{ github.repository }}/model-server:${{ github.sha }}
```

---

## Best Practices

### 1. Image Size Optimization

```dockerfile
# ❌ BAD: 5GB+ image
FROM nvidia/cuda:12.1.0-devel-ubuntu22.04
RUN pip install torch torchvision tensorflow keras ...

# ✅ GOOD: Multi-stage, runtime only, ~2GB
FROM nvidia/cuda:12.1.0-devel-ubuntu22.04 AS builder
RUN pip install --target=/install torch --index-url https://download.pytorch.org/whl/cu121

FROM nvidia/cuda:12.1.0-runtime-ubuntu22.04
COPY --from=builder /install /usr/local/lib/python3.10/site-packages
```

### 2. Security Best Practices

```dockerfile
# Run as non-root user
RUN useradd --create-home --shell /bin/bash mluser
USER mluser

# Don't store secrets in images
# ❌ BAD:
ENV AWS_SECRET_KEY=mysecret123

# ✅ GOOD: Use runtime secrets
# Pass at runtime: docker run -e AWS_SECRET_KEY=$SECRET my-image
```

### 3. Model Loading Pattern

```python
# ✅ Download model at container startup, not baked into image
# This way, image stays small and model can be updated independently

import os
import boto3

def download_model():
    """Download model from S3 at startup"""
    model_path = os.getenv("MODEL_PATH", "/app/models/model.pt")
    
    if not os.path.exists(model_path):
        s3 = boto3.client('s3')
        s3.download_file(
            Bucket=os.getenv("MODEL_BUCKET"),
            Key=os.getenv("MODEL_KEY"),
            Filename=model_path
        )
    
    return model_path
```

---

## Common Mistakes

### 1. Not Using .dockerignore

```bash
# Without .dockerignore, your 10GB training data gets sent to Docker daemon
# Build takes 10 minutes instead of 10 seconds

# ❌ Build context: 12GB (includes data/, checkpoints/, venv/)
# ✅ Build context: 50MB (only needed files)
```

### 2. Installing Dependencies Wrong

```dockerfile
# ❌ WRONG: Single layer, no caching benefit
COPY . .
RUN pip install -r requirements.txt

# ✅ CORRECT: Separate layers, cached until requirements change
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

### 3. Using `latest` Tag in Production

```dockerfile
# ❌ WRONG: "latest" can change without notice
FROM python:latest
# What does "latest" mean? Python 3.10? 3.11? 3.12?

# ✅ CORRECT: Pin exact version
FROM python:3.10.13-slim-bookworm
```

### 4. Running as Root

```dockerfile
# ❌ WRONG: Running as root (security vulnerability)
CMD ["python", "app.py"]

# ✅ CORRECT: Non-root user
RUN useradd --create-home appuser
USER appuser
CMD ["python", "app.py"]
```

### 5. Forgetting Model-Container Coupling

```
# ❌ WRONG: Model baked into image
# Every model retrain requires rebuilding entire image

# ✅ CORRECT: Mount model as volume or download at startup
docker run -v /models/latest:/app/model my-server:v1.0
```

---

## Interview Questions

### Conceptual

**Q1: Why use containers for ML instead of just virtual environments?**
> **A:** Virtual environments only isolate Python packages. Containers isolate everything: OS, system libraries (CUDA, cuDNN), Python version, environment variables. A virtualenv won't help when your model needs a specific CUDA version or system library that conflicts with another model.

**Q2: Explain multi-stage Docker builds and why they matter for ML.**
> **A:** Multi-stage builds use multiple FROM statements. First stage installs build tools and compiles packages. Second stage only copies the compiled outputs. This reduces image size dramatically (often 2-5x) by excluding compilers, headers, and build artifacts. Critical for ML because PyTorch/TF have heavy build dependencies but lightweight runtime requirements.

**Q3: How do you handle GPU access inside Docker containers?**
> **A:** Using NVIDIA Container Toolkit (nvidia-docker2). The host needs NVIDIA drivers installed, plus the container toolkit. Then use `--gpus all` flag. The container includes CUDA/cuDNN but uses the host's GPU driver. The container CUDA version must be ≤ host driver's supported CUDA version.

**Q4: How would you deploy 50 different ML models efficiently using containers?**
> **A:** 
> - Use a common base image with shared libraries (Python, CUDA)
> - Models loaded dynamically from object storage (S3/GCS), not baked in
> - Kubernetes with node pools for different resource requirements
> - Multi-model serving (Triton) to share GPU across models
> - HPA for auto-scaling individual models independently

### System Design

**Q5: Design a containerized ML serving platform for a company with 20 ML models.**
> **A:**
> - Container registry (ECR/GCR) for versioned model images
> - Kubernetes cluster with GPU and CPU node pools
> - Shared base images to reduce storage and build time
> - Helm charts for standardized deployment
> - Prometheus + Grafana for monitoring
> - Canary deployments for safe model updates
> - Model stored in S3, downloaded at startup via init container
> - Auto-scaling based on request latency and GPU utilization

---

## Quick Reference

### Dockerfile Template Cheatsheet

```dockerfile
# Production ML Dockerfile Template
FROM python:3.10-slim                          # 1. Base image (pinned version)
WORKDIR /app                                   # 2. Working directory
RUN apt-get update && apt-get install -y \     # 3. System deps (rarely change)
    libgomp1 curl && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .                        # 4. Python deps (change sometimes)
RUN pip install --no-cache-dir -r requirements.txt
COPY model.joblib .                            # 5. Model (changes on retrain)
COPY src/ ./src/                               # 6. Code (changes frequently)
RUN useradd --create-home appuser && \         # 7. Security
    chown -R appuser:appuser /app
USER appuser
EXPOSE 8000                                    # 8. Port declaration
HEALTHCHECK CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Essential Commands

| Task | Command |
|------|---------|
| Build image | `docker build -t name:tag .` |
| Run container | `docker run -p 8000:8000 name:tag` |
| Run with GPU | `docker run --gpus all name:tag` |
| Interactive shell | `docker run -it name:tag /bin/bash` |
| Mount volume | `docker run -v /host/path:/container/path name:tag` |
| View logs | `docker logs -f <container_id>` |
| Compose up | `docker-compose up -d --build` |
| Compose scale | `docker-compose up -d --scale service=3` |
| Image size | `docker images name` |
| Clean up | `docker system prune -a` |
| Push to registry | `docker push registry/name:tag` |

### K8s Quick Reference

| Task | Command |
|------|---------|
| Apply config | `kubectl apply -f deployment.yaml` |
| Get pods | `kubectl get pods` |
| Pod logs | `kubectl logs <pod-name>` |
| Scale | `kubectl scale deployment/name --replicas=5` |
| Rollout status | `kubectl rollout status deployment/name` |
| Rollback | `kubectl rollout undo deployment/name` |
| Port forward | `kubectl port-forward pod/name 8000:8000` |
| Describe pod | `kubectl describe pod <pod-name>` |

### Image Size Optimization Checklist

| Technique | Savings | Effort |
|-----------|---------|--------|
| Use `-slim` base | 700MB+ | Low |
| Multi-stage build | 1-3GB | Medium |
| `.dockerignore` | Varies (up to 10GB+) | Low |
| `--no-cache-dir` pip | 100-500MB | Low |
| Remove apt cache | 50-200MB | Low |
| Use runtime (not devel) CUDA | 2-3GB | Low |
| Pin and minimize dependencies | Varies | Medium |

---

> **Pro Tip:** Your first Docker image will be 5GB. After applying all optimizations, it'll be 1-2GB. The biggest wins come from: multi-stage builds, using runtime (not devel) CUDA images, and proper `.dockerignore`.
