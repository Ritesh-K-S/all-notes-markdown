# Chapter 14: EC2 Auto Scaling

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Auto Scaling Concepts](#part-1-auto-scaling-concepts)
- [Part 2: Launch Templates](#part-2-launch-templates)
- [Part 3: Auto Scaling Group (Full Walkthrough)](#part-3-auto-scaling-group-full-walkthrough)
- [Part 4: Scaling Policies (Deep Dive)](#part-4-scaling-policies-deep-dive)
- [Part 5: Lifecycle Hooks](#part-5-lifecycle-hooks)
- [Part 6: Instance Refresh (Rolling Updates)](#part-6-instance-refresh-rolling-updates)
- [Part 7: Warm Pools](#part-7-warm-pools)
- [Part 8: Termination Policies & AZ Rebalancing](#part-8-termination-policies--az-rebalancing)
- [Part 9: Terraform Example](#part-9-terraform-example)
- [Part 10: CLI Reference](#part-10-cli-reference)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Auto Scaling? Why Can't I Just Run More Servers?

> **Real-World Analogy:** Auto Scaling is like a restaurant manager who hires extra waiters during the dinner rush and sends them home when it's quiet. You set the rules ("never fewer than 2 waiters, never more than 10") and the manager handles everything — hiring, firing, and replacing anyone who gets sick.

**Why does this matter?** Without Auto Scaling, you either over-provision (paying for idle servers 24/7) or under-provision (your app crashes during traffic spikes). Auto Scaling gives you the right number of servers at the right time — automatically.

EC2 Auto Scaling automatically adjusts the number of EC2 instances in your fleet based on demand. You define the desired capacity, minimum, and maximum — Auto Scaling handles launching, terminating, and health-checking instances. This chapter covers Launch Templates, Auto Scaling Groups (ASGs), scaling policies, lifecycle hooks, and instance refresh.

```
What you'll learn:
├── Auto Scaling Concepts
│   ├── Why Auto Scale (availability + cost)
│   ├── Components (Launch Template, ASG, Scaling Policy)
│   └── How it works (desired, min, max)
├── Launch Templates
│   ├── Full creation walkthrough
│   ├── Versioning
│   └── Launch Template vs Launch Configuration (legacy)
├── Auto Scaling Groups (ASG)
│   ├── Full creation walkthrough (every field)
│   ├── Multi-AZ deployment
│   ├── Load balancer integration
│   ├── Health checks (EC2 vs ELB vs custom)
│   └── Capacity settings (desired, min, max)
├── Scaling Policies
│   ├── Target Tracking (recommended)
│   ├── Step Scaling
│   ├── Simple Scaling (legacy)
│   ├── Scheduled Scaling
│   └── Predictive Scaling
├── Lifecycle Hooks
├── Instance Refresh (rolling updates)
├── Warm Pools
├── Instance Maintenance Policy
├── Terraform examples
├── CLI reference
└── Real-world patterns
```

---

## Part 1: Auto Scaling Concepts

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHY AUTO SCALING?                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Without Auto Scaling:                                                │
│ ├── Peak traffic → servers overloaded → site goes down            │
│ ├── Low traffic → servers idle → wasting money                    │
│ ├── Manual scaling → slow reaction, human error                   │
│ └── Server crash → manual replacement                              │
│                                                                       │
│ With Auto Scaling:                                                   │
│ ├── Peak traffic → automatically add instances                    │
│ ├── Low traffic → automatically remove instances                  │
│ ├── Instant reaction → based on CloudWatch metrics                │
│ ├── Server crash → automatically replaced (self-healing)         │
│ └── Cost optimized → only pay for what you need                  │
│                                                                       │
│ How it works:                                                        │
│                                                                       │
│                    ┌─── Maximum: 10 instances (safety limit)      │
│                    │                                                │
│  Instances    ─────┤   ← Auto Scaling adjusts between min/max    │
│                    │                                                │
│                    ├─── Desired: 4 instances (current target)     │
│                    │                                                │
│                    └─── Minimum: 2 instances (always running)     │
│                                                                       │
│ Components:                                                          │
│                                                                       │
│ ┌──────────────────┐   ┌──────────────────┐   ┌────────────────┐ │
│ │ Launch Template   │──▶│ Auto Scaling     │──▶│ Scaling Policy │ │
│ │ (WHAT to launch)  │   │ Group (WHERE)    │   │ (WHEN to scale)│ │
│ │                   │   │                  │   │                │ │
│ │ • AMI             │   │ • VPC/subnets    │   │ • Target track │ │
│ │ • Instance type   │   │ • Min/max/desired│   │ • Step scaling │ │
│ │ • Key pair        │   │ • Health checks  │   │ • Scheduled    │ │
│ │ • Security groups │   │ • Load balancer  │   │ • Predictive   │ │
│ │ • User data       │   │ • AZ distribution│   │                │ │
│ │ • IAM role        │   │                  │   │                │ │
│ └──────────────────┘   └──────────────────┘   └────────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Launch Templates

```
Console → EC2 → Launch Templates → Create launch template

┌─────────────────────────────────────────────────────────────────┐
│           CREATE LAUNCH TEMPLATE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Launch template name and description ──                      │
│ Name: [lt-web-prod]                                              │
│ Description: [Web server - Node.js 20, nginx, AL2023 ARM]      │
│ ☑ Provide guidance to help me set up a template                │
│   that I can use with EC2 Auto Scaling                          │
│                                                                   │
│ Template version description: [v1 - initial setup]             │
│                                                                   │
│ ── Application and OS Images (AMI) ──                           │
│ [ami-web-prod-v1.2.3 ▼] (your custom golden AMI)             │
│ Or Quick Start: Amazon Linux 2023 (arm64)                      │
│                                                                   │
│ ── Instance type ──                                             │
│ Primary: [m7g.large ▼] (2 vCPU, 8 GB)                        │
│                                                                   │
│ ☑ Include additional instance types (for mixed instances):    │
│   Secondary 1: [m7g.xlarge ▼]                                 │
│   Secondary 2: [m6g.large ▼]                                  │
│   Secondary 3: [c7g.large ▼]                                  │
│   ⚡ Mix instance types for better availability and Spot!       │
│                                                                   │
│ ── Key pair ──                                                  │
│ [kp-prod-web ▼] (or "Don't include" if using SSM only)       │
│ ⚡ For SSM-only access, skip key pair entirely                  │
│                                                                   │
│ ── Network settings ──                                          │
│ ⚠️ Do NOT specify subnet here — ASG controls subnet/AZ!        │
│ Security groups: [sg-web-prod ▼]                               │
│                                                                   │
│ ── Storage (volumes) ──                                         │
│ Root: 20 GiB, gp3, Encrypted ☑, Delete on termination ☑      │
│ [Add new volume]:                                               │
│   /dev/sdb: 50 GiB, gp3, Encrypted ☑                         │
│                                                                   │
│ ── Resource tags ──                                             │
│ ☑ Tag instances, volumes, and network interfaces               │
│ Name: web-prod                                                  │
│ Environment: prod                                               │
│ Team: backend                                                   │
│ ManagedBy: auto-scaling                                        │
│                                                                   │
│ ── Advanced details ──                                          │
│                                                                   │
│ IAM instance profile: [profile-ec2-web-prod ▼]                │
│ ⚡ Same IAM role as Ch13 — SSM, CloudWatch, S3                  │
│                                                                   │
│ Purchasing option:                                              │
│   ⚠️ Don't set here — configure in ASG for mixed fleet!        │
│                                                                   │
│ Metadata version: ● V2 only (IMDSv2)                          │
│ Metadata response hop limit: [2]                               │
│                                                                   │
│ User data:                                                      │
│ #!/bin/bash                                                     │
│ # Pull latest config from S3                                   │
│ aws s3 cp s3://config-bucket/web/nginx.conf /etc/nginx/        │
│ aws s3 cp s3://config-bucket/web/app.env /opt/app/.env         │
│ # Start application                                            │
│ systemctl enable app                                           │
│ systemctl start app                                            │
│ systemctl start nginx                                          │
│ # Signal ASG that instance is ready (for lifecycle hooks)      │
│ INSTANCE_ID=$(TOKEN=$(curl -s -X PUT                           │
│   "http://169.254.169.254/latest/api/token"                    │
│   -H "X-aws-ec2-metadata-token-ttl-seconds: 21600") &&        │
│   curl -s -H "X-aws-ec2-metadata-token: $TOKEN"               │
│   http://169.254.169.254/latest/meta-data/instance-id)         │
│                                                                   │
│ Detailed monitoring: ☑ Enable (1-minute CloudWatch metrics)   │
│ ⚡ Required for responsive auto scaling!                        │
│                                                                   │
│ [Create launch template]                                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Launch Template Versioning

```
┌─────────────────────────────────────────────────────────────────────┐
│           VERSIONING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Versions: Each change creates a new version.                       │
│ ├── v1: Initial (AL2023, m7g.large, Node.js 18)                  │
│ ├── v2: Updated AMI (Node.js 20)                                  │
│ ├── v3: Changed instance type (m7g.xlarge)                        │
│ └── v4: Updated user data script                                  │
│                                                                       │
│ Default version: What ASG uses when set to "$Default"            │
│ Latest version: Most recently created version                     │
│                                                                       │
│ ASG can reference:                                                   │
│ ├── Specific version: lt-web-prod version 3                      │
│ ├── $Default: Uses the default version (you control which)       │
│ └── $Latest: Always uses newest version                           │
│   ⚡ Use $Default for production (explicit control)                │
│   ⚡ Use $Latest for dev/test (auto picks newest)                  │
│                                                                       │
│ Create new version:                                                  │
│ EC2 → Launch Templates → Select → Actions →                      │
│ Modify template (Create new version)                               │
│   Source template version: [3 ▼] (base on version 3)             │
│   Change what you need → Create                                   │
│                                                                       │
│ Set default version:                                                 │
│ EC2 → Launch Templates → Select → Actions →                      │
│ Set default version → Version [4]                                 │
│                                                                       │
│ ⚠️ Launch Configuration (legacy):                                   │
│    Old way, can't be versioned, can't be modified.               │
│    ALWAYS use Launch Templates instead.                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Auto Scaling Group (Full Walkthrough)

```
Console → EC2 → Auto Scaling Groups → Create Auto Scaling group

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: CHOOSE LAUNCH TEMPLATE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Auto Scaling group name: [asg-web-prod]                         │
│                                                                   │
│ Launch template:                                                │
│   Template: [lt-web-prod ▼]                                    │
│   Version: [$Default ▼]                                        │
│                                                                   │
│ [Next]                                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: CHOOSE INSTANCE LAUNCH OPTIONS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Network ──                                                   │
│ VPC: [vpc-prod ▼]                                              │
│ Availability Zones and subnets:                                │
│   ☑ subnet-priv-1a (ap-south-1a)                             │
│   ☑ subnet-priv-1b (ap-south-1b)                             │
│   ☑ subnet-priv-1c (ap-south-1c)                             │
│   ⚡ Select ALL AZs for high availability!                     │
│   ASG distributes instances evenly across selected AZs.        │
│                                                                   │
│ ── Instance type requirements ──                                │
│                                                                   │
│ ○ Use launch template (single instance type)                  │
│ ● Override launch template (RECOMMENDED):                     │
│                                                                   │
│   Instance type selection:                                     │
│   ○ Manually add instance types                                │
│   ● Specify instance type attributes                          │
│     vCPUs: Min [2] Max [8]                                    │
│     Memory: Min [4 GiB] Max [32 GiB]                          │
│     ☐ Require burstable instances                              │
│     Architecture: ☑ arm64 ☑ x86_64                           │
│     ⚡ Attribute-based selection finds all matching types!      │
│     Preview: m7g.large, m7g.xlarge, m6g.large, c7g.large,    │
│              c7g.xlarge, m7i.large, t4g.large, etc.           │
│                                                                   │
│   Instance purchase options:                                    │
│   ● Combine purchase options and instance types (Spot mix)    │
│                                                                   │
│   On-Demand base capacity: [2]                                 │
│   (These 2 instances are ALWAYS On-Demand — your baseline)    │
│                                                                   │
│   On-Demand percentage above base: [30%]                       │
│   Spot percentage above base: [70%]                            │
│   (After base 2: 30% On-Demand, 70% Spot)                    │
│                                                                   │
│   Spot allocation strategy:                                    │
│   ● Price-capacity-optimized (RECOMMENDED)                    │
│     Picks instance types with best availability + price       │
│   ○ Capacity-optimized (focus on availability)                │
│   ○ Lowest-price (cheapest, but more interruptions)           │
│                                                                   │
│   ☑ Capacity rebalance                                        │
│   (Proactively replaces Spot instances at risk of interruption)│
│                                                                   │
│   ⚡ Example with desired=6:                                   │
│   2 On-Demand base + (4 × 30% = 1 OD) + (4 × 70% = 3 Spot)  │
│   = 3 On-Demand + 3 Spot = 6 total                            │
│   ⚡ Save 40-60% on the Spot portion!                          │
│                                                                   │
│ [Next]                                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: CONFIGURE ADVANCED OPTIONS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Load balancing ──                                            │
│ ● Attach to an existing load balancer                          │
│   Choose from your load balancer target groups:                │
│   ☑ tg-web-prod-https (ALB target group)                     │
│   ⚡ ASG auto-registers new instances with the target group!   │
│   ⚡ Terminated instances auto-deregistered!                    │
│                                                                   │
│ ○ Attach to a new load balancer                                │
│ ○ No load balancer                                             │
│                                                                   │
│ VPC Lattice integration options: (advanced — skip for now)    │
│                                                                   │
│ ── Health checks ──                                             │
│ ☑ Turn on Elastic Load Balancing health checks                │
│                                                                   │
│ Health check types:                                             │
│ ├── EC2 (default): AWS hypervisor status check               │
│ │   Detects: Hardware failure, OS crash, network unreachable  │
│ │   ⚠️ Does NOT detect app crashes!                            │
│ │                                                               │
│ ├── ELB: ALB/NLB health check                                │
│ │   Detects: App not responding on health check path          │
│ │   e.g., /health returns 200 OK                              │
│ │   ⚡ ALWAYS enable if using a load balancer!                 │
│ │                                                               │
│ └── Custom: External health check API                         │
│     (Call ASG API to set instance health)                     │
│                                                                   │
│ Health check grace period: [300] seconds                       │
│ ⚡ Time after launch before health checks start.               │
│    Set to your app startup time (e.g., 300s = 5 minutes).     │
│    Too short → ASG terminates healthy instances still booting!│
│    Too long → Unhealthy instances stay around too long.       │
│                                                                   │
│ ── Default instance warmup ──                                   │
│ [300] seconds                                                   │
│ ⚡ Scaling metrics exclude new instances for this period.       │
│    Prevents scaling up again before new instances handle load.│
│                                                                   │
│ ── Monitoring ──                                                │
│ ☑ Enable group metrics collection                             │
│   (GroupDesiredCapacity, GroupInServiceInstances,              │
│    GroupPendingInstances, GroupTerminatingInstances, etc.)     │
│                                                                   │
│ [Next]                                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 4: CONFIGURE GROUP SIZE AND SCALING               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Group size ──                                                │
│ Desired capacity: [4]   ← ASG launches 4 instances now        │
│ Minimum capacity: [2]   ← Never fewer than 2                  │
│ Maximum capacity: [10]  ← Never more than 10                  │
│                                                                   │
│ Desired capacity type: ● Units (instances)                    │
│                        ○ vCPUs                                 │
│                        ○ Memory (GiB)                          │
│ ⚡ Units = count of instances (most common)                    │
│ ⚡ vCPU/Memory useful with mixed instance types               │
│                                                                   │
│ ── Scaling policies ──                                          │
│ ● Target tracking scaling policy                               │
│                                                                   │
│ Policy name: [target-cpu-50]                                   │
│ Metric type: [Average CPU utilization ▼]                      │
│   ● Average CPU utilization                                    │
│   ○ Average network in (bytes)                                │
│   ○ Average network out (bytes)                               │
│   ○ Application Load Balancer request count per target        │
│                                                                   │
│ Target value: [50] %                                           │
│ ⚡ ASG will add/remove instances to keep CPU at ~50%.          │
│ ⚡ Lower target = more aggressive scaling (more instances)     │
│ ⚡ Higher target = less aggressive (fewer instances, riskier)  │
│                                                                   │
│ Instances need: [300] seconds warm up before including in metric│
│                                                                   │
│ ☐ Disable scale in (only scale out, never remove instances)  │
│   ⚠️ Only for special cases — usually leave unchecked!         │
│                                                                   │
│ Additional policies:                                            │
│ [Create a target tracking scaling policy]                      │
│ Policy name: [target-alb-1000]                                 │
│ Metric: ALB request count per target                          │
│ Target: [1000] requests/target                                 │
│ ⚡ Scale based on incoming requests — great for web servers!   │
│                                                                   │
│ ── Instance scale-in protection ──                             │
│ ☐ Enable instance scale-in protection                         │
│   (Protects specific instances from being terminated)         │
│   Use for instances running long batch jobs                   │
│                                                                   │
│ [Next]                                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 5: ADD NOTIFICATIONS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ [Add notification]                                              │
│ SNS Topic: [arn:aws:sns:ap-south-1:123:asg-events ▼]         │
│ Event types:                                                   │
│   ☑ Launch                                                    │
│   ☑ Terminate                                                 │
│   ☑ Fail to launch                                            │
│   ☑ Fail to terminate                                         │
│                                                                   │
│ ⚡ Get Slack/email alerts when instances launch or terminate.   │
│                                                                   │
│ [Next]                                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 6: ADD TAGS                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Tags (applied to ASG AND launched instances):                  │
│ ┌──────────────┬───────────────────┬────────────────────────┐  │
│ │ Key          │ Value             │ Tag new instances       │  │
│ ├──────────────┼───────────────────┼────────────────────────┤  │
│ │ Name         │ web-prod          │ ☑ Yes                  │  │
│ │ Environment  │ prod              │ ☑ Yes                  │  │
│ │ Team         │ backend           │ ☑ Yes                  │  │
│ │ ManagedBy    │ auto-scaling      │ ☑ Yes                  │  │
│ └──────────────┴───────────────────┴────────────────────────┘  │
│                                                                   │
│ [Next] → [Create Auto Scaling group]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Scaling Policies (Deep Dive)

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCALING POLICIES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┬────────────────────────────────────────────┐  │
│ │ Type             │ How it works                               │  │
│ ├──────────────────┼────────────────────────────────────────────┤  │
│ │ Target Tracking  │ "Keep CPU at 50%"                         │  │
│ │ ⚡ RECOMMENDED    │ ASG figures out how many instances needed.│  │
│ │                  │ Like a thermostat. Set target, ASG adjusts.│  │
│ │                  │ Handles both scale-out and scale-in.      │  │
│ │                  │                                            │  │
│ │ Step Scaling     │ "If CPU > 70% add 2, if CPU > 90% add 4" │  │
│ │                  │ Multiple thresholds = different actions.   │  │
│ │                  │ You define CloudWatch alarm + steps.       │  │
│ │                  │ More control but more complex.             │  │
│ │                  │                                            │  │
│ │ Simple Scaling   │ "If CPU > 70% add 1 instance"             │  │
│ │ ⚠️ Legacy         │ Waits for cooldown before next action.    │  │
│ │                  │ Slow. Use Target Tracking or Step instead. │  │
│ │                  │                                            │  │
│ │ Scheduled        │ "At 9AM set desired=8, at 6PM set desired=4"│ │
│ │                  │ Cron-based. For predictable traffic.       │  │
│ │                  │ e.g., Business hours, batch windows.       │  │
│ │                  │                                            │  │
│ │ Predictive       │ ML-based. AWS analyzes historical patterns.│  │
│ │                  │ Pre-scales BEFORE demand hits.             │  │
│ │                  │ Great for daily/weekly traffic patterns.    │  │
│ │                  │ Forecast mode (dry run) or Forecast+Scale.│  │
│ └──────────────────┴────────────────────────────────────────────┘  │
│                                                                       │
│ Target Tracking metrics:                                             │
│ ├── ASGAverageCPUUtilization: Average CPU across all instances  │
│ ├── ASGAverageNetworkIn/Out: Average network traffic            │
│ ├── ALBRequestCountPerTarget: Requests per target on ALB        │
│ └── Custom metric: Your own CloudWatch metric (e.g., queue depth)│
│                                                                       │
│ ⚡ Best combo for web servers:                                       │
│    Target Tracking (CPU 50%) + Target Tracking (ALB requests)    │
│    + Scheduled (morning ramp-up if predictable)                  │
│    + Predictive (auto-learn traffic patterns)                    │
│                                                                       │
│ Multiple policies: ASG evaluates ALL policies, picks the one    │
│ that results in the HIGHEST capacity (most aggressive wins).    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Scheduled Scaling

```
EC2 → Auto Scaling Groups → asg-web-prod → Automatic scaling →
Scheduled actions → Create scheduled action

┌─────────────────────────────────────────────────────────────────┐
│           SCHEDULED ACTION                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [scale-up-morning]                                        │
│ Desired capacity: [8]                                           │
│ Min: [4]                                                         │
│ Max: [12]                                                        │
│ Recurrence: ● Cron                                              │
│   0 9 * * MON-FRI  (9 AM UTC, weekdays)                       │
│ Time zone: [Asia/Kolkata ▼]                                    │
│                                                                   │
│ Name: [scale-down-evening]                                      │
│ Desired capacity: [2]                                           │
│ Min: [2]                                                         │
│ Max: [4]                                                         │
│ Recurrence: 0 18 * * MON-FRI  (6 PM UTC, weekdays)            │
│                                                                   │
│ ⚡ Combine with Target Tracking:                                 │
│    Scheduled sets the baseline, target tracking adjusts         │
│    within the min/max range based on actual demand.             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Predictive Scaling

```
EC2 → Auto Scaling Groups → asg-web-prod → Automatic scaling →
Predictive scaling policies → Create

┌─────────────────────────────────────────────────────────────────┐
│           PREDICTIVE SCALING                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Policy name: [predictive-web-traffic]                           │
│                                                                   │
│ Metrics and target utilization:                                 │
│   ● Pre-defined: ASG CPU utilization                           │
│   ○ Custom: Pair of load + scaling metrics                    │
│   Target utilization: [40] %                                   │
│   (Pre-scale to keep CPU at 40% after new instances arrive)   │
│                                                                   │
│ Scaling mode:                                                   │
│   ○ Forecast only (view predictions, no action)               │
│   ● Forecast and scale (actually pre-scale)                   │
│   ⚡ Start with "Forecast only" for 2 weeks to validate.       │
│   Then switch to "Forecast and scale".                         │
│                                                                   │
│ Pre-launch time: [5] minutes                                   │
│ (Launch instances this many minutes BEFORE predicted demand)  │
│                                                                   │
│ Max capacity behavior:                                          │
│   ● Set to equal max capacity (respect the max)               │
│   ○ Set to value above max capacity (allow burst)             │
│     Upper bound: [15]                                          │
│                                                                   │
│ ⚡ Requires 24 hours of data minimum (2 weeks for accuracy).   │
│                                                                   │
│ View forecast:                                                  │
│ Shows predicted traffic and planned capacity for next 48 hours.│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Lifecycle Hooks

```
┌─────────────────────────────────────────────────────────────────────┐
│           LIFECYCLE HOOKS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Pause instance launch or termination to perform custom      │
│ actions (install software, drain connections, save logs).          │
│                                                                       │
│ Normal lifecycle:                                                    │
│ Pending → InService → Terminating → Terminated                    │
│                                                                       │
│ With hooks:                                                          │
│ Pending → Pending:Wait → (your action) → Pending:Proceed →      │
│ InService → Terminating:Wait → (your action) →                   │
│ Terminating:Proceed → Terminated                                  │
│                                                                       │
│ Launch hook use cases:                                                │
│ ├── Pull configuration from Parameter Store                      │
│ ├── Register with monitoring system                               │
│ ├── Run smoke tests before accepting traffic                     │
│ ├── Install/configure additional software                        │
│ └── Load data into cache                                          │
│                                                                       │
│ Terminate hook use cases:                                            │
│ ├── Drain connections (wait for active requests to finish)       │
│ ├── Deregister from service discovery                             │
│ ├── Save logs to S3 before termination                           │
│ ├── Take final snapshot of instance data                         │
│ └── Notify downstream systems                                    │
│                                                                       │
│ Create:                                                              │
│ ASG → Lifecycle hooks → Create lifecycle hook                     │
│                                                                       │
│ Name: [hook-launch-configure]                                      │
│ Lifecycle transition:                                                │
│   ● Instance launch                                               │
│   ○ Instance terminate                                            │
│ Heartbeat timeout: [300] seconds (max 7200 = 2 hours)            │
│   ⚡ Time your action has to complete.                              │
│   If timeout expires, default result applies.                    │
│ Default result:                                                      │
│   ○ ABANDON (terminate the instance — launch failed)             │
│   ● CONTINUE (put instance in service anyway)                    │
│ Notification target:                                                 │
│   SNS topic or SQS queue (triggers Lambda/EventBridge)           │
│                                                                       │
│ Complete the hook (from your script/Lambda):                       │
│ aws autoscaling complete-lifecycle-action \                        │
│   --lifecycle-hook-name hook-launch-configure \                   │
│   --auto-scaling-group-name asg-web-prod \                        │
│   --instance-id i-0abc123 \                                       │
│   --lifecycle-action-result CONTINUE                              │
│                                                                       │
│ ⚡ Common pattern: EventBridge → Lambda → SSM Run Command →       │
│    Configure instance → Complete lifecycle action                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Instance Refresh (Rolling Updates)

```
┌─────────────────────────────────────────────────────────────────────┐
│           INSTANCE REFRESH                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Replace ALL instances in an ASG with new ones (e.g., after  │
│ updating the AMI or launch template version). Like a rolling      │
│ deployment for EC2.                                                │
│                                                                       │
│ Without Instance Refresh:                                            │
│ ├── Update launch template → only NEW instances use it           │
│ ├── Existing instances still run old AMI/config                   │
│ └── Must manually terminate old instances one by one              │
│                                                                       │
│ With Instance Refresh:                                               │
│ ├── ASG automatically replaces instances in batches               │
│ ├── Respects min healthy percentage (e.g., 90%)                  │
│ ├── Waits for new instances to be healthy before proceeding      │
│ └── Can rollback on failure                                       │
│                                                                       │
│ Start:                                                               │
│ ASG → Instance refresh → Start instance refresh                   │
│                                                                       │
│ Minimum healthy percentage: [90] %                                 │
│ ⚡ 90% means: Replace 10% at a time.                                │
│   With 10 instances → replace 1, wait for healthy, repeat.       │
│                                                                       │
│ Maximum healthy percentage: [120] %                                │
│ ⚡ 120% means: Launch new instances BEFORE terminating old.         │
│   Ensures extra capacity during transition.                       │
│                                                                       │
│ Instance warmup: [300] seconds                                     │
│ (Wait this long after launch before checking health)             │
│                                                                       │
│ Skip matching: ☑ Yes                                               │
│ (Don't replace instances already matching desired config)        │
│                                                                       │
│ Desired configuration:                                               │
│   Launch template: [lt-web-prod ▼]                                │
│   Version: [$Default ▼] (use current default version)            │
│                                                                       │
│ Rollback:                                                            │
│ ├── ☑ Enable auto rollback                                       │
│ ├── CloudWatch alarms for rollback:                               │
│ │   [alarm-5xx-errors ▼]                                         │
│ │   [alarm-high-latency ▼]                                       │
│ └── If alarm triggers → automatically rollback to old config    │
│                                                                       │
│ ⚡ Workflow:                                                         │
│ 1. Build new AMI (Packer)                                         │
│ 2. Create new launch template version with new AMI               │
│ 3. Set as $Default version                                        │
│ 4. Start instance refresh                                         │
│ 5. ASG replaces instances in rolling fashion                     │
│ 6. Monitor — auto rollback if CloudWatch alarm fires             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Warm Pools

```
┌─────────────────────────────────────────────────────────────────────┐
│           WARM POOLS                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Pre-initialized instances in a "stopped" or "running"       │
│ state, ready to be put in service faster than a cold launch.     │
│                                                                       │
│ Problem: Cold launch takes 3-5+ minutes (boot + user data +     │
│ app startup). Too slow for sudden traffic spikes.                │
│                                                                       │
│ Warm pool states:                                                    │
│ ├── Stopped: Instance is stopped (cheapest — no compute charge) │
│ │   Start time: ~1-2 minutes (just boot, already configured)    │
│ ├── Running: Instance is running but not serving traffic         │
│ │   Start time: Seconds (already booted, just attach)            │
│ │   ⚠️ You pay for running instances!                              │
│ └── Hibernated: RAM preserved, fastest resume                    │
│     Start time: ~30 seconds                                      │
│                                                                       │
│ Configure:                                                           │
│ ASG → Warm pool → Create warm pool                                │
│                                                                       │
│ Warm pool instance state: [Stopped ▼]                             │
│ Minimum prepared capacity: [2]                                     │
│   (Always keep 2 instances in warm pool, ready to go)            │
│ Maximum prepared capacity: Defined by ASG max minus in-service  │
│                                                                       │
│ ⚡ Use when apps take >2 minutes to start and you need             │
│    fast scaling response for traffic spikes.                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Termination Policies & AZ Rebalancing

```
┌─────────────────────────────────────────────────────────────────────┐
│           TERMINATION POLICIES                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ When ASG scales in, which instance to terminate?                   │
│                                                                       │
│ Default behavior:                                                    │
│ 1. Find AZ with most instances (rebalance)                        │
│ 2. Within that AZ, apply termination policy:                      │
│                                                                       │
│ ┌──────────────────────┬─────────────────────────────────────────┐ │
│ │ Policy               │ What it does                            │ │
│ ├──────────────────────┼─────────────────────────────────────────┤ │
│ │ Default              │ Oldest launch template → closest to    │ │
│ │ (RECOMMENDED)        │ billing hour                            │ │
│ │ AllocationStrategy   │ Follows Spot/OD allocation strategy     │ │
│ │ OldestLaunchTemplate │ Oldest launch template version first   │ │
│ │ OldestInstance       │ Oldest instance first                   │ │
│ │ NewestInstance       │ Newest instance first                   │ │
│ │ ClosestToNextBilling │ Closest to next billing hour            │ │
│ └──────────────────────┴─────────────────────────────────────────┘ │
│                                                                       │
│ AZ rebalancing:                                                      │
│ ASG keeps instances evenly distributed across AZs.                │
│ If AZ-a has 3 and AZ-b has 1, ASG launches in AZ-b               │
│ and terminates from AZ-a during scale-in.                         │
│                                                                       │
│ Configure:                                                           │
│ ASG → Details → Advanced → Termination policies                  │
│ [Default ▼] [AllocationStrategy ▼] (can stack multiple)          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform Example

```hcl
# Launch Template
resource "aws_launch_template" "web" {
  name          = "lt-web-prod"
  description   = "Web server - Node.js 20, AL2023 ARM"
  image_id      = data.aws_ami.al2023_arm.id
  instance_type = "m7g.large"
  key_name      = aws_key_pair.prod.key_name

  vpc_security_group_ids = [aws_security_group.web.id]

  iam_instance_profile {
    name = aws_iam_instance_profile.web.name
  }

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 2
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 20
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  monitoring {
    enabled = true  # detailed monitoring
  }

  user_data = base64encode(templatefile("userdata.sh", {
    environment = "prod"
    config_bucket = "config-bucket-prod"
  }))

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "web-prod"
      Environment = "prod"
      ManagedBy   = "auto-scaling"
    }
  }

  tag_specifications {
    resource_type = "volume"
    tags = {
      Name        = "web-prod-vol"
      Environment = "prod"
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "web" {
  name                = "asg-web-prod"
  desired_capacity    = 4
  min_size            = 2
  max_size            = 10
  vpc_zone_identifier = [
    aws_subnet.priv_1a.id,
    aws_subnet.priv_1b.id,
    aws_subnet.priv_1c.id,
  ]
  target_group_arns         = [aws_lb_target_group.web.arn]
  health_check_type         = "ELB"
  health_check_grace_period = 300
  default_instance_warmup   = 300
  termination_policies      = ["Default"]

  mixed_instances_policy {
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.web.id
        version            = "$Default"
      }

      override {
        instance_type = "m7g.large"
      }
      override {
        instance_type = "m7g.xlarge"
      }
      override {
        instance_type = "m6g.large"
      }
      override {
        instance_type = "c7g.large"
      }
    }

    instances_distribution {
      on_demand_base_capacity                  = 2
      on_demand_percentage_above_base_capacity = 30
      spot_allocation_strategy                 = "price-capacity-optimized"
    }
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 90
      max_healthy_percentage = 120
      instance_warmup        = 300
      skip_matching          = true
      auto_rollback          = true
      alarm_specification {
        alarms = [aws_cloudwatch_metric_alarm.high_5xx.alarm_name]
      }
    }
  }

  enabled_metrics = [
    "GroupDesiredCapacity",
    "GroupInServiceInstances",
    "GroupPendingInstances",
    "GroupTerminatingInstances",
    "GroupTotalInstances",
  ]

  tag {
    key                 = "Name"
    value               = "web-prod"
    propagate_at_launch = true
  }

  tag {
    key                 = "Environment"
    value               = "prod"
    propagate_at_launch = true
  }
}

# Target Tracking: CPU
resource "aws_autoscaling_policy" "cpu" {
  name                   = "target-cpu-50"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 50.0
  }
}

# Target Tracking: ALB requests
resource "aws_autoscaling_policy" "alb_requests" {
  name                   = "target-alb-1000"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.web.arn_suffix}/${aws_lb_target_group.web.arn_suffix}"
    }
    target_value = 1000.0
  }
}

# Predictive Scaling
resource "aws_autoscaling_policy" "predictive" {
  name                   = "predictive-web"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "PredictiveScaling"

  predictive_scaling_configuration {
    mode                          = "ForecastAndScale"
    scheduling_buffer_time        = 300
    max_capacity_breach_behavior  = "HonorMaxCapacity"

    metric_specification {
      target_value = 50.0
      predefined_scaling_metric_specification {
        predefined_metric_type = "ASGAverageCPUUtilization"
        resource_label         = ""
      }
      predefined_load_metric_specification {
        predefined_metric_type = "ASGTotalCPUUtilization"
        resource_label         = ""
      }
    }
  }
}

# Scheduled actions
resource "aws_autoscaling_schedule" "morning" {
  scheduled_action_name  = "scale-up-morning"
  autoscaling_group_name = aws_autoscaling_group.web.name
  desired_capacity       = 8
  min_size               = 4
  max_size               = 12
  recurrence             = "0 3 * * MON-FRI"  # 9AM IST = 3:30 UTC
  time_zone              = "Asia/Kolkata"
}

resource "aws_autoscaling_schedule" "evening" {
  scheduled_action_name  = "scale-down-evening"
  autoscaling_group_name = aws_autoscaling_group.web.name
  desired_capacity       = 2
  min_size               = 2
  max_size               = 4
  recurrence             = "0 12 * * MON-FRI"  # 6PM IST = 12:30 UTC
  time_zone              = "Asia/Kolkata"
}

# Lifecycle hooks
resource "aws_autoscaling_lifecycle_hook" "launch" {
  name                   = "hook-launch-configure"
  autoscaling_group_name = aws_autoscaling_group.web.name
  lifecycle_transition   = "autoscaling:EC2_INSTANCE_LAUNCHING"
  heartbeat_timeout      = 300
  default_result         = "CONTINUE"
}

resource "aws_autoscaling_lifecycle_hook" "terminate" {
  name                   = "hook-terminate-drain"
  autoscaling_group_name = aws_autoscaling_group.web.name
  lifecycle_transition   = "autoscaling:EC2_INSTANCE_TERMINATING"
  heartbeat_timeout      = 300
  default_result         = "CONTINUE"
}
```

---

## Part 10: CLI Reference

```bash
# Create ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name asg-web-prod \
  --launch-template LaunchTemplateId=lt-0abc123,Version='$Default' \
  --min-size 2 --max-size 10 --desired-capacity 4 \
  --vpc-zone-identifier "subnet-1a,subnet-1b,subnet-1c" \
  --target-group-arns arn:aws:elasticloadbalancing:... \
  --health-check-type ELB \
  --health-check-grace-period 300

# Update ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name asg-web-prod \
  --desired-capacity 6

# Describe ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names asg-web-prod \
  --query 'AutoScalingGroups[0].{Desired:DesiredCapacity,Min:MinSize,Max:MaxSize,Instances:Instances[].InstanceId}' \
  --output table

# Start instance refresh
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name asg-web-prod \
  --preferences '{
    "MinHealthyPercentage": 90,
    "MaxHealthyPercentage": 120,
    "InstanceWarmup": 300,
    "SkipMatching": true
  }'

# Check instance refresh status
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name asg-web-prod \
  --query 'InstanceRefreshes[0].{Status:Status,Progress:PercentageComplete}'

# Manual scale
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name asg-web-prod \
  --desired-capacity 8

# Complete lifecycle hook
aws autoscaling complete-lifecycle-action \
  --lifecycle-hook-name hook-launch-configure \
  --auto-scaling-group-name asg-web-prod \
  --instance-id i-0abc123 \
  --lifecycle-action-result CONTINUE

# Describe scaling activities
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name asg-web-prod \
  --max-items 5 \
  --query 'Activities[].[StatusCode,Description]' \
  --output table
```

---

## Part 11: Real-World Patterns

### Startup

```
ASG: 1 group for web/API tier

Launch template:
├── AMI: Amazon Linux 2023 (arm64)
├── Instance type: t4g.medium
├── User data: Install app, start nginx
└── IAM role: SSM + CloudWatch + S3

ASG config:
├── Min: 1, Desired: 2, Max: 4
├── AZs: 2 AZs (ap-south-1a, ap-south-1b)
├── Target group: ALB target group
├── Health check: ELB (HTTP /health)
├── Scaling: Target tracking CPU 60%
└── All On-Demand (too small for Spot savings)

Deployment: Update AMI → Instance refresh (90% healthy)

Cost: ~$60/month (2x t4g.medium) + scales up on demand
```

### Mid-Size

```
ASGs: 3 separate groups (web, API, workers)

ASG: asg-web-prod
├── Launch template: lt-web-prod (m7g.large, custom AMI)
├── Min: 2, Desired: 4, Max: 12
├── AZs: 3 AZs
├── Mixed fleet: 2 On-Demand base + 70% Spot above
├── Spot types: m7g.large, m6g.large, c7g.large, m7i.large
├── Scaling:
│   ├── Target tracking CPU 50%
│   ├── Target tracking ALB requests 1000/target
│   ├── Scheduled: Scale up 8AM, down 8PM weekdays
│   └── Predictive: Forecast and scale
├── Instance refresh with auto-rollback
├── Lifecycle hooks: Launch (pull config) + Terminate (drain)
└── Warm pool: 2 stopped instances (fast scaling)

ASG: asg-api-prod (similar to web)
ASG: asg-worker-prod
├── Min: 1, Desired: 2, Max: 8
├── 100% Spot (workers are fault-tolerant)
├── Scaling: Custom metric (SQS queue depth)
└── No ALB (pulls work from SQS)

Deployment pipeline:
1. Packer builds new AMI
2. Terraform updates launch template
3. Instance refresh triggered (rolling update)
4. CloudWatch alarm monitors 5xx errors → auto rollback

Cost: ~$400-800/month (Spot + Savings Plans)
```

### Enterprise

```
ASGs: 10+ groups across multiple services

Per-service pattern:
├── Separate ASG per microservice
├── Mixed fleet (4+ instance types for Spot diversity)
├── On-Demand base (2-3 per AZ) + Spot (70% of variable)
├── 3 AZs per region, 2+ regions
├── Health check: ELB + custom (deep health check)
├── Scaling:
│   ├── Target tracking (CPU + ALB requests)
│   ├── Predictive scaling (ML-based)
│   ├── Scheduled (business hours + sales events)
│   └── Custom metrics (queue depth, latency P99)
├── Instance refresh + auto rollback (CloudWatch alarms)
├── Lifecycle hooks: Launch (SSM Run Command), Terminate (drain)
├── Warm pools: 2-4 per ASG (sub-minute scaling)
└── Capacity Reservations for critical On-Demand capacity

Deployment:
├── Blue/green: Two ASGs, switch ALB target groups
├── Or rolling: Instance refresh with 95% min healthy
├── Canary: Weight 5% traffic to new ASG, monitor, promote
└── Auto rollback on: 5xx rate > 1%, P99 latency > 500ms

Monitoring:
├── CloudWatch dashboards per ASG
├── Scaling activity notifications → Slack
├── Spot interruption notifications → EventBridge → Lambda
├── Cost dashboards: On-Demand vs Spot vs Savings Plans
└── Capacity forecast reports

Cost: ~$3,000-30,000/month (50-60% savings vs all On-Demand)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| Launch Template | Defines WHAT to launch (AMI, type, SG, user data, IAM) |
| ASG | Manages fleet, WHICH AZs, min/max/desired capacity |
| Target Tracking | "Keep metric at X" — thermostat-style (RECOMMENDED) |
| Step Scaling | Multiple thresholds → different actions |
| Scheduled | Cron-based, for predictable patterns |
| Predictive | ML-based, pre-scales before demand |
| Health check | EC2 (hypervisor) + ELB (app-level) — use BOTH |
| Grace period | Wait before health checking new instances |
| Lifecycle hooks | Pause launch/terminate for custom actions |
| Instance refresh | Rolling replacement of all instances |
| Warm pools | Pre-initialized instances for fast scaling |
| Termination policy | Which instance to remove on scale-in |
| Mixed fleet | On-Demand base + Spot for savings |
| GCP equivalent | Managed Instance Groups + Autoscaler |
| Azure equivalent | VM Scale Sets (VMSS) |

---

## What's Next?

In the next chapter, we'll cover AWS Lambda — serverless compute (no servers to manage).

→ Next: [Chapter 15: Lambda](15-lambda.md)

---

*Last Updated: May 2026*
