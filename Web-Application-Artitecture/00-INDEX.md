# 🌐 Web Application Architecture — From Zero to Planet Scale

> **A Complete Guide**: From building your first web app to architecting systems that serve **billions of requests** like Google, Amazon, Netflix, and Flipkart.
>
> Written as if the world's best teacher is sitting next to you, explaining everything simply — with diagrams, examples, and real-world stories.

---

## How to Use This Guide

```
Start from Chapter 1 if you are a complete beginner.
Each chapter builds on the previous one.
By the end, you will understand how the biggest companies in the world build their systems.

📁 Each topic has its own file (linked below).
📊 Diagrams and flow arrows are included wherever possible.
🐍 Python & ☕ Java are used for code examples.
🗄️ PostgreSQL, MongoDB, Redis, Kafka, etc. are used for infrastructure examples.
```

---

## 📚 TABLE OF CONTENTS

---

### **PART 1: THE ABSOLUTE BASICS — How the Web Works**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 1.1 | What is a Web Application? (with real-life analogy) | [01-what-is-a-web-application.md](./chapters/01-basics/01-what-is-a-web-application.md) | ⭐ Beginner |
| 1.2 | How the Internet Works — IP, TCP, HTTP in Plain English | [02-how-the-internet-works.md](./chapters/01-basics/02-how-the-internet-works.md) | ⭐ Beginner |
| 1.3 | DNS — How Your Browser Finds a Website | [03-dns-how-browser-finds-website.md](./chapters/01-basics/03-dns-how-browser-finds-website.md) | ⭐ Beginner |
| 1.4 | HTTP & HTTPS — The Language of the Web | [04-http-and-https.md](./chapters/01-basics/04-http-and-https.md) | ⭐ Beginner |
| 1.5 | Client-Server Model — The Foundation of Everything | [05-client-server-model.md](./chapters/01-basics/05-client-server-model.md) | ⭐ Beginner |
| 1.6 | Request-Response Lifecycle — What Happens When You Hit Enter | [06-request-response-lifecycle.md](./chapters/01-basics/06-request-response-lifecycle.md) | ⭐ Beginner |
| 1.7 | Ports, Sockets & Connections — The Plumbing of the Web | [07-ports-sockets-connections.md](./chapters/01-basics/07-ports-sockets-connections.md) | ⭐ Beginner |

---

