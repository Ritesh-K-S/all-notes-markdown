# Cache Eviction Policies (LRU, LFU, TTL)

> **What you'll learn**: How caches decide which entries to remove when they're full, the algorithms behind LRU, LFU, FIFO, and TTL-based eviction, how they're implemented internally, and which policy to choose for your use case.

---

## Real-Life Analogy

Your desk can only hold 10 books. You currently have 10 books on it, and you need to add a new one. Which book do you remove?

- **LRU (Least Recently Used)**: Remove the book you haven't touched in the longest time. "If I haven't read it recently, I probably won't need it soon."
- **LFU (Least Frequently Used)**: Remove the book you've opened the fewest times total. "If I rarely use it, it's probably not important."
- **FIFO (First In, First Out)**: Remove the book that's been on your desk the longest. "Oldest goes first."
- **TTL (Time To Live)**: Each book has a sticky note saying "remove after 3 days." When time's up, it goes — regardless of how often you use it.
- **Random**: Close your eyes and pick one. (Surprisingly, not terrible at scale!)

---

## Core Concept Explained Step-by-Step

### Step 1: Why Eviction is Necessary

Cache memory is **finite and expensive**. A 64 GB Redis server costs far more than a 1 TB hard drive. You can only store a fraction of your total data in cache, so when it's full, something must go.

```
Cache is FULL (100% capacity)
     │
     ▼
New item arrives → Need to INSERT
     │
     ▼
┌─────────────────────────────────────┐
│ EVICTION POLICY decides:            │
│ "Which existing item should I       │
│  remove to make room?"              │
└─────────────────────────────────────┘
     │
     ▼
Remove chosen victim → Insert new item
```

### Step 2: The Goal — Maximize Hit Rate

The **best** eviction policy removes items that are **least likely to be requested in the future**. The theoretical optimum is **Bélády's algorithm** (evict the item that won't be used for the longest time) — but this requires knowing the future, which is impossible.

So we use **heuristics** that approximate the optimal:

```
┌──────────────────────────────────────────────────────────────┐
│          EVICTION POLICIES RANKED BY HIT RATE                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Best ──▶  W-TinyLFU (Caffeine)  ≈ Near-optimal              │
│            LFU with aging                                      │
│            LRU (most common)                                   │
│            FIFO                                                │
│  Worst ──▶ Random                                             │
│                                                                │
│  (Actual ranking varies by workload pattern)                  │
└──────────────────────────────────────────────────────────────┘
```

---

## Policy 1: LRU (Least Recently Used)

### Concept

Evict the item that was **accessed longest ago**. Based on the assumption: "If you used it recently, you'll probably use it again soon."

```
Cache state (most recent at top):
┌───────────────────────────┐
│  A  (accessed 1 sec ago)  │ ← Most Recently Used
│  C  (accessed 5 sec ago)  │
│  B  (accessed 30 sec ago) │
│  D  (accessed 2 min ago)  │ ← Least Recently Used → EVICT THIS
└───────────────────────────┘

After evicting D and inserting E:
┌───────────────────────────┐
│  E  (just inserted/used)  │ ← Most Recently Used
│  A  (accessed 1 sec ago)  │
│  C  (accessed 5 sec ago)  │
│  B  (accessed 30 sec ago) │ ← Now the LRU candidate
└───────────────────────────┘
```

### Internal Implementation — HashMap + Doubly Linked List

LRU requires O(1) for both `get` and `put`. This is achieved with:

```
HashMap: key → pointer to node in linked list
Doubly Linked List: ordered by access time (head=MRU, tail=LRU)

┌──────────────────────────────────────────────────────────────┐
│                    LRU CACHE INTERNALS                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  HashMap:                                                      │
│  ┌───────┬──────────┐                                        │
│  │ Key A │ → Node A │──┐                                     │
│  │ Key B │ → Node B │──┼─┐                                   │
│  │ Key C │ → Node C │──┼─┼─┐                                 │
│  └───────┴──────────┘  │ │ │                                 │
│                          │ │ │                                 │
│  Doubly Linked List:    │ │ │                                 │
│  HEAD ↔ [A] ↔ [B] ↔ [C] ↔ TAIL                             │
│  (MRU)                    (LRU)                               │
│                                                                │
│  On access(B): Move B to head                                 │
│  HEAD ↔ [B] ↔ [A] ↔ [C] ↔ TAIL                             │
│                                                                │
│  On eviction: Remove from tail (C)                            │
│  HEAD ↔ [B] ↔ [A] ↔ TAIL                                    │
│                                                                │
│  Both operations: O(1)                                         │
└──────────────────────────────────────────────────────────────┘
```

