# Circuit Breaker Pattern — Stop Calling a Dead Service

> **What you'll learn**: How the Circuit Breaker pattern detects failing services, stops wasting resources on doomed requests, and automatically recovers when the service comes back — just like an electrical circuit breaker protects your home.

---

## Real-Life Analogy

Think of the **electrical circuit breaker** in your home's fuse box. When there's a short circuit or power surge:

1. **Normal state** — Electricity flows freely. Everything works.
2. **Overload detected** — Too much current flowing (a fault).
3. **Circuit TRIPS** — The breaker flips OFF. No more electricity flows to that circuit. This *protects* your house from burning down.
4. **Manual reset** — After you fix the problem, you flip the breaker back ON.

In software, a **Circuit Breaker** works the same way:
- It monitors calls to a downstream service.
- If too many calls fail, it **trips** (opens the circuit).
- While open, it **immediately rejects** all calls without even trying — no waiting, no wasted resources.
- After a cooldown period, it **tentatively tries** again to see if the service recovered.

---

## Core Concept Explained Step-by-Step

### The Three States

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CIRCUIT BREAKER STATES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────┐         failures          ┌──────────┐              │
│   │  CLOSED  │ ─────── exceed ──────────▶ │   OPEN   │              │
│   │          │         threshold          │          │              │
│   │ (Normal) │                            │ (Broken) │              │
│   │ Requests │                            │ Requests │              │
│   │ flow     │                            │ rejected │              │
│   │ through  │◀────── success ──────────  │ immediately│            │
│   └──────────┘                            └────┬─────┘              │
│        ▲                                       │                    │
│        │                                       │ after timeout      │
│        │                                       ▼                    │
│        │          success            ┌────────────────┐             │
│        └──────────────────────────── │  HALF-OPEN     │             │
│                                      │                │             │
│                                      │ Allow LIMITED  │             │
│              failure ──────────────▶ │ test requests  │             │
│              (back to OPEN)          └────────────────┘             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### State Details

```
STATE: CLOSED (Normal operation)
┌─────────────────────────────────────────┐
│  • All requests pass through normally    │
│  • Failures are COUNTED                  │
│  • Success resets the failure count      │
│  • When failures > threshold → OPEN      │
│                                          │
│  Example: 5 failures in 10 seconds       │
│           triggers state change to OPEN   │
└─────────────────────────────────────────┘

STATE: OPEN (Service is considered DOWN)
┌─────────────────────────────────────────┐
│  • ALL requests are REJECTED immediately │
│  • No network call is made at all        │
│  • Returns error/fallback instantly      │
│  • A timer runs (e.g., 30 seconds)       │
│  • After timer expires → HALF-OPEN       │
│                                          │
│  Benefits:                               │
│  - No wasted threads/connections         │
│  - Fast failure (user gets error in <1ms)│
│  - Gives downstream time to recover      │
└─────────────────────────────────────────┘

STATE: HALF-OPEN (Testing if service recovered)
┌─────────────────────────────────────────┐
│  • Allow a LIMITED number of requests    │
│    (e.g., just 1 or 3 test requests)    │
│  • If test request SUCCEEDS → CLOSED    │
│  • If test request FAILS → OPEN again   │
│                                          │
│  Think of it as "tentatively checking    │
│  if the service is back"                 │
└─────────────────────────────────────────┘
```

### Timeline Example

```
Time →   0s    5s    10s   15s   20s   25s   30s   35s   40s   45s
         │     │     │     │     │     │     │     │     │     │
State:   CLOSED                  OPEN                   HALF   CLOSED
         │     │     │     │     │     │     │     │     │OPEN │
         │     │     │     │     │     │     │     │     │     │
Calls:   ✓✓✓✓  ✓✓✗✗  ✗✗✗✗✗ ████  ████  ████  ████  ████  ✓    ✓✓✓✓
              failures     │     │     │     │     │     │
              mounting!    ▼     ▼     ▼     ▼     ▼     │
                        REJECTED (instant, no network)   │
                                                         │
                                                    Test call succeeds!
                                                    → Back to CLOSED
```

---

## How It Works Internally

### Failure Counting Strategies

