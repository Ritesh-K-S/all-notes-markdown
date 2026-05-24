# Docker Deep Dive — Images, Containers, Volumes, Networks

> **What you'll learn**: Master Docker's core building blocks — how images are built layer by layer, how containers run, how data persists with volumes, and how containers communicate through networks.

---

## Real-Life Analogy: A Recipe, a Meal, and a Kitchen

Think of Docker like cooking:

- **Docker Image** = A **recipe** (written instructions for making a dish). You can share it, copy it, and anyone following it gets the same result.
- **Docker Container** = The **actual meal** made from the recipe. You can make multiple meals from one recipe, and each is independent.
- **Docker Volume** = A **shared pantry** that persists even if you clean the kitchen. The meal is gone, but ingredients you stored remain.
- **Docker Network** = The **intercom system** between different kitchens. Chefs (containers) can communicate without shouting across the building.

```
Recipe (Image)  ──cook──▶  Meal (Container)  ──save──▶  Pantry (Volume)
     │                          │                              │
     │  Immutable               │  Ephemeral                   │  Persistent
     │  Shareable               │  Isolated                    │  Shared
     │  Versioned               │  Running process             │  Survives restarts
```

---

## Part 1: Docker Images — The Blueprint

### What is a Docker Image?

An image is a **read-only template** containing:
- A minimal operating system (Alpine Linux, Debian, etc.)
- Your application code
- All dependencies (libraries, runtimes)
- Configuration files
- Instructions on how to start the app

### Image Layers — The Secret Sauce

Every instruction in a Dockerfile creates a **new layer**. Layers are cached and shared:

```
┌─────────────────────────────────────────────┐
│  Layer 5: CMD ["python", "app.py"]          │  ← Metadata only
├─────────────────────────────────────────────┤
│  Layer 4: COPY . /app                       │  ← 50 KB (your code)
├─────────────────────────────────────────────┤
│  Layer 3: RUN pip install -r requirements   │  ← 80 MB (packages)
├─────────────────────────────────────────────┤
│  Layer 2: COPY requirements.txt /app        │  ← 1 KB
├─────────────────────────────────────────────┤
│  Layer 1: FROM python:3.11-slim             │  ← 120 MB (base)
└─────────────────────────────────────────────┘

Total image size: ~200 MB
But if another image also uses python:3.11-slim,
that 120 MB layer is stored ONLY ONCE on disk!
```

### How Layer Caching Works

```
First build:                          Second build (only code changed):
                                      
Layer 1: FROM python:3.11 ──build──   Layer 1: FROM python:3.11 ──CACHED ✓
Layer 2: COPY requirements ──build──  Layer 2: COPY requirements ──CACHED ✓
Layer 3: RUN pip install   ──build──  Layer 3: RUN pip install   ──CACHED ✓
Layer 4: COPY . /app       ──build──  Layer 4: COPY . /app       ──REBUILD (changed!)
Layer 5: CMD               ──build──  Layer 5: CMD               ──REBUILD

Time: 2 minutes                       Time: 5 seconds! 🚀
```

> **Pro Tip**: Put things that change LEAST at the top of your Dockerfile, and things that change MOST at the bottom. This maximizes cache hits.

### Dockerfile Deep Dive

```dockerfile
# ─── Dockerfile with best practices ───

# 1. Use specific version tags (never :latest in production)
FROM python:3.11.7-slim-bookworm

# 2. Set metadata
LABEL maintainer="team@example.com"
LABEL version="1.0"

# 3. Create non-root user (security!)
RUN groupadd -r appuser && useradd -r -g appuser appuser

# 4. Set working directory
WORKDIR /app

# 5. Copy dependency file FIRST (cache optimization)
COPY requirements.txt .

# 6. Install dependencies in a single layer
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# 7. Copy application code LAST (changes most often)
COPY --chown=appuser:appuser . .

# 8. Switch to non-root user
USER appuser

# 9. Expose port (documentation only, doesn't publish)
EXPOSE 8000

# 10. Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 11. Set the startup command
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:app"]
```

### Multi-Stage Builds — Smaller Production Images

The biggest secret to small images — use one container to BUILD, another to RUN:

