# Distributed System — Low-Level Design

Distributed systems involve multiple services/nodes working together. LLD for distributed systems focuses on **patterns, data consistency, communication, and failure handling** at the code/class level.

---

## 1. Communication Patterns

### Synchronous (Request-Response)

```
Client → Service A → Service B → Response flows back
```

- REST, gRPC, GraphQL
- Simple but causes **tight coupling** and **cascading failures**

### Asynchronous (Event-Driven)

```
Service A → Message Queue → Service B (consumes when ready)
```

- Message brokers: Kafka, RabbitMQ, SQS
- Loose coupling, better fault tolerance

### Code: REST Client with Retry

```java
public class OrderService {
    private final RestTemplate restTemplate;
    private final RetryTemplate retryTemplate;

    public OrderService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
        this.retryTemplate = RetryTemplate.builder()
            .maxAttempts(3)
            .fixedBackoff(1000)  // 1 second between retries
            .retryOn(RestClientException.class)
            .build();
    }

    public InventoryResponse checkInventory(String productId) {
        return retryTemplate.execute(context ->
            restTemplate.getForObject(
                "http://inventory-service/api/products/" + productId + "/stock",
                InventoryResponse.class
            )
        );
    }
}
```

### Code: Event Publisher

```java
public class OrderService {
    private final EventPublisher eventPublisher;
    private final OrderRepository orderRepository;

    public Order placeOrder(OrderRequest request) {
        Order order = Order.create(request);
        orderRepository.save(order);

        // Publish event asynchronously
        eventPublisher.publish(new OrderPlacedEvent(
            order.getId(),
            order.getUserId(),
            order.getItems(),
            order.getTotalAmount()
        ));

        return order;
    }
}

// Event class
public class OrderPlacedEvent {
    private final String orderId;
    private final String userId;
    private final List<OrderItem> items;
    private final BigDecimal totalAmount;
    private final Instant occurredAt;

    public OrderPlacedEvent(String orderId, String userId,
                            List<OrderItem> items, BigDecimal totalAmount) {
        this.orderId = orderId;
        this.userId = userId;
        this.items = items;
        this.totalAmount = totalAmount;
        this.occurredAt = Instant.now();
    }
}
```

---

## 2. Circuit Breaker Pattern

Prevents cascading failures by stopping calls to a failing downstream service.

### States

```
CLOSED  →  (failures exceed threshold)  →  OPEN
  ↑                                           |
  |         (timeout expires)                 ↓
  └─── HALF_OPEN ←────────────────────── (allow one test call)
```

| State | Behavior |
|-------|----------|
| **CLOSED** | Requests pass through normally. Failures are counted. |
| **OPEN** | All requests fail fast with fallback. No calls to downstream. |
| **HALF_OPEN** | One test request allowed. Success → CLOSED, Failure → OPEN |

### Implementation

```java
public class CircuitBreaker {
    private State state = State.CLOSED;
    private int failureCount = 0;
    private final int failureThreshold;
    private final long openTimeoutMs;
    private long lastFailureTime;

    public enum State { CLOSED, OPEN, HALF_OPEN }

    public CircuitBreaker(int failureThreshold, long openTimeoutMs) {
        this.failureThreshold = failureThreshold;
        this.openTimeoutMs = openTimeoutMs;
    }

    public <T> T execute(Supplier<T> action, Supplier<T> fallback) {
        if (state == State.OPEN) {
            if (System.currentTimeMillis() - lastFailureTime > openTimeoutMs) {
                state = State.HALF_OPEN;
            } else {
                return fallback.get();  // Fail fast
            }
        }

        try {
            T result = action.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            return fallback.get();
        }
    }

    private synchronized void onSuccess() {
        failureCount = 0;
        state = State.CLOSED;
    }

    private synchronized void onFailure() {
        failureCount++;
        lastFailureTime = System.currentTimeMillis();
        if (failureCount >= failureThreshold) {
            state = State.OPEN;
        }
    }
}
```

### Usage with Resilience4j (Library)

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)           // Open if 50% fail
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .slidingWindowSize(10)
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("inventoryService", config);

Supplier<InventoryResponse> decorated = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> inventoryClient.check(productId));

Try<InventoryResponse> result = Try.ofSupplier(decorated)
    .recover(throwable -> fallbackResponse());
