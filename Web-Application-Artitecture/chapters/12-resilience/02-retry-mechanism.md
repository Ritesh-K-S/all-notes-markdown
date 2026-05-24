# Retry Mechanism — Try Again, But Smartly

> **What you'll learn**: How to automatically retry failed requests using exponential backoff, jitter, and retry budgets — without overwhelming already-struggling services.

---

## Real-Life Analogy

You call a friend and they don't pick up. What do you do?

- **Naive retry**: You call again immediately. And again. And again. Every second. 50 times. Your friend now has 50 missed calls and thinks you're insane. If they were busy, you just made their phone buzz non-stop.

- **Smart retry**: You wait 1 minute, try again. Then wait 5 minutes, try again. Then wait 30 minutes. If they still don't answer after 3 attempts, you send a text instead (fallback).

That's the difference between **naive retry** (which can kill systems) and **smart retry with exponential backoff** (which gives systems time to recover).

---

## Core Concept Explained Step-by-Step

### The Problem: Transient Failures Are Normal

In distributed systems, temporary failures happen ALL the time:

```
┌─────────────────────────────────────────────────────────────┐
│            WHY REQUESTS FAIL TEMPORARILY                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  • Network blip (packet lost for 100ms)                    │
│  • Server restarting during deployment                     │
│  • Database connection pool temporarily full               │
│  • Garbage collection pause on the server                  │
│  • Load balancer rotating to a new instance                │
│  • Rate limit hit (429 Too Many Requests)                  │
│  • Temporary DNS resolution failure                        │
│                                                             │
│  These are TRANSIENT — they fix themselves in seconds.      │
│  A simple retry often succeeds!                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Naive Retry vs Smart Retry

```
NAIVE RETRY (Immediate, unlimited):

Time: 0s   → Request FAILS
Time: 0s   → Retry 1 FAILS
Time: 0s   → Retry 2 FAILS
Time: 0s   → Retry 3 FAILS
...
Time: 0s   → Retry 100 FAILS

Result: You HAMMERED the already-struggling server
        with 100 extra requests in < 1 second.
        You made the problem WORSE.


SMART RETRY (Exponential backoff):

Time: 0s    → Request FAILS
Time: 1s    → Retry 1 FAILS       (wait 1 second)
Time: 3s    → Retry 2 FAILS       (wait 2 seconds)
Time: 7s    → Retry 3 SUCCEEDS ✓  (wait 4 seconds)

Result: Server had time to recover.
        Only 3 extra requests. System heals.
```

### Exponential Backoff — The Core Algorithm

```
Wait time = base_delay × 2^(attempt_number)