```dockerfile
# ─── Stage 1: Build (includes compilers, build tools) ───
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

# ─── Stage 2: Run (minimal image, just the binary) ───
FROM alpine:3.19
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/server .
USER nobody
EXPOSE 8080
CMD ["./server"]
```

```
Without multi-stage: 1.2 GB (Go compiler + source + binary)
With multi-stage:    15 MB  (just the compiled binary + Alpine)
```

### Common Dockerfile Instructions Reference

| Instruction | Purpose | Example |
|-------------|---------|---------|
| `FROM` | Base image | `FROM node:18-alpine` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy files from host | `COPY package.json .` |
| `ADD` | Copy + extract archives/URLs | `ADD app.tar.gz /app` |
| `RUN` | Execute command during build | `RUN npm install` |
| `ENV` | Set environment variable | `ENV NODE_ENV=production` |
| `ARG` | Build-time variable | `ARG VERSION=1.0` |
| `EXPOSE` | Document port (metadata) | `EXPOSE 3000` |
| `USER` | Switch user | `USER node` |
| `CMD` | Default run command | `CMD ["node", "app.js"]` |
| `ENTRYPOINT` | Fixed run command | `ENTRYPOINT ["java", "-jar"]` |
| `HEALTHCHECK` | Container health test | See example above |
| `VOLUME` | Declare mount point | `VOLUME /data` |

---

## Part 2: Containers — Running Instances

### Container Lifecycle

```
Image ──docker run──▶ Created ──start──▶ Running ──stop──▶ Stopped ──rm──▶ Removed
                         │                   │                  │
                         │                   │ ──pause──▶ Paused│
                         │                   │                  │
                         │                   │ ──kill──▶ Stopped│
                         │                   │                  │
                         └───────────────────┴──────────────────┘
                                    docker inspect (inspect any state)
```

### Essential Container Commands

```bash
# Run a container (pull image if not cached)
docker run -d \
  --name myapp \
  -p 8080:8080 \
  -e DB_HOST=localhost \
  --memory=512m \
  --cpus=1.5 \
  --restart=unless-stopped \
  myimage:1.0

# List running containers
docker ps

# List ALL containers (including stopped)
docker ps -a

# View logs (real-time follow)
docker logs -f --tail 100 myapp

# Execute command inside running container
docker exec -it myapp /bin/sh

# Inspect container metadata
docker inspect myapp

# View resource usage (CPU, memory, network)
docker stats myapp

# Stop gracefully (SIGTERM, then SIGKILL after 10s)
docker stop myapp

# Remove container
docker rm myapp

# Remove all stopped containers
docker container prune
```

### Container Resource Limits

```bash
# CPU limits
docker run --cpus=2.0 myapp          # Max 2 CPU cores
docker run --cpu-shares=512 myapp     # Relative weight (default: 1024)

# Memory limits
docker run --memory=1g myapp          # Max 1 GB RAM
docker run --memory=1g --memory-swap=2g myapp  # 1GB RAM + 1GB swap

# Combined example for production
docker run -d \
  --memory=512m \
  --memory-reservation=256m \
  --cpus=1.0 \
  --pids-limit=100 \
  --read-only \
  --tmpfs /tmp \
  myapp:latest
```

---

## Part 3: Volumes — Persistent Data

### The Problem: Container Data is Ephemeral

```
Container starts → writes data to /app/data → container crashes → data is GONE!

┌─────────────────┐         ┌─────────────────┐
│  Container v1   │ ──rm──▶ │  Container v2   │
│  /data: 500 MB  │         │  /data: empty!  │  ← Data lost!
└─────────────────┘         └─────────────────┘
```

### The Solution: Volumes

```
┌─────────────────┐         ┌─────────────────┐
│  Container v1   │ ──rm──▶ │  Container v2   │
│  /data ──mount──┼────┐    │  /data ──mount──┼────┐
└─────────────────┘    │    └─────────────────┘    │
                       ▼                           ▼
              ┌────────────────────────────────────────┐
              │  Docker Volume: "app_data"              │
              │  Location: /var/lib/docker/volumes/...  │
              │  Data: 500 MB  ← PERSISTS!             │
              └────────────────────────────────────────┘
```

