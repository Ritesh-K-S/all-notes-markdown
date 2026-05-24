# Chapter 60: Real-World Architecture Patterns

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Three-Tier Web Application](#part-1-three-tier-web-application)
- [Part 2: Serverless Architecture](#part-2-serverless-architecture)
- [Part 3: Microservices Architecture](#part-3-microservices-architecture)
- [Part 4: Data Lake Architecture](#part-4-data-lake-architecture)
- [Part 5: Event-Driven Architecture](#part-5-event-driven-architecture)
- [Part 6: Multi-Region Active-Active](#part-6-multi-region-active-active)
- [Part 7: Hybrid Cloud Architecture](#part-7-hybrid-cloud-architecture)
- [Part 8: Real-World Decision Guide](#part-8-real-world-decision-guide)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### Which Architecture Pattern Should I Use?

When starting a new project, the first question is: **"How should I structure my application on AWS?"** The answer depends on your team size, traffic patterns, and requirements.

**Quick decision guide:**
- 👤 **Just me or a small team, simple app** → Start with Serverless (API Gateway + Lambda + DynamoDB)
- 👥 **Small team, traditional web app** → Three-Tier (ALB + EC2 + RDS)
- 👨‍👩‍👧‍👦 **Large team, complex domain** → Microservices (ECS/EKS + per-service databases)
- 📊 **Analyzing lots of data** → Data Lake (S3 + Glue + Athena)
- 🌍 **Global users, 99.99%+ uptime** → Multi-Region Active-Active

> ⚡ **Rule of thumb:** Start simple, evolve as your needs grow. Don't over-architect.

This chapter covers proven AWS architecture patterns used in production by organizations of all sizes. Each pattern includes when to use it, key components, and trade-offs.

```
What you'll learn:
├── Three-tier web application
├── Serverless architecture
├── Microservices with containers
├── Data lake architecture
├── Event-driven architecture
├── Multi-region active-active
├── Hybrid cloud patterns
└── Decision guide: which pattern for which use case
```

---

## Part 1: Three-Tier Web Application

```
┌─────────────────────────────────────────────────────────────────────┐
│           THREE-TIER WEB APPLICATION                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                    ┌──────────────┐                                  │
│                    │ Route 53      │ DNS                              │
│                    └──────┬───────┘                                  │
│                           │                                           │
│                    ┌──────▼───────┐                                  │
│                    │ CloudFront    │ CDN (static assets + caching)  │
│                    └──────┬───────┘                                  │
│                           │                                           │
│           ┌───────────────▼───────────────┐                         │
│           │ Application Load Balancer      │ Layer 7 routing         │
│           └───────┬───────────────┬───────┘                         │
│                   │               │                                    │
│    ┌──────────────▼──┐  ┌────────▼──────────┐                      │
│    │ EC2 / ECS (AZ-a) │  │ EC2 / ECS (AZ-b)  │  Web/App tier       │
│    │ Auto Scaling      │  │ Auto Scaling       │                      │
│    └──────────┬───────┘  └────────┬──────────┘                      │
│               │                    │                                    │
│           ┌───▼────────────────────▼───┐                             │
│           │ RDS Multi-AZ (Primary)      │ Database tier              │
│           │ + Read Replicas             │                              │
│           └─────────────────────────────┘                             │
│                                                                       │
│ Static assets: S3 → CloudFront                                     │
│ Sessions: ElastiCache Redis                                         │
│ Secrets: Secrets Manager / Parameter Store                          │
│                                                                       │
│ When to use:                                                         │
│ ├── Traditional web applications                                  │
│ ├── Predictable traffic patterns                                  │
│ ├── Team familiar with servers/containers                        │
│ └── Need full control over runtime                               │
│                                                                       │
│ Trade-offs:                                                          │
│ ├── ✅ Well-understood pattern, easy to reason about             │
│ ├── ✅ Supports stateful applications                             │
│ ├── ❌ Must manage servers/patches                                │
│ └── ❌ Minimum cost even at zero traffic                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Serverless Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVERLESS ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│              ┌──────────────┐                                        │
│              │ API Gateway   │ REST/HTTP API                         │
│              └──────┬───────┘                                        │
│                     │                                                  │
│              ┌──────▼───────┐                                        │
│              │ Lambda        │ Business logic                        │
│              └──┬────┬──┬──┘                                        │
│                 │    │  │                                              │
│    ┌────────────▼┐ ┌─▼──▼────────┐  ┌────────────┐                │
│    │ DynamoDB     │ │ S3           │  │ SQS/SNS    │                │
│    │ (data)       │ │ (files)      │  │ (messaging)│                │
│    └─────────────┘ └─────────────┘  └────────────┘                │
│                                                                       │
│ Auth: Cognito → API Gateway authorizer                             │
│ Frontend: S3 + CloudFront (SPA: React/Vue/Angular)                │
│ Monitoring: CloudWatch + X-Ray                                      │
│ CI/CD: SAM / CDK + CodePipeline                                    │
│                                                                       │
│ When to use:                                                         │
│ ├── Variable/unpredictable traffic (including zero)             │
│ ├── API backends, data processing, event handling               │
│ ├── Startups (pay only for what you use)                       │
│ ├── Rapid prototyping                                            │
│ └── Event-driven workloads                                      │
│                                                                       │
│ Trade-offs:                                                          │
│ ├── ✅ Zero cost at zero traffic                                  │
│ ├── ✅ No server management                                      │
│ ├── ✅ Auto-scales to any load                                   │
│ ├── ❌ Cold starts (mitigated with provisioned concurrency)    │
│ ├── ❌ 15-minute Lambda timeout                                  │
│ └── ❌ Vendor lock-in                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Microservices Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│           MICROSERVICES WITH CONTAINERS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                    ┌──────────────┐                                  │
│                    │ ALB / API GW  │                                  │
│                    └──────┬───────┘                                  │
│                           │ path-based routing                       │
│          ┌────────────────┼────────────────┐                        │
│          │                │                │                          │
│    ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐                 │
│    │ User Svc   │   │ Order Svc  │   │ Payment Svc│                 │
│    │ (ECS/EKS)  │   │ (ECS/EKS)  │   │ (ECS/EKS)  │                 │
│    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘                 │
│          │                │                │                          │
│    ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐                 │
│    │ DynamoDB   │   │ RDS        │   │ DynamoDB   │                 │
│    └───────────┘   └───────────┘   └───────────┘                 │
│                                                                       │
│ Service mesh: App Mesh (optional)                                  │
│ Service discovery: Cloud Map                                        │
│ Async communication: SQS/SNS/EventBridge between services       │
│ Tracing: X-Ray (distributed tracing across services)              │
│ CI/CD: CodePipeline per service (independent deployments)        │
│                                                                       │
│ When to use:                                                         │
│ ├── Large teams (each team owns a service)                      │
│ ├── Independent scaling per component                            │
│ ├── Polyglot: different languages per service                   │
│ └── Complex domain with clear boundaries                        │
│                                                                       │
│ Trade-offs:                                                          │
│ ├── ✅ Independent deployment and scaling                        │
│ ├── ✅ Team autonomy                                              │
│ ├── ❌ Operational complexity (many services to manage)        │
│ ├── ❌ Distributed system challenges (eventual consistency)   │
│ └── ❌ Requires mature DevOps practices                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Data Lake Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│           DATA LAKE ON AWS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Ingestion:                                                           │
│ ├── Kinesis Data Streams / Firehose (streaming)               │
│ ├── DMS (database replication)                                  │
│ ├── DataSync / Snow Family (bulk/on-prem)                     │
│ └── S3 direct upload / Transfer Family                         │
│           │                                                          │
│           ▼                                                          │
│ ┌─────────────────────────────────────┐                             │
│ │ S3 Data Lake                         │                             │
│ │ ├── Raw zone (s3://lake/raw/)       │                             │
│ │ ├── Processed zone (s3://lake/proc/)│                             │
│ │ └── Curated zone (s3://lake/curated/)│                            │
│ └──────────┬──────────────────────────┘                             │
│            │                                                          │
│ Catalog & ETL:                                                      │
│ ├── Glue Crawler → Data Catalog (schema discovery)            │
│ ├── Glue ETL Jobs (transform raw → processed → curated)     │
│ └── Lake Formation (governance, fine-grained access control) │
│            │                                                          │
│ Analytics:                                                           │
│ ├── Athena (serverless SQL on S3)                               │
│ ├── Redshift Spectrum (extend warehouse to S3)                │
│ ├── EMR (Spark, Hive for big data processing)                 │
│ ├── QuickSight (BI dashboards)                                 │
│ └── SageMaker (ML on data lake data)                          │
│                                                                       │
│ When to use:                                                         │
│ ├── Centralize data from multiple sources                      │
│ ├── Analytics, BI, ML on large datasets                        │
│ ├── Schema-on-read (don't know queries upfront)              │
│ └── Cost-effective storage of raw data                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Event-Driven Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENT-DRIVEN ARCHITECTURE                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────┐    ┌────────────────┐    ┌──────────────────┐        │
│ │ Producers │ →  │ EventBridge     │ →  │ Consumers         │        │
│ │           │    │ (Event Bus)     │    │                    │        │
│ │ S3 events │    │ Rules match     │    │ Lambda functions  │        │
│ │ API calls │    │ and route       │    │ Step Functions    │        │
│ │ SaaS apps │    │ events          │    │ SQS queues        │        │
│ │ Custom    │    │                 │    │ SNS topics        │        │
│ └──────────┘    └────────────────┘    └──────────────────┘        │
│                                                                       │
│ Example flows:                                                       │
│ ├── S3 upload → EventBridge → Lambda (process file)           │
│ ├── Order placed → EventBridge → fan-out to:                  │
│ │   ├── Lambda (send confirmation email)                      │
│ │   ├── SQS → Lambda (update inventory)                      │
│ │   └── Step Functions (fulfillment workflow)                 │
│ └── Scheduled → EventBridge rule → Lambda (nightly cleanup) │
│                                                                       │
│ When to use:                                                         │
│ ├── Loose coupling between components                          │
│ ├── React to changes in real-time                              │
│ ├── Fan-out processing (one event, many consumers)           │
│ └── Asynchronous processing                                    │
│                                                                       │
│ Trade-offs:                                                          │
│ ├── ✅ Loose coupling, high scalability                         │
│ ├── ✅ Easy to add new consumers                                │
│ ├── ❌ Eventual consistency                                     │
│ ├── ❌ Harder to debug (distributed flow)                      │
│ └── ❌ Need idempotent consumers                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Multi-Region Active-Active

```
┌─────────────────────────────────────────────────────────────────────┐
│           MULTI-REGION ACTIVE-ACTIVE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│              ┌──────────────────┐                                    │
│              │ Route 53          │ Latency-based / Weighted routing │
│              └────┬─────────┬──┘                                    │
│                   │         │                                         │
│         ┌─────────▼───┐  ┌─▼───────────┐                           │
│         │ US-EAST-1    │  │ EU-WEST-1    │                           │
│         │              │  │              │                           │
│         │ CloudFront   │  │ CloudFront   │                           │
│         │ ALB          │  │ ALB          │                           │
│         │ ECS/EKS      │  │ ECS/EKS      │                           │
│         │ Aurora Global │  │ Aurora Global │                           │
│         │ (Primary)    │  │ (Secondary)  │                           │
│         │ DynamoDB GT  │  │ DynamoDB GT  │                           │
│         │ ElastiCache  │  │ ElastiCache  │                           │
│         └──────────────┘  └──────────────┘                           │
│                                                                       │
│ Key components:                                                      │
│ ├── Route 53: Latency-based routing + health checks             │
│ ├── Aurora Global Database: < 1s replication, RPO < 1s        │
│ ├── DynamoDB Global Tables: Multi-region, multi-active         │
│ ├── S3 Cross-Region Replication                                  │
│ ├── ECR Cross-Region Replication                                │
│ └── Parameter Store / Secrets Manager replication              │
│                                                                       │
│ When to use:                                                         │
│ ├── Strict latency requirements for global users               │
│ ├── Regulatory: data must stay in specific regions             │
│ ├── Near-zero RPO/RTO requirements                              │
│ └── Business-critical applications (99.99%+ availability)    │
│                                                                       │
│ Trade-offs:                                                          │
│ ├── ✅ Lowest latency, highest availability                     │
│ ├── ❌ 2x+ cost (duplicate infrastructure)                     │
│ ├── ❌ Data consistency challenges (conflict resolution)      │
│ └── ❌ Complex operations and deployment                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Hybrid Cloud Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│           HYBRID CLOUD PATTERNS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern A: Burst to Cloud                                           │
│ On-prem handles baseline → AWS handles spikes                     │
│ ├── VPN / Direct Connect for connectivity                       │
│ ├── Auto Scaling in AWS for burst capacity                     │
│ └── Use case: Seasonal traffic spikes                           │
│                                                                       │
│ Pattern B: Backup & DR in Cloud                                    │
│ On-prem runs production → AWS as DR site                         │
│ ├── AWS Backup / DMS for data replication                      │
│ ├── Pilot light or warm standby in AWS                         │
│ └── Use case: Cost-effective disaster recovery                 │
│                                                                       │
│ Pattern C: Data Processing in Cloud                                │
│ On-prem generates data → process/analyze in AWS                  │
│ ├── DataSync / Kinesis to move data                             │
│ ├── Analytics: Athena, EMR, Redshift                           │
│ └── Use case: ML/analytics without on-prem investment         │
│                                                                       │
│ Connectivity options:                                                │
│ ├── Site-to-Site VPN: Quick, encrypted, over internet         │
│ ├── Direct Connect: Dedicated line, consistent latency        │
│ ├── Transit Gateway: Hub-spoke for multiple VPCs + VPN/DX   │
│ └── Outposts: AWS infrastructure in your data center          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Real-World Decision Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHICH PATTERN TO USE?                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Requirement                  │ Recommended Pattern                  │
│ ─────────────────────────────┼──────────────────────────────────────│
│ Simple web app, small team  │ Three-Tier (ALB + EC2 + RDS)       │
│ Variable traffic, APIs      │ Serverless (API GW + Lambda + DDB) │
│ Large team, complex domain  │ Microservices (ECS/EKS + per-svc DB)│
│ Analytics on large datasets │ Data Lake (S3 + Glue + Athena)     │
│ Real-time event processing  │ Event-Driven (EventBridge + Lambda)│
│ Global users, 99.99%+ SLA   │ Multi-Region Active-Active          │
│ Keep some workloads on-prem │ Hybrid Cloud (VPN/DX + AWS)        │
│ Startup, unknown scale      │ Serverless (zero cost at zero use) │
│ Legacy migration             │ Three-Tier → evolve to serverless │
│                                                                       │
│ ⚡ Start simple, evolve as needed                                  │
│ ⚡ Don't over-architect — match pattern to team size              │
│ ⚡ Combine patterns: serverless APIs + event-driven processing  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Architecture Patterns Quick Reference:
├── Three-Tier: ALB → EC2/ECS → RDS (classic, simple)
├── Serverless: API GW → Lambda → DynamoDB (zero idle cost)
├── Microservices: ALB → ECS/EKS services → per-service DB
├── Data Lake: Ingest → S3 zones → Glue ETL → Athena/Redshift
├── Event-Driven: Producers → EventBridge → Lambda/SQS consumers
├── Multi-Region: Route 53 → duplicated infra per region
├── Hybrid: VPN/DX → burst/DR/analytics in cloud
├── ⚡ Start simple, evolve as complexity grows
├── ⚡ Serverless for startups and variable traffic
├── ⚡ Microservices only with large teams + mature DevOps
└── ⚡ Multi-region only when business requires 99.99%+
```

---

## What's Next?

In **Chapter 61: Cost Optimization**, we'll cover AWS Cost Explorer, Budgets, Savings Plans, Reserved Instances, Spot Instances, and practical strategies to reduce your AWS bill.
