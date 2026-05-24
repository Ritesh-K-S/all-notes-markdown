# Amazon Web Services (AWS) - Complete Guide

> A comprehensive, hands-on guide to AWS for full-stack developers and DevOps engineers.  
> Goal: Master every aspect of AWS to independently manage cloud infrastructure for your company.

---

## How to Use This Guide

- Each chapter is a separate `.md` file linked below.
- Chapters follow a logical learning path — from account setup to advanced services.
- Each service file includes: portal walkthrough, all fields/options explained, diagrams, and real-world usage.

---

## Table of Contents

### Part 1: Foundation & Account Management

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 1 | Introduction to AWS | `01-introduction-to-aws.md` | What is AWS, global infrastructure, regions, availability zones, edge locations |
| 2 | Account Setup & Organization | `02-account-setup-and-organization.md` | Personal vs Org accounts, AWS Organizations, consolidated billing, SCPs |
| 3 | Resource Hierarchy & Management | `03-resource-hierarchy.md` | Account → Organization → OU → Account structure, tagging strategies, resource groups |
| 4 | IAM - Identity & Access Management | `04-iam.md` | Users, groups, roles, policies, MFA, identity federation, SSO, permission boundaries |
| 5 | Billing, Cost Management & Budgets | `05-billing-and-cost-management.md` | Cost Explorer, Budgets, Savings Plans, Reserved Instances, cost allocation tags |

### Part 2: Networking

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 6 | VPC - Virtual Private Cloud | `06-vpc.md` | VPC creation, CIDR blocks, subnets (public/private), route tables, internet gateway |
| 7 | VPC Advanced Networking | `07-vpc-advanced.md` | NAT Gateway, VPC Peering, Transit Gateway, VPC Endpoints, PrivateLink |
| 8 | Security Groups & NACLs | `08-security-groups-nacls.md` | Inbound/outbound rules, stateful vs stateless, best practices |
| 9 | Route 53 - DNS Management | `09-route53.md` | Hosted zones, record types, routing policies, health checks, domain registration |
| 10 | CloudFront - CDN | `10-cloudfront.md` | Distributions, origins, behaviors, cache policies, SSL/TLS, signed URLs |
| 11 | Elastic Load Balancing | `11-elastic-load-balancing.md` | ALB, NLB, GLB, target groups, health checks, listeners, rules |
| 12 | AWS Global Accelerator | `12-global-accelerator.md` | Static IPs, endpoint groups, traffic dials |

### Part 3: Compute

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 13 | EC2 - Elastic Compute Cloud | `13-ec2.md` | Instance types, AMIs, key pairs, user data, placement groups, Spot/On-Demand/Reserved |
| 14 | EC2 Auto Scaling | `14-ec2-auto-scaling.md` | Launch templates, ASG, scaling policies, lifecycle hooks, instance refresh |
| 15 | Lambda - Serverless Compute | `15-lambda.md` | Functions, triggers, layers, concurrency, versions & aliases, destinations |
| 15a | Lambda Deep Dive | `15a-lambda-deep-dive.md` | Advanced Lambda — Lambda@Edge, container images, Lambda URLs, VPC access, destinations, DLQs |
| 16 | ECS - Elastic Container Service | `16-ecs.md` | Clusters, task definitions, services, Fargate vs EC2, service discovery |
| 17 | EKS - Elastic Kubernetes Service | `17-eks.md` | Cluster creation, node groups, Fargate profiles, add-ons, IRSA |
| 18 | Elastic Beanstalk | `18-elastic-beanstalk.md` | Environments, platforms, deployment policies, configuration |
| 19 | App Runner | `19-app-runner.md` | Source configuration, auto scaling, custom domains |