```
Strategy 1: COUNT-BASED
┌─────────────────────────────────────────────┐
│  Track last N requests in a ring buffer     │
│                                             │
│  Buffer: [✓ ✓ ✗ ✓ ✗ ✗ ✗ ✗ ✓ ✗]           │
│           └─── last 10 requests ───┘        │
│                                             │
│  Failure rate: 6/10 = 60%                   │
│  Threshold: 50%                             │
│  60% > 50% → OPEN the circuit!             │
└─────────────────────────────────────────────┘

Strategy 2: TIME-BASED (Sliding Window)
┌─────────────────────────────────────────────┐
│  Track failures in the last T seconds       │
│                                             │
│  Window: last 60 seconds                    │
│  Failures in window: 15                     │
│  Total calls in window: 20                  │
│  Failure rate: 15/20 = 75%                  │
│  Threshold: 50%                             │
│  75% > 50% → OPEN the circuit!             │
└─────────────────────────────────────────────┘
```

### Configuration Parameters

```
┌───────────────────────┬─────────────────────────────────────────────┐
│  Parameter            │  Meaning                                    │
├───────────────────────┼─────────────────────────────────────────────┤
│  failureThreshold     │  % of failures to trip (e.g., 50%)         │
│  slidingWindowSize    │  Number of calls to evaluate (e.g., 10)    │
│  slowCallThreshold    │  Response time that counts as "slow"       │
│  slowCallRate         │  % of slow calls to trip (e.g., 80%)       │
│  waitDurationInOpen   │  How long to stay OPEN (e.g., 30s)         │
│  permittedInHalfOpen  │  Number of test calls in HALF-OPEN (e.g., 3)│
│  minimumCalls         │  Min calls before evaluating (e.g., 5)     │
└───────────────────────┴─────────────────────────────────────────────┘
```

### Internal State Machine

```
┌─────────────────────────────────────────────────────────┐
│              CIRCUIT BREAKER INTERNALS                   │
│                                                         │
│  ┌─────────────────────┐                               │
│  │  Ring Buffer         │  Tracks outcomes of last N    │
│  │  [✓][✗][✓][✗][✗]   │  requests                     │
│  └──────────┬──────────┘                               │
│             │                                           │
│             ▼                                           │
│  ┌─────────────────────┐                               │
│  │  Failure Rate Calc   │  failures / total = rate     │
│  └──────────┬──────────┘                               │
│             │                                           │
│             ▼                                           │
│  ┌─────────────────────┐                               │
│  │  State Machine       │  CLOSED→OPEN→HALF_OPEN→...  │
│  └──────────┬──────────┘                               │
│             │                                           │
│             ▼                                           │
│  ┌─────────────────────┐                               │
│  │  Timer (for OPEN)    │  Countdown to HALF-OPEN      │
│  └──────────┬──────────┘                               │
│             │                                           │
│             ▼                                           │
│  ┌─────────────────────┐                               │
│  │  Decision: Allow     │  Allow request? Or reject?   │
│  │  or Reject?          │                              │
│  └─────────────────────┘                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Circuit Breaker from Scratch

```python
import time
import threading
from enum import Enum
from collections import deque