```

---

## 3. Saga Pattern (Distributed Transactions)

Manages distributed transactions across services using a sequence of local transactions with compensating actions.

### Types

| Type | Description | Use When |
|------|-------------|----------|
| **Choreography** | Each service listens for events and reacts | Few services, simple flow |
| **Orchestration** | Central orchestrator coordinates steps | Complex flows, many services |

### Choreography-Based Saga

```
OrderService          PaymentService         InventoryService
     |                      |                       |
     |── OrderCreated ──►   |                       |
     |                      |── PaymentProcessed ──►|
     |                      |                       |── InventoryReserved
     |                      |                       |
     |◄── OrderConfirmed ───|◄── StockConfirmed ────|
```

### Orchestration-Based Saga

```java
public class OrderSagaOrchestrator {
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final ShippingService shippingService;
    private final OrderRepository orderRepository;

    public OrderResult executeSaga(Order order) {
        List<SagaStep> completedSteps = new ArrayList<>();

        try {
            // Step 1: Reserve inventory
            inventoryService.reserve(order.getItems());
            completedSteps.add(SagaStep.INVENTORY_RESERVED);

            // Step 2: Process payment
            paymentService.charge(order.getUserId(), order.getTotalAmount());
            completedSteps.add(SagaStep.PAYMENT_CHARGED);

            // Step 3: Create shipment
            shippingService.createShipment(order);
            completedSteps.add(SagaStep.SHIPMENT_CREATED);

            order.setStatus(OrderStatus.CONFIRMED);
            orderRepository.save(order);
            return OrderResult.success(order);

        } catch (Exception e) {
            // Compensate in reverse order
            compensate(order, completedSteps);
            order.setStatus(OrderStatus.FAILED);
            orderRepository.save(order);
            return OrderResult.failure(order, e.getMessage());
        }
    }

    private void compensate(Order order, List<SagaStep> completedSteps) {
        Collections.reverse(completedSteps);

        for (SagaStep step : completedSteps) {
            try {
                switch (step) {
                    case SHIPMENT_CREATED -> shippingService.cancelShipment(order.getId());
                    case PAYMENT_CHARGED  -> paymentService.refund(order.getUserId(), order.getTotalAmount());
                    case INVENTORY_RESERVED -> inventoryService.release(order.getItems());
                }
            } catch (Exception e) {
                // Log compensation failure — may need manual intervention
                log.error("Compensation failed for step: {} order: {}", step, order.getId(), e);
            }
        }
    }
}
```

---

## 4. Idempotency

Ensuring that performing the same operation multiple times has the same effect as doing it once.

### Why It Matters
- Network retries may send duplicate requests
- Message queues may deliver the same message twice
- Users may double-click buttons

### Idempotency Key Pattern

```java
public class PaymentService {
    private final PaymentRepository paymentRepository;
    private final IdempotencyKeyStore keyStore;

    public PaymentResponse processPayment(PaymentRequest request) {
        String idempotencyKey = request.getIdempotencyKey();

        // Check if already processed
        Optional<PaymentResponse> existing = keyStore.get(idempotencyKey);
        if (existing.isPresent()) {
            return existing.get();  // Return cached response
        }

        // Process payment
        Payment payment = Payment.create(request);
        paymentRepository.save(payment);

        PaymentResponse response = new PaymentResponse(payment.getId(), "SUCCESS");

        // Store result with TTL
        keyStore.put(idempotencyKey, response, Duration.ofHours(24));

        return response;
    }
}
```

### Database-Level Idempotency

```sql
-- Unique constraint prevents duplicate processing
ALTER TABLE payments ADD CONSTRAINT uq_idempotency_key UNIQUE (idempotency_key);
```

```java
// Insert-or-skip approach
public void processEvent(OrderPlacedEvent event) {
    try {
        orderRepository.insertIfNotExists(event.getOrderId(), event);
    } catch (DuplicateKeyException e) {
        log.info("Event already processed: {}", event.getOrderId());
    }
}
```

---

## 5. Distributed Locking

Ensures only one node/service processes a resource at a time.

### Redis-Based Distributed Lock

```java
public class RedisDistributedLock {
    private final RedisTemplate<String, String> redis;

    public boolean acquireLock(String lockKey, String ownerId, Duration ttl) {
        Boolean acquired = redis.opsForValue()
            .setIfAbsent(lockKey, ownerId, ttl);
        return Boolean.TRUE.equals(acquired);
    }

    public boolean releaseLock(String lockKey, String ownerId) {
        // Atomic check-and-delete using Lua script
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        Long result = redis.execute(
            new DefaultRedisScript<>(script, Long.class),
            List.of(lockKey), ownerId
        );
        return result != null && result == 1;
    }
}

