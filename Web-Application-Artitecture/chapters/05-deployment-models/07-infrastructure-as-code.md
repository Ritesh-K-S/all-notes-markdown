# Infrastructure as Code (Terraform, CloudFormation)

> **What you'll learn**: How to define your entire infrastructure — servers, networks, databases, load balancers, DNS — as version-controlled code files that can be reviewed, tested, and deployed repeatably, eliminating "click ops" and snowflake servers forever.

---

## Real-Life Analogy

Imagine building a house. You have two approaches:

**Approach 1 (Manual / "ClickOps")**: You show up at the construction site each day and tell workers verbally: "Put a wall here... no wait, make it 2 feet to the left... add a window... actually make it bigger." No blueprints. If the house burns down, nobody remembers exactly how it was built. Rebuilding takes months of guessing.

**Approach 2 (Infrastructure as Code)**: You have detailed **architectural blueprints** stored in a file. Every wall dimension, pipe location, and wire gauge is documented. If the house burns down, hand the blueprint to any construction crew and they'll build an **identical copy** in days. Want a second house? Use the same blueprint. Want to change a window? Update the blueprint, get it reviewed, then apply the change.

IaC is Approach 2. Your infrastructure is defined in **code files** (`.tf`, `.yaml`, `.json`) that are:
- Version controlled (Git)
- Peer reviewed (Pull Requests)
- Tested before applying
- Repeatable (same code → same infrastructure, every time)

---

## Core Concept Explained Step-by-Step

### The Problem IaC Solves

```
WITHOUT Infrastructure as Code ("ClickOps"):

┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  Engineer logs into AWS Console...                                 │
│                                                                    │
│  Click "Create Instance"... select t3.medium... click...          │
│  Click "Create Security Group"... add port 443... click...        │
│  Click "Create Database"... PostgreSQL 15... click...             │
│  Click "Create Load Balancer"... attach instances... click...     │
│                                                                    │
│  PROBLEMS:                                                         │
│  ❌ Nobody knows exactly what was configured                       │
│  ❌ Can't reproduce it exactly (human memory fails)               │
│  ❌ No audit trail ("who changed the security group?")            │
│  ❌ "Works on prod, can't recreate in staging"                    │
│  ❌ One wrong click → production outage                           │
│  ❌ Scaling = more clicking (create 50 servers manually? No.)     │
│  ❌ "Snowflake servers" — each one is slightly different          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘


WITH Infrastructure as Code:

┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  infrastructure/main.tf:                                           │
│                                                                    │
│  resource "aws_instance" "app" {                                   │
│    count         = 5                                               │
│    instance_type = "t3.medium"                                     │
│    ami           = "ami-0c55b159c"                                 │
│  }                                                                 │
│                                                                    │
│  $ terraform apply                                                 │
│  → Creates 5 identical instances automatically                     │
│                                                                    │
│  BENEFITS:                                                         │
│  ✅ Everything documented in code                                  │
│  ✅ Reproducible (same .tf file → same infrastructure)             │
│  ✅ Version controlled (git log shows ALL changes)                 │
│  ✅ Peer reviewed (PR review before infrastructure changes)        │
│  ✅ Staging = same code, different variables                       │
│  ✅ Disaster recovery: re-run the code → everything recreated      │
│  ✅ Scale: change count=5 to count=50 → done                      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### How IaC Works (Declarative Model)

```
IMPERATIVE (scripting — HOW to do it):
"Create a server. If it exists, skip. Then create a security group. 
 Then attach it. If the security group already exists, modify it..."

