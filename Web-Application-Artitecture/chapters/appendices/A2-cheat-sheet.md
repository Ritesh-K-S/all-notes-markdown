# Cheat Sheet — Quick Reference for Architecture Decisions

> A one-page decision guide for the most common architecture questions. Print this, bookmark it, come back to it when designing systems.

---

## 🎯 Database Selection Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────┐
│                 WHICH DATABASE SHOULD I USE?                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Need                          │ Use                                 │
│  ═════════════════════════════ │ ════════════════════════════════    │
│  Structured data + relations   │ PostgreSQL / MySQL                  │
│  Flexible schema, documents    │ MongoDB / CouchDB                   │
│  Simple key-value lookups      │ Redis / DynamoDB                    │
│  Time-series data (IoT, logs)  │ InfluxDB / TimescaleDB             │
│  Relationships/graphs          │ Neo4j / Amazon Neptune              │
│  Full-text search              │ Elasticsearch / OpenSearch          │
│  Wide-column, massive scale    │ Cassandra / HBase                   │
│  Global distribution + SQL     │ CockroachDB / Google Spanner       │
│  Caching (fast reads)          │ Redis / Memcached                   │
│  Analytics (OLAP)              │ ClickHouse / BigQuery / Redshift   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Architecture Pattern Decision Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│               WHEN TO USE WHICH ARCHITECTURE?                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Situation                              │ Pattern                    │
│  ══════════════════════════════════════ │ ══════════════════════════ │
│  Small team, MVP, startup               │ Monolith                   │
│  Monolith getting too big               │ Modular Monolith           │
│  Independent teams, fast deploys        │ Microservices              │
│  Migrating from monolith                │ Strangler Fig              │
│  High read/write asymmetry              │ CQRS                       │
│  Need complete audit trail              │ Event Sourcing             │
│  Async workflows, decoupled services   │ Event-Driven (EDA)         │
│  Minimal ops, variable traffic          │ Serverless                 │
│  Complex business domain                │ Domain-Driven Design       │
│  Multiple teams sharing services        │ SOA                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📡 Communication Pattern Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────┐
│              HOW SHOULD SERVICES COMMUNICATE?                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Scenario                            │ Protocol/Pattern              │
│  ═══════════════════════════════════ │ ═══════════════════════════   │
│  Simple CRUD operations              │ REST (HTTP/JSON)              │
│  Client needs flexible queries       │ GraphQL                       │
│  Internal service-to-service (fast)  │ gRPC (Protobuf)              │
│  Real-time bidirectional             │ WebSockets                    │
│  Server → Client push (one-way)     │ Server-Sent Events (SSE)     │
│  Fire-and-forget, decouple services  │ Message Queue (RabbitMQ)     │
│  Event streaming at scale            │ Apache Kafka                  │
│  Long-running workflows              │ Saga Pattern                  │
│                                                                      │
│  Decision Tree:                                                      │
│  Is it synchronous?                                                  │
│    ├── Yes → Is it external API? → REST / GraphQL                   │
│    │       → Is it internal? → gRPC                                  │
│    └── No  → Do you need ordering? → Kafka                          │
│            → Simple task queue? → RabbitMQ / SQS                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Caching Strategy Decision Tree