### **PART 2: FRONTEND / UI — How the User Interface Works**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 2.1 | How the Browser Renders a Web Page (DOM, CSSOM, Paint) | [01-how-browser-renders.md](./chapters/02-frontend/01-how-browser-renders.md) | ⭐ Beginner |
| 2.2 | Client-Side Rendering (CSR) vs Server-Side Rendering (SSR) | [02-csr-vs-ssr.md](./chapters/02-frontend/02-csr-vs-ssr.md) | ⭐⭐ Intermediate |
| 2.3 | Static Site Generation (SSG) & Incremental Static Regeneration (ISR) | [03-ssg-and-isr.md](./chapters/02-frontend/03-ssg-and-isr.md) | ⭐⭐ Intermediate |
| 2.4 | CDN — Serving UI Globally at Lightning Speed | [04-cdn-for-frontend.md](./chapters/02-frontend/04-cdn-for-frontend.md) | ⭐⭐ Intermediate |
| 2.5 | Frontend Caching Strategies (Browser Cache, Service Workers, ETags) | [05-frontend-caching.md](./chapters/02-frontend/05-frontend-caching.md) | ⭐⭐ Intermediate |
| 2.6 | Single Page Applications (SPA) vs Multi Page Applications (MPA) | [06-spa-vs-mpa.md](./chapters/02-frontend/06-spa-vs-mpa.md) | ⭐⭐ Intermediate |
| 2.7 | Micro-Frontends — Breaking the UI into Independent Pieces | [07-micro-frontends.md](./chapters/02-frontend/07-micro-frontends.md) | ⭐⭐⭐ Advanced |
| 2.8 | Edge Computing for Frontend — Running UI Logic at the Edge | [08-edge-computing-frontend.md](./chapters/02-frontend/08-edge-computing-frontend.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 3: BACKEND FUNDAMENTALS — Servers, APIs & Application Logic**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 3.1 | What is a Backend Server? (Web Servers vs App Servers) | [01-what-is-backend-server.md](./chapters/03-backend/01-what-is-backend-server.md) | ⭐ Beginner |
| 3.2 | REST APIs — The Standard Way to Talk to a Server | [02-rest-apis.md](./chapters/03-backend/02-rest-apis.md) | ⭐ Beginner |
| 3.3 | GraphQL — Flexible Queries from the Client | [03-graphql.md](./chapters/03-backend/03-graphql.md) | ⭐⭐ Intermediate |
| 3.4 | gRPC — High-Performance Communication Between Services | [04-grpc.md](./chapters/03-backend/04-grpc.md) | ⭐⭐⭐ Advanced |
| 3.5 | WebSockets — Real-Time Bidirectional Communication | [05-websockets.md](./chapters/03-backend/05-websockets.md) | ⭐⭐ Intermediate |
| 3.6 | Server-Sent Events (SSE) & Long Polling | [06-sse-and-long-polling.md](./chapters/03-backend/06-sse-and-long-polling.md) | ⭐⭐ Intermediate |
| 3.7 | Thread Models — How Servers Handle Multiple Requests | [07-thread-models.md](./chapters/03-backend/07-thread-models.md) | ⭐⭐⭐ Advanced |
| 3.8 | Concurrency & Parallelism — Serving Thousands at Once | [08-concurrency-and-parallelism.md](./chapters/03-backend/08-concurrency-and-parallelism.md) | ⭐⭐⭐ Advanced |
| 3.9 | How Many Concurrent Requests Can a Server Handle? (Benchmarks & Config) | [09-concurrent-requests-and-server-config.md](./chapters/03-backend/09-concurrent-requests-and-server-config.md) | ⭐⭐⭐ Advanced |
| 3.10 | Connection Pooling — Reusing Expensive Connections | [10-connection-pooling.md](./chapters/03-backend/10-connection-pooling.md) | ⭐⭐⭐ Advanced |
| 3.11 | Async Programming — Non-Blocking I/O (Event Loop, Coroutines) | [11-async-programming.md](./chapters/03-backend/11-async-programming.md) | ⭐⭐⭐ Advanced |
| 3.12 | Concurrency Control & Locking — Preventing Double Execution (Mutex, Locks, Execution Guards) | [12-concurrency-control-locking.md](./chapters/03-backend/12-concurrency-control-locking.md) | ⭐⭐⭐ Advanced |

---

### **PART 4: ARCHITECTURE PATTERNS — From Simple to Complex**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 4.1 | Monolithic Architecture — Everything in One Place | [01-monolithic-architecture.md](./chapters/04-architecture-patterns/01-monolithic-architecture.md) | ⭐ Beginner |
| 4.2 | Monolithic with Separate UI & API (Decoupled Monolith) | [02-monolith-separate-ui-api.md](./chapters/04-architecture-patterns/02-monolith-separate-ui-api.md) | ⭐⭐ Intermediate |
| 4.3 | Layered (N-Tier) Architecture | [03-layered-n-tier.md](./chapters/04-architecture-patterns/03-layered-n-tier.md) | ⭐⭐ Intermediate |
| 4.4 | Service-Oriented Architecture (SOA) | [04-service-oriented-architecture.md](./chapters/04-architecture-patterns/04-service-oriented-architecture.md) | ⭐⭐ Intermediate |
| 4.5 | Microservices Architecture — Small, Independent & Powerful | [05-microservices-architecture.md](./chapters/04-architecture-patterns/05-microservices-architecture.md) | ⭐⭐⭐ Advanced |
| 4.6 | Event-Driven Architecture (EDA) | [06-event-driven-architecture.md](./chapters/04-architecture-patterns/06-event-driven-architecture.md) | ⭐⭐⭐ Advanced |
| 4.7 | CQRS — Command Query Responsibility Segregation | [07-cqrs.md](./chapters/04-architecture-patterns/07-cqrs.md) | ⭐⭐⭐⭐ Expert |
| 4.8 | Event Sourcing — Storing Every Change as an Event | [08-event-sourcing.md](./chapters/04-architecture-patterns/08-event-sourcing.md) | ⭐⭐⭐⭐ Expert |
| 4.9 | Hexagonal Architecture (Ports & Adapters) | [09-hexagonal-architecture.md](./chapters/04-architecture-patterns/09-hexagonal-architecture.md) | ⭐⭐⭐⭐ Expert |
| 4.10 | Serverless Architecture — No Servers to Manage | [10-serverless-architecture.md](./chapters/04-architecture-patterns/10-serverless-architecture.md) | ⭐⭐⭐ Advanced |
| 4.11 | Modular Monolith — Best of Both Worlds | [11-modular-monolith.md](./chapters/04-architecture-patterns/11-modular-monolith.md) | ⭐⭐⭐ Advanced |
| 4.12 | Strangler Fig Pattern — Migrating from Monolith to Microservices | [12-strangler-fig-pattern.md](./chapters/04-architecture-patterns/12-strangler-fig-pattern.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 5: DEPLOYMENT MODELS — Same Server, Multiple Servers, Multiple Instances**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 5.1 | Single Server Setup — Everything on One Machine | [01-single-server-setup.md](./chapters/05-deployment-models/01-single-server-setup.md) | ⭐ Beginner |
| 5.2 | Separate Servers for Frontend, Backend & Database | [02-separate-servers.md](./chapters/05-deployment-models/02-separate-servers.md) | ⭐⭐ Intermediate |
| 5.3 | Multiple Instances of the Same Application (Horizontal Scaling) | [03-multiple-instances.md](./chapters/05-deployment-models/03-multiple-instances.md) | ⭐⭐ Intermediate |
| 5.4 | Multi-Region Deployment — Serving Users Across the Globe | [04-multi-region-deployment.md](./chapters/05-deployment-models/04-multi-region-deployment.md) | ⭐⭐⭐⭐ Expert |
| 5.5 | Blue-Green & Canary Deployments — Zero Downtime Releases | [05-blue-green-canary.md](./chapters/05-deployment-models/05-blue-green-canary.md) | ⭐⭐⭐ Advanced |
| 5.6 | Rolling Deployments & Feature Flags | [06-rolling-deployments-feature-flags.md](./chapters/05-deployment-models/06-rolling-deployments-feature-flags.md) | ⭐⭐⭐ Advanced |
| 5.7 | Infrastructure as Code (Terraform, CloudFormation) | [07-infrastructure-as-code.md](./chapters/05-deployment-models/07-infrastructure-as-code.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 6: LOAD BALANCING — Distributing Traffic Like a Pro**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 6.1 | What is Load Balancing? (with Restaurant Analogy) | [01-what-is-load-balancing.md](./chapters/06-load-balancing/01-what-is-load-balancing.md) | ⭐ Beginner |
| 6.2 | Layer 4 vs Layer 7 Load Balancing | [02-l4-vs-l7-load-balancing.md](./chapters/06-load-balancing/02-l4-vs-l7-load-balancing.md) | ⭐⭐ Intermediate |
| 6.3 | Load Balancing Algorithms (Round Robin, Least Connections, IP Hash, Weighted) | [03-lb-algorithms.md](./chapters/06-load-balancing/03-lb-algorithms.md) | ⭐⭐ Intermediate |
| 6.4 | Health Checks — How Load Balancers Know a Server is Alive | [04-health-checks.md](./chapters/06-load-balancing/04-health-checks.md) | ⭐⭐ Intermediate |
| 6.5 | Sticky Sessions (Session Affinity) | [05-sticky-sessions.md](./chapters/06-load-balancing/05-sticky-sessions.md) | ⭐⭐⭐ Advanced |
| 6.6 | Global Server Load Balancing (GSLB) & GeoDNS | [06-gslb-and-geodns.md](./chapters/06-load-balancing/06-gslb-and-geodns.md) | ⭐⭐⭐⭐ Expert |
| 6.7 | Tools — Nginx, HAProxy, AWS ALB/NLB, Envoy | [07-lb-tools.md](./chapters/06-load-balancing/07-lb-tools.md) | ⭐⭐⭐ Advanced |
| 6.8 | Service Mesh Load Balancing (Istio, Linkerd) | [08-service-mesh-lb.md](./chapters/06-load-balancing/08-service-mesh-lb.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 7: AUTO SCALING & ELASTICITY — Growing and Shrinking Automatically**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 7.1 | Vertical Scaling vs Horizontal Scaling | [01-vertical-vs-horizontal-scaling.md](./chapters/07-auto-scaling/01-vertical-vs-horizontal-scaling.md) | ⭐ Beginner |
| 7.2 | Auto Scaling — Let the System Scale Itself | [02-auto-scaling.md](./chapters/07-auto-scaling/02-auto-scaling.md) | ⭐⭐⭐ Advanced |
| 7.3 | Scaling Policies (CPU, Memory, Request Count, Custom Metrics) | [03-scaling-policies.md](./chapters/07-auto-scaling/03-scaling-policies.md) | ⭐⭐⭐ Advanced |
| 7.4 | Kubernetes HPA, VPA & Cluster Autoscaler | [04-kubernetes-autoscaling.md](./chapters/07-auto-scaling/04-kubernetes-autoscaling.md) | ⭐⭐⭐⭐ Expert |
| 7.5 | Elasticity in Cloud — AWS Auto Scaling, Azure VMSS, GCP MIG | [05-cloud-elasticity.md](./chapters/07-auto-scaling/05-cloud-elasticity.md) | ⭐⭐⭐⭐ Expert |
| 7.6 | Predictive Scaling & Scheduled Scaling | [06-predictive-and-scheduled-scaling.md](./chapters/07-auto-scaling/06-predictive-and-scheduled-scaling.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 8: API GATEWAY — The Front Door of Your System**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 8.1 | What is an API Gateway? Why Do We Need One? | [01-what-is-api-gateway.md](./chapters/08-api-gateway/01-what-is-api-gateway.md) | ⭐⭐ Intermediate |
| 8.2 | Authentication & Authorization at the Gateway | [02-auth-at-gateway.md](./chapters/08-api-gateway/02-auth-at-gateway.md) | ⭐⭐ Intermediate |
| 8.3 | Rate Limiting & Throttling — Protecting Your System | [03-rate-limiting-throttling.md](./chapters/08-api-gateway/03-rate-limiting-throttling.md) | ⭐⭐⭐ Advanced |
| 8.4 | Request Routing, Transformation & Aggregation | [04-request-routing-transformation.md](./chapters/08-api-gateway/04-request-routing-transformation.md) | ⭐⭐⭐ Advanced |
| 8.5 | API Versioning Strategies | [05-api-versioning.md](./chapters/08-api-gateway/05-api-versioning.md) | ⭐⭐⭐ Advanced |
| 8.6 | Tools — Kong, AWS API Gateway, Apigee, Zuul, Spring Cloud Gateway | [06-api-gateway-tools.md](./chapters/08-api-gateway/06-api-gateway-tools.md) | ⭐⭐⭐ Advanced |
| 8.7 | Backend for Frontend (BFF) Pattern | [07-bff-pattern.md](./chapters/08-api-gateway/07-bff-pattern.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 9: DATABASES — Storing, Querying & Managing Data**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 9.1 | Relational Databases (SQL) — PostgreSQL, MySQL | [01-relational-databases.md](./chapters/09-databases/01-relational-databases.md) | ⭐ Beginner |
| 9.2 | NoSQL Databases — Types & When to Use What | [02-nosql-databases.md](./chapters/09-databases/02-nosql-databases.md) | ⭐⭐ Intermediate |
| 9.3 | Document Stores (MongoDB, CouchDB) | [03-document-stores.md](./chapters/09-databases/03-document-stores.md) | ⭐⭐ Intermediate |
| 9.4 | Key-Value Stores (Redis, DynamoDB) | [04-key-value-stores.md](./chapters/09-databases/04-key-value-stores.md) | ⭐⭐ Intermediate |
| 9.5 | Column-Family Stores (Cassandra, HBase) | [05-column-family-stores.md](./chapters/09-databases/05-column-family-stores.md) | ⭐⭐⭐ Advanced |
| 9.6 | Graph Databases (Neo4j, Amazon Neptune) | [06-graph-databases.md](./chapters/09-databases/06-graph-databases.md) | ⭐⭐⭐ Advanced |
| 9.7 | Time-Series Databases (InfluxDB, TimescaleDB) | [07-time-series-databases.md](./chapters/09-databases/07-time-series-databases.md) | ⭐⭐⭐ Advanced |
| 9.8 | Search Engines as Databases (Elasticsearch, OpenSearch) | [08-search-engines.md](./chapters/09-databases/08-search-engines.md) | ⭐⭐⭐ Advanced |
| 9.9 | Database Indexing — Making Queries Lightning Fast | [09-database-indexing.md](./chapters/09-databases/09-database-indexing.md) | ⭐⭐ Intermediate |
| 9.10 | Database Replication — Master-Slave, Master-Master | [10-database-replication.md](./chapters/09-databases/10-database-replication.md) | ⭐⭐⭐ Advanced |
| 9.11 | Database Sharding — Splitting Data Across Machines | [11-database-sharding.md](./chapters/09-databases/11-database-sharding.md) | ⭐⭐⭐⭐ Expert |
| 9.12 | Database Partitioning (Horizontal & Vertical) | [12-database-partitioning.md](./chapters/09-databases/12-database-partitioning.md) | ⭐⭐⭐⭐ Expert |
| 9.13 | ACID vs BASE — Consistency Models for Databases | [13-acid-vs-base.md](./chapters/09-databases/13-acid-vs-base.md) | ⭐⭐⭐ Advanced |
| 9.14 | Database Connection Pooling & Query Optimization | [14-db-connection-pooling.md](./chapters/09-databases/14-db-connection-pooling.md) | ⭐⭐⭐ Advanced |
| 9.15 | Polyglot Persistence — Using Multiple Databases Together | [15-polyglot-persistence.md](./chapters/09-databases/15-polyglot-persistence.md) | ⭐⭐⭐⭐ Expert |
| 9.16 | NewSQL Databases (CockroachDB, Google Spanner, TiDB) | [16-newsql-databases.md](./chapters/09-databases/16-newsql-databases.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 10: CACHING — Speed Up Everything**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 10.1 | What is Caching? Why is it So Important? | [01-what-is-caching.md](./chapters/10-caching/01-what-is-caching.md) | ⭐ Beginner |
| 10.2 | Application-Level Caching (In-Memory: Guava, Caffeine) | [02-application-level-caching.md](./chapters/10-caching/02-application-level-caching.md) | ⭐⭐ Intermediate |
| 10.3 | Distributed Caching (Redis, Memcached) | [03-distributed-caching.md](./chapters/10-caching/03-distributed-caching.md) | ⭐⭐⭐ Advanced |
| 10.4 | CDN Caching — Caching at the Edge | [04-cdn-caching.md](./chapters/10-caching/04-cdn-caching.md) | ⭐⭐ Intermediate |
| 10.5 | Database Query Caching | [05-database-query-caching.md](./chapters/10-caching/05-database-query-caching.md) | ⭐⭐⭐ Advanced |
| 10.6 | Cache Strategies (Write-Through, Write-Behind, Write-Around, Read-Through) | [06-cache-strategies.md](./chapters/10-caching/06-cache-strategies.md) | ⭐⭐⭐ Advanced |
| 10.7 | Cache Invalidation — The Hardest Problem in CS | [07-cache-invalidation.md](./chapters/10-caching/07-cache-invalidation.md) | ⭐⭐⭐⭐ Expert |
| 10.8 | Cache Eviction Policies (LRU, LFU, TTL) | [08-cache-eviction-policies.md](./chapters/10-caching/08-cache-eviction-policies.md) | ⭐⭐⭐ Advanced |
| 10.9 | Cache Stampede, Thundering Herd & Cache Penetration | [09-cache-problems.md](./chapters/10-caching/09-cache-problems.md) | ⭐⭐⭐⭐ Expert |
| 10.10 | Multi-Layer Caching Architecture | [10-multi-layer-caching.md](./chapters/10-caching/10-multi-layer-caching.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 11: MESSAGE QUEUES & EVENT STREAMING — Async Communication**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 11.1 | Synchronous vs Asynchronous Communication | [01-sync-vs-async.md](./chapters/11-message-queues/01-sync-vs-async.md) | ⭐⭐ Intermediate |
| 11.2 | Message Queues — RabbitMQ, Amazon SQS | [02-message-queues.md](./chapters/11-message-queues/02-message-queues.md) | ⭐⭐⭐ Advanced |
| 11.3 | Pub/Sub Pattern — Publishing & Subscribing to Events | [03-pub-sub-pattern.md](./chapters/11-message-queues/03-pub-sub-pattern.md) | ⭐⭐⭐ Advanced |
| 11.4 | Apache Kafka — Distributed Event Streaming at Scale | [04-apache-kafka.md](./chapters/11-message-queues/04-apache-kafka.md) | ⭐⭐⭐⭐ Expert |
| 11.5 | Dead Letter Queues — Handling Failed Messages | [05-dead-letter-queues.md](./chapters/11-message-queues/05-dead-letter-queues.md) | ⭐⭐⭐ Advanced |
| 11.6 | Exactly-Once, At-Least-Once, At-Most-Once Delivery | [06-delivery-guarantees.md](./chapters/11-message-queues/06-delivery-guarantees.md) | ⭐⭐⭐⭐ Expert |
| 11.7 | Event-Driven Microservices with Kafka/RabbitMQ | [07-event-driven-microservices.md](./chapters/11-message-queues/07-event-driven-microservices.md) | ⭐⭐⭐⭐ Expert |
| 11.8 | Stream Processing (Kafka Streams, Apache Flink) | [08-stream-processing.md](./chapters/11-message-queues/08-stream-processing.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 12: RESILIENCE & FAULT TOLERANCE — Building Systems That Never Die**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 12.1 | Request Timeouts — Don't Wait Forever | [01-request-timeouts.md](./chapters/12-resilience/01-request-timeouts.md) | ⭐⭐ Intermediate |
| 12.2 | Retry Mechanism — Try Again, But Smartly | [02-retry-mechanism.md](./chapters/12-resilience/02-retry-mechanism.md) | ⭐⭐⭐ Advanced |
| 12.3 | Circuit Breaker Pattern — Stop Calling a Dead Service | [03-circuit-breaker.md](./chapters/12-resilience/03-circuit-breaker.md) | ⭐⭐⭐ Advanced |
| 12.4 | Bulkhead Pattern — Isolating Failures | [04-bulkhead-pattern.md](./chapters/12-resilience/04-bulkhead-pattern.md) | ⭐⭐⭐⭐ Expert |
| 12.5 | Fallback Strategies — What to Do When Things Fail | [05-fallback-strategies.md](./chapters/12-resilience/05-fallback-strategies.md) | ⭐⭐⭐ Advanced |
| 12.6 | Backpressure — Slowing Down When Overwhelmed | [06-backpressure.md](./chapters/12-resilience/06-backpressure.md) | ⭐⭐⭐⭐ Expert |
| 12.7 | Idempotency — Making Operations Safe to Retry | [07-idempotency.md](./chapters/12-resilience/07-idempotency.md) | ⭐⭐⭐⭐ Expert |
| 12.8 | Graceful Degradation — Keep Core Features Working | [08-graceful-degradation.md](./chapters/12-resilience/08-graceful-degradation.md) | ⭐⭐⭐⭐ Expert |
| 12.9 | Chaos Engineering — Breaking Things on Purpose (Netflix Chaos Monkey) | [09-chaos-engineering.md](./chapters/12-resilience/09-chaos-engineering.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 13: DISTRIBUTED SYSTEMS — The Hard Stuff**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 13.1 | What is a Distributed System? Why is it Hard? | [01-what-is-distributed-system.md](./chapters/13-distributed-systems/01-what-is-distributed-system.md) | ⭐⭐ Intermediate |
| 13.2 | CAP Theorem — You Can Only Pick Two | [02-cap-theorem.md](./chapters/13-distributed-systems/02-cap-theorem.md) | ⭐⭐⭐ Advanced |
| 13.3 | PACELC Theorem — Beyond CAP | [03-pacelc-theorem.md](./chapters/13-distributed-systems/03-pacelc-theorem.md) | ⭐⭐⭐⭐ Expert |
| 13.4 | Consistency Models (Strong, Eventual, Causal) | [04-consistency-models.md](./chapters/13-distributed-systems/04-consistency-models.md) | ⭐⭐⭐⭐ Expert |
| 13.5 | Distributed Consensus (Paxos, Raft) | [05-distributed-consensus.md](./chapters/13-distributed-systems/05-distributed-consensus.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 13.6 | Distributed Transactions (2PC, Saga Pattern) | [06-distributed-transactions.md](./chapters/13-distributed-systems/06-distributed-transactions.md) | ⭐⭐⭐⭐ Expert |
| 13.7 | Leader Election — Who's the Boss? | [07-leader-election.md](./chapters/13-distributed-systems/07-leader-election.md) | ⭐⭐⭐⭐ Expert |
| 13.8 | Distributed Locking (Redlock, ZooKeeper) | [08-distributed-locking.md](./chapters/13-distributed-systems/08-distributed-locking.md) | ⭐⭐⭐⭐ Expert |
| 13.9 | Vector Clocks & Conflict Resolution | [09-vector-clocks.md](./chapters/13-distributed-systems/09-vector-clocks.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 13.10 | Consistent Hashing — Distributing Data Evenly | [10-consistent-hashing.md](./chapters/13-distributed-systems/10-consistent-hashing.md) | ⭐⭐⭐⭐ Expert |
| 13.11 | Gossip Protocol — How Nodes Talk to Each Other | [11-gossip-protocol.md](./chapters/13-distributed-systems/11-gossip-protocol.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 13.12 | Split Brain Problem & How to Handle It | [12-split-brain.md](./chapters/13-distributed-systems/12-split-brain.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 14: SECURITY — Protecting Your Application**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 14.1 | Authentication vs Authorization — Who Are You & What Can You Do? | [01-authn-vs-authz.md](./chapters/14-security/01-authn-vs-authz.md) | ⭐ Beginner |
| 14.2 | Sessions, Cookies & Tokens (JWT) | [02-sessions-cookies-jwt.md](./chapters/14-security/02-sessions-cookies-jwt.md) | ⭐⭐ Intermediate |
| 14.3 | OAuth 2.0 & OpenID Connect — Login with Google/GitHub | [03-oauth2-oidc.md](./chapters/14-security/03-oauth2-oidc.md) | ⭐⭐⭐ Advanced |
| 14.4 | SSL/TLS & Certificate Management | [04-ssl-tls-certificates.md](./chapters/14-security/04-ssl-tls-certificates.md) | ⭐⭐ Intermediate |
| 14.5 | OWASP Top 10 — Most Common Security Vulnerabilities | [05-owasp-top-10.md](./chapters/14-security/05-owasp-top-10.md) | ⭐⭐⭐ Advanced |
| 14.6 | CORS, CSRF, XSS — Web Security Essentials | [06-cors-csrf-xss.md](./chapters/14-security/06-cors-csrf-xss.md) | ⭐⭐⭐ Advanced |
| 14.7 | API Security Best Practices | [07-api-security.md](./chapters/14-security/07-api-security.md) | ⭐⭐⭐ Advanced |
| 14.8 | Secrets Management (Vault, AWS Secrets Manager) | [08-secrets-management.md](./chapters/14-security/08-secrets-management.md) | ⭐⭐⭐⭐ Expert |
| 14.9 | DDoS Protection & WAF (Web Application Firewall) | [09-ddos-and-waf.md](./chapters/14-security/09-ddos-and-waf.md) | ⭐⭐⭐⭐ Expert |
| 14.10 | Zero Trust Architecture | [10-zero-trust.md](./chapters/14-security/10-zero-trust.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 15: DNS — The Phone Book of the Internet (Deep Dive)**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 15.1 | DNS Deep Dive — Records (A, AAAA, CNAME, MX, TXT, NS) | [01-dns-deep-dive.md](./chapters/15-dns/01-dns-deep-dive.md) | ⭐⭐ Intermediate |
| 15.2 | DNS Resolution Flow — Step by Step | [02-dns-resolution-flow.md](./chapters/15-dns/02-dns-resolution-flow.md) | ⭐⭐ Intermediate |
| 15.3 | DNS-Based Load Balancing & Failover | [03-dns-load-balancing.md](./chapters/15-dns/03-dns-load-balancing.md) | ⭐⭐⭐ Advanced |
| 15.4 | DNS Caching, TTL & Propagation | [04-dns-caching-ttl.md](./chapters/15-dns/04-dns-caching-ttl.md) | ⭐⭐⭐ Advanced |
| 15.5 | Managed DNS Services (Route 53, Cloudflare DNS, Google Cloud DNS) | [05-managed-dns.md](./chapters/15-dns/05-managed-dns.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 16: CONTAINERS & ORCHESTRATION — Docker & Kubernetes**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 16.1 | What are Containers? (Docker Explained Simply) | [01-what-are-containers.md](./chapters/16-containers/01-what-are-containers.md) | ⭐⭐ Intermediate |
| 16.2 | Docker Deep Dive — Images, Containers, Volumes, Networks | [02-docker-deep-dive.md](./chapters/16-containers/02-docker-deep-dive.md) | ⭐⭐ Intermediate |
| 16.3 | Container Registries (Docker Hub, ECR, GCR) | [03-container-registries.md](./chapters/16-containers/03-container-registries.md) | ⭐⭐⭐ Advanced |
| 16.4 | Kubernetes Architecture — Pods, Services, Deployments | [04-kubernetes-architecture.md](./chapters/16-containers/04-kubernetes-architecture.md) | ⭐⭐⭐ Advanced |
| 16.5 | Kubernetes Networking & Service Discovery | [05-k8s-networking.md](./chapters/16-containers/05-k8s-networking.md) | ⭐⭐⭐⭐ Expert |
| 16.6 | Kubernetes Storage, ConfigMaps & Secrets | [06-k8s-storage-config.md](./chapters/16-containers/06-k8s-storage-config.md) | ⭐⭐⭐⭐ Expert |
| 16.7 | Helm Charts — Package Manager for Kubernetes | [07-helm-charts.md](./chapters/16-containers/07-helm-charts.md) | ⭐⭐⭐⭐ Expert |
| 16.8 | Service Mesh (Istio, Linkerd) — Advanced Traffic Management | [08-service-mesh.md](./chapters/16-containers/08-service-mesh.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 17: CI/CD — Automating Build, Test & Deploy**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 17.1 | What is CI/CD? (Continuous Integration / Continuous Deployment) | [01-what-is-cicd.md](./chapters/17-cicd/01-what-is-cicd.md) | ⭐⭐ Intermediate |
| 17.2 | CI/CD Pipelines — Building, Testing & Deploying Automatically | [02-cicd-pipelines.md](./chapters/17-cicd/02-cicd-pipelines.md) | ⭐⭐ Intermediate |
| 17.3 | Tools — Jenkins, GitHub Actions, GitLab CI, ArgoCD | [03-cicd-tools.md](./chapters/17-cicd/03-cicd-tools.md) | ⭐⭐⭐ Advanced |
| 17.4 | GitOps — Infrastructure Managed Through Git | [04-gitops.md](./chapters/17-cicd/04-gitops.md) | ⭐⭐⭐⭐ Expert |
| 17.5 | Artifact Management & Container Image Pipelines | [05-artifact-management.md](./chapters/17-cicd/05-artifact-management.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 18: MONITORING, LOGGING & OBSERVABILITY — Seeing Everything**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 18.1 | Why Monitoring Matters — You Can't Fix What You Can't See | [01-why-monitoring.md](./chapters/18-observability/01-why-monitoring.md) | ⭐⭐ Intermediate |
| 18.2 | The Three Pillars: Logs, Metrics & Traces | [02-three-pillars.md](./chapters/18-observability/02-three-pillars.md) | ⭐⭐ Intermediate |
| 18.3 | Centralized Logging (ELK Stack, Fluentd, Loki) | [03-centralized-logging.md](./chapters/18-observability/03-centralized-logging.md) | ⭐⭐⭐ Advanced |
| 18.4 | Metrics & Dashboards (Prometheus, Grafana) | [04-metrics-and-dashboards.md](./chapters/18-observability/04-metrics-and-dashboards.md) | ⭐⭐⭐ Advanced |
| 18.5 | Distributed Tracing (Jaeger, Zipkin, OpenTelemetry) | [05-distributed-tracing.md](./chapters/18-observability/05-distributed-tracing.md) | ⭐⭐⭐⭐ Expert |
| 18.6 | Alerting & On-Call — PagerDuty, OpsGenie | [06-alerting-oncall.md](./chapters/18-observability/06-alerting-oncall.md) | ⭐⭐⭐ Advanced |
| 18.7 | SLIs, SLOs & SLAs — Defining Reliability | [07-sli-slo-sla.md](./chapters/18-observability/07-sli-slo-sla.md) | ⭐⭐⭐⭐ Expert |
| 18.8 | APM Tools (Datadog, New Relic, Dynatrace) | [08-apm-tools.md](./chapters/18-observability/08-apm-tools.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 19: PERFORMANCE ENGINEERING — Making Things Blazingly Fast**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 19.1 | Latency, Throughput & Response Time — Key Metrics | [01-latency-throughput.md](./chapters/19-performance/01-latency-throughput.md) | ⭐⭐ Intermediate |
| 19.2 | Profiling Applications — Finding Bottlenecks | [02-profiling.md](./chapters/19-performance/02-profiling.md) | ⭐⭐⭐ Advanced |
| 19.3 | Load Testing & Stress Testing (JMeter, k6, Locust) | [03-load-testing.md](./chapters/19-performance/03-load-testing.md) | ⭐⭐⭐ Advanced |
| 19.4 | Database Performance Tuning | [04-db-performance-tuning.md](./chapters/19-performance/04-db-performance-tuning.md) | ⭐⭐⭐⭐ Expert |
| 19.5 | N+1 Query Problem & Batch Loading | [05-n-plus-1-problem.md](./chapters/19-performance/05-n-plus-1-problem.md) | ⭐⭐⭐ Advanced |
| 19.6 | Compression (Gzip, Brotli) & Minification | [06-compression-minification.md](./chapters/19-performance/06-compression-minification.md) | ⭐⭐ Intermediate |
| 19.7 | HTTP/2 & HTTP/3 (QUIC) — Faster Protocols | [07-http2-http3.md](./chapters/19-performance/07-http2-http3.md) | ⭐⭐⭐⭐ Expert |
| 19.8 | Performance Budgets & Core Web Vitals | [08-performance-budgets.md](./chapters/19-performance/08-performance-budgets.md) | ⭐⭐⭐ Advanced |

---

### **PART 20: DATA ENGINEERING IN WEB APPS — Pipelines, Lakes & Warehouses**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 20.1 | OLTP vs OLAP — Transactional vs Analytical | [01-oltp-vs-olap.md](./chapters/20-data-engineering/01-oltp-vs-olap.md) | ⭐⭐⭐ Advanced |
| 20.2 | Data Pipelines (ETL & ELT) | [02-data-pipelines.md](./chapters/20-data-engineering/02-data-pipelines.md) | ⭐⭐⭐ Advanced |
| 20.3 | Data Lakes & Data Warehouses (S3, BigQuery, Snowflake, Redshift) | [03-data-lakes-warehouses.md](./chapters/20-data-engineering/03-data-lakes-warehouses.md) | ⭐⭐⭐⭐ Expert |
| 20.4 | Change Data Capture (CDC) — Debezium | [04-change-data-capture.md](./chapters/20-data-engineering/04-change-data-capture.md) | ⭐⭐⭐⭐ Expert |
| 20.5 | Real-Time Analytics Architecture | [05-real-time-analytics.md](./chapters/20-data-engineering/05-real-time-analytics.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 21: SERVICE DISCOVERY, CONFIGURATION & COORDINATION**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 21.1 | Service Discovery — How Services Find Each Other | [01-service-discovery.md](./chapters/21-service-discovery/01-service-discovery.md) | ⭐⭐⭐ Advanced |
| 21.2 | Service Registry (Consul, Eureka, etcd) | [02-service-registry.md](./chapters/21-service-discovery/02-service-registry.md) | ⭐⭐⭐⭐ Expert |
| 21.3 | Configuration Management (Spring Cloud Config, Consul KV) | [03-configuration-management.md](./chapters/21-service-discovery/03-configuration-management.md) | ⭐⭐⭐⭐ Expert |
| 21.4 | Feature Toggles & Dynamic Configuration | [04-feature-toggles.md](./chapters/21-service-discovery/04-feature-toggles.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 22: STORAGE & FILE SYSTEMS — Beyond Databases**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 22.1 | Object Storage (S3, GCS, Azure Blob) | [01-object-storage.md](./chapters/22-storage/01-object-storage.md) | ⭐⭐ Intermediate |
| 22.2 | Block Storage vs File Storage vs Object Storage | [02-storage-types.md](./chapters/22-storage/02-storage-types.md) | ⭐⭐⭐ Advanced |
| 22.3 | Distributed File Systems (HDFS, GFS) | [03-distributed-file-systems.md](./chapters/22-storage/03-distributed-file-systems.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 22.4 | Content Delivery & Media Processing Pipelines | [04-media-processing.md](./chapters/22-storage/04-media-processing.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 23: NETWORKING DEEP DIVE — Proxies, Tunnels & Protocols**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 23.1 | Forward Proxy vs Reverse Proxy | [01-forward-vs-reverse-proxy.md](./chapters/23-networking/01-forward-vs-reverse-proxy.md) | ⭐⭐ Intermediate |
| 23.2 | Nginx & HAProxy Deep Dive | [02-nginx-haproxy.md](./chapters/23-networking/02-nginx-haproxy.md) | ⭐⭐⭐ Advanced |
| 23.3 | TCP vs UDP — When to Use What | [03-tcp-vs-udp.md](./chapters/23-networking/03-tcp-vs-udp.md) | ⭐⭐ Intermediate |
| 23.4 | VPC, Subnets & Network Security Groups | [04-vpc-subnets.md](./chapters/23-networking/04-vpc-subnets.md) | ⭐⭐⭐⭐ Expert |
| 23.5 | API Gateway vs Reverse Proxy vs Load Balancer — What's the Difference? | [05-gateway-vs-proxy-vs-lb.md](./chapters/23-networking/05-gateway-vs-proxy-vs-lb.md) | ⭐⭐⭐ Advanced |

---

### **PART 24: REAL-WORLD SYSTEM DESIGNS — How Big Companies Do It**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 24.1 | How Google Search Works — Architecture Overview | [01-google-search.md](./chapters/24-real-world/01-google-search.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 24.2 | How Amazon/Flipkart E-Commerce Architecture Works | [02-amazon-ecommerce.md](./chapters/24-real-world/02-amazon-ecommerce.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 24.3 | How Netflix Streams to 200 Million Users | [03-netflix-streaming.md](./chapters/24-real-world/03-netflix-streaming.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 24.4 | How WhatsApp/Messenger Handles Billions of Messages | [04-whatsapp-messaging.md](./chapters/24-real-world/04-whatsapp-messaging.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 24.5 | How YouTube/TikTok Serves Billions of Videos | [05-youtube-video.md](./chapters/24-real-world/05-youtube-video.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 24.6 | How Twitter/X Handles the Firehose of Tweets | [06-twitter-timeline.md](./chapters/24-real-world/06-twitter-timeline.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 24.7 | How Uber/Ola Handles Real-Time Location & Matching | [07-uber-ride-matching.md](./chapters/24-real-world/07-uber-ride-matching.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 24.8 | How Stripe/Razorpay Handles Payment Processing | [08-payment-processing.md](./chapters/24-real-world/08-payment-processing.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 24.9 | How Zoom Handles Millions of Video Calls | [09-zoom-video-calls.md](./chapters/24-real-world/09-zoom-video-calls.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 24.10 | How Instagram Handles Photo Uploads at Scale | [10-instagram-photos.md](./chapters/24-real-world/10-instagram-photos.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 25: DESIGN PATTERNS & BEST PRACTICES FOR LARGE SYSTEMS**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 25.1 | 12-Factor App Methodology | [01-twelve-factor-app.md](./chapters/25-design-patterns/01-twelve-factor-app.md) | ⭐⭐⭐ Advanced |
| 25.2 | Domain-Driven Design (DDD) Essentials | [02-domain-driven-design.md](./chapters/25-design-patterns/02-domain-driven-design.md) | ⭐⭐⭐⭐ Expert |
| 25.3 | API Design Best Practices (REST, Pagination, Error Handling) | [03-api-design-best-practices.md](./chapters/25-design-patterns/03-api-design-best-practices.md) | ⭐⭐⭐ Advanced |
| 25.4 | Saga Pattern — Managing Distributed Transactions | [04-saga-pattern.md](./chapters/25-design-patterns/04-saga-pattern.md) | ⭐⭐⭐⭐ Expert |
| 25.5 | Outbox Pattern — Reliable Event Publishing | [05-outbox-pattern.md](./chapters/25-design-patterns/05-outbox-pattern.md) | ⭐⭐⭐⭐ Expert |
| 25.6 | Sidecar Pattern, Ambassador Pattern & Adapter Pattern | [06-sidecar-ambassador-adapter.md](./chapters/25-design-patterns/06-sidecar-ambassador-adapter.md) | ⭐⭐⭐⭐ Expert |
| 25.7 | Data Partitioning Strategies for Planet-Scale Apps | [07-data-partitioning-strategies.md](./chapters/25-design-patterns/07-data-partitioning-strategies.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 25.8 | Multi-Tenancy Architecture | [08-multi-tenancy.md](./chapters/25-design-patterns/08-multi-tenancy.md) | ⭐⭐⭐⭐ Expert |

---

### **PART 26: COST OPTIMIZATION & CAPACITY PLANNING**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 26.1 | Server Sizing — How Much CPU, RAM & Disk Do You Need? | [01-server-sizing.md](./chapters/26-cost-optimization/01-server-sizing.md) | ⭐⭐ Intermediate |
| 26.2 | Cost Optimization in Cloud (Reserved, Spot, On-Demand) | [02-cloud-cost-optimization.md](./chapters/26-cost-optimization/02-cloud-cost-optimization.md) | ⭐⭐⭐ Advanced |
| 26.3 | Capacity Planning — Preparing for Growth | [03-capacity-planning.md](./chapters/26-cost-optimization/03-capacity-planning.md) | ⭐⭐⭐⭐ Expert |
| 26.4 | FinOps — Managing Cloud Costs at Scale | [04-finops.md](./chapters/26-cost-optimization/04-finops.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **PART 27: ARCHITECTURE EVOLUTION — The Journey of a Real App**

| #  | Topic | File | Level |
|----|-------|------|-------|
| 27.1 | Stage 1: Single Server — 0 to 100 Users | [01-stage1-single-server.md](./chapters/27-architecture-evolution/01-stage1-single-server.md) | ⭐ Beginner |
| 27.2 | Stage 2: Separate DB — 100 to 1,000 Users | [02-stage2-separate-db.md](./chapters/27-architecture-evolution/02-stage2-separate-db.md) | ⭐⭐ Intermediate |
| 27.3 | Stage 3: Load Balancer + Multiple Servers — 1K to 10K Users | [03-stage3-load-balancer.md](./chapters/27-architecture-evolution/03-stage3-load-balancer.md) | ⭐⭐ Intermediate |
| 27.4 | Stage 4: Caching + CDN — 10K to 100K Users | [04-stage4-caching-cdn.md](./chapters/27-architecture-evolution/04-stage4-caching-cdn.md) | ⭐⭐⭐ Advanced |
| 27.5 | Stage 5: DB Replication + Read Replicas — 100K to 1M Users | [05-stage5-db-replication.md](./chapters/27-architecture-evolution/05-stage5-db-replication.md) | ⭐⭐⭐ Advanced |
| 27.6 | Stage 6: Microservices + Message Queues — 1M to 10M Users | [06-stage6-microservices.md](./chapters/27-architecture-evolution/06-stage6-microservices.md) | ⭐⭐⭐⭐ Expert |
| 27.7 | Stage 7: Sharding + Multi-Region — 10M to 100M Users | [07-stage7-sharding-multi-region.md](./chapters/27-architecture-evolution/07-stage7-sharding-multi-region.md) | ⭐⭐⭐⭐⭐ Planet-Scale |
| 27.8 | Stage 8: Planet-Scale — 100M to Billions of Users | [08-stage8-planet-scale.md](./chapters/27-architecture-evolution/08-stage8-planet-scale.md) | ⭐⭐⭐⭐⭐ Planet-Scale |

---

### **APPENDICES**

| #  | Topic | File | Level |
|----|-------|------|-------|
| A.1 | Glossary of Terms — Every Term Explained Simply | [A1-glossary.md](./chapters/appendices/A1-glossary.md) | All |
| A.2 | Cheat Sheet — Quick Reference for Architecture Decisions | [A2-cheat-sheet.md](./chapters/appendices/A2-cheat-sheet.md) | All |
| A.3 | Recommended Reading & Resources | [A3-resources.md](./chapters/appendices/A3-resources.md) | All |
| A.4 | System Design Interview Preparation Guide | [A4-interview-prep.md](./chapters/appendices/A4-interview-prep.md) | All |

---

## 📊 VISUAL ROADMAP

```
                        YOUR LEARNING JOURNEY
                        =====================

    ⭐ BEGINNER                    ⭐⭐ INTERMEDIATE
    ┌─────────────┐               ┌──────────────────┐
    │ How Web     │               │ CDN, Caching     │
    │ Works       │──────────────▶│ Load Balancing   │
    │ HTTP, DNS   │               │ REST APIs        │
    │ Client-     │               │ Databases (SQL/  │
    │ Server      │               │ NoSQL)           │
    └─────────────┘               └────────┬─────────┘
                                           │
                                           ▼
    ⭐⭐⭐ ADVANCED                ⭐⭐⭐⭐ EXPERT
    ┌─────────────────┐           ┌──────────────────────┐
    │ Microservices   │           │ Distributed Systems  │
    │ Message Queues  │──────────▶│ CQRS, Event Sourcing │
    │ Circuit Breaker │           │ Consensus Algorithms │
    │ Auto Scaling    │           │ Sharding, Partitions │
    │ Kubernetes      │           │ Service Mesh         │
    └─────────────────┘           └────────┬─────────────┘
                                           │
                                           ▼
                          ⭐⭐⭐⭐⭐ PLANET-SCALE
                          ┌──────────────────────────┐
                          │ Google / Amazon / Netflix │
                          │ Architecture             │
                          │ Multi-Region, Billions   │
                          │ of Users                 │
                          │ Chaos Engineering        │
                          │ Stream Processing        │
                          └──────────────────────────┘
```

---

## 📈 STATS

| Metric | Count |
|--------|-------|
| Total Parts | 27 + Appendices |
| Total Topics | 170+ |
| Beginner Topics | ~20 |
| Intermediate Topics | ~35 |
| Advanced Topics | ~50 |
| Expert Topics | ~40 |
| Planet-Scale Topics | ~25 |

---

> **Next Step**: Tell me which chapter or topic you'd like me to create first, or say **"start from chapter 1"** and I'll begin creating all the files in order!