DECLARATIVE (IaC — WHAT the end state should be):
"I want 3 servers, 1 load balancer, 1 database. Make it so."

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  YOU WRITE:                   IaC TOOL FIGURES OUT:              │
│                                                                  │
│  "I want 3 servers"    →     Current state: 1 server            │
│                               Desired state: 3 servers           │
│                               Action: CREATE 2 more servers      │
│                                                                  │
│  "I want 3 servers"    →     Current state: 5 servers           │
│  (same code, next day)        Desired state: 3 servers           │
│                               Action: DESTROY 2 servers          │
│                                                                  │
│  "I want 3 servers"    →     Current state: 3 servers           │
│  (no change)                  Desired state: 3 servers           │
│                               Action: NOTHING (already correct)  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### The IaC Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IaC WORKFLOW                                      │
│                                                                     │
│  1. WRITE code (main.tf / template.yaml)                           │
│     │                                                               │
│     ▼                                                               │
│  2. VERSION CONTROL (git commit, push to branch)                   │
│     │                                                               │
│     ▼                                                               │
│  3. PULL REQUEST (team reviews infrastructure changes)             │
│     │                                                               │
│     ▼                                                               │
│  4. PLAN (terraform plan — shows what WILL change)                 │
│     │  "Plan: 2 to add, 0 to change, 1 to destroy"               │
│     │                                                               │
│     ▼                                                               │
│  5. APPROVE (human reviews the plan)                               │
│     │                                                               │
│     ▼                                                               │
│  6. APPLY (terraform apply — makes the changes)                    │
│     │  "Apply complete! Resources: 2 added, 0 changed, 1 destroyed"│
│     │                                                               │
│     ▼                                                               │
│  7. STATE updated (Terraform knows current infrastructure state)   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Major IaC Tools Comparison

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  Tool            │ Provider │ Language │ State    │ Best For          │
│  ────────────────┼──────────┼──────────┼──────────┼─────────────── │
│  Terraform       │ Multi    │ HCL      │ Remote   │ Multi-cloud,     │
│                  │ cloud    │          │ backend  │ most popular     │
│                  │          │          │          │                  │
│  CloudFormation  │ AWS only │ YAML/    │ AWS      │ AWS-only shops   │
│                  │          │ JSON     │ managed  │                  │
│                  │          │          │          │                  │
│  Pulumi          │ Multi    │ Python/  │ Remote   │ Devs who prefer  │
│                  │ cloud    │ TS/Go    │ backend  │ real languages   │
│                  │          │          │          │                  │
│  AWS CDK         │ AWS only │ TS/Py/  │ AWS      │ AWS devs who     │
│                  │          │ Java     │ managed  │ hate YAML        │
│                  │          │          │          │                  │
│  Ansible         │ Multi    │ YAML     │ Stateless│ Configuration    │
│                  │          │          │          │ management       │
│                  │          │          │          │                  │
│  OpenTofu        │ Multi    │ HCL      │ Remote   │ Open-source      │
│                  │ cloud    │          │ backend  │ Terraform fork   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Terraform's Internal Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TERRAFORM INTERNALS                              │
│                                                                     │
│  ┌─────────────────┐                                               │
│  │  .tf Files       │  Your infrastructure definition              │
│  │  (HCL code)     │  (what you WANT)                             │
│  └────────┬────────┘                                               │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                               │
│  │  Terraform Core  │  The engine                                  │
│  │                  │                                               │
│  │  1. Parse .tf    │                                               │
│  │  2. Build graph  │  (dependency graph of resources)             │
│  │  3. Compare to   │                                               │
│  │     state file   │                                               │
│  │  4. Generate plan│                                               │
│  └────────┬────────┘                                               │
│           │                                                         │
│     ┌─────┼─────┐                                                  │
│     ▼     ▼     ▼                                                  │
│  ┌─────┐┌─────┐┌─────┐                                            │
│  │ AWS ││ GCP ││Azure│  Providers (plugins for each cloud)        │
│  │Prov.││Prov.││Prov.│                                            │
│  └──┬──┘└──┬──┘└──┬──┘                                            │
│     │      │      │                                                │
│     ▼      ▼      ▼                                                │
│  ┌─────┐┌─────┐┌─────┐                                            │
│  │ AWS ││ GCP ││Azure│  Actual cloud APIs                         │
│  │ API ││ API ││ API │                                            │
│  └─────┘└─────┘└─────┘                                            │
│                                                                     │
│  ┌─────────────────┐                                               │
│  │  State File      │  Records what EXISTS (current reality)       │
│  │  (terraform.     │  Stored in S3, GCS, or Terraform Cloud      │
│  │   tfstate)       │                                               │
│  └─────────────────┘                                               │
│                                                                     │
│  Plan = Diff(desired state from .tf, current state from .tfstate)  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### The State File — Terraform's Memory

