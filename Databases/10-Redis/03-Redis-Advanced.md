# 🧪 Chapter 3C.3 — Redis Advanced — Pub/Sub, Lua, Transactions & Modules

> **Level:** 🔴 Advanced
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 3C.1 (Architecture), Chapter 3C.2 (Caching)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Build **real-time systems** with Redis Pub/Sub and Streams
- Write **atomic server-side logic** with Lua scripting
- Understand Redis **transactions** (MULTI/EXEC) and their limitations
- Use **Pipelining** to achieve 10x throughput improvement
- Know the **Redis Modules** ecosystem (RediSearch, RedisJSON, RedisBloom, RedisTimeSeries)
- Implement **distributed locks** that actually work in production

---

## 📡 Pub/Sub — Real-Time Messaging

> **Pub/Sub = Publish/Subscribe.** Senders (publishers) don't send messages to specific receivers. They publish to **channels**. Receivers (subscribers) listen to channels they care about.

### How It Works

```
                         REDIS SERVER
                    ┌───────────────────┐
  Publishers:       │    CHANNELS       │       Subscribers:
                    │                   │
  ┌──────────┐     │  ┌─────────────┐  │     ┌──────────┐
  │ App A    │──── │─▶│  chat:room1 │──│────▶│ User 1   │
  └──────────┘     │  └─────────────┘  │     └──────────┘
                    │        │          │     ┌──────────┐
  ┌──────────┐     │        └──────────│────▶│ User 2   │
  │ App B    │──── │─▶┌─────────────┐  │     └──────────┘
  └──────────┘     │  │  chat:room2 │──│     ┌──────────┐
                    │  └─────────────┘  │────▶│ User 3   │
  ┌──────────┐     │                   │     └──────────┘
  │ App C    │──── │─▶┌─────────────┐  │
  └──────────┘     │  │  alerts     │──│     ┌──────────┐
                    │  └─────────────┘  │────▶│ Monitor  │
                    └───────────────────┘     └──────────┘

  ⚡ Messages are FIRE-AND-FORGET
  ⚠️ If no subscriber is listening → message is LOST
  ⚠️ No message persistence (unlike Streams)
```

### Basic Pub/Sub Commands

```bash
# ═══════════════════════════════════════
#  Terminal 1: SUBSCRIBER (listener)
# ═══════════════════════════════════════

SUBSCRIBE chat:general
# Reading messages... (waiting)
# 1) "subscribe"
# 2) "chat:general"
# 3) (integer) 1

# When a message arrives:
# 1) "message"
# 2) "chat:general"
# 3) "Hello everyone!"

# ═══════════════════════════════════════
#  Terminal 2: PUBLISHER (sender)
# ═══════════════════════════════════════

PUBLISH chat:general "Hello everyone!"
# → (integer) 2   (2 subscribers received the message)

PUBLISH chat:general "Redis is awesome!"
# → (integer) 2

# ═══════════════════════════════════════
#  Pattern Subscriptions (wildcards!)
# ═══════════════════════════════════════

PSUBSCRIBE chat:*              # Subscribe to ALL chat channels!
# Receives messages from: chat:general, chat:room1, chat:vip, etc.

PSUBSCRIBE alert:*:critical    # Subscribe to all critical alerts
# Matches: alert:server:critical, alert:db:critical, etc.

# Unsubscribe:
UNSUBSCRIBE chat:general
PUNSUBSCRIBE chat:*
```

### Real-World Pub/Sub Example

```python
# Python — Real-Time Chat System

import redis
import threading

r = redis.Redis()

# ═══════════════════════════════════════
#  Subscriber (runs in background thread)
# ═══════════════════════════════════════

def message_handler():
    pubsub = r.pubsub()
    pubsub.subscribe("chat:general", "notifications")
    pubsub.psubscribe("alert:*")  # Pattern subscribe
    
    for message in pubsub.listen():
        if message["type"] == "message":
            channel = message["channel"].decode()
            data = message["data"].decode()
            print(f"[{channel}] {data}")
        elif message["type"] == "pmessage":
            pattern = message["pattern"].decode()
            channel = message["channel"].decode()
            data = message["data"].decode()
            print(f"[{pattern} → {channel}] {data}")

listener = threading.Thread(target=message_handler, daemon=True)
listener.start()

# ═══════════════════════════════════════
#  Publisher
# ═══════════════════════════════════════

r.publish("chat:general", "Alice: Hey everyone!")
r.publish("chat:general", "Bob: Hey Alice!")
r.publish("notifications", "New follower: Charlie")
r.publish("alert:server:critical", "CPU > 95%!")
```