### Three Types of Mounts

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  1. VOLUME MOUNT (Managed by Docker — RECOMMENDED)             │
│     docker run -v mydata:/app/data myapp                       │
│     → Stored in /var/lib/docker/volumes/                       │
│     → Docker manages lifecycle                                 │
│     → Works on all platforms                                   │
│                                                                │
│  2. BIND MOUNT (Maps host directory)                           │
│     docker run -v /home/user/code:/app myapp                   │
│     → Direct mapping to host filesystem                        │
│     → Great for development (live code reload)                 │
│     → Host path must exist                                     │
│                                                                │
│  3. TMPFS MOUNT (In-memory only, Linux)                        │
│     docker run --tmpfs /app/cache myapp                        │
│     → Stored in RAM, never written to disk                     │
│     → Fast but ephemeral                                       │
│     → Good for sensitive temp data                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Volume Commands

```bash
# Create a named volume
docker volume create pgdata

# Run container with volume
docker run -d \
  --name postgres \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect pgdata

# Backup a volume
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata-backup.tar.gz /data

# Remove unused volumes
docker volume prune
```

### Volume in Docker Compose

```yaml
services:
  postgres:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data        # Named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Bind mount

  redis:
    image: redis:7
    volumes:
      - redis_data:/data

volumes:
  pgdata:
    driver: local
  redis_data:
    driver: local
```

---

## Part 4: Docker Networking — How Containers Talk

### Default Network Types

```
┌──────────────────────────────────────────────────────────────────┐
│  HOST MACHINE                                                     │
│                                                                   │
│  ┌─── bridge (default) ───────────────────────────────────┐      │
│  │  172.17.0.0/16                                         │      │
│  │                                                        │      │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐        │      │
│  │  │Container1│    │Container2│    │Container3│        │      │
│  │  │172.17.0.2│◄──▶│172.17.0.3│◄──▶│172.17.0.4│        │      │
│  │  └──────────┘    └──────────┘    └──────────┘        │      │
│  │       Can communicate via IP addresses                 │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                   │
│  ┌─── custom bridge (user-defined) ──────────────────────┐      │
│  │  10.0.1.0/24                                           │      │
│  │                                                        │      │
│  │  ┌──────────┐    ┌──────────┐                         │      │
│  │  │  webapp  │◄──▶│  redis   │  ← DNS resolution!     │      │
│  │  │ 10.0.1.2 │    │ 10.0.1.3 │    "redis" resolves    │      │
│  │  └──────────┘    └──────────┘    to 10.0.1.3         │      │
│  │       Can communicate via CONTAINER NAMES              │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                   │
│  ┌─── host network ─────────────────────────────────────┐       │
│  │  Container uses host's network directly               │       │
│  │  No port mapping needed, best performance             │       │
│  │  No network isolation (less secure)                   │       │
│  └───────────────────────────────────────────────────────┘       │
│                                                                   │
│  ┌─── none ─────────────────────────────────────────────┐       │
│  │  Container has NO network access                      │       │
│  │  Completely isolated (for security-sensitive tasks)    │       │
│  └───────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────┘
```

### Why User-Defined Bridge > Default Bridge

| Feature | Default Bridge | User-Defined Bridge |
|---------|---------------|-------------------|
| DNS resolution by name | ❌ No | ✅ Yes |
| Isolation between networks | ❌ All on same | ✅ Separate networks |
| Connect/disconnect live | ❌ No | ✅ Yes |
| Fine-grained control | ❌ Limited | ✅ Full |

### Network Commands

```bash
# Create a custom network
docker network create --driver bridge app-network

# Run containers on custom network
docker run -d --name webapp --network app-network myapp
docker run -d --name redis --network app-network redis:7

# Now webapp can reach redis by name:
# redis://redis:6379 (DNS resolves "redis" to its container IP)

# Connect existing container to network
docker network connect app-network existing-container

# Inspect network
docker network inspect app-network

# List networks
docker network ls
```

### Port Mapping — Exposing Containers to the Outside

