# GitOps — Infrastructure Managed Through Git

> **What you'll learn**: How GitOps uses Git as the single source of truth for both application code AND infrastructure — making deployments auditable, repeatable, and self-healing.

---

## Real-Life Analogy

Imagine you're an **architect** designing a building.

**Traditional approach (pre-GitOps):**
You draw blueprints, hand them to the construction crew, they build the building. Six months later, someone adds a wall here, removes a door there — nobody updates the blueprints. Soon, the blueprints don't match the actual building at all. When something breaks, nobody knows what it was *supposed* to look like.

**GitOps approach:**
You keep the blueprints in a **locked glass case** (Git). Anytime someone wants to change the building, they MUST first update the blueprints (submit a pull request). A robot inspector (ArgoCD/Flux) constantly compares the actual building against the blueprints. If anything is different — someone moved a wall without updating the blueprint — the robot automatically fixes it back to match the blueprint.

**The blueprint (Git) is the truth. The building (cluster) always converges to match.**

---

## Core Concept Explained Step-by-Step

### Step 1: What is GitOps?

**GitOps** is a set of practices where:
1. The **entire system** (app code + infrastructure + configuration) is described **declaratively**
2. The desired state is stored in **Git** (the source of truth)
3. Changes are made through **pull requests** (auditable, reviewable)
4. An **agent** automatically ensures the live system matches Git

```
┌─────────────────────────────────────────────────────────────────────┐
│                     GITOPS CORE PRINCIPLES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. DECLARATIVE                                                       │
│     "I want 3 replicas running image v2.1"                            │
│     (NOT: "SSH into server, run docker pull, restart")                │
│                                                                       │
│  2. VERSIONED & IMMUTABLE                                             │
│     Everything in Git. Every change = a commit.                       │
│     Full history. Easy rollback (git revert).                         │
│                                                                       │
│  3. PULLED AUTOMATICALLY                                              │
│     Agent PULLS desired state from Git (not pushed by CI)             │
│     No external access to cluster needed!                             │
│                                                                       │
│  4. CONTINUOUSLY RECONCILED                                           │
│     Agent constantly checks: Does cluster match Git?                  │
│     If not → auto-fix (self-healing)                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Step 2: GitOps vs Traditional CI/CD — The Key Difference

```
┌─────────────────────────────────────────────────────────────────────┐
│            TRADITIONAL CI/CD (PUSH-BASED)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Developer ──▶ Git Push ──▶ CI Server ──▶ Build ──▶ Test             │
│                                                      │                │
│                                                      ▼                │
│                                               CI PUSHES to cluster    │
│                                               (kubectl apply)         │
│                                                      │                │
│                                                      ▼                │
│                                               Kubernetes Cluster      │
│                                                                       │
│  Problem: CI server needs cluster credentials (security risk)         │
│  Problem: If someone changes cluster directly, CI doesn't know        │
│  Problem: No way to detect "drift" from desired state                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│            GITOPS (PULL-BASED)                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Developer ──▶ Git Push ──▶ CI Server ──▶ Build ──▶ Test             │
│                                                      │                │
│                                                      ▼                │
│                                        CI updates Git repo            │
│                                        (new image tag)                │
│                                                      │                │
│                                                      ▼                │
│                                              Git Repository           │
│                                             (desired state)           │
│                                                      │                │
│                                              Agent PULLS from Git     │
│                                                      │                │
│                                                      ▼                │
│                                               Kubernetes Cluster      │
│                                                                       │
│  ✅ Cluster pulls changes (no external access needed)                │
│  ✅ Agent detects drift and self-heals                               │
│  ✅ Git history = deployment history (audit trail)                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Step 3: The GitOps Workflow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE GITOPS WORKFLOW                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  TWO REPOSITORIES:                                                        │
│                                                                           │
│  ┌───────────────────────┐         ┌───────────────────────────┐         │
│  │  APP REPO             │         │  CONFIG REPO               │         │
│  │  (application code)   │         │  (K8s manifests/Helm)      │         │
│  │                       │         │                           │         │
│  │  src/                 │         │  apps/                    │         │
│  │  tests/               │         │    my-app/                │         │
│  │  Dockerfile           │         │      base/                │         │
│  │  .github/workflows/   │         │      staging/             │         │
│  └───────────────────────┘         │      production/          │         │
│           │                         └───────────────────────────┘         │
│           │ 1. Push code                        ▲                         │
│           ▼                                     │                         │
│  ┌───────────────────────┐                     │ 3. CI updates           │
│  │  CI Pipeline          │                     │    image tag            │
│  │  (GitHub Actions)     │─────────────────────┘                         │
│  │                       │                                               │
│  │  Build → Test → Push  │                                               │
│  │  Docker image to      │   2. Push image                               │
│  │  registry             │──────────────┐                                │
│  └───────────────────────┘              ▼                                │
│                                  ┌──────────────┐                        │
│                                  │  Container   │                        │
│                                  │  Registry    │                        │
│                                  └──────────────┘                        │
│                                         ▲                                │
│                                         │ 5. Pull image                  │
│                                         │                                │
│  ┌───────────────────────────────────────────────────┐                  │
│  │                 ARGOCD / FLUX                      │                  │
│  │                                                   │                  │
│  │  4. Detect change in Config Repo                  │                  │
│  │  5. Compare desired state vs actual state         │                  │
│  │  6. Apply diff to Kubernetes cluster              │                  │
│  │                                                   │                  │
│  └───────────────────────────────────────────────────┘                  │
│                         │                                                │
│                         ▼                                                │
│              ┌─────────────────────┐                                    │
│              │  Kubernetes Cluster  │                                    │
│              │  (actual state)      │                                    │
│              └─────────────────────┘                                    │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Step 4: Repository Structure for GitOps