### Pub/Sub Use Cases & Limitations

```
╔═══════════════════════════════════════════════════════════════════════╗
║  ✅ PERFECT FOR:                                                      ║
╠═══════════════════════════════════════════════════════════════════════╣
║  • Real-time chat messages                                            ║
║  • Live notifications (new follower, new message)                    ║
║  • Cache invalidation across multiple servers                        ║
║  • Live dashboard updates                                            ║
║  • Configuration change propagation                                  ║
║  • WebSocket message broadcasting                                    ║
╠═══════════════════════════════════════════════════════════════════════╣
║  ❌ NOT SUITABLE FOR:                                                 ║
╠═══════════════════════════════════════════════════════════════════════╣
║  • Message persistence (messages are lost if no subscriber)          ║
║  • Guaranteed delivery (fire-and-forget, no ACK)                     ║
║  • Message replay (can't read past messages)                         ║
║  • Consumer groups (can't load-balance across consumers)             ║
║  • For these → use Redis STREAMS or a message broker (Kafka/RabbitMQ)║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

## 🌊 Streams Deep Dive — Kafka-Lite in Redis

> **Streams fix every Pub/Sub limitation.** Persistent, replayable, consumer groups, acknowledgment. Added in Redis 5.0.

### Pub/Sub vs Streams — Key Differences

```
╔═══════════════════════════════════════════════════════════════════════╗
║  Feature                    │  Pub/Sub          │  Streams           ║
╠═══════════════════════════════════════════════════════════════════════╣
║  Persistence                │  ❌ Fire-and-forget│  ✅ Persisted      ║
║  Message replay             │  ❌ No             │  ✅ Yes            ║
║  Consumer groups            │  ❌ No             │  ✅ Yes            ║
║  Acknowledgment             │  ❌ No             │  ✅ XACK           ║
║  Message ordering           │  ❌ No guarantee   │  ✅ Guaranteed     ║
║  Backpressure               │  ❌ No             │  ✅ XREADGROUP     ║
║  Dead letter handling       │  ❌ No             │  ✅ Pending list   ║
║  Speed                      │  ⚡ Fastest        │  ⚡ Very fast      ║
║  Use when                   │  Real-time, lossy  │  Reliable queues  ║
║                             │  notifications     │  Event sourcing    ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Consumer Groups — The Killer Feature

```bash
# ═══════════════════════════════════════
#  Setup: Create stream and consumer group
# ═══════════════════════════════════════

# Create consumer group (reads new messages only)
XGROUP CREATE orders processors $ MKSTREAM

# ═══════════════════════════════════════
#  Producers: Add events
# ═══════════════════════════════════════

XADD orders * action "purchase" product "laptop" amount 999
XADD orders * action "purchase" product "phone" amount 699
XADD orders * action "refund" product "tablet" amount 499
XADD orders * action "purchase" product "mouse" amount 29

# ═══════════════════════════════════════
#  Consumers: Process events (load balanced!)
# ═══════════════════════════════════════

# Consumer 1 reads:
XREADGROUP GROUP processors worker1 COUNT 2 STREAMS orders >
# Gets message 1 & 2 (laptop + phone)

# Consumer 2 reads:
XREADGROUP GROUP processors worker2 COUNT 2 STREAMS orders >
# Gets message 3 & 4 (tablet + mouse) — DIFFERENT messages!

# ═══════════════════════════════════════
#  Acknowledge processed messages
# ═══════════════════════════════════════

XACK orders processors "1704067200000-0"   # Worker1 done with message 1
XACK orders processors "1704067200001-0"   # Worker1 done with message 2

# ═══════════════════════════════════════
#  Check pending (unacknowledged) messages
# ═══════════════════════════════════════

XPENDING orders processors - + 10
# Shows messages that were read but NOT yet acknowledged
# If a worker crashes, these can be re-assigned!

# ═══════════════════════════════════════
#  Claim stale messages (worker died)
# ═══════════════════════════════════════

# If worker2 crashed and has pending messages for > 30 seconds:
XAUTOCLAIM orders processors worker1 30000 0-0 COUNT 10
# worker1 takes over worker2's pending messages!
```

