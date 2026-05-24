# Tools — Jenkins, GitHub Actions, GitLab CI, ArgoCD

> **What you'll learn**: A deep comparison of the most popular CI/CD tools — their architectures, strengths, weaknesses, and when to choose each one, with real pipeline configurations for every tool.

---

## Real-Life Analogy

Think of CI/CD tools like **different types of delivery services**:

- **Jenkins** = Owning your own delivery fleet. Maximum control, you maintain everything — trucks, drivers, routes. Powerful, but expensive to manage.
- **GitHub Actions** = Using the postal service that's built into your mailbox. Super convenient if you already use GitHub. No setup, just works.
- **GitLab CI** = An all-in-one logistics company. They handle warehousing (code), shipping (CI), and delivery tracking (monitoring) in one platform.
- **ArgoCD** = A specialized "last-mile" delivery service. It doesn't build your package — it ensures what's in your warehouse (Git) matches what's on the shelf (Kubernetes).

Each tool is excellent at something different. Let's explore them all.

---

## Core Concept Explained Step-by-Step

### The CI/CD Tool Landscape

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CI/CD TOOL CATEGORIES                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  GENERAL-PURPOSE CI/CD (Build + Test + Deploy):                          │
│  ┌────────────┐ ┌────────────────┐ ┌────────────────┐ ┌──────────────┐  │
│  │  Jenkins   │ │ GitHub Actions │ │  GitLab CI/CD  │ │  CircleCI    │  │
│  └────────────┘ └────────────────┘ └────────────────┘ └──────────────┘  │
│                                                                           │
│  CD-ONLY (Deployment focused):                                           │
│  ┌────────────┐ ┌────────────────┐ ┌────────────────┐                   │
│  │  ArgoCD    │ │    Flux CD     │ │   Spinnaker    │                   │
│  └────────────┘ └────────────────┘ └────────────────┘                   │
│                                                                           │
│  CLOUD-NATIVE CI:                                                        │
│  ┌────────────┐ ┌────────────────┐ ┌────────────────┐                   │
│  │  AWS Code  │ │  Azure DevOps  │ │  Google Cloud  │                   │
│  │  Pipeline  │ │   Pipelines    │ │     Build      │                   │
│  └────────────┘ └────────────────┘ └────────────────┘                   │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Tool 1: Jenkins

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     JENKINS ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────┐                                    │
│  │     JENKINS CONTROLLER   │ ← Orchestrates everything          │
│  │                          │                                    │
│  │  • Job scheduling        │                                    │
│  │  • Web UI                │                                    │
│  │  • Plugin management     │                                    │
│  │  • Configuration storage │                                    │
│  └──────────────────────────┘                                    │
│           │         │         │                                   │
│           ▼         ▼         ▼                                   │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                          │
│  │ Agent 1 │  │ Agent 2 │  │ Agent 3 │  ← Execute actual work   │
│  │ (Linux) │  │ (Windows│  │ (Docker)│                           │
│  │         │  │         │  │         │                           │
│  └─────────┘  └─────────┘  └─────────┘                          │
│                                                                   │
│  Communication: JNLP or SSH between controller & agents          │
│  Storage: Jobs stored as XML on controller filesystem            │
│  Plugins: 1,800+ plugins for every imaginable integration        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Jenkins Pipeline (Jenkinsfile)

```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'registry.company.com'
        APP_NAME = 'my-service'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean compile -DskipTests'
            }
        }
        
        stage('Test') {
            parallel {  // Run tests in parallel
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'mvn verify -Pintegration'
                    }
                }
                stage('Security Scan') {
                    steps {
                        sh 'trivy fs --severity HIGH,CRITICAL .'
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def image = docker.build(
                        "${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}"
                    )
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'registry-creds') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER} \
                        --namespace=staging
                """
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Yes, deploy!"
            }
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER} \
                        --namespace=production
                """
            }
        }
    }
    
    post {
        failure {
            slackSend channel: '#deployments',
                      color: 'danger',
                      message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        success {
            slackSend channel: '#deployments',
                      color: 'good',
                      message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

### Jenkins Pros & Cons

| Pros | Cons |
|------|------|
| Extremely flexible (1800+ plugins) | Complex to set up and maintain |
| Self-hosted (full control) | UI feels dated |
| Massive community | Plugin compatibility issues |
| Supports any language/platform | Requires dedicated infrastructure |
| Pipeline-as-code (Jenkinsfile) | Groovy DSL has a learning curve |
| Battle-tested (20+ years) | Security patches require attention |

---

## Tool 2: GitHub Actions

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  GITHUB ACTIONS ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────┐        │
│  │                 GITHUB.COM                            │        │
│  │                                                      │        │
│  │  Repository ──▶ Event (push/PR) ──▶ Workflow Engine  │        │
│  │                                         │            │        │
│  │                                         ▼            │        │
│  │                                    ┌─────────┐       │        │
│  │                                    │Scheduler│       │        │
│  │                                    └─────────┘       │        │
│  └──────────────────────────────────────────────────────┘        │
│                          │                                        │
│              ┌───────────┼───────────┐                            │
│              ▼           ▼           ▼                            │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────────┐   │
│  │ GitHub-hosted  │ │ GitHub-hosted  │ │ Self-hosted Runner │   │
│  │ Runner (Linux) │ │ Runner (macOS) │ │ (your machine)     │   │
│  │                │ │                │ │                    │   │
│  │ Fresh VM each  │ │ Fresh VM each  │ │ Persistent         │   │
│  │ run. Auto-     │ │ run. More      │ │ You manage it.     │   │
│  │ cleaned after. │ │ expensive.     │ │ Free minutes.      │   │
│  └────────────────┘ └────────────────┘ └────────────────────┘   │
│                                                                   │
│  Key Concepts:                                                    │
│  • Workflow = YAML file in .github/workflows/                    │
│  • Action = Reusable step (from marketplace or custom)           │
│  • Runner = Machine that executes jobs                            │
│  • Secret = Encrypted variable (injected at runtime)             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### GitHub Actions Pipeline

```yaml
# .github/workflows/full-pipeline.yml
name: Full CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # ─── Quality Checks (parallel) ───
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # ─── Tests (parallel, depends on nothing) ───
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
      - name: Upload coverage
        uses: codecov/codecov-action@v4

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - name: Run Playwright E2E tests
        run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  # ─── Build & Push (depends on tests passing) ───
  build:
    needs: [lint, security, unit-tests, e2e-tests]
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ─── Deploy (only on main, requires approval) ───
  deploy:
    needs: [build]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # Has required reviewers
    steps:
      - name: Deploy to EKS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: task-definition.json
          service: my-service
          cluster: production
```

### GitHub Actions: Reusable Workflows

```yaml
# .github/workflows/reusable-deploy.yml (shared template)
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      KUBE_CONFIG:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
      - name: Deploy
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          kubectl set image deployment/app app=${{ inputs.image-tag }}
```

### GitHub Actions Pros & Cons

| Pros | Cons |
|------|------|
| Zero setup (built into GitHub) | Vendor lock-in to GitHub |
| Huge marketplace (20,000+ actions) | Debugging failed runs can be tricky |
| Free for public repos | Expensive at scale (runner minutes) |
| Matrix builds built-in | Limited self-hosted runner features |
| Great for open-source projects | YAML can get complex for large pipelines |
| Tight integration with PRs | No built-in dashboard/analytics |

---

## Tool 3: GitLab CI/CD

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   GITLAB CI/CD ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                GITLAB SERVER                              │    │
│  │                                                          │    │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────────┐ │    │
│  │  │  Source   │  │  CI/CD    │  │  Registry / Package  │ │    │
│  │  │  Code     │  │  Engine   │  │  Registry / Pages    │ │    │
│  │  └──────────┘  └───────────┘  └──────────────────────┘ │    │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────────┐ │    │
│  │  │  Issues   │  │  Security │  │  Environments /      │ │    │
│  │  │  & Boards │  │  Scanning │  │  Monitoring          │ │    │
│  │  └──────────┘  └───────────┘  └──────────────────────┘ │    │
│  └──────────────────────────────────────────────────────────┘    │
│                              │                                    │
│              ┌───────────────┼───────────────┐                   │
│              ▼               ▼               ▼                   │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐       │
│  │ GitLab Runner  │ │ GitLab Runner  │ │ GitLab Runner  │       │
│  │ (Shared)       │ │ (Group)        │ │ (Project)      │       │
│  │                │ │                │ │                │       │
│  │ Executor:      │ │ Executor:      │ │ Executor:      │       │
│  │ - Docker       │ │ - Kubernetes   │ │ - Shell        │       │
│  │ - Docker+Machine│ │ - Docker       │ │ - Docker       │       │
│  └────────────────┘ └────────────────┘ └────────────────┘       │
│                                                                   │
│  All-in-one platform: Code → CI → Registry → Deploy → Monitor   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### GitLab CI Pipeline

```yaml
# .gitlab-ci.yml
include:
  - template: Security/SAST.gitlab-ci.yml  # Built-in security scanning

stages:
  - build
  - test
  - security
  - package
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# ─── Build Stage ───
build:
  stage: build
  image: node:20-alpine
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

# ─── Test Stage (parallel jobs) ───
unit-tests:
  stage: test
  image: node:20-alpine
  needs: [build]  # DAG: only needs build, not full stage
  script:
    - npm ci
    - npm run test:unit -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'  # Regex to extract coverage %
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

integration-tests:
  stage: test
  image: node:20-alpine
  needs: [build]
  services:
    - name: postgres:15
      alias: db
    - name: redis:7
      alias: cache
  variables:
    DATABASE_URL: postgresql://postgres:password@db:5432/test
    REDIS_URL: redis://cache:6379
    POSTGRES_PASSWORD: password
  script:
    - npm ci
    - npm run test:integration

# ─── Package Stage ───
docker-build:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  needs: [unit-tests, integration-tests]
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ─── Deploy Stages ───
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: [docker-build]
  script:
    - kubectl config use-context staging
    - |
      helm upgrade --install my-app ./helm-chart \
        --namespace staging \
        --set image.tag=$CI_COMMIT_SHA
  environment:
    name: staging
    url: https://staging.myapp.com
    on_stop: stop-staging

deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: [deploy-staging]
  script:
    - kubectl config use-context production
    - |
      helm upgrade --install my-app ./helm-chart \
        --namespace production \
        --set image.tag=$CI_COMMIT_SHA
  environment:
    name: production
    url: https://myapp.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # Requires manual click
```

### GitLab CI Pros & Cons

| Pros | Cons |
|------|------|
| All-in-one platform (code + CI + registry + deploy) | Heavier than needed if you only want CI |
| Built-in security scanning (SAST, DAST, deps) | Self-hosted requires significant resources |
| Auto DevOps (zero-config pipeline) | Complex YAML for large projects |
| Built-in container registry | Runner management overhead |
| Excellent Kubernetes integration | Feature parity varies between tiers |
| DAG-based pipeline execution | Can be expensive (Premium/Ultimate) |

---

## Tool 4: ArgoCD

### What Makes ArgoCD Different?

ArgoCD is **NOT** a CI tool. It's a **GitOps continuous delivery** tool specifically for Kubernetes. It doesn't build or test your code — it ensures your Kubernetes cluster matches what's defined in Git.

```
┌─────────────────────────────────────────────────────────────────┐
│            WHERE ARGOCD FITS IN THE PIPELINE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │
│  │   Code   │──▶│  Build   │──▶│   Test   │──▶│ Push to  │   │
│  │  Change  │    │  (CI)    │    │  (CI)    │    │ Registry │   │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘   │
│                                                        │          │
│       Jenkins / GitHub Actions / GitLab CI              │          │
│  ─────────────────────────────────────────────────────────────   │
│                                                        │          │
│                                                        ▼          │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                    UPDATE GIT REPO                         │    │
│  │  (Change image tag in Kubernetes manifests / Helm values) │    │
│  └──────────────────────────────────────────────────────────┘    │
│                              │                                    │
│       ArgoCD takes over here │                                    │
│  ─────────────────────────────────────────────────────────────   │
│                              ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                      ARGOCD                               │    │
│  │                                                          │    │
│  │  1. Watches Git repo for changes                          │    │
│  │  2. Compares desired state (Git) vs actual (K8s cluster) │    │
│  │  3. Syncs: applies changes to make cluster match Git      │    │
│  │  4. Reports health status                                 │    │
│  └──────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ▼                                    │
│                    Kubernetes Cluster                              │
│                    (pods updated automatically)                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### ArgoCD Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    ARGOCD ARCHITECTURE                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │              ARGOCD SERVER (runs in K8s)                  │     │
│  │                                                          │     │
│  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │     │
│  │  │  API Server │  │  Repo Server │  │  Application  │  │     │
│  │  │             │  │              │  │  Controller   │  │     │
│  │  │ Web UI +    │  │ Clones Git   │  │              │  │     │
│  │  │ gRPC API    │  │ repos, renders│  │ Reconciles   │  │     │
│  │  │             │  │ manifests     │  │ desired vs   │  │     │
│  │  │             │  │ (Helm, Kust.) │  │ actual state │  │     │
│  │  └─────────────┘  └──────────────┘  └───────────────┘  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                              │                                     │
│         ┌────────────────────┼────────────────────┐               │
│         ▼                    ▼                    ▼               │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │  Git Repo   │    │  K8s Cluster │    │  K8s Cluster │        │
│  │  (desired   │    │  (staging)   │    │  (prod)      │        │
│  │   state)    │    │              │    │              │        │
│  └─────────────┘    └──────────────┘    └──────────────┘        │
│                                                                    │
│  Sync Loop (every 3 minutes or webhook-triggered):                │
│  1. Pull manifests from Git                                        │
│  2. Render templates (Helm/Kustomize)                              │
│  3. Compare with live cluster state                                │
│  4. If different → OUT OF SYNC (alert + optional auto-sync)        │
│  5. Apply diff to cluster                                          │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### ArgoCD Application Definition

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  
  # WHERE to get desired state
  source:
    repoURL: https://github.com/company/k8s-manifests.git
    targetRevision: main
    path: apps/my-app/overlays/production
    # For Helm charts:
    # helm:
    #   valueFiles:
    #     - values-production.yaml
  
  # WHERE to deploy
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  # HOW to sync
  syncPolicy:
    automated:
      prune: true       # Delete resources removed from Git
      selfHeal: true    # Revert manual changes in cluster
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### ArgoCD with Kustomize (Multi-Environment)

```
# Repository structure for ArgoCD
k8s-manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── staging/
│   │   ├── kustomization.yaml    ← replicas: 2, staging image
│   │   └── patch-resources.yaml
│   └── production/
│       ├── kustomization.yaml    ← replicas: 5, prod image
│       └── patch-resources.yaml
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: registry.company.com/my-app:latest  # Will be overridden
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../../base
images:
  - name: registry.company.com/my-app
    newTag: abc123def  # Updated by CI pipeline
replicas:
  - name: my-app
    count: 5
```

### ArgoCD Pros & Cons

| Pros | Cons |
|------|------|
| GitOps native (Git = single source of truth) | Only for Kubernetes (not for VMs, Lambda, etc.) |
| Beautiful web UI showing app health | Requires separate CI tool for build/test |
| Auto-sync + self-heal (drift detection) | Learning curve for GitOps workflow |
| Multi-cluster support | Can be complex for simple deployments |
| RBAC for deployment approvals | Secrets management requires extra tooling |
| Declarative app definitions | Webhook setup needed for instant sync |

---

## Tool Comparison Matrix

```
┌────────────────────┬────────────┬───────────────┬────────────┬──────────┐
│ Feature            │ Jenkins    │ GitHub Actions│ GitLab CI  │ ArgoCD   │
├────────────────────┼────────────┼───────────────┼────────────┼──────────┤
│ Type               │ CI + CD    │ CI + CD       │ CI + CD    │ CD only  │
│ Hosting            │ Self-hosted│ Cloud/Self    │ Cloud/Self │ Self(K8s)│
│ Config Format      │ Groovy     │ YAML          │ YAML       │ YAML     │
│ Learning Curve     │ Steep      │ Easy          │ Moderate   │ Moderate │
│ Plugin Ecosystem   │ 1800+      │ 20000+ actions│ Built-in   │ Limited  │
│ K8s Integration    │ Via plugin │ Via action    │ Built-in   │ Native   │
│ GitOps Support     │ Manual     │ Manual        │ Partial    │ Native   │
│ Container Registry │ Plugin     │ GHCR built-in │ Built-in   │ N/A      │
│ Security Scanning  │ Plugin     │ Via actions   │ Built-in   │ N/A      │
│ Cost               │ Free (OSS) │ Free/Paid     │ Free/Paid  │ Free(OSS)│
│ Best For           │ Enterprise │ GitHub users  │ All-in-one │ K8s CD   │
│                    │ complex    │ open-source   │ DevOps     │ GitOps   │
└────────────────────┴────────────┴───────────────┴────────────┴──────────┘
```

---

## How It Works Internally

### Runner Execution Models Compared

```
┌─────────────────────────────────────────────────────────────────────┐
│              HOW EACH TOOL EXECUTES JOBS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  JENKINS:                                                             │
│  Controller ──▶ Agent (persistent machine) ──▶ Execute in workspace  │
│  • Agent stays running between jobs                                   │
│  • Workspace may persist (or be cleaned)                              │
│  • Max flexibility, max maintenance                                   │
│                                                                       │
│  GITHUB ACTIONS:                                                      │
│  Event ──▶ Fresh VM provisioned ──▶ Execute ──▶ VM destroyed         │
│  • Every run gets a brand new machine                                 │
│  • No state leaks between runs                                        │
│  • Cache layer for dependencies                                       │
│                                                                       │
│  GITLAB CI:                                                           │
│  Event ──▶ Runner picks job ──▶ Spins Docker container ──▶ Destroyed │
│  • Runner is long-lived, containers are ephemeral                     │
│  • Docker executor most common                                        │
│  • Kubernetes executor for auto-scaling                               │
│                                                                       │
│  ARGOCD:                                                              │
│  Git change ──▶ Controller detects diff ──▶ kubectl apply            │
│  • No "runner" concept — it IS the deployer                           │
│  • Runs as pods inside the K8s cluster                                │
│  • Continuous reconciliation loop                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Common Tool Combinations in Production

```
┌─────────────────────────────────────────────────────────────────────┐
│              POPULAR CI/CD TOOL STACKS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Startup (small team):                                                │
│  GitHub + GitHub Actions + Vercel/Railway                             │
│  (Simple, cheap, fast to set up)                                      │
│                                                                       │
│  Mid-size Company:                                                    │
│  GitLab (self-hosted) + GitLab CI + ArgoCD + K8s                     │
│  (All-in-one source + CI, GitOps deployment)                         │
│                                                                       │
│  Enterprise:                                                          │
│  GitHub Enterprise + Jenkins + ArgoCD + Multiple K8s clusters        │
│  (Maximum control, compliance, multi-team)                            │
│                                                                       │
│  Cloud-Native:                                                        │
│  GitHub + GitHub Actions (CI) + ArgoCD (CD) + EKS                    │
│  (Best of both: Actions for build/test, Argo for deploy)             │
│                                                                       │
│  Google/Netflix Scale:                                                 │
│  Custom internal tools (Borg, Spinnaker)                              │
│  (Nothing off-the-shelf works at their scale)                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Solution |
|---------|--------|----------|
| **Choosing Jenkins for a 3-person team** | Maintaining Jenkins becomes a full-time job | Use GitHub Actions or GitLab CI instead |
| **Using only ArgoCD (no CI)** | No automated testing before deploy | Combine ArgoCD with a CI tool |
| **Not using secrets management** | API keys exposed in pipeline logs | Use GitHub Secrets, Vault, or sealed-secrets |
| **Running CI on self-hosted without security** | Untrusted PRs can run arbitrary code on your infra | Use sandboxed runners or ephemeral VMs |
| **No pipeline caching** | 5-minute install step every single run | Cache node_modules, .m2, pip cache |
| **Ignoring runner costs** | $10K/month GitHub Actions bill surprise | Monitor usage; use self-hosted for heavy jobs |

---

## When to Use / When NOT to Use

| Choose This | When You Need |
|-------------|---------------|
| **Jenkins** | Maximum flexibility, complex enterprise workflows, self-hosted requirement, Windows/.NET heavy workloads |
| **GitHub Actions** | You're already on GitHub, open-source projects, quick setup, don't want to manage infrastructure |
| **GitLab CI** | All-in-one platform (code + CI + registry + security), self-hosted requirement with better UX than Jenkins |
| **ArgoCD** | Kubernetes deployments, GitOps workflow, multi-cluster management, deployment drift detection |
| **Actions + ArgoCD** | Best combo for K8s: Actions handles CI (build/test), ArgoCD handles CD (deploy) |

---

## Key Takeaways

- 🔑 **Jenkins** = maximum power and flexibility, but high maintenance cost
- 🔑 **GitHub Actions** = easiest to start, best for GitHub-hosted projects
- 🔑 **GitLab CI** = best all-in-one platform (source + CI + registry + security)
- 🔑 **ArgoCD** = best for Kubernetes GitOps deployments (not a CI tool!)
- 🔑 **Combine tools**: Use GitHub Actions/GitLab for CI, ArgoCD for CD — separation of concerns
- 🔑 The "best" tool depends on your team size, existing stack, and deployment target
- 🔑 All modern tools use **declarative YAML configs** stored alongside your code

---

## What's Next?

Now that you know the tools, the next chapter explores **GitOps** — the practice of using Git as the single source of truth for both application code AND infrastructure, with ArgoCD and Flux as the enforcement mechanism.

**Next: [GitOps — Infrastructure Managed Through Git](./04-gitops.md)**
