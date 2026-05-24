# ACID vs BASE — Consistency Models for Databases

> **What you'll learn**: The two fundamental approaches to database consistency — ACID (strict guarantees) and BASE (relaxed guarantees for availability), when each model is appropriate, how they relate to the CAP theorem, and how real systems make this trade-off.

---

## Real-Life Analogy

### ACID = Bank Transfer
When you transfer ₹1000 from Account A to Account B:
- **Either BOTH happen** (A loses ₹1000 AND B gains ₹1000) — or **NEITHER happens**
- You can **never** see a state where A lost money but B didn't receive it
- Once confirmed, the transfer is **permanent** — even if the bank's server crashes
- Two people transferring simultaneously get **correct results** (no double-spend)

### BASE = Social Media Likes
When you like a post:
- The like counter might show "999" to you and "1000" to someone else for a few seconds
- **Eventually** everyone sees the same number
- It's OK if it's temporarily "wrong" — no one loses money
- The system **never goes down** to ensure consistency — it stays available

**The question every architect must answer**: "Is my data more like bank transfers or social media likes?"

---

## Core Concept Explained Step-by-Step

### ACID Properties

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ACID PROPERTIES                                │
├───────────────┬─────────────────────────────────────────────────────┤
│               │                                                      │
│  ATOMICITY    │  All-or-nothing. A transaction either completes     │
│  (A)          │  entirely or has no effect at all.                   │
│               │  Like: "Undo everything if ANY step fails"           │
│               │                                                      │
├───────────────┼─────────────────────────────────────────────────────┤
│               │                                                      │
│  CONSISTENCY  │  Database moves from one valid state to another.    │
│  (C)          │  All rules/constraints are satisfied.               │
│               │  Like: "Balance can never be negative"               │
│               │                                                      │
├───────────────┼─────────────────────────────────────────────────────┤
│               │                                                      │
│  ISOLATION    │  Concurrent transactions don't interfere with       │
│  (I)          │  each other. Each sees a consistent snapshot.       │
│               │  Like: "Pretend you're the only user"                │
│               │                                                      │
├───────────────┼─────────────────────────────────────────────────────┤
│               │                                                      │
│  DURABILITY   │  Once committed, data survives crashes, power       │
│  (D)          │  outages, and disk failures.                        │
│               │  Like: "Carved in stone — can't be lost"            │
│               │                                                      │
└───────────────┴─────────────────────────────────────────────────────┘
```

### ACID in Action — Bank Transfer

```
BEGIN TRANSACTION;
  Step 1: Read Alice's balance = ₹5000 ✓
  Step 2: Deduct ₹1000 from Alice → ₹4000 ✓
  Step 3: Read Bob's balance = ₹3000 ✓
  Step 4: Add ₹1000 to Bob → ₹4000 ✓
COMMIT;

ATOMICITY: If Step 4 fails → Steps 1-3 are ROLLED BACK
CONSISTENCY: Total money before (₹8000) = Total money after (₹8000)
ISOLATION: Another transaction reading balances during this sees
           either the before state OR the after state, never in-between
DURABILITY: After COMMIT, even if server crashes 1ms later, the
            transfer is permanent (written to WAL on disk)
```

### BASE Properties

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BASE PROPERTIES                                │
├───────────────────────┬─────────────────────────────────────────────┤
│                       │                                              │
│  BASICALLY AVAILABLE  │  System guarantees availability.            │
│  (BA)                 │  Every request gets a response              │
│                       │  (might not be the latest data).            │
│                       │                                              │
├───────────────────────┼─────────────────────────────────────────────┤
│                       │                                              │
│  SOFT STATE           │  System state may change over time,         │
│  (S)                  │  even without new input (due to             │
│                       │  background replication catching up).       │
│                       │                                              │
├───────────────────────┼─────────────────────────────────────────────┤
│                       │                                              │
│  EVENTUALLY           │  Given enough time without new updates,     │
│  CONSISTENT (E)       │  all replicas will converge to the          │
│                       │  same value. (But not immediately!)         │
│                       │                                              │
└───────────────────────┴─────────────────────────────────────────────┘
```

### BASE in Action — Social Media Like Counter

```
User likes post at 10:00:00:

Time        │ Server A (US)    │ Server B (Europe) │ Server C (Asia)
────────────┼──────────────────┼───────────────────┼────────────────
10:00:00    │ likes = 1000     │ likes = 999       │ likes = 999
            │ (just received   │ (replication lag)  │ (replication lag)
            │  the like)       │                    │
10:00:01    │ likes = 1000     │ likes = 1000      │ likes = 999
            │                  │ (caught up!)       │ (still lagging)
10:00:02    │ likes = 1000     │ likes = 1000      │ likes = 1000
            │                  │                    │ (caught up!)

EVENTUALLY: All servers show 1000 (consistent!)
But for 2 seconds, different users saw different numbers.
That's OK for likes. NOT OK for bank balances.
```

