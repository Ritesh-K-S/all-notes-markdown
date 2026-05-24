# Helm Charts — Package Manager for Kubernetes

> **What you'll learn**: How Helm packages, templates, and manages Kubernetes applications — think of it as "apt-get" or "npm" for Kubernetes. Master chart structure, templating, values overrides, and production lifecycle management.

---

## Real-Life Analogy: IKEA Furniture Instructions

Imagine deploying a Kubernetes app is like assembling IKEA furniture. Without Helm, you get:
- 50 individual parts (YAML files) in a pile
- No instructions on which order to assemble
- Need to manually customize every measurement for your room

**Helm is like getting IKEA furniture with**:
- A **single package** (chart) with everything included
- **Step-by-step instructions** (templates with correct ordering)
- A **measurement form** (values.yaml) where you fill in your room size, color preference — and it adjusts everything automatically
- **Version history** — if the new desk doesn't fit, roll back to the old one

```
Without Helm:                           With Helm:
├── deployment.yaml                     helm install my-app ./my-chart \
├── service.yaml                          --set replicas=5 \
├── configmap.yaml                        --set image.tag=v2.1.0 \
├── secret.yaml                           --set ingress.host=api.myco.com
├── ingress.yaml                        
├── hpa.yaml                            One command. Done. ✅
├── pdb.yaml                            
├── serviceaccount.yaml                 
└── networkpolicy.yaml                  
                                        
9 files to manage manually 😰           
```

---

## What is Helm?

```
┌─────────────────────────────────────────────────────────────────┐
│                         HELM                                     │
│                                                                   │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐       │
│  │   CHART     │     │   VALUES    │     │   RELEASE   │       │
│  │             │     │             │     │             │       │
│  │  Package of │  +  │  Custom     │  =  │  Running    │       │
│  │  templates  │     │  settings   │     │  instance   │       │
│  │             │     │             │     │  in cluster │       │
│  └─────────────┘     └─────────────┘     └─────────────┘       │
│                                                                   │
│  Think:                                                          │
│  Chart = Recipe     Values = Ingredients    Release = Cooked meal│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Key concepts:**
- **Chart**: A package of pre-configured Kubernetes resources (templates + defaults)
- **Values**: Configuration that customizes a chart for your specific needs
- **Release**: A chart installed into a cluster (you can have multiple releases of the same chart)
- **Repository**: A place where charts are stored and shared (like Docker Hub for images)

---

## Helm Chart Structure

```
my-webapp/
├── Chart.yaml              # Chart metadata (name, version, description)
├── Chart.lock              # Locked dependency versions
├── values.yaml             # Default configuration values
├── values-staging.yaml     # Environment-specific overrides
├── values-production.yaml  # Environment-specific overrides
├── templates/              # Kubernetes manifest templates
│   ├── _helpers.tpl        # Template helper functions
│   ├── deployment.yaml     # Deployment template
│   ├── service.yaml        # Service template
│   ├── ingress.yaml        # Ingress template
│   ├── configmap.yaml      # ConfigMap template
│   ├── secret.yaml         # Secret template
│   ├── hpa.yaml            # HorizontalPodAutoscaler
│   ├── pdb.yaml            # PodDisruptionBudget
│   ├── serviceaccount.yaml # ServiceAccount
│   ├── NOTES.txt           # Post-install notes shown to user
│   └── tests/
│       └── test-connection.yaml  # Helm test (verify install)
├── charts/                 # Sub-charts (dependencies)
│   └── redis/             # Bundled dependency
└── .helmignore            # Files to exclude from chart
```

---

## Chart.yaml — The Package Manifest

```yaml
apiVersion: v2                  # Helm 3 chart API version
name: my-webapp
version: 1.2.0                  # Chart version (SemVer)
appVersion: "2.1.0"             # Application version
description: A production-ready web application
type: application               # 'application' or 'library'
keywords:
  - web
  - api
  - flask
maintainers:
  - name: Platform Team
    email: platform@mycompany.com
home: https://github.com/mycompany/my-webapp
sources:
  - https://github.com/mycompany/my-webapp

dependencies:
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled     # Only include if redis.enabled=true
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

---

## values.yaml — Default Configuration

