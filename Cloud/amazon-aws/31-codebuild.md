# Chapter 31: CodeBuild

---

## Table of Contents

- [Overview](#overview)
- [Part 1: CodeBuild Fundamentals](#part-1-codebuild-fundamentals)
- [Part 2: Creating a Build Project (Full Portal Walkthrough)](#part-2-creating-a-build-project-full-portal-walkthrough)
- [Part 3: buildspec.yml Deep Dive](#part-3-buildspecyml-deep-dive)
- [Part 4: Build Environments & Caching](#part-4-build-environments--caching)
- [Part 5: Artifacts & Reports](#part-5-artifacts--reports)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Build Automation? Why Do We Need It?

**Building software** means taking your source code and turning it into something that can run — compiling Java to a JAR, bundling a React app into static files, creating a Docker image, or running tests to make sure nothing is broken.

Doing this manually ("let me run `npm build` on my laptop and upload the files") is slow, error-prone, and doesn't scale. **Build automation** means a system does this automatically every time you push code.

**CodeBuild is like a robot factory worker** that takes your raw code, follows your instructions (buildspec.yml), and produces a finished product (artifact) every time. You don't need to set up or maintain any build servers — AWS provides the compute on demand.

**Simple example:** You push code to GitHub → CodeBuild automatically runs your tests → if tests pass, it builds a Docker image → pushes it to ECR → ready to deploy.

AWS CodeBuild is a fully managed build service that compiles source code, runs tests, and produces deployable artifacts. No build servers to manage — it scales automatically.

```
What you'll learn:
├── CodeBuild Fundamentals
├── Creating a build project (every field explained)
├── buildspec.yml (phases, commands, artifacts)
├── Build environments (images, compute)
├── Caching (local, S3)
├── Artifacts & test reports
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: CodeBuild Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW CODEBUILD WORKS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────┐    ┌───────────────────────────────────┐    ┌────────┐│
│ │ Source    │───►│         CodeBuild                  │───►│Artifact││
│ │CodeCommit│    │ ┌─────────────────────────────┐   │    │  S3    ││
│ │GitHub    │    │ │  Docker Container           │   │    │  ECR   ││
│ │BitBucket │    │ │  ┌─────────┐ ┌──────────┐  │   │    └────────┘│
│ │S3        │    │ │  │buildspec│ │Source Code│  │   │             │
│ └──────────┘    │ │  │.yml     │ │           │  │   │             │
│                 │ │  └─────────┘ └──────────┘  │   │             │
│                 │ │  install → pre_build →      │   │             │
│                 │ │  build → post_build         │   │             │
│                 │ └─────────────────────────────┘   │             │
│                 └───────────────────────────────────┘             │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Build project: Configuration (source, env, buildspec)        │
│ ├── Build run: Single execution of a build project               │
│ ├── buildspec.yml: Build instructions file (in source root)     │
│ ├── Build environment: Docker container where build runs         │
│ ├── Artifacts: Output files from the build                       │
│ └── Fully managed: No servers, pay per build-minute             │
│                                                                       │
│ Pricing (us-east-1):                                                │
│ ├── general1.small: $0.005/min (3 GB, 2 vCPU)                  │
│ ├── general1.medium: $0.01/min (7 GB, 4 vCPU)                  │
│ ├── general1.large: $0.02/min (15 GB, 8 vCPU)                  │
│ └── Free tier: 100 build-minutes/month (general1.small)         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Build Project (Full Portal Walkthrough)

```
Console → CodeBuild → Build projects → Create build project

┌─────────────────────────────────────────────────────────────────┐
│           CREATE BUILD PROJECT                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Project configuration ──                                    │
│ Project name: [my-app-build]                                  │
│ Description: [Build and test my application]                  │
│ Build badge: ☑ Enable                                         │
│ → Generates a badge URL showing build status                 │
│ Concurrent build limit: [1] (empty = unlimited)               │
│                                                                   │
│ ── Source ──                                                    │
│ Source provider: [GitHub ▼]                                   │
│   ├── AWS CodeCommit                                           │
│   ├── Amazon S3                                                │
│   ├── GitHub (via OAuth or CodeStar connection)               │
│   ├── GitHub Enterprise Server                                │
│   ├── Bitbucket                                                │
│   └── No source (use buildspec commands only)                 │
│                                                                   │
│ Connection: [Connect to GitHub ▼]                              │
│ → Uses CodeStar Connections for GitHub/Bitbucket              │
│                                                                   │
│ Repository: [my-org/my-application ▼]                         │
│                                                                   │
│ Source version: [main]                                         │
│ → Branch name, tag, or commit ID                              │
│                                                                   │
│ Git clone depth: [1 ▼]                                        │
│ → 1 = shallow clone (faster). Full = all history.            │
│                                                                   │
│ Git submodules: ☐ Include                                     │
│                                                                   │
│ ── Primary source webhook events ──                            │
│ ☑ Rebuild every time a code change is pushed                  │
│   Event type:                                                  │
│   ☑ PUSH                                                       │
│   ☑ PULL_REQUEST_CREATED                                      │
│   ☑ PULL_REQUEST_UPDATED                                      │
│   ☐ PULL_REQUEST_MERGED                                       │
│   ☐ PULL_REQUEST_REOPENED                                     │
│                                                                   │
│   Filter group (optional):                                    │
│   HEAD_REF: [^refs/heads/main$]                               │
│   → Only build on pushes to main branch                      │
│                                                                   │
│ ── Environment ──                                               │
│ Environment image:                                             │
│ ● Managed image  ○ Custom image                               │
│                                                                   │
│ Compute: ● EC2  ○ Lambda                                      │
│ → Lambda: Faster startup, limited features                   │
│ → EC2: Full control, Docker support                          │
│                                                                   │
│ Operating system: [Amazon Linux ▼]                            │
│   ├── Amazon Linux (⚡ recommended)                           │
│   ├── Ubuntu                                                   │
│   └── Windows Server                                           │
│                                                                   │
│ Runtime: [Standard ▼]                                         │
│ Image: [aws/codebuild/amazonlinux2-x86_64-standard:5.0 ▼]   │
│ → Pre-installed: Docker, Python, Node.js, Java, Go, .NET    │
│                                                                   │
│ Image version: [Always use the latest image for this runtime]│
│                                                                   │
│ Environment type:                                              │
│ ● Linux EC2  ○ Linux EC2 GPU  ○ ARM EC2                      │
│                                                                   │
│ Compute:                                                       │
│   ● 3 GB memory, 2 vCPUs (small)                             │
│   ○ 7 GB memory, 4 vCPUs (medium)                            │
│   ○ 15 GB memory, 8 vCPUs (large)                            │
│   ○ 145 GB memory, 72 vCPUs (2xlarge)                        │
│                                                                   │
│ ☑ Privileged                                                   │
│ → ⚠️ Required if building Docker images                      │
│ → Enables docker daemon in build container                   │
│                                                                   │
│ Service role:                                                  │
│ ● New service role  ○ Existing service role                   │
│ Role name: [codebuild-my-app-service-role]                    │
│ → Needs: S3, CloudWatch Logs, ECR, Secrets Manager, etc.    │
│                                                                   │
│ ── Additional configuration ──                                 │
│                                                                   │
│ Timeout: [60] minutes (default, max 480)                      │
│ Queued timeout: [480] minutes                                 │
│                                                                   │
│ VPC: [vpc-main ▼] (optional)                                  │
│ Subnets: [private-a, private-b ▼]                             │
│ Security groups: [sg-codebuild ▼]                             │
│ → VPC access needed for: private resources (RDS, ES, etc.)  │
│ → ⚠️ Needs NAT Gateway for internet access in VPC           │
│                                                                   │
│ Environment variables:                                         │
│ Name: [DOCKER_REGISTRY]  Value: [123456.dkr.ecr...]          │
│ Type: ● Plaintext ○ Secrets Manager ○ Parameter Store        │
│ → ⚡ Use Secrets Manager for passwords/tokens                │
│                                                                   │
│ ── Buildspec ──                                                 │
│ ● Use a buildspec file                                         │
│   Buildspec name: [buildspec.yml] (in source root)            │
│ ○ Insert build commands                                        │
│   → Inline buildspec (not recommended for production)        │
│                                                                   │
│ ── Batch configuration ──                                      │
│ ☐ Define batch configuration                                  │
│ → Run multiple builds in parallel (matrix builds)            │
│                                                                   │
│ ── Artifacts ──                                                 │
│ Type: [Amazon S3 ▼]                                           │
│   ├── No artifacts                                             │
│   └── Amazon S3                                                │
│                                                                   │
│ Bucket name: [my-build-artifacts ▼]                           │
│ Path prefix: [builds/]                                        │
│ Name: [my-app-build.zip]                                      │
│ Artifacts packaging: ● Zip ○ None                             │
│ ☑ Enable semantic versioning                                  │
│ → Uses buildspec version for naming                          │
│                                                                   │
│ Encryption key: ● Default AWS managed key                     │
│                                                                   │
│ Additional artifacts: Add up to 12 secondary artifacts       │
│                                                                   │
│ ── Logs ──                                                      │
│ CloudWatch logs:                                               │
│ ☑ Enable  Group: [/aws/codebuild/my-app]  Stream: [build]   │
│                                                                   │
│ S3 logs:                                                       │
│ ☐ Enable (optional, for long-term log storage)               │
│                                                                   │
│                    [Create build project]                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: buildspec.yml Deep Dive

```yaml
# buildspec.yml — must be in source root
version: 0.2

env:
  variables:              # Plain text env vars
    NODE_ENV: "production"
    APP_NAME: "my-app"
  parameter-store:        # From SSM Parameter Store
    DB_HOST: "/prod/db/host"
  secrets-manager:        # From Secrets Manager
    DB_PASSWORD: "prod/db:password"
  exported-variables:     # Share with CodePipeline
    - BUILD_VERSION

phases:
  install:
    runtime-versions:
      nodejs: 18
      python: 3.11
    commands:
      - echo "Installing dependencies..."
      - npm ci
      # → Use 'npm ci' not 'npm install' for CI builds (deterministic)

  pre_build:
    commands:
      - echo "Running pre-build steps..."
      - echo "Logging into ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION |
        docker login --username AWS --password-stdin $DOCKER_REGISTRY
      - export BUILD_VERSION=$(date +%Y%m%d%H%M%S)-$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | head -c 8)

  build:
    commands:
      - echo "Building application..."
      - npm run build
      - npm test
      - echo "Building Docker image..."
      - docker build -t $DOCKER_REGISTRY/$APP_NAME:$BUILD_VERSION .
      - docker tag $DOCKER_REGISTRY/$APP_NAME:$BUILD_VERSION $DOCKER_REGISTRY/$APP_NAME:latest

  post_build:
    commands:
      - echo "Pushing Docker image..."
      - docker push $DOCKER_REGISTRY/$APP_NAME:$BUILD_VERSION
      - docker push $DOCKER_REGISTRY/$APP_NAME:latest
      - echo "Writing image definitions..."
      - printf '[{"name":"app","imageUri":"%s"}]' $DOCKER_REGISTRY/$APP_NAME:$BUILD_VERSION > imagedefinitions.json

reports:
  test-reports:
    files:
      - "junit-report.xml"
    base-directory: "test-results"
    file-format: "JUNITXML"    # or CUCUMBERJSON, VISUALSTUDIOTRX

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yml
    - taskdef.json
  discard-paths: yes
  secondary-artifacts:
    coverage:
      files:
        - "**/*"
      base-directory: "coverage"

cache:
  paths:
    - "node_modules/**/*"        # Cache npm packages
    - "/root/.m2/**/*"           # Cache Maven
    - "/root/.cache/pip/**/*"    # Cache pip
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           BUILDSPEC PHASES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ install    → Install dependencies, runtime versions               │
│     ↓                                                                │
│ pre_build  → Login to registries, setup, pre-checks              │
│     ↓                                                                │
│ build      → Compile, test, build artifacts/images                │
│     ↓                                                                │
│ post_build → Push images, generate deployment files               │
│                                                                       │
│ Each phase has:                                                      │
│ ├── commands: List of shell commands                             │
│ ├── on-failure: ABORT (default) or CONTINUE                     │
│ ├── run-as: Linux user to run as                                 │
│ └── finally: Commands that always run (cleanup)                 │
│                                                                       │
│ Built-in environment variables:                                     │
│ ├── CODEBUILD_BUILD_ID: Unique build ID                          │
│ ├── CODEBUILD_BUILD_NUMBER: Sequential build number             │
│ ├── CODEBUILD_RESOLVED_SOURCE_VERSION: Git commit hash          │
│ ├── CODEBUILD_SOURCE_REPO_URL: Source repository URL            │
│ ├── CODEBUILD_SOURCE_VERSION: Branch/tag/PR reference           │
│ └── CODEBUILD_WEBHOOK_TRIGGER: Webhook event type               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Build Environments & Caching

```
┌─────────────────────────────────────────────────────────────────────┐
│           BUILD ENVIRONMENTS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Managed images:                                                      │
│ ├── Amazon Linux 2: aws/codebuild/amazonlinux2-x86_64-standard │
│ ├── Ubuntu: aws/codebuild/standard (7.0, 6.0)                  │
│ ├── Windows: aws/codebuild/windows-base                         │
│ └── Pre-installed: Docker, Python, Node, Java, Go, .NET, Ruby  │
│                                                                       │
│ Custom images:                                                       │
│ ├── ECR (your account): Install exactly what you need           │
│ ├── ECR Public: Public images                                    │
│ ├── Docker Hub: Any Docker Hub image                             │
│ └── ⚡ Custom image for: Specific tool versions, large deps     │
│                                                                       │
│ Caching strategies:                                                  │
│ ├── Local cache (in-build container):                            │
│ │   ├── Source cache: Git repo cached between builds            │
│ │   ├── Docker layer cache: Docker build layers cached          │
│ │   └── Custom cache: buildspec cache paths                     │
│ └── S3 cache:                                                    │
│     ├── node_modules, .m2, pip cache → stored in S3            │
│     ├── Shared across builds                                     │
│     └── ⚡ Best for npm/Maven/pip dependency caching            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Artifacts & Reports

```
┌─────────────────────────────────────────────────────────────────────┐
│           ARTIFACTS & TEST REPORTS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Artifacts:                                                           │
│ ├── Primary artifact: Main build output (ZIP to S3)             │
│ ├── Secondary artifacts: Up to 12 additional outputs            │
│ ├── No artifacts: Build runs but produces no output             │
│ └── Artifacts used by CodePipeline for next stage               │
│                                                                       │
│ Test Reports:                                                        │
│ Console → CodeBuild → Report groups                               │
│                                                                       │
│ Supported formats:                                                   │
│ ├── JUnit XML (Java, JavaScript, Python)                        │
│ ├── Cucumber JSON                                                 │
│ ├── Visual Studio TRX (.NET)                                    │
│ ├── TestNG XML                                                    │
│ └── NUnit XML                                                    │
│                                                                       │
│ Code coverage reports:                                               │
│ ├── Clover XML, Cobertura XML, JaCoCo XML                      │
│ ├── SimpleCov JSON (Ruby)                                        │
│ └── View line/branch coverage in console                        │
│                                                                       │
│ ⚡ Reports show pass/fail trends across builds                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & CLI Examples

```hcl
resource "aws_codebuild_project" "app" {
  name          = "my-app-build"
  description   = "Build and test my application"
  service_role  = aws_iam_role.codebuild.arn
  build_timeout = 60

  source {
    type            = "GITHUB"
    location        = "https://github.com/my-org/my-app.git"
    git_clone_depth = 1
    buildspec       = "buildspec.yml"
  }

  environment {
    compute_type                = "BUILD_GENERAL1_MEDIUM"
    image                       = "aws/codebuild/amazonlinux2-x86_64-standard:5.0"
    type                        = "LINUX_CONTAINER"
    privileged_mode             = true  # For Docker builds
    image_pull_credentials_type = "CODEBUILD"

    environment_variable {
      name  = "DOCKER_REGISTRY"
      value = "${data.aws_caller_identity.current.account_id}.dkr.ecr.us-east-1.amazonaws.com"
    }
    environment_variable {
      name  = "DB_PASSWORD"
      value = "prod/db:password"
      type  = "SECRETS_MANAGER"
    }
  }

  artifacts {
    type      = "S3"
    location  = aws_s3_bucket.artifacts.id
    packaging = "ZIP"
  }

  cache {
    type     = "S3"
    location = "${aws_s3_bucket.cache.id}/codebuild-cache"
  }

  logs_config {
    cloudwatch_logs {
      group_name  = "/aws/codebuild/my-app"
      stream_name = "build"
    }
  }

  vpc_config {
    vpc_id             = aws_vpc.main.id
    subnets            = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.codebuild.id]
  }

  tags = { Environment = "prod" }
}
```

```bash
# Start a build
aws codebuild start-build \
  --project-name my-app-build \
  --source-version main

# Start build with environment variable override
aws codebuild start-build \
  --project-name my-app-build \
  --environment-variables-override name=NODE_ENV,value=staging,type=PLAINTEXT

# Get build status
aws codebuild batch-get-builds --ids my-app-build:build-id-123

# List builds for project
aws codebuild list-builds-for-project --project-name my-app-build

# View build logs
aws logs get-log-events \
  --log-group-name /aws/codebuild/my-app \
  --log-stream-name build/latest
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Docker Build & Push to ECR                               │
│ buildspec: pre_build → ECR login                                   │
│            build → docker build + tag                               │
│            post_build → docker push + imagedefinitions.json        │
│ ⚡ Enable privileged mode for Docker-in-Docker                     │
│                                                                       │
│ Pattern 2: Multi-Stage Pipeline                                     │
│ ├── Build stage: Compile + unit tests                            │
│ ├── Test stage: Integration tests (separate CodeBuild project)  │
│ └── Deploy stage: CodeDeploy or CloudFormation                   │
│                                                                       │
│ Pattern 3: Matrix/Batch Builds                                      │
│ buildspec-batch:                                                     │
│   build-matrix:                                                      │
│     static:                                                          │
│       env:                                                           │
│         variables:                                                   │
│           NODE_VERSION: ["16", "18", "20"]                         │
│ → Tests against multiple Node.js versions in parallel            │
│                                                                       │
│ Pattern 4: Secrets Management                                       │
│ ├── DB passwords: Secrets Manager (type: SECRETS_MANAGER)       │
│ ├── Config values: Parameter Store (type: PARAMETER_STORE)      │
│ ├── Docker Hub: Secrets Manager for registry credentials        │
│ └── ⚠️ Never hardcode secrets in buildspec.yml                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
CodeBuild Quick Reference:
├── What: Fully managed build service (compile, test, package)
├── Spec file: buildspec.yml (in source root)
├── Phases: install → pre_build → build → post_build
├── Sources: CodeCommit, GitHub, Bitbucket, S3
├── Environments: Managed or custom Docker images
├── Caching: S3 (shared) or local (per-build)
├── Artifacts: S3 (ZIP or individual files)
├── Reports: JUnit, Cucumber, coverage reports
├── Pricing: Per build-minute ($0.005-$0.02/min)
├── ⚡ Enable privileged mode for Docker builds
└── ⚡ Use Secrets Manager for credentials
```

---

## What's Next?

In **Chapter 32: CodeDeploy**, we'll cover automated deployment to EC2, Lambda, and ECS — deployment groups, deployment configurations, appspec.yml, and rollback strategies.