### Code — Python LRU Cache Implementation

```python
from collections import OrderedDict

class LRUCache:
    """LRU Cache using Python's OrderedDict (maintains insertion order)."""
    
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
    
    def get(self, key):
        """Get item and mark as most recently used."""
        if key not in self.cache:
            return None  # Miss
        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]
    
    def put(self, key, value):
        """Insert item, evicting LRU if at capacity."""
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        
        if len(self.cache) > self.capacity:
            # Remove first item (least recently used)
            self.cache.popitem(last=False)

# Usage
cache = LRUCache(capacity=3)
cache.put("A", 1)  # [A]
cache.put("B", 2)  # [A, B]
cache.put("C", 3)  # [A, B, C]
cache.get("A")     # [B, C, A]  ← A moved to end
cache.put("D", 4)  # [C, A, D]  ← B evicted (was LRU)
```

### Code — Java LRU Cache

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    
    private final int capacity;

    public LRUCache(int capacity) {
        // accessOrder=true makes it maintain access order (LRU)
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // Remove oldest when size exceeds capacity
        return size() > capacity;
    }
}

// Usage
LRUCache<String, String> cache = new LRUCache<>(1000);
cache.put("user:123", userData);
String data = cache.get("user:123");  // Marks as recently used
```

---

## Policy 2: LFU (Least Frequently Used)

### Concept

Evict the item that has been **accessed the fewest times**. "If you barely ever use it, it's probably not important."

```
Cache state (with access counts):
┌────────────────────────────────────┐
│  Key A  │  accessed 100 times      │ ← Keep (popular)
│  Key B  │  accessed 50 times       │ ← Keep
│  Key C  │  accessed 3 times        │ ← EVICT (rarely used)
│  Key D  │  accessed 25 times       │ ← Keep
└────────────────────────────────────┘
```

### The Problem with Pure LFU

```
Problem: "Frequency pollution"