```
┌──────── HOST (your machine) ────────────────────────────┐
│                                                          │
│  Port 80 ─────────┐                                     │
│                    ▼                                     │
│            ┌──────────────┐                              │
│            │   Container  │                              │
│            │   Port 8080  │  ← App listens on 8080      │
│            └──────────────┘                              │
│                                                          │
│  -p 80:8080 means:                                       │
│  "Map host port 80 → container port 8080"               │
│                                                          │
│  Port 5432 ───────┐                                     │
│                    ▼                                     │
│            ┌──────────────┐                              │
│            │  PostgreSQL  │                              │
│            │  Port 5432   │                              │
│            └──────────────┘                              │
│                                                          │
│  -p 5432:5432 (same port mapping)                       │
└──────────────────────────────────────────────────────────┘
```

```bash
# Map host port 80 to container port 8080
docker run -p 80:8080 myapp

# Map to specific interface (only localhost, not external)
docker run -p 127.0.0.1:80:8080 myapp

# Map random host port (Docker chooses)
docker run -P myapp  # uses EXPOSE ports from Dockerfile

# Multiple port mappings
docker run -p 80:8080 -p 443:8443 myapp
```

---

## Complete Example: Full-Stack Application

### Python Flask + PostgreSQL + Redis + Nginx

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - webapp
    networks:
      - frontend

  webapp:
    build:
      context: ./app
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://appuser:secret@postgres:5432/mydb
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY_FILE=/run/secrets/app_secret
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    secrets:
      - app_secret
    networks:
      - frontend
      - backend

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/01-init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    secrets:
      - db_password
    networks:
      - backend

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - backend

volumes:
  pgdata:
  redis_data:

networks:
  frontend:
  backend:

secrets:
  db_password:
    file: ./secrets/db_password.txt
  app_secret:
    file: ./secrets/app_secret.txt
```

**Architecture of this setup:**

```
                    Internet
                       │
                       ▼
              ┌────────────────┐
              │   Port 80      │
              │   Nginx (LB)   │
              └───────┬────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ webapp  │ │ webapp  │ │ webapp  │   ← 3 replicas
    │ :8000   │ │ :8000   │ │ :8000   │
    └────┬────┘ └────┬────┘ └────┬────┘
         │           │           │
         └─────┬─────┴─────┬─────┘
               │            │
          ┌────┴────┐  ┌────┴────┐
          │PostgreSQL│  │  Redis  │
          │  :5432   │  │  :6379  │
          └─────────┘  └─────────┘
```

---

## Java Multi-Stage Docker Build (Spring Boot)

```dockerfile
# === Stage 1: Download dependencies (cached layer) ===
FROM maven:3.9-eclipse-temurin-17 AS deps
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B

# === Stage 2: Build application ===
FROM deps AS builder
COPY src ./src
RUN mvn clean package -DskipTests -B

