# Chapter 63 — Real-World Architecture Patterns

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Pattern Selection Guide](#part-1--pattern-selection-guide)
- [Part 2: Three-Tier Web Application](#part-2--three-tier-web-application)
- [Part 3: Microservices on GKE](#part-3--microservices-on-gke)
- [Part 4: Serverless Event-Driven Architecture](#part-4--serverless-event-driven-architecture)
- [Part 5: Data Analytics Pipeline](#part-5--data-analytics-pipeline)
- [Part 6: Real-Time Streaming Platform](#part-6--real-time-streaming-platform)
- [Part 7: E-Commerce Platform](#part-7--e-commerce-platform)
- [Part 8: SaaS Multi-Tenant Architecture](#part-8--saas-multi-tenant-architecture)
- [Part 9: IoT Data Processing](#part-9--iot-data-processing)
- [Part 10: Machine Learning Platform](#part-10--machine-learning-platform)
- [Part 11: Content Management & Media Platform](#part-11--content-management--media-platform)
- [Part 12: Financial Services Architecture](#part-12--financial-services-architecture)
- [Part 13: Healthcare & HIPAA Compliant Architecture](#part-13--healthcare--hipaa-compliant-architecture)
- [Part 14: Gaming Backend Architecture](#part-14--gaming-backend-architecture)
- [Part 15: Startup MVP to Scale Architecture](#part-15--startup-mvp-to-scale-architecture)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

This chapter presents production-tested architecture patterns for common real-world applications on Google Cloud. Each pattern includes a detailed architecture diagram, component selection rationale, scaling strategy, cost considerations, and a summary of GCP services used. Use these as blueprints to accelerate your own architecture decisions.

---

## Part 1 — Pattern Selection Guide

### Choosing the Right Pattern

```
┌────────────────────────────────────────────────────────────────────┐
│         PATTERN SELECTION GUIDE                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Application Type          │ Recommended Pattern                  │
│  ─────────────────────────── │ ──────────────────────────────────  │
│  Web app (standard)         │ Three-Tier (Part 2)                 │
│  Complex microservices      │ Microservices on GKE (Part 3)       │
│  Event-driven / webhooks    │ Serverless Event-Driven (Part 4)    │
│  Data warehouse / BI        │ Data Analytics Pipeline (Part 5)    │
│  Real-time analytics        │ Streaming Platform (Part 6)         │
│  Online store               │ E-Commerce (Part 7)                 │
│  Multi-tenant SaaS          │ SaaS Multi-Tenant (Part 8)          │
│  IoT sensor data            │ IoT Processing (Part 9)             │
│  ML/AI platform             │ ML Platform (Part 10)               │
│  Media / content site       │ Content & Media (Part 11)           │
│  Banking / fintech          │ Financial Services (Part 12)        │
│  Health records             │ HIPAA Compliant (Part 13)           │
│  Game backend               │ Gaming Backend (Part 14)            │
│  Startup / MVP              │ MVP to Scale (Part 15)              │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 2 — Three-Tier Web Application

```
┌──────────────────────────────────────────────────────────────────────┐
│     THREE-TIER WEB APPLICATION                                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Users                                                                │
│    │                                                                   │
│    ▼                                                                   │
│  Cloud DNS → Global HTTP(S) Load Balancer                            │
│    │                     │                                             │
│    │                     ├── Cloud Armor (WAF, DDoS)                  │
│    │                     └── Cloud CDN (static assets)                │
│    │                                                                   │
│    ├── PRESENTATION TIER                                              │
│    │   ┌──────────────────────────────────────────┐                  │
│    │   │ Cloud Run (frontend — React/Next.js SSR) │                  │
│    │   │ Auto-scales 0 → 100 instances             │                  │
│    │   │ Custom domain + managed TLS               │                  │
│    │   └──────────────────┬───────────────────────┘                  │
│    │                      │                                            │
│    ├── APPLICATION TIER                                               │
│    │   ┌──────────────────▼───────────────────────┐                  │
│    │   │ Cloud Run (API — Node.js / Python)       │                  │
│    │   │ Internal traffic only (no public access) │                  │
│    │   │ Connects to DB via Private IP            │                  │
│    │   └──────────────────┬───────────────────────┘                  │
│    │                      │                                            │
│    └── DATA TIER                                                      │
│        ┌──────────────────▼───────────────────────┐                  │
│        │ Cloud SQL PostgreSQL (HA, multi-zone)    │                  │
│        │ + Memorystore Redis (session cache)      │                  │
│        │ + GCS (file uploads, static assets)      │                  │
│        └──────────────────────────────────────────┘                  │
│                                                                        │
│  Services: Cloud DNS, Cloud LB, Cloud Armor, Cloud CDN,             │
│            Cloud Run (x2), Cloud SQL, Memorystore, GCS              │
│                                                                        │
│  Est. cost (low traffic): ~$80-150/month                             │
│  Est. cost (medium traffic): ~$300-800/month                         │
│  Scales to millions of requests with no architecture changes        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 3 — Microservices on GKE

```
┌──────────────────────────────────────────────────────────────────────┐
│     MICROSERVICES ON GKE                                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Users → Global HTTP(S) LB → GKE Ingress (Gateway API)             │
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │ GKE Autopilot Cluster (regional)                           │       │
│  │                                                             │       │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │       │
│  │ │ Auth Svc │ │ User Svc │ │Order Svc │ │Payment   │    │       │
│  │ │ (Go)     │ │ (Go)     │ │(Java)    │ │Svc (Go)  │    │       │
│  │ │ HPA:2-10 │ │ HPA:2-20 │ │HPA:3-30  │ │HPA:2-10  │    │       │
│  │ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘    │       │
│  │      │            │            │            │            │       │
│  │      └────────────┼────────────┼────────────┘            │       │
│  │                   │            │                          │       │
│  │           ┌───────▼────────┐   │                          │       │
│  │           │ Service Mesh   │   │                          │       │
│  │           │ (Istio/Anthos) │   │                          │       │
│  │           │ mTLS, tracing  │   │                          │       │
│  │           └────────────────┘   │                          │       │
│  │                                │                          │       │
│  │  Internal communication:  gRPC + Pub/Sub (async)        │       │
│  │  Config: ConfigMaps + Secret Manager                    │       │
│  │  Identity: Workload Identity (no SA keys)               │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
│  Data stores (per service):                                          │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │ Auth → Memorystore Redis (sessions)                       │       │
│  │ Users → Cloud SQL PostgreSQL                              │       │
│  │ Orders → Cloud SQL PostgreSQL (separate instance)        │       │
│  │ Payments → Cloud Spanner (strong consistency)            │       │
│  │ Events → Pub/Sub → BigQuery (analytics)                  │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
│  CI/CD: Cloud Build → Artifact Registry → Cloud Deploy → GKE     │
│  Observability: Cloud Monitoring + Cloud Trace + Cloud Logging     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 4 — Serverless Event-Driven Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│     SERVERLESS EVENT-DRIVEN                                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  Event Sources:                                            │       │
│  │  ┌──────────┐  ┌───────────┐  ┌────────────┐             │       │
│  │  │ API      │  │ GCS file  │  │ Scheduled  │             │       │
│  │  │ Gateway  │  │ upload    │  │ (cron)     │             │       │
│  │  └────┬─────┘  └─────┬─────┘  └──────┬─────┘             │       │
│  │       │               │               │                    │       │
│  │       ▼               ▼               ▼                    │       │
│  │  ┌──────────────────────────────────────────┐             │       │
│  │  │           Eventarc (event router)         │             │       │
│  │  └──────┬──────────┬──────────┬─────────────┘             │       │
│  │         │          │          │                             │       │
│  │         ▼          ▼          ▼                             │       │
│  │  ┌──────────┐ ┌─────────┐ ┌──────────┐                   │       │
│  │  │Cloud Run │ │Cloud    │ │Cloud Run │                   │       │
│  │  │(process  │ │Functions│ │(generate │                   │       │
│  │  │ order)   │ │(resize  │ │ report)  │                   │       │
│  │  └────┬─────┘ │ image)  │ └────┬─────┘                   │       │
│  │       │        └────┬────┘      │                          │       │
│  │       │             │           │                          │       │
│  │       ▼             ▼           ▼                          │       │
│  │  ┌──────────┐  ┌────────┐  ┌──────────┐                  │       │
│  │  │Firestore │  │GCS     │  │BigQuery  │                  │       │
│  │  │(state)   │  │(output)│  │(reports) │                  │       │
│  │  └──────────┘  └────────┘  └──────────┘                  │       │
│  │                                                             │       │
│  │  Async processing:                                         │       │
│  │  Event → Pub/Sub → Cloud Run → notification              │       │
│  │                                                             │       │
│  │  Retry: built-in with dead-letter topics                  │       │
│  │  Scale: 0 to 1000+ concurrent instances                   │       │
│  │  Cost: pay only when processing events                    │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
│  Services: Eventarc, Cloud Run, Cloud Functions, Pub/Sub,           │
│            Firestore, GCS, BigQuery, Cloud Scheduler                │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 5 — Data Analytics Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     DATA ANALYTICS PIPELINE                                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  DATA SOURCES                                              │       │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │       │
│  │  │ App DBs  │ │ S3/Azure │ │ SaaS     │ │ Streaming│   │       │
│  │  │ (Cloud   │ │ (Storage │ │ (BQDTS   │ │ (Pub/Sub │   │       │
│  │  │  SQL)    │ │  Transfer│ │  Google  │ │  events) │   │       │
│  │  │          │ │  Service)│ │  Ads)    │ │          │   │       │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘   │       │
│  │       │             │            │            │          │       │
│  │       ▼             ▼            ▼            ▼          │       │
│  │  ┌──────────────────────────────────────────────────┐    │       │
│  │  │              GCS (Data Lake — raw zone)           │    │       │
│  │  └──────────────────────┬───────────────────────────┘    │       │
│  │                          │                                │       │
│  │                   ┌──────▼──────┐                        │       │
│  │                   │  Dataflow   │  ETL/ELT               │       │
│  │                   │  (Apache    │  Transform, clean,     │       │
│  │                   │   Beam)     │  enrich                │       │
│  │                   └──────┬──────┘                        │       │
│  │                          │                                │       │
│  │                   ┌──────▼──────────────────────────┐    │       │
│  │                   │  BigQuery (Data Warehouse)       │    │       │
│  │                   │  ├── raw_zone (landing)          │    │       │
│  │                   │  ├── staging (transformed)      │    │       │
│  │                   │  ├── curated (business models)  │    │       │
│  │                   │  └── serving (optimized views)  │    │       │
│  │                   └──────┬──────────────────────────┘    │       │
│  │                          │                                │       │
│  │            ┌─────────────┼─────────────┐                 │       │
│  │            ▼             ▼             ▼                  │       │
│  │    ┌──────────┐  ┌──────────┐  ┌──────────┐             │       │
│  │    │ Looker   │  │ Vertex   │  │ Data     │             │       │
│  │    │ Studio   │  │ AI (ML)  │  │ exports  │             │       │
│  │    │ (BI)     │  │          │  │ (API)    │             │       │
│  │    └──────────┘  └──────────┘  └──────────┘             │       │
│  │                                                             │       │
│  │  Orchestration: Cloud Composer (Airflow) DAGs              │       │
│  │  Schedule: hourly ETL, daily aggregations, weekly reports  │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 6 — Real-Time Streaming Platform

```
┌──────────────────────────────────────────────────────────────────────┐
│     REAL-TIME STREAMING PLATFORM                                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  Event producers:                                          │       │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                  │       │
│  │  │ Mobile   │ │ Web app  │ │ IoT      │                  │       │
│  │  │ app      │ │ clicks   │ │ sensors  │                  │       │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘                  │       │
│  │       └─────────────┼─────────────┘                        │       │
│  │                     ▼                                       │       │
│  │              ┌──────────────┐                               │       │
│  │              │   Pub/Sub    │  millions of msgs/sec        │       │
│  │              │   (ingest)   │  global, durable             │       │
│  │              └──────┬───────┘                               │       │
│  │                     │                                       │       │
│  │        ┌────────────┼────────────┐                         │       │
│  │        ▼            ▼            ▼                          │       │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                  │       │
│  │  │ Dataflow │ │ Dataflow │ │ Cloud    │                  │       │
│  │  │ (enrich  │ │ (agg to  │ │ Run     │                  │       │
│  │  │  + write │ │ Bigtable)│ │ (alerts)│                  │       │
│  │  │  to BQ)  │ │          │ │         │                  │       │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘                  │       │
│  │       ▼            ▼            ▼                          │       │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                  │       │
│  │  │ BigQuery │ │ Bigtable │ │ Pub/Sub  │                  │       │
│  │  │ (hist-   │ │ (real-   │ │ (alerts  │                  │       │
│  │  │  orical) │ │  time    │ │  topic)  │                  │       │
│  │  └──────────┘ │  serving)│ └──────────┘                  │       │
│  │               └──────────┘                                 │       │
│  │                                                             │       │
│  │  Latency: < 1 second end-to-end                           │       │
│  │  Throughput: millions of events/second                    │       │
│  │  Use: clickstream, real-time dashboards, fraud detection  │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 7 — E-Commerce Platform

```
┌──────────────────────────────────────────────────────────────────────┐
│     E-COMMERCE PLATFORM                                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  Global HTTP(S) LB + Cloud CDN + Cloud Armor               │       │
│  │       │                                                     │       │
│  │       ├── Static assets → GCS + CDN                        │       │
│  │       └── Dynamic → GKE Autopilot                          │       │
│  │                                                             │       │
│  │  ┌─────────────────────────────────────────────────┐       │       │
│  │  │ GKE Autopilot                                     │       │       │
│  │  │ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐│       │       │
│  │  │ │Storefront│ │Catalog  │ │Cart     │ │Search  ││       │       │
│  │  │ │(Next.js) │ │Service  │ │Service  │ │Service ││       │       │
│  │  │ └─────────┘ └─────────┘ └─────────┘ └────────┘│       │       │
│  │  │ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐│       │       │
│  │  │ │Order    │ │Payment  │ │Inventory│ │Notif.  ││       │       │
│  │  │ │Service  │ │Service  │ │Service  │ │Service ││       │       │
│  │  │ └─────────┘ └─────────┘ └─────────┘ └────────┘│       │       │
│  │  └─────────────────────────────────────────────────┘       │       │
│  │                                                             │       │
│  │  Data stores:                                              │       │
│  │  ├── Cloud SQL (users, orders — relational)               │       │
│  │  ├── Firestore (product catalog — flexible schema)        │       │
│  │  ├── Memorystore Redis (cart, sessions, cache)            │       │
│  │  ├── GCS (product images, static assets)                  │       │
│  │  └── BigQuery (analytics, recommendations training)       │       │
│  │                                                             │       │
│  │  Async processing:                                         │       │
│  │  ├── Pub/Sub → order processing pipeline                  │       │
│  │  ├── Pub/Sub → inventory sync                             │       │
│  │  ├── Pub/Sub → notification service (email, SMS)          │       │
│  │  └── Recommendations AI (product recommendations)         │       │
│  │                                                             │       │
│  │  Search: Vertex AI Search (or Elasticsearch on GKE)       │       │
│  │  Payments: Payment gateway (Stripe) via Payment Service   │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
│  Scale: 10K→1M+ concurrent users with HPA + CDN                    │
│  Peak handling: auto-scale + Pub/Sub for order queue buffering     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 8 — SaaS Multi-Tenant Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│     SAAS MULTI-TENANT                                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  Tenant isolation models:                                  │       │
│  │                                                             │       │
│  │  Option A: Shared database, tenant column                 │       │
│  │  ┌──────────────────────────────────────────────┐         │       │
│  │  │ Cloud SQL (single instance)                   │         │       │
│  │  │ Every table has tenant_id column              │         │       │
│  │  │ Row-level security enforced at app layer     │         │       │
│  │  │ Cost: lowest    Isolation: lowest             │         │       │
│  │  └──────────────────────────────────────────────┘         │       │
│  │                                                             │       │
│  │  Option B: Shared instance, separate schemas/databases   │       │
│  │  ┌──────────────────────────────────────────────┐         │       │
│  │  │ Cloud SQL (single instance)                   │         │       │
│  │  │ tenant_a database, tenant_b database          │         │       │
│  │  │ Cost: low    Isolation: medium                │         │       │
│  │  └──────────────────────────────────────────────┘         │       │
│  │                                                             │       │
│  │  Option C: Separate database instances per tenant        │       │
│  │  ┌──────────────────────────────────────────────┐         │       │
│  │  │ Cloud SQL instance per enterprise tenant      │         │       │
│  │  │ Full data isolation                           │         │       │
│  │  │ Cost: highest    Isolation: highest           │         │       │
│  │  └──────────────────────────────────────────────┘         │       │
│  │                                                             │       │
│  │  Architecture:                                             │       │
│  │  ┌──────────────────────────────────────────────┐         │       │
│  │  │ Global LB → Cloud Run (API — stateless)      │         │       │
│  │  │     │                                          │         │       │
│  │  │     ├── Auth: Firebase Auth / Identity Platform│        │       │
│  │  │     ├── Tenant resolution (subdomain/header)  │         │       │
│  │  │     ├── Feature flags: per-tenant plan        │         │       │
│  │  │     └── Rate limiting: Cloud Armor per-tenant │         │       │
│  │  │                                                │         │       │
│  │  │ Data: Cloud SQL (per-tenant schema)           │         │       │
│  │  │ Files: GCS (per-tenant prefix/bucket)         │         │       │
│  │  │ Cache: Memorystore (namespaced by tenant)     │         │       │
│  │  │ Queue: Pub/Sub (tenant-labeled messages)      │         │       │
│  │  │ Analytics: BigQuery (tenant_id in all tables) │         │       │
│  │  └──────────────────────────────────────────────┘         │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 9 — IoT Data Processing

```
┌──────────────────────────────────────────────────────────────────────┐
│     IoT DATA PROCESSING                                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  Devices (sensors, vehicles, machines)                    │       │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                   │       │
│  │  │ Temp │ │ GPS  │ │ Power│ │ Cam  │  thousands+       │       │
│  │  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘                   │       │
│  │     └────────┼────────┼────────┘                         │       │
│  │              ▼                                            │       │
│  │     ┌───────────────────┐                                │       │
│  │     │ MQTT / HTTP       │  IoT Core (deprecated) →      │       │
│  │     │ Gateway           │  Use Pub/Sub directly or      │       │
│  │     │ (Cloud Run)       │  3rd party (HiveMQ, EMQX)    │       │
│  │     └────────┬──────────┘                                │       │
│  │              ▼                                            │       │
│  │     ┌───────────────────┐                                │       │
│  │     │     Pub/Sub       │  ingest all telemetry          │       │
│  │     └───┬─────────┬─────┘                                │       │
│  │         │         │                                       │       │
│  │    ┌────▼────┐ ┌──▼──────────┐                           │       │
│  │    │Dataflow │ │Cloud Run    │                           │       │
│  │    │(stream  │ │(real-time   │                           │       │
│  │    │ to BQ + │ │ alerts if   │                           │       │
│  │    │ Bigtable│ │ threshold   │                           │       │
│  │    │)        │ │ exceeded)   │                           │       │
│  │    └─┬───┬───┘ └─────┬──────┘                           │       │
│  │      │   │           │                                    │       │
│  │      ▼   ▼           ▼                                    │       │
│  │  ┌──────┐┌────────┐┌──────────┐                         │       │
│  │  │ BQ   ││Bigtable││Notification│                       │       │
│  │  │(hist)││(latest ││(Pub/Sub → │                        │       │
│  │  │      ││ values)││ email/SMS)│                         │       │
│  │  └──────┘└────────┘└──────────┘                         │       │
│  │                                                             │       │
│  │  Dashboard: Looker Studio (BigQuery) + Grafana (Bigtable) │       │
│  │  ML: Vertex AI (predictive maintenance models)            │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 10 — Machine Learning Platform

```
┌──────────────────────────────────────────────────────────────────────┐
│     ML PLATFORM (MLOps)                                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  DATA LAYER                                                │       │
│  │  BigQuery (feature tables) ← ETL ← Cloud SQL (app data)  │       │
│  │  Vertex Feature Store (online + offline serving)          │       │
│  │                                                             │       │
│  │  EXPERIMENTATION                                           │       │
│  │  Vertex AI Workbench (Jupyter) → Experiments → Metrics   │       │
│  │                                                             │       │
│  │  TRAINING PIPELINE (Vertex AI Pipelines — weekly)         │       │
│  │  ┌──────────────────────────────────────────────┐         │       │
│  │  │ 1. Data validation (schema + stats)           │         │       │
│  │  │ 2. Feature engineering (Feature Store sync)   │         │       │
│  │  │ 3. Training (Custom Job — GPU / TPU)          │         │       │
│  │  │ 4. Evaluation (compare vs champion)           │         │       │
│  │  │ 5. Model Registry (version + tag)             │         │       │
│  │  │ 6. Deploy canary (10% traffic)                │         │       │
│  │  │ 7. Monitor (skew + drift detection)           │         │       │
│  │  │ 8. Promote to 100% if OK                     │         │       │
│  │  └──────────────────────────────────────────────┘         │       │
│  │                                                             │       │
│  │  SERVING                                                   │       │
│  │  Vertex AI Endpoint (online prediction)                   │       │
│  │  Vertex AI Batch Prediction (nightly scoring)             │       │
│  │                                                             │       │
│  │  MONITORING                                                │       │
│  │  Model Monitoring → alert on drift → retrigger pipeline  │       │
│  │                                                             │       │
│  │  GenAI LAYER                                               │       │
│  │  Gemini (RAG chatbot) + Vector Search (knowledge base)   │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 11 — Content Management & Media Platform

```
┌──────────────────────────────────────────────────────────────────────┐
│     CONTENT & MEDIA PLATFORM                                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  Content upload flow:                                      │       │
│  │  User → signed URL → GCS (raw-media bucket)              │       │
│  │    │                                                       │       │
│  │    ▼ Eventarc (GCS finalize trigger)                      │       │
│  │  Cloud Run (media processor)                              │       │
│  │    ├── Transcode video (Transcoder API)                   │       │
│  │    ├── Generate thumbnails (Cloud Functions + Vision API) │       │
│  │    ├── Extract metadata (Document AI / Vision API)        │       │
│  │    ├── Moderate content (Vision API safe search)          │       │
│  │    ├── Generate captions (Speech-to-Text)                 │       │
│  │    └── Store metadata → Firestore + BigQuery              │       │
│  │                                                             │       │
│  │  Content delivery:                                         │       │
│  │  Global HTTP(S) LB → Cloud CDN → GCS (processed-media)  │       │
│  │    ├── Adaptive bitrate streaming (HLS/DASH)              │       │
│  │    ├── Image resizing (Cloud Functions on-the-fly)        │       │
│  │    └── Signed URLs for premium content                    │       │
│  │                                                             │       │
│  │  CMS API:                                                  │       │
│  │  Cloud Run (headless CMS API) → Firestore (content)      │       │
│  │  Frontend: Cloud Run (SSR) + Cloud CDN                    │       │
│  │                                                             │       │
│  │  Search: Vertex AI Search (semantic search over content)  │       │
│  │  Analytics: BigQuery (views, engagement, revenue)         │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 12 — Financial Services Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│     FINANCIAL SERVICES                                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  Requirements: PCI-DSS, strong consistency, audit trail   │       │
│  │                                                             │       │
│  │  ┌──────────────────────────────────────────────┐         │       │
│  │  │ VPC Service Controls (data exfiltration guard)│         │       │
│  │  │                                                │         │       │
│  │  │ GKE (private cluster, Binary Authorization)   │         │       │
│  │  │ ┌─────────┐ ┌─────────┐ ┌─────────┐         │         │       │
│  │  │ │Account  │ │Transact.│ │Fraud    │         │         │       │
│  │  │ │Service  │ │Service  │ │Detection│         │         │       │
│  │  │ └────┬────┘ └────┬────┘ └────┬────┘         │         │       │
│  │  │      │           │           │               │         │       │
│  │  │      ▼           ▼           ▼               │         │       │
│  │  │ Cloud Spanner (global strong consistency)    │         │       │
│  │  │ • Multi-region (99.999% SLA)                 │         │       │
│  │  │ • ACID transactions                          │         │       │
│  │  │ • CMEK encryption                            │         │       │
│  │  │                                                │         │       │
│  │  │ Pub/Sub (transaction events)                  │         │       │
│  │  │ → Dataflow → BigQuery (analytics, reporting) │         │       │
│  │  │                                                │         │       │
│  │  │ Vertex AI Endpoint (fraud detection model)    │         │       │
│  │  │ • Feature Store (customer behavior features) │         │       │
│  │  │ • < 50ms inference latency                   │         │       │
│  │  └──────────────────────────────────────────────┘         │       │
│  │                                                             │       │
│  │  Security layers:                                          │       │
│  │  ├── Cloud Armor (DDoS, WAF, geo-restrictions)            │       │
│  │  ├── IAP (internal admin tools)                            │       │
│  │  ├── Cloud KMS (CMEK for all data stores)                 │       │
│  │  ├── Secret Manager (API keys, credentials)               │       │
│  │  ├── Audit logs → BigQuery (compliance reporting)         │       │
│  │  └── SCC Premium (continuous vulnerability scanning)      │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 13 — Healthcare & HIPAA Compliant Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│     HIPAA-COMPLIANT HEALTHCARE                                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  Prerequisites:                                            │       │
│  │  ├── BAA (Business Associate Agreement) with Google       │       │
│  │  ├── Assured Workloads (HIPAA compliance guardrails)     │       │
│  │  └── Organization policies enforced                      │       │
│  │                                                             │       │
│  │  ┌──────────────────────────────────────────────┐         │       │
│  │  │ Assured Workloads (HIPAA folder)              │         │       │
│  │  │                                                │         │       │
│  │  │ Cloud Run (HIPAA-eligible)                    │         │       │
│  │  │ ├── Patient portal API                        │         │       │
│  │  │ ├── Provider API                              │         │       │
│  │  │ └── FHIR API proxy                           │         │       │
│  │  │                                                │         │       │
│  │  │ Cloud Healthcare API (FHIR store)             │         │       │
│  │  │ ├── FHIR R4 compliant                         │         │       │
│  │  │ ├── DICOM store (medical imaging)            │         │       │
│  │  │ └── HL7v2 store (legacy integration)         │         │       │
│  │  │                                                │         │       │
│  │  │ Cloud SQL (app data — CMEK encrypted)         │         │       │
│  │  │ GCS (documents, reports — CMEK encrypted)     │         │       │
│  │  │                                                │         │       │
│  │  │ All data: CMEK (Cloud KMS)                    │         │       │
│  │  │ All access: Audit-logged (Cloud Logging)      │         │       │
│  │  │ Network: VPC Service Controls perimeter       │         │       │
│  │  │ PHI access: IAM + IAP + Cloud Identity       │         │       │
│  │  └──────────────────────────────────────────────┘         │       │
│  │                                                             │       │
│  │  AI/ML (de-identified data only):                         │       │
│  │  ├── DLP API (de-identify PHI before analytics)          │       │
│  │  ├── BigQuery (de-identified analytics)                   │       │
│  │  └── Vertex AI (clinical prediction models)              │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Gaming Backend Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│     GAMING BACKEND                                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                                                             │       │
│  │  Game clients (mobile, PC, console)                       │       │
│  │       │                                                    │       │
│  │       ├── REST API → Cloud Run (game services)            │       │
│  │       │    ├── Auth, matchmaking, leaderboard             │       │
│  │       │    └── Auto-scales per region                     │       │
│  │       │                                                    │       │
│  │       └── WebSocket → GKE (game servers)                  │       │
│  │            ├── Dedicated game server pods (Agones)        │       │
│  │            ├── Spot VMs for cost savings                  │       │
│  │            └── Regional clusters for low latency          │       │
│  │                                                             │       │
│  │  Data stores:                                              │       │
│  │  ├── Memorystore Redis (leaderboards, sessions, cache)   │       │
│  │  ├── Firestore (player profiles — real-time sync)        │       │
│  │  ├── Cloud Spanner (game state — global consistency)     │       │
│  │  └── BigQuery (analytics, player behavior)                │       │
│  │                                                             │       │
│  │  Matchmaking:                                              │       │
│  │  Cloud Run → Open Match (on GKE) → assign game server   │       │
│  │                                                             │       │
│  │  Events:                                                   │       │
│  │  Game events → Pub/Sub → Dataflow → BigQuery             │       │
│  │  → Vertex AI (player churn prediction, recommendations) │       │
│  │                                                             │       │
│  │  Global distribution:                                      │       │
│  │  Global LB → nearest region (us, eu, asia)               │       │
│  │  CDN for game assets (patches, textures)                  │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
│  Scale: 100K → 10M+ concurrent players                              │
│  Agones: open-source game server management on K8s                  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 15 — Startup MVP to Scale Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│     STARTUP: MVP → GROWTH → SCALE                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  PHASE 1: MVP ($0-50/month)                                         │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │  Frontend: Firebase Hosting (free tier)                    │       │
│  │  Backend:  Cloud Run (free tier — 2M requests/month)     │       │
│  │  Database: Firestore (free tier — 1 GiB)                 │       │
│  │  Auth:     Firebase Auth (free for most use cases)       │       │
│  │  Storage:  GCS (free tier — 5 GB)                         │       │
│  │  CI/CD:    Cloud Build (free tier — 120 min/day)         │       │
│  │  DNS:      Cloud DNS ($0.20/zone/month)                   │       │
│  │                                                             │       │
│  │  Total: $0-50/month for first 1000 users                 │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
│  PHASE 2: GROWTH ($200-2000/month)                                   │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │  + Cloud SQL (replace Firestore for relational needs)     │       │
│  │  + Memorystore Redis (caching, sessions)                  │       │
│  │  + Cloud CDN (faster global delivery)                     │       │
│  │  + Cloud Armor (basic DDoS protection)                    │       │
│  │  + Pub/Sub (async processing — emails, webhooks)         │       │
│  │  + Cloud Monitoring & Alerting                            │       │
│  │  + Secret Manager (credentials management)                │       │
│  │                                                             │       │
│  │  10K-100K users                                           │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
│  PHASE 3: SCALE ($2000-20000+/month)                                 │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │  + GKE Autopilot (multiple microservices)                 │       │
│  │  + Cloud Spanner (if global scale needed)                 │       │
│  │  + Multi-region deployment                                 │       │
│  │  + BigQuery (analytics, data warehouse)                   │       │
│  │  + Vertex AI (ML features)                                 │       │
│  │  + Committed Use Discounts                                 │       │
│  │  + VPC Service Controls (security compliance)             │       │
│  │  + Cloud Deploy (formal CD pipeline)                      │       │
│  │                                                             │       │
│  │  100K-1M+ users                                            │       │
│  └───────────────────────────────────────────────────────────┘       │
│                                                                        │
│  Key principle: Start simple, add complexity only when needed       │
│  Don't over-engineer the MVP — iterate based on real demand        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Pattern | Primary Compute | Primary DB | Key Services |
|---------|---------------|----------|-------------|
| Three-Tier | Cloud Run | Cloud SQL | CDN, LB, Memorystore |
| Microservices | GKE Autopilot | Cloud SQL/Spanner | Pub/Sub, Service Mesh |
| Serverless Event | Cloud Run/Functions | Firestore | Eventarc, Pub/Sub |
| Data Analytics | Dataflow | BigQuery | GCS, Composer, Looker |
| Streaming | Dataflow | BigQuery + Bigtable | Pub/Sub |
| E-Commerce | GKE Autopilot | Cloud SQL + Firestore | CDN, Recommendations AI |
| SaaS Multi-Tenant | Cloud Run | Cloud SQL | Identity Platform |
| IoT | Dataflow + Cloud Run | BigQuery + Bigtable | Pub/Sub |
| ML Platform | Vertex AI | BigQuery | Feature Store, Pipelines |
| Content/Media | Cloud Run | Firestore | CDN, Transcoder, Vision |
| Financial | GKE (private) | Cloud Spanner | KMS, SCC, VPC-SC |
| Healthcare | Cloud Run | Healthcare API | DLP, KMS, Assured |
| Gaming | GKE (Agones) | Spanner + Firestore | Memorystore, CDN |
| Startup MVP | Cloud Run | Firestore → Cloud SQL | Firebase, free tiers |

---

## What's Next?

Continue to **Chapter 64: Cost Optimization Strategies** → `64-cost-optimization.md`
