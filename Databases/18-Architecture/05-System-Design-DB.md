# 🏗️ Chapter 7.5 — Database System Design — Interview Prep

> **Level:** 🔴 Advanced | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~6-8 hours
> **Prerequisites:** All of Part 7 (Chapters 7.1-7.4), Part 1 Foundations

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Design databases for **5 real-world systems** from scratch
- Apply **every pattern** from Part 7 (sharding, replication, caching, sagas, CQRS)
- Think like a **Staff/Principal engineer** in system design interviews
- Know the **database trade-offs** that interviewers look for
- Have a **repeatable framework** for approaching any database design problem
- Confidently answer: "How would you design the database for X?"

---

## 🧠 The System Design Interview Framework (DB-Focused)

```
┌──────────────────────────────────────────────────────────────────┐
│          DATABASE SYSTEM DESIGN FRAMEWORK                        │
│          ═════════════════════════════════                       │
│                                                                   │
│  STEP 1: REQUIREMENTS (2-3 minutes)                              │
│  ─────────────────────────────────                               │
│  • Functional: What does the system DO?                          │
│  • Non-functional: Scale? Latency? Consistency? Availability?   │
│  • Read/Write ratio: Read-heavy? Write-heavy? Balanced?         │
│  • Data volume: How much data? Growth rate?                      │
│  • Query patterns: What queries are most common?                 │
│                                                                   │
│  STEP 2: DATA MODEL (5-8 minutes)                                │
│  ─────────────────────────────────                               │
│  • Entities and relationships                                     │
│  • Choose: SQL vs NoSQL (with reasoning)                         │
│  • Schema design (tables/collections)                            │
│  • Indexes                                                        │
│                                                                   │
│  STEP 3: ARCHITECTURE (5-8 minutes)                              │
│  ──────────────────────────────────                              │
│  • Single DB or distributed?                                     │
│  • Replication strategy                                           │
│  • Sharding (if needed) + shard key selection                    │
│  • Caching layer                                                  │
│  • Read/Write path (how data flows)                              │
│                                                                   │
│  STEP 4: DEEP DIVE (5-10 minutes)                                │
│  ─────────────────────────────────                               │
│  • Handle edge cases and failure scenarios                       │
│  • Consistency vs availability trade-offs                        │
│  • Scale to 10x, 100x                                            │
│  • Monitor and observe                                            │
│                                                                   │
│  STEP 5: TRADE-OFFS (2-3 minutes)                                │
│  ─────────────────────────────────                               │
│  • What did we sacrifice? Why?                                   │
│  • Alternative approaches considered                              │
│  • What breaks at 1000x scale?                                   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔗 Design 1: URL Shortener (like bit.ly)

### Requirements

```
  Functional:
  • Shorten long URL → short URL (e.g., https://short.ly/abc123)
  • Redirect short URL → original long URL
  • Optional: Custom aliases, expiration, click analytics

  Non-Functional:
  • 100M URLs created per month (~40 URLs/sec writes)
  • 10B redirects per month (~4,000 reads/sec)
  • Read:Write ratio = 100:1 (extremely read-heavy)
  • Redirect latency < 50ms
  • URLs should be unique and not predictable
  • High availability (redirects must NEVER fail)
  • Eventually consistent is OK for analytics
```

### Data Model

```sql
-- ═══════════════════════════════════════════════════════
-- Option A: Relational (PostgreSQL / MySQL)
-- ═══════════════════════════════════════════════════════

CREATE TABLE urls (
    short_code   VARCHAR(8) PRIMARY KEY,    -- "abc123"
    long_url     TEXT NOT NULL,             -- Original URL
    user_id      BIGINT,                    -- Who created it
    created_at   TIMESTAMP DEFAULT NOW(),
    expires_at   TIMESTAMP,                 -- Optional expiry
    click_count  BIGINT DEFAULT 0           -- Quick counter
);

-- Index for reverse lookup (find short code by long URL)
CREATE INDEX idx_long_url ON urls (long_url);

-- Index for user's URLs
CREATE INDEX idx_user_urls ON urls (user_id, created_at DESC);

-- Analytics table (append-only, write-heavy)
CREATE TABLE clicks (
    click_id     BIGSERIAL PRIMARY KEY,
    short_code   VARCHAR(8) NOT NULL,
    clicked_at   TIMESTAMP DEFAULT NOW(),
    ip_address   INET,
    user_agent   TEXT,
    referrer     TEXT,
    country      VARCHAR(2)
);

-- Partition clicks by month (it'll grow FAST)
CREATE TABLE clicks (
    click_id     BIGSERIAL,
    short_code   VARCHAR(8) NOT NULL,
    clicked_at   TIMESTAMP DEFAULT NOW(),
    ip_address   INET,
    country      VARCHAR(2)
) PARTITION BY RANGE (clicked_at);

CREATE TABLE clicks_2025_01 PARTITION OF clicks
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE clicks_2025_02 PARTITION OF clicks
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

```javascript
// ═══════════════════════════════════════════════════════
// Option B: NoSQL (DynamoDB / MongoDB) — Better fit!
// ═══════════════════════════════════════════════════════

// DynamoDB Table Design (single-table):
// Table: urls
// PK: short_code
// Attributes: long_url, user_id, created_at, expires_at, ttl

// Why DynamoDB is ideal:
// • Simple key-value lookups (perfect for DynamoDB)
// • Managed, auto-scales
// • TTL built-in (auto-delete expired URLs)
// • DAX (DynamoDB Accelerator) for caching = sub-ms reads

// MongoDB alternative:
db.urls.createIndex({ short_code: 1 }, { unique: true })
db.urls.createIndex({ long_url: 1 })
db.urls.createIndex({ expires_at: 1 }, { expireAfterSeconds: 0 }) // TTL index!
```

### Architecture

```
                URL SHORTENER ARCHITECTURE
                ═════════════════════════

    ┌──────────────────────────────────────────────────────────┐
    │                                                           │
    │    Client                                                 │
    │      │                                                    │
    │      ▼                                                    │
    │  ┌──────────┐                                             │
    │  │   CDN    │ ← Cache redirect (301) at edge              │
    │  │(CloudFlare)                                            │
    │  └────┬─────┘                                             │
    │       │ MISS                                               │
    │       ▼                                                    │
    │  ┌──────────┐         ┌──────────┐                        │
    │  │  Load    │────────►│ App      │                        │
    │  │ Balancer │         │ Servers  │                        │
    │  └──────────┘         │ (×10)    │                        │
    │                       └────┬─────┘                        │
    │                            │                               │
    │              ┌─────────────┼──────────────┐               │
    │              │             │              │                │
    │              ▼             ▼              ▼                │
    │        ┌──────────┐  ┌────────┐    ┌──────────┐          │
    │        │  Redis   │  │  DB    │    │  Kafka   │          │
    │        │  Cache   │  │(Primary│    │(Analytics│          │
    │        │          │  │+ Read  │    │ events)  │          │
    │        │ Hot URLs │  │Replicas│    │          │          │
    │        │ ~90% hit │  │)       │    │    │     │          │
    │        └──────────┘  └────────┘    │    ▼     │          │
    │                                    │Analytics │          │
    │                                    │  DB      │          │
    │                                    │(ClickHse)│          │
    │                                    └──────────┘          │
    │                                                           │
    └──────────────────────────────────────────────────────────┘

    READ PATH (Redirect):
    1. CDN check → HIT? Return 301 redirect ⚡
    2. Redis check → HIT? Return redirect
    3. DB query → Store in Redis → Return redirect

    WRITE PATH (Create short URL):
    1. Generate unique short_code (Base62 encoding)
    2. Write to DB
    3. Write to Redis cache
    4. Return short URL

    ANALYTICS PATH:
    1. On redirect → publish click event to Kafka (async)
    2. Kafka → Analytics service → ClickHouse (batch insert)
    3. Dashboard reads from ClickHouse
```

### Key Design Decisions

```
┌──────────────────────────────────────────────────────────────────┐
│  DECISION 1: Short Code Generation                               │
│                                                                   │
│  Option A: Counter-based                                         │
│  → Auto-increment ID → Base62 encode                            │
│  → ID 12345 → Base62 → "dnh"                                   │
│  → Problem: Predictable (user can guess next URL)               │
│                                                                   │
│  Option B: Hash-based                                            │
│  → MD5(long_url) → Take first 7 chars                          │
│  → Problem: Collisions possible                                  │
│                                                                   │
│  Option C: Pre-generated + Counter (BEST)                        │
│  → Pre-generate pool of unique random codes                     │
│  → Assign from pool atomically (counter server)                 │
│  → No collision, not predictable ✅                              │
│                                                                   │
│  Base62 encoding: [a-z, A-Z, 0-9] = 62 chars                   │
│  7-char code = 62^7 = 3.5 TRILLION combinations                │
│  At 100M/month → lasts 35,000 years! ✅                         │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  DECISION 2: SQL vs NoSQL                                        │
│                                                                   │
│  This is a KEY-VALUE lookup problem!                             │
│  → Given short_code → return long_url                           │
│  → Perfect for: DynamoDB, Redis, MongoDB                        │
│  → SQL works too, but it's overkill for this pattern            │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  DECISION 3: 301 vs 302 Redirect                                │
│                                                                   │
│  301 (Permanent): Browser caches → fewer server hits            │
│                   But: Can't track clicks accurately!            │
│                                                                   │
│  302 (Temporary): Browser always hits server                     │
│                   ✅ Can track every click                       │
│                   Use 302 if analytics matter.                    │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  DECISION 4: Scaling                                             │
│                                                                   │
│  Phase 1 (< 1M URLs): Single PostgreSQL + Redis                 │
│  Phase 2 (< 100M URLs): Read replicas + Redis cluster           │
│  Phase 3 (< 10B URLs): Shard by short_code hash                │
│  Phase 4 (Global): Multi-region with CDN caching                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 💬 Design 2: Chat System (like WhatsApp/Slack)

### Requirements

```
  Functional:
  • 1-on-1 messaging and group chats (up to 1000 members)
  • Message delivery: sent, delivered, read receipts
  • Message history with pagination
  • Online/offline presence indicators
  • Push notifications

  Non-Functional:
  • 50M DAU (Daily Active Users)
  • Average user sends 40 messages/day → 2B messages/day
  • ~23,000 writes/sec, ~100,000 reads/sec
  • Message delivery latency < 200ms
  • Message history: indefinite retention
  • Strong ordering within a conversation
  • No message loss (at-least-once delivery)
```

### Data Model

```
  WHY NOT a single SQL table for messages?
  
  messages table with 2 BILLION rows/day = 730 BILLION rows/year
  → No single database can handle this
  → Need: Sharding + Time-based partitioning

  ═══════════════════════════════════════════════════════
  APPROACH: Cassandra for Messages (Write-Heavy, Append-Only)
  ═══════════════════════════════════════════════════════
```

```cql
-- ═══════════════════════════════════════════════════════
-- Cassandra Schema for Chat Messages
-- ═══════════════════════════════════════════════════════

-- Messages table (partitioned by conversation + time bucket)
CREATE TABLE messages (
    conversation_id  UUID,
    bucket           TEXT,           -- "2025-06-01" (daily bucket)
    message_id       TIMEUUID,      -- Time-based UUID (sortable!)
    sender_id        UUID,
    content          TEXT,
    message_type     TEXT,           -- 'text', 'image', 'video'
    media_url        TEXT,
    created_at       TIMESTAMP,
    PRIMARY KEY ((conversation_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- WHY this design?
-- Partition key: (conversation_id, bucket)
--   → All messages for one conversation in one day = one partition
--   → Prevents unbounded partition growth
--   → Enables efficient "load today's messages" query
--
-- Clustering key: message_id (TIMEUUID)
--   → Messages sorted by time within partition
--   → DESC = newest first (natural for chat scroll)

-- Query: Get recent messages for a conversation
SELECT * FROM messages 
WHERE conversation_id = ? AND bucket = '2025-06-01'
ORDER BY message_id DESC
LIMIT 50;

-- Query: Load older messages (pagination)
SELECT * FROM messages 
WHERE conversation_id = ? AND bucket = '2025-05-31'
ORDER BY message_id DESC
LIMIT 50;
```

```sql
-- ═══════════════════════════════════════════════════════
-- PostgreSQL/MySQL for User & Conversation Metadata
-- ═══════════════════════════════════════════════════════

-- Users (relatively small, relational data)
CREATE TABLE users (
    user_id         UUID PRIMARY KEY,
    username        VARCHAR(50) UNIQUE,
    display_name    VARCHAR(100),
    avatar_url      TEXT,
    last_seen_at    TIMESTAMP,
    is_online       BOOLEAN DEFAULT FALSE
);

-- Conversations (1-on-1 and groups)
CREATE TABLE conversations (
    conversation_id UUID PRIMARY KEY,
    type            VARCHAR(10) NOT NULL, -- 'direct', 'group'
    name            VARCHAR(100),         -- Group name (null for direct)
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Conversation membership
CREATE TABLE conversation_members (
    conversation_id UUID REFERENCES conversations(conversation_id),
    user_id         UUID REFERENCES users(user_id),
    joined_at       TIMESTAMP DEFAULT NOW(),
    last_read_at    TIMESTAMP,            -- For read receipts!
    role            VARCHAR(20) DEFAULT 'member',
    PRIMARY KEY (conversation_id, user_id)
);

-- User's conversation list (denormalized for fast loading)
CREATE TABLE user_conversations (
    user_id             UUID,
    conversation_id     UUID,
    last_message_text   TEXT,             -- Preview text
    last_message_at     TIMESTAMP,
    unread_count        INT DEFAULT 0,
    PRIMARY KEY (user_id, conversation_id)
);

CREATE INDEX idx_user_convos 
    ON user_conversations (user_id, last_message_at DESC);
```

### Architecture

```
                   CHAT SYSTEM ARCHITECTURE
                   ═══════════════════════

    ┌───────────────────────────────────────────────────────────┐
    │                                                           │
    │  Mobile/Web Clients                                       │
    │       │                                                   │
    │       │ WebSocket (persistent connection)                 │
    │       ▼                                                   │
    │  ┌──────────────┐                                         │
    │  │  WebSocket   │ ← Maintains millions of connections    │
    │  │  Gateway     │   Each user connected to ONE gateway    │
    │  │  (×20 nodes) │                                         │
    │  └──────┬───────┘                                         │
    │         │                                                 │
    │         ▼                                                 │
    │  ┌──────────────┐    ┌──────────────┐                     │
    │  │ Message      │───►│    Kafka     │                     │
    │  │ Service      │    │ (per-convo   │                     │
    │  │              │    │  partitions) │                     │
    │  └──────────────┘    └──────┬───────┘                     │
    │                             │                              │
    │              ┌──────────────┼──────────────┐              │
    │              │              │              │               │
    │              ▼              ▼              ▼               │
    │        ┌──────────┐  ┌──────────┐   ┌──────────┐         │
    │        │Cassandra │  │ Redis    │   │ Push     │         │
    │        │(Messages │  │(Presence │   │Notif Svc │         │
    │        │ Storage) │  │ + User   │   │          │         │
    │        │          │  │ Session  │   │(Offline  │         │
    │        │          │  │ → Gateway│   │ users)   │         │
    │        └──────────┘  │ mapping) │   └──────────┘         │
    │                      └──────────┘                         │
    │                                                           │
    │  ┌──────────────┐                                         │
    │  │ PostgreSQL   │ ← User profiles, conversation metadata │
    │  │ (Metadata)   │                                         │
    │  └──────────────┘                                         │
    │                                                           │
    └───────────────────────────────────────────────────────────┘

    MESSAGE FLOW:

    1. Alice sends message to conversation XYZ
    2. WebSocket Gateway receives message
    3. Message Service:
       a. Validate (auth, rate limit, content)
       b. Generate TIMEUUID for message_id
       c. Write to Cassandra (persistent storage)
       d. Publish to Kafka topic: "conversation-XYZ"
    4. Kafka consumers:
       a. Look up members of conversation XYZ (Redis/PostgreSQL)
       b. For each online member:
          → Find their WebSocket Gateway (Redis session store)
          → Forward message to that gateway
          → Gateway pushes to client via WebSocket
       c. For offline members:
          → Send push notification (APNs/FCM)
       d. Update user_conversations table (last_message, unread_count)
```

### Key Design Decisions

```
┌──────────────────────────────────────────────────────────────────┐
│  DECISION: Message Ordering                                      │
│                                                                   │
│  Messages MUST be ordered within a conversation.                 │
│  → Cassandra: TIMEUUID (contains timestamp) as clustering key   │
│  → Kafka: Partition by conversation_id → ordered within partition│
│  → Both ensure per-conversation ordering ✅                      │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  DECISION: Presence (Online/Offline Status)                      │
│                                                                   │
│  Option A: Heartbeat (every 30 seconds)                          │
│  → Client sends ping → Redis SET user:42:online TTL 45s        │
│  → If no ping for 45s → key expires → user is offline           │
│  → At 50M users, 1.6M heartbeats/second!                       │
│                                                                   │
│  Option B: On-demand (check only when needed)                    │
│  → Only check presence when user opens a conversation            │
│  → Much less traffic                                             │
│                                                                   │
│  WhatsApp approach: Presence shown only for contacts,            │
│  not all users. Reduces scope dramatically.                      │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  DECISION: Read Receipts                                         │
│                                                                   │
│  conversation_members.last_read_at = timestamp of last read msg │
│  When user opens chat → UPDATE last_read_at = NOW()             │
│  To show "read" status:                                          │
│  → If message.created_at < recipient.last_read_at → READ ✅    │
│  → Efficient: one timestamp per user per conversation            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📰 Design 3: News Feed / Timeline (like Twitter/Instagram)

### Requirements

```
  Functional:
  • Users create posts (text, images, videos)
  • Users follow other users
  • Home feed: see posts from people you follow, sorted by time
  • Like, comment, share/repost

  Non-Functional:
  • 500M DAU
  • Average user follows 200 people
  • Average user views feed 10x/day → 5B feed reads/day
  • Average user posts 0.5 posts/day → 250M posts/day
  • Read:Write = 20,000:1 (extremely read-heavy)
  • Feed load time < 200ms
  • Eventually consistent feed is acceptable
```

### The Core Database Challenge

```
  THE FANOUT PROBLEM:
  
  Taylor Swift has 100M followers.
  She posts one tweet.
  
  APPROACH A: FANOUT ON WRITE (Push Model)
  → Pre-compute feed: Write this tweet to 100M users' feeds
  → 100 MILLION database writes for ONE tweet! 💀
  → Feed read = just fetch your pre-computed feed (fast!)
  
  APPROACH B: FANOUT ON READ (Pull Model)
  → On feed read: "Get all accounts I follow → get their recent posts → merge & sort"
  → Feed read = 200 queries (one per followed user) + merge
  → Slow for users following many accounts 🐌
  → No writes on post creation ✅
  
  APPROACH C: HYBRID (What Twitter actually does!) ✅
  → Regular users (<10K followers): Fanout on write
  → Celebrities (>10K followers): Fanout on read
  → Celebrity posts merged into feed at read time
```

### Data Model

```sql
-- ═══════════════════════════════════════════════════════
-- PostgreSQL — Core tables (Metadata)
-- ═══════════════════════════════════════════════════════

CREATE TABLE users (
    user_id      BIGINT PRIMARY KEY,
    username     VARCHAR(50) UNIQUE,
    display_name VARCHAR(100),
    follower_count  INT DEFAULT 0,
    following_count INT DEFAULT 0,
    is_celebrity    BOOLEAN DEFAULT FALSE -- > 10K followers
);

CREATE TABLE follows (
    follower_id  BIGINT REFERENCES users(user_id),
    followee_id  BIGINT REFERENCES users(user_id),
    created_at   TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);

-- Index for "who follows me" queries
CREATE INDEX idx_followee ON follows (followee_id);

CREATE TABLE posts (
    post_id      BIGINT PRIMARY KEY,  -- Snowflake ID
    user_id      BIGINT NOT NULL,
    content      TEXT,
    media_urls   JSONB,
    like_count   INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    created_at   TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_posts ON posts (user_id, created_at DESC);
```

```
  ═══════════════════════════════════════════════════════
  Redis — Pre-computed Feed (Fanout on Write)
  ═══════════════════════════════════════════════════════

  Each user has a Redis sorted set as their feed:
  
  Key: feed:{user_id}
  Score: post timestamp
  Value: post_id

  ZADD feed:42 1717200000 "post:999"  ← Add post to user 42's feed
  ZADD feed:42 1717200060 "post:1000"
  ZADD feed:42 1717200120 "post:1001"

  // Get feed (newest first, paginated):
  ZREVRANGE feed:42 0 19  ← Get top 20 posts
  
  // Keep feed bounded (only latest 1000 posts):
  ZREMRANGEBYRANK feed:42 0 -1001  ← Remove oldest beyond 1000
```

### Architecture

```
                   NEWS FEED ARCHITECTURE
                   ═════════════════════

    POST CREATION (Write Path):
    
    ┌──────┐     ┌──────────┐     ┌──────────┐
    │ User │────►│ Post     │────►│PostgreSQL│ (store post)
    │      │     │ Service  │     └──────────┘
    └──────┘     └────┬─────┘
                      │
                      ▼
                ┌──────────┐
                │  Kafka   │ (PostCreated event)
                └────┬─────┘
                     │
              ┌──────┴──────┐
              │             │
              ▼             ▼
    ┌──────────────┐  ┌──────────────┐
    │ Fanout       │  │ Celebrity    │
    │ Service      │  │ Detector     │
    │              │  │              │
    │ IF regular   │  │ IF celebrity │
    │ user (<10K): │  │ (>10K):     │
    │              │  │              │
    │ Get followers│  │ DON'T fanout │
    │ → ZADD post  │  │ → Just store │
    │   to each    │  │   the post   │
    │   follower's │  │              │
    │   Redis feed │  │              │
    └──────────────┘  └──────────────┘

    FEED READING (Read Path):

    ┌──────┐     ┌──────────┐
    │ User │────►│ Feed     │
    │      │     │ Service  │
    └──────┘     └────┬─────┘
                      │
            ┌─────────┼─────────────┐
            │         │             │
            ▼         ▼             ▼
    ┌────────────┐ ┌──────────┐ ┌──────────────┐
    │   Redis    │ │PostgreSQL│ │   Merge &    │
    │            │ │          │ │   Rank       │
    │ Pre-built  │ │ Celebrity│ │              │
    │ feed       │ │ posts    │ │ Combine both │
    │ (regular   │ │ (query   │ │ → Sort by    │
    │  users'    │ │  on read)│ │   time       │
    │  posts)    │ │          │ │ → Return     │
    └────────────┘ └──────────┘ │   top 20     │
                                └──────────────┘
```

### Key Design Decisions

```
┌──────────────────────────────────────────────────────────────────┐
│  DECISION: Fanout Threshold                                      │
│                                                                   │
│  < 10K followers → Fanout on write (push to Redis feeds)        │
│  > 10K followers → Fanout on read (merge at read time)          │
│                                                                   │
│  Math: Taylor Swift posts 5 times/day, 100M followers            │
│  Fanout on write: 5 × 100M = 500M Redis writes/day → 💀         │
│  Fanout on read: Merge her posts at read time → manageable      │
│                                                                   │
│  Typical user follows ~5 celebrities                             │
│  Feed read = Redis feed + 5 celebrity post lookups = fast! ✅    │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  DECISION: Feed Storage (Redis Sorted Set)                       │
│                                                                   │
│  Memory calculation:                                              │
│  500M users × 1000 posts/feed × 16 bytes/entry = 8 TB           │
│  Redis cluster with 8 TB → expensive but feasible               │
│  Alternative: Only cache active users' feeds (last 7 days)      │
│  Inactive users → rebuild feed from DB on next login             │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  DECISION: Like/Comment Counts                                   │
│                                                                   │
│  Problem: Millions of likes on a viral post                      │
│  → UPDATE posts SET like_count = like_count + 1 WHERE id = X    │
│  → Hot row! Lock contention! 💀                                  │
│                                                                   │
│  Solution: Redis counter + async sync to DB                      │
│  INCR post:999:likes  ← Redis atomic increment (fast!)          │
│  Every 30 seconds: sync count to PostgreSQL (batch update)      │
│                                                                   │
│  Display shows Redis count (real-time).                          │
│  DB has slightly delayed count (for analytics/persistence).      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🚦 Design 4: Rate Limiter

### Requirements

```
  Functional:
  • Limit API requests per user/IP: e.g., 100 requests per minute
  • Return 429 Too Many Requests when exceeded
  • Support multiple rate limit rules (per endpoint, per user, global)

  Non-Functional:
  • Must be FAST (< 1ms overhead per request)
  • Must be ACCURATE (no over-counting or under-counting)
  • Must work in DISTRIBUTED systems (multiple API servers)
  • Must be ATOMIC (no race conditions between servers)
```

### Database Solutions

```
  ═══════════════════════════════════════════════════════
  ALGORITHM 1: FIXED WINDOW COUNTER
  ═══════════════════════════════════════════════════════

  Track count per time window (e.g., per minute).

  Redis:
  Key:  ratelimit:{user_id}:{window}
        ratelimit:user42:2025-06-01T10:30

  INCR  ratelimit:user42:2025-06-01T10:30
  EXPIRE ratelimit:user42:2025-06-01T10:30 60

  If count > 100 → REJECT ❌
  
  ⚠️ PROBLEM: Boundary issue
  Minute 1 (10:30-10:31): 99 requests at 10:30:59
  Minute 2 (10:31-10:32): 99 requests at 10:31:01
  → 198 requests in 2 seconds! But both windows pass ❌

  ═══════════════════════════════════════════════════════
  ALGORITHM 2: SLIDING WINDOW LOG
  ═══════════════════════════════════════════════════════

  Store timestamp of each request. Count requests in last 60s.

  Redis Sorted Set:
  ZADD ratelimit:user42 1717200000.123 "req-uuid-1"
  ZADD ratelimit:user42 1717200001.456 "req-uuid-2"

  // Remove entries older than 60 seconds
  ZREMRANGEBYSCORE ratelimit:user42 0 (now - 60)
  
  // Count entries in window
  ZCARD ratelimit:user42
  
  If count > 100 → REJECT ❌

  ✅ Accurate! No boundary issues
  ❌ Memory heavy (stores every request timestamp)

  ═══════════════════════════════════════════════════════
  ALGORITHM 3: SLIDING WINDOW COUNTER (Best Balance)
  ═══════════════════════════════════════════════════════

  Weighted average of current and previous window.

  Previous minute (10:30): 80 requests
  Current minute (10:31): 30 requests so far
  Time into current minute: 20 seconds (33%)

  Estimated count = 80 × (1 - 0.33) + 30 = 53.6 + 30 = 83.6

  If estimated > 100 → REJECT

  ✅ Accurate enough
  ✅ Low memory (just two counters)
  ✅ Used by: Cloudflare, Kong, AWS API Gateway
```

```python
# ═══════════════════════════════════════════════════════
# Redis Rate Limiter — Sliding Window Counter
# ═══════════════════════════════════════════════════════

import redis
import time

r = redis.Redis()

def is_rate_limited(user_id: str, limit: int = 100, window: int = 60) -> bool:
    """Check if user exceeds rate limit using sliding window counter."""
    now = time.time()
    current_window = int(now // window)
    previous_window = current_window - 1
    position_in_window = (now % window) / window  # 0.0 to 1.0
    
    current_key = f"rate:{user_id}:{current_window}"
    previous_key = f"rate:{user_id}:{previous_window}"
    
    # Atomic pipeline
    pipe = r.pipeline()
    pipe.get(previous_key)
    pipe.incr(current_key)
    pipe.expire(current_key, window * 2)
    results = pipe.execute()
    
    previous_count = int(results[0] or 0)
    current_count = int(results[1])
    
    # Weighted estimate
    estimated = previous_count * (1 - position_in_window) + current_count
    
    return estimated > limit

# Usage
if is_rate_limited("user:42"):
    return HttpResponse(status=429, body="Too Many Requests")
```

---

## 🛒 Design 5: E-Commerce Inventory System

### The Hardest Problem: Overselling

```
  PROBLEM: 
  Concert ticket: only 1 seat left.
  100 users click "Buy" at the same time.
  
  WITHOUT proper design:
  
  Thread 1: SELECT quantity FROM inventory WHERE item = 'seat-A1' → 1
  Thread 2: SELECT quantity FROM inventory WHERE item = 'seat-A1' → 1
  Thread 3: SELECT quantity FROM inventory WHERE item = 'seat-A1' → 1
  ...
  Thread 100: SELECT quantity FROM inventory WHERE item = 'seat-A1' → 1
  
  ALL 100 threads see quantity = 1 → ALL proceed to purchase!
  
  Thread 1: UPDATE inventory SET quantity = 0 WHERE item = 'seat-A1' ✅
  Thread 2: UPDATE inventory SET quantity = -1 WHERE item = 'seat-A1' ❌
  ...
  Result: 100 people "bought" 1 ticket = OVERSOLD! 💀
```

### Solutions

```sql
-- ═══════════════════════════════════════════════════════
-- SOLUTION 1: Pessimistic Locking (SELECT FOR UPDATE)
-- ═══════════════════════════════════════════════════════

BEGIN;

-- Lock the row — other transactions WAIT
SELECT quantity FROM inventory 
WHERE item_id = 'seat-A1' 
FOR UPDATE;

-- Check if available
-- If quantity > 0:
UPDATE inventory 
SET quantity = quantity - 1 
WHERE item_id = 'seat-A1' AND quantity > 0;

-- If no rows updated → out of stock
COMMIT;

-- ✅ Strong consistency, no overselling
-- ❌ Locks = contention under high load
-- ❌ Deadlock risk with multiple items

-- ═══════════════════════════════════════════════════════
-- SOLUTION 2: Optimistic Locking (Version check)
-- ═══════════════════════════════════════════════════════

-- Read current state
SELECT quantity, version FROM inventory WHERE item_id = 'seat-A1';
-- Returns: quantity=1, version=5

-- Update ONLY if version hasn't changed
UPDATE inventory 
SET quantity = quantity - 1, version = version + 1
WHERE item_id = 'seat-A1' 
  AND version = 5 
  AND quantity > 0;

-- If 0 rows updated → someone else got there first → RETRY
-- If 1 row updated → SUCCESS ✅

-- ✅ No locks, high throughput
-- ❌ Retries under high contention
-- ❌ Fairness issues (lucky threads always win)

-- ═══════════════════════════════════════════════════════
-- SOLUTION 3: Atomic Decrement (Best for High Throughput)
-- ═══════════════════════════════════════════════════════

-- Single atomic statement — no read-then-write race!
UPDATE inventory 
SET quantity = quantity - 1 
WHERE item_id = 'seat-A1' AND quantity > 0
RETURNING quantity;  -- PostgreSQL: returns new quantity

-- If returns a row → SUCCESS (you got the ticket!)
-- If returns 0 rows → OUT OF STOCK

-- ✅ Atomic, no race condition
-- ✅ No version tracking needed
-- ❌ Still a hot row under extreme concurrency
```

```python
# ═══════════════════════════════════════════════════════
# SOLUTION 4: Redis Pre-Decrement + Async DB Update
# ═══════════════════════════════════════════════════════

# For FLASH SALES with extreme concurrency (100K+ requests/sec)

import redis
r = redis.Redis()

def try_purchase(item_id: str, user_id: str) -> bool:
    """
    Step 1: Decrement in Redis (atomic, fast)
    Step 2: If successful, queue DB update async
    """
    key = f"inventory:{item_id}"
    
    # Atomic decrement — Redis handles concurrency!
    remaining = r.decr(key)
    
    if remaining >= 0:
        # Success! Queue the actual order creation
        r.lpush("order_queue", json.dumps({
            "item_id": item_id,
            "user_id": user_id,
            "timestamp": time.time()
        }))
        return True
    else:
        # Oversold in Redis — restore
        r.incr(key)
        return False

# Background worker processes order_queue → 
# Creates orders in PostgreSQL
# If DB write fails → Redis INCR to restore inventory

# Setup: Before flash sale, sync inventory to Redis
# r.set("inventory:seat-A1", 100)  ← 100 tickets available
```

### E-Commerce Architecture

```
                E-COMMERCE INVENTORY ARCHITECTURE
                ═════════════════════════════════

    ┌───────────────────────────────────────────────────────────┐
    │                                                           │
    │  ┌──────────┐                                             │
    │  │  Client  │                                             │
    │  └────┬─────┘                                             │
    │       │                                                   │
    │       ▼                                                   │
    │  ┌──────────┐    ┌──────────┐                             │
    │  │ API      │───►│ Redis    │ ← Inventory counter (fast)  │
    │  │ Gateway  │    │ (DECR)   │   + Rate limiting           │
    │  └──────────┘    └────┬─────┘                             │
    │                       │ Success?                           │
    │                       ▼                                    │
    │                 ┌──────────┐                               │
    │                 │  Kafka   │ ← OrderRequested event       │
    │                 └────┬─────┘                               │
    │                      │                                     │
    │       ┌──────────────┼──────────────┐                     │
    │       │              │              │                      │
    │       ▼              ▼              ▼                      │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
    │  │ Order    │  │ Payment  │  │ Inventory│                │
    │  │ Service  │  │ Service  │  │ Service  │                │
    │  │          │  │          │  │          │                │
    │  │PostgreSQL│  │PostgreSQL│  │PostgreSQL│                │
    │  │(orders)  │  │(payments)│  │(stock)   │                │
    │  └──────────┘  └──────────┘  └──────────┘                │
    │                                                           │
    │  Saga Orchestrator manages:                               │
    │  Reserve Inventory → Charge Payment → Confirm Order       │
    │  If any fails → Compensating transactions                 │
    │                                                           │
    └───────────────────────────────────────────────────────────┘
```

---

## 🧰 Design Pattern Quick Reference

```
┌──────────────────────────────────────────────────────────────────┐
│           SYSTEM DESIGN — DATABASE PATTERN MATRIX                │
├──────────────────────┬───────────────────────────────────────────┤
│  System              │  Key DB Patterns                          │
├──────────────────────┼───────────────────────────────────────────┤
│  URL Shortener       │  Key-Value store, CDN cache, Base62 IDs  │
│                      │  Partition analytics by time              │
│                      │                                           │
│  Chat System         │  Cassandra (messages), Redis (presence), │
│                      │  PostgreSQL (metadata), Kafka (delivery)  │
│                      │  Time-bucketed partitions                 │
│                      │                                           │
│  News Feed           │  Hybrid fanout (push + pull),             │
│                      │  Redis Sorted Sets, PostgreSQL (posts),   │
│                      │  Async counters                           │
│                      │                                           │
│  Rate Limiter        │  Redis sliding window counter,            │
│                      │  Atomic INCR/DECR                        │
│                      │                                           │
│  E-Commerce          │  Saga pattern, Redis pre-decrement,      │
│                      │  Optimistic locking, Outbox pattern       │
│                      │                                           │
│  Search Engine       │  Elasticsearch (inverted index),          │
│                      │  CDC from primary DB → ES                │
│                      │                                           │
│  Notification System │  Kafka (fan-out), Cassandra (history),   │
│                      │  Redis (dedup), Priority queues           │
│                      │                                           │
│  File Storage        │  Object store (S3) + metadata DB,        │
│                      │  Consistent hashing for distribution      │
│                      │                                           │
│  Leaderboard         │  Redis Sorted Set (ZADD, ZRANK),         │
│                      │  Periodic snapshot to persistent DB       │
│                      │                                           │
│  Booking System      │  Pessimistic locking (SELECT FOR UPDATE),│
│  (Hotels/Flights)    │  Calendar table, Idempotency keys        │
│                      │                                           │
│  Social Graph        │  Neo4j or DGraph (graph DB),              │
│                      │  Adjacency list in SQL for simple cases  │
│                      │                                           │
│  Metrics/Monitoring  │  InfluxDB/TimescaleDB (time-series),     │
│                      │  Downsampling, Retention policies         │
└──────────────────────┴───────────────────────────────────────────┘
```

---

## 🎯 Interview Tips — Database System Design

```
┌──────────────────────────────────────────────────────────────────┐
│              INTERVIEW CHEAT SHEET                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. ALWAYS start with requirements!                              │
│     → "What's the read/write ratio?"                             │
│     → "How much data? How fast does it grow?"                    │
│     → "What consistency level is needed?"                        │
│     → "What's the latency requirement?"                          │
│                                                                   │
│  2. JUSTIFY your database choice!                                │
│     → "I chose PostgreSQL because we need ACID for payments"    │
│     → "I chose Cassandra because we need high write throughput"  │
│     → "I chose Redis because sub-millisecond reads are needed"  │
│     → Never say "I always use X" — show you can evaluate        │
│                                                                   │
│  3. DISCUSS trade-offs explicitly!                               │
│     → "We sacrifice consistency for availability here because…" │
│     → "This adds complexity but solves the hot-key problem"     │
│     → Interviewers LOVE hearing trade-off reasoning             │
│                                                                   │
│  4. SCALE progressively!                                         │
│     → Start simple: single DB                                    │
│     → Add caching when reads are slow                            │
│     → Add replicas when read load is high                        │
│     → Shard when write load is high                              │
│     → DON'T over-engineer from the start                         │
│                                                                   │
│  5. KNOW the numbers!                                            │
│     ┌────────────────────────────────────────────┐               │
│     │ Redis read:         ~0.1 ms                │               │
│     │ SSD random read:    ~0.1 ms                │               │
│     │ DB indexed query:   ~1-10 ms               │               │
│     │ DB full scan:       ~100 ms - 10 s         │               │
│     │ Cross-DC round trip: ~50-150 ms            │               │
│     │ 1 billion = 10^9                           │               │
│     │ 1 TB = 10^12 bytes                         │               │
│     │ QPS for single PostgreSQL: ~10K-50K        │               │
│     │ QPS for Redis: ~100K-300K (single node)    │               │
│     └────────────────────────────────────────────┘               │
│                                                                   │
│  6. MENTION monitoring and observability!                        │
│     → "I'd monitor replication lag with pg_stat_replication"    │
│     → "Cache hit rate should be tracked with Grafana"           │
│     → "I'd set up alerts for shard imbalance"                   │
│     → Shows production experience!                               │
│                                                                   │
│  7. RED FLAGS (things interviewers notice):                      │
│     ❌ "Just use MongoDB for everything"                         │
│     ❌ Sharding on day one for a startup                         │
│     ❌ Ignoring failure scenarios                                │
│     ❌ Not considering read/write patterns                       │
│     ❌ No caching layer mentioned                                │
│     ❌ No index strategy discussed                               │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🗺️ Database Selection Decision Tree

```
          What Kind of Data Do You Have?
          ═══════════════════════════════

                ┌──────────────────┐
                │  What's your     │
                │  primary need?   │
                └────────┬─────────┘
                         │
         ┌───────────────┼───────────────────┐
         │               │                   │
    Structured      Semi-structured     Relationships
    (Tables)        (Documents)         (Graph)
         │               │                   │
         ▼               ▼                   ▼
    Need ACID?     Flexible schema?    Graph queries?
    JOINs?         Nested data?        Path finding?
         │               │                   │
         ▼               ▼                   ▼
    ┌────────┐     ┌──────────┐        ┌─────────┐
    │SQL DB  │     │Document  │        │Graph DB │
    │        │     │DB        │        │         │
    │Postgres│     │MongoDB   │        │Neo4j    │
    │MySQL   │     │DynamoDB  │        │DGraph   │
    │SQL Srv │     │CosmosDB  │        │         │
    └────────┘     └──────────┘        └─────────┘

    Need caching?      Time-series?      Full-text search?
         │                  │                   │
         ▼                  ▼                   ▼
    ┌────────┐     ┌──────────────┐     ┌──────────────┐
    │Redis   │     │TimescaleDB   │     │Elasticsearch │
    │Memcached     │InfluxDB      │     │Solr          │
    └────────┘     │Prometheus    │     └──────────────┘
                   └──────────────┘

    Need distributed SQL?    Need massive write scale?
         │                         │
         ▼                         ▼
    ┌──────────────┐        ┌──────────────┐
    │CockroachDB   │        │Cassandra     │
    │Google Spanner │        │ScyllaDB      │
    │YugabyteDB    │        │DynamoDB      │
    │TiDB          │        │              │
    └──────────────┘        └──────────────┘
```

---

## 🔗 Part 7 Complete!

Congratulations! You've mastered the architecture and system design of databases. You now have the knowledge to:

- **Shard** databases when single machines aren't enough (Chapter 7.1)
- **Replicate** data for high availability and read scaling (Chapter 7.2)
- Design databases for **microservices** with Sagas, CQRS, and Outbox (Chapter 7.3)
- Implement **caching** that handles thundering herds and hot keys (Chapter 7.4)
- **Design real systems** from the database perspective like a staff engineer (Chapter 7.5)

> **Next:** [Part 8 — Interview Preparation & Cheat Sheets →](../19-Interview-Prep/01-SQL-Interview-Questions.md)

---

> _"The best database architecture is the simplest one that meets your requirements today, with clear paths to evolve when requirements change."_
> — Every wise architect
