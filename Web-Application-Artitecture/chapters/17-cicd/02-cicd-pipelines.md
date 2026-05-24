# CI/CD Pipelines — Building, Testing & Deploying Automatically

> **What you'll learn**: How to design and structure a CI/CD pipeline — the automated assembly line that takes your code from commit to production, with every stage explained in detail.

---

## Real-Life Analogy

Think of a **car manufacturing assembly line**.

A car doesn't magically appear at the end of the factory. It moves through specific **stations** in order:

1. **Frame welding** → the body takes shape
2. **Quality check** → is the frame straight?
3. **Paint shop** → apply color
4. **Quality check** → no scratches?
5. **Engine installation** → add the motor
6. **Electronics** → wiring, dashboard
7. **Final inspection** → does everything work?
8. **Ship to dealer** → deliver to customer

If any station finds a defect, the car **stops** and goes back for repair. It never reaches the customer broken.

**A CI/CD pipeline is your code's assembly line.** Each stage transforms, validates, or packages your code. If any stage fails, the pipeline **stops** — broken code never reaches production.

---

## Core Concept Explained Step-by-Step

### Step 1: Pipeline Anatomy

A pipeline is a series of **stages**, each containing one or more **jobs**, each containing one or more **steps**.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CI/CD PIPELINE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   STAGE 1        STAGE 2         STAGE 3        STAGE 4              │
│  ┌────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐          │
│  │  Build  │──▶│   Test   │──▶│  Package │──▶│  Deploy  │          │
│  └────────┘    └──────────┘    └─────────┘    └──────────┘          │
│                                                                       │
│  Each STAGE contains JOBS (can run in parallel):                      │
│                                                                       │
│  ┌──────────────────────────────────────┐                            │
│  │           TEST STAGE                  │                            │
│  │  ┌──────────┐  ┌──────────────────┐  │                            │
│  │  │Unit Tests│  │Integration Tests │  │  ← Jobs run in parallel   │
│  │  └──────────┘  └──────────────────┘  │                            │
│  │  ┌──────────┐  ┌──────────────────┐  │                            │
│  │  │Lint/Style│  │Security Scan     │  │                            │
│  │  └──────────┘  └──────────────────┘  │                            │
│  └──────────────────────────────────────┘                            │
│                                                                       │
│  Each JOB contains STEPS (run sequentially):                          │
│                                                                       │
│  ┌──────────────────────────────────────┐                            │
│  │         UNIT TEST JOB                 │                            │
│  │  Step 1: Checkout code                │                            │
│  │  Step 2: Install dependencies         │                            │
│  │  Step 3: Run pytest                   │                            │
│  │  Step 4: Upload coverage report       │                            │
│  └──────────────────────────────────────┘                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

**Hierarchy:**
```
Pipeline
  └── Stage (sequential — one after another)
        └── Job (parallel within a stage)
              └── Step (sequential within a job)
```

---

### Step 2: The Standard Pipeline Stages

Here's a typical production-grade pipeline:

```
┌────────┐    ┌────────┐    ┌────────┐    ┌─────────┐    ┌─────────┐    ┌──────────┐
│ Source │──▶│ Build  │──▶│  Test  │──▶│ Package │──▶│ Staging │──▶│Production│
│        │    │        │    │        │    │         │    │         │    │          │
│ git    │    │compile │    │ unit   │    │ docker  │    │ deploy  │    │ deploy   │
│ push   │    │install │    │ integ  │    │ image   │    │ to test │    │ to prod  │
│        │    │deps    │    │ e2e    │    │ or JAR  │    │ env     │    │          │
└────────┘    └────────┘    └────────┘    └─────────┘    └─────────┘    └──────────┘
```

Let's break down each stage:

---

### Stage 1: Source (Trigger)

**What**: Something triggers the pipeline to start.