---

## How It Works Internally

### Isolation Levels (ACID Detail)

```
┌─────────────────────────────────────────────────────────────────────┐
│              SQL ISOLATION LEVELS (from weakest to strongest)         │
├────────────────────┬────────────────────────────────────────────────┤
│ READ UNCOMMITTED   │ Can see uncommitted changes from other txns    │
│                    │ Problem: Dirty reads ❌                         │
│                    │ Speed: Fastest                                  │
├────────────────────┼────────────────────────────────────────────────┤
│ READ COMMITTED     │ Only sees committed data                       │
│ (PostgreSQL default)│ Problem: Non-repeatable reads ❌               │
│                    │ Speed: Fast                                     │
├────────────────────┼────────────────────────────────────────────────┤
│ REPEATABLE READ    │ Same query in same txn → same results          │
│                    │ Problem: Phantom reads ❌                       │
│                    │ Speed: Moderate                                 │
├────────────────────┼────────────────────────────────────────────────┤
│ SERIALIZABLE       │ Transactions behave as if executed one-by-one  │
│                    │ No anomalies at all ✅                          │
│                    │ Speed: Slowest (conflicts → retries)           │
└────────────────────┴────────────────────────────────────────────────┘

Trade-off visualization:
    
CONSISTENCY ─────────────────────────────────────────── HIGH
READ UNCOMMITTED → READ COMMITTED → REPEATABLE READ → SERIALIZABLE
PERFORMANCE ─────────────────────────────────────────── LOW
```

### Eventual Consistency — How It Works

```
DYNAMO-STYLE (DynamoDB, Cassandra):
─────────────────────────────────────────────────
Write arrives:
  1. Write to node A (coordinator)
  2. Replicate to node B (async)
  3. Replicate to node C (async)
  4. Return "success" to client IMMEDIATELY

Read arrives (at node C, before replication completes):
  → Returns STALE data! (node C hasn't received the update yet)

After replication completes (milliseconds to seconds):
  → All nodes have same data = EVENTUALLY consistent

ANTI-ENTROPY (background repair):
  Nodes periodically compare their data using Merkle trees
  If differences found → sync missing data
  Guarantees convergence even if replication messages are lost
```

### Consistency Spectrum

```
STRONG ◀────────────────────────────────────────────────────▶ WEAK

LINEARIZABLE │ SEQUENTIAL │ CAUSAL │ EVENTUAL │ NONE
             │            │        │          │
             │            │        │          │
 "Everyone   │ "Operations│"If A   │ "All     │"No
  sees the   │  appear in │caused  │ copies   │guarantees"
  same value │  SOME      │B, then │ will     │
  at the     │  consistent│everyone│ EVENTUALLY│
  same time" │  order"    │sees A  │ agree"   │
             │            │before  │          │
             │            │B"      │          │

USE CASE:    │            │        │          │
Banking      │ Messaging  │Social  │ DNS      │ Best-effort
Inventory    │ Sequences  │feeds   │ Caches   │ logging
```

---

## Comparison Table

```
┌────────────────────────────────────────────────────────────────────┐
│            ACID vs BASE — Detailed Comparison                       │
├──────────────────────┬──────────────────────┬──────────────────────┤
│ Aspect               │ ACID                 │ BASE                 │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Consistency          │ Immediate (strong)   │ Eventual (relaxed)   │
│ Availability         │ May sacrifice for    │ Always available     │
│                      │ consistency          │                      │
│ Transactions         │ Full multi-statement │ Single-record atomic │
│ Scaling              │ Vertical (harder to  │ Horizontal (designed │
│                      │ distribute)          │ for distribution)    │
│ Performance          │ Lower (locking,      │ Higher (no locks,    │
│                      │ coordination)        │ no coordination)     │
│ Complexity           │ Database handles it  │ Application handles  │
│                      │                      │ conflicts            │
│ Data Model           │ Relational (SQL)     │ NoSQL (flexible)     │
│ Best For             │ Financial, inventory │ Social, analytics,   │
│                      │ bookings, healthcare │ IoT, content         │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Databases            │ PostgreSQL, MySQL,   │ Cassandra, DynamoDB, │
│                      │ Oracle, SQL Server   │ MongoDB*, CouchDB    │
└──────────────────────┴──────────────────────┴──────────────────────┘
* MongoDB supports ACID for multi-doc transactions since v4.0
```

---

## Code Examples

### Python — ACID Transaction (PostgreSQL)