### Part 4: Storage

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 20 | S3 - Simple Storage Service | `20-s3.md` | Buckets, objects, storage classes, versioning, lifecycle rules, replication |
| 21 | S3 Advanced Features | `21-s3-advanced.md` | Bucket policies, ACLs, presigned URLs, event notifications, S3 Select, Transfer Acceleration |
| 22 | EBS - Elastic Block Store | `22-ebs.md` | Volume types (gp3, io2, st1, sc1), snapshots, encryption, multi-attach |
| 23 | EFS - Elastic File System | `23-efs.md` | File systems, mount targets, performance modes, throughput modes, access points |
| 24 | FSx & Storage Gateway | `24-fsx-storage-gateway.md` | FSx for Windows/Lustre/NetApp, Storage Gateway types |

### Part 5: Databases

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 25 | RDS - Relational Database Service | `25-rds.md` | Engine types, instance classes, Multi-AZ, read replicas, parameter groups, backups |
| 26 | Aurora | `26-aurora.md` | Aurora MySQL/PostgreSQL, serverless v2, global databases, cloning |
| 27 | DynamoDB | `27-dynamodb.md` | Tables, partition/sort keys, indexes (GSI/LSI), capacity modes, streams, DAX |
| 28 | ElastiCache | `28-elasticache.md` | Redis vs Memcached, cluster mode, replication groups, parameter groups |
| 29 | DocumentDB, Neptune & Other DBs | `29-other-databases.md` | DocumentDB, Neptune, Timestream, QLDB, Keyspaces |

### Part 6: CI/CD & Developer Tools

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 30 | CodeCommit | `30-codecommit.md` | Repositories, branches, pull requests, triggers, notifications |
| 31 | CodeBuild | `31-codebuild.md` | Build projects, buildspec.yml, environment, artifacts, caching |
| 32 | CodeDeploy | `32-codedeploy.md` | Applications, deployment groups, deployment configurations, appspec |
| 33 | CodePipeline | `33-codepipeline.md` | Pipeline structure, stages, actions, artifacts, manual approvals |
| 34 | CloudFormation | `34-cloudformation.md` | Templates, stacks, change sets, drift detection, nested stacks, StackSets |
| 35 | CDK - Cloud Development Kit | `35-cdk.md` | Constructs, stacks, apps, synthesis, bootstrapping |

### Part 7: Monitoring & Logging

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 36 | CloudWatch | `36-cloudwatch.md` | Metrics, alarms, dashboards, logs, log insights, custom metrics, agent |
| 37 | CloudTrail | `37-cloudtrail.md` | Trails, events, insights, organization trails, log file validation |
| 38 | X-Ray | `38-xray.md` | Traces, segments, subsegments, service map, sampling rules |
| 39 | AWS Config | `39-aws-config.md` | Rules, conformance packs, remediation, aggregators |

### Part 8: Security & Compliance

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 40 | KMS - Key Management Service | `40-kms.md` | CMKs, key policies, grants, encryption context, key rotation |
| 41 | Secrets Manager & Parameter Store | `41-secrets-and-parameters.md` | Secret rotation, versioning, Parameter Store tiers, hierarchies |
| 42 | WAF & Shield | `42-waf-shield.md` | Web ACLs, rules, rule groups, Shield Standard vs Advanced |
| 43 | GuardDuty & Security Hub | `43-guardduty-security-hub.md` | Threat detection, findings, security standards, integrations |
| 44 | Certificate Manager (ACM) | `44-acm.md` | Public/private certificates, validation, auto-renewal, integration |

### Part 9: Application Integration & Messaging

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 45 | SQS - Simple Queue Service | `45-sqs.md` | Standard vs FIFO, visibility timeout, DLQ, long polling, message attributes |
| 46 | SNS - Simple Notification Service | `46-sns.md` | Topics, subscriptions, filtering, fan-out pattern, FIFO topics |
| 47 | EventBridge | `47-eventbridge.md` | Event buses, rules, targets, schemas, archive & replay |
| 48 | Step Functions | `48-step-functions.md` | State machines, ASL, standard vs express, error handling, parallel states |
| 49 | API Gateway | `49-api-gateway.md` | REST vs HTTP vs WebSocket, stages, authorizers, throttling, usage plans |