**Common triggers:**
| Trigger | When It Fires |
|---------|--------------|
| `push` | Code pushed to any branch |
| `pull_request` | PR opened/updated |
| `merge` | PR merged to main |
| `schedule` | Cron job (nightly builds) |
| `manual` | Human clicks "Run Pipeline" |
| `tag` | Git tag created (for releases) |

---

### Stage 2: Build

**What**: Compile the code, install dependencies, ensure the project can be built.

```
┌─────────────────────────────────────────┐
│              BUILD STAGE                  │
├─────────────────────────────────────────┤
│                                           │
│  Python:  pip install -r requirements.txt │
│  Java:    mvn compile                     │
│  Node.js: npm ci                          │
│  Go:      go build ./...                  │
│  Rust:    cargo build                     │
│                                           │
│  Output: Compiled binaries / installed    │
│          packages ready for testing       │
└─────────────────────────────────────────┘
```

> **Why `npm ci` instead of `npm install`?**  
> `npm ci` does a clean install from `package-lock.json` exactly, ensuring reproducible builds. `npm install` might update the lock file.

---

### Stage 3: Test

**What**: Run various levels of automated tests.

```
┌─────────────────────────────────────────────────────────────────┐
│                      TEST PYRAMID                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│                          /\                                        │
│                         /  \     E2E Tests (slow, expensive)      │
│                        / E2E\    - Selenium, Cypress, Playwright  │
│                       /______\   - 5-15 minutes                   │
│                      /        \                                    │
│                     /Integration\  Integration Tests              │
│                    /    Tests    \ - Real DB, real APIs            │
│                   /______________\ - 2-5 minutes                  │
│                  /                \                                │
│                 /    Unit Tests    \  Unit Tests (fast, many)     │
│                /                    \ - Mock dependencies          │
│               /______________________\ - 30 seconds - 2 min      │
│                                                                   │
│  Also run in parallel:                                            │
│  • Linting (code style)                                           │
│  • Static Analysis (find bugs without running)                    │
│  • Security Scanning (find vulnerabilities)                       │
│  • Coverage Analysis (measure test coverage %)                    │
└─────────────────────────────────────────────────────────────────┘
```

---

### Stage 4: Package / Build Artifact

**What**: Create a deployable artifact — something you can ship.

```
Source Code  ──▶  Artifact (deployable unit)

Examples:
┌─────────────────┬──────────────────────────────┐
│ Language/Stack   │ Artifact Type                 │
├─────────────────┼──────────────────────────────┤
│ Java            │ JAR / WAR file                │
│ Python          │ Docker image / wheel package  │
│ Node.js         │ Docker image / zip bundle     │
│ Go              │ Static binary                 │
│ .NET            │ NuGet package / Docker image  │
│ Frontend (React)│ Static files (HTML/CSS/JS)    │
└─────────────────┴──────────────────────────────┘
```

**Key principle**: Build the artifact **once**, deploy it to **every environment** (staging, prod). Never rebuild for each environment.

```
                    ┌─────────────────────────────────────┐
                    │  Build artifact ONCE with tag v1.2.3 │
                    └─────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              ┌──────────┐   ┌──────────┐   ┌──────────┐
              │   Dev    │   │ Staging  │   │   Prod   │
              │ (config  │   │ (config  │   │ (config  │
              │  for dev)│   │  for stg)│   │  for prd)│
              └──────────┘   └──────────┘   └──────────┘
              
    Same artifact, different configuration (env vars, secrets)
```

---

### Stage 5: Deploy to Staging

**What**: Deploy to a production-like environment for final validation.

```
┌─────────────────────────────────────────────────────────┐
│                  STAGING ENVIRONMENT                      │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Purpose:                                                 │
│  • Smoke tests (does the app start?)                      │
│  • Integration with real services                         │
│  • Performance baseline testing                           │
│  • Manual QA (if applicable)                              │
│  • User acceptance testing (UAT)                          │
│                                                           │
│  Should mirror production:                                │
│  • Same OS, same runtime version                          │
│  • Same database type (but smaller dataset)               │
│  • Same network topology                                  │
│  • Same monitoring/logging                                │
└─────────────────────────────────────────────────────────┘
```