```
Consumer Group Architecture:

  Producers                     Stream                   Consumer Group
  ┌──────┐                                              ┌─────────────┐
  │App 1 │──XADD───▶ ┌─────────────────────┐           │ "processors"│
  └──────┘           │ orders stream        │           │             │
  ┌──────┐           │ ┌───┐┌───┐┌───┐┌───┐│ XREADGROUP│ ┌─────────┐│
  │App 2 │──XADD───▶ │ │m1 ││m2 ││m3 ││m4 ││──────────▶│ │worker1 ││ → m1, m2
  └──────┘           │ └───┘└───┘└───┘└───┘│           │ └─────────┘│
  ┌──────┐           │                      │           │ ┌─────────┐│
  │App 3 │──XADD───▶ │ Persistent!          │──────────▶│ │worker2 ││ → m3, m4
  └──────┘           │ Replayable!          │           │ └─────────┘│
                      └─────────────────────┘           │ ┌─────────┐│
                                                        │ │worker3 ││ → (idle)
                                                        │ └─────────┘│
                                                        └─────────────┘
  
  Each message delivered to exactly ONE consumer in the group!
  Failed messages can be reclaimed by other consumers.
```

### Stream Trimming (Memory Management)

```bash
# Limit stream size:
XADD orders MAXLEN ~ 10000 * product "laptop" price 999
# ~ means "approximately" — Redis may keep slightly more (efficient)

# Trim explicitly:
XTRIM orders MAXLEN 10000

# By minimum ID:
XTRIM orders MINID 1704067200000-0    # Remove entries older than this ID

# Check stream info:
XLEN orders                            # → Number of entries
XINFO STREAM orders                    # Detailed stream information
XINFO GROUPS orders                    # Consumer group details
```

---

## 🔒 Transactions — MULTI/EXEC

> Redis transactions execute a group of commands **atomically** (all or nothing). No other client can interrupt in the middle.

### Basic Transactions

```bash
# ═══════════════════════════════════════
#  MULTI / EXEC — Atomic command batch
# ═══════════════════════════════════════

MULTI                          # Start transaction
SET account:alice 1000         # QUEUED (not executed yet!)
SET account:bob 500            # QUEUED
DECRBY account:alice 200       # QUEUED
INCRBY account:bob 200         # QUEUED
EXEC                           # Execute ALL at once — atomically!
# → [OK, OK, 800, 700]

# Nobody can see an intermediate state where Alice has 800
# but Bob still has 500. It's all or nothing!

# ═══════════════════════════════════════
#  DISCARD — Cancel transaction
# ═══════════════════════════════════════

MULTI
SET key1 "value1"              # QUEUED
SET key2 "value2"              # QUEUED
DISCARD                        # Cancel! Nothing executed.
```

### WATCH — Optimistic Locking

```bash
# ═══════════════════════════════════════
#  WATCH — Check-and-Set (CAS)
# ═══════════════════════════════════════

# Scenario: Transfer $200 from Alice to Bob (safely)

WATCH account:alice            # Watch for changes
GET account:alice              # → "1000"

# Before executing, check if balance >= 200
# If another client changes account:alice between WATCH and EXEC:

MULTI
DECRBY account:alice 200
INCRBY account:bob 200
EXEC
# Returns nil if account:alice was modified by another client!
# → Transaction aborted! Retry needed.

# Returns [800, 700] if no conflict → Success!
```

```python
# Python — Optimistic Locking Pattern
def transfer_money(from_account, to_account, amount):
    with r.pipeline() as pipe:
        while True:
            try:
                # Watch the source account
                pipe.watch(f"account:{from_account}")
                
                # Check balance
                balance = int(pipe.get(f"account:{from_account}"))
                if balance < amount:
                    raise ValueError("Insufficient funds")
                
                # Start transaction
                pipe.multi()
                pipe.decrby(f"account:{from_account}", amount)
                pipe.incrby(f"account:{to_account}", amount)
                
                # Execute (raises WatchError if key changed)
                pipe.execute()
                print("Transfer successful! ✅")
                break
                
            except redis.WatchError:
                print("Conflict detected! Retrying... 🔄")
                continue  # Retry the entire operation
```

### ⚠️ Transaction Limitations

```
╔════════════════════════════════════════════════════════════════════╗
║  REDIS TRANSACTIONS ≠ SQL TRANSACTIONS                            ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  ❌ No ROLLBACK on error!                                         ║
║     If command 3 of 5 fails → commands 1, 2 already executed     ║
║     Commands 4, 5 still execute. Only command 3 returns error.    ║
║                                                                    ║
║  ❌ No conditional logic inside transaction                       ║
║     Can't do: IF balance > 100 THEN decrby ...                   ║
║     → For this: use Lua scripts!                                  ║
║                                                                    ║
║  ❌ Commands are queued, not executed during MULTI                ║
║     Can't read values and use them within the same transaction    ║
║     → For this: use WATCH (optimistic) or Lua (server-side)      ║
║                                                                    ║
║  ✅ Atomicity: No other client can interleave commands            ║
║  ✅ Isolation: Other clients see before or after, never during    ║
║  ✅ WATCH provides optimistic locking                             ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## 🧬 Lua Scripting — Server-Side Superpowers

> **Lua scripts execute ATOMICALLY on the Redis server.** No other command can run in the middle. This is THE solution for complex atomic operations.

### Why Lua?

```
Problem: You need to READ a value and conditionally WRITE based on it.

  ❌ Client-side approach (NOT ATOMIC):
     balance = GET account:alice         # → 1000
     if balance >= 200:                  # Check on client
         DECRBY account:alice 200       # Another client might change
                                         # balance between GET and DECRBY!
     
  ✅ Lua script (ATOMIC — runs on server):
     local balance = redis.call('GET', 'account:alice')
     if tonumber(balance) >= 200 then
         redis.call('DECRBY', 'account:alice', 200)
         return "OK"
     end
     return "INSUFFICIENT_FUNDS"