```
┌─────────────────────────────────────────────────────────────────────┐
│              WHICH CACHING STRATEGY?                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Scenario                              │ Strategy                    │
│  ═════════════════════════════════════ │ ═══════════════════════    │
│  Read-heavy, cache miss tolerance OK   │ Cache-Aside (Lazy)         │
│  Always want cache fresh               │ Write-Through               │
│  Write-heavy, read eventually          │ Write-Behind (Write-Back)   │
│  Data rarely read after write          │ Write-Around                │
│  Static assets (CSS, JS, images)       │ CDN caching                 │
│  API responses (per-user)              │ Application cache (Redis)  │
│  Database query results                │ Query cache                  │
│                                                                      │
│  Eviction Policy:                                                    │
│  • LRU — When you want "recently popular" items cached             │
│  • LFU — When you want "most frequently used" items cached         │
│  • TTL — When data expires after fixed time (DNS, sessions)        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## ⚖️ Load Balancing Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│              LOAD BALANCING ALGORITHMS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Algorithm             │ When to Use                                 │
│  ═════════════════════ │ ════════════════════════════════════════    │
│  Round Robin           │ All servers equal, stateless requests       │
│  Weighted Round Robin  │ Servers have different capacities           │
│  Least Connections     │ Long-lived connections (WebSocket)          │
│  IP Hash               │ Need session affinity without cookies       │
│  Least Response Time   │ Heterogeneous server performance           │
│  Random                │ Simple, surprisingly effective              │
│                                                                      │
│  Layer 4 vs Layer 7:                                                │
│  • L4: Fast, simple (TCP/UDP level). Can't inspect content.        │
│  • L7: Slower, smart (HTTP level). Can route by URL, header.       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🛡️ Resilience Patterns Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│              RESILIENCE PATTERNS — WHEN TO USE WHAT                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Problem                              │ Pattern                      │
│  ════════════════════════════════════ │ ════════════════════════     │
│  Downstream service slow/dead         │ Circuit Breaker              │
│  Transient failures (network blips)   │ Retry with Exponential      │
│                                       │ Backoff + Jitter             │
│  One service failure affecting all    │ Bulkhead                     │
│  Need safe-to-retry operations        │ Idempotency Keys            │
│  System overwhelmed                   │ Backpressure + Rate Limit   │
│  Partial functionality better         │ Graceful Degradation         │
│  than total failure                   │                              │
│  Waiting forever for response         │ Timeouts (connect + read)   │
│  Need fallback data                   │ Fallback Pattern             │
│                                                                      │
│  Always combine: Timeout + Retry + Circuit Breaker = Safety Net    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Scaling Decision Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│              SCALING DECISIONS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Users      │ What to Do                                            │
│  ══════════ │ ═══════════════════════════════════════════════════   │
│  0-100      │ Single server. Don't over-engineer.                   │
│  100-1K     │ Separate DB. Add caching.                             │
│  1K-10K     │ Load balancer + multiple app servers                  │
│  10K-100K   │ CDN + Redis + Read replicas                           │
│  100K-1M    │ Microservices + Message queues + DB replication       │
│  1M-10M     │ Sharding + Multi-region + Event-driven               │
│  10M-100M   │ Full distributed systems + Custom solutions          │
│  100M+      │ Planet-scale: everything above + innovation          │
│                                                                      │
│  Vertical vs Horizontal:                                            │
│  • Try vertical first (cheaper, simpler) until you can't          │
│  • Go horizontal when: need HA, hit machine limits, need          │
│    geographic distribution                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Security Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│              SECURITY — MINIMUM REQUIREMENTS                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ✅ HTTPS everywhere (TLS 1.3)                                      │
│  ✅ Input validation & sanitization                                  │
│  ✅ Parameterized queries (prevent SQL injection)                    │
│  ✅ CORS properly configured                                         │
│  ✅ Rate limiting on all endpoints                                   │
│  ✅ JWT with short expiry + refresh tokens                           │
│  ✅ Secrets in vault (never in code/env vars in plain text)         │
│  ✅ Least privilege access (RBAC)                                    │
│  ✅ Audit logging                                                    │
│  ✅ Dependencies scanned for vulnerabilities                         │
│  ✅ Security headers (CSP, X-Frame-Options, etc.)                   │
│  ✅ WAF in front of public endpoints                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📏 Key Numbers Every Developer Should Know

```
┌─────────────────────────────────────────────────────────────────────┐
│              LATENCY NUMBERS (APPROXIMATE)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Operation                          │ Time                           │
│  ══════════════════════════════════ │ ═══════════════════           │
│  L1 cache reference                 │ 0.5 ns                        │
│  L2 cache reference                 │ 7 ns                          │
│  Main memory reference              │ 100 ns                        │
│  SSD random read                    │ 16 μs                         │
│  Read 1 MB from memory              │ 250 μs                        │
│  Read 1 MB from SSD                 │ 1 ms                          │
│  Read 1 MB from HDD                 │ 20 ms                         │
│  Round trip same datacenter         │ 0.5 ms                        │
│  Round trip US coast to coast       │ 40 ms                         │
│  Round trip US to Europe            │ 80 ms                         │
│  Round trip US to India             │ 150 ms                        │
│  Redis GET                          │ 0.1 ms                        │
│  PostgreSQL simple query            │ 1-5 ms                        │
│  HTTP API call (same region)        │ 5-50 ms                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              AVAILABILITY NUMBERS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Availability │ Downtime/Year │ Downtime/Month │ Downtime/Week     │
│  ════════════ │ ═════════════ │ ══════════════ │ ═══════════════   │
│  99%          │ 3.65 days     │ 7.3 hours      │ 1.68 hours       │
│  99.9%        │ 8.77 hours    │ 43.8 minutes   │ 10.1 minutes     │
│  99.99%       │ 52.6 minutes  │ 4.38 minutes   │ 1.01 minutes     │
│  99.999%      │ 5.26 minutes  │ 26.3 seconds   │ 6.05 seconds     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              CAPACITY ESTIMATES (RULES OF THUMB)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  • 1 web server ≈ 1,000-10,000 concurrent connections             │
│  • 1 PostgreSQL instance ≈ 10,000-50,000 queries/sec              │
│  • 1 Redis instance ≈ 100,000+ operations/sec                     │
│  • 1 Kafka broker ≈ 100,000+ messages/sec                         │
│  • 1 MB of data ≈ 1 tweet, 1 small image                         │
│  • 1 GB = 1,000 tweets, 500 high-res photos                      │
│  • Daily Active Users × 10 ≈ Peak requests/day                    │
│  • Peak QPS ≈ (DAU × queries/user) / 86,400 × 3 (peak factor)  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🧭 Technology Stack Recommendations