---

### Stage 6: Deploy to Production

**What**: Ship it to real users!

Deployment strategies (covered in Chapter 5.5):
- **Rolling**: Replace instances one at a time
- **Blue-Green**: Switch traffic from old to new all at once
- **Canary**: Send 1% traffic to new version, gradually increase

```
Production Deployment with Canary:

  Time 0:    [Old v1.0] ████████████████████  100% traffic
  
  Time 1:    [Old v1.0] ███████████████████   99% traffic
             [New v1.1] █                      1% traffic  ← canary
  
  Time 2:    [Old v1.0] ██████████████        70% traffic
             [New v1.1] ██████                 30% traffic  ← if healthy
  
  Time 3:    [New v1.1] ████████████████████  100% traffic  ← full rollout
```

---

## How It Works Internally

### Pipeline Execution Model

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PIPELINE EXECUTION ENGINE                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  1. PARSE: Read YAML config → build Directed Acyclic Graph (DAG)      │
│                                                                        │
│     build ──▶ test ──▶ package ──▶ deploy                             │
│                 │                                                       │
│                 ├──▶ lint (parallel)                                    │
│                 └──▶ scan (parallel)                                    │
│                                                                        │
│  2. SCHEDULE: Topological sort of DAG → execution order                │
│                                                                        │
│  3. EXECUTE: For each node in order:                                   │
│     a. Provision runner (container/VM)                                  │
│     b. Inject environment variables & secrets                          │
│     c. Execute steps sequentially                                      │
│     d. Capture stdout/stderr as logs                                   │
│     e. Check exit code (0 = pass, non-zero = fail)                     │
│     f. Store artifacts (if produced)                                   │
│     g. Tear down runner                                                │
│                                                                        │
│  4. GATE CHECK: Between stages, evaluate:                              │
│     - Did all required jobs pass?                                      │
│     - Is manual approval needed?                                       │
│     - Are deployment windows open?                                     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Caching in Pipelines

Downloading dependencies every single run is slow. Caching fixes this:

```
First Run (no cache):
┌──────────────────────────────────────┐
│  npm ci  →  downloads 500MB          │  ⏱️ 2 minutes
│  Save node_modules to cache          │
└──────────────────────────────────────┘

Subsequent Runs (cache hit):
┌──────────────────────────────────────┐
│  Restore node_modules from cache     │  ⏱️ 5 seconds
│  npm ci  →  "already up to date"     │
└──────────────────────────────────────┘

Cache Key Strategy:
  key: deps-{{ hashFiles('package-lock.json') }}
  
  If lock file changes → cache miss → fresh install
  If lock file same    → cache hit  → instant restore
```

### Parallelization Strategies

```
SEQUENTIAL (slow):
┌───────┐   ┌───────┐   ┌───────┐   ┌───────┐
│ Test1 │──▶│ Test2 │──▶│ Test3 │──▶│ Test4 │  Total: 40 min
│ 10min │   │ 10min │   │ 10min │   │ 10min │
└───────┘   └───────┘   └───────┘   └───────┘

PARALLEL (fast):
┌───────┐
│ Test1 │  10min
└───────┘
┌───────┐
│ Test2 │  10min      Total: 10 min (4x faster!)
└───────┘
┌───────┐
│ Test3 │  10min
└───────┘
┌───────┐
│ Test4 │  10min
└───────┘
```

**Test splitting techniques:**
- **By file**: Split test files across runners
- **By timing**: Balance based on historical run times
- **By type**: Unit tests on one runner, integration on another

---

## Code Examples

### Python: Complete GitHub Actions Pipeline