Attempt 0: Request (immediate)
Attempt 1: Wait 1s,  then retry     (1 × 2^0 = 1s)
Attempt 2: Wait 2s,  then retry     (1 × 2^1 = 2s)
Attempt 3: Wait 4s,  then retry     (1 × 2^2 = 4s)
Attempt 4: Wait 8s,  then retry     (1 × 2^3 = 8s)
Attempt 5: Wait 16s, then retry     (1 × 2^4 = 16s)
           ↕ MAX CAP ↕
           Capped at 30s (don't wait too long)
```

### The Thundering Herd Problem & Jitter

Without jitter, all retrying clients back off on the same schedule:

```
WITHOUT JITTER:
   
Thousands of clients all retry at the SAME moments:

Time 0s:  ████████████████  (all fail together)
Time 1s:  ████████████████  (all retry together → OVERLOAD!)
Time 3s:  ████████████████  (all retry together → OVERLOAD!)
Time 7s:  ████████████████  (all retry together → OVERLOAD!)

The retries themselves cause the next failure!


WITH JITTER (random spread):

Time 0s:    ████████████████  (all fail together)
Time 0.5s:  ██                (some retry)
Time 1.2s:  ███               (some retry)
Time 1.8s:  ██                (some retry)
Time 2.5s:  ███               (some retry)

Retries are SPREAD OUT — server can handle them!
```

### Types of Jitter

```
┌────────────────────┬────────────────────────────────────────┐
│ Jitter Type        │ Formula                                │
├────────────────────┼────────────────────────────────────────┤
│ Full Jitter        │ wait = random(0, base × 2^attempt)     │
│ (Recommended)      │ Most spread, least correlated          │
├────────────────────┼────────────────────────────────────────┤
│ Equal Jitter       │ half = (base × 2^attempt) / 2          │
│                    │ wait = half + random(0, half)           │
│                    │ Guarantees minimum wait                 │
├────────────────────┼────────────────────────────────────────┤
│ Decorrelated       │ wait = random(base, prev_wait × 3)     │
│ Jitter             │ Each wait independent of attempt count  │
└────────────────────┴────────────────────────────────────────┘
```

---

## How It Works Internally

### Retry Decision Flow

```
┌──────────────┐
│ Make Request │
└──────┬───────┘
       │
       ▼
┌──────────────┐     Success
│  Got Response?│─────────────────▶ Return Response ✓
└──────┬───────┘
       │ Error/Timeout
       ▼
┌──────────────────────────┐
│  Is error retryable?     │
│  (5xx, timeout, network) │
└──────┬───────────────────┘
       │ Yes          │ No (4xx, auth error)
       ▼              ▼
┌──────────────┐    Return Error ✗
│ Attempts left?│
└──────┬───────┘
       │ Yes          │ No
       ▼              ▼
┌──────────────┐    Return Last Error ✗
│  Wait with   │
│  backoff +   │
│  jitter      │
└──────┬───────┘
       │
       ▼
   Go back to "Make Request"
```

### What's Retryable vs Not Retryable?

```
┌────────────────────────────────────────────────────────────┐
│   RETRYABLE (Transient)        │  NOT RETRYABLE (Permanent)│
├────────────────────────────────┼───────────────────────────┤
│ 500 Internal Server Error      │ 400 Bad Request           │
│ 502 Bad Gateway                │ 401 Unauthorized          │
│ 503 Service Unavailable        │ 403 Forbidden             │
│ 504 Gateway Timeout            │ 404 Not Found             │
│ 429 Too Many Requests          │ 409 Conflict              │
│ Connection refused             │ 422 Validation Error      │
│ Connection reset               │ Business logic errors     │
│ DNS resolution failed          │ Invalid input errors      │
│ Socket timeout                 │                           │
│ SSL handshake timeout          │                           │
└────────────────────────────────┴───────────────────────────┘
```

### Retry Budgets — Don't Retry Too Much

Even with backoff, retries consume resources. A **retry budget** limits the total percentage of requests that can be retries:

```
┌─────────────────────────────────────────────────────────────┐
│                    RETRY BUDGET                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Rule: Retries should be ≤ 20% of total requests           │
│                                                             │
│  If you're making 1000 req/s → max 200 retries/s           │
│                                                             │
│  If retry rate exceeds budget:                              │
│    → Stop retrying (the system is truly down)              │
│    → Trigger circuit breaker (Chapter 12.3)                │
│    → Return error immediately                              │
│                                                             │
│  Why? If everyone retries 3× during an outage:             │
│    1000 req/s → 3000 req/s hitting the server              │
│    That's a 3× traffic amplification!                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Idempotency Requirement

```
⚠️  CRITICAL: Only retry operations that are SAFE to repeat!

SAFE TO RETRY:
  ✓ GET /users/123          (reading data — always safe)
  ✓ PUT /users/123          (idempotent — same result each time)
  ✓ DELETE /orders/456      (deleting twice = same as once)

DANGEROUS TO RETRY:
  ✗ POST /payments          (might charge twice!)
  ✗ POST /orders            (might create duplicate orders!)
  ✗ POST /emails/send       (might send email twice!)

Solution: Use idempotency keys (see Chapter 12.7)
```

---

## Code Examples

### Python — Retry with Exponential Backoff + Jitter

```python
import time
import random
import requests
from requests.exceptions import RequestException

def retry_with_backoff(
    url: str,
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    retryable_status_codes: set = {500, 502, 503, 504, 429}
):
    """Retry a request with exponential backoff and full jitter."""
    
    for attempt in range(max_retries + 1):
        try:
            response = requests.get(url, timeout=(3, 10))
            
            # Check if we should retry based on status code
            if response.status_code in retryable_status_codes:
                if attempt == max_retries:
                    return response  # Last attempt, return what we got
                
                # Special handling for 429: use Retry-After header
                if response.status_code == 429:
                    retry_after = int(response.headers.get("Retry-After", 0))
                    delay = max(retry_after, base_delay)
                else:
                    # Exponential backoff with full jitter
                    exp_delay = min(base_delay * (2 ** attempt), max_delay)
                    delay = random.uniform(0, exp_delay)  # Full jitter
                
                print(f"Attempt {attempt + 1} failed ({response.status_code}). "
                      f"Retrying in {delay:.2f}s...")
                time.sleep(delay)
                continue
            
            return response  # Success or non-retryable error
            
        except RequestException as e:
            if attempt == max_retries:
                raise  # All retries exhausted
            
            exp_delay = min(base_delay * (2 ** attempt), max_delay)
            delay = random.uniform(0, exp_delay)
            print(f"Attempt {attempt + 1} failed ({e}). Retrying in {delay:.2f}s...")
            time.sleep(delay)

# Usage
response = retry_with_backoff("https://api.inventory.com/stock/SKU-123")
```

### Python — Using `tenacity` Library (Production-Grade)

```python
from tenacity import (
    retry, stop_after_attempt, wait_exponential_jitter,
    retry_if_exception_type, before_sleep_log
)
import logging
import requests

logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(4),                    # Max 4 attempts (1 + 3 retries)
    wait=wait_exponential_jitter(                   # Exponential backoff with jitter
        initial=1, max=30, jitter=5                # Start at 1s, max 30s, ±5s jitter
    ),
    retry=retry_if_exception_type(                 # Only retry on these exceptions
        (requests.exceptions.Timeout,
         requests.exceptions.ConnectionError)
    ),
    before_sleep=before_sleep_log(logger, logging.WARNING)  # Log each retry
)
def fetch_user_profile(user_id: str) -> dict:
    """Fetch user profile with automatic retry on transient failures."""
    response = requests.get(
        f"https://api.users.com/profiles/{user_id}",
        timeout=(3, 10)
    )
    response.raise_for_status()  # Raises on 4xx/5xx
    return response.json()
```

### Java — Retry with Exponential Backoff

```java
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.URI;
import java.time.Duration;
import java.util.Set;
import java.util.concurrent.ThreadLocalRandom;

public class RetryableHttpClient {
    
    private static final Set<Integer> RETRYABLE_CODES = Set.of(500, 502, 503, 504, 429);
    private final HttpClient client;
    private final int maxRetries;
    private final long baseDelayMs;
    private final long maxDelayMs;
    
    public RetryableHttpClient(int maxRetries, long baseDelayMs, long maxDelayMs) {
        this.client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(3))
            .build();
        this.maxRetries = maxRetries;
        this.baseDelayMs = baseDelayMs;
        this.maxDelayMs = maxDelayMs;
    }
    
    public HttpResponse<String> executeWithRetry(HttpRequest request) throws Exception {
        Exception lastException = null;
        
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                HttpResponse<String> response = client.send(
                    request, HttpResponse.BodyHandlers.ofString());
                
                if (!RETRYABLE_CODES.contains(response.statusCode())) {
                    return response;  // Success or non-retryable error
                }
                
                if (attempt == maxRetries) return response;
                
                long delay = calculateDelay(attempt);
                System.out.printf("Attempt %d failed (HTTP %d). Retrying in %dms...%n",
                    attempt + 1, response.statusCode(), delay);
                Thread.sleep(delay);
                
            } catch (java.net.http.HttpTimeoutException | java.net.ConnectException e) {
                lastException = e;
                if (attempt == maxRetries) throw e;
                
                long delay = calculateDelay(attempt);
                System.out.printf("Attempt %d failed (%s). Retrying in %dms...%n",
                    attempt + 1, e.getClass().getSimpleName(), delay);
                Thread.sleep(delay);
            }
        }
        throw lastException;
    }
    
    private long calculateDelay(int attempt) {
        // Exponential backoff with full jitter
        long expDelay = Math.min(baseDelayMs * (1L << attempt), maxDelayMs);
        return ThreadLocalRandom.current().nextLong(0, expDelay + 1);
    }
}
```

### Java — Using Resilience4j (Production-Grade)

```java
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;
import io.github.resilience4j.retry.RetryRegistry;
import java.time.Duration;
import java.net.http.HttpResponse;

public class Resilience4jRetryExample {
    
    public static void main(String[] args) {
        // Configure retry with exponential backoff
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(4)                                    // 1 initial + 3 retries
            .waitDuration(Duration.ofSeconds(1))              // Initial wait
            .retryOnResult(response ->                        // Retry on 5xx responses
                ((HttpResponse<?>) response).statusCode() >= 500)
            .retryExceptions(                                 // Retry on these exceptions
                java.net.http.HttpTimeoutException.class,
                java.net.ConnectException.class)
            .ignoreExceptions(                                // Don't retry on these
                IllegalArgumentException.class)
            .intervalFunction(IntervalFunction               // Exponential backoff
                .ofExponentialRandomBackoff(
                    Duration.ofSeconds(1),    // Initial interval
                    2.0,                      // Multiplier
                    Duration.ofSeconds(30)))  // Max interval
            .build();
        
        Retry retry = RetryRegistry.of(config).retry("paymentService");
        
        // Wrap your function with retry
        var decoratedSupplier = Retry.decorateSupplier(retry, 
            () -> callPaymentService("order-123"));
        
        try {
            HttpResponse<String> result = decoratedSupplier.get();
            System.out.println("Success: " + result.body());
        } catch (Exception e) {
            System.err.println("All retries exhausted: " + e.getMessage());
        }
    }
}
```

---

## Infrastructure Examples

### AWS SDK — Built-in Retry Configuration

```python
import boto3
from botocore.config import Config

# AWS SDK has built-in retry with exponential backoff
config = Config(
    retries={
        'max_attempts': 5,        # Total attempts (including first)
        'mode': 'adaptive'        # 'standard' or 'adaptive'
        # 'adaptive' mode adjusts retry behavior based on error type
        # and includes token bucket to limit retries
    }
)

s3_client = boto3.client('s3', config=config)
dynamodb = boto3.resource('dynamodb', config=config)
```

### gRPC — Retry Policy

```json
{
  "methodConfig": [{
    "name": [{"service": "payment.PaymentService"}],
    "retryPolicy": {
      "maxAttempts": 4,
      "initialBackoff": "0.5s",
      "maxBackoff": "30s",
      "backoffMultiplier": 2.0,
      "retryableStatusCodes": [
        "UNAVAILABLE",
        "DEADLINE_EXCEEDED",
        "RESOURCE_EXHAUSTED"
      ]
    }
  }]
}
```

### Envoy Proxy — Retry Configuration

```yaml
# Envoy proxy automatic retry config
routes:
  - match:
      prefix: "/api/v1/orders"
    route:
      cluster: order-service
      retry_policy:
        retry_on: "5xx,reset,connect-failure,retriable-4xx"
        num_retries: 3
        per_try_timeout: 5s
        retry_back_off:
          base_interval: 0.5s
          max_interval: 10s
        retry_budget:
          budget_percent: 20.0      # Max 20% of requests can be retries
          min_retry_concurrency: 3   # Always allow at least 3 concurrent retries
```

### Spring Boot — Retry Configuration

```yaml
# application.yml
resilience4j:
  retry:
    instances:
      paymentService:
        max-attempts: 4
        wait-duration: 1s
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.net.SocketTimeoutException
          - java.net.ConnectException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - org.springframework.web.client.HttpClientErrorException
```

---

## Real-World Example

### Google Cloud's Retry Strategy

Google's client libraries use a sophisticated retry approach:

```
┌─────────────────────────────────────────────────────────────────┐
│           GOOGLE CLOUD RETRY STRATEGY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Initial delay:    100ms                                        │
│  Delay multiplier: 1.3                                          │
│  Max delay:        60 seconds                                   │
│  Total timeout:    600 seconds (10 minutes)                     │
│  Jitter:           ±20% of calculated delay                     │
│                                                                 │
│  Timeline:                                                      │
│  ─────────────────────────────────────────────────────          │
│  0ms    → First attempt                                         │
│  100ms  → Retry 1  (100ms × 1.3^0 = 100ms)                    │
│  230ms  → Retry 2  (100ms × 1.3^1 = 130ms later)              │
│  399ms  → Retry 3  (100ms × 1.3^2 = 169ms later)              │
│  619ms  → Retry 4  (100ms × 1.3^3 = 220ms later)              │
│  ...and so on until total timeout                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Uber's Retry Architecture

```
                    ┌──────────────┐
                    │   Rider App  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  API Gateway │  ← Retry: 2 attempts, 500ms backoff
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │   Trip   │ │ Pricing  │ │  Driver  │
        │ Service  │ │ Service  │ │ Matching │
        └────┬─────┘ └──────────┘ └────┬─────┘
             │                          │
             │  Retry: 3 attempts       │  Retry: 1 attempt
             │  1s backoff              │  200ms backoff
             ▼                          ▼
        ┌──────────┐              ┌──────────┐
        │  Database │              │  Redis   │
        │ (Cassandra)│              │  Cache   │
        └──────────┘              └──────────┘

Key insight: Different services have different retry
strategies based on their latency profiles and criticality.
```

---

## Common Mistakes / Pitfalls

### 1. Retrying Non-Idempotent Requests

```
❌ DANGEROUS:
   POST /api/payments/charge (amount=100)
   → Timeout
   → Retry POST /api/payments/charge (amount=100)
   → Customer charged $200!

✅ SAFE:
   POST /api/payments/charge
   Headers: Idempotency-Key: "order-123-attempt-1"
   → Timeout
   → Retry with SAME idempotency key
   → Server says "already processed" → charges $100 only once
```

### 2. No Maximum Retry Limit

```
❌ BAD: Retry forever (or 100 times)
   → Wastes resources for hours during extended outages

✅ GOOD: Max 3-5 retries, then fail fast
   → Return error to user quickly
   → Let circuit breaker handle the outage
```

### 3. Retrying Without Backoff

```
❌ BAD: Retry immediately (0ms delay)
   → 1000 clients × 3 retries = 3000 req/s hitting a dying server
   → The retries PREVENT the server from recovering

✅ GOOD: Exponential backoff + jitter
   → Spread load over time
   → Give server breathing room to recover
```

### 4. Retry Amplification in Microservices

```
❌ DANGEROUS CASCADE:
   A → B → C → D

   If D fails:
   - C retries D 3 times
   - B retries C 3 times (each triggers C's 3 retries to D)
   - A retries B 3 times (each triggers the whole chain)
   
   Total calls to D: 3 × 3 × 3 = 27 requests!
   
   With deeper chains, this grows EXPONENTIALLY.

✅ SOLUTION:
   - Only retry at ONE layer (usually the closest to the failure)
   - Use retry budgets to cap total retry traffic
   - Use circuit breakers to stop retrying entirely
```

### 5. Not Respecting Retry-After Header

```python
# ❌ Ignoring the server's guidance
response = requests.get(url)
if response.status_code == 429:
    time.sleep(1)  # Random guess

# ✅ Respecting Retry-After header
response = requests.get(url)
if response.status_code == 429:
    retry_after = int(response.headers.get("Retry-After", 5))
    time.sleep(retry_after)  # Wait as long as server asked
```

---

## When to Use / When NOT to Use

### ✅ Use Retries When

| Scenario | Example |
|----------|---------|
| Transient network errors | Connection reset, timeout |
| Server overload (5xx) | 502, 503, 504 responses |
| Rate limiting (429) | With Retry-After header |
| Database connection pool full | Temporary resource exhaustion |
| DNS resolution blip | Momentary lookup failure |
| During rolling deployments | Some instances restarting |

### ❌ Do NOT Retry When

| Scenario | Why |
|----------|-----|
| Client error (4xx) | Request is wrong, retrying won't fix it |
| Authentication failure (401) | Need new credentials, not retry |
| Request validation error (422) | Fix the request payload instead |
| Business logic rejection | "Insufficient funds" won't change |
| Non-idempotent operations | Without idempotency keys |
| Circuit is OPEN | Service is confirmed down |

---

## Key Takeaways

- **Always use exponential backoff** — never retry immediately. Give the failing system time to recover.
- **Add jitter (randomness)** — prevents the "thundering herd" where all clients retry at the same moment.
- **Only retry transient errors** — 5xx and timeouts are retryable; 4xx errors are not.
- **Set a max retry limit** — 3-5 retries is typical. After that, fail fast or use a fallback.
- **Use retry budgets** — cap retries at 10-20% of total traffic to prevent retry storms.
- **Be careful with retries in service chains** — retry amplification can multiply load exponentially.
- **Non-idempotent operations need idempotency keys** — never blindly retry a payment or order creation.

---

## What's Next?

Retries help when a service has a *temporary* hiccup. But what if a service is down for minutes or hours? Retrying repeatedly is wasteful and harmful. That's where the **Circuit Breaker Pattern** comes in — it detects when a service is truly down and stops all calls to it. See [03-circuit-breaker.md](./03-circuit-breaker.md).