```

### Basic Lua Scripts

```bash
# ═══════════════════════════════════════
#  EVAL — Run Lua script inline
# ═══════════════════════════════════════

# Simple example: Get and increment atomically
EVAL "local val = redis.call('GET', KEYS[1]); redis.call('SET', KEYS[1], val .. ' World'); return val" 1 greeting
# KEYS[1] = "greeting"
# Returns old value, sets new value — atomically!

# ═══════════════════════════════════════
#  Rate Limiter (Sliding Window — production-grade!)
# ═══════════════════════════════════════

EVAL "
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])
  
  -- Remove old entries outside the window
  redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)
  
  -- Count current requests
  local count = redis.call('ZCARD', key)
  
  if count < limit then
    -- Under limit: allow and record
    redis.call('ZADD', key, now, now .. ':' .. math.random())
    redis.call('EXPIRE', key, window)
    return 1  -- ALLOWED
  else
    return 0  -- RATE LIMITED
  end
" 1 rate:user:42 100 60 1704067200
# Args: key, limit=100, window=60s, current_time

# ═══════════════════════════════════════
#  Distributed Lock (Correct Implementation!)
# ═══════════════════════════════════════

# ACQUIRE lock:
EVAL "
  if redis.call('SET', KEYS[1], ARGV[1], 'NX', 'EX', ARGV[2]) then
    return 1
  end
  return 0
" 1 lock:resource:42 "worker-abc-123" 30
# Only sets if NOT exists, with 30s expiry
# ARGV[1] = unique lock owner ID (prevents someone else from releasing)

# RELEASE lock (only if YOU own it!):
EVAL "
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  end
  return 0
" 1 lock:resource:42 "worker-abc-123"
# Only deletes if the lock value matches your worker ID!
# Without this check: Worker A could release Worker B's lock! 💀
```

### EVALSHA — Cached Scripts (Production Use)

```bash
# Problem: Sending entire Lua script every time wastes bandwidth

# Solution: Load script once, call by SHA hash

# Step 1: Load script
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → "e0e1f9fabfc9d4800c877a703b823ac0578ff831" (SHA1 hash)

# Step 2: Call by hash (much faster!)
EVALSHA "e0e1f9fabfc9d4800c877a703b823ac0578ff831" 1 mykey
# Same result, less bandwidth!

# Check if script exists:
SCRIPT EXISTS "e0e1f9fabfc9d4800c877a703b823ac0578ff831"
# → 1 (yes) or 0 (no)

# Flush all cached scripts:
SCRIPT FLUSH
```

### Redis Functions (Redis 7.0+ — Next-Gen Lua)

```bash
# Redis 7.0 introduced Functions — named, persistent Lua functions!

# Register a function library:
FUNCTION LOAD "#!lua name=mylib
  redis.register_function('my_hset_ttl', function(keys, args)
    redis.call('HSET', keys[1], args[1], args[2])
    redis.call('EXPIRE', keys[1], args[3])
    return 'OK'
  end)
"

# Call the function:
FCALL my_hset_ttl 1 user:42 name "Alice" 3600
# Sets hash field AND expiry in one atomic call!

