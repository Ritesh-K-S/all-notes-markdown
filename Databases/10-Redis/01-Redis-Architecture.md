# ⚡ Chapter 3C.1 — Redis Architecture & Data Structures

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 3A.1 (NoSQL Overview), Chapter 1.5 (ACID vs BASE)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why Redis is insanely fast** (it's not just "because it's in-memory")
- Know the **complete architecture** — single-threaded model, event loop, memory management
- Master **all 10+ data structures** Redis offers (most people know only 3)
- Write Redis commands with the **confidence of a senior engineer**
- Understand **when to use Redis** and when it's the **wrong choice**
- Know how Redis handles **persistence** without sacrificing speed

---

## 🧠 The Big Question

> *"Why does the world need a database that forgets everything when the power goes out?"*

Here's the thing — **it doesn't forget**. That's the #1 myth about Redis.

Redis started as a **pure in-memory store**, but modern Redis is a **full-fledged database** with persistence, replication, clustering, and even scripting.

But let's start with the real question:

### 🐌 The Speed Problem

```
Traditional Database (PostgreSQL, MySQL):

  User Request → Network → Query Parser → Optimizer → Disk I/O → Return
                                                         ↑
                                                    THIS IS SLOW
                                                    (~5-15ms per read)

  Why? Because data lives on DISK (SSD/HDD)
  Even SSDs: ~0.1ms per random read
  HDDs: ~5-10ms per random read

Redis:

  User Request → Network → Memory Lookup → Return
                              ↑
                         THIS IS FAST
                         (~0.1-0.5ms per read)

  Why? Because data lives in RAM
  RAM: ~100 NANOSECONDS per read (that's 0.0001ms)
```

### ⚡ Speed Comparison — Feel the Difference

```
╔══════════════════════════════════════════════════════════╗
║              OPERATIONS PER SECOND (OPS)                 ║
╠══════════════════════════════════════════════════════════╣
║  Redis          │ 100,000 - 1,000,000+ ops/sec  🚀🚀🚀  ║
║  MongoDB        │ 10,000 - 50,000 ops/sec       🚗       ║
║  PostgreSQL     │ 5,000 - 30,000 ops/sec        🚗       ║
║  MySQL          │ 5,000 - 25,000 ops/sec        🚗       ║
║  Cassandra      │ 20,000 - 100,000 ops/sec      🚀       ║
╠══════════════════════════════════════════════════════════╣
║  Redis is 10x-100x FASTER than disk-based databases     ║
╚══════════════════════════════════════════════════════════╝
```

> 💡 **Real Talk:** Google, Twitter, GitHub, StackOverflow, Instagram, Uber — they ALL use Redis. If you're building anything with more than 100 users, you'll probably need Redis.

---

## 🏗️ Redis Architecture — Why It's So Fast

### The 5 Pillars of Redis Speed

```
╔══════════════════════════════════════════════════════╗
║            WHY REDIS IS BLAZING FAST                  ║
╠══════════════════════════════════════════════════════╣
║                                                       ║
║  1️⃣  IN-MEMORY STORAGE                               ║
║     └─ Data lives in RAM, not disk                    ║
║                                                       ║
║  2️⃣  SINGLE-THREADED EVENT LOOP                      ║
║     └─ No locks, no context switches, no overhead     ║
║                                                       ║
║  3️⃣  I/O MULTIPLEXING (epoll/kqueue)                 ║
║     └─ Handles 10,000+ connections with 1 thread      ║
║                                                       ║
║  4️⃣  EFFICIENT DATA STRUCTURES                       ║
║     └─ Purpose-built C implementations                ║
║                                                       ║
║  5️⃣  SIMPLE PROTOCOL (RESP)                          ║
║     └─ Minimal parsing overhead                       ║
║                                                       ║
╚══════════════════════════════════════════════════════╝
```

---

### 1️⃣ In-Memory Storage

```
┌─────────────────────────────────────────────────────┐
│              TRADITIONAL DATABASE                    │
│                                                      │
│  Application → Query Engine → Buffer Pool → DISK     │
│                                    ↕                 │
│                              Page Cache              │
│                                                      │
│  Data might be in memory (cache hit) or on disk      │
│  (cache miss). You never know → UNPREDICTABLE        │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                    REDIS                             │
│                                                      │
│  Application → Redis Process → RAM (always)          │
│                                                      │
│  Data is ALWAYS in memory → PREDICTABLE latency      │
│  Sub-millisecond. Every. Single. Time.               │
└─────────────────────────────────────────────────────┘
```

> 💡 **Fun Fact:** 1 GB of RAM can store ~25-30 million simple key-value pairs. A server with 256 GB RAM? That's **billions** of entries.

---

### 2️⃣ Single-Threaded Model — The Counterintuitive Secret

> *"Wait... single-threaded? How can that be fast?"*

This is the part that confuses everyone. Let me explain:

```
MULTI-THREADED DATABASE (e.g., MySQL):
┌─────────────────────────────────────────┐
│  Thread 1: UPDATE users SET ...         │
│  Thread 2: SELECT * FROM users WHERE... │
│  Thread 3: INSERT INTO orders ...       │
│  Thread 4: DELETE FROM sessions ...     │
│                                          │
│  Problem: Threads fight over same data!  │
│  → Need LOCKS (mutex, semaphore)         │
│  → Lock acquisition = ~1-10 μs each     │
│  → Context switching = ~5-15 μs each    │
│  → Deadlocks = debugging nightmare 💀   │
└─────────────────────────────────────────┘

SINGLE-THREADED REDIS:
┌─────────────────────────────────────────┐
│  Event Loop (one thread):               │
│                                          │
│  ┌──→ Read request from Client A        │
│  │    Execute command (~1 μs)            │
│  │    Send response                      │
│  │                                       │
│  ├──→ Read request from Client B        │
│  │    Execute command (~1 μs)            │
│  │    Send response                      │
│  │                                       │
│  ├──→ Read request from Client C        │
│  │    Execute command (~1 μs)            │
│  │    Send response                      │
│  │                                       │
│  └──→ (repeat forever)                  │
│                                          │
│  ✅ No locks needed (only 1 thread)     │
│  ✅ No context switching                │
│  ✅ No race conditions                  │
│  ✅ No deadlocks                        │
│  ⚡ Each command: ~1 microsecond        │
│  ⚡ 1 million commands/sec is normal    │
└─────────────────────────────────────────┘
```

> 💡 **Analogy:** Think of a **single cashier** who is incredibly fast vs **10 cashiers** who keep bumping into each other, arguing over the cash register, and dropping change. The single fast cashier wins.

**But wait — modern Redis (6.0+) IS multi-threaded for I/O:**

```
Redis 6.0+ Architecture:
┌──────────────────────────────────────────────────┐
│                                                   │
│  I/O Threads (multiple):                         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐            │
│  │ Thread 1│ │ Thread 2│ │ Thread 3│             │
│  │ Read/   │ │ Read/   │ │ Read/   │             │
│  │ Write   │ │ Write   │ │ Write   │             │
│  │ Network │ │ Network │ │ Network │             │
│  └────┬────┘ └────┬────┘ └────┬────┘             │
│       │           │           │                   │
│       └───────────┼───────────┘                   │
│                   ▼                               │
│  ┌─────────────────────────────────┐             │
│  │     MAIN THREAD (single)        │             │
│  │     Executes ALL commands       │             │
│  │     Serialized, no locks        │             │
│  └─────────────────────────────────┘             │
│                                                   │
│  I/O is parallel, execution is serial ✅         │
└──────────────────────────────────────────────────┘
```

---

### 3️⃣ I/O Multiplexing — Handling 10,000+ Clients

```
Traditional Approach:
  1 client = 1 thread = 1 system resource
  10,000 clients = 10,000 threads = 💀 (memory + CPU explosion)

Redis uses epoll (Linux) / kqueue (macOS):
  1 thread monitors ALL client sockets simultaneously
  
  ┌─────────────────────────────────────────────┐
  │              Event Loop (epoll)              │
  │                                              │
  │  Socket Pool:                                │
  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐       │
  │  │Client│ │Client│ │Client│ │Client│  ...   │
  │  │  1   │ │  2   │ │  3   │ │ 9999 │       │
  │  └───┬──┘ └───┬──┘ └───┬──┘ └───┬──┘       │
  │      │        │        │        │            │
  │      └────────┴────────┴────────┘            │
  │                   │                           │
  │           ┌───────▼──────┐                   │
  │           │  epoll_wait  │                   │
  │           │  "Who has    │                   │
  │           │   data?"     │                   │
  │           └───────┬──────┘                   │
  │                   │                           │
  │           Client 2 & 9999 ready!             │
  │           → Process Client 2 command          │
  │           → Process Client 9999 command       │
  │           → Back to epoll_wait               │
  └─────────────────────────────────────────────┘
```

---

### 4️⃣ The RESP Protocol — Simple = Fast

```
RESP (REdis Serialization Protocol):

  Client sends:
  *3\r\n        ← Array of 3 elements
  $3\r\n        ← Bulk string of 3 bytes
  SET\r\n       ← "SET"
  $4\r\n        ← Bulk string of 4 bytes
  name\r\n      ← "name"
  $5\r\n        ← Bulk string of 5 bytes
  Redis\r\n     ← "Redis"

  Server responds:
  +OK\r\n       ← Simple string "OK"

  Compare to SQL:
  "INSERT INTO key_value (key, val) VALUES ('name', 'Redis')"
  → Parse SQL → Build AST → Optimize → Execute → Return
  → 100x more overhead!
```

---

## 📦 Redis In a Box — What It Looks Like

```
┌──────────────────────────────────────────────────────────┐
│                    REDIS SERVER                           │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                  MEMORY (RAM)                        │ │
│  │                                                      │ │
│  │  ┌──────────────────────────────────────────────┐   │ │
│  │  │              KEY SPACE                        │   │ │
│  │  │                                               │   │ │
│  │  │  db0: { "user:1" → Hash,                     │   │ │
│  │  │         "session:abc" → String,               │   │ │
│  │  │         "cart:42" → List,                     │   │ │
│  │  │         "online_users" → Set,                 │   │ │
│  │  │         "leaderboard" → Sorted Set }          │   │ │
│  │  │                                               │   │ │
│  │  │  db1: { ... }   (16 databases: db0-db15)      │   │ │
│  │  │  db2: { ... }                                 │   │ │
│  │  └──────────────────────────────────────────────┘   │ │
│  │                                                      │ │
│  │  ┌──────────────┐  ┌──────────────┐                 │ │
│  │  │ Pub/Sub      │  │ Lua Scripts  │                 │ │
│  │  │ Channels     │  │ Engine       │                 │ │
│  │  └──────────────┘  └──────────────┘                 │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌───────────────────────┐  ┌──────────────────────────┐ │
│  │  PERSISTENCE          │  │  REPLICATION              │ │
│  │  ┌─────┐ ┌─────┐     │  │  Master → Replica 1       │ │
│  │  │ RDB │ │ AOF │     │  │         → Replica 2       │ │
│  │  └─────┘ └─────┘     │  │         → Replica N       │ │
│  └───────────────────────┘  └──────────────────────────┘ │
│                                                           │
│  Port: 6379 (default)     Protocol: RESP                 │
└──────────────────────────────────────────────────────────┘
```

---

## 🔌 Getting Started — Installation & First Commands

### Installation

```bash
# 🐧 Linux (Ubuntu/Debian)
sudo apt update
sudo apt install redis-server
sudo systemctl start redis
sudo systemctl enable redis

# 🍎 macOS
brew install redis
brew services start redis

# 🪟 Windows (WSL2 recommended, or use Memurai/Redis Stack)
# Option 1: WSL2 (recommended)
wsl --install
# Then follow Linux instructions inside WSL

# Option 2: Docker (works everywhere)
docker run -d --name redis -p 6379:6379 redis:latest

# ☁️ Cloud (easiest for production)
# Redis Cloud: https://redis.com/try-free/
# AWS ElastiCache
# Azure Cache for Redis
# GCP Memorystore
```

### First Connection

```bash
# Connect to Redis
redis-cli

# Test connection
127.0.0.1:6379> PING
PONG    ← Redis is alive! 🎉

# Check server info
127.0.0.1:6379> INFO server
# redis_version:7.2.4
# redis_mode:standalone
# tcp_port:6379
# uptime_in_seconds:12345
```

---

## 🗃️ Redis Data Structures — The Complete Arsenal

> **This is where Redis becomes MAGICAL.** Most people think Redis is just `SET key value`. That's like saying Python is just `print("hello")`.

Redis has **10+ data structures**, each solving a different problem:

```
╔════════════════════════════════════════════════════════════════════╗
║                  REDIS DATA STRUCTURES MAP                         ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  ┌────────────┐  Simple values, counters, flags, JSON             ║
║  │  STRING     │  "user:1:name" → "Alice"                         ║
║  └────────────┘                                                    ║
║                                                                    ║
║  ┌────────────┐  Object/record storage (like a row)               ║
║  │  HASH       │  "user:1" → {name: "Alice", age: 30}            ║
║  └────────────┘                                                    ║
║                                                                    ║
║  ┌────────────┐  Ordered collection, queues, stacks               ║
║  │  LIST       │  "queue:emails" → ["msg1", "msg2", "msg3"]      ║
║  └────────────┘                                                    ║
║                                                                    ║
║  ┌────────────┐  Unique collection, tags, memberships             ║
║  │  SET        │  "user:1:roles" → {"admin", "editor"}           ║
║  └────────────┘                                                    ║
║                                                                    ║
║  ┌────────────┐  Ranked data, leaderboards, priority queues       ║
║  │  SORTED SET │  "leaderboard" → {Alice:950, Bob:870}           ║
║  │  (ZSET)     │                                                   ║
║  └────────────┘                                                    ║
║                                                                    ║
║  ┌────────────┐  Event streaming, message queues, logs            ║
║  │  STREAM     │  "events" → [{id: ..., data: ...}, ...]         ║
║  └────────────┘                                                    ║
║                                                                    ║
║  ┌────────────┐  Bit-level operations, feature flags              ║
║  │  BITMAP     │  "logins:2024-01" → 01101001...                  ║
║  └────────────┘                                                    ║
║                                                                    ║
║  ┌────────────┐  Approximate unique counting                      ║
║  │ HYPERLOGLOG│  "unique_visitors" → ~1,543,221                   ║
║  └────────────┘                                                    ║
║                                                                    ║
║  ┌────────────┐  Geospatial queries (nearby locations)            ║
║  │  GEO        │  "restaurants" → {lat, lng, name}                ║
║  └────────────┘                                                    ║
║                                                                    ║
║  ┌────────────┐  Probabilistic "is member?" checks                ║
║  │BLOOM FILTER│  "seen_urls" → maybe_yes / definitely_no         ║
║  │(Module)     │                                                   ║
║  └────────────┘                                                    ║
║                                                                    ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## 📝 1. STRING — The Swiss Army Knife

> **The most used data type.** But don't let the name fool you — it stores strings, numbers, JSON, binary data, even images (up to 512 MB).

### Basic Operations

```bash
# ═══════════════════════════════════════
#  SET / GET — The bread and butter
# ═══════════════════════════════════════

SET name "Redis Mastery"       # Store a string
GET name                        # → "Redis Mastery"

# SET with options
SET session:abc "user123" EX 3600      # Expires in 3600 seconds (1 hour)
SET session:def "user456" PX 60000     # Expires in 60000 milliseconds
SET lock:order:42 "worker-1" NX        # Set only if NOT exists (distributed lock!)
SET counter "100" XX                    # Set only if ALREADY exists (update only)

# ═══════════════════════════════════════
#  GETSET — Atomic get-and-replace
# ═══════════════════════════════════════
SET greeting "Hello"
GETSET greeting "Hi"           # Returns "Hello", sets to "Hi"
# (Redis 6.2+: use GETDEL or GETEX instead)

# ═══════════════════════════════════════
#  MSET / MGET — Batch operations (FAST!)
# ═══════════════════════════════════════
MSET user:1:name "Alice" user:1:email "alice@test.com" user:1:age "30"
MGET user:1:name user:1:email user:1:age
# → ["Alice", "alice@test.com", "30"]

# Why batch? 3 commands = 3 round trips. MSET = 1 round trip = 3x faster!
```

### Numeric Operations (Counters)

```bash
# ═══════════════════════════════════════
#  INCR / DECR — Atomic counters
# ═══════════════════════════════════════

SET page_views 0
INCR page_views                # → 1  (atomic increment by 1)
INCR page_views                # → 2
INCR page_views                # → 3
DECR page_views                # → 2

INCRBY page_views 100          # → 102  (increment by N)
DECRBY page_views 50           # → 52   (decrement by N)

INCRBYFLOAT price 9.99         # → "9.99"  (floating point increment)
INCRBYFLOAT price 0.01         # → "10"

# ⚡ WHY THIS MATTERS:
# INCR is ATOMIC — even with 1000 concurrent clients,
# you'll never lose a count. No locks needed!

# 🔥 Real Use Case: Rate Limiting
SET api:user:42:requests 0 EX 60    # Reset every 60 seconds
INCR api:user:42:requests           # Count each request
# If result > 100 → reject (rate limit exceeded)
```

### String Manipulation

```bash
# ═══════════════════════════════════════
#  String functions
# ═══════════════════════════════════════

SET msg "Hello"
APPEND msg " World"            # → 11 (length), value = "Hello World"
STRLEN msg                     # → 11

SETRANGE msg 6 "Redis"         # "Hello Redis" (overwrite starting at offset 6)
GETRANGE msg 0 4               # → "Hello" (substring from index 0 to 4)
```

### 🔥 Real-World STRING Use Cases

```
╔══════════════════════════════════════════════════════════════╗
║  USE CASE                    │  EXAMPLE                      ║
╠══════════════════════════════════════════════════════════════╣
║  Session Storage             │  SET sess:abc '{"user":1}'    ║
║  Page View Counter           │  INCR page:/home/views        ║
║  Rate Limiter                │  INCR rate:user:42            ║
║  Distributed Lock            │  SET lock:res1 "w1" NX EX 30 ║
║  JSON Document Cache         │  SET user:1 '{"name":"A"}'   ║
║  Feature Flags               │  SET feature:dark_mode "on"   ║
║  Short-lived OTP/Token       │  SET otp:9876 "42" EX 300    ║
║  API Response Cache          │  SET cache:/api/v1/... "..."  ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 🗂️ 2. HASH — The Mini Database Inside Redis

> Think of a Hash as a **single database row** or a **JavaScript object**. Perfect for storing objects with multiple fields.

```bash
# ═══════════════════════════════════════
#  HSET / HGET — Store objects
# ═══════════════════════════════════════

# Store a user profile (like a database row)
HSET user:1 name "Alice" age 30 email "alice@dev.com" role "admin"
#     ↑ key    ↑ field-value pairs

HGET user:1 name               # → "Alice"
HGET user:1 email              # → "alice@dev.com"

# Get ALL fields
HGETALL user:1
# → {name: "Alice", age: "30", email: "alice@dev.com", role: "admin"}

# Get multiple specific fields
HMGET user:1 name email
# → ["Alice", "alice@dev.com"]

# ═══════════════════════════════════════
#  Field operations
# ═══════════════════════════════════════

HEXISTS user:1 phone           # → 0 (doesn't exist)
HDEL user:1 role               # Remove the "role" field
HLEN user:1                    # → 3 (remaining fields)
HKEYS user:1                   # → ["name", "age", "email"]
HVALS user:1                   # → ["Alice", "30", "alice@dev.com"]

# ═══════════════════════════════════════
#  Numeric fields
# ═══════════════════════════════════════

HSET product:1 name "Laptop" price 999 stock 50
HINCRBY product:1 stock -1     # → 49  (sold one!)
HINCRBY product:1 stock -1     # → 48
HINCRBYFLOAT product:1 price 50.50  # → "1049.5" (price increase)
```

### Hash vs Multiple Strings — Why Hash Wins

```
❌ BAD: Storing user data as separate strings
   SET user:1:name "Alice"     ← 3 keys in Redis
   SET user:1:age "30"         ← More memory overhead
   SET user:1:email "alice@"   ← 3 GET calls to fetch user

✅ GOOD: Storing as a Hash
   HSET user:1 name "Alice" age 30 email "alice@"
   ← 1 key in Redis
   ← Less memory (internal ziplist encoding for small hashes)
   ← 1 HGETALL call to fetch entire user

Memory comparison (1 million users):
   Strings: ~120 MB
   Hash:    ~70 MB    ← 40% less memory!
```

> 💡 **Pro Tip:** Redis internally uses **ziplist** encoding for small hashes (≤128 fields, ≤64 bytes per value). This is extremely memory-efficient. When a hash grows beyond these thresholds, it switches to a **hashtable** encoding.

### 🔥 Real-World HASH Use Cases

```
╔══════════════════════════════════════════════════════════════════╗
║  USE CASE              │  KEY                │  FIELDS           ║
╠══════════════════════════════════════════════════════════════════╣
║  User Profile Cache    │  user:1001          │  name, email, ... ║
║  Shopping Cart         │  cart:user:42       │  item1:qty, ...   ║
║  Session Data          │  session:abc123     │  userId, role, ip ║
║  Product Catalog       │  product:SKU-001    │  name, price, qty ║
║  Config / Settings     │  config:app         │  theme, lang, ... ║
║  Real-time Game State  │  game:room:7        │  p1_score, ...    ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 📋 3. LIST — Ordered, Repeatable Collection

> A doubly-linked list. Think of it as an **array** where you can push/pop from both ends in O(1). Perfect for **queues**, **stacks**, and **activity feeds**.

```bash
# ═══════════════════════════════════════
#  LPUSH / RPUSH — Add to left or right
# ═══════════════════════════════════════

RPUSH queue:emails "email1"    # Push to RIGHT (tail)
RPUSH queue:emails "email2"
RPUSH queue:emails "email3"
# queue:emails → ["email1", "email2", "email3"]

LPUSH queue:emails "email0"    # Push to LEFT (head)
# queue:emails → ["email0", "email1", "email2", "email3"]

# ═══════════════════════════════════════
#  LPOP / RPOP — Remove from left or right
# ═══════════════════════════════════════

LPOP queue:emails              # → "email0" (removes from head)
RPOP queue:emails              # → "email3" (removes from tail)
# queue:emails → ["email1", "email2"]

# ═══════════════════════════════════════
#  Accessing elements
# ═══════════════════════════════════════

RPUSH tasks "task1" "task2" "task3" "task4" "task5"

LINDEX tasks 0                 # → "task1" (first element)
LINDEX tasks -1                # → "task5" (last element)
LRANGE tasks 0 2               # → ["task1", "task2", "task3"] (range)
LRANGE tasks 0 -1              # → ALL elements
LLEN tasks                     # → 5 (length)

# ═══════════════════════════════════════
#  Queue Pattern (FIFO)
# ═══════════════════════════════════════

# Producer:
RPUSH job_queue '{"type":"resize","img":"photo.jpg"}'
RPUSH job_queue '{"type":"email","to":"alice@test.com"}'

# Consumer:
LPOP job_queue                 # Process first job (FIFO)

# ═══════════════════════════════════════
#  Stack Pattern (LIFO)
# ═══════════════════════════════════════

# Push to stack:
LPUSH undo_stack "action1"
LPUSH undo_stack "action2"
LPUSH undo_stack "action3"

# Pop from stack (last in, first out):
LPOP undo_stack                # → "action3" (most recent)

# ═══════════════════════════════════════
#  Blocking Pop — The Smart Consumer
# ═══════════════════════════════════════

# BLPOP blocks until data arrives (perfect for worker queues!)
BLPOP job_queue 30             # Wait up to 30 seconds for a job
# Returns immediately if queue has data
# Blocks (waits) if queue is empty
# Returns nil after 30 seconds if still empty
```

### Trimming Lists (Fixed-Size Lists)

```bash
# Keep only the last 100 notifications
RPUSH notifications:user:1 "You got a like!"
LTRIM notifications:user:1 -100 -1    # Keep only last 100

# 🔥 Activity Feed Pattern:
LPUSH feed:user:42 '{"action":"post","id":123}'
LTRIM feed:user:42 0 99               # Keep latest 100 items
LRANGE feed:user:42 0 9               # Get latest 10 for display
```

---

## 🎯 4. SET — Unique, Unordered Collection

> A collection of **unique strings** (no duplicates). Think of it as a **mathematical set**. Supports union, intersection, and difference operations.

```bash
# ═══════════════════════════════════════
#  SADD / SMEMBERS — Add & list
# ═══════════════════════════════════════

SADD skills:alice "Python" "Redis" "Docker" "Python"  # "Python" added once!
SMEMBERS skills:alice          # → {"Python", "Redis", "Docker"}
SCARD skills:alice             # → 3 (cardinality = count)

SADD skills:bob "Java" "Redis" "Kubernetes" "Docker"

# ═══════════════════════════════════════
#  Membership check
# ═══════════════════════════════════════

SISMEMBER skills:alice "Python"     # → 1 (true)
SISMEMBER skills:alice "Go"         # → 0 (false)
# O(1) lookup! Ultra fast for "does this exist?" checks

# ═══════════════════════════════════════
#  Set Operations — THIS IS POWERFUL 🔥
# ═══════════════════════════════════════

# Skills both Alice AND Bob have:
SINTER skills:alice skills:bob
# → {"Redis", "Docker"}

# All skills combined:
SUNION skills:alice skills:bob
# → {"Python", "Redis", "Docker", "Java", "Kubernetes"}

# Skills Alice has but Bob doesn't:
SDIFF skills:alice skills:bob
# → {"Python"}

# Store result in a new set:
SINTERSTORE common_skills skills:alice skills:bob
# "common_skills" → {"Redis", "Docker"}

# ═══════════════════════════════════════
#  Random members
# ═══════════════════════════════════════

SADD lottery:pool "Alice" "Bob" "Charlie" "Diana" "Eve"
SRANDMEMBER lottery:pool 2     # → 2 random winners (non-destructive)
SPOP lottery:pool              # → Remove and return 1 random member
```

### 🔥 Real-World SET Use Cases

```bash
# Tag System
SADD post:42:tags "redis" "database" "tutorial"
SADD post:43:tags "redis" "caching" "performance"
SINTER post:42:tags post:43:tags    # → {"redis"} (common tags)

# Online Users Tracker
SADD online_users "user:1" "user:2" "user:3"
SREM online_users "user:2"          # User 2 went offline
SCARD online_users                  # → 2 online users

# Unique Visitors (per day)
SADD visitors:2024-01-15 "ip:1.2.3.4" "ip:5.6.7.8"
SCARD visitors:2024-01-15           # → 2 unique visitors

# Mutual Friends
SADD friends:alice "bob" "charlie" "diana"
SADD friends:bob "alice" "charlie" "eve"
SINTER friends:alice friends:bob    # → {"charlie"} (mutual friend!)

# Spam/Block List
SADD blocked_ips "1.2.3.4" "5.6.7.8"
SISMEMBER blocked_ips "1.2.3.4"    # → 1 (blocked!)  O(1)
```

---

## 🏆 5. SORTED SET (ZSET) — The Leaderboard King

> Like a SET, but every member has a **score**. Members are automatically sorted by score. This is Redis's **most powerful** data structure.

```bash
# ═══════════════════════════════════════
#  ZADD — Add with scores
# ═══════════════════════════════════════

ZADD leaderboard 950 "Alice" 870 "Bob" 920 "Charlie" 890 "Diana"
# Automatically sorted by score!

# ═══════════════════════════════════════
#  Rankings (sorted queries)
# ═══════════════════════════════════════

# Top 3 players (highest scores first):
ZREVRANGE leaderboard 0 2 WITHSCORES
# → [("Alice", 950), ("Charlie", 920), ("Diana", 890)]

# Bottom 3 players (lowest scores first):
ZRANGE leaderboard 0 2 WITHSCORES
# → [("Bob", 870), ("Diana", 890), ("Charlie", 920)]

# What's Alice's rank? (0-indexed)
ZREVRANK leaderboard "Alice"   # → 0 (she's #1!)
ZREVRANK leaderboard "Bob"     # → 3 (he's #4)

# What's Alice's score?
ZSCORE leaderboard "Alice"     # → "950"

# ═══════════════════════════════════════
#  Score updates
# ═══════════════════════════════════════

ZINCRBY leaderboard 30 "Bob"   # Bob scored 30 more → 900
# Bob is now ranked higher automatically!

# ═══════════════════════════════════════
#  Range queries by score
# ═══════════════════════════════════════

# Players with score between 890 and 950:
ZRANGEBYSCORE leaderboard 890 950 WITHSCORES
# → [("Diana", 890), ("Bob", 900), ("Charlie", 920), ("Alice", 950)]

# How many players have score > 900?
ZCOUNT leaderboard 900 +inf    # → 3

# Remove players with score < 800:
ZREMRANGEBYSCORE leaderboard -inf 799
```

### Sorted Set Internal Magic

```
How does Redis keep it sorted AND provide O(1) rank lookups?

Answer: SKIP LIST + HASH TABLE (dual data structure!)

Skip List (for sorted order):
Level 4: ────────────────────────────────────── 950
Level 3: ──────────── 890 ─────────────── 950
Level 2: ──── 870 ── 890 ── 920 ──────── 950
Level 1: 870 ── 890 ── 900 ── 920 ── 950
          Bob   Diana  Bob    Charlie  Alice

Hash Table (for O(1) score lookup):
  "Alice"   → 950
  "Bob"     → 900
  "Charlie" → 920
  "Diana"   → 890

Search for Alice's rank:
  Skip list traversal → O(log n) → Find position = #1

This is why ZSCORE is O(1) and ZRANK is O(log n)!
```

### 🔥 Real-World SORTED SET Use Cases

```bash
# 🎮 Gaming Leaderboard
ZADD game:leaderboard 15000 "player:42"
ZREVRANGE game:leaderboard 0 9     # Top 10 players

# 📰 Trending Articles (by view count)
ZINCRBY trending:today 1 "article:123"  # Someone viewed article
ZREVRANGE trending:today 0 4           # Top 5 trending

# ⏰ Delayed Job Queue (score = timestamp)
ZADD delayed_jobs 1704067200 '{"task":"send_email","to":"alice"}'
# Process all jobs where score ≤ current_time:
ZRANGEBYSCORE delayed_jobs -inf <current_timestamp>

# 🏷️ Priority Queue
ZADD priority_queue 1 "critical_bug" 5 "feature_request" 3 "ui_fix"
ZPOPMIN priority_queue        # → "critical_bug" (lowest = highest priority)

# 📊 Time-Series Data (sliding window)
ZADD api_requests:<user_id> <timestamp> <request_id>
ZREMRANGEBYSCORE api_requests:<user_id> -inf <timestamp-60>  # Remove old
ZCARD api_requests:<user_id>  # Count requests in last 60 seconds
```

---

## 🌊 6. STREAM — The Event Streaming Powerhouse

> Redis Streams = **Kafka-like event streaming** built into Redis. Append-only log with consumer groups. Added in Redis 5.0.

```bash
# ═══════════════════════════════════════
#  XADD — Append events to stream
# ═══════════════════════════════════════

XADD orders * product "laptop" price 999 customer "alice"
# → "1704067200000-0"  (* = auto-generate ID based on timestamp)

XADD orders * product "phone" price 699 customer "bob"
# → "1704067200001-0"

XADD orders * product "tablet" price 499 customer "charlie"
# → "1704067200002-0"

# ═══════════════════════════════════════
#  XRANGE / XREAD — Read events
# ═══════════════════════════════════════

# Read all events:
XRANGE orders - +
# → All entries from beginning (-) to end (+)

# Read last 2 events:
XREVRANGE orders + - COUNT 2

# Read new events (blocking — like BLPOP for streams):
XREAD BLOCK 5000 STREAMS orders $
# Waits for new events, returns after 5 seconds or when data arrives

# ═══════════════════════════════════════
#  Consumer Groups — Distributed Processing
# ═══════════════════════════════════════

# Create a consumer group:
XGROUP CREATE orders order_processors $ MKSTREAM

# Consumer 1 reads:
XREADGROUP GROUP order_processors consumer1 COUNT 1 STREAMS orders >
# → Gets 1 unprocessed message

# Consumer 2 reads:
XREADGROUP GROUP order_processors consumer2 COUNT 1 STREAMS orders >
# → Gets the NEXT unprocessed message (different from consumer1!)

# Acknowledge processed message:
XACK orders order_processors "1704067200000-0"
```

```
Stream Architecture:

  Producers                     Stream                     Consumers
  ┌──────┐                 ┌─────────────┐              ┌──────────┐
  │App A │──XADD──────────▶│  orders     │              │Consumer 1│
  └──────┘                 │             │   XREADGROUP │(worker)  │
  ┌──────┐                 │ [msg1]      │◄─────────────│          │
  │App B │──XADD──────────▶│ [msg2]      │              └──────────┘
  └──────┘                 │ [msg3]      │              ┌──────────┐
  ┌──────┐                 │ [msg4]      │   XREADGROUP │Consumer 2│
  │App C │──XADD──────────▶│ [msg5]      │◄─────────────│(worker)  │
  └──────┘                 │ [...]       │              └──────────┘
                           └─────────────┘
                           
  Each consumer gets DIFFERENT messages (load balancing!)
  If a consumer crashes, pending messages can be reclaimed.
```

> 💡 **Redis Streams vs Kafka:** Redis Streams is simpler, faster to set up, and great for moderate throughput. Kafka wins for massive-scale (millions of events/sec) and long-term storage.

---

## 🖼️ 7. BITMAP — Bit-Level Operations

> Store and manipulate individual **bits**. Perfect for boolean flags at massive scale. 1 billion flags = only 125 MB!

```bash
# ═══════════════════════════════════════
#  Track daily user logins
# ═══════════════════════════════════════

# User 42 logged in today:
SETBIT logins:2024-01-15 42 1      # Set bit at position 42 to 1

# Did user 42 log in?
GETBIT logins:2024-01-15 42        # → 1 (yes!)

# How many users logged in today?
BITCOUNT logins:2024-01-15         # Count all 1-bits

# ═══════════════════════════════════════
#  Bit operations across days
# ═══════════════════════════════════════

# Users who logged in on BOTH Monday AND Tuesday:
BITOP AND active_both logins:2024-01-15 logins:2024-01-16
BITCOUNT active_both               # → Number of users active both days

# Users who logged in on Monday OR Tuesday:
BITOP OR active_either logins:2024-01-15 logins:2024-01-16

# 🔥 Memory efficiency:
# 10 million users, daily login tracking:
# String approach: 10M keys × ~50 bytes = ~500 MB
# Bitmap approach: 10M bits = ~1.25 MB   ← 400x less memory!
```

---

## 📊 8. HYPERLOGLOG — Count Unique Items (Approximately)

> Count unique elements with **0.81% standard error** using only **12 KB of memory** — regardless of whether you're counting 10 or 10 billion items.

```bash
# ═══════════════════════════════════════
#  Count unique visitors
# ═══════════════════════════════════════

PFADD visitors:today "user:1" "user:2" "user:3"
PFADD visitors:today "user:1" "user:4"     # user:1 already counted!

PFCOUNT visitors:today                      # → 4 (approximately)

# Merge multiple days:
PFADD visitors:monday "user:1" "user:2"
PFADD visitors:tuesday "user:2" "user:3"
PFMERGE visitors:week visitors:monday visitors:tuesday
PFCOUNT visitors:week                       # → 3 (unique across both days)

# 🔥 Why HyperLogLog is magical:
# SET approach: Store all 100M user IDs → needs ~1.6 GB
# HyperLogLog: Approximate count → needs only 12 KB!
# Trade-off: ~0.81% error (100M actual → reports ~99.2M to ~100.8M)
```

---

## 🌍 9. GEO — Geospatial Queries

> Store latitude/longitude coordinates and find nearby locations. Built on top of Sorted Sets.

```bash
# ═══════════════════════════════════════
#  Add locations
# ═══════════════════════════════════════

GEOADD restaurants -73.935242 40.730610 "Joe's Pizza"
GEOADD restaurants -73.985130 40.758896 "Times Square Deli"
GEOADD restaurants -73.968285 40.785091 "Central Park Cafe"

# ═══════════════════════════════════════
#  Find nearby restaurants
# ═══════════════════════════════════════

# Within 5 km of a location, sorted by distance:
GEOSEARCH restaurants FROMLONLAT -73.950000 40.750000 BYRADIUS 5 km ASC WITHCOORD WITHDIST
# → "Joe's Pizza" (2.3 km), "Times Square Deli" (3.1 km), ...

# Distance between two restaurants:
GEODIST restaurants "Joe's Pizza" "Times Square Deli" km
# → "4.8"  (4.8 km apart)

# Get coordinates:
GEOPOS restaurants "Joe's Pizza"
# → [(-73.935242, 40.730610)]

# 🔥 Real-World Use Cases:
# - "Find nearest Uber drivers"
# - "Show restaurants within 2 miles"
# - "Alert users when friends are nearby"
```

---

## ⏰ Key Expiry & TTL — Redis's Memory Management

> Every key in Redis can have an **expiration time**. When it expires, Redis automatically deletes it. This is the foundation of caching.

```bash
# ═══════════════════════════════════════
#  Setting expiry
# ═══════════════════════════════════════

SET session:abc "user_data" EX 3600        # Expire in 3600 seconds
SET session:def "user_data" PX 60000       # Expire in 60000 milliseconds
SET session:ghi "user_data" EXAT 1704067200  # Expire at Unix timestamp

EXPIRE existing_key 300                     # Set expiry on existing key
EXPIREAT existing_key 1704067200           # Set expiry at timestamp

# ═══════════════════════════════════════
#  Checking TTL
# ═══════════════════════════════════════

TTL session:abc                # → 3542 (seconds remaining)
PTTL session:abc               # → 3542000 (milliseconds remaining)
# → -1 means no expiry set
# → -2 means key doesn't exist

# ═══════════════════════════════════════
#  Remove expiry
# ═══════════════════════════════════════

PERSIST session:abc            # Remove expiry — key lives forever

# ═══════════════════════════════════════
#  How Redis handles expired keys
# ═══════════════════════════════════════

# Two strategies (both active):
# 
# 1. LAZY DELETION:
#    When a client tries to access an expired key → Redis checks → deletes
#    Problem: What if no one accesses it? Zombie keys eat memory!
#
# 2. ACTIVE EXPIRATION (background):
#    Redis randomly samples 20 keys with expiry every 100ms
#    If >25% are expired → repeat immediately
#    This ensures memory cleanup even for unaccessed keys
```

---

## 🔑 Key Naming Conventions — The Redis Way

```bash
# ═══════════════════════════════════════
#  Use colons as namespace separators
# ═══════════════════════════════════════

# ✅ GOOD: Clear, hierarchical naming
user:1001:profile              # User 1001's profile
user:1001:sessions             # User 1001's sessions
order:5042:items               # Order 5042's items
cache:api:/v1/products         # API cache
rate:ip:192.168.1.1            # Rate limiter
lock:resource:inventory:42     # Distributed lock

# ❌ BAD: Unclear, inconsistent
"alice_profile"                # No namespace
"12345"                        # What is this?
"SESSION-abc"                  # Mixed conventions

# ═══════════════════════════════════════
#  Key management commands
# ═══════════════════════════════════════

KEYS user:*                    # Find all user keys (⚠️ SLOW — blocks server!)
SCAN 0 MATCH user:* COUNT 100 # Safe alternative — cursor-based iteration
TYPE user:1                    # → "hash" (what data type?)
EXISTS user:1                  # → 1 (exists) or 0 (doesn't exist)
DEL user:1                     # Delete key (blocking)
UNLINK user:1                  # Delete key (non-blocking, async)
RENAME user:1 user:1001        # Rename key
OBJECT ENCODING user:1         # → "ziplist" or "hashtable" (internal encoding)
```

> ⚠️ **CRITICAL WARNING:** Never use `KEYS *` in production! It scans ALL keys and blocks the server. Use `SCAN` instead.

---

## 🆚 Redis Data Structure Decision Matrix

```
╔═══════════════════════════════════════════════════════════════════════════╗
║  WHAT YOU NEED                          │  USE THIS         │ TIME       ║
╠═══════════════════════════════════════════════════════════════════════════╣
║  Simple key-value (cache, session)      │  STRING           │ O(1)       ║
║  Counter / Rate limiter                 │  STRING (INCR)    │ O(1)       ║
║  Object with fields (user profile)      │  HASH             │ O(1)/field ║
║  Queue (FIFO) / Stack (LIFO)            │  LIST             │ O(1) push  ║
║  Unique items / Tags / Memberships      │  SET              │ O(1) add   ║
║  Leaderboard / Rankings / Priorities    │  SORTED SET       │ O(log n)   ║
║  Event log / Message queue              │  STREAM           │ O(1) add   ║
║  Boolean flags at scale                 │  BITMAP           │ O(1)       ║
║  Approximate unique count               │  HYPERLOGLOG      │ O(1)       ║
║  Nearby locations / Distance            │  GEO              │ O(log n)   ║
║  "Maybe exists" probabilistic check     │  BLOOM FILTER*    │ O(k)       ║
║  Full-text search inside Redis          │  REDISEARCH*      │ varies     ║
║  Time-series data natively              │  REDISTIMESERIES* │ O(1) add   ║
╠═══════════════════════════════════════════════════════════════════════════╣
║  * = Requires Redis Stack / Redis Modules                               ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

---

## 🏗️ Redis Memory Model — Understanding Internals

```
┌──────────────────────────────────────────────────────────────────┐
│                    REDIS MEMORY LAYOUT                            │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Key Space (main dictionary)                               │  │
│  │  ┌──────────┬─────────────┬─────────────────────────────┐ │  │
│  │  │   Key    │ Type/Encode │        Value Object          │ │  │
│  │  ├──────────┼─────────────┼─────────────────────────────┤ │  │
│  │  │ "name"   │ string/embr │ "Alice"                     │ │  │
│  │  │ "user:1" │ hash/ziplist│ {name:"A", age:30}          │ │  │
│  │  │ "scores" │ zset/skip   │ Skip List + Hash Table      │ │  │
│  │  │ "queue"  │ list/quick  │ Quicklist (linked ziplists) │ │  │
│  │  └──────────┴─────────────┴─────────────────────────────┘ │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Internal Encodings (Redis chooses automatically!)         │  │
│  │                                                            │  │
│  │  String: int (if numeric) / embstr (≤44 bytes) / raw      │  │
│  │  Hash:   ziplist (small) → hashtable (large)               │  │
│  │  List:   quicklist (always, since Redis 3.2)               │  │
│  │  Set:    intset (all integers) → hashtable                 │  │
│  │  ZSet:   ziplist (small) → skiplist + hashtable            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────┐                                │
│  │  Memory Overhead Per Key     │                                │
│  │  Key pointer:    ~8 bytes    │                                │
│  │  Value pointer:  ~8 bytes    │                                │
│  │  Expiry (if set): ~8 bytes   │                                │
│  │  Redis object:   ~16 bytes   │                                │
│  │  Dict entry:     ~24 bytes   │                                │
│  │  ─────────────────────────   │                                │
│  │  Overhead per key: ~64 bytes │                                │
│  └──────────────────────────────┘                                │
└──────────────────────────────────────────────────────────────────┘
```

### Memory Optimization Tricks

```bash
# Check memory usage:
INFO memory
# used_memory: 1.2 GB
# used_memory_peak: 1.8 GB
# mem_fragmentation_ratio: 1.05 (healthy: 1.0-1.5)

MEMORY USAGE user:1            # → 120 bytes (exact memory for this key)
MEMORY DOCTOR                  # Redis memory diagnostics

# ═══════════════════════════════════════
#  Memory optimization strategies
# ═══════════════════════════════════════

# 1. Use Hashes for small objects (ziplist encoding):
#    Keep < 128 fields and < 64 bytes per value
#    Config: hash-max-ziplist-entries 128
#            hash-max-ziplist-value 64

# 2. Use short key names in high-volume scenarios:
#    "u:1:n" instead of "user:1:name" (saves bytes × millions)

# 3. Set maxmemory + eviction policy:
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru   # Evict least recently used

# 4. Use UNLINK instead of DEL for large keys:
UNLINK large_sorted_set       # Non-blocking deletion
```

---

## 🔄 Eviction Policies — What Happens When Memory Is Full?

```
╔════════════════════════════════════════════════════════════════════╗
║  POLICY               │  BEHAVIOR                                 ║
╠════════════════════════════════════════════════════════════════════╣
║  noeviction (default) │  Return error on write (safest)           ║
║  allkeys-lru          │  Evict Least Recently Used (any key)      ║
║  allkeys-lfu          │  Evict Least Frequently Used (any key)    ║
║  allkeys-random       │  Evict random keys                        ║
║  volatile-lru         │  LRU but only keys WITH expiry            ║
║  volatile-lfu         │  LFU but only keys WITH expiry            ║
║  volatile-random      │  Random but only keys WITH expiry         ║
║  volatile-ttl         │  Evict keys closest to expiring           ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  🔥 RECOMMENDATION:                                               ║
║  Cache-only Redis: allkeys-lru or allkeys-lfu                     ║
║  Mixed (cache + persistent): volatile-lru                         ║
║  Data store (no eviction): noeviction                             ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## 💾 Persistence — Redis Doesn't Forget!

### RDB Snapshots (Point-in-Time Snapshots)

```
How RDB works:

  Time 0:        Redis is running, data in RAM
  Time T:        SAVE or BGSAVE triggered
                 └─ Redis forks child process
                 └─ Child writes entire dataset to dump.rdb
                 └─ Parent continues serving requests (no downtime!)
  Time T+5s:     dump.rdb is complete
  
  ┌──────────────────────────┐       ┌──────────────┐
  │    Redis Main Process    │ fork  │ Child Process │
  │    (serves clients)      │──────▶│ (writes RDB)  │
  │    RAM: 8 GB data        │       │ dump.rdb      │
  └──────────────────────────┘       └──────────────┘
```

```bash
# RDB configuration in redis.conf:
save 900 1        # Save if 1 key changed in 900 seconds
save 300 10       # Save if 10 keys changed in 300 seconds
save 60 10000     # Save if 10000 keys changed in 60 seconds

# Manual trigger:
SAVE              # ⚠️ BLOCKS the server! (don't use in production)
BGSAVE            # Background save (non-blocking, use this!)
```

```
RDB Pros & Cons:
╔════════════════════════════╦════════════════════════════╗
║  ✅ PROS                   ║  ❌ CONS                   ║
╠════════════════════════════╬════════════════════════════╣
║  Compact single file       ║  Data loss between saves   ║
║  Fast restart (load .rdb)  ║  fork() can be slow on     ║
║  Perfect for backups       ║  large datasets (8GB+ RAM) ║
║  Minimal performance hit   ║  Not suitable for "zero    ║
║                            ║  data loss" requirements   ║
╚════════════════════════════╩════════════════════════════╝
```

### AOF (Append-Only File)

```
How AOF works:

  Every write command → appended to appendonly.aof file

  Client: SET user:1 "Alice"
  AOF File: *3\r\n$3\r\nSET\r\n$6\r\nuser:1\r\n$5\r\nAlice\r\n

  Client: INCR counter
  AOF File: *2\r\n$4\r\nINCR\r\n$7\r\ncounter\r\n

  On restart: Redis replays the AOF file → data restored!
```

```bash
# AOF configuration:
appendonly yes

# Sync policies:
appendfsync always      # Safest: fsync every command (slowest)
appendfsync everysec    # ⭐ RECOMMENDED: fsync every second (good balance)
appendfsync no          # Fastest: let OS decide when to flush (risky)
```

```
AOF Pros & Cons:
╔════════════════════════════╦════════════════════════════╗
║  ✅ PROS                   ║  ❌ CONS                   ║
╠════════════════════════════╬════════════════════════════╣
║  Near-zero data loss       ║  File grows larger than    ║
║  (everysec = max 1s loss)  ║  RDB (commands, not data)  ║
║  Human-readable log        ║  Slower restart (replay)   ║
║  Can be replayed/edited    ║  Higher write latency      ║
║                            ║  with appendfsync always   ║
╚════════════════════════════╩════════════════════════════╝
```

### RDB + AOF = Best of Both Worlds (Recommended!)

```bash
# redis.conf — Recommended production settings:
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes    # ⭐ Hybrid! AOF file starts with RDB format
                             # Faster restarts + near-zero data loss
```

```
Hybrid Persistence (Redis 4.0+):

  appendonly.aof file structure:
  ┌────────────────────────────────────────┐
  │  [RDB PREAMBLE - full snapshot]        │  ← Fast to load
  │  [AOF COMMANDS - since last snapshot]  │  ← Recent changes
  └────────────────────────────────────────┘

  Restart: Load RDB (fast) → Replay few AOF commands → DONE!
  Best of both worlds: Fast restart + minimal data loss
```

---

## 🌐 When to Use Redis (And When NOT To)

```
╔════════════════════════════════════════════════════════════════════╗
║  ✅ USE REDIS WHEN:                                               ║
╠════════════════════════════════════════════════════════════════════╣
║  • You need sub-millisecond response times                        ║
║  • Caching database queries, API responses, sessions              ║
║  • Real-time leaderboards, rankings, counting                     ║
║  • Message queues, pub/sub, event streaming                       ║
║  • Rate limiting, distributed locks                               ║
║  • Geospatial queries (nearby locations)                          ║
║  • Real-time analytics, metrics collection                        ║
║  • Session storage for web applications                           ║
║  • Feature flags, A/B testing flags                               ║
╠════════════════════════════════════════════════════════════════════╣
║  ❌ DO NOT USE REDIS WHEN:                                        ║
╠════════════════════════════════════════════════════════════════════╣
║  • Data is larger than available RAM (use disk-based DB)          ║
║  • You need complex queries (JOINs, aggregations → use SQL)       ║
║  • Strong ACID transactions across multiple keys (use PostgreSQL) ║
║  • Long-term archival storage (use S3, data warehouse)            ║
║  • Full-text search at scale (use Elasticsearch)                  ║
║  • You need relationships between data (use Graph DB)             ║
║  • Cost is a concern for large datasets (RAM is expensive!)       ║
╠════════════════════════════════════════════════════════════════════╣
║  💡 GOLDEN RULE: Use Redis AS A COMPLEMENT to your primary DB.   ║
║     PostgreSQL/MySQL = source of truth                            ║
║     Redis = speed layer / cache / real-time features              ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## 🧪 Quick Hands-On Lab — Build a Mini Twitter Backend in Redis

```bash
# ═══════════════════════════════════════
#  1. Users (Hash)
# ═══════════════════════════════════════
HSET user:1 username "alice" name "Alice Dev" followers 0 following 0
HSET user:2 username "bob" name "Bob Coder" followers 0 following 0

# ═══════════════════════════════════════
#  2. Follow System (Set)
# ═══════════════════════════════════════
SADD followers:user:1 "user:2"          # Bob follows Alice
SADD following:user:2 "user:1"          # Bob is following Alice
HINCRBY user:1 followers 1             # Alice's follower count
HINCRBY user:2 following 1             # Bob's following count

# ═══════════════════════════════════════
#  3. Post a Tweet (Hash + List + Sorted Set)
# ═══════════════════════════════════════
SET next_tweet_id 0
INCR next_tweet_id                      # → 1
HSET tweet:1 text "Hello Redis!" author "user:1" timestamp 1704067200 likes 0

# Add to Alice's timeline:
LPUSH timeline:user:1 "tweet:1"
# Add to followers' timelines (fan-out):
SMEMBERS followers:user:1               # → ["user:2"]
# For each follower: LPUSH timeline:user:2 "tweet:1"

# ═══════════════════════════════════════
#  4. Like a Tweet (Set + Hash)
# ═══════════════════════════════════════
SADD likes:tweet:1 "user:2"            # Bob liked tweet:1
HINCRBY tweet:1 likes 1                # Increment like counter

# Check if Bob already liked:
SISMEMBER likes:tweet:1 "user:2"       # → 1 (already liked — prevent double-like!)

# ═══════════════════════════════════════
#  5. Trending Hashtags (Sorted Set)
# ═══════════════════════════════════════
ZINCRBY trending:hashtags 1 "#redis"
ZINCRBY trending:hashtags 1 "#databases"
ZINCRBY trending:hashtags 1 "#redis"   # Redis trending harder!
ZREVRANGE trending:hashtags 0 4 WITHSCORES  # Top 5 trending

# ═══════════════════════════════════════
#  6. Online Users (Set)
# ═══════════════════════════════════════
SADD online_users "user:1" "user:2"
SCARD online_users                     # → 2 users online

# ═══════════════════════════════════════
#  7. Feed (List — get latest 10 tweets)
# ═══════════════════════════════════════
LRANGE timeline:user:2 0 9            # Bob's latest 10 tweets
```

> 🔥 **You just built the core of Twitter's backend in ~25 Redis commands!** That's the power of Redis — the right data structure for each problem.

---

## 📊 Chapter Summary — Quick Reference Card

```
╔══════════════════════════════════════════════════════════════════════╗
║                    REDIS ARCHITECTURE CHEAT SHEET                    ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  WHAT IS REDIS?                                                      ║
║  → In-memory data structure store (database + cache + broker)        ║
║  → Single-threaded command execution, multi-threaded I/O (6.0+)     ║
║  → Sub-millisecond latency, 100K-1M+ ops/sec                        ║
║                                                                      ║
║  DATA STRUCTURES:                                                    ║
║  STRING  → simple values, counters, cache       │ O(1)               ║
║  HASH    → objects, profiles, configs            │ O(1) per field     ║
║  LIST    → queues, stacks, feeds                 │ O(1) push/pop     ║
║  SET     → unique items, tags, intersections     │ O(1) add/check    ║
║  ZSET    → rankings, leaderboards, priorities    │ O(log n)          ║
║  STREAM  → event log, consumer groups            │ O(1) add          ║
║  BITMAP  → boolean flags at scale                │ O(1)              ║
║  HYPERLL → approximate unique counting           │ O(1), 12 KB       ║
║  GEO     → nearby locations, distances           │ O(log n)          ║
║                                                                      ║
║  PERSISTENCE:                                                        ║
║  RDB  → point-in-time snapshots (compact, fast restart)              ║
║  AOF  → append every command (durable, larger files)                 ║
║  Both → ⭐ recommended (hybrid mode with RDB preamble)              ║
║                                                                      ║
║  MEMORY MANAGEMENT:                                                  ║
║  maxmemory → set limit │ allkeys-lru → best for caching             ║
║  KEYS * is EVIL → use SCAN │ UNLINK > DEL for large keys            ║
║                                                                      ║
║  KEY NAMING: object:id:field (e.g., user:42:profile)                 ║
║  DEFAULT PORT: 6379                                                  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|---|---|
| [3C.2 → Redis as Cache](./02-Redis-Caching.md) | Cache-aside, Write-through, TTL strategies, Cache invalidation — master the #1 Redis use case |
| [3C.3 → Redis Advanced](./03-Redis-Advanced.md) | Pub/Sub, Streams deep dive, Lua scripting, Pipelining, Transactions |
| [3C.4 → Redis Cluster & HA](./04-Redis-Cluster.md) | Redis Cluster, Sentinel, persistence deep dive, production deployment |

---

> 💡 **Remember:** Redis is not just a cache. It's a **data structure server**. The moment you stop thinking "key-value" and start thinking "what data structure solves my problem?", you unlock Redis's true power.

---

*Next up: Let's make you a caching expert →* [Chapter 3C.2 — Redis as Cache](./02-Redis-Caching.md)
