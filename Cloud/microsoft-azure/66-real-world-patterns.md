# Chapter 66: Real-World Architecture Patterns

---

## Table of Contents

- [Overview](#overview)
- [Part 1: N-Tier Architecture](#part-1-n-tier-architecture)
- [Part 2: Microservices on AKS](#part-2-microservices-on-aks)
- [Part 3: Serverless Architecture](#part-3-serverless-architecture)
- [Part 4: Event-Driven Architecture](#part-4-event-driven-architecture)
- [Part 5: CQRS & Event Sourcing](#part-5-cqrs--event-sourcing)
- [Part 6: Static Web App + API](#part-6-static-web-app--api)
- [Part 7: Multi-Region High Availability](#part-7-multi-region-high-availability)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

This chapter covers common architecture patterns used in production Azure applications. Each pattern includes when to use it, the Azure services involved, and a visual diagram.

```
What you'll learn:
├── N-Tier (classic web application)
├── Microservices on AKS
├── Serverless (Functions + managed services)
├── Event-Driven (reactive architecture)
├── CQRS & Event Sourcing
├── Static Web App + API
├── Multi-Region High Availability
└── When to use which pattern
```

---

## Part 1: N-Tier Architecture

```
Classic web application pattern (most common starting point)

Users
  │
  ▼
┌──────────────────┐
│ Azure Front Door │  CDN + WAF + Global LB
└────────┬─────────┘
         │
┌────────▼─────────┐
│ App Service      │  Web tier (Node.js, .NET, Java)
│ (Web App)        │  Deployment slots for zero-downtime
└────────┬─────────┘
         │
┌────────▼─────────┐
│ App Service      │  API tier (REST API)
│ (API App)        │  or Azure Functions
└────────┬─────────┘
         │
┌────────▼─────────┐
│ Azure SQL        │  Data tier
│ + Redis Cache    │  SQL for data, Redis for session/cache
└──────────────────┘

When to use:
├── Traditional web applications
├── Small-medium teams
├── Simple deployment model
├── Getting started with cloud
└── Monolith or modular monolith

Azure services: Front Door + App Service + Azure SQL + Redis Cache
Cost: ~$100-500/month for small apps
```

---

## Part 2: Microservices on AKS

```
Independent services communicating via APIs and events

Users
  │
  ▼
┌──────────────────┐
│ Azure Front Door │  Global routing + WAF
└────────┬─────────┘
         │
┌────────▼─────────┐
│ Application      │  API Gateway / Ingress
│ Gateway (AGIC)   │  SSL termination, routing
└────────┬─────────┘
         │
┌────────▼─────────────────────────────────────┐
│ AKS Cluster                                    │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐         │
│ │ Order   │ │ Payment │ │ Shipping│         │
│ │ Service │ │ Service │ │ Service │         │
│ │ (pods)  │ │ (pods)  │ │ (pods)  │         │
│ └────┬────┘ └────┬────┘ └────┬────┘         │
│      │           │           │               │
│ ┌────▼───────────▼───────────▼─────────┐     │
│ │ Service Bus (async communication)      │     │
│ └────────────────────────────────────────┘     │
└──────────────────────────────────────────────┘
         │
    ┌────▼────┐  ┌────────┐  ┌────────┐
    │Cosmos DB│  │ Azure  │  │  ACR   │
    │(NoSQL)  │  │ SQL    │  │(images)│
    └─────────┘  └────────┘  └────────┘

Each service:
├── Has its own database (database per service)
├── Independently deployable
├── Can use different technologies
├── Communicates via Service Bus (async) or HTTP (sync)
└── Scaled independently (KEDA for event-driven scaling)

When to use:
├── Large applications with multiple teams
├── Need independent deployment and scaling
├── Complex domain with bounded contexts
├── High scalability requirements
└── ⚠️ Don't use for small apps (overhead not worth it)
```

---

## Part 3: Serverless Architecture

```
No servers to manage — just code and managed services

Users
  │
  ▼
┌──────────────────┐
│ API Management   │  API Gateway (throttling, auth, docs)
└────────┬─────────┘
         │
┌────────▼─────────┐
│ Azure Functions  │  Business logic (event-driven)
│ (Consumption)    │  Scale from 0 to 1000s automatically
└────────┬─────────┘
         │
    ┌────┼──────────┬──────────┐
    │    │          │          │
┌───▼──┐ ┌───▼──┐ ┌───▼──┐ ┌──▼───┐
│Cosmos│ │Queue │ │Blob  │ │Event │
│  DB  │ │Storage│ │Storage│ │Grid  │
└──────┘ └──────┘ └──────┘ └──────┘

Example flow (image processing):
  User uploads image → Blob trigger → Function resizes
  → Saves thumbnail → Event Grid → Another Function sends notification

When to use:
├── Event-driven workloads
├── Unpredictable/spiky traffic (scale to zero)
├── Low-cost for low-traffic apps
├── Glue between services (integration)
├── APIs with variable load
└── Background processing (queue-triggered)

Cost: Pay per execution — can be $0 for low traffic!
```

---

## Part 4: Event-Driven Architecture

```
Services react to events — loose coupling, high scalability

┌──────────────┐
│ Order Service│ ─── publishes "OrderCreated" event
└──────┬───────┘
       │
┌──────▼───────┐
│ Event Grid   │ ─── routes events to subscribers
│ / Event Hub  │
└──────┬───────┘
       │
  ┌────┼────────────┬──────────────┐
  │    │            │              │
┌─▼──┐ ┌───▼───┐ ┌───▼────┐ ┌────▼───┐
│Pay │ │Notify │ │Inventory│ │Analytics│
│Svc │ │Svc    │ │Svc      │ │Svc     │
└────┘ └───────┘ └────────┘ └────────┘

Each subscriber processes the event independently.
Adding new subscribers doesn't change the publisher!

Patterns:
├── Event Notification: "Something happened" (Event Grid)
│   Event is small, consumer fetches details if needed
│
├── Event-Carried State Transfer: Event contains full data
│   Consumer has all info needed, no callback
│
├── Event Sourcing: Store events as the source of truth
│   Never delete/update — only append new events
│
└── Saga Pattern: Distributed transactions via events
    OrderCreated → PaymentProcessed → InventoryReserved → ShipmentCreated
    If any step fails → Compensating events (refund, release inventory)

When to use:
├── Loose coupling between services
├── Multiple consumers for same event
├── Audit trail requirements
├── Real-time reactions
└── Scalable, independent processing
```

---

## Part 5: CQRS & Event Sourcing

```
CQRS = Command Query Responsibility Segregation
Separate the write model from the read model

Traditional:
  App → Same DB → Read and Write same tables

CQRS:
  Write side                          Read side
  ┌──────────┐                       ┌──────────┐
  │ Commands │                       │ Queries  │
  │ (create, │                       │ (get,    │
  │  update, │                       │  list,   │
  │  delete) │                       │  search) │
  └────┬─────┘                       └────┬─────┘
       │                                  │
  ┌────▼─────┐    Events            ┌────▼─────┐
  │ Write DB │ ─────────────────── │ Read DB  │
  │ (SQL)    │  (sync via Event    │ (Cosmos, │
  │ Normalized│   Hub/Service Bus)  │  Redis,  │
  └──────────┘                      │  Search) │
                                    │ Optimized│
                                    │ for reads│
                                    └──────────┘

Benefits:
├── Read and write models optimized independently
├── Read model can use denormalized views (fast!)
├── Scale read and write independently
├── Multiple read models (different projections)
└── Combined with Event Sourcing for full audit trail

Event Sourcing:
├── Don't store current state — store ALL events
│   OrderCreated → ItemAdded → ItemRemoved → OrderPaid
├── Replay events to reconstruct current state
├── Full audit trail (compliance, debugging)
├── Time travel (reconstruct state at any point)
└── Azure: Event Hub or Cosmos DB (change feed) for events

When to use:
├── High-read, low-write workloads
├── Complex domains with different read/write needs
├── Audit requirements (financial, healthcare)
└── ⚠️ Adds complexity — don't use for simple CRUD apps
```

---

## Part 6: Static Web App + API

```
Modern SPA (Single Page App) architecture

Users
  │
  ▼
┌──────────────────────────────────────┐
│ Azure Static Web Apps                  │
│ ┌──────────────┐ ┌──────────────────┐ │
│ │ Static files │ │ API (Functions)  │ │
│ │ (React/Vue/  │ │ (Node.js/.NET)   │ │
│ │  Angular)    │ │                  │ │
│ │ Global CDN   │ │ Managed backend  │ │
│ └──────────────┘ └──────────────────┘ │
└──────────────────────────────────────┘
         │
    ┌────┼────────────┐
    │    │            │
┌───▼──┐ ┌───▼──┐ ┌───▼──┐
│Cosmos│ │Auth  │ │Blob  │
│  DB  │ │(Entra│ │Storage│
└──────┘ │ ID)  │ └──────┘
         └──────┘

Features of Static Web Apps:
├── Free SSL, custom domains
├── GitHub/Azure DevOps CI/CD (auto-deploy on push)
├── Staging environments (PR preview)
├── Built-in auth (Entra ID, GitHub, Twitter)
├── API routes (Azure Functions backend)
└── Global CDN distribution

When to use:
├── SPAs (React, Vue, Angular, Svelte)
├── Static sites (documentation, blogs)
├── JAMstack architecture
├── Low-cost web apps
└── Free tier available!
```

---

## Part 7: Multi-Region High Availability

```
Deploy across multiple Azure regions for maximum availability

Users (Global)
  │
  ▼
┌──────────────────┐
│ Azure Front Door │  Global LB + failover
│ or Traffic Mgr   │  Health probes per region
└────────┬─────────┘
    ┌────┴────┐
    │         │
┌───▼───┐ ┌──▼────┐
│Region1│ │Region2│
│Primary│ │Second.│
│       │ │       │
│App Svc│ │App Svc│
│Azure  │ │Azure  │
│SQL    │ │SQL    │  ← Active geo-replication
│Redis  │ │Redis  │
│Storage│ │Storage│  ← GRS/GZRS
└───────┘ └───────┘

Patterns:
├── Active-Passive: Primary handles traffic, secondary on standby
│   Failover when primary is unhealthy
│   Cost: Pay for standby resources (lower SKU OK)
│
├── Active-Active: Both regions handle traffic simultaneously
│   Front Door routes to nearest healthy region
│   Cost: Full resources in both regions
│   Best: Global users, maximum availability
│
└── Design considerations:
    ├── Data replication lag (RPO)
    ├── DNS failover time
    ├── Stateless app tier (session in Redis)
    ├── Database: Geo-replication or Cosmos DB multi-region
    ├── Storage: GRS/GZRS for blobs and files
    └── Testing: Regular DR drills!

SLA improvement:
  Single region (3 AZs): 99.99%
  Multi-region (active-active): 99.999%+
```

---

## Quick Reference

```
Architecture Patterns:

N-Tier: Simple, traditional (App Service + SQL + Redis)
  → Small-medium apps, getting started

Microservices: Independent services (AKS + Service Bus + Cosmos)
  → Large apps, multiple teams, independent scaling

Serverless: No servers (Functions + managed services)
  → Event-driven, spiky traffic, low cost

Event-Driven: React to events (Event Grid/Hub + Functions)
  → Loose coupling, multiple consumers

CQRS: Separate read/write (different DBs for each)
  → High-read workloads, complex domains

Static Web App + API: SPA + Functions backend
  → Modern web apps, JAMstack, low cost

Multi-Region: Deploy across regions (Front Door + geo-replication)
  → Global users, maximum availability, DR

Rule of thumb:
├── Start simple (N-tier or serverless)
├── Evolve as needed (don't over-architect day 1)
└── Microservices only if team/domain complexity justifies it
```

---

## What's Next?

Next chapter: [Chapter 67: Cost Optimization Strategies](67-cost-optimization.md) — Practical strategies to reduce your Azure bill.