// Usage
public void processExclusively(String resourceId) {
    String lockKey = "lock:" + resourceId;
    String ownerId = UUID.randomUUID().toString();

    if (distributedLock.acquireLock(lockKey, ownerId, Duration.ofSeconds(30))) {
        try {
            // Critical section — only one node executes this
            doWork(resourceId);
        } finally {
            distributedLock.releaseLock(lockKey, ownerId);
        }
    } else {
        throw new ResourceBusyException("Resource is being processed by another node");
    }
}
```

---

## 6. Rate Limiting

Controls how many requests a client can make in a given time window.

### Token Bucket Algorithm

```java
public class TokenBucketRateLimiter {
    private final int maxTokens;
    private final int refillRate;  // tokens per second
    private double availableTokens;
    private long lastRefillTimestamp;

    public TokenBucketRateLimiter(int maxTokens, int refillRate) {
        this.maxTokens = maxTokens;
        this.refillRate = refillRate;
        this.availableTokens = maxTokens;
        this.lastRefillTimestamp = System.nanoTime();
    }

    public synchronized boolean tryConsume() {
        refill();
        if (availableTokens >= 1) {
            availableTokens--;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.nanoTime();
        double elapsedSeconds = (now - lastRefillTimestamp) / 1_000_000_000.0;
        availableTokens = Math.min(maxTokens, availableTokens + elapsedSeconds * refillRate);
        lastRefillTimestamp = now;
    }
}
```

### Sliding Window Counter

```java
public class SlidingWindowRateLimiter {
    private final int maxRequests;
    private final long windowSizeMs;
    private final ConcurrentLinkedDeque<Long> requestTimestamps = new ConcurrentLinkedDeque<>();

    public SlidingWindowRateLimiter(int maxRequests, long windowSizeMs) {
        this.maxRequests = maxRequests;
        this.windowSizeMs = windowSizeMs;
    }

    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();
        long windowStart = now - windowSizeMs;

        // Remove expired entries
        while (!requestTimestamps.isEmpty() && requestTimestamps.peekFirst() < windowStart) {
            requestTimestamps.pollFirst();
        }

        if (requestTimestamps.size() < maxRequests) {
            requestTimestamps.addLast(now);
            return true;
        }
        return false;
    }
}
```

---

## 7. Service Discovery & Registry

How services find and communicate with each other.

### Service Registry Pattern

```java
// Service Registration
public class ServiceRegistration {
    private final String serviceName;
    private final String instanceId;
    private final String host;
    private final int port;
    private final Map<String, String> metadata;
    private ServiceStatus status;

    public String getAddress() {
        return host + ":" + port;
    }
}

// Service Registry Interface
public interface ServiceRegistry {
    void register(ServiceRegistration registration);
    void deregister(String serviceName, String instanceId);
    List<ServiceRegistration> getInstances(String serviceName);
    void heartbeat(String serviceName, String instanceId);
}

// Client-Side Load Balancing
public class RoundRobinLoadBalancer {
    private final ServiceRegistry registry;
    private final AtomicInteger counter = new AtomicInteger(0);

    public ServiceRegistration getNextInstance(String serviceName) {
        List<ServiceRegistration> instances = registry.getInstances(serviceName);
        if (instances.isEmpty()) {
            throw new NoAvailableInstanceException(serviceName);
        }
        int index = counter.getAndIncrement() % instances.size();
        return instances.get(index);
    }
}
```

---

## 8. Outbox Pattern (Reliable Event Publishing)

Guarantees that database changes and events are published atomically.

### Problem
```
1. Save order to DB       ✅ Success
2. Publish OrderCreated   ❌ Fails (message broker down)
→ Inconsistency: order exists but no event was published
```

### Solution: Transactional Outbox

```java
// Step 1: Write both order AND outbox entry in same transaction
@Transactional
public Order createOrder(OrderRequest request) {
    Order order = Order.create(request);
    orderRepository.save(order);

    // Save event to outbox table (same DB transaction)
    OutboxEvent event = new OutboxEvent(
        UUID.randomUUID().toString(),
        "OrderCreated",
        objectMapper.writeValueAsString(new OrderCreatedPayload(order)),
        Instant.now()
    );
    outboxRepository.save(event);

    return order;
}

// Step 2: Background poller publishes events from outbox
@Scheduled(fixedRate = 1000)
public void publishOutboxEvents() {
    List<OutboxEvent> pendingEvents = outboxRepository.findUnpublished();

    for (OutboxEvent event : pendingEvents) {
        try {
            eventPublisher.publish(event.getType(), event.getPayload());
            event.markPublished();
            outboxRepository.save(event);
        } catch (Exception e) {
            log.warn("Failed to publish outbox event: {}", event.getId(), e);
        }
    }
}