# === Stage 3: Extract Spring Boot layers (for better caching) ===
FROM builder AS extractor
RUN java -Djarmode=layertools -jar target/*.jar extract --destination /extracted

# === Stage 4: Production image ===
FROM eclipse-temurin:17-jre-alpine AS production
WORKDIR /app

# Create non-root user
RUN addgroup -S spring && adduser -S spring -G spring

# Copy layers in order of change frequency (least → most)
COPY --from=extractor /extracted/dependencies/ ./
COPY --from=extractor /extracted/spring-boot-loader/ ./
COPY --from=extractor /extracted/snapshot-dependencies/ ./
COPY --from=extractor /extracted/application/ ./

USER spring
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## How It Works Internally — Docker Engine Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Docker Engine                              │
│                                                                   │
│  ┌─────────────────┐                                             │
│  │   Docker CLI    │  ← You type commands here                   │
│  │  (docker run)   │                                             │
│  └────────┬────────┘                                             │
│           │ REST API (/var/run/docker.sock)                       │
│           ▼                                                       │
│  ┌─────────────────┐                                             │
│  │  Docker Daemon  │  ← Long-running process (dockerd)           │
│  │   (dockerd)     │     Manages images, containers, networks    │
│  └────────┬────────┘                                             │
│           │ gRPC                                                  │
│           ▼                                                       │
│  ┌─────────────────┐                                             │
│  │   containerd    │  ← Container runtime (manages lifecycle)    │
│  │                 │     Start, stop, pause containers            │
│  └────────┬────────┘                                             │
│           │ OCI Runtime Spec                                      │
│           ▼                                                       │
│  ┌─────────────────┐                                             │
│  │      runc       │  ← Low-level runtime (creates namespaces,   │
│  │                 │     sets up cgroups, starts process)         │
│  └────────┬────────┘                                             │
│           │                                                       │
│           ▼                                                       │
│  ┌─────────────────┐                                             │
│  │  Linux Kernel   │  ← Namespaces + Cgroups + OverlayFS        │
│  └─────────────────┘                                             │
└──────────────────────────────────────────────────────────────────┘
```

### What happens when you run `docker run nginx`:

1. **CLI** sends REST request to Docker daemon
2. **Daemon** checks if `nginx` image exists locally
3. If not, **pulls** from Docker Hub (layer by layer)
4. **Daemon** tells containerd to create a container
5. **containerd** calls runc to set up namespaces and cgroups
6. **runc** starts the nginx process as PID 1 in the container
7. **runc** exits, containerd manages the running container
8. Container is now running and accessible

---

## Real-World Example: Spotify's Docker Usage

Spotify containerized their entire backend:

```
Before Docker:                          After Docker:
- 500+ services                         - Same 500+ services
- 4-hour deploy cycle                   - 2-minute deploy cycle
- "Works on my machine" daily           - Consistent everywhere
- Manual server provisioning            - Automated container orchestration
- 30% server utilization               - 70%+ utilization
```

They built **Helios** (later moved to Kubernetes) to orchestrate their Docker containers, handling thousands of deployments per day.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using `latest` tag | Unpredictable deploys | Use specific version tags |
| Not using `.dockerignore` | Huge build context, slow builds | Add `.git`, `node_modules`, `target/` |
| Running apt-get in multiple RUNs | Each creates a layer | Chain with `&&` in single RUN |
| Not cleaning apt cache | Bloated images | `RUN apt-get install -y pkg && rm -rf /var/lib/apt/lists/*` |
| COPY before dependency install | Rebuilds deps on every code change | COPY deps file → install → COPY code |
| No health checks | Orchestrators can't detect failures | Always add HEALTHCHECK |
| Bind mounts in production | Tight coupling to host filesystem | Use named volumes |
| Not setting restart policy | Container dies = stays dead | Use `--restart=unless-stopped` |

---

## When to Use / When NOT to Use

### ✅ Use Docker When:
- You need **reproducible environments** across dev/staging/prod
- You're building a **CI/CD pipeline** (build once, run anywhere)
- You want to **run multiple services** on one machine without conflicts
- You need **fast, lightweight isolation** between applications
- You're preparing for **Kubernetes** deployment

### ❌ Volumes vs Bind Mounts — Decision Guide:
- **Development**: Bind mounts (live code reload)
- **Production databases**: Named volumes (managed, portable)
- **Secrets/configs**: tmpfs (never touch disk) or Docker secrets

### ❌ Network Mode — Decision Guide:
- **Default (bridge)**: Most containers, isolated networking
- **Host**: When you need maximum network performance (benchmarks, monitoring)
- **None**: Security-sensitive batch jobs (no network access)
- **Custom bridge**: Multi-container apps (service discovery via DNS)

---

## Key Takeaways

1. **Images are immutable, layered blueprints** — optimize layer order for caching (deps first, code last)
2. **Multi-stage builds** dramatically reduce image size (build in one stage, run in another)
3. **Containers are ephemeral** — never store important data inside them; use volumes
4. **Named volumes** persist data and survive container removal; **bind mounts** are for development
5. **User-defined bridge networks** give you DNS-based service discovery (containers find each other by name)
6. **Always run as non-root**, add health checks, set resource limits, and use specific image tags
7. **Docker Compose** is the standard for multi-container development environments

---

## What's Next?

Now that you understand Docker's internals, let's explore how teams share and distribute images through container registries in [Chapter 16.3: Container Registries](./03-container-registries.md).