# Functions persist across restarts (unlike EVAL scripts)!
```

### ⚠️ Lua Script Rules

```
╔════════════════════════════════════════════════════════════════════════╗
║  LUA SCRIPTING RULES:                                                 ║
╠════════════════════════════════════════════════════════════════════════╣
║                                                                        ║
║  1. Scripts are ATOMIC — blocking the server while running            ║
║     → Keep scripts SHORT (< 5ms recommended)                         ║
║     → Long scripts = frozen Redis = angry users                      ║
║                                                                        ║
║  2. All keys accessed MUST be passed as KEYS[]                       ║
║     → Required for Redis Cluster to route correctly                  ║
║     → KEYS[1], KEYS[2], etc. for keys                               ║
║     → ARGV[1], ARGV[2], etc. for other arguments                    ║
║                                                                        ║
║  3. Scripts must be DETERMINISTIC                                     ║
║     → No random(), no time() (use ARGV to pass these in)            ║
║     → Required for replication to work correctly                     ║
║                                                                        ║
║  4. redis.call() vs redis.pcall()                                    ║
║     → redis.call(): raises error on failure (stops script)           ║
║     → redis.pcall(): returns error as value (you handle it)          ║
║                                                                        ║
║  5. Default timeout: 5 seconds (configurable: lua-time-limit)       ║
║     → If exceeded: server accepts SCRIPT KILL or SHUTDOWN NOSAVE    ║
╚════════════════════════════════════════════════════════════════════════╝
```

---

## 🚀 Pipelining — 10x Throughput Boost

> **Pipelining sends multiple commands without waiting for individual responses.** Dramatically reduces network round-trips.

```
Without Pipelining:
  Client          Network          Redis
  ──────          ───────          ─────
  SET key1 ──────────────────────▶ Process
            ◀────────────────────── OK
  SET key2 ──────────────────────▶ Process
            ◀────────────────────── OK
  SET key3 ──────────────────────▶ Process
            ◀────────────────────── OK
  
  3 commands × 2 round trips = 6 network hops
  Total: 3 × RTT (~0.5ms each) = ~1.5ms

With Pipelining:
  Client          Network          Redis
  ──────          ───────          ─────
  SET key1 ─┐
  SET key2 ─┼────────────────────▶ Process all
  SET key3 ─┘                      ┌── OK
            ◀──────────────────────┤── OK
                                   └── OK
  
  3 commands × 1 round trip = 2 network hops
  Total: 1 × RTT = ~0.5ms (3x faster!)
```

```python
# Python Pipelining Example

# ❌ Without pipeline: 1000 commands = 1000 round trips = ~500ms
for i in range(1000):
    r.set(f"key:{i}", f"value:{i}")

# ✅ With pipeline: 1000 commands = 1 round trip = ~5ms (100x faster!)
pipe = r.pipeline(transaction=False)  # No MULTI/EXEC wrapper
for i in range(1000):
    pipe.set(f"key:{i}", f"value:{i}")
results = pipe.execute()  # All 1000 results at once!

# ✅ Pipeline with reads:
pipe = r.pipeline(transaction=False)
for i in range(100):
    pipe.get(f"user:{i}:name")
    pipe.get(f"user:{i}:email")
names_and_emails = pipe.execute()  # 200 results in one batch!
```

### Pipeline vs MULTI/EXEC vs Lua

```
╔══════════════════════════════════════════════════════════════════╗
║  Feature           │ Pipeline    │ MULTI/EXEC  │ Lua Script    ║
╠══════════════════════════════════════════════════════════════════╣
║  Batching          │ ✅ Yes       │ ✅ Yes       │ ✅ Yes        ║
║  Atomic            │ ❌ No        │ ✅ Yes       │ ✅ Yes        ║
║  Read + Write      │ ❌ Can't use │ ❌ Can't use │ ✅ Yes        ║
║  in same batch     │  results    │  results    │ (full logic)  ║
║  Network round     │ 1           │ 1           │ 1             ║
║  trips             │             │             │               ║
║  Server blocking   │ ❌ No        │ Brief       │ ⚠️ Yes        ║
║  Conditional logic │ ❌ No        │ ❌ No        │ ✅ Yes        ║
║  Best for          │ Bulk ops    │ Simple      │ Complex       ║
║                    │ (no deps)   │ atomic ops  │ atomic logic  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔐 Distributed Locks — The Redlock Algorithm

> Need to ensure only ONE worker processes a task across multiple servers? You need a distributed lock.

### Simple Lock (Single Redis Instance)

```python
import uuid
import time

class RedisLock:
    def __init__(self, redis_client, resource, ttl=30):
        self.redis = redis_client
        self.resource = f"lock:{resource}"
        self.ttl = ttl
        self.token = str(uuid.uuid4())  # Unique per lock acquisition
    
    def acquire(self, timeout=10):
        """Try to acquire lock with timeout"""
        end_time = time.time() + timeout
        while time.time() < end_time:
            # SET NX = only if not exists
            if self.redis.set(self.resource, self.token, nx=True, ex=self.ttl):
                return True  # Lock acquired!
            time.sleep(0.01)  # Wait 10ms and retry
        return False  # Timeout — couldn't acquire lock
    
    def release(self):
        """Release lock only if we own it (Lua for atomicity)"""
        script = """
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            return redis.call('DEL', KEYS[1])
        end
        return 0
        """
        return self.redis.eval(script, 1, self.resource, self.token)

# Usage:
lock = RedisLock(r, "process_order:42")
if lock.acquire():
    try:
        # Critical section — only one worker runs this!
        process_order(42)
    finally:
        lock.release()
else:
    print("Could not acquire lock — another worker is processing")
```