```
TERRAFORM STATE FILE (terraform.tfstate):

{
  "resources": [
    {
      "type": "aws_instance",
      "name": "app",
      "instances": [
        {
          "attributes": {
            "id": "i-0abc123def456",       ← Actual AWS instance ID
            "instance_type": "t3.medium",
            "public_ip": "54.23.45.67",
            "ami": "ami-0c55b159c"
          }
        }
      ]
    }
  ]
}

WHY STATE MATTERS:
- Terraform needs to know what ALREADY exists
- Without state, it would try to create everything from scratch every time
- State maps your code resources to REAL cloud resources

STATE MUST BE:
- Stored remotely (S3, Terraform Cloud) — not on a laptop!
- Locked during changes (prevent two people from modifying simultaneously)
- Never edited manually (use terraform import / terraform state commands)
```

### Dependency Graph

```
Terraform automatically determines the ORDER to create resources:

Your Code:
├── VPC
├── Subnet (depends on VPC)
├── Security Group (depends on VPC)
├── EC2 Instance (depends on Subnet + Security Group)
├── RDS Database (depends on Subnet + Security Group)
└── Load Balancer (depends on EC2 Instances)

Terraform's Execution Graph:
                    ┌─────┐
                    │ VPC │
                    └──┬──┘
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
         ┌────────┐┌───────┐┌──────────┐
         │ Subnet ││Subnet ││Sec Group │
         │ (pub)  ││(priv) ││          │
         └───┬────┘└───┬───┘└────┬─────┘
             │         │         │
        ┌────┴─────────┼─────────┘
        ▼              ▼
   ┌─────────┐   ┌──────────┐
   │   EC2   │   │   RDS    │
   │Instance │   │ Database │
   └────┬────┘   └──────────┘
        │
        ▼
   ┌─────────┐
   │   ALB   │
   └─────────┘

Terraform creates VPC first, then subnets/SG in parallel, then EC2/RDS, then ALB.
```

---

## Code Examples

### Terraform (Complete Web Application Stack)

```hcl
# main.tf — Full web application infrastructure
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  
  # Store state remotely in S3 (not on your laptop!)
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/web-app/terraform.tfstate"
    region = "us-east-1"
    
    # Lock state to prevent concurrent modifications
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
}

# --- NETWORKING ---
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "${var.app_name}-vpc" }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  map_public_ip_on_launch = true
  tags = { Name = "${var.app_name}-public-${count.index + 1}" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "${var.app_name}-private-${count.index + 1}" }
}

# --- SECURITY GROUPS ---
resource "aws_security_group" "alb" {
  name   = "${var.app_name}-alb-sg"
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Public internet → ALB
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "app" {
  name   = "${var.app_name}-app-sg"
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Only from ALB!
  }
}

resource "aws_security_group" "db" {
  name   = "${var.app_name}-db-sg"
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # Only from app!
  }
}

# --- LOAD BALANCER ---
resource "aws_lb" "main" {
  name               = "${var.app_name}-alb"
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
}

# --- DATABASE ---
resource "aws_db_instance" "main" {
  identifier     = "${var.app_name}-db"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = var.db_instance_class  # "db.t3.medium"
  
  allocated_storage = 100
  storage_encrypted = true
  
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
  
  publicly_accessible = false  # Private subnet only!
  multi_az           = var.environment == "production"
  
  backup_retention_period = 7
  skip_final_snapshot    = var.environment != "production"
}

# --- APPLICATION (ECS) ---
resource "aws_ecs_service" "app" {
  name            = "${var.app_name}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.app_instance_count  # 3 for prod, 1 for staging
  
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 8080
  }
}

# --- OUTPUTS ---
output "app_url" {
  value = "https://${aws_lb.main.dns_name}"
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = true
}
```

### Variables File (Reuse for Multiple Environments)

