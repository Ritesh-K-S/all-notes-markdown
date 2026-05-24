# Distributed Locking (Redlock, ZooKeeper)

> **What you'll learn**: How to ensure only one process across multiple machines can access a shared resource at a time — the algorithms, their guarantees, their flaws, and when to use each approach.

---

## Real-Life Analogy: The Bathroom Key

Imagine an office with a single bathroom and 50 employees. There's ONE key hanging on a hook by the door:

- **Take the key** = acquire the lock
- **Use the bathroom** = access the shared resource
- **Return the key** = release the lock

Simple! But in a distributed system:
- What if someone takes the key and NEVER comes back (crashes)?
- What if two people both grab for the key at the same instant?
- What if the hook is on a different floor (different server) and the elevator is broken (network partition)?
- What if someone photocopied the key? (stale lock)

**Distributed locking** solves these problems with timeouts (auto-expire if you don't return it), atomic operations (only one person can grab it), and fencing tokens (the copied key is detected as outdated).

---

## Core Concept Explained Step-by-Step

### Why Do We Need Distributed Locks?

```
SCENARIO: Two payment processors, one user
═══════════════════════════════════════════════════

Without lock:
  Server A: Read balance = $100
  Server B: Read balance = $100    (both see $100!)
  Server A: Deduct $80 → Write balance = $20
  Server B: Deduct $80 → Write balance = $20   ← WRONG!
  
  User spent $160 from a $100 account! 💀

With distributed lock:
  Server A: LOCK("user:123:balance") → acquired! ✅
  Server A: Read balance = $100
  Server A: Deduct $80 → Write balance = $20
  Server A: UNLOCK("user:123:balance")
  
  Server B: LOCK("user:123:balance") → acquired! ✅
  Server B: Read balance = $20
  Server B: Deduct $80 → INSUFFICIENT FUNDS ❌
  Server B: UNLOCK("user:123:balance")
  
  Correct behavior! ✅
```

### Lock Requirements

```
PROPERTIES OF A GOOD DISTRIBUTED LOCK:
═══════════════════════════════════════════════════

1. MUTUAL EXCLUSION
   Only ONE client holds the lock at any time.
   
2. DEADLOCK-FREE
   If the lock holder crashes, the lock is eventually released.
   (Usually via TTL/timeout)
   
3. FAULT-TOLERANT
   Lock service continues working if some nodes fail.
   
4. OWNER-ONLY RELEASE
   Only the client that acquired the lock can release it.
   (Prevent accidental release by others)
```

---

## Approach 1: Single Redis Lock (Simple but Risky)

```
SIMPLE REDIS LOCK:
═══════════════════════════════════════════════════

ACQUIRE:
  SET lock:resource "owner-id" NX PX 30000
  │                  │         │  │
  │                  │         │  └── Expires in 30 seconds (deadlock prevention)
  │                  │         └── Only set if NOT EXISTS (atomicity)
  │                  └── Unique owner identifier
  └── Key name

  If returns OK → Lock acquired! ✅
  If returns nil → Someone else has it ❌ (retry later)

RELEASE:
  -- Only release if we own it (Lua script for atomicity)
  if redis.call("GET", "lock:resource") == "owner-id" then
      return redis.call("DEL", "lock:resource")
  end

PROBLEM: Single Redis instance = Single Point of Failure!
  If Redis crashes → all locks are lost → mutual exclusion broken
```

### The Auto-Expiry Problem

```
THE DANGEROUS SCENARIO:
═══════════════════════════════════════════════════

Time 0s: Client A acquires lock (TTL = 30s)
Time 0s: Client A starts long operation...
Time 29s: Client A is STILL working (GC pause, slow I/O)
Time 30s: Lock EXPIRES! (TTL reached)
Time 30s: Client B acquires same lock!
Time 31s: Client A finishes, writes result ← UNSAFE!
          (A thinks it still holds the lock!)

Both A and B operated on the resource simultaneously! 💀

SOLUTION: Fencing tokens (see below)
```

---

## Approach 2: Redlock (Multi-Node Redis Lock)

Proposed by Redis creator Salvatore Sanfilippo for fault tolerance:

```
REDLOCK ALGORITHM:
═══════════════════════════════════════════════════

Use 5 INDEPENDENT Redis instances (not replicas):

  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │ Redis 1 │  │ Redis 2 │  │ Redis 3 │  │ Redis 4 │  │ Redis 5 │
  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘
       │            │            │            │            │
       ▼            ▼            ▼            ▼            ▼
  Independent machines, independent failures

ACQUIRE LOCK:
─────────────────────────────────────────────────
1. Get current time (T1)
2. Try to acquire lock on ALL 5 instances (with short timeout each)
3. Lock acquired if:
   a. Got lock on MAJORITY (3+ out of 5)
   b. Total time elapsed < lock TTL
4. Effective lock time = TTL - elapsed time
5. If failed → release lock on all instances

EXAMPLE:
  Client tries to lock "resource:X" with TTL=30s
  
  Redis 1: SET lock:X "client-A" NX PX 30000 → OK ✅ (2ms)
  Redis 2: SET lock:X "client-A" NX PX 30000 → OK ✅ (3ms)
  Redis 3: SET lock:X "client-A" NX PX 30000 → OK ✅ (2ms)
  Redis 4: SET lock:X "client-A" NX PX 30000 → TIMEOUT ❌ (5ms)
  Redis 5: SET lock:X "client-A" NX PX 30000 → OK ✅ (2ms)
  
  Got 4/5 (≥ majority of 3) ✅
  Elapsed: 14ms
  Effective TTL: 30000 - 14 = 29986ms ✅ (still valid)
  
  LOCK ACQUIRED! ✅
```

### Redlock Controversy

```
THE MARTIN KLEPPMANN vs ANTIREZ DEBATE (2016):
═══════════════════════════════════════════════════

Martin Kleppmann (distributed systems researcher) argued Redlock is UNSAFE:

Problem 1: Clock Drift
  If Redis node 3's clock jumps forward → lock expires early
  Another client acquires it → two clients hold "the lock"!

Problem 2: Process Pauses
  Client A gets lock → GC pause for 30 seconds → lock expires
  Client B gets lock → Client A resumes → BOTH think they have it

Problem 3: Network Delays
  Lock acquired at T=0, response delayed until T=29
  Effective TTL = 1 second! (Almost no safety margin)

Kleppmann's Solution: USE FENCING TOKENS (see below)

Antirez (Redis creator) disagreed on some points,
but acknowledged: Redlock is for EFFICIENCY (preventing
duplicate work), not for CORRECTNESS (preventing data corruption).

CONCLUSION:
  ┌───────────────────────────────────────────────────────────┐
  │ If a double-execution is "wasteful" → Redlock is fine    │
  │ If a double-execution is "catastrophic" → Use ZooKeeper  │
  │ + fencing tokens                                          │
  └───────────────────────────────────────────────────────────┘
```

---

## Approach 3: ZooKeeper Distributed Locks

```
ZOOKEEPER LOCK (strongest guarantee):
═══════════════════════════════════════════════════

ZooKeeper provides:
1. Linearizable writes (consensus-based)
2. Ephemeral nodes (auto-deleted on disconnect)
3. Sequential nodes (ordered)
4. Watches (notifications)

ALGORITHM:
─────────────────────────────────────────────────
1. Client creates EPHEMERAL SEQUENTIAL node:
   /locks/resource/__lock-0000000023

2. Client gets all children of /locks/resource/
   → [__lock-0000000021, __lock-0000000022, __lock-0000000023]

3. If MY node is the SMALLEST → I have the lock! ✅

4. If NOT smallest → watch the node JUST before me
   (e.g., I watch __lock-0000000022)

5. When watched node is deleted → check again
   (maybe now I'M the smallest!)

RELEASE: Delete my ephemeral node
CRASH: Ephemeral node auto-deleted → next in line gets lock

VISUALIZATION:
   /locks/payment-user123/
   ├── __lock-0000000021  (Client A) ← LOCK HOLDER
   ├── __lock-0000000022  (Client B) ← waiting, watches 021
   └── __lock-0000000023  (Client C) ← waiting, watches 022
   
   Client A finishes → deletes 021
   Client B wakes up → is now smallest → ACQUIRES LOCK ✅
```

---

## Fencing Tokens: The Ultimate Safety Net

```
FENCING TOKENS:
═══════════════════════════════════════════════════

Every time a lock is acquired, a MONOTONICALLY INCREASING
token is issued.

Lock acquire #1: token = 34
Lock acquire #2: token = 35
Lock acquire #3: token = 36

The resource (database, file, etc.) REJECTS operations
with an old token:

Timeline:
─────────────────────────────────────────────────
Client A: acquires lock (token=34)
Client A: GC pause (lock expires!)
Client B: acquires lock (token=35)
Client B: writes to DB with token=35 → DB accepts ✅
Client A: wakes up, writes to DB with token=34
          → DB says "I've seen token 35 already. 
             Token 34 is STALE. REJECTED!" ❌

The resource validates tokens, not the lock service!
This is the ONLY truly safe approach.

   Client ──── write(data, token=35) ────▶ Storage
                                               │
                                          Check: is 35 ≥ 
                                          last seen token?
                                               │
                                          YES → accept ✅
                                          NO → reject ❌
```

---

## Code Examples

### Python: Redis Distributed Lock with Fencing

```python
import redis
import uuid
import time
from typing import Optional, Tuple

class DistributedLock:
    """Redis distributed lock with fencing token support."""
    
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.owner_id = str(uuid.uuid4())  # Unique per client
    
    def acquire(self, resource: str, ttl_ms: int = 30000, 
                retry_count: int = 3, retry_delay_ms: int = 200
                ) -> Tuple[bool, Optional[int]]:
        """
        Acquire lock. Returns (success, fencing_token).
        Fencing token is a monotonically increasing integer.
        """
        lock_key = f"lock:{resource}"
        token_key = f"lock_token:{resource}"
        
        for attempt in range(retry_count):
            # Atomic: set lock + increment fencing token
            pipe = self.redis.pipeline(True)
            try:
                # Try to acquire
                acquired = self.redis.set(
                    lock_key, self.owner_id, 
                    nx=True,  # Only if not exists
                    px=ttl_ms  # Expire after TTL
                )
                
                if acquired:
                    # Get monotonically increasing fencing token
                    token = self.redis.incr(token_key)
                    print(f"[{self.owner_id[:8]}] Lock acquired! "
                          f"Token: {token}, TTL: {ttl_ms}ms")
                    return True, token
                    
            except redis.WatchError:
                pass  # Retry
            
            # Wait before retry
            time.sleep(retry_delay_ms / 1000)
        
        print(f"[{self.owner_id[:8]}] Failed to acquire lock after {retry_count} attempts")
        return False, None
    
    def release(self, resource: str) -> bool:
        """Release lock only if we own it (atomic check-and-delete)."""
        lock_key = f"lock:{resource}"
        
        # Lua script ensures atomicity of check + delete
        lua_script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """
        
        result = self.redis.eval(lua_script, 1, lock_key, self.owner_id)
        
        if result == 1:
            print(f"[{self.owner_id[:8]}] Lock released ✅")
            return True
        else:
            print(f"[{self.owner_id[:8]}] Lock release failed (not owner or expired)")
            return False
    
    def extend(self, resource: str, additional_ttl_ms: int) -> bool:
        """Extend lock TTL if we still own it."""
        lock_key = f"lock:{resource}"
        
        lua_script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("PEXPIRE", KEYS[1], ARGV[2])
        else
            return 0
        end
        """
        
        result = self.redis.eval(
            lua_script, 1, lock_key, self.owner_id, additional_ttl_ms
        )
        return result == 1


class FencedResource:
    """A resource that validates fencing tokens to prevent stale writes."""
    
    def __init__(self):
        self.data = {}
        self.last_token = 0  # Highest token seen
    
    def write(self, key: str, value: str, fencing_token: int) -> bool:
        """Only accept writes with token >= last seen token."""
        if fencing_token < self.last_token:
            print(f"  ❌ REJECTED write ({key}={value}): "
                  f"token {fencing_token} < last seen {self.last_token}")
            return False
        
        self.last_token = fencing_token
        self.data[key] = value
        print(f"  ✅ ACCEPTED write ({key}={value}): token {fencing_token}")
        return True


# --- Usage Demo ---
# r = redis.Redis(host='localhost', port=6379)
# lock = DistributedLock(r)
# 
# success, token = lock.acquire("payment:user123")
# if success:
#     resource.write("balance", "500", token)
#     lock.release("payment:user123")
```

### Java: ZooKeeper Distributed Lock

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Demonstrates ZooKeeper-style distributed lock.
 * In production, use Apache Curator's InterProcessMutex.
 */
public class ZooKeeperLock {
    
    // Simulates ZooKeeper's coordination service
    static class ZKSimulator {
        private final TreeMap<String, String> nodes = new TreeMap<>();
        private final ReentrantLock internalLock = new ReentrantLock();
        private int sequenceCounter = 0;
        
        public String createEphemeralSequential(String path, String data) {
            internalLock.lock();
            try {
                String fullPath = path + String.format("%010d", sequenceCounter++);
                nodes.put(fullPath, data);
                return fullPath;
            } finally {
                internalLock.unlock();
            }
        }
        
        public List<String> getChildren(String parentPath) {
            internalLock.lock();
            try {
                return nodes.keySet().stream()
                        .filter(k -> k.startsWith(parentPath))
                        .sorted()
                        .toList();
            } finally {
                internalLock.unlock();
            }
        }
        
        public void delete(String path) {
            internalLock.lock();
            try {
                nodes.remove(path);
            } finally {
                internalLock.unlock();
            }
        }
    }
    
    static class DistributedMutex {
        private final ZKSimulator zk;
        private final String lockPath;
        private final String clientId;
        private String myNode;
        private long fencingToken;
        private static long globalTokenCounter = 0;
        
        public DistributedMutex(ZKSimulator zk, String resource, String clientId) {
            this.zk = zk;
            this.lockPath = "/locks/" + resource + "/";
            this.clientId = clientId;
        }
        
        public long acquire() throws InterruptedException {
            // Create ephemeral sequential node
            myNode = zk.createEphemeralSequential(lockPath + "lock-", clientId);
            
            while (true) {
                List<String> children = zk.getChildren(lockPath);
                
                if (children.isEmpty()) {
                    throw new IllegalStateException("Lock node disappeared!");
                }
                
                // Am I the smallest (first in sorted order)?
                if (children.get(0).equals(myNode)) {
                    // YES! I have the lock!
                    fencingToken = ++globalTokenCounter;
                    System.out.printf("[%s] 🔒 Lock ACQUIRED (token=%d)%n", 
                            clientId, fencingToken);
                    return fencingToken;
                }
                
                // NO — wait for predecessor to be deleted
                // (In real ZK: set a watch on predecessor)
                System.out.printf("[%s] Waiting for lock... (position %d)%n",
                        clientId, children.indexOf(myNode) + 1);
                Thread.sleep(100); // Simulate watch wait
            }
        }
        
        public void release() {
            if (myNode != null) {
                zk.delete(myNode);
                System.out.printf("[%s] 🔓 Lock RELEASED%n", clientId);
                myNode = null;
            }
        }
        
        public long getFencingToken() { return fencingToken; }
    }
    
    public static void main(String[] args) throws Exception {
        ZKSimulator zk = new ZKSimulator();
        
        // Simulate 3 clients competing for same lock
        ExecutorService executor = Executors.newFixedThreadPool(3);
        CountDownLatch startLatch = new CountDownLatch(1);
        
        for (int i = 1; i <= 3; i++) {
            final String clientId = "Client-" + i;
            executor.submit(() -> {
                try {
                    startLatch.await(); // All start at same time
                    DistributedMutex lock = new DistributedMutex(zk, "shared-resource", clientId);
                    
                    long token = lock.acquire();
                    
                    // Critical section
                    System.out.printf("[%s] Working with resource (token=%d)...%n", 
                            clientId, token);
                    Thread.sleep(200); // Simulate work
                    
                    lock.release();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
        }
        
        startLatch.countDown(); // Start all clients
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
    }
}
```

---

## Infrastructure Examples

### Redis + Redisson (Java)

```java
// Production Redis lock using Redisson library
Config config = new Config();
config.useSingleServer().setAddress("redis://127.0.0.1:6379");
RedissonClient redisson = Redisson.create(config);

RLock lock = redisson.getLock("order:12345");

try {
    // Wait up to 10 seconds to acquire, auto-release after 30 seconds
    boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
    
    if (acquired) {
        // Critical section — only one instance executes this
        processPayment(orderId);
    }
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### Consul Distributed Lock

```
CONSUL LOCK (Session-based):
═══════════════════════════════════════════════════

1. Create a session with TTL:
   PUT /v1/session/create
   {"TTL": "30s", "Behavior": "delete"}

2. Acquire lock (KV with session):
   PUT /v1/kv/locks/my-resource?acquire=<session-id>
   
   Returns true if acquired, false if held by another session

3. Release:
   PUT /v1/kv/locks/my-resource?release=<session-id>

4. If holder dies → session expires → lock auto-released

Architecture:
  ┌──────────────┐
  │ Consul Cluster│ (Raft consensus — CP system)
  │  (3-5 nodes)  │
  └──────┬───────┘
         │
    ┌────┴────┬──────────┐
    ▼         ▼          ▼
 Service A  Service B  Service C
 (acquires)  (waits)   (waits)
```

---

## Real-World Example

### Google Chubby: The Original Distributed Lock Service

```
GOOGLE CHUBBY:
═══════════════════════════════════════════════════

Used by nearly every Google service for:
- GFS master election
- Bigtable tablet assignment
- MapReduce job coordination

Design:
- 5 replicas using Paxos consensus
- Coarse-grained locks (held for hours/days, not milliseconds)
- Lock with "sequencer" (= fencing token)
- Client-side caching with invalidation

Chubby lock flow:
  Client → Chubby Cell (5 nodes, Paxos) → Lock acquired + sequencer
  Client → Resource Server: "Write X (sequencer=42)"
  Resource Server: "Sequencer 42 ≥ last seen (41)? YES → Accept"
  
Key lesson from Google:
  "We found that programmers make more errors using
   lock-free approaches than using locks with good
   implementations." — The Chubby Paper (2006)
```

### Shopify: Distributed Locks for Flash Sales

```
SHOPIFY FLASH SALE SCENARIO:
═══════════════════════════════════════════════════

10,000 people try to buy 100 limited-edition items simultaneously.

Without lock: 500+ orders placed for 100 items (oversell!)
With lock: Exactly 100 orders succeed.

Approach:
  For each item:
    lock = redis.lock("item:SHOE-001", ttl=5s)
    if lock.acquired:
        stock = db.get_stock("SHOE-001")
        if stock > 0:
            db.decrement_stock("SHOE-001")
            create_order()
        lock.release()

Optimization: Use Redis WATCH + MULTI for item-level atomicity
              (locks per-item, not global lock)

Scale: 100,000+ lock operations/second on Redis Cluster
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Lock without TTL | If client crashes, lock held forever (deadlock) | Always set a TTL. Choose TTL > max expected operation time. |
| Releasing a lock you don't own | Client B accidentally releases Client A's lock | Use unique owner ID + Lua script for atomic check-and-delete |
| Trusting lock without fencing | After GC pause, client may use expired lock | Always use fencing tokens for critical operations |
| Single Redis for critical locks | Redis crash = all locks lost | Use Redlock (multi-node) or ZooKeeper for critical paths |
| Lock TTL too short | Operation takes longer than TTL → lock expires mid-operation | Set generous TTL + implement lock extension (renewal) |
| Locking too broadly | Locking entire table instead of a single row | Lock at the finest granularity possible (per-resource ID) |

---

## Comparison: Redis vs ZooKeeper vs etcd

```
┌──────────────┬────────────────┬────────────────┬────────────────┐
│              │    Redis       │   ZooKeeper    │     etcd       │
├──────────────┼────────────────┼────────────────┼────────────────┤
│ Consensus    │ None (single)  │ ZAB            │ Raft           │
│              │ or Redlock     │ (consensus)    │ (consensus)    │
├──────────────┼────────────────┼────────────────┼────────────────┤
│ Safety       │ Best-effort    │ Strong         │ Strong         │
│ guarantee    │ (clock issues) │ (linearizable) │ (linearizable) │
├──────────────┼────────────────┼────────────────┼────────────────┤
│ Performance  │ Fastest        │ Moderate       │ Moderate       │
│              │ (~0.1ms)       │ (~5-10ms)      │ (~5-10ms)      │
├──────────────┼────────────────┼────────────────┼────────────────┤
│ Auto-release │ TTL expiry     │ Ephemeral nodes│ Lease expiry   │
├──────────────┼────────────────┼────────────────┼────────────────┤
│ Use case     │ Efficiency     │ Correctness    │ Correctness    │
│              │ locks (perf)   │ locks (safety) │ locks (safety) │
├──────────────┼────────────────┼────────────────┼────────────────┤
│ K8s native?  │ No             │ No             │ YES (built-in) │
└──────────────┴────────────────┴────────────────┴────────────────┘
```

---

## When to Use / When NOT to Use

### Use Distributed Locks When:
- ✅ Preventing duplicate processing (idempotency via locking)
- ✅ Exclusive access to external resources (file, API rate limit)
- ✅ Preventing overselling in e-commerce
- ✅ Distributed cron (only one instance runs the job)
- ✅ Database migrations (only one instance migrates)

### Do NOT Use When:
- ❌ You can redesign to avoid shared state (preferred!)
- ❌ Database-level locking suffices (SELECT FOR UPDATE)
- ❌ Idempotent operations make locks unnecessary
- ❌ The "double execution" consequence is merely wasteful (use a simple Redis lock, not full correctness)
- ❌ High-contention scenarios where locks become bottlenecks (use queues instead)

---

## Key Takeaways

- 🔑 **Distributed locks** ensure mutual exclusion across machines — only one process accesses a resource at a time.
- 🔑 **Always use TTL** (auto-expiry) to prevent deadlocks from crashed clients.
- 🔑 **Single Redis lock** is fast but not safe against Redis failures. Good for efficiency, not correctness.
- 🔑 **Redlock** (multi-Redis) improves fault tolerance but is still vulnerable to clock issues and process pauses.
- 🔑 **ZooKeeper/etcd** provide the strongest guarantees via consensus — use for correctness-critical locks.
- 🔑 **Fencing tokens** are the ONLY truly safe mechanism — the resource itself validates that the lock is still current.
- 🔑 **Prefer avoiding locks** when possible — idempotent operations, queue-based processing, or database-level atomicity are often simpler and more reliable.

---

## What's Next?

Locks help prevent conflicts, but how do we even KNOW if two events conflict? How do we determine which happened first when there's no global clock? The next chapter covers **[Vector Clocks & Conflict Resolution](./09-vector-clocks.md)** — the mechanism for tracking causality and detecting conflicts in distributed systems.