**Best practice: Separate application code from deployment configuration.**

```
# Application Repository (contains code + Dockerfile)
my-app/
├── src/
├── tests/
├── Dockerfile
├── .github/workflows/ci.yml    ← Only builds + tests + pushes image
└── README.md

# Configuration Repository (contains K8s manifests)
k8s-config/
├── apps/
│   ├── my-app/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── ingress.yaml
│   │   │   └── kustomization.yaml
│   │   ├── overlays/
│   │   │   ├── dev/
│   │   │   │   ├── kustomization.yaml
│   │   │   │   └── patch-replicas.yaml
│   │   │   ├── staging/
│   │   │   │   ├── kustomization.yaml
│   │   │   │   └── patch-replicas.yaml
│   │   │   └── production/
│   │   │       ├── kustomization.yaml
│   │   │       ├── patch-replicas.yaml
│   │   │       └── patch-resources.yaml
│   │   └── argocd-app.yaml
│   ├── another-service/
│   └── ...
├── infrastructure/
│   ├── cert-manager/
│   ├── ingress-nginx/
│   ├── monitoring/
│   └── ...
└── README.md
```

---

### Step 5: The Reconciliation Loop

The heart of GitOps is **continuous reconciliation**:

```
┌────────────────────────────────────────────────────────────────┐
│               RECONCILIATION LOOP (runs every 3 min)            │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│              ┌────────────┐                                      │
│              │   START    │                                      │
│              └─────┬──────┘                                      │
│                    │                                              │
│                    ▼                                              │
│         ┌──────────────────┐                                     │
│         │ Pull latest from │                                     │
│         │ Git repository   │                                     │
│         └────────┬─────────┘                                     │
│                  │                                                │
│                  ▼                                                │
│         ┌──────────────────┐                                     │
│         │ Render manifests │ (Helm template / Kustomize build)   │
│         │ = DESIRED state  │                                     │
│         └────────┬─────────┘                                     │
│                  │                                                │
│                  ▼                                                │
│         ┌──────────────────┐                                     │
│         │ Get ACTUAL state │ (kubectl get from cluster)          │
│         │ from K8s API     │                                     │
│         └────────┬─────────┘                                     │
│                  │                                                │
│                  ▼                                                │
│         ┌──────────────────┐                                     │
│         │  Compare:        │                                     │
│         │  Desired == Actual? │                                  │
│         └────────┬─────────┘                                     │
│                  │                                                │
│           ┌──────┴──────┐                                        │
│           │             │                                        │
│          YES           NO                                        │
│           │             │                                        │
│           ▼             ▼                                        │
│    ┌──────────┐  ┌──────────────┐                               │
│    │  SYNCED  │  │ OUT OF SYNC  │                               │
│    │  (done)  │  │              │                               │
│    └──────────┘  └──────┬───────┘                               │
│                         │                                        │
│                         ▼                                        │
│                  ┌──────────────┐                                │
│                  │  Auto-Sync?  │                                │
│                  └──────┬───────┘                                │
│                         │                                        │
│                  ┌──────┴──────┐                                 │
│                 YES           NO                                  │
│                  │             │                                  │
│                  ▼             ▼                                  │
│           ┌──────────┐  ┌──────────────┐                        │
│           │  Apply   │  │ Alert human  │                        │
│           │  changes │  │ (manual sync)│                        │
│           └──────────┘  └──────────────┘                        │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Drift Detection — How ArgoCD Knows Something Changed

ArgoCD uses a **three-way diff** to detect drift:

```
┌─────────────────────────────────────────────────────────────────┐
│                    THREE-WAY DIFF                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. DESIRED STATE    ← What Git says the resource should be       │
│  2. LIVE STATE       ← What's actually running in the cluster     │
│  3. LAST APPLIED    ← What was last applied (annotation on K8s)  │
│                                                                   │
│  Scenarios:                                                       │
│                                                                   │
│  Git says replicas=3, Cluster has replicas=3  → ✅ IN SYNC       │
│  Git says replicas=3, Cluster has replicas=5  → ❌ OUT OF SYNC   │
│     (Someone manually scaled without updating Git)                │
│  Git says replicas=5, Cluster has replicas=3  → ❌ OUT OF SYNC   │
│     (Git was updated, cluster not yet synced)                     │
│                                                                   │
│  Self-Heal: If enabled, Argo reverts cluster to match Git        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Image Updater — Automating Tag Updates