```python
import psycopg2
from psycopg2 import errors

conn = psycopg2.connect("host=localhost dbname=bank user=admin password=secret")

def transfer_money(from_account, to_account, amount):
    """ACID-compliant money transfer — all-or-nothing"""
    try:
        with conn.cursor() as cursor:
            # Start transaction (implicit with psycopg2)
            
            # Lock rows to prevent concurrent modifications (ISOLATION)
            cursor.execute(
                "SELECT balance FROM accounts WHERE id = %s FOR UPDATE",
                (from_account,)
            )
            sender_balance = cursor.fetchone()[0]
            
            # CONSISTENCY check: sufficient balance?
            if sender_balance < amount:
                conn.rollback()
                raise ValueError("Insufficient balance!")
            
            # Deduct from sender
            cursor.execute(
                "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                (amount, from_account)
            )
            
            # Add to receiver
            cursor.execute(
                "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                (amount, to_account)
            )
            
            # ATOMICITY: Both updates committed together
            conn.commit()
            # DURABILITY: After commit, survives crashes (WAL)
            print(f"Transferred ₹{amount} from {from_account} to {to_account}")
            
    except errors.SerializationFailure:
        # ISOLATION conflict — another transaction modified same rows
        conn.rollback()
        print("Conflict detected, retrying...")
        transfer_money(from_account, to_account, amount)  # Retry
    except Exception as e:
        conn.rollback()  # ATOMICITY: undo everything on any failure
        raise

# Use SERIALIZABLE isolation for strictest guarantees
conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_SERIALIZABLE)
transfer_money("ACC001", "ACC002", 1000)
```

### Python — BASE (Eventually Consistent with DynamoDB)

```python
import boto3
from datetime import datetime
import time

dynamodb = boto3.resource('dynamodb', region_name='ap-south-1')
table = dynamodb.Table('SocialPosts')

def like_post(post_id, user_id):
    """BASE model: available immediately, eventually consistent"""
    
    # Optimistic update — no locking, no transaction
    # Might overcount briefly if two users like simultaneously
    table.update_item(
        Key={'post_id': post_id},
        UpdateExpression='SET like_count = like_count + :inc, '
                        'last_liked_at = :time',
        ExpressionAttributeValues={
            ':inc': 1,
            ':time': datetime.now().isoformat()
        },
        # Conditional: only if user hasn't already liked (idempotent)
        ConditionExpression='NOT contains(liked_by, :user)',
        ExpressionAttributeValues={
            ':inc': 1,
            ':time': datetime.now().isoformat(),
            ':user': user_id
        }
    )
    # Returns immediately — other regions will see this eventually

def get_like_count(post_id, strong=False):
    """Read with configurable consistency"""
    response = table.get_item(
        Key={'post_id': post_id},
        ConsistentRead=strong  # True = strong consistency (slower)
                               # False = eventual consistency (faster, default)
    )
    return response['Item']['like_count']

# Eventually consistent read (fast, might be stale)
count = get_like_count("post_123", strong=False)  # ~5ms

# Strongly consistent read (slower, guaranteed latest)
count = get_like_count("post_123", strong=True)   # ~15ms
```

### Java — Choosing Consistency Level

```java
import java.sql.*;

public class ConsistencyExample {
    
    // ACID: Booking a hotel room (must be consistent!)
    public boolean bookRoom(Connection conn, int roomId, int userId) 
            throws SQLException {
        conn.setAutoCommit(false);
        conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
        
        try {
            // Check availability (with row lock)
            PreparedStatement check = conn.prepareStatement(
                "SELECT available FROM rooms WHERE id = ? FOR UPDATE");
            check.setInt(1, roomId);
            ResultSet rs = check.executeQuery();
            
            if (rs.next() && rs.getBoolean("available")) {
                // Book the room
                PreparedStatement book = conn.prepareStatement(
                    "UPDATE rooms SET available = false, booked_by = ? WHERE id = ?");
                book.setInt(1, userId);
                book.setInt(2, roomId);
                book.executeUpdate();
                
                // Create booking record
                PreparedStatement insert = conn.prepareStatement(
                    "INSERT INTO bookings (room_id, user_id, booked_at) VALUES (?, ?, NOW())");
                insert.setInt(1, roomId);
                insert.setInt(2, userId);
                insert.executeUpdate();
                
                conn.commit();  // ATOMIC: both succeed or both fail
                return true;
            }
            
            conn.rollback();
            return false;  // Room already taken
            
        } catch (SQLException e) {
            conn.rollback();
            throw e;
        }
    }
}
```

---