```yaml
# values.yaml — Sensible defaults (override per environment)

# Application settings
replicaCount: 2

image:
  repository: myregistry.com/webapp
  tag: "2.1.0"
  pullPolicy: IfNotPresent

# Service configuration
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# Ingress configuration
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: app.mycompany.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: webapp-tls
      hosts:
        - app.mycompany.com

# Resource limits
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

# Auto-scaling
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# Health checks
healthCheck:
  liveness:
    path: /health
    initialDelaySeconds: 15
    periodSeconds: 10
  readiness:
    path: /ready
    initialDelaySeconds: 5
    periodSeconds: 5

# Environment configuration
config:
  LOG_LEVEL: "info"
  DB_HOST: "postgres.databases"
  DB_PORT: "5432"
  REDIS_HOST: "redis.caching"

# Secrets (reference external secret)
secrets:
  existingSecret: "webapp-secrets"

# Pod disruption budget
podDisruptionBudget:
  enabled: true
  minAvailable: 1

# Redis sub-chart config
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    existingSecret: redis-secret

# PostgreSQL sub-chart config
postgresql:
  enabled: false    # Using external database
```

---

## Templates — The Power of Helm

### Deployment Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-webapp.fullname" . }}
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-webapp.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "my-webapp.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "my-webapp.serviceAccountName" . }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "my-webapp.fullname" . }}-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.secrets.existingSecret }}
                  key: db-password
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.liveness.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.healthCheck.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.liveness.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.readiness.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.healthCheck.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.readiness.periodSeconds }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### Helpers Template

```yaml
# templates/_helpers.tpl

{{/*
Full name of the release (truncated to 63 chars)
*/}}
{{- define "my-webapp.fullname" -}}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-webapp.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{ include "my-webapp.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels (used in matchLabels)
*/}}
{{- define "my-webapp.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Service account name
*/}}
{{- define "my-webapp.serviceAccountName" -}}
{{- default (include "my-webapp.fullname" .) .Values.serviceAccount.name }}
{{- end }}
```

### Conditional Resources

```yaml
# templates/hpa.yaml — Only created if autoscaling is enabled
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-webapp.fullname" . }}
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-webapp.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

---

## How It Works Internally

```
┌────────────────── Helm Install Flow ──────────────────────────────┐
│                                                                     │
│  helm install my-release ./my-webapp -f values-prod.yaml           │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────┐                       │
│  │  1. Load Chart (templates + defaults)    │                       │
│  └──────────────────┬──────────────────────┘                       │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                       │
│  │  2. Merge values:                        │                       │
│  │     defaults ← values-prod.yaml ← --set │                       │
│  └──────────────────┬──────────────────────┘                       │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                       │
│  │  3. Render templates (Go templating)     │                       │
│  │     {{ .Values.replicaCount }} → 5      │                       │
│  │     {{ .Values.image.tag }} → v2.1.0    │                       │
│  └──────────────────┬──────────────────────┘                       │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                       │
│  │  4. Validate rendered YAML               │                       │
│  │     (dry-run against K8s API schemas)    │                       │
│  └──────────────────┬──────────────────────┘                       │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                       │
│  │  5. Apply to cluster (kubectl apply)     │                       │
│  │     In correct order:                    │                       │
│  │     Namespace → SA → ConfigMap →         │                       │
│  │     Secret → Deployment → Service →      │                       │
│  │     Ingress → HPA                        │                       │
│  └──────────────────┬──────────────────────┘                       │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                       │
│  │  6. Store release metadata in cluster    │                       │
│  │     (K8s Secret in helm namespace)       │                       │
│  └─────────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Essential Helm Commands

```bash
# === Chart Management ===
helm create my-chart                     # Scaffold a new chart
helm package ./my-chart                  # Package chart as .tgz
helm lint ./my-chart                     # Validate chart syntax
helm template my-release ./my-chart      # Render templates locally (dry-run)
helm dependency update ./my-chart        # Download sub-chart dependencies

# === Install / Upgrade ===
helm install my-release ./my-chart \
  --namespace production \
  --create-namespace \
  -f values-production.yaml \
  --set image.tag=v2.1.0 \
  --wait --timeout 5m                    # Wait for resources to be ready

# Upgrade existing release (or install if not exists)
helm upgrade --install my-release ./my-chart \
  --namespace production \
  -f values-production.yaml \
  --set image.tag=v2.2.0 \
  --atomic                               # Auto-rollback on failure!

# === Release Management ===
helm list -n production                  # List all releases
helm status my-release -n production     # Show release status
helm history my-release -n production    # Show revision history

# === Rollback ===
helm rollback my-release 3 -n production # Rollback to revision 3
helm rollback my-release 0               # Rollback to previous version

# === Repository Management ===
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo redis                   # Search for charts
helm show values bitnami/redis           # View default values

# === Debugging ===
helm template my-release ./my-chart -f values.yaml --debug  # See rendered output
helm get manifest my-release             # Get deployed manifests
helm get values my-release               # Get applied values
```

---

## Code Examples

### Python — Helm-Ready Application