Instead of manually updating image tags in Git, tools like **Argo CD Image Updater** can do it automatically:

```
┌─────────────────────────────────────────────────────────────────┐
│              AUTOMATED IMAGE TAG UPDATE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  CI Pipeline pushes: registry.com/my-app:v1.2.3                  │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────┐                                    │
│  │  Argo Image Updater      │                                    │
│  │                          │                                    │
│  │  Watches container       │                                    │
│  │  registry for new tags   │                                    │
│  │  matching policy:        │                                    │
│  │  "semver: ^1.x"          │                                    │
│  └──────────────────────────┘                                    │
│         │                                                         │
│         │ Detects new tag v1.2.3                                  │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────┐                                    │
│  │  Commits to Git:         │                                    │
│  │  image.tag: v1.2.3       │                                    │
│  └──────────────────────────┘                                    │
│         │                                                         │
│         │ ArgoCD detects Git change                               │
│         ▼                                                         │
│  ┌──────────────────────────┐                                    │
│  │  ArgoCD syncs cluster    │                                    │
│  │  to new image            │                                    │
│  └──────────────────────────┘                                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Secrets in GitOps — The Challenge

You can't store secrets in plain text in Git. Solutions:

```
┌─────────────────────────────────────────────────────────────────┐
│              SECRETS MANAGEMENT IN GITOPS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Option 1: Sealed Secrets (Bitnami)                              │
│  ─────────────────────────────────                               │
│  Encrypt secrets with cluster's public key.                       │
│  Store encrypted version in Git. Only your cluster can decrypt.   │
│                                                                   │
│  Git stores: SealedSecret (encrypted) → K8s decrypts → Secret   │
│                                                                   │
│  Option 2: External Secrets Operator                             │
│  ────────────────────────────────                                │
│  Git stores a reference: "get secret X from AWS Secrets Manager"  │
│  Operator fetches actual value at runtime.                        │
│                                                                   │
│  Git stores:         ExternalSecret { key: "prod/db-password" }  │
│  Operator creates:   Secret { data: "actual-password-value" }    │
│                                                                   │
│  Option 3: SOPS (Mozilla)                                        │
│  ───────────────────────                                         │
│  Encrypt YAML values in-place using KMS/PGP keys.                │
│  Store encrypted files in Git. Decrypt at apply time.             │
│                                                                   │
│  Option 4: HashiCorp Vault + CSI Driver                          │
│  ──────────────────────────────────────                          │
│  Vault stores secrets. K8s pods mount them directly.              │
│  No secrets in Git at all — only Vault policies.                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: CI Pipeline That Triggers GitOps Deployment