Scenario:
  - Item X was accessed 1000 times last week
  - X hasn't been accessed at all THIS week
  - Item Y was accessed 5 times today (it's new and trending)
  
  Pure LFU keeps X forever (high count) and evicts Y
  But Y is more useful NOW!

Solution: LFU with AGING
  - Periodically halve all frequency counts
  - Or use time-windowed frequency (last N minutes)
  - Caffeine's W-TinyLFU solves this elegantly
```

### Internal Implementation — Min-Heap or Frequency Buckets

```
Frequency-Bucket approach (O(1) eviction):

freq=1: [C, E]       ← Evict from lowest frequency
freq=2: [D]
freq=5: [B]
freq=10: [A]

On access(C):
  Remove C from freq=1
  Add C to freq=2

freq=1: [E]           ← E is now eviction candidate
freq=2: [D, C]
freq=5: [B]
freq=10: [A]
```

### Code — Python LFU Cache

```python
from collections import defaultdict

class LFUCache:
    """LFU Cache with O(1) operations using frequency buckets."""
    
    def __init__(self, capacity):
        self.capacity = capacity
        self.min_freq = 0
        self.key_to_val = {}          # key → value
        self.key_to_freq = {}         # key → frequency
        self.freq_to_keys = defaultdict(OrderedDict)  # freq → {keys} (LRU within same freq)
    
    def get(self, key):
        if key not in self.key_to_val:
            return None
        self._increment_freq(key)
        return self.key_to_val[key]
    
    def put(self, key, value):
        if self.capacity == 0:
            return
        
        if key in self.key_to_val:
            self.key_to_val[key] = value
            self._increment_freq(key)
            return
        
        # Evict if full
        if len(self.key_to_val) >= self.capacity:
            # Remove least frequent (and least recent within that frequency)
            evict_key, _ = self.freq_to_keys[self.min_freq].popitem(last=False)
            del self.key_to_val[evict_key]
            del self.key_to_freq[evict_key]
        
        # Insert new key with frequency 1
        self.key_to_val[key] = value
        self.key_to_freq[key] = 1
        self.freq_to_keys[1][key] = None
        self.min_freq = 1
    
    def _increment_freq(self, key):
        freq = self.key_to_freq[key]
        self.key_to_freq[key] = freq + 1
        del self.freq_to_keys[freq][key]
        
        if not self.freq_to_keys[freq] and freq == self.min_freq:
            self.min_freq += 1
        
        self.freq_to_keys[freq + 1][key] = None
```

---

## Policy 3: TTL (Time To Live)

### Concept

Each entry has a **timer**. When the timer expires, the entry is automatically removed — regardless of access patterns.

```
┌────────────────────────────────────────────┐
│ Key        │ Value    │ TTL    │ Expires At │
├────────────┼──────────┼────────┼────────────┤
│ session:a  │ {...}    │ 1800s  │ 14:30:00   │
│ product:1  │ {...}    │ 600s   │ 14:10:00   │  ← EXPIRED! (it's 14:15)
│ config:app │ {...}    │ 86400s │ tomorrow   │
└────────────┴──────────┴────────┴────────────┘
```

### Two TTL Approaches

```
1. PASSIVE EXPIRATION (Lazy):
   - Check TTL only when key is accessed
   - If expired: delete and return miss
   - Pro: No background work
   - Con: Expired keys linger in memory until accessed

2. ACTIVE EXPIRATION (Eager):
   - Background thread periodically scans for expired keys
   - Proactively removes them
   - Pro: Memory freed promptly
   - Con: CPU cost for scanning

Redis uses BOTH:
  - Passive: check on every access
  - Active: randomly sample 20 keys 10 times/sec, delete expired ones
    If >25% of sample is expired, repeat immediately
```

---

## Policy 4: FIFO (First In, First Out)

### Concept

Evict the **oldest inserted** item, regardless of access pattern.

```
Queue: [A (oldest), B, C, D (newest)]

Insert E → Evict A:
Queue: [B, C, D, E (newest)]
```

Simple but often poor hit rate — a frequently accessed item gets evicted just because it was inserted early.

---

## Policy 5: Random Eviction

### Concept

Pick a random entry and remove it. Surprisingly effective at scale!

```
Why random isn't terrible:
  - O(1) with zero overhead (no tracking recency or frequency)
  - At large cache sizes, probability of evicting a "hot" item is low
  - Redis offers this as "allkeys-random" policy
  - Works well when access is relatively uniform
```

---

## Redis Eviction Policies

Redis offers 8 eviction policies configured via `maxmemory-policy`:

```
┌────────────────────────────────────────────────────────────────────┐
│                   REDIS EVICTION POLICIES                            │
├────────────────────┬───────────────────────────────────────────────┤
│ Policy             │ Behavior                                       │
├────────────────────┼───────────────────────────────────────────────┤
│ noeviction         │ Return error on writes when memory is full     │
│ allkeys-lru        │ LRU across ALL keys                           │
│ allkeys-lfu        │ LFU across ALL keys (Redis 4.0+)             │
│ allkeys-random     │ Random eviction across all keys               │
│ volatile-lru       │ LRU only among keys WITH an expiry set        │
│ volatile-lfu       │ LFU only among keys with an expiry set        │
│ volatile-random    │ Random among keys with an expiry set          │
│ volatile-ttl       │ Remove keys with shortest remaining TTL       │
└────────────────────┴───────────────────────────────────────────────┘

Most common: allkeys-lru (general purpose)
Best for mixed: volatile-lru (preserves permanent keys)
```

### Redis Configuration

```bash
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru

# Redis uses APPROXIMATED LRU (samples 5-10 keys, evicts the oldest)
# This is more memory-efficient than true LRU
maxmemory-samples 10  # More samples = more accurate but slightly slower
```

---

## Comparison Table — All Policies

```
┌──────────────┬────────────┬────────────┬───────────────────┬──────────────────┐
│ Policy       │ Hit Rate   │ Overhead   │ Best For          │ Weakness          │
├──────────────┼────────────┼────────────┼───────────────────┼──────────────────┤
│ LRU          │ Good       │ Low        │ Temporal locality │ Scan pollution    │
│              │            │            │ (recent = useful) │ (one-time scans)  │
├──────────────┼────────────┼────────────┼───────────────────┼──────────────────┤
│ LFU          │ Very Good  │ Medium     │ Frequency matters │ New items starve  │
│              │            │            │ (popular items)   │ (frequency bias)  │
├──────────────┼────────────┼────────────┼───────────────────┼──────────────────┤
│ W-TinyLFU    │ Near-      │ Medium     │ Mixed workloads   │ Complexity        │
│ (Caffeine)   │ Optimal    │            │                   │                    │
├──────────────┼────────────┼────────────┼───────────────────┼──────────────────┤
│ FIFO         │ Fair       │ Very Low   │ Simple needs      │ Ignores access    │
│              │            │            │                   │ patterns          │
├──────────────┼────────────┼────────────┼───────────────────┼──────────────────┤
│ TTL          │ N/A        │ Low        │ Time-sensitive    │ Not capacity-based│
│              │            │            │ data (sessions)   │                    │
├──────────────┼────────────┼────────────┼───────────────────┼──────────────────┤
│ Random       │ Fair       │ Zero       │ Uniform access,   │ Can evict hot     │
│              │            │            │ extreme simplicity │ items randomly    │
└──────────────┴────────────┴────────────┴───────────────────┴──────────────────┘
```

---

## Real-World Example

### How YouTube Uses LRU + LFU Hybrid

YouTube caches video chunks at edge servers. Their challenge:

```
Problem:
  - New viral video: accessed 1M times in 1 hour, then drops
  - Evergreen content: accessed 10K times/day, every day

  Pure LRU: Keeps viral video too long after it stops being popular
  Pure LFU: Viral video needs time to accumulate frequency count

Solution: Segmented LRU + Frequency awareness
  
  ┌────────────────────────────────────────────────────┐
  │  PROBATION segment (20%)  │  PROTECTED segment (80%) │
  ├───────────────────────────┼──────────────────────────┤
  │ New items enter here      │ Items promoted here after  │
  │ Evicted first (LRU)       │ multiple accesses          │
  │                            │ Evicted last               │
  └───────────────────────────┴──────────────────────────┘
  
  New video enters probation → if accessed again → promoted to protected
  One-hit-wonders stay in probation and get evicted quickly
```

---

## The Scan Pollution Problem (LRU's Weakness)

```
Scenario: A batch job scans ALL products (100K items) once

Before scan:
  Cache: [frequently_used_A, frequently_used_B, hot_item_C, ...]
  Hit rate: 95%

After scan (pure LRU):
  Cache: [product_99998, product_99999, product_100000, ...]  ← FULL of scan items!
  Hit rate: 5%  ← CATASTROPHIC!

The scan "polluted" the cache with items that won't be accessed again.

Fix: Use LRU-K (require K accesses before caching) or W-TinyLFU
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using `noeviction` without monitoring | Writes fail when full! | Use `allkeys-lru` unless you have specific needs |
| Not setting `maxmemory` in Redis | Redis grows until OOM kills it | Always set maxmemory in production |
| LRU for scan-heavy workloads | Scan pollution destroys hit rate | Use LFU or W-TinyLFU |
| LFU without aging | Old popular items never evicted | Use time-decayed frequency |
| Too small cache | High eviction rate → low hit rate | Size cache to hold working set |
| Too large cache | Wasted memory, longer GC pauses (Java) | Profile actual usage |

---

## When to Use / When NOT to Use

| Policy | Use When | Don't Use When |
|--------|----------|----------------|
| **LRU** | Temporal locality (recently used = likely reused); general purpose | Heavy batch scans; frequency matters more than recency |
| **LFU** | Access frequency varies widely; want to keep "popular" items | New items need to be cached quickly (cold start problem) |
| **TTL** | Data has natural expiry (sessions, tokens); freshness guarantees | Need capacity-based eviction (TTL alone won't limit size) |
| **FIFO** | Extremely simple needs; all items equally likely to be reused | Access patterns have any locality |
| **Random** | Uniform access; want zero overhead; huge caches | Workload has clear hot/cold items |

---

## Key Takeaways

1. **Eviction** decides what to remove when the cache is full — critical for maintaining a high hit rate.
2. **LRU** (Least Recently Used) is the most common default — good for most workloads with temporal locality.
3. **LFU** (Least Frequently Used) is better when some items are consistently more popular than others.
4. **W-TinyLFU** (Caffeine) combines both and achieves near-optimal hit rates — use it in Java apps.
5. **Redis** uses approximated LRU/LFU (sampling) for memory efficiency — configure with `maxmemory-policy`.
6. **TTL is not a replacement** for capacity-based eviction — use both together.
7. Watch out for **scan pollution** with LRU — a single full table scan can destroy your cache hit rate.

---

## What's Next?

Next, we'll explore the **dangerous failure modes** of caching — Cache Stampede, Thundering Herd, and Cache Penetration. These are the scenarios that can bring down your entire system when caching goes wrong.

→ [09-cache-problems.md](./09-cache-problems.md)
