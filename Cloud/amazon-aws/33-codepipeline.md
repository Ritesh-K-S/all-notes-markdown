# Chapter 33: CodePipeline

---

## Table of Contents

- [Overview](#overview)
- [Part 1: CodePipeline Fundamentals](#part-1-codepipeline-fundamentals)
- [Part 2: Creating a Pipeline (Full Portal Walkthrough)](#part-2-creating-a-pipeline-full-portal-walkthrough)
- [Part 3: Stages & Actions](#part-3-stages--actions)
- [Part 4: Manual Approvals & Notifications](#part-4-manual-approvals--notifications)
- [Part 5: Cross-Account & Cross-Region](#part-5-cross-account--cross-region)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is a CI/CD Pipeline? Why Do We Need It?

**CI/CD** stands for Continuous Integration / Continuous Delivery — it's the practice of automatically building, testing, and deploying your code every time you make a change.

Without CI/CD, the workflow looks like: write code → manually test → manually build → manually deploy → pray it works. With CI/CD, you push code and everything happens automatically.

**CodePipeline is the orchestrator** that connects all the pieces together:

```
Your workflow without CodePipeline:
  Developer → push code → manually trigger build → manually deploy → 😰

Your workflow WITH CodePipeline:
  Developer → push code → CodePipeline automatically:
    1. Pulls code (Source: GitHub/CodeCommit)
    2. Builds & tests (Build: CodeBuild)
    3. Deploys to staging (Deploy: CodeDeploy/ECS)
    4. Waits for approval
    5. Deploys to production (Deploy: CodeDeploy/ECS)
```

AWS CodePipeline is a fully managed continuous delivery (CD) service that orchestrates your build, test, and deploy workflows. It connects source repositories, build services, and deployment targets into an automated pipeline.

```
What you'll learn:
├── CodePipeline Fundamentals (stages, actions, artifacts)
├── Creating a pipeline (full portal walkthrough)
├── Action types (source, build, test, deploy, approval, invoke)
├── Manual approvals & notifications
├── Cross-account and cross-region pipelines
├── Terraform & CLI examples
└── Real-world CI/CD pipeline patterns
```

---

## Part 1: CodePipeline Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW CODEPIPELINE WORKS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌──────┐  ┌─────────┐│
│ │ Source   │─►│  Build    │─►│  Test    │─►│Approve│─►│ Deploy  ││
│ │ Stage    │  │  Stage    │  │  Stage   │  │ Stage │  │ Stage   ││
│ │          │  │           │  │          │  │       │  │         ││
│ │GitHub    │  │CodeBuild  │  │CodeBuild │  │Manual │  │CodeDeploy│
│ │CodeCommit│  │Jenkins    │  │3rd party │  │SNS    │  │ECS      ││
│ │S3       │  │           │  │          │  │       │  │CloudFmt ││
│ │ECR      │  │           │  │          │  │       │  │S3       ││
│ └──────────┘  └───────────┘  └──────────┘  └──────┘  └─────────┘│
│       │              │              │                       │      │
│       └──── Artifacts stored in S3 (encrypted) ────────────┘      │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Pipeline: Workflow definition (stages in sequence)           │
│ ├── Stage: Group of actions (runs in order)                      │
│ ├── Action: Single task within a stage                           │
│ ├── Artifact: Input/output files passed between stages          │
│ ├── Transition: Connection between stages (can be disabled)     │
│ └── Trigger: What starts the pipeline (push, schedule, manual)  │
│                                                                       │
│ Pricing:                                                             │
│ ├── V1 pipeline type: $1.00/month per active pipeline           │
│ ├── V2 pipeline type: $0.002/min action execution               │
│ └── Free tier: 1 V1 pipeline free per month                     │
│                                                                       │
│ V1 vs V2 pipelines:                                                 │
│ ├── V2: Supports triggers (webhooks, scheduled), variables     │
│ ├── V2: Pipeline-level variables and overrides                  │
│ ├── V2: Git tags as triggers                                     │
│ └── ⚡ Use V2 for new pipelines                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Pipeline (Full Portal Walkthrough)

```
Console → CodePipeline → Pipelines → Create pipeline

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: PIPELINE SETTINGS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Pipeline name: [my-app-pipeline]                               │
│ Pipeline type: ● V2 (recommended) ○ V1                        │
│                                                                   │
│ Execution mode:                                                 │
│ ● Queued (⚡ recommended)                                      │
│ ○ Superseded (new execution cancels in-progress)              │
│ ○ Parallel (run concurrently)                                 │
│                                                                   │
│ → Queued: Executions wait in order                            │
│ → Superseded: Only latest version deploys                    │
│ → Parallel: Multiple runs at same time                       │
│                                                                   │
│ Service role:                                                   │
│ ● New service role                                             │
│ ○ Existing service role                                       │
│ Role name: [AWSCodePipelineServiceRole-my-app]                │
│                                                                   │
│ Variables (V2 only):                                           │
│ Name: [DEPLOY_ENV]  Default value: [production]               │
│ → Variables accessible as #{variables.DEPLOY_ENV}            │
│                                                                   │
│ ── Advanced settings ──                                        │
│ Artifact store:                                                │
│ ● Default location (S3 bucket auto-created)                   │
│ ○ Custom location (specify S3 bucket)                         │
│                                                                   │
│ Encryption key:                                                │
│ ● Default AWS managed key                                     │
│ ○ Customer managed key [arn:aws:kms:... ▼]                   │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: SOURCE STAGE                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Source provider: [GitHub (via connection) ▼]                   │
│   ├── AWS CodeCommit                                           │
│   ├── Amazon ECR (container image changes)                    │
│   ├── Amazon S3 (file changes)                                │
│   ├── GitHub (Version 2 — via CodeStar connection)            │
│   ├── Bitbucket (via connection)                              │
│   └── GitLab (via connection)                                 │
│                                                                   │
│ Connection: [my-github-connection ▼]                          │
│ → CodeStar Connections → Create connection                   │
│                                                                   │
│ Repository name: [my-org/my-application ▼]                    │
│ Default branch: [main]                                        │
│                                                                   │
│ Output artifact format:                                        │
│ ● CodePipeline default (ZIP)                                  │
│ ○ Full clone (passes Git metadata)                            │
│                                                                   │
│ Trigger (V2):                                                  │
│ ● Push on branch [main]                                       │
│ ○ Push on tag matching pattern [v*]                           │
│ ○ Pull request (on open/update/close)                         │
│ ○ No trigger (manual or scheduled)                            │
│                                                                   │
│ Filter:                                                        │
│ Include: File paths: [src/**]                                 │
│ Exclude: File paths: [docs/**, README.md]                     │
│ → Only trigger when source files change                      │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: BUILD STAGE                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Build provider: [AWS CodeBuild ▼]                             │
│   ├── AWS CodeBuild                                            │
│   ├── Jenkins                                                  │
│   ├── CloudBees                                                │
│   └── TeamCity                                                 │
│                                                                   │
│ Region: [US East (N. Virginia) ▼]                             │
│ Project name: [my-app-build ▼]                                │
│                                                                   │
│ Build type: ● Single build ○ Batch build                      │
│                                                                   │
│ Environment variables:                                         │
│ Name: [DEPLOY_ENV]                                            │
│ Value: [#{variables.DEPLOY_ENV}]                              │
│ → Pass pipeline variables to build                           │
│                                                                   │
│ ○ Skip build stage                                            │
│ → Direct source to deploy (e.g., S3 static hosting)         │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 4: DEPLOY STAGE                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Deploy provider: [AWS CodeDeploy ▼]                           │
│   ├── AWS CodeDeploy (EC2, Lambda, ECS)                       │
│   ├── Amazon ECS (direct ECS deploy)                          │
│   ├── Amazon S3 (deploy files to S3)                          │
│   ├── AWS CloudFormation (IaC deploy)                         │
│   ├── AWS CloudFormation Stack Set                            │
│   ├── AWS Elastic Beanstalk                                   │
│   ├── AWS AppConfig                                            │
│   └── AWS Service Catalog                                     │
│                                                                   │
│ Application name: [my-web-app ▼]                              │
│ Deployment group: [prod-deployment-group ▼]                   │
│                                                                   │
│ ○ Skip deploy stage                                           │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 5: REVIEW                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Pipeline settings:                                              │
│ ├── Name: my-app-pipeline                                     │
│ ├── Type: V2                                                   │
│ └── Service role: AWSCodePipelineServiceRole-my-app           │
│                                                                   │
│ Source: GitHub → my-org/my-application (main)                 │
│ Build: CodeBuild → my-app-build                               │
│ Deploy: CodeDeploy → prod-deployment-group                    │
│                                                                   │
│                    [Create pipeline]                            │
│ → Pipeline executes immediately on creation                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Stages & Actions

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACTION TYPES                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Source actions:                                                      │
│ ├── CodeCommit, GitHub, Bitbucket, GitLab                        │
│ ├── Amazon S3 (file upload triggers pipeline)                   │
│ └── Amazon ECR (image push triggers pipeline)                   │
│                                                                       │
│ Build actions:                                                       │
│ ├── AWS CodeBuild (⚡ native)                                    │
│ ├── Jenkins, CloudBees, TeamCity                                 │
│ └── Custom action (your own build provider)                      │
│                                                                       │
│ Test actions:                                                        │
│ ├── AWS CodeBuild (run tests)                                    │
│ ├── AWS Device Farm (mobile testing)                             │
│ ├── BlazeMeter, Ghost Inspector                                  │
│ └── Custom test provider                                          │
│                                                                       │
│ Deploy actions:                                                      │
│ ├── CodeDeploy, ECS, S3, CloudFormation                         │
│ ├── Elastic Beanstalk, Service Catalog                          │
│ └── AppConfig, CloudFormation StackSets                          │
│                                                                       │
│ Approval actions:                                                    │
│ ├── Manual approval (SNS notification + URL)                    │
│ └── Blocks pipeline until approved/rejected                     │
│                                                                       │
│ Invoke actions:                                                      │
│ ├── AWS Lambda (run custom logic)                                │
│ ├── AWS Step Functions                                            │
│ └── Custom action                                                 │
│                                                                       │
│ Actions within a stage:                                              │
│ ├── Can run in sequence (run order 1, 2, 3...)                  │
│ ├── Can run in parallel (same run order number)                 │
│ └── Stage completes when ALL actions complete                   │
│                                                                       │
│ Example: Parallel deploy to staging + integration test:            │
│ ┌─────────────────────────────────────────────────────────────┐  │
│ │ Deploy Stage                                                 │  │
│ │ ├── Action: Deploy-to-Staging (runOrder: 1)                │  │
│ │ ├── Action: Run-Integration-Tests (runOrder: 2)            │  │
│ │ └── Action: Deploy-to-Canary (runOrder: 2) ← parallel     │  │
│ └─────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Manual Approvals & Notifications

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANUAL APPROVALS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Add approval action:                                                 │
│ Console → Pipeline → Edit → Add stage → Add action                 │
│                                                                       │
│ Action name: [approval-before-prod]                                 │
│ Action provider: [Manual approval ▼]                                │
│ SNS topic: [arn:aws:sns:us-east-1:123456:pipeline-approvals]      │
│ URL for review: [https://staging.myapp.com]                        │
│ Comments: [Review staging deployment before prod deploy]           │
│                                                                       │
│ → Email sent to SNS subscribers with approve/reject link          │
│ → Pipeline pauses until someone approves (or times out)           │
│ → Timeout: 7 days (default)                                       │
│                                                                       │
│ Notifications (separate from approvals):                            │
│ Console → Pipeline → Settings → Create notification rule           │
│                                                                       │
│ Events:                                                              │
│ ☑ Pipeline execution: Started, Succeeded, Failed                  │
│ ☑ Stage execution: Started, Succeeded, Failed                     │
│ ☑ Action execution: Started, Succeeded, Failed                    │
│ ☑ Manual approval: Needed, Succeeded, Failed                     │
│                                                                       │
│ Target: SNS topic or AWS Chatbot (Slack/Teams)                    │
│ → ⚡ Slack notifications via AWS Chatbot = best practice          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Cross-Account & Cross-Region

```
┌─────────────────────────────────────────────────────────────────────┐
│           CROSS-ACCOUNT & CROSS-REGION                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cross-Region:                                                        │
│ ├── Deploy to different regions from same pipeline               │
│ ├── Pipeline creates artifact bucket in each target region      │
│ ├── Artifacts replicated to target region automatically         │
│ └── Common: Deploy to us-east-1 + eu-west-1 from single pipe  │
│                                                                       │
│ Cross-Account:                                                       │
│ ├── Pipeline in Account A, deploy to Account B                  │
│ ├── Requires:                                                     │
│ │   ├── KMS key shared between accounts                        │
│ │   ├── S3 bucket policy allowing cross-account access        │
│ │   ├── IAM role in target account (CodePipeline assumes)     │
│ │   └── IAM role in source account with assume-role           │
│ └── Pattern: Dev pipeline → Staging account → Prod account    │
│                                                                       │
│ ⚡ Cross-account is the recommended pattern:                       │
│    Dev Account → Build/Test                                       │
│    Staging Account → Deploy + Integration Test                    │
│    Prod Account → Deploy (with approval gate)                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & CLI Examples

```hcl
resource "aws_codepipeline" "app" {
  name          = "my-app-pipeline"
  role_arn      = aws_iam_role.codepipeline.arn
  pipeline_type = "V2"

  artifact_store {
    location = aws_s3_bucket.pipeline_artifacts.id
    type     = "S3"
    encryption_key {
      id   = aws_kms_key.pipeline.arn
      type = "KMS"
    }
  }

  stage {
    name = "Source"
    action {
      name             = "GitHub"
      category         = "Source"
      owner            = "AWS"
      provider         = "CodeStarSourceConnection"
      version          = "1"
      output_artifacts = ["source_output"]
      configuration = {
        ConnectionArn    = aws_codestarconnections_connection.github.arn
        FullRepositoryId = "my-org/my-app"
        BranchName       = "main"
      }
    }
  }

  stage {
    name = "Build"
    action {
      name             = "Build"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      version          = "1"
      input_artifacts  = ["source_output"]
      output_artifacts = ["build_output"]
      configuration = {
        ProjectName = aws_codebuild_project.app.name
      }
    }
  }

  stage {
    name = "Approval"
    action {
      name     = "ManualApproval"
      category = "Approval"
      owner    = "AWS"
      provider = "Manual"
      version  = "1"
      configuration = {
        NotificationArn = aws_sns_topic.approvals.arn
        CustomData      = "Review staging at https://staging.myapp.com"
      }
    }
  }

  stage {
    name = "Deploy"
    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "CodeDeploy"
      version         = "1"
      input_artifacts = ["build_output"]
      configuration = {
        ApplicationName     = aws_codedeploy_app.web.name
        DeploymentGroupName = aws_codedeploy_deployment_group.prod.deployment_group_name
      }
    }
  }
}
```

```bash
# Create pipeline from JSON definition
aws codepipeline create-pipeline --cli-input-json file://pipeline.json

# Get pipeline state
aws codepipeline get-pipeline-state --name my-app-pipeline

# Start pipeline execution
aws codepipeline start-pipeline-execution --name my-app-pipeline

# Approve manual approval
aws codepipeline put-approval-result \
  --pipeline-name my-app-pipeline \
  --stage-name Approval \
  --action-name ManualApproval \
  --result summary="Approved by admin",status=Approved \
  --token approval-token-id

# Disable/enable transition between stages
aws codepipeline disable-stage-transition \
  --pipeline-name my-app-pipeline \
  --stage-name Deploy \
  --transition-type Inbound \
  --reason "Maintenance window"
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD CI/CD PATTERNS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Full CI/CD Pipeline                                      │
│ Source → Build → Test → Staging → Approval → Production           │
│ ├── Source: GitHub (V2 connection)                                │
│ ├── Build: CodeBuild (compile + unit tests)                      │
│ ├── Test: CodeBuild (integration tests)                          │
│ ├── Staging: CodeDeploy to staging environment                   │
│ ├── Approval: Manual approval (Slack notification)               │
│ └── Production: CodeDeploy blue/green to production              │
│                                                                       │
│ Pattern 2: Container Pipeline                                       │
│ Source → Build+Push → Deploy ECS                                   │
│ ├── Source: GitHub + ECR (dual source for code + base image)    │
│ ├── Build: CodeBuild (docker build, push to ECR)                │
│ └── Deploy: ECS (blue/green with CodeDeploy)                    │
│                                                                       │
│ Pattern 3: Infrastructure Pipeline                                  │
│ Source → Validate → Deploy CF                                      │
│ ├── Source: GitHub (CloudFormation templates)                    │
│ ├── Build: CodeBuild (cfn-lint, cfn-nag validation)             │
│ ├── Deploy: CloudFormation CreateChangeSet                       │
│ ├── Approval: Review change set                                  │
│ └── Execute: CloudFormation ExecuteChangeSet                     │
│                                                                       │
│ Pattern 4: Multi-Region Deploy                                      │
│ Source → Build → Deploy(us-east-1) + Deploy(eu-west-1)            │
│ ├── Single source and build                                       │
│ └── Parallel deploy to multiple regions in same stage            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
CodePipeline Quick Reference:
├── What: Managed CI/CD orchestration service
├── Components: Pipeline → Stages → Actions → Artifacts
├── Sources: GitHub, CodeCommit, S3, ECR, Bitbucket, GitLab
├── Build: CodeBuild, Jenkins
├── Deploy: CodeDeploy, ECS, CloudFormation, S3, Beanstalk
├── Approvals: Manual (SNS + Slack), blocks pipeline
├── Pipeline type: V2 (⚡ recommended) — triggers, variables
├── Execution: Queued, Superseded, or Parallel
├── Cross-account: IAM roles + KMS + S3 bucket policy
├── Cross-region: Auto artifact replication
├── Pricing: V2 = $0.002/min, V1 = $1/month/pipeline
└── ⚡ Slack notifications via AWS Chatbot
```

---

## What's Next?

In **Chapter 34: CloudFormation**, we'll dive into Infrastructure as Code — templates, stacks, change sets, drift detection, nested stacks, and StackSets.
