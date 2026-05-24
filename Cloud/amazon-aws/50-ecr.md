# Chapter 50: Amazon ECR (Elastic Container Registry)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: ECR Fundamentals](#part-1-ecr-fundamentals)
- [Part 2: Creating a Repository (Full Portal Walkthrough)](#part-2-creating-a-repository-full-portal-walkthrough)
- [Part 3: Push & Pull Images](#part-3-push--pull-images)
- [Part 4: Image Scanning & Lifecycle Policies](#part-4-image-scanning--lifecycle-policies)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is a Container Registry? Why Do We Need ECR?

When you build a mobile app, you publish it to the **App Store** so devices can download and run it. Similarly, when you build a container (a packaged application with all its dependencies), you need a **registry** to store it so your servers can download and run it.

**ECR (Elastic Container Registry) is the App Store for your Docker containers.** You build a Docker image on your laptop, push it to ECR, and then ECS/EKS/Lambda pulls it from ECR to run your application.

**Why ECR instead of Docker Hub?**
- 🔐 **Private by default**: Your images aren't visible to the public
- ⚡ **Fast pulls**: Same AWS network, no internet download needed
- 🔍 **Image scanning**: Automatically scans for security vulnerabilities
- 🔗 **IAM integration**: Control who can push/pull images using AWS permissions
- 💰 **No rate limits**: Docker Hub free tier limits you to 100 pulls/6 hours

Amazon ECR is a fully managed container image registry that stores, manages, and deploys Docker and OCI container images. It integrates natively with ECS, EKS, and Lambda.

```
What you'll learn:
├── ECR fundamentals (repositories, images, tags)
├── Creating a repository (portal walkthrough)
├── Push & pull images (Docker CLI)
├── Image scanning (vulnerabilities)
├── Lifecycle policies (auto-cleanup old images)
├── Cross-account & cross-region replication
└── Terraform & CLI examples
```

---

## Part 1: ECR Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW ECR WORKS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Build → Push → Store → Pull → Deploy                              │
│                                                                       │
│ ┌──────────┐  push  ┌──────────┐  pull  ┌──────────────────┐    │
│ │ Docker   │───────▶│  ECR     │───────▶│ ECS / EKS /      │    │
│ │ Build    │        │ Registry │        │ Lambda / Fargate  │    │
│ └──────────┘        │          │        └──────────────────┘    │
│                     │ ┌──────┐ │                                   │
│                     │ │Repos │ │                                   │
│                     │ │├─ app│ │                                   │
│                     │ │├─ api│ │                                   │
│                     │ │└─ web│ │                                   │
│                     │ └──────┘ │                                   │
│                     └──────────┘                                   │
│                                                                       │
│ Registry types:                                                      │
│ ├── Private registry (default): Your account's private images   │
│ │   → 123456789012.dkr.ecr.us-east-1.amazonaws.com             │
│ └── ECR Public (public.ecr.aws): Public images                  │
│     → public.ecr.aws/myalias/myimage                           │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Registry: Account-level container (one per account/region) │
│ ├── Repository: Collection of related images (like Docker Hub)  │
│ ├── Image: Container image (Docker/OCI format)                  │
│ ├── Tag: Image identifier (e.g., latest, v1.2.3, sha-abc123)  │
│ └── Image manifest: Image metadata and layer information        │
│                                                                       │
│ Pricing:                                                             │
│ ├── Storage: $0.10/GB/month                                     │
│ ├── Data transfer out: Standard rates                           │
│ ├── Data transfer to ECS/EKS in same region: FREE              │
│ └── Public ECR: 50 GB free storage, 500 GB free transfer/month │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Repository (Full Portal Walkthrough)

```
Console → ECR → Repositories → Create repository

┌─────────────────────────────────────────────────────────────────┐
│           CREATE REPOSITORY                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Visibility settings:                                           │
│ ● Private                                                      │
│   → Only accessible within your account (+ cross-account)   │
│ ○ Public                                                       │
│   → Publicly accessible on public.ecr.aws                   │
│                                                                   │
│ Repository name: [my-app/web-server]                          │
│ → ⚡ Use namespace: team/service or project/component        │
│ → Example: platform/api, platform/worker, frontend/web     │
│                                                                   │
│ Tag immutability:                                              │
│ ○ Mutable (default)                                           │
│   → Tags can be overwritten (e.g., push new "latest")       │
│ ● Immutable (⚡ recommended for production)                  │
│   → Tags cannot be overwritten once pushed                   │
│   → Prevents accidental overwrites                           │
│   → ⚡ Forces unique tags (git SHA, version numbers)        │
│                                                                   │
│ Image scan settings:                                           │
│ ● Scan on push (⚡ recommended)                              │
│   → Automatically scan for vulnerabilities on push          │
│ ○ Manual scan                                                 │
│   → Scan manually when needed                               │
│ ○ Enhanced scanning (Amazon Inspector)                       │
│   → Continuous scanning + OS and language package vulns     │
│   → ⚡ Best for production (additional cost)                │
│                                                                   │
│ Encryption:                                                    │
│ ● AES-256 (default, free)                                    │
│   → AWS managed encryption                                   │
│ ○ AWS KMS                                                     │
│   → Customer managed key                                     │
│   → KMS key: [alias/ecr-key ▼]                             │
│   → ⚡ Required for compliance requirements                 │
│                                                                   │
│                    [Create repository]                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Push & Pull Images

```
┌─────────────────────────────────────────────────────────────────────┐
│           PUSH & PULL WORKFLOW                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Authenticate Docker to ECR                                 │
│ aws ecr get-login-password --region us-east-1 | \                 │
│   docker login --username AWS --password-stdin \                   │
│   123456789012.dkr.ecr.us-east-1.amazonaws.com                    │
│ → Auth token valid for 12 hours                                   │
│                                                                       │
│ Step 2: Build image                                                 │
│ docker build -t my-app/web-server .                                │
│                                                                       │
│ Step 3: Tag image for ECR                                          │
│ docker tag my-app/web-server:latest \                              │
│   123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app/web-server:v1.0.0│
│ → ⚡ Use meaningful tags: v1.0.0, git-sha-abc123                 │
│ → ⚠️ Avoid relying solely on "latest"                            │
│                                                                       │
│ Step 4: Push image                                                  │
│ docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app/web-server:v1.0.0│
│                                                                       │
│ Step 5: Pull image                                                  │
│ docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app/web-server:v1.0.0│
│                                                                       │
│ Console → ECR → Repository → Images:                               │
│ ┌───────────────────────────────────────────────────────┐          │
│ │ Tag      │ Pushed      │ Size   │ Scan Status        │          │
│ ├──────────┼─────────────┼────────┼────────────────────┤          │
│ │ v1.0.0   │ 2 hours ago │ 145 MB │ ✅ 0 findings     │          │
│ │ v0.9.0   │ 3 days ago  │ 142 MB │ ⚠️ 3 HIGH         │          │
│ │ v0.8.0   │ 1 week ago  │ 140 MB │ ✅ 0 findings     │          │
│ └───────────────────────────────────────────────────────┘          │
│                                                                       │
│ Pull-through cache (cache public registries):                       │
│ Console → ECR → Pull through cache → Create rule                  │
│ Upstream registry: Docker Hub / ECR Public / Quay / GitHub        │
│ ECR repository prefix: [docker-hub/]                               │
│ → Caches public images locally in your ECR                        │
│ → Reduces pull rate limits and improves speed                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Image Scanning & Lifecycle Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│           IMAGE SCANNING                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Basic scanning (free):                                               │
│ ├── CVE database (Common Vulnerabilities and Exposures)         │
│ ├── Scans OS packages only                                       │
│ ├── On-push or manual trigger                                    │
│ └── Findings: CRITICAL, HIGH, MEDIUM, LOW, INFORMATIONAL       │
│                                                                       │
│ Enhanced scanning (Amazon Inspector, paid):                        │
│ ├── Continuous scanning (not just on push)                      │
│ ├── OS packages + programming language packages                │
│ ├── Scans: Python, Node.js, Java, .NET, Go, Ruby, Rust        │
│ ├── Findings sent to Security Hub + EventBridge                │
│ └── ⚡ Recommended for production workloads                    │
│                                                                       │
│ Console → ECR → Image → Scan findings:                             │
│ ┌────────────────────────────────────────────────────┐             │
│ │ Severity  │ CVE           │ Package    │ Fixed in  │             │
│ ├───────────┼───────────────┼────────────┼───────────┤             │
│ │ CRITICAL  │ CVE-2024-1234 │ openssl    │ 3.0.13   │             │
│ │ HIGH      │ CVE-2024-5678 │ curl       │ 8.6.0    │             │
│ │ MEDIUM    │ CVE-2024-9012 │ zlib       │ 1.3.1    │             │
│ └────────────────────────────────────────────────────┘             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           LIFECYCLE POLICIES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → ECR → Repository → Lifecycle policy → Create rule       │
│                                                                       │
│ Rule priority: [1]                                                  │
│ Rule description: [Remove untagged images after 1 day]            │
│                                                                       │
│ Image status: ● Untagged ○ Tagged ○ Any                          │
│ Match criteria: [Since image pushed]                               │
│ Count: [1] days                                                     │
│ Action: [Expire]                                                    │
│                                                                       │
│ Common lifecycle rules:                                              │
│ Rule 1: Remove untagged images after 1 day                        │
│ Rule 2: Keep only last 10 tagged images                            │
│ Rule 3: Remove images older than 90 days (except latest 5)       │
│                                                                       │
│ Example policy JSON:                                                 │
│ {                                                                     │
│   "rules": [                                                        │
│     {                                                                 │
│       "rulePriority": 1,                                            │
│       "description": "Remove untagged after 1 day",               │
│       "selection": {                                                 │
│         "tagStatus": "untagged",                                    │
│         "countType": "sinceImagePushed",                           │
│         "countUnit": "days",                                        │
│         "countNumber": 1                                             │
│       },                                                              │
│       "action": { "type": "expire" }                                │
│     },                                                                │
│     {                                                                 │
│       "rulePriority": 2,                                            │
│       "description": "Keep last 10 images",                       │
│       "selection": {                                                 │
│         "tagStatus": "tagged",                                      │
│         "tagPrefixList": ["v"],                                     │
│         "countType": "imageCountMoreThan",                         │
│         "countNumber": 10                                            │
│       },                                                              │
│       "action": { "type": "expire" }                                │
│     }                                                                 │
│   ]                                                                   │
│ }                                                                     │
│                                                                       │
│ Replication:                                                          │
│ Console → ECR → Private registry → Replication                    │
│ ├── Cross-region: Replicate images to other regions             │
│ ├── Cross-account: Replicate to other AWS accounts              │
│ └── Filter: Prefix-based repository filtering                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
resource "aws_ecr_repository" "app" {
  name                 = "my-app/web-server"
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }
}

resource "aws_ecr_lifecycle_policy" "app" {
  repository = aws_ecr_repository.app.name
  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Remove untagged after 1 day"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 1
        }
        action = { type = "expire" }
      },
      {
        rulePriority = 2
        description  = "Keep last 20 images"
        selection = {
          tagStatus      = "tagged"
          tagPrefixList  = ["v"]
          countType      = "imageCountMoreThan"
          countNumber    = 20
        }
        action = { type = "expire" }
      }
    ]
  })
}