## Decision Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│              WHICH MODEL DO I NEED?                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Ask these questions:                                                │
│                                                                       │
│  1. "Can two users see different values for a few seconds?"         │
│     YES → BASE is fine (social feeds, analytics, recommendations)   │
│     NO  → Need ACID (banking, inventory, booking)                   │
│                                                                       │
│  2. "Does incorrect data cause MONEY LOSS or SAFETY issues?"        │
│     YES → ACID (payments, medical records, flight bookings)         │
│     NO  → BASE is fine (view counts, suggestions, notifications)   │
│                                                                       │
│  3. "Do I need to update MULTIPLE RECORDS atomically?"              │
│     YES → ACID with transactions                                    │
│     NO  → BASE (single-record updates are atomic even in NoSQL)    │
│                                                                       │
│  4. "Is availability MORE important than consistency?"               │
│     YES → BASE (e-commerce product page, social media, search)     │
│     NO  → ACID (checkout, payment, inventory deduction)            │
│                                                                       │
│  REAL-WORLD HYBRID:                                                  │
│  Most apps use BOTH! Example e-commerce:                            │
│  • Product catalog: BASE (stale stock count OK for browsing)        │
│  • Shopping cart: BASE (eventual sync across devices)               │
│  • Checkout/Payment: ACID (must deduct inventory atomically)        │
│  • Order history: BASE (eventual consistency fine)                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Amazon — Both ACID and BASE in One Transaction

```
┌─────────────────────────────────────────────────────────────────┐
│              Amazon "Buy Now" — Hybrid Consistency                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Step 1: "Add to Cart" (BASE)                                   │
│  → Eventually consistent across devices                         │
│  → Cart stored in DynamoDB (highly available)                    │
│                                                                   │
│  Step 2: "Place Order" (ACID)                                   │
│  → Transaction: deduct inventory + charge payment + create order │
│  → If ANY fails → rollback entire operation                     │
│  → Uses ACID database for inventory (prevents overselling)      │
│                                                                   │
│  Step 3: "Order Confirmation Email" (BASE)                      │
│  → Sent asynchronously via message queue                        │
│  → Might arrive seconds later (eventually)                      │
│  → System doesn't wait for email to confirm order               │
│                                                                   │
│  Step 4: "Shipping Update" (BASE)                               │
│  → Tracking info propagates eventually                          │
│  → Different services see update at different times             │
│                                                                   │
│  KEY INSIGHT: ACID where money/inventory is at stake            │
│               BASE everywhere else for speed + availability     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using ACID for everything | Limits scalability, adds latency | Only use ACID where correctness is critical |
| Using BASE for financial data | Inconsistency = money loss, legal issues | Always use ACID for money |
| "Eventually" means "whenever" | Lag can be seconds to minutes | Monitor consistency lag, set SLOs |
| Not handling conflicts in BASE | Silent data loss or corruption | Implement conflict resolution (LWW, CRDT) |
| Ignoring isolation levels | Default might allow anomalies | Explicitly set isolation level per transaction |
| Assuming NoSQL = no ACID | MongoDB 4.0+, DynamoDB have transactions | Know your database's capabilities |
| Not testing failure scenarios | Assumptions break under network partitions | Test: what happens if write succeeds locally but replication fails? |

---

## When to Use / When NOT to Use

### ✅ Use ACID When:
- **Financial transactions** (banking, payments, billing)
- **Inventory management** (prevent overselling)
- **Booking systems** (hotel rooms, flights, appointments)
- **User registration** (unique email/username enforcement)
- **Healthcare records** (data integrity is life-critical)
- Any operation where **incorrect data causes harm**

### ✅ Use BASE When:
- **Social media** (likes, comments, view counts)
- **Analytics and metrics** (approximate counts are fine)
- **Product catalogs** (slightly stale data is OK)
- **Search indexes** (near-real-time is sufficient)
- **Notifications** (delayed delivery acceptable)
- **Session/preference storage** (eventual sync is fine)
- Any system where **availability > consistency**

---

## Key Takeaways

1. **ACID guarantees correctness** — transactions are atomic, state is always consistent, concurrent operations are isolated, and committed data is durable.
2. **BASE prioritizes availability** — the system is always responsive, state may be temporarily inconsistent, but will eventually converge.
3. **Most real systems use BOTH** — ACID for critical operations (payments) and BASE for everything else (browsing, social, analytics).
4. **The CAP theorem forces this choice** for distributed systems — you can't have perfect consistency AND availability during network partitions.
5. **Isolation levels** let you tune the ACID trade-off — SERIALIZABLE for strictest guarantees, READ COMMITTED for a practical balance.
6. **Eventual consistency is a spectrum** — from milliseconds (DynamoDB) to minutes (DNS propagation). Know YOUR system's lag.
7. **The right choice depends on the business question**: "What happens if this data is wrong for 5 seconds?" If the answer is "nothing bad" → BASE. If it's "someone loses money" → ACID.

---

## What's Next?

Next, we'll explore **Database Connection Pooling & Query Optimization** (Chapter 9.14), where you'll learn how to manage database connections efficiently and make queries run faster through optimization techniques.