```python
# scripts/update_image_tag.py
# This script is called by CI after building a new image.
# It updates the image tag in the GitOps config repository.

import subprocess
import os
import sys

def update_gitops_repo(new_image_tag: str, app_name: str, env: str):
    """
    Clone the config repo, update the image tag, commit, and push.
    ArgoCD will detect the change and deploy automatically.
    """
    config_repo = os.environ["GITOPS_REPO_URL"]
    token = os.environ["GIT_TOKEN"]
    
    # Clone the config repository
    repo_url_with_auth = config_repo.replace(
        "https://", f"https://bot:{token}@"
    )
    subprocess.run(
        ["git", "clone", "--depth=1", repo_url_with_auth, "config-repo"],
        check=True
    )
    
    # Update the image tag using kustomize
    overlay_path = f"config-repo/apps/{app_name}/overlays/{env}"
    subprocess.run(
        ["kustomize", "edit", "set", "image",
         f"registry.company.com/{app_name}={new_image_tag}"],
        cwd=overlay_path,
        check=True
    )
    
    # Commit and push the change
    subprocess.run(
        ["git", "add", "."],
        cwd="config-repo", check=True
    )
    subprocess.run(
        ["git", "commit", "-m",
         f"chore: update {app_name} to {new_image_tag} in {env}"],
        cwd="config-repo", check=True
    )
    subprocess.run(
        ["git", "push", "origin", "main"],
        cwd="config-repo", check=True
    )
    
    print(f"✅ Updated {app_name} to {new_image_tag} in {env}")
    print("ArgoCD will auto-sync within 3 minutes.")

if __name__ == "__main__":
    # Called from CI: python update_image_tag.py my-app:v1.2.3 my-app production
    image_tag = sys.argv[1]
    app = sys.argv[2]
    environment = sys.argv[3]
    update_gitops_repo(image_tag, app, environment)
```

### Java: Validating Kubernetes Manifests Before Commit

```java
// src/main/java/com/company/gitops/ManifestValidator.java
package com.company.gitops;

import io.fabric8.kubernetes.api.model.apps.Deployment;
import io.fabric8.kubernetes.client.utils.Serialization;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

/**
 * Validates Kubernetes manifests before they're committed to the
 * GitOps config repository. Run this in a pre-commit hook or CI.
 */
public class ManifestValidator {
    
    private final List<String> errors = new ArrayList<>();
    
    public boolean validateDeployment(File manifestFile) throws IOException {
        try (FileInputStream fis = new FileInputStream(manifestFile)) {
            Deployment deployment = Serialization.unmarshal(fis, Deployment.class);
            
            // Rule 1: Must have resource limits
            var containers = deployment.getSpec().getTemplate()
                .getSpec().getContainers();
            for (var container : containers) {
                if (container.getResources() == null ||
                    container.getResources().getLimits() == null) {
                    errors.add(String.format(
                        "%s: container '%s' missing resource limits",
                        manifestFile.getName(), container.getName()
                    ));
                }
            }
            
            // Rule 2: Must have liveness/readiness probes
            for (var container : containers) {
                if (container.getLivenessProbe() == null) {
                    errors.add(String.format(
                        "%s: container '%s' missing liveness probe",
                        manifestFile.getName(), container.getName()
                    ));
                }
                if (container.getReadinessProbe() == null) {
                    errors.add(String.format(
                        "%s: container '%s' missing readiness probe",
                        manifestFile.getName(), container.getName()
                    ));
                }
            }
            
            // Rule 3: Must have at least 2 replicas in production
            if (manifestFile.getPath().contains("production")) {
                int replicas = deployment.getSpec().getReplicas();
                if (replicas < 2) {
                    errors.add(String.format(
                        "%s: production must have >= 2 replicas (has %d)",
                        manifestFile.getName(), replicas
                    ));
                }
            }
        }
        
        return errors.isEmpty();
    }
    
    public List<String> getErrors() { return errors; }
    
    public static void main(String[] args) throws IOException {
        ManifestValidator validator = new ManifestValidator();
        boolean valid = validator.validateDeployment(new File(args[0]));
        
        if (!valid) {
            System.err.println("❌ Validation failed:");
            validator.getErrors().forEach(e -> System.err.println("  - " + e));
            System.exit(1);
        }
        System.out.println("✅ Manifest is valid");
    }
}
```

