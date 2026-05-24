# 18 - Rate Limiter

## 📋 Problem Statement

Design a rate limiter that:
- Limits the number of requests a client can make in a time window
- Supports multiple algorithms (Token Bucket, Sliding Window, Fixed Window)
- Can be applied per user, per API, or globally

---

## 📌 Requirements

### Functional Requirements
1. Limit requests per client per time window
2. Support **Token Bucket**, **Sliding Window Log**, **Fixed Window**, **Sliding Window Counter**
3. Return **429 Too Many Requests** when limit exceeded
4. Configurable **rate** and **window size**
5. Support per-user and per-API limits

---

## 💻 Code Implementation

### Rate Limiter Interface

```java
public interface RateLimiter {
    boolean allowRequest(String clientId);
}
```

### 1. Token Bucket Algorithm

```
- Bucket holds tokens (max = capacity)
- Tokens added at a fixed rate
- Each request consumes one token
- Request allowed only if tokens > 0
```

```java
public class TokenBucketLimiter implements RateLimiter {
    private final int capacity;
    private final int refillRate; // tokens per second
    private final Map<String, TokenBucket> buckets;

    public TokenBucketLimiter(int capacity, int refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.buckets = new ConcurrentHashMap<>();
    }

    @Override
    public boolean allowRequest(String clientId) {
        TokenBucket bucket = buckets.computeIfAbsent(clientId,
            k -> new TokenBucket(capacity, refillRate));
        return bucket.tryConsume();
    }

    private static class TokenBucket {
        private int tokens;
        private final int capacity;
        private final int refillRate;
        private long lastRefillTime;

        public TokenBucket(int capacity, int refillRate) {
            this.capacity = capacity;
            this.refillRate = refillRate;
            this.tokens = capacity;
            this.lastRefillTime = System.nanoTime();
        }

        public synchronized boolean tryConsume() {
            refill();
            if (tokens > 0) {
                tokens--;
                return true;
            }
            return false;
        }

        private void refill() {
            long now = System.nanoTime();
            double elapsed = (now - lastRefillTime) / 1_000_000_000.0;
            int newTokens = (int) (elapsed * refillRate);
            if (newTokens > 0) {
                tokens = Math.min(capacity, tokens + newTokens);
                lastRefillTime = now;
            }
        }
    }
}
```

### 2. Fixed Window Counter

```
- Divide time into fixed windows (e.g., 1 minute)
- Count requests in current window
- Reset counter at window boundary
```

```java
public class FixedWindowLimiter implements RateLimiter {
    private final int maxRequests;
    private final long windowSizeMs;
    private final Map<String, WindowCounter> counters;

    public FixedWindowLimiter(int maxRequests, long windowSizeMs) {
        this.maxRequests = maxRequests;
        this.windowSizeMs = windowSizeMs;
        this.counters = new ConcurrentHashMap<>();
    }

    @Override
    public boolean allowRequest(String clientId) {
        WindowCounter counter = counters.computeIfAbsent(clientId,
            k -> new WindowCounter());
        return counter.tryIncrement();
    }

    private class WindowCounter {
        private long windowStart;
        private int count;

        public WindowCounter() {
            this.windowStart = System.currentTimeMillis();
            this.count = 0;
        }

        public synchronized boolean tryIncrement() {
            long now = System.currentTimeMillis();
            if (now - windowStart >= windowSizeMs) {
                // New window
                windowStart = now;
                count = 0;
            }
            if (count < maxRequests) {
                count++;
                return true;
            }
            return false;
        }
    }
}
```

### 3. Sliding Window Log

```
- Store timestamp of each request
- Count requests in last N seconds
- Most accurate but memory-intensive
```