```
┌─────────────────────────────────────────────────────────────────────┐
│              MODERN TECH STACK (2024-2025)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Layer              │ Popular Choices                                │
│  ═════════════════ │ ══════════════════════════════════════════     │
│  Frontend           │ React / Next.js / Vue / Svelte                │
│  Backend            │ Node.js / Go / Java (Spring) / Python (Fast)  │
│  API Gateway        │ Kong / AWS API Gateway / Envoy               │
│  Database (SQL)     │ PostgreSQL / MySQL                            │
│  Database (NoSQL)   │ MongoDB / DynamoDB                            │
│  Cache              │ Redis / Memcached                              │
│  Message Queue      │ Kafka / RabbitMQ / AWS SQS                   │
│  Search             │ Elasticsearch / Meilisearch                   │
│  Container Runtime  │ Docker + Kubernetes                           │
│  CI/CD              │ GitHub Actions / GitLab CI / ArgoCD          │
│  Monitoring         │ Prometheus + Grafana / Datadog                │
│  Logging            │ ELK Stack / Loki + Grafana                   │
│  Tracing            │ OpenTelemetry + Jaeger                        │
│  Cloud              │ AWS / GCP / Azure                             │
│  CDN                │ Cloudflare / AWS CloudFront / Fastly         │
│  DNS                │ Cloudflare DNS / Route 53                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🎓 CAP Theorem Quick Lookup

```
┌─────────────────────────────────────────────────────────────────────┐
│              CAP THEOREM — DATABASE CLASSIFICATION                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CP (Consistency + Partition Tolerance):                             │
│  • MongoDB (in strict mode), HBase, Redis (cluster)                │
│  • Trade: May reject requests during partition                      │
│                                                                      │
│  AP (Availability + Partition Tolerance):                            │
│  • Cassandra, DynamoDB, CouchDB                                     │
│  • Trade: May return stale data during partition                    │
│                                                                      │
│  CA (Consistency + Availability):                                    │
│  • Single-node PostgreSQL, MySQL (non-distributed)                  │
│  • Trade: Can't survive network partitions (single point failure)  │
│                                                                      │
│  Note: In practice, partition tolerance is required in distributed  │
│  systems, so the real choice is CP vs AP.                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

> **Usage**: Keep this cheat sheet handy during system design discussions, interviews, and architecture reviews. It won't replace deep understanding, but it will help you make quick, informed decisions.