---

## Infrastructure Examples

### Complete GitOps Setup with ArgoCD

```yaml
# 1. Install ArgoCD in the cluster
# kubectl create namespace argocd
# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Define ArgoCD Application for each microservice
# argocd-apps/my-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-config.git
    targetRevision: main
    path: apps/my-app/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Remove resources deleted from Git
      selfHeal: true     # Revert manual cluster changes
      allowEmpty: false  # Don't delete everything if Git is empty
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  # Health checks and notifications
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # Ignore if HPA changes replicas
```

### Flux CD Alternative

```yaml
# flux-system/gotk-components.yaml (installed via flux bootstrap)

# GitRepository source
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: k8s-config
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/company/k8s-config.git
  ref:
    branch: main
  secretRef:
    name: git-credentials

---
# Kustomization (what to deploy from the repo)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-production
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/my-app/overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: k8s-config
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      namespace: production
  timeout: 3m
```

### GitHub Actions CI + GitOps CD Integration

```yaml
# .github/workflows/ci-and-gitops.yml (in the APP repository)
name: CI → Build → Trigger GitOps Deploy

on:
  push:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.build.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tests
        run: |
          pip install -r requirements.txt
          pytest tests/
      
      - name: Build and push Docker image
        id: build
        run: |
          TAG="registry.company.com/my-app:${{ github.sha }}"
          docker build -t $TAG .
          docker push $TAG
          echo "tag=$TAG" >> $GITHUB_OUTPUT

  update-gitops:
    needs: ci
    runs-on: ubuntu-latest
    steps:
      - name: Checkout config repo
        uses: actions/checkout@v4
        with:
          repository: company/k8s-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}
          path: config
      
      - name: Update image tag
        run: |
          cd config/apps/my-app/overlays/production
          kustomize edit set image \
            registry.company.com/my-app=${{ needs.ci.outputs.image-tag }}
      
      - name: Commit and push
        run: |
          cd config
          git config user.name "CI Bot"
          git config user.email "ci@company.com"
          git add .
          git commit -m "deploy: my-app ${{ github.sha }}"
          git push
      
      # ArgoCD will now automatically detect and deploy the change!
```

---

## Real-World Example

### Weaveworks (Creators of GitOps) Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 GITOPS AT SCALE (Multi-Cluster)                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────────────────────────────────────┐                    │
│  │              Config Repository (Git)              │                    │
│  │                                                  │                    │
│  │  clusters/                                       │                    │
│  │  ├── us-east-1/                                  │                    │
│  │  │   ├── infrastructure/   (cert-manager, nginx) │                    │
│  │  │   └── apps/             (all microservices)   │                    │
│  │  ├── eu-west-1/                                  │                    │
│  │  │   ├── infrastructure/                         │                    │
│  │  │   └── apps/                                   │                    │
│  │  └── ap-south-1/                                 │                    │
│  │      ├── infrastructure/                         │                    │
│  │      └── apps/                                   │                    │
│  └──────────────────────────────────────────────────┘                    │
│                    │            │            │                            │
│                    ▼            ▼            ▼                            │
│         ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │
│         │ ArgoCD       │ │ ArgoCD       │ │ ArgoCD       │             │
│         │ (us-east-1)  │ │ (eu-west-1)  │ │ (ap-south-1) │             │
│         └──────┬───────┘ └──────┬───────┘ └──────┬───────┘             │
│                │                │                │                       │
│                ▼                ▼                ▼                       │
│         ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │
│         │ EKS Cluster  │ │ EKS Cluster  │ │ EKS Cluster  │             │
│         │ US East      │ │ EU West      │ │ AP South     │             │
│         └──────────────┘ └──────────────┘ └──────────────┘             │
│                                                                           │
│  How to deploy globally:                                                  │
│  1. Update image tag in Git → all clusters sync automatically            │
│  2. Use progressive delivery: update us-east first, then others          │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Companies Using GitOps