```java
public class SlidingWindowLogLimiter implements RateLimiter {
    private final int maxRequests;
    private final long windowSizeMs;
    private final Map<String, Deque<Long>> requestLogs;

    public SlidingWindowLogLimiter(int maxRequests, long windowSizeMs) {
        this.maxRequests = maxRequests;
        this.windowSizeMs = windowSizeMs;
        this.requestLogs = new ConcurrentHashMap<>();
    }

    @Override
    public boolean allowRequest(String clientId) {
        Deque<Long> log = requestLogs.computeIfAbsent(clientId,
            k -> new ConcurrentLinkedDeque<>());

        long now = System.currentTimeMillis();
        long windowStart = now - windowSizeMs;

        // Remove expired entries
        while (!log.isEmpty() && log.peekFirst() <= windowStart) {
            log.pollFirst();
        }

        if (log.size() < maxRequests) {
            log.addLast(now);
            return true;
        }
        return false;
    }
}
```

### 4. Sliding Window Counter (Hybrid)

```
- Combines fixed window + sliding window
- Uses weighted count from current + previous window
- Good balance of accuracy and memory
```

```java
public class SlidingWindowCounterLimiter implements RateLimiter {
    private final int maxRequests;
    private final long windowSizeMs;
    private final Map<String, SlidingWindow> windows;

    public SlidingWindowCounterLimiter(int maxRequests, long windowSizeMs) {
        this.maxRequests = maxRequests;
        this.windowSizeMs = windowSizeMs;
        this.windows = new ConcurrentHashMap<>();
    }

    @Override
    public boolean allowRequest(String clientId) {
        SlidingWindow window = windows.computeIfAbsent(clientId,
            k -> new SlidingWindow());
        return window.tryIncrement();
    }

    private class SlidingWindow {
        private long currentWindowStart;
        private int currentCount;
        private int previousCount;

        public synchronized boolean tryIncrement() {
            long now = System.currentTimeMillis();
            long currentWindow = now / windowSizeMs;
            long storedWindow = currentWindowStart / windowSizeMs;

            if (currentWindow != storedWindow) {
                previousCount = (currentWindow - storedWindow == 1) ? currentCount : 0;
                currentCount = 0;
                currentWindowStart = now;
            }

            // Weighted count
            double elapsedRatio = (now % windowSizeMs) / (double) windowSizeMs;
            double estimatedCount = previousCount * (1 - elapsedRatio) + currentCount;

            if (estimatedCount < maxRequests) {
                currentCount++;
                return true;
            }
            return false;
        }
    }
}
```

### Comparison Table

```
┌─────────────────────────┬──────────┬──────────┬───────────┐
│ Algorithm               │ Memory   │ Accuracy │ Burst     │
├─────────────────────────┼──────────┼──────────┼───────────┤
│ Token Bucket            │ O(1)     │ Medium   │ Allows    │
│ Fixed Window            │ O(1)     │ Low      │ Allows 2x │
│ Sliding Window Log      │ O(N)     │ High     │ No        │
│ Sliding Window Counter  │ O(1)     │ Medium+  │ No        │
└─────────────────────────┴──────────┴──────────┴───────────┘
```

---

## 🧪 Usage Example

```java
// Allow 100 requests per minute per client
RateLimiter limiter = new TokenBucketLimiter(100, 2); // capacity=100, refill=2/sec

for (int i = 0; i < 110; i++) {
    if (limiter.allowRequest("user-123")) {
        System.out.println("Request " + i + " allowed");
    } else {
        System.out.println("Request " + i + " REJECTED (rate limited)");
    }
}
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Different rate limiting algorithms |
| **Singleton** | Global rate limiter instance |
| **Factory** | Create limiter based on config |

---

## 🔑 Key Takeaways

1. **Token Bucket** is most commonly used (AWS, Stripe)
2. **Fixed Window** has boundary burst problem (2x burst at window edge)
3. **Sliding Window Log** is most accurate but uses most memory
4. **Sliding Window Counter** is the best balance
5. All implementations must be **thread-safe** (synchronized or ConcurrentHashMap)