### Redlock Algorithm (Multi-Instance — Production Grade)

```
For HIGH AVAILABILITY locks, use Redlock across N independent Redis instances:

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Redis Node 1 │  │ Redis Node 2 │  │ Redis Node 3 │
  │ (Independent)│  │ (Independent)│  │ (Independent)│
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                 │
         └────────────┬────┴────┬────────────┘
                      │  Client │
                      └─────────┘

  Algorithm:
  1. Get current time T1
  2. Try to acquire lock on ALL N nodes (with short timeout)
  3. Lock acquired if:
     a. Lock set on MAJORITY (N/2 + 1) of nodes
     b. Total time < TTL
  4. Effective TTL = initial_TTL - (current_time - T1)
  5. If failed: release lock on ALL nodes

  Why 3+ nodes? 
  → If 1 node crashes, you still have majority (2/3)
  → Single-point-of-failure eliminated!
```

```python
# Use the redis-py Redlock implementation:
# pip install redis
from redis.lock import Lock

# Simple (single instance) — most common:
with r.lock("my_resource", timeout=30, blocking_timeout=10):
    # Critical section
    process_order(42)
# Lock automatically released when block exits!
```

---

## 🧩 Redis Modules — Extending Redis

> Redis Modules add **new data types and commands** to Redis. Think of them as **plugins**.

### Redis Stack (All-in-One Bundle)

```
Redis Stack = Redis + Popular Modules:

┌──────────────────────────────────────────────────────────────┐
│                        REDIS STACK                            │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  RediSearch   │  │  RedisJSON   │  │  RedisBloom  │       │
│  │  Full-text    │  │  Native JSON │  │  Bloom filter│       │
│  │  search +     │  │  storage &   │  │  Cuckoo      │       │
│  │  secondary    │  │  querying    │  │  TopK         │       │
│  │  indexes      │  │              │  │  Count-Min   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐                          │
│  │RedisTimeSeries│  │  RedisGraph  │                          │
│  │  Time-series  │  │  Graph DB    │                          │
│  │  data with    │  │  with Cypher │                          │
│  │  downsampling │  │  queries     │                          │
│  └──────────────┘  └──────────────┘                          │
│                                                               │
│  Install: docker run -p 6379:6379 redis/redis-stack:latest   │
└──────────────────────────────────────────────────────────────┘
```

### RedisJSON — Native JSON Support

```bash
# Store and query JSON documents natively!

# Set JSON document:
JSON.SET user:1 $ '{"name":"Alice","age":30,"address":{"city":"NYC","zip":"10001"},"skills":["Python","Redis"]}'

# Get entire document:
JSON.GET user:1 $
# → [{"name":"Alice","age":30,...}]

# Get specific field:
JSON.GET user:1 $.name
# → ["Alice"]

# Get nested field:
JSON.GET user:1 $.address.city
# → ["NYC"]

# Update field:
JSON.SET user:1 $.age 31

# Increment number:
JSON.NUMINCRBY user:1 $.age 1
# → [32]

# Append to array:
JSON.ARRAPPEND user:1 $.skills '"Docker"'
JSON.GET user:1 $.skills
# → [["Python","Redis","Docker"]]

# Delete field:
JSON.DEL user:1 $.address.zip

# Why RedisJSON vs String + json.dumps?
# → Query specific fields without downloading entire document
# → Atomic updates to nested fields
# → Array operations (append, insert, trim)
# → Type-safe (numbers stay numbers, not strings)
```

### RediSearch — Full-Text Search Engine