```hcl
# variables.tf — Parameterize for different environments
variable "app_name" {
  description = "Application name"
  type        = string
  default     = "mywebapp"
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment (staging or production)"
  type        = string
}

variable "app_instance_count" {
  description = "Number of application instances"
  type        = number
  default     = 3
}

variable "db_instance_class" {
  description = "RDS instance size"
  type        = string
  default     = "db.t3.medium"
}
```

```hcl
# environments/production.tfvars
environment        = "production"
app_instance_count = 5
db_instance_class  = "db.r5.xlarge"
aws_region         = "us-east-1"

# environments/staging.tfvars
environment        = "staging"
app_instance_count = 2
db_instance_class  = "db.t3.small"
aws_region         = "us-east-1"
```

### AWS CloudFormation (YAML)

```yaml
# cloudformation.yaml — Same infrastructure, AWS-native tool
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Web Application Infrastructure'

Parameters:
  Environment:
    Type: String
    AllowedValues: [staging, production]
  AppInstanceCount:
    Type: Number
    Default: 3

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: '10.0.0.0/16'
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-vpc'

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.1.0/24'
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true

  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: postgres
      EngineVersion: '15.4'
      DBInstanceClass: db.t3.medium
      AllocatedStorage: 100
      StorageEncrypted: true
      PubliclyAccessible: false
      MultiAZ: !If [IsProduction, true, false]
      BackupRetentionPeriod: 7

  AppService:
    Type: AWS::ECS::Service
    Properties:
      Cluster: !Ref ECSCluster
      DesiredCount: !Ref AppInstanceCount
      TaskDefinition: !Ref AppTaskDefinition

Conditions:
  IsProduction: !Equals [!Ref Environment, 'production']

Outputs:
  AppURL:
    Value: !GetAtt LoadBalancer.DNSName
```

### Python (Pulumi — IaC in Real Code)

```python
# __main__.py — Pulumi: Infrastructure as Code in Python
import pulumi
import pulumi_aws as aws

config = pulumi.Config()
environment = config.require("environment")
instance_count = config.get_int("instance_count") or 3

# Create VPC
vpc = aws.ec2.Vpc("main-vpc",
    cidr_block="10.0.0.0/16",
    enable_dns_hostnames=True,
    tags={"Name": f"myapp-{environment}-vpc"}
)

# Create subnets
public_subnets = []
for i in range(2):
    subnet = aws.ec2.Subnet(f"public-subnet-{i}",
        vpc_id=vpc.id,
        cidr_block=f"10.0.{i+1}.0/24",
        map_public_ip_on_launch=True,
        tags={"Name": f"public-{i}"}
    )
    public_subnets.append(subnet)

# Create database
db = aws.rds.Instance("app-db",
    engine="postgres",
    engine_version="15.4",
    instance_class="db.t3.medium" if environment == "staging" else "db.r5.xlarge",
    allocated_storage=100,
    publicly_accessible=False,
    multi_az=(environment == "production"),
    skip_final_snapshot=(environment != "production"),
)

# Output the endpoints
pulumi.export("vpc_id", vpc.id)
pulumi.export("db_endpoint", db.endpoint)
```

---

## Infrastructure Example

### Full GitOps Workflow with Terraform

