# Chapter 16: AWS ECS - Elastic Container Service

---

## Table of Contents

- [Overview](#overview)
- [Part 1: ECS Fundamentals](#part-1-ecs-fundamentals)
- [Part 2: Creating a Cluster](#part-2-creating-a-cluster)
- [Part 3: Task Definitions (Full Walkthrough)](#part-3-task-definitions-full-walkthrough)
- [Part 4: Creating a Service](#part-4-creating-a-service)
- [Part 5: Deployment Strategies](#part-5-deployment-strategies)
- [Part 6: Service Discovery & Task Networking](#part-6-service-discovery--task-networking)
- [Part 7: Capacity Providers](#part-7-capacity-providers)
- [Part 8: Terraform & CLI Examples](#part-8-terraform--cli-examples)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is ECS? Do I Need to Know Docker First?

> **Real-World Analogy:** If Docker containers are like shipping containers (standardized boxes for your app), then ECS is the port authority — it decides which dock (server) each container goes to, ensures broken containers are replaced, and scales the number of docks based on how many ships (requests) arrive.

**Why does this matter?** Containers are the industry standard for deploying applications. ECS is the simplest way to run containers on AWS — simpler than Kubernetes (EKS). If you're new to containers, start here.

**ECS vs EKS — which do I need?**
- 👉 **ECS**: Simpler, AWS-native, great for teams not already using Kubernetes
- 👉 **EKS**: Use if your team already knows Kubernetes or needs multi-cloud portability

Amazon ECS (Elastic Container Service) is a fully managed container orchestration service for running Docker containers. You define what containers to run, and ECS handles placement, scheduling, and scaling. ECS supports two launch types: **Fargate** (serverless — no servers to manage) and **EC2** (you manage the underlying instances).

```
What you'll learn:
├── ECS Fundamentals
│   ├── Containers & Docker basics (what & why)
│   ├── ECS architecture (Clusters → Services → Tasks)
│   ├── Fargate vs EC2 launch type
│   └── ECS vs EKS (when to use which)
├── Clusters (Full Walkthrough)
│   ├── Cluster creation
│   ├── Fargate cluster
│   ├── EC2 cluster (with ASG capacity provider)
│   └── Cluster settings (Container Insights, namespaces)
├── Task Definitions (Blueprint for Containers)
│   ├── Full creation walkthrough
│   ├── Container definitions (image, ports, env vars)
│   ├── CPU/Memory allocation
│   ├── IAM task role & execution role
│   ├── Volumes (EFS, bind mounts)
│   ├── Logging (CloudWatch, FireLens)
│   └── Task definition revisions
├── Services (Long-Running Containers)
│   ├── Service creation walkthrough
│   ├── Desired count & deployment
│   ├── Load balancer integration
│   ├── Auto scaling (target tracking, step)
│   ├── Deployment strategies (rolling, blue/green)
│   └── Circuit breaker (auto-rollback)
├── Service Discovery (Cloud Map)
├── Task Networking (awsvpc, bridge, host)
├── Capacity Providers
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: ECS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHY CONTAINERS?                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: "Works on my machine" but fails in production.            │
│ Solution: Package app + ALL dependencies into a container.        │
│                                                                       │
│ Container = Your code + runtime + libraries + config               │
│ Runs the same everywhere (laptop, staging, production).           │
│                                                                       │
│ VM vs Container:                                                     │
│ ┌──────────────────────┬────────────────────────────────────────┐  │
│ │ VM                   │ Container                              │  │
│ ├──────────────────────┼────────────────────────────────────────┤  │
│ │ Full OS per VM       │ Shared OS kernel                       │  │
│ │ GB-sized             │ MB-sized                               │  │
│ │ Minutes to start     │ Seconds to start                      │  │
│ │ Heavy resource usage │ Lightweight                            │  │
│ │ Strong isolation     │ Process-level isolation                │  │
│ └──────────────────────┴────────────────────────────────────────┘  │
│                                                                       │
│ Docker: Most popular container runtime.                             │
│ Dockerfile → docker build → Docker Image → docker run → Container│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           ECS ARCHITECTURE                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │                      ECS CLUSTER                             │   │
│ │  (Logical grouping of tasks and services)                   │   │
│ │                                                              │   │
│ │  ┌─────────────────────────────────────────────────────┐   │   │
│ │  │              SERVICE (web-api)                       │   │   │
│ │  │  desired_count: 3                                    │   │   │
│ │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │   │   │
│ │  │  │  TASK 1  │  │  TASK 2  │  │  TASK 3  │         │   │   │
│ │  │  │┌────────┐│  │┌────────┐│  │┌────────┐│         │   │   │
│ │  │  ││Container││  ││Container││  ││Container││         │   │   │
│ │  │  ││ nginx   ││  ││ nginx   ││  ││ nginx   ││         │   │   │
│ │  │  │├────────┤│  │├────────┤│  │├────────┤│         │   │   │
│ │  │  ││Container││  ││Container││  ││Container││         │   │   │
│ │  │  ││ app     ││  ││ app     ││  ││ app     ││         │   │   │
│ │  │  │└────────┘│  │└────────┘│  │└────────┘│         │   │   │
│ │  │  └──────────┘  └──────────┘  └──────────┘         │   │   │
│ │  └─────────────────────────────────────────────────────┘   │   │
│ │                                                              │   │
│ │  Running on: Fargate (serverless) or EC2 instances          │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Hierarchy:                                                           │
│ ├── Cluster: Logical boundary (like a Kubernetes namespace)     │
│ ├── Service: Manages long-running tasks (maintains desired count)│
│ ├── Task: Running instance of a task definition                  │
│ ├── Task Definition: Blueprint (like a docker-compose.yml)      │
│ └── Container Definition: Settings for one container in a task  │
│                                                                       │
│ Key concept:                                                         │
│ ├── Task Definition = WHAT to run (image, CPU, memory, ports)   │
│ ├── Service = HOW to run it (count, LB, scaling, deployment)   │
│ └── Cluster = WHERE to run it (Fargate or EC2 fleet)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           FARGATE vs EC2 LAUNCH TYPE                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────┬─────────────────┬──────────────────────┐  │
│ │ Feature              │ Fargate         │ EC2                  │  │
│ ├──────────────────────┼─────────────────┼──────────────────────┤  │
│ │ Server management    │ None (AWS)      │ You manage instances │  │
│ │ Scaling              │ Task-level      │ Task + Instance level│  │
│ │ Pricing              │ Per vCPU/GB/sec │ EC2 instance pricing │  │
│ │ Cost (small)         │ Cheaper ✅      │ More expensive       │  │
│ │ Cost (large/steady)  │ More expensive  │ Cheaper ✅ (RI/SP)   │  │
│ │ GPU support          │ No ❌           │ Yes ✅               │  │
│ │ Docker privileges    │ Limited         │ Full access          │  │
│ │ Local storage        │ 200 GB ephemeral│ Instance storage     │  │
│ │ Startup time         │ ~30-60s         │ Faster (if warm)     │  │
│ │ Patching             │ AWS patches     │ You patch AMI/OS     │  │
│ │ Spot support         │ Fargate Spot ✅ │ EC2 Spot ✅          │  │
│ │ Daemon tasks         │ No ❌           │ Yes ✅               │  │
│ │ Complexity           │ Simple ✅       │ More complex         │  │
│ └──────────────────────┴─────────────────┴──────────────────────┘  │
│                                                                       │
│ Decision:                                                            │
│ ├── Start with Fargate (simpler, no servers to manage)          │
│ ├── Move to EC2 when: Need GPU, Docker privileges, cost savings│
│ ├── Fargate Spot: 70% cheaper Fargate for fault-tolerant tasks │
│ └── Mix: Fargate for web/API, EC2 for GPU/batch workloads      │
│                                                                       │
│ ── ECS vs EKS ──                                                    │
│ ┌──────────────────────┬─────────────────┬──────────────────────┐  │
│ │ Feature              │ ECS             │ EKS (Kubernetes)     │  │
│ ├──────────────────────┼─────────────────┼──────────────────────┤  │
│ │ Learning curve       │ Low ✅          │ High (K8s knowledge) │  │
│ │ AWS integration      │ Deep (native)   │ Good (add-ons)       │  │
│ │ Portability          │ AWS only        │ Multi-cloud ✅       │  │
│ │ Ecosystem            │ AWS tools       │ Huge K8s ecosystem   │  │
│ │ Control plane cost   │ Free ✅         │ $72/month per cluster│  │
│ │ Best for             │ AWS-native apps │ Multi-cloud, complex │  │
│ └──────────────────────┴─────────────────┴──────────────────────┘  │
│                                                                       │
│ ⚡ If staying AWS-only → ECS (simpler, cheaper)                     │
│ ⚡ If multi-cloud or existing K8s → EKS                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Cluster

```
Console → ECS → Clusters → Create cluster

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE CLUSTER                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cluster name: [prod-cluster]                                       │
│ ⚡ Naming: {env}-cluster or {project}-{env}-cluster                │
│                                                                       │
│ ── Infrastructure ──                                                │
│                                                                       │
│ ☑ AWS Fargate (serverless)                                         │
│   ⚡ No infrastructure to manage. Pay per task.                    │
│                                                                       │
│ ☐ Amazon EC2 instances                                              │
│   (Expand if checked:)                                             │
│   ├── Provisioning model:                                          │
│   │   ● On-Demand  ○ Spot                                        │
│   ├── EC2 instance type: [t3.medium ▼]                           │
│   ├── Desired capacity: Min [1] Max [10]                         │
│   ├── Operating system: [Amazon Linux 2023 ▼]                   │
│   ├── Root EBS volume: [30] GiB (gp3)                            │
│   ├── SSH key pair: [my-key-pair ▼]                              │
│   └── ⚡ Creates an Auto Scaling Group for you                    │
│                                                                       │
│ ── Monitoring ──                                                    │
│                                                                       │
│ ☑ Use Container Insights                                           │
│ ⚡ Enhanced monitoring: Per-task CPU/memory/network/disk metrics. │
│   Costs extra (~$0.30/task/month) but ESSENTIAL for production.  │
│                                                                       │
│ ── Encryption ──                                                    │
│ ☐ AWS Fargate ephemeral storage encryption with KMS key          │
│                                                                       │
│ ── Tags ──                                                          │
│ environment: prod                                                    │
│ team: platform                                                       │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ ⚡ After creation, cluster is empty. Next: Create task definitions │
│    and services.                                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Task Definitions (Full Walkthrough)

```
Console → ECS → Task definitions → Create new task definition

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE TASK DEFINITION                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Task definition family: [web-api]                                  │
│ ⚡ Family name. Each update creates a new REVISION (web-api:1,     │
│   web-api:2, etc.). Like versioning.                              │
│                                                                       │
│ ── Infrastructure requirements ──                                   │
│                                                                       │
│ Launch type:                                                        │
│   ☑ AWS Fargate                                                    │
│   ☐ Amazon EC2 instances                                           │
│                                                                       │
│ Operating system/Architecture: [Linux/X86_64 ▼]                   │
│   ○ Linux/X86_64                                                   │
│   ○ Linux/ARM64 ← 20% cheaper on Fargate                        │
│   ○ Windows/X86_64                                                 │
│                                                                       │
│ Network mode: [awsvpc ▼]                                           │
│   ● awsvpc (each task gets its own ENI + IP — REQUIRED for Fargate)│
│   ○ bridge (Docker bridge networking — EC2 only)                 │
│   ○ host (use host network — EC2 only)                           │
│   ⚡ awsvpc is required for Fargate, recommended for EC2 too.    │
│      Each task gets its own security group & private IP.         │
│                                                                       │
│ Task size:                                                          │
│ CPU: [0.5 vCPU ▼]                                                  │
│   0.25 vCPU │ 0.5 vCPU │ 1 vCPU │ 2 vCPU │ 4 vCPU │ 8 vCPU   │
│   │ 16 vCPU                                                       │
│                                                                       │
│ Memory: [1 GB ▼]                                                   │
│   ⚡ Memory options depend on CPU selected:                        │
│   ┌──────────┬──────────────────────────────────────┐             │
│   │ CPU      │ Memory options                       │             │
│   ├──────────┼──────────────────────────────────────┤             │
│   │ 0.25 vCPU│ 0.5 GB, 1 GB, 2 GB                  │             │
│   │ 0.5 vCPU │ 1 GB, 2 GB, 3 GB, 4 GB              │             │
│   │ 1 vCPU   │ 2 GB, 3 GB, 4 GB, 5-8 GB            │             │
│   │ 2 vCPU   │ 4-16 GB (1 GB increments)            │             │
│   │ 4 vCPU   │ 8-30 GB                              │             │
│   │ 8 vCPU   │ 16-60 GB                             │             │
│   │ 16 vCPU  │ 32-120 GB                            │             │
│   └──────────┴──────────────────────────────────────┘             │
│                                                                       │
│ Ephemeral storage: [21] GiB (20-200 GiB for Fargate)             │
│                                                                       │
│ ── Task role ──                                                     │
│                                                                       │
│ Task role: [ecsTaskRole-web-api ▼]                                │
│ ⚡ IAM role that YOUR CODE assumes (like EC2 instance profile).   │
│   Gives your container access to AWS services.                   │
│   Example: S3 read, DynamoDB read/write, SQS send.              │
│                                                                       │
│ Task execution role: [ecsTaskExecutionRole ▼]                     │
│ ⚡ IAM role that ECS AGENT uses (not your code).                   │
│   Needs: ECR pull, CloudWatch Logs, Secrets Manager access.     │
│   AWS provides default: ecsTaskExecutionRole.                    │
│                                                                       │
│ ⚠️ IMPORTANT: These are TWO different roles!                       │
│   Task role = permissions for YOUR app code                      │
│   Execution role = permissions for ECS infrastructure            │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ CONTAINER DEFINITIONS (the actual containers in the task)         │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ [Container - 1]                                                      │
│                                                                       │
│ Name: [web-api]                                                     │
│ Image URI: [123456789012.dkr.ecr.ap-south-1.amazonaws.com/web-api:latest]│
│ ⚡ From ECR (recommended), Docker Hub, or any registry.            │
│ ⚡ ALWAYS use specific tag, not :latest in production!            │
│   Example: web-api:v2.1.0 or web-api:abc123 (git SHA)           │
│                                                                       │
│ Essential container: ☑ Yes                                        │
│ ⚡ If essential container stops → entire task stops.              │
│   At least one container must be essential.                      │
│                                                                       │
│ ── Port mappings ──                                                 │
│ Container port: [8080]                                              │
│ Protocol: [tcp ▼]                                                  │
│ Port name: [web-api-8080]                                          │
│ App protocol: [http ▼]                                             │
│ ⚡ With awsvpc, container port = host port (no mapping needed).   │
│                                                                       │
│ ── Environment variables ──                                         │
│ Type: Value                                                         │
│ ┌──────────────────┬──────────────────────────────────────────┐   │
│ │ Key              │ Value                                      │   │
│ ├──────────────────┼──────────────────────────────────────────┤   │
│ │ NODE_ENV         │ production                                │   │
│ │ DB_HOST          │ mydb.cluster.rds.amazonaws.com           │   │
│ │ LOG_LEVEL        │ info                                      │   │
│ └──────────────────┴──────────────────────────────────────────┘   │
│                                                                       │
│ Type: ValueFrom (Secrets Manager / SSM Parameter Store)           │
│ ┌──────────────────┬──────────────────────────────────────────┐   │
│ │ Key              │ ValueFrom                                  │   │
│ ├──────────────────┼──────────────────────────────────────────┤   │
│ │ DB_PASSWORD      │ arn:aws:secretsmanager:...:db-password   │   │
│ │ API_KEY          │ arn:aws:ssm:...:parameter/api-key        │   │
│ └──────────────────┴──────────────────────────────────────────┘   │
│ ⚡ Secrets injected at task start. Execution role needs           │
│   secretsmanager:GetSecretValue or ssm:GetParameters.           │
│                                                                       │
│ ── Resource limits (per container) ──                               │
│ CPU: [256] (0.25 vCPU units — 1024 units = 1 vCPU)              │
│ Memory hard limit: [512] MiB (container killed if exceeded)      │
│ Memory soft limit: [256] MiB (reservation, can burst beyond)     │
│ ⚡ Total container resources cannot exceed task-level CPU/memory. │
│                                                                       │
│ ── Health check ──                                                  │
│ Command: ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]│
│ Interval: [30] seconds                                              │
│ Timeout: [5] seconds                                                │
│ Retries: [3]                                                        │
│ Start period: [60] seconds (grace period for startup)             │
│                                                                       │
│ ── Logging ──                                                       │
│ Log driver: [awslogs ▼]                                            │
│ ├── awslogs (CloudWatch Logs — most common)                     │
│ ├── awsfirelens (FluentBit/Fluentd — advanced routing)         │
│ ├── splunk                                                       │
│ └── none                                                          │
│                                                                       │
│ awslogs-group: [/ecs/web-api]                                     │
│ awslogs-region: [ap-south-1]                                      │
│ awslogs-stream-prefix: [ecs]                                      │
│ ⚡ Log stream: ecs/{container-name}/{task-id}                      │
│                                                                       │
│ ── Docker labels ──                                                 │
│ com.mycompany.service: web-api                                     │
│ com.mycompany.version: 2.1.0                                      │
│                                                                       │
│ [+ Add more containers] (sidecar pattern)                          │
│ Example sidecars:                                                    │
│ ├── Envoy proxy (service mesh)                                  │
│ ├── Log router (FluentBit)                                      │
│ ├── Datadog agent (monitoring)                                  │
│ └── X-Ray daemon (tracing)                                      │
│                                                                       │
│ ── Storage (Volumes) ──                                             │
│                                                                       │
│ [+ Add volume]                                                      │
│ Volume type:                                                        │
│ ├── Bind mount (share between containers in same task)          │
│ ├── EFS (Elastic File System — persistent, shared across tasks)│
│ └── Docker volume                                                │
│                                                                       │
│ EFS volume (persistent storage):                                   │
│ Name: [shared-data]                                                 │
│ EFS file system ID: [fs-12345678]                                  │
│ Root directory: [/app-data]                                        │
│ Transit encryption: ☑ (encrypt in transit)                        │
│ IAM authorization: ☑ (use task role for EFS access)              │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Creating a Service

```
Console → ECS → Cluster → prod-cluster → Services → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE SERVICE                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Compute configuration ──                                         │
│                                                                       │
│ Compute options:                                                    │
│   ● Capacity provider strategy (recommended)                     │
│   ○ Launch type                                                   │
│                                                                       │
│ Capacity provider strategy:                                        │
│ ├── FARGATE: Weight [1], Base [2]                                │
│ ├── FARGATE_SPOT: Weight [3], Base [0]                           │
│ └── ⚡ Base 2 = always 2 tasks on regular Fargate.               │
│     Weight 1:3 = additional tasks 25% Fargate, 75% Fargate Spot│
│     → Saves ~70% on scaling tasks!                               │
│                                                                       │
│ ── Deployment configuration ──                                      │
│                                                                       │
│ Application type:                                                   │
│   ● Service (long-running, maintains desired count)              │
│   ○ Task (one-off, run to completion)                            │
│                                                                       │
│ Task definition:                                                    │
│ Family: [web-api ▼]                                                │
│ Revision: [LATEST ▼] or specific [3]                              │
│ ⚡ LATEST = always use newest revision.                             │
│   Production: Consider pinning to specific revision.             │
│                                                                       │
│ Service name: [web-api-service]                                    │
│                                                                       │
│ Service type:                                                       │
│   ● Replica (run N copies across cluster — most common)         │
│   ○ Daemon (one task per EC2 instance — EC2 only, monitoring)   │
│                                                                       │
│ Desired tasks: [3]                                                  │
│ ⚡ ECS maintains exactly 3 running tasks.                          │
│   If a task dies → ECS automatically launches replacement.      │
│                                                                       │
│ ── Deployment options ──                                            │
│                                                                       │
│ Min running tasks %: [100]                                         │
│ Max running tasks %: [200]                                         │
│ ⚡ 100/200 = Rolling update: keeps 100% capacity, creates up to  │
│   200% during deployment, then drains old tasks.                 │
│   With 3 desired: min 3, max 6 during deploy.                   │
│                                                                       │
│ Deployment failure detection:                                      │
│ ☑ Use the Amazon ECS deployment circuit breaker                  │
│ ☑ Rollback on failures                                            │
│ ⚡ If new tasks keep failing → auto-rollback to previous version!│
│   ALWAYS enable this for production.                             │
│                                                                       │
│ ── Networking ──                                                    │
│                                                                       │
│ VPC: [vpc-prod ▼]                                                  │
│ Subnets: [private-1a, private-1b, private-1c ▼]                 │
│ ⚡ Use PRIVATE subnets (tasks behind load balancer).              │
│                                                                       │
│ Security group: [sg-ecs-web-api ▼]  [Create new]                 │
│ ⚡ Allow:                                                           │
│   Inbound: Port 8080 from ALB security group only               │
│   Outbound: All (for ECR pull, AWS APIs, external calls)        │
│                                                                       │
│ Public IP: ● Turned off                                            │
│ ⚡ No public IP. Traffic comes through load balancer.             │
│ ⚠️ For Fargate in PRIVATE subnet: Need NAT Gateway for ECR pull │
│   (or use VPC endpoints for ECR, Logs, S3, Secrets Manager).   │
│                                                                       │
│ ── Load balancing ──                                                │
│                                                                       │
│ Load balancer type:                                                 │
│   ● Application Load Balancer (HTTP/HTTPS — most common)       │
│   ○ Network Load Balancer (TCP/UDP)                              │
│   ○ None                                                          │
│                                                                       │
│ Load balancer: [alb-prod ▼]  or [Create new ALB]                 │
│                                                                       │
│ Listener: [443:HTTPS ▼]                                           │
│ Target group: [tg-web-api ▼]  or [Create new]                    │
│                                                                       │
│ New target group:                                                   │
│ Name: [tg-web-api]                                                  │
│ Protocol: HTTP                                                       │
│ Port: 8080                                                           │
│ Target type: IP (required for awsvpc/Fargate)                     │
│ Health check path: [/health]                                       │
│ Health check grace period: [60] seconds                            │
│ ⚡ Grace period: Time before ECS starts checking LB health.       │
│   Set >= app startup time to avoid killing starting tasks!       │
│                                                                       │
│ ── Service Auto Scaling ──                                          │
│                                                                       │
│ ☑ Enable auto scaling                                              │
│ Minimum tasks: [2]                                                  │
│ Maximum tasks: [10]                                                 │
│ Desired tasks: [3]                                                  │
│                                                                       │
│ Scaling policy type:                                                │
│   ● Target tracking (recommended)                                 │
│   ○ Step scaling                                                   │
│                                                                       │
│ Target tracking policies:                                           │
│ ├── ECSServiceAverageCPUUtilization: [60]%                      │
│ ├── ECSServiceAverageMemoryUtilization: [70]%                   │
│ ├── ALBRequestCountPerTarget: [1000] requests/target            │
│ └── Scale-in cooldown: [300]s  Scale-out cooldown: [60]s        │
│                                                                       │
│ ── Service discovery (optional) ──                                  │
│                                                                       │
│ ☑ Use service discovery                                            │
│ Namespace: [prod.local ▼]  (Cloud Map namespace)                 │
│ Service name: [web-api]                                             │
│ DNS record type: [A ▼]                                             │
│ TTL: [10] seconds                                                   │
│ ⚡ Other services can reach this as: web-api.prod.local           │
│   No load balancer needed for service-to-service communication! │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Deployment Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEPLOYMENT STRATEGIES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Rolling Update (default — ECS native):                           │
│    ├── Gradually replace old tasks with new tasks                │
│    ├── Min healthy: 100%, Max: 200%                              │
│    ├── Zero-downtime (old tasks serve while new tasks start)    │
│    ├── Circuit breaker: Auto-rollback if new tasks keep failing │
│    └── ⚡ Good for most deployments. Simple and effective.        │
│                                                                       │
│    Timeline:                                                        │
│    [v1][v1][v1] → [v1][v1][v1][v2][v2][v2] → [v2][v2][v2]     │
│                    ↑ Both running              ↑ Old drained     │
│                                                                       │
│ 2. Blue/Green (via CodeDeploy):                                    │
│    ├── Creates entirely new set of tasks (green)                │
│    ├── Tests green → shifts traffic → terminates blue           │
│    ├── Instant rollback: Shift traffic back to blue             │
│    ├── Options: All-at-once, linear (10% per min), canary (10%)│
│    └── ⚡ Safest for critical production services.               │
│                                                                       │
│    Timeline:                                                        │
│    [v1][v1][v1] (blue - 100% traffic)                           │
│    [v1][v1][v1] + [v2][v2][v2] (green created, 0% traffic)    │
│    [v1][v1][v1] + [v2][v2][v2] (shift: 10% to green)          │
│    [v2][v2][v2] (100% green, blue terminated)                  │
│                                                                       │
│    Blue/Green requires:                                            │
│    ├── CodeDeploy deployment group                               │
│    ├── Two target groups on ALB (blue + green)                  │
│    ├── Deployment config: AllAtOnce / Linear10Percent / Canary10│
│    └── Hooks: BeforeAllowTraffic, AfterAllowTraffic (test!)    │
│                                                                       │
│ Circuit breaker (auto-rollback):                                   │
│ ├── ECS tracks deployment health                                │
│ ├── If new tasks fail health checks repeatedly → rollback      │
│ ├── Reverts to previous task definition revision                │
│ └── ⚡ ALWAYS enable for production services                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Service Discovery & Task Networking

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVICE DISCOVERY (AWS Cloud Map)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: DNS-based service discovery for ECS tasks.                   │
│                                                                       │
│ Without service discovery:                                          │
│ ├── Service A needs to call Service B                            │
│ ├── Put Service B behind internal ALB ($18/month + processing) │
│ ├── Service A calls ALB DNS name                                │
│ └── Works but costs money for every service-to-service call    │
│                                                                       │
│ With service discovery (Cloud Map):                                │
│ ├── Service B registers as: order-service.prod.local            │
│ ├── Service A calls: http://order-service.prod.local:8080      │
│ ├── DNS resolves to task IPs directly                           │
│ ├── No load balancer needed for internal communication!        │
│ └── Cost: ~free (just DNS queries)                              │
│                                                                       │
│ Setup:                                                               │
│ 1. Create Cloud Map namespace: prod.local (private DNS)          │
│ 2. Enable service discovery on ECS service                       │
│ 3. Tasks auto-register/deregister                                │
│ 4. Other services discover via DNS: servicename.prod.local      │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────┐      │
│ │ ECS Cluster                                              │      │
│ │                                                          │      │
│ │ web-api.prod.local → [Task A1] [Task A2] [Task A3]    │      │
│ │        ↑                                                 │      │
│ │        │ DNS lookup                                     │      │
│ │        │                                                 │      │
│ │ order-service.prod.local → [Task B1] [Task B2]        │      │
│ │        ↑                                                 │      │
│ │        │ DNS lookup                                     │      │
│ │        │                                                 │      │
│ │ payment-service.prod.local → [Task C1] [Task C2]     │      │
│ └──────────────────────────────────────────────────────────┘      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TASK NETWORKING MODES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ awsvpc (Fargate default, EC2 recommended):                         │
│ ├── Each task gets its own ENI (network interface)              │
│ ├── Own private IP address in your VPC subnet                   │
│ ├── Own security group per task                                  │
│ ├── ⚡ Best isolation, like a mini-VM networking                  │
│ └── Required for Fargate                                         │
│                                                                       │
│ bridge (EC2 only):                                                  │
│ ├── Docker bridge network on EC2 host                           │
│ ├── Dynamic port mapping (host:random → container:8080)        │
│ ├── Multiple tasks share host network via port mapping          │
│ └── Legacy — use awsvpc instead                                  │
│                                                                       │
│ host (EC2 only):                                                    │
│ ├── Task uses EC2 host's network directly                      │
│ ├── Best network performance (no NAT overhead)                  │
│ ├── Only one task per port per host                             │
│ └── Use for: High-performance, low-latency needs              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Capacity Providers

```
┌─────────────────────────────────────────────────────────────────────┐
│           CAPACITY PROVIDERS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Define WHERE tasks run and manage scaling of infrastructure. │
│                                                                       │
│ Built-in providers:                                                  │
│ ├── FARGATE: Regular Fargate (serverless)                       │
│ ├── FARGATE_SPOT: Spot pricing (~70% cheaper, can be interrupted)│
│ └── Custom: Your Auto Scaling Group (EC2 instances)             │
│                                                                       │
│ Strategy (mix providers):                                           │
│ ┌──────────────────────────────────────────────────────────┐      │
│ │ Strategy: FARGATE (base=2, weight=1),                    │      │
│ │           FARGATE_SPOT (base=0, weight=3)                │      │
│ │                                                          │      │
│ │ Result (10 tasks):                                       │      │
│ │ FARGATE: 2 (base) + 2 (weight 1/4 of remaining 8) = 4  │      │
│ │ FARGATE_SPOT: 0 (base) + 6 (weight 3/4 of remaining 8) = 6│    │
│ │                                                          │      │
│ │ → 4 stable + 6 spot = significant savings!              │      │
│ └──────────────────────────────────────────────────────────┘      │
│                                                                       │
│ EC2 Capacity Provider (with managed scaling):                      │
│ ├── Associates ASG with ECS cluster                             │
│ ├── ECS auto-scales EC2 instances when tasks need capacity     │
│ ├── Target capacity %: 100% (scale EC2 to match task demand)  │
│ ├── Managed termination protection (don't kill running tasks) │
│ └── ⚡ ECS manages the ASG for you — just define task count     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform & CLI Examples

### Terraform

```hcl
# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "prod-cluster"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
  
  tags = {
    environment = "prod"
  }
}

# Cluster capacity providers
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name
  
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
  
  default_capacity_provider_strategy {
    base              = 2
    weight            = 1
    capacity_provider = "FARGATE"
  }
  
  default_capacity_provider_strategy {
    base              = 0
    weight            = 3
    capacity_provider = "FARGATE_SPOT"
  }
}

# Task execution role (for ECS agent)
resource "aws_iam_role" "ecs_execution" {
  name = "ecsTaskExecutionRole"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Task role (for your application code)
resource "aws_iam_role" "task_role" {
  name = "ecsTaskRole-web-api"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "task_s3" {
  name = "s3-access"
  role = aws_iam_role.task_role.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "${aws_s3_bucket.uploads.arn}/*"
    }]
  })
}

# Task definition
resource "aws_ecs_task_definition" "web_api" {
  family                   = "web-api"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 512    # 0.5 vCPU
  memory                   = 1024   # 1 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.task_role.arn
  
  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "ARM64"   # 20% cheaper
  }
  
  container_definitions = jsonencode([
    {
      name      = "web-api"
      image     = "${aws_ecr_repository.web_api.repository_url}:v2.1.0"
      essential = true
      
      portMappings = [{
        containerPort = 8080
        protocol      = "tcp"
        appProtocol   = "http"
      }]
      
      environment = [
        { name = "NODE_ENV", value = "production" },
        { name = "PORT", value = "8080" },
      ]
      
      secrets = [
        {
          name      = "DB_PASSWORD"
          valueFrom = aws_secretsmanager_secret.db_password.arn
        },
      ]
      
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/web-api"
          "awslogs-region"        = "ap-south-1"
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
}

# CloudWatch log group
resource "aws_cloudwatch_log_group" "web_api" {
  name              = "/ecs/web-api"
  retention_in_days = 30
}

# Security group for ECS tasks
resource "aws_security_group" "ecs_tasks" {
  name        = "sg-ecs-web-api"
  description = "Allow traffic from ALB to ECS tasks"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ECS Service
resource "aws_ecs_service" "web_api" {
  name            = "web-api-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.web_api.arn
  desired_count   = 3
  
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    base              = 2
    weight            = 1
  }
  
  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    base              = 0
    weight            = 3
  }
  
  network_configuration {
    subnets          = [aws_subnet.private_1.id, aws_subnet.private_2.id]
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.web_api.arn
    container_name   = "web-api"
    container_port   = 8080
  }
  
  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
  
  deployment_minimum_healthy_percent = 100
  deployment_maximum_percent         = 200
  health_check_grace_period_seconds  = 60
  
  service_registries {
    registry_arn = aws_service_discovery_service.web_api.arn
  }
}

# Auto scaling
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.web_api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "cpu-target-tracking"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 60.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

resource "aws_appautoscaling_policy" "requests" {
  name               = "alb-request-count"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.main.arn_suffix}/${aws_lb_target_group.web_api.arn_suffix}"
    }
    target_value       = 1000
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# Service Discovery
resource "aws_service_discovery_private_dns_namespace" "main" {
  name = "prod.local"
  vpc  = aws_vpc.main.id
}

resource "aws_service_discovery_service" "web_api" {
  name = "web-api"
  
  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id
    
    dns_records {
      ttl  = 10
      type = "A"
    }
    
    routing_policy = "MULTIVALUE"
  }
  
  health_check_custom_config {
    failure_threshold = 1
  }
}
```

### AWS CLI

```bash
# Create cluster
aws ecs create-cluster \
  --cluster-name prod-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,base=2,weight=1 \
    capacityProvider=FARGATE_SPOT,base=0,weight=3 \
  --settings name=containerInsights,value=enabled

# Register task definition
aws ecs register-task-definition --cli-input-json file://task-def.json

# Create service
aws ecs create-service \
  --cluster prod-cluster \
  --service-name web-api-service \
  --task-definition web-api:3 \
  --desired-count 3 \
  --capacity-provider-strategy \
    capacityProvider=FARGATE,base=2,weight=1 \
    capacityProvider=FARGATE_SPOT,base=0,weight=3 \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc,subnet-def],securityGroups=[sg-123],assignPublicIp=DISABLED}" \
  --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:...:tg-web-api,containerName=web-api,containerPort=8080 \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}" \
  --health-check-grace-period-seconds 60

# Update service (deploy new task definition revision)
aws ecs update-service \
  --cluster prod-cluster \
  --service web-api-service \
  --task-definition web-api:4 \
  --force-new-deployment

# Scale service
aws ecs update-service \
  --cluster prod-cluster \
  --service web-api-service \
  --desired-count 5

# List services
aws ecs list-services --cluster prod-cluster

# Describe service
aws ecs describe-services \
  --cluster prod-cluster \
  --services web-api-service \
  --query "services[0].{status:status,desired:desiredCount,running:runningCount,pending:pendingCount}" \
  --output table

# List tasks
aws ecs list-tasks --cluster prod-cluster --service-name web-api-service

# Execute command (debug into running container)
aws ecs execute-command \
  --cluster prod-cluster \
  --task abc123 \
  --container web-api \
  --interactive \
  --command "/bin/sh"

# View task logs
aws logs tail /ecs/web-api --since 1h --follow

# Stop task
aws ecs stop-task --cluster prod-cluster --task abc123

# Deregister task definition
aws ecs deregister-task-definition --task-definition web-api:2
```

---

## Part 9: Real-World Patterns

### Startup

```
Architecture: Single service on Fargate

Cluster: prod-cluster (Fargate only)

Service: web-api-service
├── Task def: web-api (1 container)
│   ├── Image: ECR (web-api:v1.0.0)
│   ├── CPU: 0.5 vCPU, Memory: 1 GB
│   ├── Port: 8080
│   ├── Secrets from Secrets Manager
│   └── Logs to CloudWatch
├── Desired: 2 tasks
├── Behind ALB (HTTPS, ACM certificate)
├── Auto scaling: 2-6 tasks, CPU 70%
├── Circuit breaker: enabled
└── Private subnets + NAT Gateway

Deploy:
├── Build image → push to ECR → update task def → update service
├── GitHub Actions CI/CD pipeline
├── Rolling update (100/200)
└── Rollback: Update service to previous task def revision

Cost: 2× Fargate (0.5 vCPU, 1 GB) = ~$30/month
      + ALB ~$18/month + NAT ~$35/month
      Total: ~$85/month
```

### Mid-Size

```
Architecture: Multiple microservices on Fargate + Spot

Cluster: prod-cluster

Services:
├── web-api (public-facing API)
│   ├── 3-10 tasks, CPU 1 vCPU, 2 GB
│   ├── Behind public ALB
│   ├── Fargate (base 2) + Fargate Spot (weight 3)
│   ├── Auto scaling: CPU 60% + ALB request count 1000
│   └── Blue/green deployment (CodeDeploy)
├── order-service (internal)
│   ├── 2-6 tasks, CPU 0.5 vCPU, 1 GB
│   ├── Service discovery: order-service.prod.local
│   └── No load balancer (other services call via DNS)
├── payment-service (internal)
│   ├── 2-4 tasks, CPU 0.5 vCPU, 1 GB
│   ├── Service discovery: payment-service.prod.local
│   └── Fargate only (no Spot for payments!)
├── worker-service (background jobs)
│   ├── 1-8 tasks, Fargate Spot 100%
│   ├── Processes SQS messages
│   └── Auto scaling: SQS queue depth
└── notification-service (async)
    ├── 1-3 tasks, Fargate Spot
    ├── Triggered by SNS/EventBridge
    └── Service discovery

Shared infra:
├── VPC endpoints: ECR, S3, Logs, Secrets Manager
│   (Eliminates NAT Gateway cost for AWS API calls!)
├── Cloud Map namespace: prod.local
├── Container Insights enabled
├── Centralized logging: CloudWatch → OpenSearch
└── X-Ray distributed tracing

Cost: ~$400-800/month (with Spot savings)
```

### Enterprise

```
Architecture: Multi-account, multi-region ECS platform

Accounts:
├── Platform account: Shared ECR, CI/CD pipelines
├── Production account: Prod ECS clusters
├── Staging account: Stage ECS clusters
└── Dev account: Dev ECS clusters

Per-region cluster:
├── prod-cluster-india (ap-south-1)
├── prod-cluster-us (us-east-1)
└── prod-cluster-eu (eu-west-1)

Services per cluster (30-50 services):
├── Public tier: Behind ALB with WAF
├── Internal tier: Service discovery (Cloud Map)
├── Worker tier: Fargate Spot, SQS-triggered
└── Scheduled: EventBridge → Fargate tasks (cron)

Deployment:
├── CI/CD: CodePipeline → CodeBuild → ECR → CodeDeploy
├── Blue/Green for public services (canary 10%)
├── Rolling update for internal services
├── Cross-region: Deploy India → canary 1h → deploy US → deploy EU
├── Image scanning: ECR image scan on push
└── Policy: Only signed images can deploy (Sigstore/Cosign)

Capacity:
├── Base: Fargate (guaranteed capacity)
├── Burst: Fargate Spot (70% savings)
├── EC2 capacity provider: GPU tasks (ML inference)
└── Mixed: base=5 FARGATE + weight=4 FARGATE_SPOT

Networking:
├── VPC endpoints (no NAT for AWS services)
├── PrivateLink for cross-account/cross-VPC
├── Service mesh: App Mesh or Envoy sidecars
├── mTLS between services
└── Network policies via security groups

Monitoring:
├── Container Insights (CloudWatch)
├── X-Ray (distributed tracing)
├── Datadog/New Relic APM
├── Prometheus + Grafana (custom metrics)
├── Alerts: Service health, deployment failures, scaling events
└── Dashboards per team/service

Security:
├── ECR: Image scanning, lifecycle policies
├── Task roles: Least privilege per service
├── Secrets Manager: Auto-rotation
├── Runtime protection: GuardDuty for ECS
├── No SSH/exec in production (audit trail via CloudTrail)
└── Compliance: SOC2, HIPAA task isolation

Cost: $5,000-25,000/month (with Spot + Savings Plans)
```

---

## Troubleshooting: Common ECS Issues

### "Service can't reach desired count" (tasks keep failing)

```
1. ☐ Check Stopped Tasks for error messages:
   Console → ECS → Cluster → Service → Tasks → Filter: Stopped
   Click the stopped task → check "Stopped reason"

2. Common reasons:
   ├── CannotPullContainerError: Wrong image URI or missing ECR permissions
   ├── OutOfMemoryError: Container exceeds memory limit
   ├── Essential container exited: App crashed (check CloudWatch logs)
   ├── ResourceNotFoundException: Task definition doesn't exist
   └── HealthCheck failed: ALB health check failing
```

### "Task Role vs Execution Role" — Which Do I Need?

```
┌──────────────────────────────────────────────────┐
│ Execution Role = ECS infrastructure tasks            │
│ └─ Pull images from ECR                              │
│ └─ Send logs to CloudWatch Logs                     │
│ └─ Pull secrets from Secrets Manager/SSM            │
│                                                      │
│ Task Role = YOUR application permissions             │
│ └─ Access S3 buckets, DynamoDB tables                │
│ └─ Publish to SNS/SQS                               │
│ └─ Call any AWS API your code uses                   │
└──────────────────────────────────────────────────┘
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Missing execution role | Can't pull images or send logs | Attach AmazonECSTaskExecutionRolePolicy |
| Wrong networking mode | Containers can't communicate | Use awsvpc for Fargate, bridge for EC2 |
| No health check grace period | Tasks killed before app starts | Set health check grace period (60-120s) |
| Logging not configured | Can't debug container crashes | Add awslogs driver in task definition |
| Container port mismatch | ALB can't reach container | Ensure container port matches target group port |

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Managed container orchestration (Docker) |
| Fargate | Serverless — no servers to manage |
| EC2 | You manage underlying instances |
| Cluster | Logical grouping of services/tasks |
| Task Definition | Blueprint (image, CPU, memory, ports, env, IAM) |
| Service | Maintains desired count of running tasks |
| Task role | IAM role for YOUR code |
| Execution role | IAM role for ECS agent (ECR pull, logs) |
| awsvpc | Each task gets own ENI/IP/security group |
| Service Discovery | Cloud Map DNS (service.namespace.local) |
| Rolling update | Gradual replacement, min/max healthy % |
| Blue/Green | CodeDeploy, instant rollback, canary |
| Circuit breaker | Auto-rollback on deployment failures |
| Capacity providers | FARGATE, FARGATE_SPOT, EC2 ASG |
| GCP equivalent | Cloud Run / GKE |
| Azure equivalent | ACI / AKS |

---

## What's Next?

In the next chapter, we'll cover Amazon EKS — Elastic Kubernetes Service.

→ Next: [Chapter 17: EKS - Elastic Kubernetes Service](17-eks.md)

---

*Last Updated: May 2026*