```bash
# Create an index:
FT.CREATE idx:users 
  ON HASH PREFIX 1 "user:" 
  SCHEMA 
    name TEXT SORTABLE 
    email TEXT 
    age NUMERIC SORTABLE 
    city TAG

# Add data (regular HSET — index auto-updates!):
HSET user:1 name "Alice Johnson" email "alice@dev.com" age 30 city "NYC"
HSET user:2 name "Bob Smith" email "bob@dev.com" age 25 city "SF"
HSET user:3 name "Alice Cooper" email "acooper@dev.com" age 35 city "NYC"

# Full-text search:
FT.SEARCH idx:users "Alice"
# → user:1 and user:3

# Search with filters:
FT.SEARCH idx:users "@name:Alice @city:{NYC} @age:[28 32]"
# → user:1 (Alice Johnson, NYC, age 30)

# Aggregation:
FT.AGGREGATE idx:users "*" 
  GROUPBY 1 @city 
  REDUCE COUNT 0 AS user_count
# → NYC: 2, SF: 1

# Autocomplete / Suggestions:
FT.SUGADD autocomplete "Alice Johnson" 1
FT.SUGADD autocomplete "Alice Cooper" 1
FT.SUGGET autocomplete "Ali"
# → ["Alice Johnson", "Alice Cooper"]
```

### RedisBloom — Probabilistic Data Structures

```bash
# Bloom Filter: "Is this element POSSIBLY in the set?"

# Create bloom filter:
BF.RESERVE seen_urls 0.001 10000000    # 0.1% error rate, 10M capacity

# Add items:
BF.ADD seen_urls "https://example.com/page1"
BF.ADD seen_urls "https://example.com/page2"

# Check membership:
BF.EXISTS seen_urls "https://example.com/page1"    # → 1 (probably yes)
BF.EXISTS seen_urls "https://example.com/page999"  # → 0 (definitely no!)

# Memory: 10M URLs
# HashSet: ~800 MB
# Bloom:   ~17 MB (47x less memory!)

# ═══════════════════════════════════════
#  Top-K: Track top K most frequent items
# ═══════════════════════════════════════

TOPK.RESERVE trending_words 10 50 7 0.925    # Track top 10
TOPK.ADD trending_words "redis" "database" "cache" "redis" "redis"
TOPK.LIST trending_words
# → ["redis", "database", "cache", ...]  (ranked by frequency)

# ═══════════════════════════════════════
#  Count-Min Sketch: Approximate frequency count
# ═══════════════════════════════════════

CMS.INITBYDIM page_views 2000 10
CMS.INCRBY page_views "/home" 1 "/about" 1 "/home" 1
CMS.QUERY page_views "/home"
# → [2]  (approximately)
```

### RedisTimeSeries — Time-Series Data

```bash
# Create time series:
TS.CREATE temperature:sensor:1 RETENTION 86400000 LABELS type temp location office

# Add data points:
TS.ADD temperature:sensor:1 * 22.5       # * = current timestamp
TS.ADD temperature:sensor:1 * 23.1
TS.ADD temperature:sensor:1 * 22.8

# Query range:
TS.RANGE temperature:sensor:1 - + COUNT 10

# Aggregation (average per 5 minutes):
TS.RANGE temperature:sensor:1 - + AGGREGATION avg 300000

# Downsampling rule (auto-aggregate old data):
TS.CREATE temperature:sensor:1:hourly RETENTION 2592000000
TS.CREATERULE temperature:sensor:1 temperature:sensor:1:hourly AGGREGATION avg 3600000
# Raw data → auto-averaged into hourly buckets!

# Query across sensors by label:
TS.MRANGE - + FILTER type=temp location=office
# Returns data from ALL temperature sensors in the office
```

---

## 🔧 Redis Configuration — Production Essentials

```bash
# ═══════════════════════════════════════
#  Key Configuration Parameters
# ═══════════════════════════════════════

# Memory:
maxmemory 4gb                          # Set memory limit
maxmemory-policy allkeys-lfu           # Eviction policy (LFU = smart)

# Persistence:
save 900 1                             # RDB: save if 1 write in 15 min
save 300 10                            # RDB: save if 10 writes in 5 min
appendonly yes                         # Enable AOF
appendfsync everysec                   # AOF sync every second

# Performance:
io-threads 4                           # I/O threads (Redis 6.0+)
io-threads-do-reads yes                # Multi-threaded reads

# Security:
requirepass YOUR_STRONG_PASSWORD       # Require password
rename-command FLUSHALL ""             # Disable dangerous commands!
rename-command CONFIG ""               # Disable CONFIG in production
rename-command KEYS ""                 # Disable KEYS (use SCAN)
protected-mode yes                     # Block external connections without password

# Network:
bind 127.0.0.1 -::1                   # Only local connections
timeout 300                            # Close idle connections after 5 min
tcp-keepalive 60                       # TCP keepalive

# Slow log (find slow commands):
slowlog-log-slower-than 10000          # Log commands > 10ms
slowlog-max-len 128                    # Keep last 128 slow entries
SLOWLOG GET 10                         # View 10 slowest commands
```

---

## 🧪 Hands-On Lab — Build a Real-Time Analytics System