class CircuitState(Enum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"

class CircuitBreaker:
    """A simple circuit breaker implementation."""
    
    def __init__(
        self,
        failure_threshold: float = 0.5,    # 50% failure rate to trip
        window_size: int = 10,             # Evaluate last 10 calls
        open_timeout: float = 30.0,        # Stay open for 30 seconds
        half_open_max_calls: int = 3       # Allow 3 test calls
    ):
        self.failure_threshold = failure_threshold
        self.window_size = window_size
        self.open_timeout = open_timeout
        self.half_open_max_calls = half_open_max_calls
        
        self.state = CircuitState.CLOSED
        self.results = deque(maxlen=window_size)  # Ring buffer
        self.opened_at = None
        self.half_open_calls = 0
        self.lock = threading.Lock()
    
    def call(self, func, *args, **kwargs):
        """Execute function through circuit breaker."""
        with self.lock:
            if self.state == CircuitState.OPEN:
                if self._should_attempt_reset():
                    self.state = CircuitState.HALF_OPEN
                    self.half_open_calls = 0
                else:
                    raise CircuitOpenError(
                        f"Circuit is OPEN. Retry after "
                        f"{self.open_timeout - (time.time() - self.opened_at):.1f}s")
            
            if self.state == CircuitState.HALF_OPEN:
                if self.half_open_calls >= self.half_open_max_calls:
                    raise CircuitOpenError("Circuit is HALF-OPEN, max test calls reached")
                self.half_open_calls += 1
        
        # Execute the actual call
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        with self.lock:
            self.results.append(True)
            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.CLOSED  # Recovery confirmed!
                self.results.clear()
    
    def _on_failure(self):
        with self.lock:
            self.results.append(False)
            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.OPEN  # Still broken
                self.opened_at = time.time()
            elif self._failure_rate_exceeded():
                self.state = CircuitState.OPEN
                self.opened_at = time.time()
    
    def _failure_rate_exceeded(self) -> bool:
        if len(self.results) < self.window_size:
            return False
        failure_rate = self.results.count(False) / len(self.results)
        return failure_rate >= self.failure_threshold
    
    def _should_attempt_reset(self) -> bool:
        return time.time() - self.opened_at >= self.open_timeout

class CircuitOpenError(Exception):
    pass

# --- Usage ---
import requests

breaker = CircuitBreaker(failure_threshold=0.5, window_size=10, open_timeout=30)

def fetch_recommendations(user_id):
    def _call():
        resp = requests.get(f"http://rec-service/users/{user_id}", timeout=3)
        resp.raise_for_status()
        return resp.json()
    
    try:
        return breaker.call(_call)
    except CircuitOpenError:
        # Circuit is open — return fallback (cached/default recommendations)
        return {"recommendations": ["popular-item-1", "popular-item-2"]}
```

### Python — Using `pybreaker` Library

```python
import pybreaker
import requests

# Configure circuit breaker
breaker = pybreaker.CircuitBreaker(
    fail_max=5,           # Trip after 5 consecutive failures
    reset_timeout=30,     # Try again after 30 seconds
    exclude=[             # Don't count these as failures
        requests.exceptions.HTTPError  # Only 5xx, not 4xx
    ]
)

@breaker
def call_payment_service(order_id: str, amount: float):
    """This function is protected by the circuit breaker."""
    response = requests.post(
        "http://payment-service/charge",
        json={"order_id": order_id, "amount": amount},
        timeout=(3, 10)
    )
    response.raise_for_status()
    return response.json()

# Usage with fallback
def process_payment(order_id, amount):
    try:
        return call_payment_service(order_id, amount)
    except pybreaker.CircuitBreakerError:
        # Circuit is open — queue payment for later
        queue_payment_for_retry(order_id, amount)
        return {"status": "pending", "message": "Payment queued"}
```

### Java — Resilience4j Circuit Breaker

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;

import java.time.Duration;
import java.util.function.Supplier;

public class PaymentServiceClient {
    
    private final CircuitBreaker circuitBreaker;
    
    public PaymentServiceClient() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)              // Trip at 50% failure rate
            .slowCallRateThreshold(80)             // Trip if 80% calls are slow
            .slowCallDurationThreshold(Duration.ofSeconds(3))  // "Slow" = >3s
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(10)                 // Evaluate last 10 calls
            .minimumNumberOfCalls(5)               // Need at least 5 calls
            .waitDurationInOpenState(Duration.ofSeconds(30))   // Stay open 30s
            .permittedNumberOfCallsInHalfOpenState(3)          // 3 test calls
            .automaticTransitionFromOpenToHalfOpenEnabled(true)
            .build();
        
        CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(config);
        this.circuitBreaker = registry.circuitBreaker("paymentService");
        
        // Listen for state transitions (for logging/alerting)
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                System.out.println("Circuit state: " + event.getStateTransition()));
    }
    
    public PaymentResult chargeCustomer(String orderId, double amount) {
        // Wrap the call with circuit breaker
        Supplier<PaymentResult> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> {
                // Actual HTTP call to payment service
                return httpClient.post("/charge", 
                    new ChargeRequest(orderId, amount));
            });
        
        try {
            return decoratedSupplier.get();
        } catch (io.github.resilience4j.circuitbreaker.CallNotPermittedException e) {
            // Circuit is OPEN — use fallback
            System.out.println("Circuit OPEN: payment queued for retry");
            return new PaymentResult("pending", "Payment service unavailable");
        }
    }
}
```

### Java — Spring Boot with Circuit Breaker Annotation

```java
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class RecommendationService {
    
    private final RestTemplate restTemplate;
    
    @CircuitBreaker(name = "recommendationService", fallbackMethod = "getDefaultRecs")
    public List<String> getRecommendations(String userId) {
        // If circuit is open, this won't execute — fallback is called instead
        ResponseEntity<List<String>> response = restTemplate.exchange(
            "http://rec-service/users/{id}/recommendations",
            HttpMethod.GET, null,
            new ParameterizedTypeReference<List<String>>() {},
            userId
        );
        return response.getBody();
    }
    
    // Fallback method — MUST have same return type + exception parameter
    private List<String> getDefaultRecs(String userId, Exception ex) {
        System.out.println("Circuit open or call failed: " + ex.getMessage());
        // Return cached popular items as fallback
        return List.of("bestseller-1", "bestseller-2", "trending-1");
    }
}
```

---

## Infrastructure Examples

### Envoy Proxy — Outlier Detection (Automatic Circuit Breaking)

```yaml
# Envoy automatically ejects unhealthy hosts from load balancing
clusters:
  - name: payment-service
    connect_timeout: 3s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    outlier_detection:
      consecutive_5xx: 5                    # Eject after 5 consecutive 5xx errors
      interval: 10s                         # Check every 10 seconds
      base_ejection_time: 30s              # First ejection lasts 30s
      max_ejection_percent: 50             # Never eject more than 50% of hosts
      consecutive_gateway_failure: 3        # Eject after 3 gateway errors
      enforcing_consecutive_5xx: 100       # 100% enforcement
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 1000            # Max concurrent connections
          max_pending_requests: 500        # Max queued requests
          max_requests: 2000              # Max concurrent requests
          max_retries: 3                  # Max concurrent retries
```

### Istio Service Mesh — Circuit Breaking

```yaml
# Istio DestinationRule with circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100              # Max TCP connections
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 50      # Max pending requests
        http2MaxRequests: 100            # Max concurrent requests
    outlierDetection:
      consecutive5xxErrors: 5            # Trip after 5 errors
      interval: 10s                      # Evaluation interval
      baseEjectionTime: 30s             # Initial ejection duration
      maxEjectionPercent: 50            # Max % of hosts ejected
```

### Spring Boot application.yml — Circuit Breaker Config

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        register-health-indicator: true   # Expose in /actuator/health
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 3s
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        minimum-number-of-calls: 5
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:
          - java.net.SocketTimeoutException
          - java.net.ConnectException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - org.springframework.web.client.HttpClientErrorException
      
      inventoryService:
        sliding-window-type: TIME_BASED
        sliding-window-size: 60           # 60-second window
        failure-rate-threshold: 60
        wait-duration-in-open-state: 20s
```

---

## Real-World Example

### Netflix Hystrix (The Pioneer)

Netflix invented the Circuit Breaker pattern for microservices at scale. Their system:

```
┌─────────────────────────────────────────────────────────────────────┐
│                 NETFLIX CIRCUIT BREAKER ARCHITECTURE                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  User Request                                                       │
│       │                                                             │
│       ▼                                                             │
│  ┌──────────────────┐                                              │
│  │   API Gateway    │                                              │
│  └────────┬─────────┘                                              │
│           │                                                         │
│           ▼                                                         │
│  ┌────────────────────────────────────────┐                        │
│  │  CIRCUIT BREAKER (per dependency)       │                        │
│  │                                         │                        │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐  │                        │
│  │  │  User   │ │ Ratings │ │  Recs   │  │                        │
│  │  │ Service │ │ Service │ │ Service │  │                        │
│  │  │ CLOSED  │ │ CLOSED  │ │  OPEN   │  │                        │
│  │  └────┬────┘ └────┬────┘ └────┬────┘  │                        │
│  │       │           │           │        │                        │
│  │       ▼           ▼           ▼        │                        │
│  │    Call it      Call it     FALLBACK    │                        │
│  │    normally     normally    (cached     │                        │
│  │                             recs)       │                        │
│  └────────────────────────────────────────┘                        │
│                                                                     │
│  Key insight: Each downstream service has its OWN circuit breaker.  │
│  One service being down doesn't affect calls to other services.     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### How Netflix Uses It in Practice

```
Without Circuit Breaker:
  Recommendation Service goes down
    → API waits 10s for timeout × 100 threads
    → All threads blocked
    → API Gateway becomes unresponsive
    → ENTIRE Netflix appears down!

With Circuit Breaker:
  Recommendation Service goes down
    → 5 failures detected → Circuit OPENS
    → All subsequent calls return fallback in <1ms
    → "Trending Now" shown instead of personalized recs
    → User doesn't even notice!
    → Rest of Netflix works perfectly
```

---

## Common Mistakes / Pitfalls

### 1. Too Sensitive (Trips Too Easily)

```
❌ BAD: failureThreshold=10%, windowSize=5
   → Just 1 failure out of 5 calls trips the circuit
   → One slow request causes ALL traffic to be rejected!

✅ GOOD: failureThreshold=50%, windowSize=20, minimumCalls=10
   → Need sustained failures before tripping
   → Tolerates occasional blips
```

### 2. Too Insensitive (Doesn't Trip When It Should)

```
❌ BAD: failureThreshold=95%, windowSize=100
   → Service must fail 95% of the time over 100 calls!
   → Wastes 95 calls before detecting the problem

✅ GOOD: Balance between sensitivity and stability
   → 50% threshold over 10-20 calls is typical
```

### 3. Open Duration Too Short

```
❌ BAD: waitDurationInOpenState=2s
   → Circuit opens, immediately tries again after 2s
   → Service hasn't had time to recover
   → Goes OPEN again, creating a rapid OPEN/HALF-OPEN loop

✅ GOOD: waitDurationInOpenState=30s
   → Gives the service real time to recover (restart, autoscale)
```

### 4. No Fallback Strategy

```
❌ BAD: Circuit opens → throw error → user sees "500 Internal Server Error"

✅ GOOD: Circuit opens → return fallback:
   - Cached data (stale but useful)
   - Default values (popular items)
   - Degraded experience (fewer features)
   - Queued for later processing
```

### 5. Circuit Breaker Without Monitoring

If you don't monitor circuit state, you won't know when services are degraded:

```
✅ Must monitor:
   - Circuit state changes (CLOSED → OPEN)  → Alert!
   - Failure rates per service
   - How long circuits stay open
   - Fallback hit rate
```

---

## When to Use / When NOT to Use

### ✅ Use Circuit Breaker When

| Scenario | Why |
|----------|-----|
| Calling external APIs | Third-party services are unreliable |
| Microservice-to-microservice calls | Prevent cascading failures |
| Database calls during high load | Protect DB from being overwhelmed |
| Calls with high latency when failing | Avoid thread pool exhaustion |
| Any remote dependency | Network is inherently unreliable |

### ❌ When NOT to Use

| Scenario | Why |
|----------|-----|
| Local in-memory operations | Can't "fail" in the same way |
| Fire-and-forget async messages | Use DLQ instead |
| Operations you MUST complete | Use saga pattern or queues |
| Between components in same process | Use bulkheads instead |
| Infrequent batch operations | Not enough calls to calculate failure rate |

---

## Circuit Breaker vs Retry — How They Work Together

```
┌───────────────────────────────────────────────────────────────┐
│           RETRY + CIRCUIT BREAKER TOGETHER                    │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Request                                                      │
│    │                                                          │
│    ▼                                                          │
│  ┌──────────────────┐                                        │
│  │ CIRCUIT BREAKER   │  ← "Is circuit open?"                 │
│  │                   │     If YES → return fallback           │
│  │                   │     If NO  → proceed                   │
│  └────────┬──────────┘                                        │
│           │                                                    │
│           ▼                                                    │
│  ┌──────────────────┐                                        │
│  │ RETRY MECHANISM   │  ← Try up to 3 times with backoff     │
│  │                   │                                        │
│  └────────┬──────────┘                                        │
│           │                                                    │
│           ▼                                                    │
│  ┌──────────────────┐                                        │
│  │ ACTUAL SERVICE    │                                        │
│  │ CALL              │                                        │
│  └──────────────────┘                                        │
│                                                               │
│  Order matters: Circuit Breaker WRAPS Retry                   │
│  (Check circuit first, then retry if circuit allows)          │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **Circuit Breakers prevent cascading failures** — when one service dies, it doesn't take everything else down with it.
- **Three states: CLOSED (normal), OPEN (rejecting), HALF-OPEN (testing)** — the state machine is simple but powerful.
- **Fast failure is better than slow failure** — rejecting in <1ms is better than waiting 10 seconds for a timeout.
- **Always provide a fallback** — cached data, defaults, or queued processing. Never just throw errors at users.
- **Each dependency gets its own circuit breaker** — isolate failures per service.
- **Monitor circuit state transitions** — an OPEN circuit is an early warning that needs investigation.
- **Combine with retries** — circuit breaker wraps retry logic. Retries handle blips, circuit breakers handle sustained outages.

---

## What's Next?

The Circuit Breaker protects you from ONE failing service. But what about **isolating** different parts of your system so that a failure in one area doesn't consume all your resources? That's the **Bulkhead Pattern** — see [04-bulkhead-pattern.md](./04-bulkhead-pattern.md).