### Part 10: Containers & Orchestration (Advanced)

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 50 | ECR - Elastic Container Registry | `50-ecr.md` | Repositories, lifecycle policies, image scanning, replication |
| 51 | Fargate Deep Dive | `51-fargate.md` | Task sizing, networking, storage, logging, cost optimization |

### Part 11: Analytics & Big Data

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 52 | Kinesis | `52-kinesis.md` | Data Streams, Firehose, Analytics, Video Streams |
| 53 | Athena & Glue | `53-athena-glue.md` | Serverless queries, crawlers, ETL jobs, data catalog |
| 54 | Redshift | `54-redshift.md` | Clusters, node types, Spectrum, serverless, data sharing |

### Part 12: Machine Learning & AI

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 55 | SageMaker | `55-sagemaker.md` | Notebooks, training jobs, endpoints, pipelines |
| 56 | AI Services | `56-ai-services.md` | Rekognition, Comprehend, Translate, Polly, Lex, Bedrock |

### Part 13: Migration & Transfer

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 57 | Migration Strategies | `57-migration-strategies.md` | 7 R's of migration, Migration Hub, DMS, SCT |
| 58 | Snow Family & DataSync | `58-snow-family-datasync.md` | Snowcone, Snowball, Snowmobile, DataSync |

### Part 14: Architecture & Best Practices

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 59 | Well-Architected Framework | `59-well-architected.md` | 6 pillars, design principles, review process |
| 60 | Real-World Architecture Patterns | `60-real-world-patterns.md` | Three-tier, microservices, serverless, event-driven architectures |
| 61 | Cost Optimization Strategies | `61-cost-optimization.md` | Right-sizing, spot instances, savings plans, architectural decisions |
| 62 | Disaster Recovery | `62-disaster-recovery.md` | Backup & restore, pilot light, warm standby, multi-site active-active |

---

## Quick Reference

### AWS Global Infrastructure
```
AWS Global Infrastructure
├── Regions (33+)
│   ├── Availability Zones (2-6 per region)
│   │   └── Data Centers (1+ per AZ)
│   └── Local Zones
├── Edge Locations (400+)
│   └── Regional Edge Caches
└── Wavelength Zones
```

### AWS Resource Hierarchy
```
AWS Organization (Management Account)
├── Root
│   ├── Organizational Unit (OU) - Production
│   │   ├── AWS Account - Prod-App1
│   │   ├── AWS Account - Prod-App2
│   │   └── AWS Account - Prod-Shared-Services
│   ├── Organizational Unit (OU) - Development
│   │   ├── AWS Account - Dev
│   │   └── AWS Account - Staging
│   ├── Organizational Unit (OU) - Security
│   │   ├── AWS Account - Log Archive
│   │   └── AWS Account - Security Tooling
│   └── Organizational Unit (OU) - Sandbox
│       └── AWS Account - Individual Developer Sandboxes
└── Service Control Policies (SCPs) applied at each level
```

---

## Learning Path Recommendation

1. **Weeks 1-2**: Chapters 1-5 (Foundation)
2. **Weeks 3-4**: Chapters 6-12 (Networking)
3. **Weeks 5-6**: Chapters 13-19 (Compute)
4. **Weeks 7-8**: Chapters 20-24 (Storage)
5. **Weeks 9-10**: Chapters 25-29 (Databases)
6. **Weeks 11-12**: Chapters 30-35 (CI/CD)
7. **Weeks 13-14**: Chapters 36-44 (Monitoring & Security)
8. **Weeks 15-16**: Chapters 45-51 (Integration & Containers)
9. **Weeks 17-18**: Chapters 52-62 (Analytics, ML, Architecture)

---

*Last Updated: May 2026*