```
┌─────────────────────────────────────────────────────────────────────┐
│                 GITOPS WORKFLOW WITH TERRAFORM                       │
│                                                                     │
│  Developer                                                          │
│    │                                                                │
│    │ 1. Makes infrastructure change                                │
│    │    (edit main.tf: change instance_count from 3 to 5)          │
│    ▼                                                                │
│  ┌────────────┐                                                    │
│  │   Git      │ 2. Push to branch, create Pull Request             │
│  │ Repository │                                                    │
│  └─────┬──────┘                                                    │
│        │                                                            │
│        ▼                                                            │
│  ┌────────────────────────────────────────┐                        │
│  │   CI Pipeline (GitHub Actions)         │                        │
│  │                                        │                        │
│  │   Step 1: terraform fmt -check         │ (code style)           │
│  │   Step 2: terraform validate           │ (syntax check)         │
│  │   Step 3: terraform plan               │ (what will change?)    │
│  │                                        │                        │
│  │   → Posts plan output as PR comment:   │                        │
│  │   "Plan: 2 to add, 0 to change,       │                        │
│  │    0 to destroy"                       │                        │
│  └────────────────────────────────────────┘                        │
│        │                                                            │
│        ▼                                                            │
│  ┌────────────┐                                                    │
│  │  Reviewer  │ 3. Reviews the plan, approves PR                   │
│  └─────┬──────┘                                                    │
│        │                                                            │
│        ▼ (merge to main)                                           │
│  ┌────────────────────────────────────────┐                        │
│  │   CD Pipeline                          │                        │
│  │                                        │                        │
│  │   Step 1: terraform plan (confirm)     │                        │
│  │   Step 2: terraform apply -auto-approve│ (make changes!)        │
│  │                                        │                        │
│  │   → 2 new instances created in AWS     │                        │
│  └────────────────────────────────────────┘                        │
│        │                                                            │
│        ▼                                                            │
│  ┌────────────────────────────────────────┐                        │
│  │   State File (S3)                      │ 4. Updated with new    │
│  │   Records: 5 instances exist now       │    resource IDs        │
│  └────────────────────────────────────────┘                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Terraform Project Structure

```
infrastructure/
├── environments/
│   ├── production/
│   │   ├── main.tf          ← References shared modules
│   │   ├── variables.tf
│   │   ├── terraform.tfvars ← Production-specific values
│   │   └── backend.tf       ← S3 state for prod
│   └── staging/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars ← Staging-specific values  
│       └── backend.tf       ← Separate S3 state for staging
├── modules/
│   ├── vpc/
│   │   ├── main.tf          ← Reusable VPC module
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ecs-service/
│   │   ├── main.tf          ← Reusable ECS module
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── rds/
│       ├── main.tf          ← Reusable database module
│       ├── variables.tf
│       └── outputs.tf
└── .github/
    └── workflows/
        └── terraform.yml    ← CI/CD pipeline
```

### GitHub Actions Pipeline for Terraform

```yaml
# .github/workflows/terraform.yml
name: Terraform Infrastructure

on:
  pull_request:
    paths: ['infrastructure/**']
  push:
    branches: [main]
    paths: ['infrastructure/**']

jobs:
  plan:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0
      
      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure/environments/production
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        working-directory: infrastructure
      
      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        working-directory: infrastructure/environments/production
      
      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📋
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Review carefully before merging!*`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

  apply:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      
      - name: Terraform Apply
        run: |
          terraform init
          terraform apply -auto-approve
        working-directory: infrastructure/environments/production
```

---

## Real-World Example

### Netflix — Fully Automated Infrastructure

```
Netflix Infrastructure (managed by "Spinnaker" + Terraform):

- 100,000+ AWS instances managed entirely through code
- No engineer EVER manually creates resources
- All infrastructure changes go through:
  1. Code review (PR on infrastructure repo)
  2. Automated validation
  3. Staged deployment (staging → canary → production)

Key principle: "If it's not in code, it doesn't exist."

Netflix's stack:
├── Terraform: VPCs, subnets, IAM, base infrastructure
├── Spinnaker: Deployment pipelines for applications
├── Titus (internal): Container orchestration
└── Chaos Monkey: Continuously tests infrastructure resilience
```

### HashiCorp (Creators of Terraform)

```
How Terraform is used globally (2024):

- Used by 80%+ of Fortune 500 companies
- Manages millions of cloud resources daily
- Supports 3000+ providers (AWS, GCP, Azure, Cloudflare, 
  Datadog, GitHub, Kubernetes, etc.)

Typical enterprise usage:
├── Platform team writes reusable modules
├── App teams use modules (like building blocks)
├── All changes through PRs
├── Automated apply after merge
└── State stored in Terraform Cloud / S3

Example:
  Module: "production-service"
  Includes: ALB + ECS + RDS + Redis + CloudWatch Alarms
  
  App team just writes:
  module "my-api" {
    source         = "modules/production-service"
    app_name       = "user-api"
    instance_count = 5
    db_size        = "db.r5.large"
  }
  
  → Gets entire production stack automatically!
```

---

## Common Mistakes / Pitfalls

