# Chapter 32: CodeDeploy

---

## Table of Contents

- [Overview](#overview)
- [Part 1: CodeDeploy Fundamentals](#part-1-codedeploy-fundamentals)
- [Part 2: EC2/On-Premises Deployment (Full Portal Walkthrough)](#part-2-ec2on-premises-deployment-full-portal-walkthrough)
- [Part 3: Lambda Deployment](#part-3-lambda-deployment)
- [Part 4: ECS Deployment (Blue/Green)](#part-4-ecs-deployment-bluegreen)
- [Part 5: appspec.yml Deep Dive](#part-5-appspecyml-deep-dive)
- [Part 6: Deployment Configurations & Rollback](#part-6-deployment-configurations--rollback)
- [Part 7: Terraform & CLI Examples](#part-7-terraform--cli-examples)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Deployment Automation? Why Do We Need It?

After you build your application, you need to **deploy** it — put the new version onto your servers so users can access it. Doing this manually (SSH into each server, stop the app, copy files, restart) is risky and slow, especially with 50+ servers.

**CodeDeploy automates this entire process.** Think of deployment strategies like updating a restaurant's menu:
- **In-place (Rolling):** Update one table's menu at a time while others keep the old one — some disruption but manageable
- **Blue/Green:** Print all new menus first, then swap every table at once — zero disruption, easy to roll back
- **Canary:** Give the new menu to just 2 tables first, see if customers complain, then roll out to everyone

AWS CodeDeploy automates application deployments to EC2, on-premises servers, Lambda functions, and ECS services. It supports in-place, rolling, blue/green, and canary deployment strategies.

```
What you'll learn:
├── CodeDeploy Fundamentals (agent, appspec, hooks)
├── EC2/On-Premises deployments
├── Lambda traffic shifting
├── ECS Blue/Green deployments
├── appspec.yml (all hooks explained)
├── Deployment configurations (rollout strategies)
├── Rollback strategies
└── Terraform & CLI examples
```

---

## Part 1: CodeDeploy Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW CODEDEPLOY WORKS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────┐    ┌─────────────────┐    ┌────────────────────┐ │
│ │  Application │    │ Deployment Group │    │  Targets           │ │
│ │  (logical    │───►│ (set of targets) │───►│  ├── EC2 instances │ │
│ │   container) │    │  + config        │    │  ├── Lambda        │ │
│ └──────────────┘    └─────────────────┘    │  ├── ECS services  │ │
│                                             │  └── On-premises   │ │
│                                             └────────────────────┘ │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Application: Logical container for deployments               │
│ ├── Deployment group: Set of target instances/services           │
│ ├── Deployment configuration: Rules for how to deploy            │
│ ├── Revision: Application code + appspec.yml                     │
│ ├── appspec.yml: Deployment instructions file                    │
│ └── CodeDeploy Agent: Runs on EC2/on-prem (not needed for       │
│     Lambda/ECS)                                                     │
│                                                                       │
│ Compute platforms:                                                   │
│ ┌──────────────────┬────────────────────────────────────────────┐ │
│ │ Platform         │ Deployment types                           │ │
│ ├──────────────────┼────────────────────────────────────────────┤ │
│ │ EC2/On-Premises  │ In-place, Blue/Green                      │ │
│ │ Lambda           │ Canary, Linear, All-at-once               │ │
│ │ ECS              │ Blue/Green (canary, linear, all-at-once)  │ │
│ └──────────────────┴────────────────────────────────────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: EC2/On-Premises Deployment (Full Portal Walkthrough)

```
Console → CodeDeploy → Applications → Create application

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: CREATE APPLICATION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Application name: [my-web-app]                                 │
│ Compute platform:                                              │
│ ● EC2/On-premises  ○ AWS Lambda  ○ Amazon ECS                 │
│                                                                   │
│                    [Create application]                         │
└─────────────────────────────────────────────────────────────────┘

Console → [Application] → Deployment groups → Create deployment group

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: CREATE DEPLOYMENT GROUP                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Deployment group name: [prod-deployment-group]                 │
│                                                                   │
│ Service role: [arn:aws:iam::123456:role/CodeDeployRole ▼]     │
│ → Needs: AmazonEC2FullAccess, AWSCodeDeployRole,             │
│   AutoScalingFullAccess, ElasticLoadBalancingFullAccess       │
│                                                                   │
│ Deployment type:                                               │
│ ● In-place                                                     │
│ ○ Blue/green                                                   │
│                                                                   │
│ → In-place: Deploys to existing instances (downtime possible)│
│   Each instance stopped → updated → restarted                │
│ → Blue/green: Creates new instances → shifts traffic →       │
│   terminates old instances (zero downtime)                    │
│                                                                   │
│ ── Environment configuration ──                                │
│ ☑ Amazon EC2 Auto Scaling groups                              │
│   ASG: [prod-web-asg ▼]                                      │
│ ☑ Amazon EC2 instances                                        │
│   Tag group:                                                   │
│   Key: [environment ▼]  Value: [production]                   │
│   → Targets instances matching these tags                    │
│ ☐ On-premises instances                                       │
│                                                                   │
│ ── Agent configuration ──                                      │
│ Install CodeDeploy Agent:                                      │
│ ● Now and schedule updates (14 days)                          │
│ ○ Only once                                                    │
│ ○ Never (assume agent already installed)                      │
│                                                                   │
│ ── Deployment settings ──                                      │
│ Deployment configuration:                                      │
│ ● CodeDeployDefault.OneAtATime                                │
│ ○ CodeDeployDefault.HalfAtATime                               │
│ ○ CodeDeployDefault.AllAtOnce                                 │
│ ○ Custom (create your own)                                    │
│                                                                   │
│ → OneAtATime: Safest, slowest. Deploy 1 instance at a time. │
│ → HalfAtATime: Deploy to 50% of instances simultaneously.   │
│ → AllAtOnce: Deploy all at once (fastest, riskiest).         │
│ → Custom: Specify minimum healthy percentage.                │
│                                                                   │
│ ── Load balancer ──                                             │
│ ☑ Enable load balancing                                       │
│ ● Application Load Balancer  ○ Network Load Balancer         │
│ ○ Classic Load Balancer                                       │
│ Target group: [prod-web-tg ▼]                                 │
│ → Deregisters instance from LB during deployment             │
│ → Re-registers after successful deployment                   │
│ → ⚡ Always use with load balancers for zero-downtime        │
│                                                                   │
│ ── Rollbacks ──                                                 │
│ ☑ Roll back when a deployment fails                           │
│ ☑ Roll back when alarm thresholds are met                     │
│                                                                   │
│ ── Alarms ──                                                    │
│ CloudWatch alarms: [prod-cpu-alarm, prod-5xx-alarm ▼]        │
│ → Triggers rollback if alarm enters ALARM state              │
│                                                                   │
│                    [Create deployment group]                    │
└─────────────────────────────────────────────────────────────────┘

Console → [Application] → [Deployment group] → Create deployment

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: CREATE DEPLOYMENT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Revision type:                                                  │
│ ● My application is stored in Amazon S3                       │
│   Revision location: [s3://my-bucket/app-v2.zip]              │
│ ○ My application is stored in GitHub                          │
│   Repository name: [my-org/my-app]                            │
│   Commit ID: [abc123]                                          │
│                                                                   │
│ Revision file type: ● .zip ○ .tar ○ .tar.gz                  │
│                                                                   │
│ Description: [Deploy v2.1.0 with bug fixes]                   │
│                                                                   │
│ Deployment group overrides:                                    │
│ ├── Deployment config (optional override)                    │
│ └── Rollback configuration (optional override)               │
│                                                                   │
│ ☐ Don't fail the deployment to an instance if lifecycle      │
│   event script fails                                           │
│                                                                   │
│                    [Create deployment]                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Lambda Deployment

```
┌─────────────────────────────────────────────────────────────────────┐
│           LAMBDA DEPLOYMENT                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ CodeDeploy shifts traffic between Lambda function versions:        │
│                                                                       │
│ ┌──────────────┐         ┌─────────────┐                           │
│ │ Lambda Alias │─── 90% ─│ Version 1   │ (current)               │
│ │ (prod)       │─── 10% ─│ Version 2   │ (new)                   │
│ └──────────────┘         └─────────────┘                           │
│                                                                       │
│ Deployment configurations:                                          │
│ ├── Canary10Percent5Minutes: 10% → wait 5 min → 100%           │
│ ├── Canary10Percent10Minutes: 10% → wait 10 min → 100%         │
│ ├── Canary10Percent15Minutes: 10% → wait 15 min → 100%         │
│ ├── Canary10Percent30Minutes: 10% → wait 30 min → 100%         │
│ ├── Linear10PercentEvery1Minute: +10% every minute              │
│ ├── Linear10PercentEvery2Minutes: +10% every 2 minutes          │
│ ├── Linear10PercentEvery3Minutes: +10% every 3 minutes          │
│ ├── Linear10PercentEvery10Minutes: +10% every 10 minutes        │
│ └── AllAtOnce: Immediate 100% shift                              │
│                                                                       │
│ appspec.yml for Lambda:                                             │
│ version: 0.0                                                        │
│ Resources:                                                           │
│   - myFunction:                                                     │
│       Type: AWS::Lambda::Function                                   │
│       Properties:                                                    │
│         Name: "my-function"                                         │
│         Alias: "prod"                                                │
│         CurrentVersion: "1"                                          │
│         TargetVersion: "2"                                           │
│ Hooks:                                                               │
│   - BeforeAllowTraffic: "CodeDeployHook_preTrafficTest"           │
│   - AfterAllowTraffic: "CodeDeployHook_postTrafficTest"           │
│                                                                       │
│ ⚡ Pre/post traffic hooks: Run validation Lambda functions         │
│ → Test new version before/after traffic shift                     │
│ → Return "Succeeded" or "Failed" to continue/rollback            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: ECS Deployment (Blue/Green)

```
┌─────────────────────────────────────────────────────────────────────┐
│           ECS BLUE/GREEN DEPLOYMENT                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────┐    ┌──────────────┐    ┌────────────────────────┐     │
│ │   ALB   │    │ Target Group │    │ ECS Service            │     │
│ │         │───►│ Blue (prod)  │───►│ Task Set (current)     │     │
│ │         │    │ Port: 80     │    │ ├── Task 1             │     │
│ │         │    └──────────────┘    │ └── Task 2             │     │
│ │         │    ┌──────────────┐    │                         │     │
│ │         │───►│ Target Group │───►│ Task Set (new/green)   │     │
│ │ Test    │    │ Green (test) │    │ ├── Task 3             │     │
│ │ listener│    │ Port: 8080   │    │ └── Task 4             │     │
│ │ :8080   │    └──────────────┘    └────────────────────────┘     │
│ └─────────┘                                                        │
│                                                                       │
│ Traffic shifting:                                                    │
│ 1. New task set created with new task definition                   │
│ 2. Test listener routes to green target group (validation)        │
│ 3. Traffic shifts from blue to green (canary/linear/all-at-once) │
│ 4. Old task set terminated after wait period                       │
│                                                                       │
│ Required:                                                            │
│ ├── ALB with TWO target groups                                   │
│ ├── Production listener (port 80)                                │
│ ├── Test listener (port 8080) — optional but recommended        │
│ └── ECS service configured for CODE_DEPLOY controller            │
│                                                                       │
│ appspec.yml for ECS:                                                │
│ version: 0.0                                                        │
│ Resources:                                                           │
│   - TargetService:                                                  │
│       Type: AWS::ECS::Service                                       │
│       Properties:                                                    │
│         TaskDefinition: "arn:aws:ecs:...:task-def/app:5"          │
│         LoadBalancerInfo:                                            │
│           ContainerName: "app"                                      │
│           ContainerPort: 8080                                       │
│ Hooks:                                                               │
│   - BeforeInstall: "LambdaFunctionForSmokeTests"                  │
│   - AfterInstall: "LambdaFunctionForIntegrationTests"             │
│   - AfterAllowTestTraffic: "LambdaFunctionForTestTraffic"        │
│   - BeforeAllowTraffic: "LambdaFunctionForPreTraffic"            │
│   - AfterAllowTraffic: "LambdaFunctionForPostTraffic"            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: appspec.yml Deep Dive

```yaml
# appspec.yml for EC2/On-Premises
version: 0.0
os: linux

files:
  - source: /
    destination: /var/www/html
  - source: /config/nginx.conf
    destination: /etc/nginx/nginx.conf

permissions:
  - object: /var/www/html
    owner: www-data
    group: www-data
    mode: "755"
    type:
      - directory
  - object: /var/www/html
    owner: www-data
    group: www-data
    mode: "644"
    pattern: "**/*.html"

hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: root
  ValidateService:
    - location: scripts/validate.sh
      timeout: 300
      runas: root
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           EC2 DEPLOYMENT LIFECYCLE HOOKS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ApplicationStop          → Stop current application                │
│        ↓                                                              │
│ DownloadBundle           → Download revision from S3/GitHub        │
│        ↓                   (managed by agent — no script)          │
│ BeforeInstall            → Pre-installation tasks                  │
│        ↓                   (backup, decrypt, cleanup)              │
│ Install                  → Copy files to destination               │
│        ↓                   (managed by agent — no script)          │
│ AfterInstall             → Configure app, set permissions          │
│        ↓                   (config files, symlinks)                │
│ ApplicationStart         → Start the application                   │
│        ↓                   (systemctl start, npm start)            │
│ ValidateService          → Verify deployment succeeded             │
│                            (health checks, smoke tests)            │
│                                                                       │
│ Blue/Green adds:                                                     │
│ BeforeBlockTraffic → Before LB deregistration                     │
│ BlockTraffic       → Deregister from LB (managed)                 │
│ AfterBlockTraffic  → After LB deregistration                      │
│ BeforeAllowTraffic → Before LB registration                       │
│ AllowTraffic       → Register with LB (managed)                   │
│ AfterAllowTraffic  → After LB registration                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Deployment Configurations & Rollback

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEPLOYMENT STRATEGIES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ EC2 In-Place:                                                        │
│ ├── AllAtOnce: All instances at same time (fastest, risky)       │
│ ├── HalfAtATime: 50% at a time                                   │
│ ├── OneAtATime: 1 instance at a time (safest, slowest)          │
│ └── Custom: MinimumHealthyHosts = count or percentage            │
│                                                                       │
│ EC2 Blue/Green:                                                      │
│ ├── Creates new ASG with new instances                           │
│ ├── Deploys to new instances                                      │
│ ├── Shifts ALB traffic to new instances                          │
│ ├── Terminates old ASG (after wait period)                       │
│ └── ⚡ Zero downtime, instant rollback                           │
│                                                                       │
│ Rollback:                                                            │
│ ├── Automatic: On deployment failure or CloudWatch alarm         │
│ ├── Manual: Console → Deployment → Stop and roll back           │
│ ├── Rollback = new deployment of last known good revision       │
│ └── ⚠️ Not a "revert" — it's a new forward deployment           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & CLI Examples

```hcl
resource "aws_codedeploy_app" "web" {
  name             = "my-web-app"
  compute_platform = "Server"  # Server, Lambda, or ECS
}

resource "aws_codedeploy_deployment_group" "prod" {
  app_name               = aws_codedeploy_app.web.name
  deployment_group_name  = "prod-deployment-group"
  service_role_arn       = aws_iam_role.codedeploy.arn
  deployment_config_name = "CodeDeployDefault.OneAtATime"

  autoscaling_groups = [aws_autoscaling_group.web.name]

  deployment_style {
    deployment_option = "WITH_TRAFFIC_CONTROL"
    deployment_type   = "IN_PLACE"
  }

  load_balancer_info {
    target_group_info {
      name = aws_lb_target_group.web.name
    }
  }

  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }

  alarm_configuration {
    alarms  = ["prod-cpu-high", "prod-5xx-high"]
    enabled = true
  }
}
```

```bash
# Create application
aws deploy create-application \
  --application-name my-web-app \
  --compute-platform Server

# Create deployment
aws deploy create-deployment \
  --application-name my-web-app \
  --deployment-group-name prod-deployment-group \
  --s3-location bucket=my-bucket,key=app-v2.zip,bundleType=zip

# Get deployment status
aws deploy get-deployment --deployment-id d-ABCDEFGH1

# Stop deployment
aws deploy stop-deployment --deployment-id d-ABCDEFGH1

# Install CodeDeploy agent on EC2
sudo yum install -y ruby wget
wget https://aws-codedeploy-us-east-1.s3.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
```

---

## Part 8: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: EC2 Blue/Green with ASG                                  │
│ ├── Original ASG (blue) running current version                  │
│ ├── CodeDeploy creates new ASG (green) with new launch template │
│ ├── Deploys to green, runs health checks                         │
│ ├── ALB shifts traffic blue → green                              │
│ ├── Wait period (e.g., 60 min) before terminating blue          │
│ └── ⚡ Roll back = shift traffic back to blue (instant)          │
│                                                                       │
│ Pattern 2: ECS Blue/Green with Canary                               │
│ ├── Canary10Percent5Minutes                                       │
│ ├── 10% traffic to new task set                                   │
│ ├── Monitor metrics for 5 minutes                                │
│ ├── If healthy: shift remaining 90%                               │
│ └── If unhealthy: rollback (automatic)                           │
│                                                                       │
│ Pattern 3: Lambda Gradual Rollout                                   │
│ ├── Linear10PercentEvery2Minutes                                 │
│ ├── PreTraffic hook: Run integration tests against new version  │
│ ├── PostTraffic hook: Verify metrics after full shift            │
│ └── CloudWatch alarm triggers rollback if errors spike          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
CodeDeploy Quick Reference:
├── Platforms: EC2/On-Premises, Lambda, ECS
├── Spec file: appspec.yml (in revision root)
├── EC2 types: In-place, Blue/Green
├── Lambda: Canary, Linear, All-at-once (traffic shifting)
├── ECS: Blue/Green with traffic shifting
├── Agent: Required on EC2/on-premises (not Lambda/ECS)
├── Rollback: Automatic on failure or alarm
├── Hooks: Lifecycle scripts for each deployment phase
├── Pricing: Free for EC2, charges for Lambda/ECS deployments
└── ⚡ Blue/Green = zero downtime + instant rollback
```

---

## What's Next?

In **Chapter 33: CodePipeline**, we'll tie everything together — building CI/CD pipelines with stages, actions, approvals, and cross-account deployments.