# Cross-account access policy
resource "aws_ecr_repository_policy" "cross_account" {
  repository = aws_ecr_repository.app.name
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowCrossAccountPull"
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::987654321098:root" }
      Action = [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ]
    }]
  })
}
```

```bash
# Authenticate
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name my-app/web-server \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true

# List images
aws ecr list-images --repository-name my-app/web-server

# Describe image scan findings
aws ecr describe-image-scan-findings \
  --repository-name my-app/web-server \
  --image-id imageTag=v1.0.0

# Delete image
aws ecr batch-delete-image \
  --repository-name my-app/web-server \
  --image-ids imageTag=v0.1.0
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: CI/CD Pipeline                                            │
│ GitHub → CodeBuild → ECR → ECS/EKS                                │
│ ├── Build image with git SHA tag                                 │
│ ├── Scan on push (block deployment if CRITICAL vulns)          │
│ ├── Immutable tags prevent overwrites                            │
│ └── Lifecycle policy cleans old images                           │
│                                                                       │
│ Pattern 2: Multi-Region Deployment                                  │
│ ECR (us-east-1) → Replicate → ECR (eu-west-1)                   │
│                              → ECR (ap-southeast-1)              │
│ ├── Push once, deploy globally                                   │
│ ├── Faster pulls from local region                               │
│ └── Disaster recovery ready                                      │
│                                                                       │
│ Pattern 3: Shared Services (Multi-Account)                          │
│ Shared Account (ECR) → Dev Account (ECS)                          │
│                       → Staging Account (ECS)                     │
│                       → Prod Account (ECS)                        │
│ ├── Central image repository                                     │
│ ├── Cross-account policies for pull access                      │
│ └── Consistent images across environments                       │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Enable immutable tags for production repos                    │
│ 2. Enable scan on push (enhanced scanning for prod)             │
│ 3. Set up lifecycle policies to control costs                    │
│ 4. Use meaningful tags (git SHA, semver), not just "latest"    │
│ 5. Enable cross-region replication for DR                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting: Common ECR Issues