// Outbox table
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id
    private String id;
    private String type;
    private String payload;
    private Instant createdAt;
    private boolean published;
    private Instant publishedAt;
}
```

---

## 9. CQRS (Command Query Responsibility Segregation)

Separates read and write models for different optimization.

```
Commands (Write)                  Queries (Read)
     │                                 │
     ▼                                 ▼
CommandHandler                   QueryHandler
     │                                 │
     ▼                                 ▼
Write Model (normalized)        Read Model (denormalized)
     │                                 ▲
     └──── Events ────────────────────┘
```

### Implementation

```java
// Command Side
public class CreateOrderCommand {
    private final String userId;
    private final List<OrderItemDTO> items;
}

public class OrderCommandHandler {
    private final OrderRepository writeRepo;
    private final EventPublisher eventPublisher;

    public String handle(CreateOrderCommand cmd) {
        Order order = Order.create(cmd.getUserId(), cmd.getItems());
        writeRepo.save(order);
        eventPublisher.publish(new OrderCreatedEvent(order));
        return order.getId();
    }
}

// Query Side
public class OrderQueryHandler {
    private final OrderReadRepository readRepo;

    public OrderSummaryDTO getOrderSummary(String orderId) {
        return readRepo.findSummaryById(orderId);  // Denormalized, fast read
    }

    public List<OrderSummaryDTO> getUserOrders(String userId, int page, int size) {
        return readRepo.findByUserId(userId, PageRequest.of(page, size));
    }
}

// Event handler to sync read model
@EventHandler
public class OrderReadModelUpdater {
    private final OrderReadRepository readRepo;

    public void on(OrderCreatedEvent event) {
        OrderReadModel readModel = new OrderReadModel(
            event.getOrderId(),
            event.getUserId(),
            event.getItems(),
            event.getTotalAmount(),
            event.getStatus()
        );
        readRepo.save(readModel);
    }
}
```

---

## 10. Bulkhead Pattern

Isolates failures by partitioning resources (like separate thread pools per service).

```java
// Separate thread pools for different downstream services
public class BulkheadConfig {

    @Bean("paymentExecutor")
    public ExecutorService paymentExecutor() {
        return new ThreadPoolExecutor(5, 10, 60, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(100));
    }

    @Bean("inventoryExecutor")
    public ExecutorService inventoryExecutor() {
        return new ThreadPoolExecutor(5, 10, 60, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(50));
    }
}

// Usage — payment failures won't exhaust threads for inventory calls
public class OrderService {
    @Qualifier("paymentExecutor")
    private final ExecutorService paymentExecutor;

    @Qualifier("inventoryExecutor")
    private final ExecutorService inventoryExecutor;

    public CompletableFuture<PaymentResult> processPayment(Order order) {
        return CompletableFuture.supplyAsync(
            () -> paymentClient.charge(order),
            paymentExecutor
        );
    }

    public CompletableFuture<InventoryResult> reserveInventory(Order order) {
        return CompletableFuture.supplyAsync(
            () -> inventoryClient.reserve(order.getItems()),
            inventoryExecutor
        );
    }
}
```

---

## Quick Reference — When to Use What

| Pattern | Use When |
|---------|----------|
| **Circuit Breaker** | Calling unreliable external services |
| **Saga** | Multi-service transactions that must be consistent |
| **Idempotency** | Handling retries or duplicate messages |
| **Distributed Lock** | Only one node should process a resource at a time |
| **Rate Limiter** | Protecting APIs from abuse / traffic spikes |
| **Outbox** | Need atomicity between DB write and event publish |
| **CQRS** | Read and write workloads have very different needs |
| **Bulkhead** | Isolating failures between downstream dependencies |
| **Retry + Backoff** | Transient failures in network calls |

---

## Common Interview Questions

| Question | Key Points |
|----------|------------|
| How do you handle distributed transactions? | Saga pattern (choreography vs orchestration), compensating transactions |
| How do you prevent duplicate processing? | Idempotency keys, unique constraints, deduplication |
| How do you handle a downstream service being down? | Circuit breaker, fallback, retry with exponential backoff |
| How do you publish events reliably? | Outbox pattern, CDC (Change Data Capture) |
| How do you handle concurrent access across nodes? | Distributed locking (Redis/Zookeeper), optimistic locking |
| What is eventual consistency? | Data will converge but may be stale temporarily; use events to sync |