```yaml
# .github/workflows/pipeline.yml
name: Python CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ──────────────── STAGE 1: Lint & Format ────────────────
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install ruff black
      - run: ruff check .         # Linting
      - run: black --check .      # Format checking

  # ──────────────── STAGE 2: Unit Tests ────────────────
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']  # Test multiple versions
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - name: Cache pip packages
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ hashFiles('requirements*.txt') }}
      - run: pip install -r requirements.txt -r requirements-test.txt
      - run: pytest tests/unit/ --cov=app --cov-report=xml
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.python-version }}
          path: coverage.xml

  # ──────────────── STAGE 3: Integration Tests ────────────────
  integration-tests:
    needs: [unit-tests]  # Only run if unit tests pass
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt -r requirements-test.txt
      - run: pytest tests/integration/ -v
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

  # ──────────────── STAGE 4: Build Docker Image ────────────────
  build:
    needs: [lint, integration-tests]
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=ref,event=branch
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ──────────────── STAGE 5: Deploy to Staging ────────────────
  deploy-staging:
    needs: [build]
    runs-on: ubuntu-latest
    environment: staging  # Requires environment protection rules
    steps:
      - name: Deploy to staging K8s cluster
        run: |
          echo "Deploying ${{ needs.build.outputs.image-tag }} to staging"
          # kubectl apply or helm upgrade command here

  # ──────────────── STAGE 6: Deploy to Production ────────────────
  deploy-production:
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'  # Only deploy main to prod
    environment: production  # Manual approval required
    steps:
      - name: Deploy to production K8s cluster
        run: |
          echo "Deploying to production"
          # kubectl apply or helm upgrade command here
```

### Java: GitLab CI Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - package
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

# Cache Maven dependencies across jobs
cache:
  paths:
    - .m2/repository/
  key: ${CI_COMMIT_REF_SLUG}

build:
  stage: build
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn compile -DskipTests
  artifacts:
    paths:
      - target/

unit-test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn test
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml

integration-test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  services:
    - postgres:15
  variables:
    POSTGRES_DB: testdb
    POSTGRES_PASSWORD: testpass
    SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/testdb
  script:
    - mvn verify -Pintegration-tests

package:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp 
        myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        --namespace=staging
  environment:
    name: staging
    url: https://staging.myapp.com

deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp 
        myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        --namespace=production
  environment:
    name: production
    url: https://myapp.com
  when: manual  # Requires human click
  only:
    - main
```

---

## Infrastructure Examples

### Multi-Environment Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MULTI-ENVIRONMENT PIPELINE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐        │
│  │   Dev    │────▶│    QA    │────▶│ Staging  │────▶│   Prod   │        │
│  │          │     │          │     │          │     │          │        │
│  │auto-deploy│     │auto-deploy│     │auto-deploy│     │manual gate│       │
│  │on commit │     │on merge  │     │on tag    │     │approval  │        │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘        │
│                                                                           │
│  Each environment has:                                                    │
│  • Own Kubernetes namespace (or separate cluster)                         │
│  • Own database instance                                                  │
│  • Own secrets (different API keys)                                       │
│  • Own monitoring dashboards                                              │
│  • Own URL (dev.app.com, qa.app.com, app.com)                            │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pipeline with Quality Gates

```yaml
# Quality gates — thresholds that must pass before proceeding
quality_gates:
  unit_tests:
    min_coverage: 80%        # At least 80% code coverage
    max_failures: 0          # Zero test failures
    
  security_scan:
    critical_vulnerabilities: 0  # No critical CVEs
    high_vulnerabilities: 3      # Max 3 high severity
    
  performance:
    p95_response_time: 200ms    # 95th percentile < 200ms
    error_rate: 0.1%            # Less than 0.1% errors
    
  staging_smoke:
    health_check: pass          # /health returns 200
    core_flows: pass            # Login, checkout working
