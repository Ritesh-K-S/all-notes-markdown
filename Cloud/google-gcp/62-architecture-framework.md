# Chapter 62 — Google Cloud Architecture Framework

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Architecture Framework Fundamentals](#part-1--architecture-framework-fundamentals)
- [Part 2: System Design Principles](#part-2--system-design-principles)
- [Part 3: Operational Excellence](#part-3--operational-excellence)
- [Part 4: Security, Privacy & Compliance](#part-4--security-privacy--compliance)
- [Part 5: Reliability & High Availability](#part-5--reliability--high-availability)
- [Part 6: Performance Optimization](#part-6--performance-optimization)
- [Part 7: Cost Optimization](#part-7--cost-optimization)
- [Part 8: Scalability Patterns](#part-8--scalability-patterns)
- [Part 9: Networking Architecture](#part-9--networking-architecture)
- [Part 10: Data Architecture](#part-10--data-architecture)
- [Part 11: Compute Architecture](#part-11--compute-architecture)
- [Part 12: CI/CD & DevOps Architecture](#part-12--cicd--devops-architecture)
- [Part 13: Hybrid & Multi-Cloud Architecture](#part-13--hybrid--multi-cloud-architecture)
- [Part 14: Well-Architected Review Checklist](#part-14--well-architected-review-checklist)
- [Part 15: Architecture Decision Records](#part-15--architecture-decision-records)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

The Google Cloud Architecture Framework provides a set of best practices, design principles, and recommendations for building reliable, secure, performant, and cost-effective systems on GCP. Analogous to AWS Well-Architected Framework and Azure Well-Architected Framework, it organizes guidance across key pillars: operational excellence, security, reliability, performance, and cost optimization. This chapter distills the framework into actionable guidelines with GCP-specific recommendations.

---

## Part 1 — Architecture Framework Fundamentals

### Framework Pillars

```
┌────────────────────────────────────────────────────────────────────┐
│         GOOGLE CLOUD ARCHITECTURE FRAMEWORK                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │     │
│  │  │ OPERATIONAL │  │  SECURITY   │  │ RELIABILITY │     │     │
│  │  │ EXCELLENCE  │  │  PRIVACY &  │  │ & HIGH      │     │     │
│  │  │             │  │  COMPLIANCE │  │ AVAILABILITY│     │     │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │     │
│  │                                                            │     │
│  │  ┌─────────────┐  ┌─────────────┐                        │     │
│  │  │ PERFORMANCE │  │    COST     │                        │     │
│  │  │ OPTIMIZATION│  │ OPTIMIZATION│                        │     │
│  │  │             │  │             │                        │     │
│  │  └─────────────┘  └─────────────┘                        │     │
│  │                                                            │     │
│  │  Foundation: System Design Principles                     │     │
│  │  • Design for change                                     │     │
│  │  • Automate everything                                   │     │
│  │  • Design for failure                                    │     │
│  │  • Think about scale from the start                     │     │
│  │  • Use managed services when possible                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Framework Comparison

| Pillar | GCP Architecture Framework | AWS Well-Architected | Azure Well-Architected |
|--------|---------------------------|---------------------|----------------------|
| Operations | Operational Excellence | Operational Excellence | Operational Excellence |
| Security | Security, Privacy & Compliance | Security | Security |
| Reliability | Reliability | Reliability | Reliability |
| Performance | Performance Optimization | Performance Efficiency | Performance Efficiency |
| Cost | Cost Optimization | Cost Optimization | Cost Optimization |
| Sustainability | — | Sustainability | — |

---

## Part 2 — System Design Principles

### Core Design Principles

```
┌────────────────────────────────────────────────────────────────────┐
│         SYSTEM DESIGN PRINCIPLES                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Design for HIGH AVAILABILITY                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Deploy across multiple zones (minimum 2)              │     │
│  │  • Use regional resources (not zonal) when possible     │     │
│  │  • Implement health checks and auto-healing             │     │
│  │  • No single points of failure                          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  2. Design for SCALABILITY                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Horizontal scaling > vertical scaling                 │     │
│  │  • Stateless services (store state in DB/cache)         │     │
│  │  • Use autoscaling for compute (MIG, Cloud Run, GKE)   │     │
│  │  • Decouple with Pub/Sub for async processing           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  3. Design for SECURITY                                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Least privilege (IAM)                                 │     │
│  │  • Defense in depth (multiple layers)                   │     │
│  │  • Zero trust networking (BeyondCorp)                   │     │
│  │  • Encrypt everywhere (at rest + in transit)            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  4. Design for RESILIENCE                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Expect failures (design for graceful degradation)    │     │
│  │  • Circuit breakers, retries with exponential backoff   │     │
│  │  • Chaos engineering (test failure scenarios)           │     │
│  │  • Automated recovery (self-healing infrastructure)    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  5. Design for OBSERVABILITY                                       │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Metrics, logs, traces (three pillars)                │     │
│  │  • Centralized logging (Cloud Logging)                  │     │
│  │  • Distributed tracing (Cloud Trace)                    │     │
│  │  • Alerting on SLOs, not just metrics                  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 3 — Operational Excellence

### Key Practices

```
┌────────────────────────────────────────────────────────────────────┐
│         OPERATIONAL EXCELLENCE                                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Infrastructure as Code (IaC):                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Terraform for all infrastructure                      │     │
│  │  • Version control (Git) for all configs                │     │
│  │  • No manual console changes in production              │     │
│  │  • State management (GCS backend for Terraform state)   │     │
│  │  • Module reuse across environments                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  CI/CD:                                                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Automated testing (unit, integration, e2e)           │     │
│  │  • Cloud Build → Artifact Registry → Cloud Deploy      │     │
│  │  • Environment promotion (dev → staging → prod)        │     │
│  │  • Canary deployments with traffic splitting            │     │
│  │  • Rollback automation                                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Monitoring & Alerting:                                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • SLI/SLO-based alerting (not threshold-only)          │     │
│  │  • Cloud Monitoring dashboards per service              │     │
│  │  • Uptime checks for all public endpoints              │     │
│  │  • Alerting channels: PagerDuty, Slack, email          │     │
│  │  • Runbooks for common incidents                        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Incident Management:                                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • On-call rotation with escalation policies            │     │
│  │  • Postmortem culture (blameless retrospectives)        │     │
│  │  • Error budgets tied to release velocity               │     │
│  │  • Game days (practice failure scenarios)               │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 4 — Security, Privacy & Compliance

### Security Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│         SECURITY ARCHITECTURE                                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Identity & Access:                                                │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Least privilege IAM roles (avoid basic roles)         │     │
│  │  • Service accounts with minimal permissions            │     │
│  │  • Workload Identity for GKE (no SA keys)              │     │
│  │  • IAM Conditions for time/resource-based access       │     │
│  │  • Organization policies to enforce guardrails         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Network Security:                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Private GKE clusters (no public IPs on nodes)        │     │
│  │  • VPC Service Controls (data exfiltration prevention) │     │
│  │  • Cloud Armor (WAF, DDoS protection)                  │     │
│  │  • Private Google Access (no public internet for APIs) │     │
│  │  • Hierarchical firewall policies                      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Data Protection:                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Encryption at rest (Google-managed or CMEK)          │     │
│  │  • Encryption in transit (TLS 1.3 everywhere)          │     │
│  │  • Cloud KMS for key management                        │     │
│  │  • Secret Manager for credentials                      │     │
│  │  • DLP API for PII detection and masking               │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Detection & Response:                                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Security Command Center (threat detection)           │     │
│  │  • Audit logging (Admin Activity + Data Access)        │     │
│  │  • Chronicle SIEM (security analytics)                 │     │
│  │  • Web Security Scanner (vulnerability scanning)       │     │
│  │  • Binary Authorization (container image signing)      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 5 — Reliability & High Availability

### Reliability Design

```
┌────────────────────────────────────────────────────────────────────┐
│         RELIABILITY ARCHITECTURE                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Availability Targets:                                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  SLA        │ Downtime/Year │ Pattern                    │     │
│  │  99.9%      │ 8.76 hours    │ Single region, multi-zone │     │
│  │  99.95%     │ 4.38 hours    │ Regional with auto-failover│    │
│  │  99.99%     │ 52.56 minutes │ Multi-region active-active│     │
│  │  99.999%    │ 5.26 minutes  │ Global LB + multi-region  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Multi-Zone (standard):                                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Region: us-central1                                      │     │
│  │  ┌────────┐  ┌────────┐  ┌────────┐                    │     │
│  │  │ Zone A │  │ Zone B │  │ Zone C │                    │     │
│  │  │ App    │  │ App    │  │ App    │                    │     │
│  │  │ replica│  │ replica│  │ replica│                    │     │
│  │  └────────┘  └────────┘  └────────┘                    │     │
│  │       │           │           │                          │     │
│  │       └───────────┼───────────┘                          │     │
│  │                   ▼                                       │     │
│  │          Internal Load Balancer                          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Multi-Region (highest availability):                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  ┌────────────────┐   ┌────────────────┐                │     │
│  │  │ us-central1    │   │ europe-west1   │                │     │
│  │  │ App + DB       │   │ App + DB       │                │     │
│  │  │ (primary)      │   │ (standby/read) │                │     │
│  │  └────────┬───────┘   └───────┬────────┘                │     │
│  │           └───────────────────┘                          │     │
│  │                   │                                       │     │
│  │          Global HTTP(S) Load Balancer                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Key services for HA:                                              │
│  • Cloud SQL: multi-zone HA, cross-region read replicas          │
│  • Cloud Spanner: built-in multi-region (99.999% SLA)            │
│  • GKE: regional clusters, pod disruption budgets                │
│  • Cloud Run: regional (auto multi-zone)                         │
│  • GCS: multi-region / dual-region buckets                       │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 6 — Performance Optimization

### Performance Best Practices

```
┌────────────────────────────────────────────────────────────────────┐
│         PERFORMANCE OPTIMIZATION                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Caching:                                                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Level 1: CDN (Cloud CDN) — static assets, API responses│     │
│  │  Level 2: In-memory (Memorystore Redis) — session, query│     │
│  │  Level 3: Application (local cache) — config, metadata  │     │
│  │                                                            │     │
│  │  Rule: cache as close to the user as possible           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Database Performance:                                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Right-size instances (don't over-provision)          │     │
│  │  • Read replicas for read-heavy workloads               │     │
│  │  • Connection pooling (pgbouncer, ProxySQL)            │     │
│  │  • Query optimization (EXPLAIN, indexes)                │     │
│  │  • BigQuery: partitioning + clustering                  │     │
│  │  • Spanner: hot-spot-free key design                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Compute Performance:                                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Choose right machine type (CPU vs memory vs GPU)     │     │
│  │  • Use custom machine types for exact needs             │     │
│  │  • Local SSD for I/O-intensive workloads               │     │
│  │  • Preemptible/Spot VMs for batch processing           │     │
│  │  • Cloud Run: min instances for cold start avoidance   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Network Performance:                                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Premium tier networking (Google backbone)             │     │
│  │  • Regional affinity (colocate compute + storage)      │     │
│  │  • Cloud CDN for global content delivery               │     │
│  │  • gRPC instead of REST for inter-service comms        │     │
│  │  • VPC Flow Logs for bottleneck analysis               │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 7 — Cost Optimization

### Cost Best Practices

```
┌────────────────────────────────────────────────────────────────────┐
│         COST OPTIMIZATION                                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Compute savings:                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Committed Use Discounts (CUDs): 1yr/3yr commitments  │     │
│  │    → Up to 57% savings                                   │     │
│  │  • Sustained Use Discounts (automatic): up to 30%       │     │
│  │  • Spot/Preemptible VMs: up to 91% savings              │     │
│  │  • Right-sizing (Recommender API suggestions)           │     │
│  │  • Custom machine types (pay only for what you need)    │     │
│  │  • Autoscaling (scale to zero when idle)                │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Storage savings:                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Lifecycle policies (auto-transition storage classes)  │     │
│  │  • Autoclass (automatic class management)               │     │
│  │  • Delete unused snapshots and images                   │     │
│  │  • Compress data before storage                         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Database savings:                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Scale down dev/staging instances                     │     │
│  │  • Use read replicas instead of larger primary          │     │
│  │  • BigQuery on-demand → flat-rate for predictable cost │     │
│  │  • Cloud SQL: stop instances outside business hours    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Network savings:                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Standard tier networking where latency doesn't matter│     │
│  │  • Cloud CDN (reduce origin egress)                     │     │
│  │  • Keep traffic in same region (free intra-zone)       │     │
│  │  • Private Google Access (avoid egress charges)        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Governance:                                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Budget alerts (50%, 90%, 100% thresholds)            │     │
│  │  • Cost allocation labels on all resources              │     │
│  │  • Billing export to BigQuery for analysis              │     │
│  │  • FinOps team / cost review cadence                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 8 — Scalability Patterns

### Scaling Strategies

```
┌────────────────────────────────────────────────────────────────────┐
│         SCALABILITY PATTERNS                                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Pattern 1: Horizontal Autoscaling                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  ┌──────┐ ┌──────┐ ┌──────┐     ┌──────┐ ┌──────┐    │     │
│  │  │ Pod  │ │ Pod  │ │ Pod  │ ··· │ Pod  │ │ Pod  │    │     │
│  │  └──────┘ └──────┘ └──────┘     └──────┘ └──────┘    │     │
│  │  min=2        scale based on CPU/custom       max=20  │     │
│  │                                                        │     │
│  │  Services: GKE HPA, Cloud Run, MIG, App Engine       │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Pattern 2: Queue-Based Load Leveling                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Producers → [Pub/Sub Queue] → Workers (autoscaled)    │     │
│  │                                                        │     │
│  │  Benefits: absorb spikes, process at own pace         │     │
│  │  Workers scale based on queue depth                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Pattern 3: Database Read Scaling                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  App → [Primary DB] ← writes                          │     │
│  │  App → [Read Replica 1] ← reads                       │     │
│  │  App → [Read Replica 2] ← reads                       │     │
│  │  App → [Memorystore Cache] ← hot data reads           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Pattern 4: CQRS (Command Query Responsibility Segregation)      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Writes → [Cloud SQL] → [Pub/Sub] → [BigQuery/views]  │     │
│  │  Reads  → [BigQuery / read-optimized store]            │     │
│  │                                                        │     │
│  │  Separate read and write paths for different scaling   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Pattern 5: Event-Driven Scaling                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Event → Eventarc → Cloud Run (auto-scale 0→N)        │     │
│  │  Event → Pub/Sub → Cloud Functions (auto-scale 0→N)   │     │
│  │                                                        │     │
│  │  Zero-to-N scaling, pay only when processing events   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 9 — Networking Architecture

### Network Design

```
┌────────────────────────────────────────────────────────────────────┐
│         NETWORKING ARCHITECTURE                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Hub-and-Spoke VPC Design:                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │              ┌──────────────┐                            │     │
│  │              │   Hub VPC    │                            │     │
│  │              │ (shared svcs)│                            │     │
│  │              │ DNS, NAT,    │                            │     │
│  │              │ monitoring   │                            │     │
│  │              └──────┬───────┘                            │     │
│  │           ┌─────────┼─────────┐                          │     │
│  │           │ VPC     │ VPC     │                          │     │
│  │           │ Peering │ Peering │                          │     │
│  │      ┌────▼────┐ ┌─▼──────┐ ┌▼───────┐                 │     │
│  │      │ Prod    │ │ Dev    │ │ Data   │                 │     │
│  │      │ VPC     │ │ VPC   │ │ VPC    │                 │     │
│  │      └─────────┘ └────────┘ └────────┘                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Alternative: Shared VPC                                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Host project owns VPC                                    │     │
│  │  Service projects share subnets                          │     │
│  │  Centralized network management                          │     │
│  │  Best for organizations with dedicated network team     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Recommended networking stack:                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Internet → Cloud Armor → Global HTTP(S) LB → Backend  │     │
│  │  Internal → Internal TCP/UDP LB → Service              │     │
│  │  GCS access → Private Google Access                    │     │
│  │  On-prem → Cloud Interconnect / HA VPN                 │     │
│  │  DNS → Cloud DNS (private + public zones)              │     │
│  │  NAT → Cloud NAT (for outbound from private subnets)  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 10 — Data Architecture

### Data Platform Design

```
┌────────────────────────────────────────────────────────────────────┐
│         DATA ARCHITECTURE                                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Choosing the right database:                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Need                        │ Service                   │     │
│  │  ──────────────────────────── │ ────────────────────────  │     │
│  │  Relational, single region   │ Cloud SQL                 │     │
│  │  Relational, global scale    │ Cloud Spanner             │     │
│  │  Relational, PG compatible   │ AlloyDB                   │     │
│  │  Document store              │ Firestore                 │     │
│  │  Wide-column, time-series    │ Bigtable                  │     │
│  │  In-memory cache             │ Memorystore               │     │
│  │  Analytics warehouse         │ BigQuery                  │     │
│  │  Search                      │ Elasticsearch / Vertex AI │     │
│  │  Graph                       │ Spanner (graph)           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Data lakehouse pattern:                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Raw data → GCS (data lake)                              │     │
│  │       │                                                    │     │
│  │       ├── ETL: Dataflow / Dataproc                       │     │
│  │       ▼                                                    │     │
│  │  Curated data → BigQuery (data warehouse)               │     │
│  │       │                                                    │     │
│  │       ├── Analytics: BigQuery SQL + BI Engine            │     │
│  │       ├── ML: Vertex AI (training on BQ data)           │     │
│  │       └── Dashboards: Looker Studio                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 11 — Compute Architecture

### Choosing the Right Compute

```
┌────────────────────────────────────────────────────────────────────┐
│         COMPUTE DECISION TREE                                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  What are you running?                                             │
│       │                                                             │
│       ├── Containers?                                              │
│       │    ├── Many microservices, need K8s? → GKE Autopilot     │
│       │    ├── Simple stateless containers? → Cloud Run           │
│       │    └── Batch container jobs? → Cloud Run Jobs            │
│       │                                                             │
│       ├── Functions (event-driven)?                                │
│       │    └── Cloud Functions (Gen 2)                            │
│       │                                                             │
│       ├── Full VM control needed?                                 │
│       │    ├── Stable workload → Compute Engine + CUD            │
│       │    ├── Batch processing → Spot VMs                       │
│       │    └── Auto-scale group → Managed Instance Group         │
│       │                                                             │
│       ├── Web application?                                        │
│       │    ├── Simple (PaaS) → App Engine                        │
│       │    └── Container (PaaS) → Cloud Run                      │
│       │                                                             │
│       └── ML training?                                             │
│            └── Vertex AI (custom jobs with GPU/TPU)               │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Service     │ Control │ Scaling     │ Mgmt Overhead    │     │
│  │  ────────────┼─────────┼─────────── │ ──────────────── │     │
│  │  GCE         │ Full    │ Manual/MIG │ Highest          │     │
│  │  GKE         │ High    │ Auto (HPA) │ Medium           │     │
│  │  Cloud Run   │ Medium  │ Auto (0→N) │ Low              │     │
│  │  App Engine  │ Low     │ Auto       │ Lowest           │     │
│  │  Cloud Funcs │ Lowest  │ Auto (0→N) │ Lowest           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 12 — CI/CD & DevOps Architecture

### Deployment Pipeline

```
┌────────────────────────────────────────────────────────────────────┐
│         CI/CD ARCHITECTURE                                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  Developer                                                │     │
│  │  └── git push                                             │     │
│  │       │                                                    │     │
│  │       ▼                                                    │     │
│  │  Cloud Build (CI)                                         │     │
│  │  ├── Lint & static analysis                              │     │
│  │  ├── Unit tests                                           │     │
│  │  ├── Build container image                               │     │
│  │  ├── Push to Artifact Registry                           │     │
│  │  ├── Security scan (vulnerability scanning)              │     │
│  │  ├── Integration tests                                    │     │
│  │  └── Binary Authorization (sign image)                   │     │
│  │       │                                                    │     │
│  │       ▼                                                    │     │
│  │  Cloud Deploy (CD)                                        │     │
│  │  ├── Deploy to dev → auto-promote                        │     │
│  │  ├── Deploy to staging → run e2e tests                  │     │
│  │  ├── Deploy to prod-canary (10% traffic)                │     │
│  │  │    └── Monitor metrics (error rate, latency)         │     │
│  │  ├── Deploy to prod (100% traffic)                      │     │
│  │  └── Rollback if error budget exceeded                  │     │
│  │                                                            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  GitOps alternative (for GKE):                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Git repo (K8s manifests) → Config Sync → GKE clusters  │     │
│  │  Single source of truth in Git                           │     │
│  │  Auto-sync on commit (reconciliation loop)              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 13 — Hybrid & Multi-Cloud Architecture

### Hybrid Patterns

```
┌────────────────────────────────────────────────────────────────────┐
│         HYBRID & MULTI-CLOUD                                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Pattern 1: Tiered Hybrid                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  On-Prem: legacy systems, regulated data               │     │
│  │  GCP: new workloads, analytics, AI/ML                   │     │
│  │  Connection: Cloud Interconnect                          │     │
│  │  Gradual migration: on-prem → GCP over time            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Pattern 2: Burst to Cloud                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  On-Prem: baseline capacity (always running)            │     │
│  │  GCP: overflow capacity (peak periods)                  │     │
│  │  Use: GKE + Anthos for consistent K8s experience       │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Pattern 3: Multi-Cloud Active-Active                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  GCP: primary for analytics + AI (BigQuery, Vertex AI)  │     │
│  │  AWS: primary for compute + specific services           │     │
│  │  Anthos: consistent management across both              │     │
│  │  Terraform: single IaC tool for both clouds             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  GCP tools for hybrid/multi-cloud:                                │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Anthos: manage K8s clusters anywhere                 │     │
│  │  • Config Sync: GitOps for multi-cluster                │     │
│  │  • Service Mesh (Anthos): unified observability         │     │
│  │  • BigQuery Omni: query data in AWS S3/Azure Blob      │     │
│  │  • Apigee: API management across environments          │     │
│  │  • Cloud Interconnect: dedicated on-prem connectivity  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Well-Architected Review Checklist

### Review Checklist by Pillar

```
┌────────────────────────────────────────────────────────────────────┐
│         ARCHITECTURE REVIEW CHECKLIST                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  OPERATIONAL EXCELLENCE                                            │
│  □ All infrastructure defined in Terraform/IaC                    │
│  □ CI/CD pipeline for every service                               │
│  □ Monitoring dashboards for every service                        │
│  □ Alerting based on SLOs (not just raw metrics)                 │
│  □ Runbooks for top 10 alerts                                     │
│  □ Incident management process defined                            │
│  □ Postmortem culture established                                  │
│                                                                      │
│  SECURITY                                                           │
│  □ No basic IAM roles (Owner/Editor) in production               │
│  □ Service accounts use Workload Identity (no exported keys)     │
│  □ VPC Service Controls on sensitive projects                    │
│  □ Audit logging enabled for all services                        │
│  □ CMEK encryption for regulated data                            │
│  □ Secret Manager for all credentials                            │
│  □ Cloud Armor on all public-facing services                     │
│  □ Organization policies enforced                                 │
│                                                                      │
│  RELIABILITY                                                        │
│  □ Services deployed across multiple zones                       │
│  □ Health checks configured for all backends                     │
│  □ Autoscaling configured with appropriate limits                │
│  □ Database backups tested (restore drill quarterly)             │
│  □ DR plan documented and tested                                  │
│  □ Circuit breakers on external service calls                    │
│  □ Error budgets defined for each service                        │
│                                                                      │
│  PERFORMANCE                                                        │
│  □ CDN enabled for static content                                │
│  □ Caching strategy defined (CDN + Redis + app-level)           │
│  □ Database queries optimized (no N+1 queries)                  │
│  □ Connection pooling for database connections                   │
│  □ Load testing performed before launch                          │
│  □ Right-sized instances (not over-provisioned)                  │
│                                                                      │
│  COST                                                               │
│  □ Budget alerts configured (50%, 90%, 100%)                     │
│  □ Labels on all resources (team, env, service)                  │
│  □ Billing export to BigQuery enabled                            │
│  □ CUDs purchased for stable workloads                           │
│  □ Spot VMs used for fault-tolerant batch workloads             │
│  □ Storage lifecycle policies configured                         │
│  □ Dev/staging environments scaled down or auto-stopped         │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 15 — Architecture Decision Records

### ADR Template

```markdown
# ADR-001: Use Cloud Run for API Services

## Status
Accepted

## Context
We need a compute platform for our REST API microservices.
Requirements: auto-scaling, minimal ops overhead, container support.

## Decision
Use Cloud Run for all stateless API services.

## Reasons
- Auto-scales to zero (cost savings for low-traffic services)
- Managed TLS, custom domains, traffic splitting
- Container-based (portable, no lock-in)
- No cluster management (vs GKE)
- Generous free tier for development

## Consequences
- Cold starts for infrequently accessed services (mitigate with min instances)
- 60-minute request timeout limit
- No WebSocket support (use GKE for WebSocket services)
- Stateless only (use Memorystore/Cloud SQL for state)

## Alternatives Considered
- GKE Autopilot: more operational overhead, but needed for WebSockets
- App Engine: less container flexibility
- Cloud Functions: event-driven only, limited concurrency control
```

### Common ADRs for GCP Projects

| ADR | Decision | Key Rationale |
|-----|----------|--------------|
| Compute platform | Cloud Run (APIs), GKE (complex workloads) | Right tool per complexity level |
| Database | Cloud SQL (relational), Firestore (document) | Managed services reduce ops |
| IaC tool | Terraform | Multi-cloud compatible, mature ecosystem |
| CI/CD | Cloud Build + Cloud Deploy | Native GCP integration |
| Networking | Shared VPC | Centralized network management |
| Secret management | Secret Manager | Integrated with IAM, audited access |
| Observability | Cloud Operations suite | Native integration, low overhead |
| Container registry | Artifact Registry | Vulnerability scanning, multi-format |

---

## Quick Reference

| Pillar | Key Principle | Top GCP Tool |
|--------|-------------|-------------|
| Operational Excellence | Automate everything | Terraform + Cloud Build |
| Security | Least privilege, defense in depth | IAM + VPC Service Controls |
| Reliability | Multi-zone, auto-healing | Regional MIGs + Health Checks |
| Performance | Cache + right-size | Cloud CDN + Memorystore |
| Cost | Commit + auto-stop | CUDs + Autoscaling |

---

## What's Next?

Continue to **Chapter 63: Real-World Architecture Patterns** → `63-real-world-patterns.md`
