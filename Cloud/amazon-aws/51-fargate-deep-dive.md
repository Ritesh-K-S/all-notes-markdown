# Chapter 51: AWS Fargate Deep Dive

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Fargate Fundamentals](#part-1-fargate-fundamentals)
- [Part 2: Fargate with ECS (Full Portal Walkthrough)](#part-2-fargate-with-ecs-full-portal-walkthrough)
- [Part 3: Fargate with EKS](#part-3-fargate-with-eks)
- [Part 4: Networking, Storage & Security](#part-4-networking-storage--security)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Fargate? Why Serverless Containers?

With regular ECS/EKS on EC2, you need to:
1. Choose EC2 instance types
2. Manage the EC2 fleet (patching, scaling, replacing failed instances)
3. Figure out how many containers fit on each instance
4. Pay for EC2 instances even when containers aren't fully using them

**Fargate removes all of this.** Think of it like the difference between:
- **EC2 launch type** = Renting an apartment (you choose the size, you manage it)
- **Fargate** = Using a hotel (just say how many rooms you need, hotel manages everything)

With Fargate, you just say: "Run this container with 1 vCPU and 2 GB memory" — AWS handles all the infrastructure. You pay only for the exact CPU and memory your containers use.

**When to use Fargate vs EC2 launch type:**
- **Fargate**: Small/medium workloads, variable traffic, teams that don't want to manage servers
- **EC2**: Large stable workloads (cheaper with Reserved Instances), GPU needs, or very specific instance requirements

AWS Fargate is a serverless compute engine for containers. You run containers without managing EC2 instances. It works with both ECS and EKS.

```
What you'll learn:
├── Fargate fundamentals (serverless containers)
├── Fargate with ECS (task definition walkthrough)
├── Fargate with EKS (pod configuration)
├── Networking (awsvpc mode, ENI per task)
├── Storage (ephemeral, EFS)
├── Security (task roles, execution roles)
└── Terraform & CLI examples
```

---

## Part 1: Fargate Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW FARGATE WORKS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Without Fargate (EC2 launch type):                                  │
│ You manage: EC2 instances, OS patching, capacity, scaling          │
│                                                                       │
│ With Fargate:                                                        │
│ You manage: Task/pod definition (CPU, memory, image)               │
│ AWS manages: Everything else (servers, OS, scaling, patching)     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────┐       │
│ │  You define:                                              │       │
│ │  ├── Container image (ECR/Docker Hub)                   │       │
│ │  ├── CPU + Memory (predefined combos)                   │       │
│ │  ├── Networking (VPC, subnets, SG)                     │       │
│ │  ├── IAM roles (task role, execution role)             │       │
│ │  └── Environment variables, secrets                     │       │
│ │                                                           │       │
│ │  AWS manages:                                             │       │
│ │  ├── Server provisioning                                 │       │
│ │  ├── Cluster capacity                                    │       │
│ │  ├── OS patching                                         │       │
│ │  ├── Container runtime                                   │       │
│ │  └── Infrastructure scaling                              │       │
│ └──────────────────────────────────────────────────────────┘       │
│                                                                       │
│ EC2 vs Fargate:                                                      │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ EC2           │ Fargate              │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Server management    │ You manage    │ AWS manages          │   │
│ │ Pricing              │ Per EC2 inst. │ Per vCPU + GB/hour   │   │
│ │ Cost                 │ Cheaper at    │ Cheaper at small     │   │
│ │                      │ scale         │ scale / variable     │   │
│ │ Scaling              │ Manage ASG    │ Automatic            │   │
│ │ GPU support          │ ✅            │ ❌                   │   │
│ │ Spot pricing         │ ✅ (Spot EC2) │ ✅ (Fargate Spot)   │   │
│ │ Persistent storage   │ EBS + EFS     │ EFS only             │   │
│ │ Host access          │ ✅ (SSH)      │ ❌                   │   │
│ │ Startup time         │ Minutes       │ 30-60 seconds        │   │
│ │ ⚡ Best for          │ Stable, large │ Variable, serverless │   │
│ │                      │ workloads     │ workloads            │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│                                                                       │
│ Fargate CPU/Memory combinations:                                    │
│ ┌────────────────┬─────────────────────────────────────┐          │
│ │ vCPU           │ Memory (GB)                          │          │
│ ├────────────────┼─────────────────────────────────────┤          │
│ │ 0.25           │ 0.5, 1, 2                            │          │
│ │ 0.5            │ 1, 2, 3, 4                           │          │
│ │ 1              │ 2, 3, 4, 5, 6, 7, 8                 │          │
│ │ 2              │ 4-16 (1 GB increments)              │          │
│ │ 4              │ 8-30 (1 GB increments)              │          │
│ │ 8              │ 16-60 (4 GB increments)             │          │
│ │ 16             │ 32-120 (8 GB increments)            │          │
│ └────────────────┴─────────────────────────────────────┘          │
│                                                                       │
│ Pricing (us-east-1):                                                │
│ ├── Linux: $0.04048/vCPU/hour + $0.004445/GB/hour               │
│ ├── Fargate Spot: Up to 70% discount                             │
│ └── Savings Plans: Up to 50% (1 or 3-year commitment)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Fargate with ECS (Full Portal Walkthrough)

```
Console → ECS → Task definitions → Create new task definition

┌─────────────────────────────────────────────────────────────────┐
│           TASK DEFINITION (FARGATE)                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Task definition family: [web-app-task]                        │
│                                                                   │
│ Infrastructure requirements:                                   │
│                                                                   │
│ Launch type: ● AWS Fargate  ○ Amazon EC2                     │
│                                                                   │
│ Operating system/Architecture:                                │
│ [Linux/X86_64 ▼]                                             │
│ ├── Linux/X86_64 (most common)                               │
│ ├── Linux/ARM64 (⚡ 20% cheaper, Graviton)                  │
│ └── Windows/X86_64                                            │
│                                                                   │
│ Task size:                                                     │
│ CPU: [1 vCPU ▼]                                               │
│ Memory: [2 GB ▼]                                              │
│ → ⚡ ARM64 provides better price/performance                 │
│                                                                   │
│ Task role: [ecsTaskRole ▼]                                   │
│ → IAM role for your containers (access AWS services)        │
│ → e.g., DynamoDB access, S3 access                          │
│                                                                   │
│ Task execution role: [ecsTaskExecutionRole ▼]                │
│ → IAM role for ECS agent (pull images, write logs)          │
│ → ⚡ Use AWS managed policy: AmazonECSTaskExecutionRolePolicy│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           CONTAINER DEFINITION                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Container - 1:                                                 │
│ Name: [web-server]                                            │
│ Image URI: [123456.dkr.ecr.us-east-1.amazonaws.com/web:v1.0]│
│ → ECR image URI or Docker Hub (e.g., nginx:latest)          │
│                                                                   │
│ Essential container: ☑                                        │
│ → If this container stops, task stops                        │
│                                                                   │
│ Port mappings:                                                 │
│ Container port: [8080]    Protocol: [TCP]                    │
│ Port name: [web]                                              │
│ App protocol: [HTTP ▼]                                       │
│ → ⚡ Fargate uses awsvpc: container port = host port        │
│                                                                   │
│ Resource limits:                                               │
│ CPU: [512] (0.5 vCPU, in CPU units)                         │
│ Memory hard limit: [1024] MB                                 │
│ Memory soft limit: [512] MB                                  │
│ → Hard limit: Container killed if exceeded                  │
│ → Soft limit: Reserved, can burst to hard limit             │
│                                                                   │
│ Environment variables:                                        │
│ [NODE_ENV] = [production]                                    │
│ [DB_HOST] = [ValueFrom: arn:aws:ssm:...:parameter/db-host] │
│ → ⚡ Use SSM Parameter Store or Secrets Manager for secrets │
│                                                                   │
│ Secrets:                                                       │
│ [DB_PASSWORD] = ValueFrom: [arn:aws:secretsmanager:...:db-pw]│
│ → ECS agent injects secret at container startup             │
│                                                                   │
│ Logging:                                                       │
│ Log driver: [awslogs ▼] (⚡ default)                        │
│ awslogs-group: [/ecs/web-app-task]                           │
│ awslogs-region: [us-east-1]                                  │
│ awslogs-stream-prefix: [ecs]                                 │
│ → Logs go to CloudWatch Logs automatically                  │
│                                                                   │
│ Health check:                                                  │
│ Command: ["CMD-SHELL", "curl -f http://localhost:8080/health"]│
│ Interval: [30] seconds                                       │
│ Timeout: [5] seconds                                         │
│ Retries: [3]                                                 │
│ Start period: [60] seconds                                   │
│                                                                   │
│ [Add more containers] (sidecar, log router, etc.)           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STORAGE                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Ephemeral storage: [21] GB (default, range 21-200 GB)        │
│ → Temporary storage, lost when task stops                    │
│ → ⚡ Increase for data processing workloads                  │
│                                                                   │
│ EFS volumes:                                                   │
│ [Add volume]                                                  │
│ Name: [shared-data]                                           │
│ EFS file system: [fs-abc123 ▼]                               │
│ Root directory: [/]                                           │
│ Access point: [fsap-xyz789 ▼] (optional)                    │
│ Transit encryption: ☑ Enabled                                │
│ → ⚡ Persistent shared storage across tasks                  │
│                                                                   │
│                    [Create]                                    │
└─────────────────────────────────────────────────────────────────┘

Creating a Fargate Service:

Console → ECS → Cluster → Create service

┌─────────────────────────────────────────────────────────────────┐
│           CREATE SERVICE (FARGATE)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Compute options: ● Capacity provider strategy ○ Launch type  │
│ Capacity providers:                                            │
│ ├── FARGATE: [Base: 1, Weight: 1]                            │
│ ├── FARGATE_SPOT: [Base: 0, Weight: 3]                      │
│ └── → ⚡ 75% Spot + 25% On-Demand for cost savings         │
│                                                                   │
│ Task definition: [web-app-task ▼] Revision: [LATEST ▼]     │
│ Service name: [web-service]                                   │
│ Desired tasks: [3]                                            │
│                                                                   │
│ Networking:                                                    │
│ VPC: [prod-vpc ▼]                                            │
│ Subnets: ☑ private-subnet-1  ☑ private-subnet-2            │
│ → ⚡ Use private subnets + NAT Gateway                      │
│ Security group: [ecs-tasks-sg ▼]                             │
│ → Allow inbound from ALB security group only                │
│ Public IP: ○ (disabled for private subnets)                  │
│                                                                   │
│ Load balancer:                                                 │
│ Type: Application Load Balancer                               │
│ ALB: [prod-alb ▼]                                            │
│ Target group: [web-tg ▼]                                     │
│ Container: web-server:8080                                    │
│ Health check path: /health                                    │
│                                                                   │
│ Service auto scaling:                                         │
│ Min tasks: [2]  Max tasks: [10]  Desired: [3]               │
│ Scaling policy: Target tracking                               │
│ ECS service metric: [ECSServiceAverageCPUUtilization ▼]     │
│ Target value: [70] %                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Fargate with EKS

```
┌─────────────────────────────────────────────────────────────────────┐
│           FARGATE WITH EKS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → EKS → Cluster → Compute → Fargate profiles              │
│                                                                       │
│ Create Fargate profile:                                              │
│ Name: [web-app-fargate]                                             │
│ Pod execution role: [eksFargatePodExecutionRole ▼]                │
│ → Allows Fargate to pull images, write logs                      │
│                                                                       │
│ Subnets: ☑ private-subnet-1  ☑ private-subnet-2                │
│ → ⚠️ Fargate pods MUST run in private subnets                   │
│                                                                       │
│ Pod selectors (which pods run on Fargate):                        │
│ Namespace: [web-app]                                                │
│ Labels (optional):                                                   │
│ [app] = [web-server]                                                │
│ → Pods matching namespace + labels → scheduled on Fargate       │
│ → All other pods → scheduled on EC2 managed node groups        │
│                                                                       │
│ How it works:                                                        │
│ 1. Create Fargate profile with namespace/label selectors         │
│ 2. Deploy pod with matching namespace/labels                      │
│ 3. EKS scheduler routes pod to Fargate                            │
│ 4. Each pod gets its own micro-VM (isolation)                    │
│                                                                       │
│ Limitations on EKS Fargate:                                         │
│ ├── No DaemonSets (use sidecar containers instead)              │
│ ├── No privileged containers                                      │
│ ├── No HostPort / HostNetwork                                    │
│ ├── No GPUs                                                       │
│ ├── No EBS volumes (use EFS)                                     │
│ ├── Max 4 vCPU, 30 GB memory per pod                            │
│ └── Private subnets only                                         │
│                                                                       │
│ Example pod spec:                                                    │
│ apiVersion: v1                                                      │
│ kind: Pod                                                            │
│ metadata:                                                            │
│   namespace: web-app                                                │
│   labels:                                                            │
│     app: web-server   # Matches Fargate profile selector        │
│ spec:                                                                │
│   containers:                                                        │
│   - name: web                                                        │
│     image: 123456.dkr.ecr.us-east-1.amazonaws.com/web:v1.0       │
│     resources:                                                       │
│       requests:                                                      │
│         cpu: "256m"                                                  │
│         memory: "512Mi"                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Networking, Storage & Security

```
┌─────────────────────────────────────────────────────────────────────┐
│           NETWORKING, STORAGE & SECURITY                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Networking (awsvpc mode):                                            │
│ ├── Each Fargate task gets its own ENI (network interface)      │
│ ├── Each task gets its own private IP address                    │
│ ├── Security groups applied at task level                        │
│ ├── Container port = Host port (no port mapping needed)        │
│ └── ⚡ Private subnets + NAT Gateway recommended                │
│                                                                       │
│ Internet access:                                                     │
│ ├── Public subnet + Public IP: Direct internet access           │
│ ├── Private subnet + NAT Gateway: Internet via NAT (⚡ secure) │
│ └── VPC endpoints: Access AWS services without internet        │
│     → ⚡ Add VPC endpoints for: ECR, S3, CloudWatch Logs       │
│     → Eliminates NAT Gateway data charges for AWS traffic     │
│                                                                       │
│ Storage:                                                              │
│ ├── Ephemeral: 21 GB default (up to 200 GB), lost on stop     │
│ ├── EFS: Persistent, shared across tasks, unlimited size       │
│ ├── ⚠️ No EBS volume support on Fargate                       │
│ └── Bind mounts: Share data between containers in same task   │
│                                                                       │
│ Security:                                                             │
│ ├── Task Role: IAM role for your application code              │
│ │   → What AWS services your containers can access              │
│ ├── Execution Role: IAM role for ECS agent                      │
│ │   → Pull ECR images, write CloudWatch Logs, get secrets     │
│ ├── Security groups: Per-task network firewall                  │
│ ├── Encryption: In-transit (TLS) + at-rest (ephemeral + EFS) │
│ └── Secrets: Inject from Secrets Manager or SSM at runtime    │
│                                                                       │
│ Fargate Spot:                                                        │
│ ├── Up to 70% discount vs regular Fargate                      │
│ ├── Can be interrupted with 30-second warning (SIGTERM)       │
│ ├── ⚡ Use for: Batch processing, dev/test, fault-tolerant    │
│ ├── ⚠️ Not for: Stateful services, databases                  │
│ └── Mix with regular Fargate (capacity provider strategy)     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
# ECS Fargate task definition
resource "aws_ecs_task_definition" "web" {
  family                   = "web-app-task"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 1024  # 1 vCPU
  memory                   = 2048  # 2 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn
  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "ARM64"  # Graviton, cheaper
  }

  container_definitions = jsonencode([{
    name      = "web-server"
    image     = "${aws_ecr_repository.app.repository_url}:v1.0.0"
    essential = true
    portMappings = [{
      containerPort = 8080
      protocol      = "tcp"
    }]
    environment = [
      { name = "NODE_ENV", value = "production" }
    ]
    secrets = [
      { name = "DB_PASSWORD", valueFrom = aws_secretsmanager_secret.db.arn }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = aws_cloudwatch_log_group.ecs.name
        awslogs-region        = "us-east-1"
        awslogs-stream-prefix = "ecs"
      }
    }
    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
      startPeriod = 60
    }
  }])
}

# ECS Fargate service with auto-scaling
resource "aws_ecs_service" "web" {
  name            = "web-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.web.arn
  desired_count   = 3

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 1
  }
  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 3
  }

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.web.arn
    container_name   = "web-server"
    container_port   = 8080
  }
}

resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.web.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "cpu-target-tracking"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace
  policy_type        = "TargetTrackingScaling"

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70
  }
}
```

```bash
# Run standalone Fargate task
aws ecs run-task \
  --cluster prod-cluster \
  --task-definition web-app-task \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc],securityGroups=[sg-123],assignPublicIp=DISABLED}"

# Update service
aws ecs update-service \
  --cluster prod-cluster \
  --service web-service \
  --task-definition web-app-task:5 \
  --desired-count 3

# Execute command in running task (ECS Exec)
aws ecs execute-command \
  --cluster prod-cluster \
  --task task-id \
  --container web-server \
  --interactive \
  --command "/bin/sh"
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Web API Service                                           │
│ ALB → Fargate (ECS) → DynamoDB                                    │
│ ├── 3 tasks across 2 AZs (high availability)                   │
│ ├── Auto-scaling on CPU utilization                              │
│ ├── 75% Fargate Spot + 25% regular                              │
│ ├── Secrets from Secrets Manager                                 │
│ └── Logs to CloudWatch, traces to X-Ray                        │
│                                                                       │
│ Pattern 2: Background Worker                                        │
│ SQS → Fargate (ECS) → Process → S3                               │
│ ├── Scale on SQS queue depth (custom metric)                   │
│ ├── Scale to 0 when no messages                                  │
│ ├── Fargate Spot for cost savings                                │
│ └── DLQ for failed messages                                      │
│                                                                       │
│ Pattern 3: Mixed EKS Cluster                                        │
│ EKS Cluster:                                                        │
│ ├── Managed node group (EC2): Stateful workloads, DaemonSets  │
│ ├── Fargate profile (web-app namespace): Web services          │
│ └── Fargate profile (batch namespace): Batch jobs               │
│ → Right compute for each workload type                          │
│                                                                       │
│ Pattern 4: Scheduled Tasks                                          │
│ EventBridge Schedule → ECS RunTask (Fargate)                      │
│ ├── Daily report generation                                      │
│ ├── Nightly data processing                                      │
│ ├── No idle compute costs (runs only when needed)              │
│ └── ⚡ Serverless cron jobs                                      │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Use ARM64 (Graviton) for 20% cost reduction                 │
│ 2. Use Fargate Spot for fault-tolerant workloads               │
│ 3. Private subnets + VPC endpoints                               │
│ 4. Inject secrets via Secrets Manager (not env vars)           │
│ 5. Enable ECS Exec for debugging                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting: Common Fargate Issues

### "Task stuck in PROVISIONING"

```
1. ☐ ENI limit reached in subnet?
   Each Fargate task gets its own ENI. Small subnets (/24 = 256 IPs)
   can run out if running many tasks.
   Fix: Use larger subnets or spread across more AZs.

2. ☐ Insufficient IP addresses in subnet?
   Console → VPC → Subnets → Available IPv4 addresses

3. ☐ Service-linked role missing?
   First Fargate task may fail if the AWSServiceRoleForECS role
   doesn't exist yet. Usually auto-creates, but check IAM.
```

### "CannotPullContainerError"

```
1. ☐ Is the image URI correct?
   Check for typos in account ID, region, repo name, tag

2. ☐ Does the task execution role have ECR permissions?
   Attach: AmazonECSTaskExecutionRolePolicy

3. ☐ For private subnets: Are VPC endpoints configured?
   Need: ecr.api, ecr.dkr, s3 (gateway), logs
   OR: NAT Gateway for internet access

4. ☐ For public images (Docker Hub): Does subnet have internet?
   Fargate in public subnet: assignPublicIp = ENABLED
   Fargate in private subnet: NAT Gateway required
```

### "Task Role vs Execution Role" Confusion

```
┌─────────────────────────────────────────────────────────┐
│ Execution Role (used by ECS agent)                        │
│ └── Pull images from ECR                                  │
│ └── Send logs to CloudWatch                               │
│ └── Retrieve secrets from Secrets Manager                 │
│                                                           │
│ Task Role (used by YOUR application code)                 │
│ └── Access S3 buckets                                     │
│ └── Read/write DynamoDB tables                            │
│ └── Send messages to SQS                                  │
│ └── Any AWS service YOUR code calls                       │
└─────────────────────────────────────────────────────────┘
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Task role has no permissions | App gets AccessDenied | Add policies for services your code calls |
| No VPC endpoints in private subnet | Can't pull images or send logs | Add ECR + S3 + Logs endpoints |
| CPU/memory too low | OOMKilled or slow performance | Monitor CloudWatch and adjust task definition |
| No health check in service | Unhealthy tasks not replaced | Add ALB health check or container health check |
| Logging not configured | Can't debug crashes | Set logDriver to "awslogs" in task definition |

---

## Quick Reference

```
Fargate Quick Reference:
├── Serverless containers: No EC2 instances to manage
├── Works with: ECS and EKS
├── CPU: 0.25 to 16 vCPU
├── Memory: 0.5 to 120 GB
├── Networking: awsvpc (each task gets own ENI + IP)
├── Storage: Ephemeral (21-200 GB) + EFS (persistent)
├── Pricing: Per vCPU + GB per hour
├── Fargate Spot: Up to 70% discount, 30s interrupt warning
├── ARM64 (Graviton): 20% cheaper, better performance
├── EKS limitations: No DaemonSets, no EBS, private subnets only
├── ⚡ Use private subnets + VPC endpoints for ECR/S3/CW
├── ⚡ ARM64 + Fargate Spot = maximum cost savings
└── ⚡ Enable ECS Exec for container debugging
```

---

## What's Next?

In **Chapter 52: Amazon Kinesis**, we'll cover real-time data streaming, Kinesis Data Streams, Data Firehose, and Data Analytics for processing streaming data at scale.
