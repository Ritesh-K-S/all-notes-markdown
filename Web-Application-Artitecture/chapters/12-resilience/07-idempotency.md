# Idempotency — Making Operations Safe to Retry

> **What you'll learn**: How to design APIs and operations so that executing them multiple times produces the same result as executing them once — making retries, network failures, and duplicate messages completely safe.

---

## Real-Life Analogy

Think about an **elevator button**:

- You press the "3rd floor" button once → elevator goes to 3rd floor.
- You press it again (because you're nervous) → nothing extra happens. Still goes to 3rd floor.
- You press it 50 times → STILL just goes to 3rd floor once.

The elevator button is **idempotent** — pressing it multiple times has the same effect as pressing it once.

Now think about a **vending machine**:

- You insert $1 and press "Cola" → you get 1 cola.
- Network glitch! Your button press is "retried" → you get ANOTHER cola (charged $2!).

The vending machine is **NOT idempotent** — repeating the operation has a different effect.

In software, **idempotency** means: **doing an operation 1 time or N times produces the same final state.**

---

## Core Concept Explained Step-by-Step

### Why Do We Need Idempotency?

In distributed systems, duplicate requests are INEVITABLE:

```
┌─────────────────────────────────────────────────────────────────┐
│         WHY DUPLICATE REQUESTS HAPPEN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Scenario 1: CLIENT TIMEOUT                                     │
│  Client ──▶ "Charge $100" ──▶ Server (processes it!)           │
│  Client ◀── Response lost in network ──◀ Server                │
│  Client: "Did it work? I'll try again..."                      │
│  Client ──▶ "Charge $100" ──▶ Server (charges AGAIN! 💀)       │
│                                                                 │
│  Scenario 2: RETRY MECHANISM                                    │
│  Client ──▶ Request ──▶ Server (processes, but response slow)  │
│  Client: "Timeout! Let me retry..."                            │
│  Client ──▶ Same Request ──▶ Server (processes AGAIN!)          │
│                                                                 │
│  Scenario 3: MESSAGE QUEUE REDELIVERY                           │
│  Queue ──▶ "Process order" ──▶ Consumer (processes it)          │
│  Consumer crashes before ACK                                    │
│  Queue ──▶ "Process order" ──▶ New Consumer (processes AGAIN!)  │
│                                                                 │
│  Scenario 4: USER DOUBLE-CLICK                                  │
│  User clicks "Pay" twice quickly → two payment requests!        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Naturally Idempotent vs Non-Idempotent Operations

```
┌────────────────────────────────────────────────────────────────┐
│  NATURALLY IDEMPOTENT                                          │
│  (Safe to repeat without special handling)                     │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  GET /users/123           → Always returns same user           │
│  PUT /users/123 {name: "Alice"}  → Sets to "Alice" every time │
│  DELETE /orders/456       → Deleted remains deleted            │
│  SET key=value (Redis)    → Same value regardless of repeats   │
│  UPDATE users SET name='Bob' WHERE id=123                      │
│                                                                │
├────────────────────────────────────────────────────────────────┤
│  NOT IDEMPOTENT                                                │
│  (Dangerous to repeat without protection!)                     │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  POST /payments/charge    → Charges again each time! 💀        │
│  POST /orders             → Creates duplicate orders! 💀       │
│  POST /emails/send        → Sends email multiple times! 💀     │
│  INSERT INTO orders (...)  → Creates duplicate rows! 💀        │
│  UPDATE accounts SET balance = balance - 100                   │
│     → Deducts multiple times! 💀                               │
│  counter++ (increment)    → Increments multiple times! 💀      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### The Idempotency Key Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│              IDEMPOTENCY KEY FLOW                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FIRST REQUEST:                                                 │
│  Client ──▶ POST /payments                                     │
│              Idempotency-Key: "abc-123-xyz"                     │
│              Body: {amount: 100, card: "..."}                   │
│                          │                                      │
│                          ▼                                      │
│              ┌─────────────────────────┐                       │
│              │ Check: seen "abc-123-xyz"│                       │
│              │ before?                  │                       │
│              └──────┬──────────────────┘                       │
│                     │ NO (first time)                           │
│                     ▼                                           │
│              ┌─────────────────────────┐                       │
│              │ Process payment          │                       │
│              │ Save: key="abc-123-xyz"  │                       │
│              │       result={id: "pay_1"}│                      │
│              └──────────────────────────┘                       │
│                     │                                           │
│  Client ◀── 200 OK {id: "pay_1"} ◀──┘                         │
│                                                                 │
│                                                                 │
│  DUPLICATE REQUEST (retry):                                     │
│  Client ──▶ POST /payments                                     │
│              Idempotency-Key: "abc-123-xyz"  (SAME key!)        │
│              Body: {amount: 100, card: "..."}                   │
│                          │                                      │
│                          ▼                                      │
│              ┌─────────────────────────┐                       │
│              │ Check: seen "abc-123-xyz"│                       │
│              │ before?                  │                       │
│              └──────┬──────────────────┘                       │
│                     │ YES! (already processed)                  │
│                     ▼                                           │
│              ┌─────────────────────────┐                       │
│              │ Return CACHED result     │                       │
│              │ (don't process again!)   │                       │
│              └──────────────────────────┘                       │
│                     │                                           │
│  Client ◀── 200 OK {id: "pay_1"} ◀──┘  (Same response!)      │
│                                                                 │
│  Result: Customer charged exactly ONCE, regardless of retries.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Idempotency Key Storage

```
┌─────────────────────────────────────────────────────────────────┐
│          IDEMPOTENCY KEY STORAGE OPTIONS                         │
├──────────────┬──────────────────────────────────────────────────┤
│ Storage      │ Trade-offs                                       │
├──────────────┼──────────────────────────────────────────────────┤
│ Database     │ + Durable (survives restarts)                    │
│ (PostgreSQL) │ + Consistent with business data (same TX)        │
│              │ - Slower than in-memory                          │
│              │ ✓ Best for financial operations                  │
├──────────────┼──────────────────────────────────────────────────┤
│ Redis        │ + Very fast lookups                              │
│              │ + TTL support (auto-cleanup)                     │
│              │ - May lose data on restart (if no persistence)   │
│              │ ✓ Good for non-critical idempotency              │
├──────────────┼──────────────────────────────────────────────────┤
│ Same DB      │ + ACID guarantee with business logic             │
│ Transaction  │ + Cannot have inconsistency                      │
│              │ - Ties idempotency to business DB                │
│              │ ✓ Best for absolute correctness                  │
└──────────────┴──────────────────────────────────────────────────┘
```

### Race Condition: Concurrent Duplicate Requests

```
What if two identical requests arrive at the EXACT same time?

Timeline without protection:
  Request A: Check key → Not found → Process payment → Store key
  Request B: Check key → Not found → Process payment → Store key
  Result: DOUBLE CHARGE! (both passed the check before either stored)

Timeline WITH database-level protection:
  Request A: INSERT INTO idempotency_keys (key, ...) 
             → Success (unique constraint)
             → Process payment
  Request B: INSERT INTO idempotency_keys (key, ...)
             → FAILS! (unique constraint violation)
             → Return "already processing" or wait for A's result

The UNIQUE CONSTRAINT is what makes this safe against races!
```

### State Machine for Idempotency Processing

```
┌──────────────────────────────────────────────────────────────┐
│           IDEMPOTENCY REQUEST STATES                          │
│                                                              │
│  ┌─────────┐    ┌────────────┐    ┌───────────┐            │
│  │  NEW     │───▶│ PROCESSING │───▶│ COMPLETED │            │
│  │         │    │            │    │           │            │
│  └─────────┘    └─────┬──────┘    └───────────┘            │
│                        │                                     │
│                        ▼                                     │
│                  ┌───────────┐                               │
│                  │  FAILED   │                               │
│                  └───────────┘                               │
│                                                              │
│  If duplicate arrives while state = PROCESSING:              │
│    → Return 409 Conflict or wait and return same result      │
│                                                              │
│  If duplicate arrives while state = COMPLETED:               │
│    → Return cached result immediately                        │
│                                                              │
│  If duplicate arrives while state = FAILED:                  │
│    → Allow retry (reset to NEW)                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Generation Strategies

```
┌────────────────────┬────────────────────────────────────────────┐
│ Strategy           │ Example                                    │
├────────────────────┼────────────────────────────────────────────┤
│ Client-generated   │ UUID v4: "550e8400-e29b-41d4-a716-..."    │
│ UUID               │ Client generates, sends in header          │
├────────────────────┼────────────────────────────────────────────┤
│ Business-derived   │ "order-{orderId}-payment-{attempt}"       │
│                    │ "user-123-transfer-456"                    │
│                    │ Naturally unique per business operation     │
├────────────────────┼────────────────────────────────────────────┤
│ Hash of request    │ SHA256(method + url + body + user_id)      │
│                    │ Same request always = same key             │
├────────────────────┼────────────────────────────────────────────┤
│ Sequence number    │ "client-A-seq-1234"                       │
│                    │ Client tracks its own sequence             │
└────────────────────┴────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Idempotent Payment API

```python
import uuid
import json
from datetime import datetime, timedelta
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel
import redis

app = FastAPI()
redis_client = redis.Redis(host='redis', port=6379, decode_responses=True)

class PaymentRequest(BaseModel):
    order_id: str
    amount: float
    currency: str = "USD"

class PaymentResponse(BaseModel):
    payment_id: str
    status: str
    amount: float

@app.post("/api/v1/payments", response_model=PaymentResponse)
async def create_payment(
    payment: PaymentRequest,
    idempotency_key: str = Header(..., alias="Idempotency-Key")
):
    """
    Idempotent payment endpoint.
    Same Idempotency-Key always returns the same result.
    """
    
    # Step 1: Check if we've seen this key before
    cache_key = f"idempotency:{idempotency_key}"
    existing = redis_client.get(cache_key)
    
    if existing:
        # Already processed! Return cached result
        cached_response = json.loads(existing)
        
        if cached_response["status"] == "processing":
            # Another request with same key is still being processed
            raise HTTPException(status_code=409, detail="Request is being processed")
        
        return PaymentResponse(**cached_response)
    
    # Step 2: Mark as "processing" (prevents race conditions)
    processing_marker = json.dumps({"status": "processing"})
    # SET NX = Set only if Not eXists (atomic operation!)
    was_set = redis_client.set(cache_key, processing_marker, nx=True, ex=3600)
    
    if not was_set:
        # Another concurrent request already claimed this key
        raise HTTPException(status_code=409, detail="Duplicate request being processed")
    
    # Step 3: Process the payment (actual business logic)
    try:
        payment_id = process_payment_with_provider(
            payment.order_id, payment.amount, payment.currency
        )
        
        result = {
            "payment_id": payment_id,
            "status": "completed",
            "amount": payment.amount
        }
        
        # Step 4: Cache the result (24-hour TTL)
        redis_client.set(cache_key, json.dumps(result), ex=86400)
        return PaymentResponse(**result)
        
    except Exception as e:
        # Payment failed — remove the key so client can retry
        redis_client.delete(cache_key)
        raise HTTPException(status_code=500, detail=str(e))


def process_payment_with_provider(order_id, amount, currency):
    """Actual payment processing — only called ONCE per idempotency key."""
    # ... call Stripe/Razorpay ...
    return f"pay_{uuid.uuid4().hex[:12]}"
```

### Python — Database-Level Idempotency (PostgreSQL)

```python
import psycopg2
from psycopg2.extras import RealDictCursor

def create_order_idempotent(db_conn, idempotency_key: str, order_data: dict):
    """
    Create an order idempotently using database unique constraint.
    Uses the same transaction for idempotency check + business logic.
    """
    with db_conn.cursor(cursor_factory=RealDictCursor) as cur:
        try:
            # Attempt to insert the idempotency record
            # If key already exists, UNIQUE constraint prevents duplicate
            cur.execute("""
                INSERT INTO idempotency_keys (key, status, created_at)
                VALUES (%s, 'processing', NOW())
                ON CONFLICT (key) DO NOTHING
                RETURNING key
            """, (idempotency_key,))
            
            result = cur.fetchone()
            
            if result is None:
                # Key already exists — fetch cached result
                cur.execute("""
                    SELECT response_body, status 
                    FROM idempotency_keys 
                    WHERE key = %s
                """, (idempotency_key,))
                existing = cur.fetchone()
                
                if existing['status'] == 'completed':
                    return json.loads(existing['response_body'])
                else:
                    raise ConflictError("Request is still being processed")
            
            # First time seeing this key — process the order
            cur.execute("""
                INSERT INTO orders (id, user_id, total, status)
                VALUES (gen_random_uuid(), %s, %s, 'confirmed')
                RETURNING id, status
            """, (order_data['user_id'], order_data['total']))
            
            order = cur.fetchone()
            response = {"order_id": str(order['id']), "status": order['status']}
            
            # Store the response for future duplicate requests
            cur.execute("""
                UPDATE idempotency_keys 
                SET status = 'completed', response_body = %s
                WHERE key = %s
            """, (json.dumps(response), idempotency_key))
            
            db_conn.commit()
            return response
            
        except Exception as e:
            db_conn.rollback()
            raise
```

### Java — Idempotent REST Controller

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import org.springframework.data.redis.core.RedisTemplate;

import java.time.Duration;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@RestController
@RequestMapping("/api/v1/payments")
public class IdempotentPaymentController {
    
    private final RedisTemplate<String, String> redis;
    private final PaymentService paymentService;
    private final ObjectMapper objectMapper;
    
    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestBody PaymentRequest request,
            @RequestHeader("Idempotency-Key") String idempotencyKey) throws Exception {
        
        String cacheKey = "idempotency:" + idempotencyKey;
        
        // Step 1: Check for existing result
        String cached = redis.opsForValue().get(cacheKey);
        if (cached != null) {
            IdempotencyRecord record = objectMapper.readValue(cached, IdempotencyRecord.class);
            
            if ("processing".equals(record.getStatus())) {
                return ResponseEntity.status(409).build();  // Still processing
            }
            
            // Return cached result
            return ResponseEntity.ok(record.getResponse());
        }
        
        // Step 2: Claim the key atomically (SET NX EX)
        Boolean claimed = redis.opsForValue().setIfAbsent(
            cacheKey, 
            objectMapper.writeValueAsString(new IdempotencyRecord("processing", null)),
            Duration.ofHours(24)
        );
        
        if (Boolean.FALSE.equals(claimed)) {
            // Another request claimed it — race condition
            return ResponseEntity.status(409).build();
        }
        
        // Step 3: Process payment (only happens ONCE)
        try {
            PaymentResponse response = paymentService.charge(
                request.getOrderId(), 
                request.getAmount()
            );
            
            // Step 4: Store result for future duplicates
            IdempotencyRecord completed = new IdempotencyRecord("completed", response);
            redis.opsForValue().set(
                cacheKey, 
                objectMapper.writeValueAsString(completed),
                24, TimeUnit.HOURS
            );
            
            return ResponseEntity.ok(response);
            
        } catch (Exception e) {
            // Failed — delete key so client can retry
            redis.delete(cacheKey);
            throw e;
        }
    }
}
```

### Java — Database-Level Idempotency with Spring

```java
import org.springframework.transaction.annotation.Transactional;
import org.springframework.stereotype.Service;

@Service
public class IdempotentOrderService {
    
    private final IdempotencyKeyRepository idempotencyRepo;
    private final OrderRepository orderRepo;
    
    @Transactional  // Everything in ONE transaction — atomic!
    public OrderResponse createOrder(String idempotencyKey, CreateOrderRequest request) {
        
        // Try to find existing result
        return idempotencyRepo.findByKey(idempotencyKey)
            .map(record -> {
                if ("completed".equals(record.getStatus())) {
                    // Already processed — return cached result
                    return record.getResponseAs(OrderResponse.class);
                }
                throw new ConflictException("Request still processing");
            })
            .orElseGet(() -> {
                // First time — process the order
                
                // Insert idempotency key (unique constraint protects against races)
                IdempotencyKey key = new IdempotencyKey(idempotencyKey, "processing");
                idempotencyRepo.save(key);  // Throws on duplicate!
                
                // Create the order
                Order order = orderRepo.save(new Order(
                    request.getUserId(),
                    request.getItems(),
                    request.getTotal()
                ));
                
                // Store result
                OrderResponse response = new OrderResponse(order.getId(), "confirmed");
                key.setStatus("completed");
                key.setResponseBody(toJson(response));
                idempotencyRepo.save(key);
                
                return response;
            });
    }
}
```

---

## Infrastructure Examples

### PostgreSQL — Idempotency Table Schema

```sql
-- Idempotency key storage table
CREATE TABLE idempotency_keys (
    key VARCHAR(255) PRIMARY KEY,        -- The idempotency key
    status VARCHAR(20) NOT NULL,          -- 'processing', 'completed', 'failed'
    request_path VARCHAR(500),            -- Original request path
    request_body JSONB,                   -- Original request body (for validation)
    response_status INT,                  -- HTTP status of original response
    response_body JSONB,                  -- Cached response
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '24 hours'
);

-- Index for cleanup job
CREATE INDEX idx_idempotency_expires ON idempotency_keys (expires_at);

-- Cleanup expired keys (run periodically)
DELETE FROM idempotency_keys WHERE expires_at < NOW();
```

### Stripe's Idempotency Implementation

```
Stripe API requires Idempotency-Key header for all POST requests:

$ curl https://api.stripe.com/v1/charges \
  -H "Authorization: Bearer sk_test_..." \
  -H "Idempotency-Key: order-123-payment-1" \
  -d amount=2000 \
  -d currency=usd \
  -d source=tok_visa

Stripe's rules:
  - Keys expire after 24 hours
  - Same key + different request body → 400 error
  - Same key + same request body → returns original response
  - Keys are scoped per Stripe account
```

### Kafka — Idempotent Producer Configuration

```properties
# Kafka producer idempotency
enable.idempotence=true           # Enable exactly-once semantics
acks=all                          # Wait for all replicas to acknowledge
max.in.flight.requests.per.connection=5  # Max unacknowledged batches
retries=2147483647                # Infinite retries (safe with idempotency)

# How it works internally:
# Kafka assigns each producer a PID (Producer ID)
# Each message gets a sequence number
# Broker deduplicates: same PID + same sequence = already received
```

### Redis — Idempotency with Lua Script (Atomic)

```lua
-- idempotency_check.lua
-- Atomic check-and-set for idempotency
-- KEYS[1] = idempotency key
-- ARGV[1] = TTL in seconds
-- ARGV[2] = processing marker

local existing = redis.call('GET', KEYS[1])
if existing then
    return existing  -- Already exists, return cached
end

-- Set with NX (only if not exists) + TTL
local was_set = redis.call('SET', KEYS[1], ARGV[2], 'NX', 'EX', ARGV[1])
if was_set then
    return nil  -- Successfully claimed, proceed with processing
else
    return redis.call('GET', KEYS[1])  -- Race condition, return other's result
end
```

```python
# Using the Lua script from Python
import redis

r = redis.Redis()
idempotency_script = r.register_script(open('idempotency_check.lua').read())

def check_idempotency(key: str, ttl: int = 86400):
    result = idempotency_script(keys=[f"idem:{key}"], args=[ttl, "processing"])
    if result is None:
        return None  # First time — proceed with processing
    return json.loads(result)  # Duplicate — return cached result
```

---

## Real-World Example

### Stripe's Idempotency Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│           STRIPE'S IDEMPOTENCY IMPLEMENTATION                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  API Request with Idempotency-Key                                   │
│       │                                                             │
│       ▼                                                             │
│  ┌──────────────────────────────────────┐                          │
│  │  1. Look up key in idempotency store  │                          │
│  └──────────┬───────────────────────────┘                          │
│             │                                                       │
│        Found?                                                       │
│        ├── YES → Return stored response (HTTP status + body)        │
│        │         (with header: Idempotent-Replayed: true)           │
│        │                                                            │
│        └── NO → Continue processing                                 │
│             │                                                       │
│             ▼                                                       │
│  ┌──────────────────────────────────────┐                          │
│  │  2. Validate request body matches     │                          │
│  │     (same key + different body = 400) │                          │
│  └──────────┬───────────────────────────┘                          │
│             │                                                       │
│             ▼                                                       │
│  ┌──────────────────────────────────────┐                          │
│  │  3. Lock key (prevent concurrent      │                          │
│  │     processing of same key)           │                          │
│  └──────────┬───────────────────────────┘                          │
│             │                                                       │
│             ▼                                                       │
│  ┌──────────────────────────────────────┐                          │
│  │  4. Process charge (call bank, etc.)  │                          │
│  └──────────┬───────────────────────────┘                          │
│             │                                                       │
│             ▼                                                       │
│  ┌──────────────────────────────────────┐                          │
│  │  5. Store response + unlock key       │                          │
│  │     (key expires after 24 hours)      │                          │
│  └──────────────────────────────────────┘                          │
│                                                                     │
│  Key properties:                                                    │
│  - Keys scoped per API key (not global)                             │
│  - TTL: 24 hours                                                    │
│  - Body mismatch = error (prevent misuse)                           │
│  - Response includes "Idempotent-Replayed: true" header             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Amazon's Order Processing

```
Amazon ensures orders are never duplicated:

1. User clicks "Place Order"
2. Client generates: Idempotency-Key = "user-123-cart-abc-1685000000"
3. If network fails and client retries:
   - Same key → same order (not duplicate!)
4. User accidentally clicks twice:
   - Same key → same order (not two orders!)

Key structure: {user_id}-{cart_id}-{timestamp}
This ensures one order per checkout session, regardless of retries.
```

---

## Common Mistakes / Pitfalls

### 1. Checking Idempotency Key AFTER Processing

```
❌ BAD (race-prone):
   1. Process payment ← DANGEROUS! Already charged!
   2. Check if key exists
   3. If exists → refund? (Too late!)

✅ GOOD:
   1. Check if key exists (atomically claim it)
   2. If exists → return cached result
   3. If new → process payment
   4. Store result
```

### 2. Non-Atomic Check-and-Set

```
❌ BAD (race condition):
   if not redis.exists(key):     # ← Thread A checks
       # Thread B also checks here (not exists yet!)
       redis.set(key, "processing")  # Both threads set!
       process_payment()              # Both process! DOUBLE CHARGE!

✅ GOOD (atomic):
   was_set = redis.set(key, "processing", nx=True)  # Atomic!
   if was_set:
       process_payment()  # Only ONE thread gets here
   else:
       return cached_result()
```

### 3. No TTL on Idempotency Keys

```
❌ BAD: Keys stored forever
   → Database/Redis grows infinitely
   → Old keys consuming storage forever

✅ GOOD: Keys expire after 24-48 hours
   → After TTL, same key can be reused
   → Keeps storage bounded
```

### 4. Deleting Key on Failure Without Allowing Retry

```
❌ BAD: Payment fails → key marked "failed" → client can never retry
   → User's order is stuck permanently!

✅ GOOD: Payment fails → key deleted or marked "retriable"
   → Client can retry with same key
   → Eventually succeeds when service recovers
```

### 5. Key Too Broad or Too Narrow

```
❌ TOO BROAD: Key = "user-123"
   → User can only make ONE payment ever!

❌ TOO NARROW: Key = random UUID (client generates new one each retry)
   → Retries create duplicates (defeats the purpose!)

✅ JUST RIGHT: Key = "order-456-payment-attempt"
   → One payment per order per checkout session
   → Retries use same key (safe!)
```

---

## When to Use / When NOT to Use

### ✅ Use Idempotency When

| Scenario | Key Strategy |
|----------|-------------|
| Payment processing | `{order_id}-{payment_method}` |
| Order creation | `{user_id}-{cart_hash}-{timestamp}` |
| Email/SMS sending | `{template}-{recipient}-{trigger_event}` |
| Webhook delivery | `{event_id}` |
| Database writes from queues | `{message_id}` |
| API mutations (POST/PATCH) | Client-generated UUID in header |
| Fund transfers | `{from}-{to}-{amount}-{reference}` |

### ❌ When NOT Needed

| Scenario | Why |
|----------|-----|
| GET requests (reads) | Already idempotent by nature |
| PUT requests (full replace) | Already idempotent (sets to same state) |
| DELETE requests | Already idempotent (deleted stays deleted) |
| Caching writes (SET key=val) | Overwriting is safe |
| Logging/metrics | Duplicates are acceptable |

---

## Key Takeaways

- **Idempotency means "safe to repeat"** — same operation N times = same result as 1 time.
- **Use idempotency keys** for non-idempotent operations (payments, order creation, email sending).
- **The check MUST be atomic** — use `SET NX` in Redis or `INSERT ... ON CONFLICT` in PostgreSQL.
- **Store the response, not just "seen"** — duplicates should return the SAME response as the original.
- **Keys should expire** — 24-48 hours is typical. Don't store forever.
- **Handle the "processing" state** — what if a duplicate arrives while the first request is still running?
- **Business-derived keys > random UUIDs** — they naturally prevent duplicates even across client sessions.

---

## What's Next?

Idempotency protects individual operations. But what about the overall user experience when parts of your system are failing? How do you keep core features working while non-essential ones are down? That's **Graceful Degradation** — see [08-graceful-degradation.md](./08-graceful-degradation.md).
