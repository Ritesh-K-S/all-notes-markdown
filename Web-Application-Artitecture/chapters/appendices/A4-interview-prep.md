# System Design Interview Preparation Guide

> A structured approach to acing system design interviews — covering framework, common questions, estimation techniques, and how to think like a senior engineer in 45 minutes.

---

## The Framework — How to Approach ANY System Design Question

```
┌──────────────────────────────────────────────────────────────────┐
│              THE 4-STEP FRAMEWORK (45 minutes)                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Step 1: REQUIREMENTS & SCOPE (5 minutes)                        │
│  ═══════════════════════════════════════                          │
│  • Ask clarifying questions                                      │
│  • Define functional requirements (what it does)                 │
│  • Define non-functional requirements (scale, latency, etc.)    │
│  • Identify constraints and assumptions                          │
│                                                                   │
│  Step 2: HIGH-LEVEL DESIGN (10 minutes)                          │
│  ═══════════════════════════════════════                          │
│  • Draw the main components (boxes and arrows)                  │
│  • Identify data flow                                            │
│  • Choose core technologies                                      │
│  • Get interviewer buy-in before going deeper                   │
│                                                                   │
│  Step 3: DETAILED DESIGN (20 minutes)                            │
│  ═══════════════════════════════════════                          │
│  • Deep dive into 2-3 most critical components                  │
│  • Discuss data model / schema                                   │
│  • API design                                                    │
│  • Algorithms and data structures                               │
│  • Handle edge cases                                             │
│                                                                   │
│  Step 4: SCALING & TRADE-OFFS (10 minutes)                       │
│  ═══════════════════════════════════════                          │
│  • Identify bottlenecks                                          │
│  • Add caching, load balancing, sharding                        │
│  • Discuss trade-offs of your choices                           │
│  • Mention monitoring, alerting, failure scenarios              │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Questions to Ask (ALWAYS)

Before designing anything, ask:

```
FUNCTIONAL:
• Who are the users? (consumers, businesses, internal?)
• What are the core features? (list top 3-5)
• What can we NOT do? (explicitly exclude)

SCALE:
• How many users? (DAU / MAU)
• How many requests per second?
• How much data stored?
• Read-heavy or write-heavy?

NON-FUNCTIONAL:
• Latency requirements? (< 200ms? < 1s?)
• Availability requirements? (99.9%? 99.99%?)
• Consistency requirements? (strong? eventual OK?)
• Geographic distribution? (single region? global?)

CONSTRAINTS:
• Budget constraints?
• Existing infrastructure to integrate with?
• Regulatory requirements? (GDPR, PCI, etc.)
```

---

## Back-of-the-Envelope Estimation

### Quick Math Formulas

```
┌──────────────────────────────────────────────────────────────────┐
│              ESTIMATION CHEAT SHEET                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  QPS (Queries Per Second):                                       │
│  ═══════════════════════                                         │
│  Daily users × actions/user ÷ 86,400 = Average QPS              │
│  Peak QPS ≈ Average QPS × 2 to 5                                │
│                                                                   │
│  Example: 10M DAU, 10 actions each                              │
│  Average: 10M × 10 / 86,400 ≈ 1,157 QPS                       │
│  Peak: ~3,000-5,000 QPS                                         │
│                                                                   │
│  ─────────────────────────────────────────────────────           │
│                                                                   │
│  Storage:                                                        │
│  ════════                                                        │
│  Daily data = users × actions × data_per_action                 │
│  Yearly data = Daily × 365                                       │
│  5-year data = Yearly × 5 (plan for growth!)                    │
│                                                                   │
│  Example: 10M users, 5 posts/day, 1KB each                     │
│  Daily: 10M × 5 × 1KB = 50 GB/day                              │
│  Yearly: 50 × 365 = 18 TB/year                                  │
│  5 years: ~100 TB (with replication × 3 = 300 TB)              │
│                                                                   │
│  ─────────────────────────────────────────────────────           │
│                                                                   │
│  Bandwidth:                                                      │
│  ══════════                                                      │
│  QPS × average_response_size = bandwidth                        │
│                                                                   │
│  Example: 5,000 QPS × 100KB average = 500 MB/s = 4 Gbps       │
│                                                                   │
│  ─────────────────────────────────────────────────────           │
│                                                                   │
│  Servers needed:                                                 │
│  ═══════════════                                                 │
│  Peak QPS ÷ capacity_per_server = number of servers             │
│  (add 30% buffer for safety)                                    │
│                                                                   │
│  Example: 5,000 QPS, each server handles 1,000 QPS             │
│  Servers = 5,000/1,000 × 1.3 ≈ 7 servers                      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Useful Numbers to Memorize