```python
# app.py — Application that reads config from environment (Helm-friendly)
import os
from flask import Flask, jsonify

app = Flask(__name__)

# All config comes from environment (injected by Helm ConfigMap)
config = {
    'db_host': os.getenv('DB_HOST', 'localhost'),
    'db_port': int(os.getenv('DB_PORT', 5432)),
    'redis_host': os.getenv('REDIS_HOST', 'localhost'),
    'log_level': os.getenv('LOG_LEVEL', 'info'),
    'app_version': os.getenv('APP_VERSION', 'unknown'),
}

@app.route('/health')
def health():
    """Liveness probe — is the process alive?"""
    return jsonify({"status": "alive"}), 200

@app.route('/ready')
def ready():
    """Readiness probe — can we handle traffic?"""
    # Check dependencies
    try:
        # Verify DB connection, Redis connection, etc.
        return jsonify({"status": "ready"}), 200
    except Exception as e:
        return jsonify({"status": "not_ready", "error": str(e)}), 503

@app.route('/info')
def info():
    """App info endpoint (useful for debugging Helm deployments)"""
    return jsonify({
        "version": config['app_version'],
        "config": {k: v for k, v in config.items() if 'password' not in k}
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.getenv('PORT', 8080)))
```

### Java — Helm-Friendly Spring Boot

```java
// application.yml — References environment variables set by Helm
// server:
//   port: ${PORT:8080}
//   shutdown: graceful
// spring:
//   datasource:
//     url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
//   redis:
//     host: ${REDIS_HOST}
// management:
//   endpoints:
//     web:
//       exposure:
//         include: health,info,prometheus
//   endpoint:
//     health:
//       probes:
//         enabled: true    # Enables /actuator/health/liveness and /readiness

@RestController
public class InfoController {
    
    @Value("${APP_VERSION:unknown}")
    private String appVersion;
    
    @Value("${HELM_RELEASE:unknown}")
    private String helmRelease;
    
    @GetMapping("/info")
    public Map<String, String> info() {
        return Map.of(
            "version", appVersion,
            "release", helmRelease,
            "java", System.getProperty("java.version")
        );
    }
}
```

---

## Infrastructure Example: Multi-Environment Deployment

```
┌────── values.yaml (defaults) ───────────────────────────────────┐
│  replicaCount: 2                                                 │
│  image.tag: "latest"                                            │
│  resources.requests.memory: 256Mi                                │
│  ingress.host: app.dev.mycompany.com                            │
└──────────────────────────────────────────────────────────────────┘
         │
         │  Merge (staging overrides defaults)
         ▼
┌────── values-staging.yaml ──────────────────────────────────────┐
│  replicaCount: 3                                                 │
│  image.tag: "v2.1.0-rc1"                                       │
│  ingress.host: app.staging.mycompany.com                        │
└──────────────────────────────────────────────────────────────────┘
         │
         │  Merge (production overrides all)
         ▼
┌────── values-production.yaml ───────────────────────────────────┐
│  replicaCount: 5                                                 │
│  image.tag: "v2.1.0"                                            │
│  resources.requests.memory: 512Mi                                │
│  resources.limits.memory: 1Gi                                    │
│  ingress.host: app.mycompany.com                                │
│  autoscaling.enabled: true                                      │
│  autoscaling.maxReplicas: 20                                    │
└──────────────────────────────────────────────────────────────────┘
```

### CI/CD Pipeline with Helm

```yaml
# .github/workflows/deploy.yaml
name: Deploy with Helm

on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name production-cluster

      - name: Deploy with Helm
        run: |
          helm upgrade --install webapp ./charts/my-webapp \
            --namespace production \
            --create-namespace \
            -f ./charts/my-webapp/values-production.yaml \
            --set image.tag=${{ github.ref_name }} \
            --set appVersion=${{ github.ref_name }} \
            --atomic \
            --timeout 10m \
            --wait

      - name: Verify deployment
        run: |
          helm status webapp -n production
          kubectl rollout status deployment/webapp -n production
```

### Helmfile — Managing Multiple Releases

```yaml
# helmfile.yaml — Manage multiple Helm releases declaratively
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

releases:
  - name: redis
    namespace: caching
    chart: bitnami/redis
    version: 17.11.0
    values:
      - architecture: standalone
      - auth:
          enabled: true
          existingSecret: redis-auth

  - name: webapp
    namespace: production
    chart: ./charts/my-webapp
    values:
      - ./charts/my-webapp/values-production.yaml
    set:
      - name: image.tag
        value: {{ requiredEnv "IMAGE_TAG" }}
    needs:
      - caching/redis    # Deploy redis first

  - name: monitoring
    namespace: monitoring
    chart: prometheus-community/kube-prometheus-stack
    version: 51.0.0
    values:
      - ./charts/monitoring-values.yaml
```

```bash
# Deploy everything in order
helmfile sync

# Diff before applying
helmfile diff

# Deploy specific release
helmfile -l name=webapp sync
```