```bash
# ═══════════════════════════════════════
#  Real-Time Dashboard: Track page views, unique visitors,
#  trending pages — all in Redis!
# ═══════════════════════════════════════

# 1. Track page view (counter):
INCR pageviews:/home                # Total views for /home
INCR pageviews:total                # Global total

# 2. Unique visitors (HyperLogLog):
PFADD visitors:2024-01-15 "user:42" "user:100" "user:42"
PFCOUNT visitors:2024-01-15         # → 2 unique visitors

# 3. Trending pages (Sorted Set):
ZINCRBY trending:pages:hourly 1 "/home"
ZINCRBY trending:pages:hourly 1 "/products"
ZINCRBY trending:pages:hourly 1 "/home"
ZREVRANGE trending:pages:hourly 0 4 WITHSCORES  # Top 5 pages

# 4. Real-time user activity (Stream):
XADD activity * user "user:42" page "/home" action "view"
XADD activity * user "user:100" page "/products" action "click"

# 5. Feature flag check:
SET feature:dark_mode "enabled"
GET feature:dark_mode               # → "enabled"

# 6. Rate limiting (Lua script):
EVAL "
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local current = tonumber(redis.call('GET', key) or '0')
  if current < limit then
    redis.call('INCR', key)
    if current == 0 then
      redis.call('EXPIRE', key, 60)
    end
    return 1
  end
  return 0
" 1 rate:api:user:42 100

# 7. Session storage (Hash + TTL):
HSET session:abc123 userId 42 role "admin" ip "1.2.3.4" loginTime "2024-01-15T10:00:00Z"
EXPIRE session:abc123 7200          # 2 hour session

# 8. Real-time notification (Pub/Sub):
PUBLISH notifications:user:42 '{"type":"like","from":"user:100","post":"post:5"}'
```

---

## 📊 Chapter Summary — Quick Reference Card

```
╔══════════════════════════════════════════════════════════════════════╗
║                REDIS ADVANCED CHEAT SHEET                            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PUB/SUB:                                                            ║
║  SUBSCRIBE ch        → Listen to channel                             ║
║  PUBLISH ch msg      → Send to channel (fire-and-forget)             ║
║  PSUBSCRIBE pat:*    → Pattern subscribe                             ║
║  ⚠️ No persistence! Use Streams for reliable messaging              ║
║                                                                      ║
║  STREAMS:                                                            ║
║  XADD stream * k v   → Append event                                 ║
║  XREADGROUP GROUP g c → Consumer group read (load balanced!)        ║
║  XACK stream g id    → Acknowledge processed message                ║
║  XPENDING / XAUTOCLAIM → Handle failed consumers                    ║
║                                                                      ║
║  TRANSACTIONS:                                                       ║
║  MULTI → EXEC        → Atomic batch (no rollback!)                  ║
║  WATCH key           → Optimistic locking (CAS)                     ║
║  ⚠️ Can't read+write in same TX → use Lua!                         ║
║                                                                      ║
║  LUA SCRIPTS:                                                        ║
║  EVAL "script" N key → Atomic server-side logic                     ║
║  EVALSHA sha N key   → Cached script (faster)                       ║
║  Keep scripts < 5ms! Blocking = frozen Redis                         ║
║                                                                      ║
║  PIPELINING:                                                         ║
║  pipe = r.pipeline() → Batch N commands in 1 round trip             ║
║  10-100x throughput improvement!                                     ║
║                                                                      ║
║  DISTRIBUTED LOCK:                                                   ║
║  SET lock:res token NX EX 30 → Acquire with unique token            ║
║  Lua: DEL only if token matches → Safe release                      ║
║  Redlock: Majority across N independent nodes                        ║
║                                                                      ║
║  REDIS MODULES (Redis Stack):                                        ║
║  RediSearch   → Full-text search + secondary indexes                ║
║  RedisJSON    → Native JSON storage & queries                        ║
║  RedisBloom   → Bloom filters, Top-K, Count-Min                     ║
║  RedisTimeSeries → Time-series with auto-downsampling               ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|---|---|
| [3C.4 → Redis Cluster, Sentinel & Persistence](./04-Redis-Cluster.md) | Redis Cluster (sharding), Sentinel (HA), RDB/AOF tuning, production deployment |

---

> 💡 **Key Insight:** Redis isn't just a key-value store. With Lua scripting, Streams, Pub/Sub, and Modules, it's a **programmable data structure server** that can replace multiple infrastructure components — cache, message broker, search engine, time-series DB — all in one.

---

*Final chapter: Let's deploy Redis for production →* [Chapter 3C.4 — Redis Cluster, Sentinel & Persistence](./04-Redis-Cluster.md)