```
• 1 day = 86,400 seconds ≈ 100,000 seconds
• 1 month ≈ 2.5 million seconds
• 1 year ≈ 30 million seconds

• 1 KB ≈ a tweet / short text message
• 1 MB ≈ a high-res photo
• 1 GB ≈ a short HD movie
• 1 TB ≈ 1 million photos or 500 hours of video

• 1 server ≈ 1,000-10,000 QPS (depending on work)
• Redis ≈ 100,000+ ops/sec
• PostgreSQL ≈ 10,000-50,000 queries/sec
• Kafka broker ≈ 100,000+ messages/sec
```

---

## Common System Design Questions

### Tier 1: Most Frequently Asked

| # | Question | Key Concepts to Cover |
|---|----------|----------------------|
| 1 | **Design a URL Shortener** (TinyURL) | Hashing, base62 encoding, read-heavy, caching, analytics |
| 2 | **Design a Chat System** (WhatsApp) | WebSockets, message queue, presence, delivery status, E2EE |
| 3 | **Design a News Feed** (Twitter/Instagram) | Fan-out on write vs read, caching, ranking, timeline |
| 4 | **Design a Rate Limiter** | Token bucket, sliding window, distributed rate limiting (Redis) |
| 5 | **Design a Notification System** | Push notifications, email, SMS, priority queue, user preferences |
| 6 | **Design a File Storage** (Google Drive/Dropbox) | Chunking, dedup, sync, conflict resolution, metadata DB |

### Tier 2: Frequently Asked

| # | Question | Key Concepts to Cover |
|---|----------|----------------------|
| 7 | **Design YouTube/Netflix** | Video upload, transcoding, CDN, adaptive streaming, recommendations |
| 8 | **Design an E-Commerce System** (Amazon) | Product catalog, cart, inventory, payments, order processing |
| 9 | **Design a Search Engine** | Crawling, indexing, ranking, inverted index, autocomplete |
| 10 | **Design a Ride-Sharing App** (Uber) | Geospatial indexing, real-time matching, ETA, surge pricing |
| 11 | **Design a Key-Value Store** | Consistent hashing, replication, conflict resolution, Dynamo-style |
| 12 | **Design a Web Crawler** | BFS/DFS, politeness, dedup, distributed crawling, URL frontier |

### Tier 3: Advanced / Senior Level

| # | Question | Key Concepts to Cover |
|---|----------|----------------------|
| 13 | **Design a Payment System** (Stripe) | Idempotency, exactly-once, ledger, reconciliation, fraud detection |
| 14 | **Design a Metrics/Monitoring System** | Time-series DB, aggregation, alerting rules, dashboards |
| 15 | **Design a Distributed Cache** (Redis) | Consistent hashing, eviction, replication, partitioning |
| 16 | **Design a Ticket Booking System** (BookMyShow) | Seat locking, concurrency, fairness, waiting queue |
| 17 | **Design Google Maps** | Tile-based maps, routing (Dijkstra/A*), real-time traffic, ETA |
| 18 | **Design a Distributed Task Scheduler** | Job queue, priority, retry, at-least-once, exactly-once |

---

## Design Template for Each Question

Use this template when practicing:

```
┌─────────────────────────────────────────────────────────────┐
│  SYSTEM: [Name]                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. REQUIREMENTS                                            │
│     Functional: [list 3-5 core features]                    │
│     Non-Functional: [scale, latency, availability]          │
│     Estimations: [QPS, storage, bandwidth]                  │
│                                                              │
│  2. HIGH-LEVEL DESIGN                                       │
│     [Draw architecture diagram]                             │
│     Components: [Client, LB, API, Service, DB, Cache, MQ] │
│                                                              │
│  3. DATA MODEL                                              │
│     [Tables/Collections with key fields]                    │
│     [Relationships]                                          │
│     [Indexing strategy]                                      │
│                                                              │
│  4. API DESIGN                                              │
│     [Key endpoints with request/response]                   │
│                                                              │
│  5. DETAILED DESIGN                                         │
│     [Deep dive into 2-3 critical paths]                    │
│     [Algorithms used]                                        │
│     [Edge cases handled]                                    │
│                                                              │
│  6. SCALING & RELIABILITY                                   │
│     [Bottlenecks identified]                                │
│     [Scaling strategies applied]                            │
│     [Failure scenarios and mitigation]                      │
│     [Trade-offs discussed]                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## What Interviewers Are Looking For

```
┌──────────────────────────────────────────────────────────────────┐
│              EVALUATION CRITERIA                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ✅ DO:                                                           │
│  • Ask clarifying questions before jumping in                   │
│  • Start with high-level, then go deep                         │
│  • Discuss trade-offs (there's no perfect solution)            │
│  • Mention alternatives you considered                         │
│  • Use back-of-envelope math to justify decisions              │
│  • Think about failure scenarios                               │
│  • Be collaborative (treat it as a design discussion)          │
│  • Draw clear diagrams                                          │
│                                                                   │
│  ❌ DON'T:                                                       │
│  • Jump straight into a solution                               │
│  • Over-engineer from the start                                │
│  • Be silent for long periods                                  │
│  • Ignore the interviewer's hints/questions                    │
│  • Design for 1 billion users when they said 10K               │
│  • Mention technologies without explaining WHY                 │
│  • Get stuck on one component (move on, come back later)       │
│  • Forget about monitoring and failure handling                 │
│                                                                   │
│  What separates SENIOR from JUNIOR:                              │
│  • Juniors: correct but generic solution                        │
│  • Seniors: trade-off analysis, real-world considerations,     │
│    operational concerns, scaling path, evolution strategy       │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Common Trade-Offs to Discuss

```
┌──────────────────────────────────────────────────────────────────┐
│              TRADE-OFFS FRAMEWORK                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Trade-Off                          │ When to Choose What        │
│  ══════════════════════════════════ │ ══════════════════════════ │
│  Consistency vs Availability        │ Banking→CP; Social→AP      │
│  Latency vs Throughput              │ API→latency; Batch→thruput │
│  SQL vs NoSQL                       │ Relations→SQL; Scale→NoSQL │
│  Cache vs Fresh Data                │ Read-heavy→cache; Writes→  │
│                                     │ go direct                  │
│  Sync vs Async communication        │ Need response→sync;        │
│                                     │ Fire-forget→async          │
│  Monolith vs Microservices          │ Small team→mono;           │
│                                     │ Scale team→micro           │
│  Push vs Pull                       │ Few updates→push;          │
│                                     │ Many readers→pull          │
│  Pre-compute vs On-demand           │ Frequent reads→precompute; │
│                                     │ Rare reads→on-demand       │
│  Normalize vs Denormalize           │ Write-heavy→normal;        │
│                                     │ Read-heavy→denormal        │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Example Walkthrough: Design a URL Shortener

### Step 1: Requirements (5 min)

```
Functional:
• Given a long URL, generate a short URL
• Given a short URL, redirect to original
• Optional: analytics (click count, geo)
• Optional: custom aliases
• URLs expire after configurable time

Non-Functional:
• 100M URLs created per month
• 10:1 read-to-write ratio → 1B redirects/month
• URL should be as short as possible
• High availability (redirects should never fail)
• Low latency redirects (< 50ms)

Estimations:
• Write QPS: 100M / (30 × 86400) ≈ 40 QPS
• Read QPS: 40 × 10 = 400 QPS (peak: ~2000 QPS)
• Storage: 100M × 12 months × 5 years × 500 bytes ≈ 3 TB
• Bandwidth: 2000 × 500 bytes = 1 MB/s (trivial)
```

### Step 2: High-Level Design (10 min)

```
┌────────┐    ┌────────┐    ┌──────────────┐    ┌─────────┐
│ Client │───▶│  Load  │───▶│ URL Shortener │───▶│  Cache  │
│        │    │Balancer│    │   Service     │    │ (Redis) │
└────────┘    └────────┘    └──────────────┘    └─────────┘
                                    │                    │
                                    ▼                    │
                            ┌──────────────┐            │
                            │  Database    │◀───────────┘
                            │ (PostgreSQL) │
                            └──────────────┘
```

### Step 3: Detailed Design (20 min)

```
URL Shortening Algorithm:
• Use base62 encoding (a-z, A-Z, 0-9) = 62 characters
• 7 characters → 62^7 = 3.5 trillion possible URLs (enough!)
• Approach: Counter-based (auto-increment ID → base62)
  OR: Hash-based (MD5 of URL → take first 7 chars)

Database Schema:
┌─────────────────────────────────────┐
│ urls                                 │
├─────────────────────────────────────┤
│ id (bigint, PK)                     │
│ short_code (varchar(7), unique idx) │
│ original_url (text)                 │
│ created_at (timestamp)              │
│ expires_at (timestamp, nullable)    │
│ user_id (bigint, nullable, FK)      │
│ click_count (bigint, default 0)     │
└─────────────────────────────────────┘

API:
• POST /api/shorten  {url: "...", expiry: "..."}  → {short_url}
• GET /:code  → 301 Redirect to original URL

Read Path (critical for latency):
1. GET /abc1234
2. Check Redis cache → if hit, redirect (< 5ms)
3. If miss, query DB → cache result → redirect
4. Async: increment click count (via Kafka)
```

### Step 4: Scaling (10 min)

```
Bottleneck: Database for writes (40 QPS → fine for now)
Bottleneck: Cache for reads (2000 QPS → single Redis handles it)

For 100× scale:
• Database: Range-based sharding by short_code first char
• Cache: Redis cluster (partition by short_code)
• ID generation: Distributed ID service (Snowflake-style)
• Analytics: Kafka → Flink → OLAP DB (ClickHouse)

Trade-offs discussed:
• 301 vs 302 redirect (301 = cached by browser, fewer hits but less analytics)
• Hash vs counter (hash = no coordination; counter = no collision)
• Chosen: Counter with distributed ID generator (Snowflake) for scale
```

---

## Preparation Timeline

```
┌──────────────────────────────────────────────────────────────────┐
│              INTERVIEW PREP PLAN                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  WEEK 1-2: Foundations                                           │
│  • Study this entire guide (Parts 1-11)                         │
│  • Understand: LB, caching, DB, queues, APIs                   │
│  • Practice: 2 easy designs (URL shortener, paste bin)          │
│                                                                   │
│  WEEK 3-4: Core Designs                                          │
│  • Study: Parts 12-19 (resilience, distributed systems)         │
│  • Practice: 4 medium designs (chat, news feed, rate limiter,   │
│    notification system)                                          │
│  • Time yourself: 45 minutes per design                         │
│                                                                   │
│  WEEK 5-6: Advanced & Mock                                       │
│  • Study: Parts 20-27 (real-world systems)                      │
│  • Practice: 4 hard designs (YouTube, Uber, payment system)    │
│  • Do mock interviews with friends or pramp.com                 │
│  • Focus on articulating trade-offs                             │
│                                                                   │
│  WEEK 7-8: Polish                                                │
│  • Review all designs, identify weak areas                      │
│  • Practice explaining under time pressure                      │
│  • Read 2-3 engineering blogs about real architectures          │
│  • Build confidence: you know more than you think!              │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Final Tips

```
┌──────────────────────────────────────────────────────────────────┐
│              GOLDEN RULES                                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. There is NO perfect answer — show your THINKING PROCESS     │
│                                                                   │
│  2. Start simple, then scale — don't jump to Kafka + K8s       │
│     for 100 users                                                │
│                                                                   │
│  3. Numbers matter — "about 1000 QPS" shows you understand     │
│     scale; "a lot of traffic" shows you don't                   │
│                                                                   │
│  4. Every choice has a trade-off — say WHY you chose it        │
│     and what you're giving up                                    │
│                                                                   │
│  5. Think about failure — "what happens when X goes down?"     │
│     separates senior from junior                                 │
│                                                                   │
│  6. Draw diagrams — a picture is worth 1000 words              │
│     (use boxes, arrows, labels)                                  │
│                                                                   │
│  7. Be collaborative — ask "should I dive deeper into X        │
│     or move to Y?" Let the interviewer guide you.              │
│                                                                   │
│  8. Practice out loud — explaining while designing is a        │
│     skill that needs practice                                    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

> **"System design interviews are not about getting the 'right' answer. They're about demonstrating that you can think through complex problems systematically, make informed trade-offs, and communicate your decisions clearly."**

---

> **Good luck! You've got this.** 🎯