### "docker push/pull: AccessDenied or authorization failed"

```
1. ☐ Did you log in to ECR?
   aws ecr get-login-password --region ap-south-1 | \
   docker login --username AWS --password-stdin \
   123456789012.dkr.ecr.ap-south-1.amazonaws.com

2. ☐ Token expired? (ECR tokens expire after 12 hours)
   Re-run the login command above

3. ☐ Does your IAM user/role have ECR permissions?
   Minimum: ecr:GetAuthorizationToken, ecr:BatchGetImage,
   ecr:GetDownloadUrlForLayer (pull) + ecr:PutImage (push)
```

### "CannotPullContainerError in ECS/EKS"

```
1. ☐ Does the ECS task execution role have ECR access?
   Attach: AmazonECSTaskExecutionRolePolicy

2. ☐ Is the image URI correct? (check region and tag)
   123456789012.dkr.ecr.REGION.amazonaws.com/REPO:TAG

3. ☐ For Fargate in private subnet: Is there a VPC endpoint for ECR?
   Need both: com.amazonaws.REGION.ecr.dkr and ecr.api
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| No lifecycle policy | Images accumulate, storage costs grow | Set policy: keep last 10 images |
| Pushing with `latest` tag only | Can't rollback to specific version | Always tag with git SHA or version number |
| Immutable tags + same tag push | Push fails | Use unique tags or disable immutability |

---

## Quick Reference

```
ECR Quick Reference:
├── Private registry: 123456.dkr.ecr.region.amazonaws.com/repo
├── Public registry: public.ecr.aws/alias/image
├── Storage: $0.10/GB/month
├── Tag immutability: ⚡ Enable for production (no overwrites)
├── Scanning: Basic (free, OS pkgs) or Enhanced (Inspector, paid)
├── Lifecycle: Auto-expire untagged + keep last N images
├── Replication: Cross-region and cross-account
├── Pull-through cache: Cache Docker Hub/public registries
├── Auth: aws ecr get-login-password (12-hour token)
├── Encryption: AES-256 (default) or KMS
├── ⚡ Immutable tags + scan on push + lifecycle = production ready
└── ⚡ Tag with git SHA or semver, not just "latest"
```

---

## What's Next?

In **Chapter 51: AWS Fargate Deep Dive**, we'll cover serverless containers, Fargate with ECS and EKS, task definitions, networking, and advanced compute configurations.