| Company | GitOps Tool | Scale |
|---------|------------|-------|
| **Intuit (TurboTax)** | ArgoCD | 100+ clusters, 1000+ engineers |
| **Tesla** | Flux CD | Vehicle firmware + cloud services |
| **Alibaba** | ArgoCD | Massive e-commerce infrastructure |
| **Under Armour** | ArgoCD | Migrated 1000+ microservices |
| **Red Hat** | ArgoCD (OpenShift GitOps) | OpenShift customers worldwide |

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Solution |
|---------|--------|----------|
| **Storing secrets in Git (plain text)** | Security breach | Use Sealed Secrets, External Secrets, or SOPS |
| **Single Git repo for app code + config** | Infinite loop (CI triggers itself) | Separate app repo from config repo |
| **No branch protection on config repo** | Anyone can deploy to production | Require PR reviews for production overlays |
| **Auto-sync without health checks** | Broken deployments go unnoticed | Add health checks and sync waves |
| **Ignoring ArgoCD out-of-sync alerts** | Drift accumulates over time | Treat out-of-sync as a P1 alert |
| **Not handling HPA vs. replicas conflict** | ArgoCD keeps reverting HPA changes | Use `ignoreDifferences` for HPA-managed fields |
| **No rollback strategy** | Panic when bad deploy happens | Rollback = `git revert` → ArgoCD auto-syncs |
| **Too many apps in one repo** | Slow sync, hard to navigate | Organize by team/domain, use ApplicationSets |

---

## When to Use / When NOT to Use

### Use GitOps When:
- ✅ Deploying to Kubernetes (ArgoCD/Flux are built for K8s)
- ✅ You need audit trail for all deployments (compliance)
- ✅ Multiple environments (dev/staging/prod) with config differences
- ✅ Multiple clusters (GitOps shines at multi-cluster management)
- ✅ You want self-healing (automatic drift correction)
- ✅ Teams need to deploy independently without cluster access

### When GitOps May NOT Be the Best Fit:
- ❌ Non-Kubernetes deployments (VMs, Lambda, bare-metal) — GitOps tools assume K8s
- ❌ Very small projects (1 service, 1 environment) — overhead not worth it
- ❌ Stateful deployments requiring careful ordering (databases, migrations) — needs extra sync wave configuration
- ❌ Teams with no Kubernetes experience — learn K8s first, then GitOps

---

## Key Takeaways

- 🔑 **GitOps = Git is the single source of truth** for infrastructure and application state
- 🔑 **Pull-based model**: agents inside the cluster pull changes from Git (more secure than push)
- 🔑 **Continuous reconciliation**: the system self-heals by constantly comparing desired vs. actual state
- 🔑 **Rollback = `git revert`**: no special tooling needed, just undo the commit
- 🔑 **Separate repos**: app code in one repo, deployment config in another (avoids CI loops)
- 🔑 **Secrets need special handling**: never store plain-text secrets in Git (use Sealed Secrets, External Secrets, or Vault)
- 🔑 GitOps gives you a **complete audit trail** — who deployed what, when, and why (Git history)

---

## What's Next?

Now that you understand how GitOps manages deployments through Git, the next chapter covers **Artifact Management & Container Image Pipelines** — how to build, store, version, and secure the Docker images and packages that GitOps deploys.

**Next: [Artifact Management & Container Image Pipelines](./05-artifact-management.md)**