---

## Real-World Example: How Deliveroo Manages 500+ Services with Helm

```
┌──────────── Deliveroo Helm Strategy ─────────────────────────────┐
│                                                                    │
│  Challenge: 500+ microservices, 200+ engineers deploying daily    │
│                                                                    │
│  Solution: "Golden Chart" Pattern                                 │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Base Chart ("deliveroo-service")                            │ │
│  │  ├── Standardized deployment patterns                       │ │
│  │  ├── Built-in security (non-root, network policies)         │ │
│  │  ├── Prometheus metrics, Datadog integration                │ │
│  │  ├── Standard health check paths                            │ │
│  │  └── PodDisruptionBudget, HPA included                      │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                        │                                          │
│            Used by every service team:                            │
│                        │                                          │
│  ┌─────────┐  ┌───────┴──┐  ┌─────────┐  ┌─────────┐          │
│  │ Payments│  │  Orders  │  │ Riders  │  │ Search  │          │
│  │ Service │  │ Service  │  │ Service │  │ Service │          │
│  │         │  │          │  │         │  │         │          │
│  │values:  │  │values:   │  │values:  │  │values:  │          │
│  │ image:..│  │ image:.. │  │ image:..│  │ image:..│          │
│  │ replicas│  │ replicas │  │ replicas│  │ replicas│          │
│  └─────────┘  └──────────┘  └─────────┘  └─────────┘          │
│                                                                    │
│  Result: Teams only define their unique values                    │
│  Platform team maintains the chart with best practices            │
└────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Problem | Solution |
|---------|---------|----------|
| Not using `--atomic` flag | Failed deploys leave cluster in broken state | Always use `--atomic` (auto-rollback) |
| Hardcoding values in templates | Can't override per environment | Use `{{ .Values.x }}` for everything |
| No `NOTES.txt` | Users don't know how to access the app | Add helpful post-install instructions |
| Storing charts only locally | Teams can't share, no versioning | Use a chart repository (ChartMuseum, OCI) |
| Not pinning chart dependency versions | Breaking changes from upstream | Pin exact versions in Chart.yaml |
| Forgetting `helm dependency update` | Sub-charts missing during install | Add to CI or use `helm dependency build` |
| Using `helm install` in CI (not upgrade) | Fails if release already exists | Always use `helm upgrade --install` |
| Too many `--set` flags | Unreadable, hard to track | Use values files, only `--set` for image tag |

---

## When to Use / When NOT to Use

### ✅ Use Helm When:
- You deploy the **same app to multiple environments** (dev, staging, prod)
- You have **reusable deployment patterns** across teams (golden chart)
- You need **versioned, rollback-able deployments**
- You want to use **community charts** (Redis, PostgreSQL, Prometheus)
- Your CI/CD pipeline needs **repeatable, parameterized deploys**

### ❌ Don't Use Helm When:
- You have **1-2 simple YAML files** (just use kubectl apply)
- You're using **GitOps with Kustomize** (alternative approach)
- Your team is **very small** and Helm's templating adds confusion
- You need **real-time, drift-detection reconciliation** (use ArgoCD/Flux with Helm)

### Helm vs Kustomize Decision

```
┌────────────────────────────────────────────────────────────────┐
│                    Helm vs Kustomize                             │
│                                                                  │
│  Helm:                          Kustomize:                      │
│  ├── Full templating engine     ├── Overlay/patch based         │
│  ├── Package management         ├── No packaging concept        │
│  ├── Chart repositories         ├── Native to kubectl           │
│  ├── Release lifecycle          ├── No release tracking         │
│  ├── Complex logic (if/range)   ├── Simple overlays             │
│  ├── Steep learning curve       ├── Gentle learning curve       │
│  └── Best for: shared charts,   └── Best for: team-owned apps, │
│      community packages              GitOps workflows            │
│                                                                  │
│  Many teams use BOTH: Helm for 3rd-party, Kustomize for own apps│
└────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Helm is the package manager for Kubernetes** — it bundles templates + values into reusable, versioned charts
2. **Templates + values = rendered manifests** — same chart, different values for dev/staging/prod
3. **Always use `helm upgrade --install --atomic`** in CI/CD for idempotent, safe deployments
4. **Community charts** (Bitnami, Prometheus) save weeks of work — don't reinvent the wheel
5. **"Golden chart" pattern** lets platform teams enforce standards while giving developers flexibility
6. **Helmfile** manages multiple releases declaratively with dependencies and ordering
7. **Version your charts** separately from your app — chart version ≠ app version

---

## What's Next?

For truly advanced traffic management, observability, and security at the network level, let's explore service meshes in [Chapter 16.8: Service Mesh (Istio, Linkerd)](./08-service-mesh.md).