### 1. State File on Local Machine
❌ **Mistake**: `terraform.tfstate` file on your laptop. If laptop dies, state is lost (can't manage infrastructure anymore).
✅ **Fix**: Store state in S3/GCS with locking (DynamoDB/GCS bucket).

```hcl
# ALWAYS use remote state in production!
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # Prevents concurrent modifications
  }
}
```

### 2. Secrets in Terraform Code
❌ **Mistake**: Database passwords hardcoded in `.tf` files committed to Git.
✅ **Fix**: Use secrets managers (AWS Secrets Manager, Vault) or sensitive variables.

```hcl
# BAD: password in code!
resource "aws_db_instance" "db" {
  password = "SuperSecret123"  # 🚨 NEVER DO THIS
}

# GOOD: reference from secrets manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db-password"
}
resource "aws_db_instance" "db" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

### 3. No Terraform Plan Review
❌ **Mistake**: Running `terraform apply` without reviewing the plan first → accidentally destroys production database.
✅ **Fix**: Always `terraform plan` first. In CI, post plan output to PR for human review.

### 4. Monolithic Terraform Configuration
❌ **Mistake**: One massive `main.tf` file with 2000 lines for ALL infrastructure.
✅ **Fix**: Split into modules and environments. Each team/service has its own state file.

```
BAD: One state for everything (blast radius = ALL infrastructure)
infrastructure/main.tf (2000 lines, one state file)

GOOD: Separate states (blast radius = one component)
infrastructure/
├── networking/     (its own state)
├── database/       (its own state)
├── app-service-a/  (its own state)
└── app-service-b/  (its own state)
```

### 5. Modifying State Manually
❌ **Mistake**: Editing `terraform.tfstate` by hand to "fix" things.
✅ **Fix**: Use `terraform import`, `terraform state mv`, `terraform state rm` commands.

---

## When to Use / When NOT to Use

### ✅ Use IaC When:

| Criteria | Why |
|----------|-----|
| **Any production infrastructure** | Reproducibility and auditability are essential |
| **Multiple environments** | Same code, different params = staging/prod parity |
| **Team > 1 person** | Everyone sees and reviews infrastructure changes |
| **Disaster recovery needed** | Re-run code → recreate everything from scratch |
| **Compliance requirements** | Audit trail of every change via Git history |
| **Scaling infrastructure frequently** | Change a number, apply, done |

### ❌ May Skip When:

| Criteria | Why |
|----------|-----|
| **Quick prototype / hackathon** | Speed matters more than repeatability |
| **One-off test server** | Not worth setting up state management |
| **Learning / exploring a new service** | Click around the console to understand it first |
| **Very simple setup (1 server)** | Overhead of Terraform isn't justified yet |

> **Pro tip**: Even "simple" setups benefit from IaC if they need to last more than a week or be shared with others.

---

## Key Takeaways

- **Infrastructure as Code** means defining all your infrastructure (servers, networks, databases) in version-controlled files instead of manual console clicks.
- **Declarative model**: You describe WHAT you want, the tool figures out HOW to get there (creating, updating, or destroying resources as needed).
- **Terraform is the most popular** multi-cloud IaC tool, using HCL (HashiCorp Configuration Language). CloudFormation is AWS-native.
- **State files are critical** — they map your code to real resources. Store remotely, lock during changes, never edit by hand.
- **Same code, different environments**: Use variables/tfvars to deploy identical infrastructure for staging and production.
- **GitOps workflow**: All changes through PRs → plan review → merge → auto-apply. No one manually creates resources.
- **Start with IaC from day one** if possible — retrofitting is much harder than starting fresh.

---

## What's Next?

You've now completed **Part 5: Deployment Models**! You understand the full journey from a single server to multi-region deployments, with zero-downtime release strategies and infrastructure defined as code.

In **Part 6: Load Balancing — Distributing Traffic Like a Pro**, we'll dive deep into how load balancers work — the algorithms they use, the different layers (L4 vs L7), health checks, sticky sessions, and the tools (Nginx, HAProxy, AWS ALB) that power traffic distribution at scale. Start with **Chapter 6.1: What is Load Balancing?**