```

---

## Real-World Example

### Shopify's CI/CD at Scale

Shopify has one of the largest Ruby on Rails monorepos in the world:

```
┌─────────────────────────────────────────────────────────────┐
│               SHOPIFY'S CI PIPELINE                           │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Monorepo: 3 million+ lines of Ruby code                     │
│  Test Suite: 170,000+ tests                                  │
│  Full run time: 40+ hours (if sequential!)                   │
│                                                               │
│  Solution: Massive parallelization                            │
│                                                               │
│  ┌───────────────────────────────────────────┐               │
│  │  CI Trigger (push)                         │               │
│  │         │                                  │               │
│  │         ▼                                  │               │
│  │  ┌─────────────────┐                      │               │
│  │  │ Test Splitter    │ ← Divides tests into │               │
│  │  │ (by timing data) │   balanced groups    │               │
│  │  └─────────────────┘                      │               │
│  │         │                                  │               │
│  │    ┌────┼────┬────┬────┬── ... ──┐        │               │
│  │    ▼    ▼    ▼    ▼    ▼         ▼        │               │
│  │  [R1] [R2] [R3] [R4] [R5] ... [R200]     │               │
│  │  200 parallel runners                      │               │
│  │                                            │               │
│  │  Result: 40 hours → 15 minutes!           │               │
│  └───────────────────────────────────────────┘               │
│                                                               │
│  Additional optimizations:                                    │
│  • Selective testing (only test affected code)                │
│  • Cached Docker layers                                       │
│  • Prebuilt base images                                       │
│  • Flaky test quarantine                                      │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Solution |
|---------|--------|----------|
| **No parallelization** | 30-min pipeline when it could be 5 min | Split tests across multiple runners |
| **Testing in production only** | Broken code reaches users | Mirror prod environment in staging |
| **No caching** | Re-downloading 500MB of deps every run | Cache dependencies with lockfile-based keys |
| **Monolithic pipeline** | One flaky test blocks everything | Separate stages; allow selective re-runs |
| **No artifact versioning** | "Which version is in prod?" mystery | Tag every artifact with commit SHA |
| **Shared mutable state** | Tests pass alone, fail together | Each test gets a fresh database/state |
| **Ignoring flaky tests** | Team stops trusting CI | Quarantine flaky tests; fix or delete them |
| **No timeout limits** | Hung pipelines waste runner time | Set max timeout per job (e.g., 15 minutes) |

---

## When to Use / When NOT to Use

### Pipeline Complexity by Team Size

| Team Size | Recommended Pipeline |
|-----------|---------------------|
| 1-2 devs | Lint + Unit Tests + Build + Deploy (simple) |
| 3-10 devs | Full pipeline with staging environment |
| 10-50 devs | Multi-environment, parallel tests, canary |
| 50+ devs | Monorepo tools, selective testing, multiple pipelines |

### Keep It Simple When:
- 🟡 Starting a new project (don't over-engineer day one)
- 🟡 Prototype/hackathon code (deploy manually, iterate fast)
- 🟡 One-person team (CI is enough, skip complex CD)

### Go Full Pipeline When:
- ✅ Multiple developers working on same codebase
- ✅ Deploying to production regularly
- ✅ Compliance/audit requirements exist
- ✅ Downtime costs real money

---

## Key Takeaways

- 🔑 A pipeline is **stages → jobs → steps**, progressing from code to production
- 🔑 **Build once, deploy everywhere** — same artifact for all environments, only config differs
- 🔑 **Parallelization** is the #1 way to speed up slow pipelines
- 🔑 Use **caching** aggressively (dependencies, Docker layers, compiled output)
- 🔑 Every pipeline needs **quality gates** — coverage thresholds, security scans, performance checks
- 🔑 **Flaky tests are poison** — quarantine them immediately or you'll lose trust in CI
- 🔑 The pipeline config is **code** — review it, version it, test it like any other code

---

## What's Next?

Now that you understand pipeline structure, the next chapter covers the **tools** that actually run these pipelines — Jenkins, GitHub Actions, GitLab CI, and ArgoCD — with hands-on comparisons and real configurations.

**Next: [Tools — Jenkins, GitHub Actions, GitLab CI, ArgoCD](./03-cicd-tools.md)**
