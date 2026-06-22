# 1.5 — ACID vs BASE — Two Philosophies of Data 🟡⭐

> **"Do you want your data to be always correct? Or always available? That's the fundamental trade-off every system must make."**

---

## 📌 What You'll Learn

- **ACID** properties — the gold standard of data integrity
- What each letter really means (with real-world examples)
- **BASE** — the alternative philosophy for distributed systems
- When to choose ACID vs BASE
- How real databases implement these guarantees
- The connection to the **CAP Theorem** (preview for Chapter 1.6)

---

## 1. The Two Schools of Thought

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  TRADITIONAL WORLD                    DISTRIBUTED WORLD          │
│  (Banks, Healthcare, ERP)             (Social Media, IoT, CDNs) │
│                                                                  │
│  ┌──────────────────────┐            ┌────────────────────────┐ │
│  │       A C I D         │            │        B A S E         │ │
│  │                       │            │                        │ │
│  │ • Data is ALWAYS      │            │ • Data is EVENTUALLY   │ │
│  │   correct             │            │   correct              │ │
│  │ • Transactions are    │            │ • Availability over    │ │
│  │   all-or-nothing      │            │   consistency          │ │
│  │ • Strong consistency  │            │ • Soft state           │ │
│  │ • Vertical scaling    │            │ • Horizontal scaling   │ │
│  │                       │            │                        │ │
│  │ Oracle, PostgreSQL,   │            │ Cassandra, DynamoDB,   │ │
│  │ MySQL, SQL Server     │            │ CouchDB, Riak          │ │
│  └──────────────────────┘            └────────────────────────┘ │
│                                                                  │
│          "I'd rather be                "I'd rather be            │
│           correct"                      available"               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. ACID — The Gold Standard ⭐

ACID is a set of **four guarantees** that a database transaction provides. If a database is "ACID-compliant," it means every transaction is reliable, even in case of errors, power failures, or crashes.

```
A = Atomicity       → All or Nothing
C = Consistency     → Valid State to Valid State
I = Isolation       → Transactions Don't Interfere
D = Durability      → Once Committed, It's Permanent
```

---

### A — Atomicity (All or Nothing) 💣

> **Definition**: A transaction is an **indivisible unit**. Either ALL operations within it succeed, or NONE of them do. There is no "half-done" state.

#### The Classic Bank Transfer Example

```
TRANSACTION: Transfer ₹5,000 from Account A to Account B

  Step 1: Debit ₹5,000 from Account A  (A: 10,000 → 5,000)
  Step 2: Credit ₹5,000 to Account B   (B: 20,000 → 25,000)

┌──────────────────────────────────────────────────────────────┐
│  SCENARIO 1: Both steps succeed ✅                           │
│  → COMMIT: Both changes are permanent                       │
│  → A = 5,000  |  B = 25,000  |  Total = 30,000 ✅           │
│                                                              │
│  SCENARIO 2: Step 1 succeeds, Step 2 FAILS 💥               │
│  → WITHOUT Atomicity:                                       │
│    A = 5,000  |  B = 20,000  |  Total = 25,000 ❌            │
│    ₹5,000 VANISHED! 💀                                      │
│                                                              │
│  → WITH Atomicity (ROLLBACK):                               │
│    A = 10,000 |  B = 20,000  |  Total = 30,000 ✅            │
│    Both changes reversed. Money is safe.                    │
└──────────────────────────────────────────────────────────────┘
```

#### How It Works Internally

```
BEGIN TRANSACTION;

  -- Write to WAL: "About to debit A"
  UPDATE accounts SET balance = balance - 5000 WHERE id = 'A';
  -- Write to WAL: "About to credit B"
  UPDATE accounts SET balance = balance + 5000 WHERE id = 'B';

  -- Everything OK?
  IF (no errors) THEN
    COMMIT;   -- Mark transaction as committed in WAL
              -- Changes are permanent
  ELSE
    ROLLBACK; -- Undo ALL changes using WAL/Undo log
              -- As if the transaction never happened
  END IF;
```

#### SQL Example

```sql
-- PostgreSQL / SQL Server / MySQL (InnoDB)
BEGIN;

UPDATE accounts SET balance = balance - 5000 WHERE id = 'A';
UPDATE accounts SET balance = balance + 5000 WHERE id = 'B';

-- Check constraint: balance cannot go negative
-- If any constraint fails → automatic ROLLBACK

COMMIT;

-- If crash happens between the two UPDATEs:
-- Database recovers using WAL → rolls back incomplete transaction
-- Atomicity guaranteed!
```

---

### C — Consistency (Valid State → Valid State)

> **Definition**: A transaction takes the database from one **valid state** to another valid state. All defined rules (constraints, triggers, cascades) must be satisfied after the transaction.

#### What "Consistent" Means

```
DATABASE RULES (Constraints):
  1. Balance must be >= 0                     (CHECK constraint)
  2. Every order must have a valid customer   (FOREIGN KEY)
  3. Email must be unique                     (UNIQUE constraint)
  4. Employee salary must be > 0              (CHECK constraint)
  5. Total credits = Total debits             (Business rule)

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  VALID STATE ───[Transaction]──→ VALID STATE                │
│                                                              │
│  State A:                       State B:                    │
│  Account A = 10,000             Account A = 5,000           │
│  Account B = 20,000             Account B = 25,000          │
│  Total    = 30,000              Total    = 30,000           │
│  All balances ≥ 0 ✅             All balances ≥ 0 ✅         │
│  Total unchanged ✅              Total unchanged ✅          │
│                                                              │
│  INVALID ATTEMPT:                                           │
│  Transfer 15,000 from A (which has only 10,000)            │
│  → Would make A = -5,000 → VIOLATES CHECK constraint      │
│  → Transaction REJECTED → Database stays in valid state    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### Types of Constraints That Enforce Consistency

| Constraint | What It Enforces | Example |
|-----------|-----------------|---------|
| **PRIMARY KEY** | Unique identifier for every row | `id INT PRIMARY KEY` |
| **FOREIGN KEY** | Referential integrity between tables | `customer_id REFERENCES customers(id)` |
| **NOT NULL** | Column must have a value | `email VARCHAR(100) NOT NULL` |
| **UNIQUE** | No duplicate values | `UNIQUE(email)` |
| **CHECK** | Custom validation rule | `CHECK(salary > 0)` |
| **DEFAULT** | Auto-fill when no value provided | `DEFAULT CURRENT_TIMESTAMP` |
| **TRIGGER** | Custom business logic | After INSERT, update inventory count |

```sql
CREATE TABLE accounts (
  id          INT PRIMARY KEY,
  owner_name  VARCHAR(100) NOT NULL,
  balance     DECIMAL(15,2) NOT NULL CHECK (balance >= 0),  -- Can't go negative
  currency    VARCHAR(3) NOT NULL DEFAULT 'INR',
  created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- This INSERT succeeds (valid state):
INSERT INTO accounts VALUES (1, 'Rahul', 10000.00, 'INR', NOW());

-- This INSERT FAILS (consistency violation):
INSERT INTO accounts VALUES (2, 'Priya', -500.00, 'INR', NOW());
-- ERROR: CHECK constraint "accounts_balance_check" violated
-- Database remains in valid state ✅
```

---

### I — Isolation (Transactions Don't Interfere) 🔒

> **Definition**: Even when multiple transactions run **simultaneously**, each one behaves as if it's the **only transaction** in the system. One transaction's incomplete changes are invisible to others.

#### The Problem Without Isolation

```
TIME    Transaction 1 (T1)              Transaction 2 (T2)
─────   ─────────────────────           ─────────────────────
t1      SELECT balance FROM A           
        → balance = 1000               
t2                                      SELECT balance FROM A
                                        → balance = 1000
t3      UPDATE A SET balance = 500      
        (debit 500)                    
t4                                      UPDATE A SET balance = 700
                                        (debit 300)
t5      COMMIT ✅                       
t6                                      COMMIT ✅

RESULT: balance = 700 ❌
EXPECTED: balance = 200 (1000 - 500 - 300)
T1's debit of 500 is LOST! This is called "Lost Update"
```

#### Isolation Levels (SQL Standard)

The SQL standard defines **four isolation levels**, each preventing different problems:

```
┌────────────────────────────────────────────────────────────────────────┐
│                    ISOLATION LEVELS (Weakest → Strongest)              │
├───────────────────┬──────────────┬──────────────┬─────────────────────┤
│                   │ Dirty Read   │ Non-Repeatable│ Phantom Read       │
│                   │              │ Read          │                    │
├───────────────────┼──────────────┼──────────────┼─────────────────────┤
│ READ UNCOMMITTED  │ ⚠️ Possible  │ ⚠️ Possible   │ ⚠️ Possible        │
│ (Chaos mode)      │              │              │                    │
├───────────────────┼──────────────┼──────────────┼─────────────────────┤
│ READ COMMITTED    │ ✅ Prevented │ ⚠️ Possible   │ ⚠️ Possible        │
│ (PostgreSQL default)│            │              │                    │
├───────────────────┼──────────────┼──────────────┼─────────────────────┤
│ REPEATABLE READ   │ ✅ Prevented │ ✅ Prevented  │ ⚠️ Possible        │
│ (MySQL default)   │              │              │ (✅ in PG via MVCC) │
├───────────────────┼──────────────┼──────────────┼─────────────────────┤
│ SERIALIZABLE      │ ✅ Prevented │ ✅ Prevented  │ ✅ Prevented       │
│ (Strongest safety)│              │              │                    │
└───────────────────┴──────────────┴──────────────┴─────────────────────┘
```

#### The Three Read Problems Explained

```
1. DIRTY READ (Reading uncommitted data)
   T1: UPDATE salary to 100000 (not committed yet)
   T2: SELECT salary → sees 100000
   T1: ROLLBACK (undoes the change)
   T2 read a value that NEVER EXISTED! 👻

2. NON-REPEATABLE READ (Same query, different result)
   T1: SELECT salary WHERE id=1 → 50000
   T2: UPDATE salary = 80000 WHERE id=1; COMMIT;
   T1: SELECT salary WHERE id=1 → 80000 ❌ (different from first read!)

3. PHANTOM READ (New rows appear between reads)
   T1: SELECT COUNT(*) FROM employees WHERE dept='Engineering' → 10
   T2: INSERT INTO employees (dept='Engineering'); COMMIT;
   T1: SELECT COUNT(*) FROM employees WHERE dept='Engineering' → 11 ❌
   A "phantom" row appeared!
```

#### Setting Isolation Level

```sql
-- PostgreSQL
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
-- your queries here
COMMIT;

-- MySQL
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- SQL Server
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

#### Default Isolation Levels by Database

| Database | Default Level | Notes |
|----------|--------------|-------|
| PostgreSQL | Read Committed | MVCC-based, Repeatable Read prevents phantoms too |
| MySQL (InnoDB) | Repeatable Read | Uses gap locking to prevent some phantoms |
| Oracle | Read Committed | Uses undo segments for MVCC |
| SQL Server | Read Committed | Can enable RCSI for MVCC-like behavior |
| SQLite | Serializable | Single-writer model |

---

### D — Durability (Once Committed, It's Permanent) 💾

> **Definition**: Once a transaction is **committed**, its changes survive **any subsequent failure** — power outage, crash, disk failure, natural disaster.

#### How Durability Works

```
┌──────────────────────────────────────────────────────────────┐
│                  HOW DURABILITY IS ACHIEVED                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. WRITE-AHEAD LOG (WAL)                                    │
│     Every change written to WAL BEFORE the data files        │
│     WAL is on durable storage (disk)                        │
│     → Crash? Replay WAL to recover                          │
│                                                              │
│  2. FSYNC / FLUSH TO DISK                                    │
│     On COMMIT, WAL is flushed to physical disk              │
│     Not just OS buffer — actual disk platter / SSD cells    │
│     → Even if OS crashes, data is on disk                   │
│                                                              │
│  3. CHECKPOINTS                                              │
│     Periodically flush dirty pages from memory to disk      │
│     → Reduces recovery time                                 │
│                                                              │
│  4. REPLICATION (extra level)                                │
│     Copy data to 2+ servers                                 │
│     → Disk explodes? Other server has the data             │
│                                                              │
│  5. BACKUP (ultimate safety net)                             │
│     Regular backups to separate location                    │
│     → Datacenter burns? Restore from backup                │
│                                                              │
└──────────────────────────────────────────────────────────────┘

TIMELINE:
  t1: BEGIN
  t2: UPDATE accounts SET balance = 500 WHERE id = 1
  t3: WAL record written to disk ← DATA IS SAFE
  t4: COMMIT ← Return "success" to client
  t5: 💥 POWER FAILURE
  t6: Database restarts
  t7: Reads WAL → Replays committed changes
  t8: balance = 500 ✅ (data recovered!)
```

#### Durability Configuration Levels

```
HIGHEST DURABILITY (safest, slowest):
  fsync = on (flush every commit to disk)
  synchronous_commit = on
  → Used for: Banking, Healthcare, Financial systems

MEDIUM DURABILITY (good trade-off):
  synchronous_commit = on
  Commit WAL in batches (group commit)
  → Used for: Most applications

RELAXED DURABILITY (fastest, small risk):
  synchronous_commit = off
  → Commits return before WAL hits disk
  → Risk: Lose last few milliseconds of commits on crash
  → Used for: Logging, analytics, non-critical data
```

```sql
-- PostgreSQL: Control durability per transaction
SET synchronous_commit = off;  -- Faster but tiny data loss risk
-- or
SET synchronous_commit = on;   -- Safest (default)
```

---

## 3. ACID in Action — Complete Example

```sql
-- E-Commerce: Place an order (must be ALL or NOTHING)

BEGIN TRANSACTION;

  -- 1. Create the order
  INSERT INTO orders (customer_id, total, status)
  VALUES (1001, 2999.00, 'confirmed');
  -- Let's say this gets order_id = 5001

  -- 2. Add order items
  INSERT INTO order_items (order_id, product_id, quantity, price)
  VALUES (5001, 42, 1, 2999.00);

  -- 3. Reduce product stock
  UPDATE products 
  SET stock = stock - 1 
  WHERE id = 42 AND stock >= 1;  -- CHECK: enough stock?

  -- 4. Charge customer's wallet
  UPDATE wallets 
  SET balance = balance - 2999.00 
  WHERE customer_id = 1001 AND balance >= 2999.00;  -- CHECK: enough balance?

  -- If ANY step fails (stock = 0, insufficient balance, constraint violation):
  -- → ROLLBACK everything. No order, no stock change, no charge.
  
  -- If ALL steps succeed:
  -- → COMMIT. Order placed, stock reduced, wallet charged.

COMMIT;

-- ACID GUARANTEES:
-- Atomicity:    All 4 steps succeed or all 4 are undone
-- Consistency:  Stock can't go negative, balance can't go negative
-- Isolation:    Another user buying the same product won't interfere
-- Durability:   Once committed, order survives any crash
```

---

## 4. BASE — The Alternative Philosophy

### Why BASE Exists

```
PROBLEM WITH ACID IN DISTRIBUTED SYSTEMS:

  Imagine your database is spread across 3 data centers:
  
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  US-East  │     │  EU-West  │     │ AP-South  │
  │  Node 1   │────→│  Node 2   │────→│  Node 3   │
  └──────────┘     └──────────┘     └──────────┘
  
  For ACID consistency: ALL 3 nodes must agree on EVERY write
  → Network between US and India has 200ms latency
  → Each write takes at least 200ms × multiple round trips
  → Under network partition: System STOPS (can't guarantee consistency)
  
  For a social media "like" button:
  → Is it worth stopping the entire system to guarantee
    that a "like count" is perfectly accurate at all times?
  → NO! It's OK if the count shows 99 instead of 100 for a few seconds.
  
  THIS is where BASE comes in.
```

### What BASE Stands For

```
B A = Basically Available
S   = Soft State
E   = Eventually Consistent
```

---

### BA — Basically Available

> **Definition**: The system **always responds** to requests, even during failures. It might return stale data or an approximation, but it never says "system unavailable."

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ACID (during network partition):                           │
│  "Sorry, I can't process your request right now.            │
│   I need to verify with all nodes first."                   │
│  → System becomes UNAVAILABLE ❌                            │
│                                                              │
│  BASE (during network partition):                           │
│  "Here's the data I have on THIS node.                      │
│   It might be slightly outdated, but I'm responding."       │
│  → System stays AVAILABLE ✅ (but maybe stale)              │
│                                                              │
│  Example:                                                    │
│  Amazon product page during a network glitch:               │
│  → Still shows products (maybe cached/slightly stale prices)│
│  → Better than showing "503 Service Unavailable"            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### S — Soft State

> **Definition**: The state of the system **may change over time**, even without new input. Data can be in an intermediate state as it propagates across nodes.

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  HARD STATE (ACID):                                         │
│  Data is always in a definitive, known state                │
│  balance = 5000 (guaranteed, right now, everywhere)         │
│                                                              │
│  SOFT STATE (BASE):                                         │
│  Data may be in flux as updates propagate                   │
│                                                              │
│  Time     Node 1          Node 2          Node 3            │
│  ─────    ──────────      ──────────      ──────────        │
│  t1       likes = 100     likes = 100     likes = 100       │
│  t2       likes = 101     likes = 100     likes = 100       │
│           (write here)    (not yet)       (not yet)         │
│  t3       likes = 101     likes = 101     likes = 100       │
│                           (propagated)    (not yet)         │
│  t4       likes = 101     likes = 101     likes = 101       │
│                                           (propagated) ✅    │
│                                                              │
│  During t2-t3: state is "soft" — different nodes disagree  │
│  At t4: all nodes agree again (eventually consistent)      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### E — Eventually Consistent

> **Definition**: If no new updates are made, **all replicas will eventually converge** to the same value. There is no guarantee of WHEN — just that it WILL happen.

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  STRONG CONSISTENCY (ACID):                                  │
│  After a write, ALL subsequent reads return the new value   │
│  (on ANY node)                                              │
│                                                              │
│  EVENTUAL CONSISTENCY (BASE):                                │
│  After a write, reads MIGHT return old value temporarily    │
│  Eventually (milliseconds to seconds), all reads return     │
│  the new value                                              │
│                                                              │
│  TIMELINE:                                                   │
│                                                              │
│  Write ────→ [Inconsistency Window] ────→ Consistent ✅     │
│              (stale reads possible)                         │
│              Usually: 10ms - 5 seconds                      │
│                                                              │
│  Real Examples:                                              │
│  • DNS propagation: up to 48 hours                          │
│  • Social media likes: a few seconds                        │
│  • CDN cache: configurable TTL                              │
│  • DynamoDB: milliseconds (with DAX cache: instant)         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### Consistency Spectrum

```
STRONGEST ◄──────────────────────────────────────► WEAKEST

Linearizable → Sequential → Causal → Eventual → No guarantee
    │              │           │          │
    │              │           │          └── DynamoDB (default)
    │              │           │              Cassandra (ONE)
    │              │           │
    │              │           └── MongoDB (causal sessions)
    │              │               CockroachDB (within region)
    │              │
    │              └── PostgreSQL (single node)
    │
    └── Spanner (global), CockroachDB (serializable)
         FaunaDB

More consistency = More latency & Less availability
Less consistency = Less latency & More availability
```

---

## 5. ACID vs BASE — Direct Comparison

| Property | ACID | BASE |
|----------|------|------|
| **Full Form** | Atomicity, Consistency, Isolation, Durability | Basically Available, Soft state, Eventually consistent |
| **Philosophy** | Correctness above all | Availability above all |
| **Consistency** | Strong (immediate) | Eventual (delayed) |
| **Availability** | May sacrifice during partitions | Always responds |
| **Scaling** | Primarily vertical | Designed for horizontal |
| **Complexity** | Simpler for developers | Developer must handle conflicts |
| **Performance** | Higher latency (coordination) | Lower latency (no coordination) |
| **Use Case** | Financial, Healthcare, ERP | Social media, IoT, CDN, Analytics |
| **Databases** | Oracle, PostgreSQL, MySQL, SQL Server | Cassandra, DynamoDB, CouchDB, Riak |
| **Example** | Bank transfer MUST be correct | Twitter like count can be approximate |

---

## 6. The Consistency Spectrum in Real Databases

Most databases don't fit neatly into "pure ACID" or "pure BASE." They offer a **spectrum**:

```
┌──────────────────────────────────────────────────────────────────┐
│                    CONSISTENCY SPECTRUM                            │
│                                                                    │
│  STRONG ◄─────────────────────────────────────────► EVENTUAL     │
│                                                                    │
│  PostgreSQL   MySQL    MongoDB   DynamoDB   Cassandra   CouchDB  │
│  Oracle       SQL Svr  (default) (default)  (ONE/LOCAL)          │
│  Spanner              (w/ write  (can be    (can be              │
│  CockroachDB           concern)  strong!)   strong!)             │
│  SQLite                                                          │
│                                                                    │
│  ◄── Most relational ────────────── Most NoSQL ──►               │
│                                                                    │
│  KEY INSIGHT: Many databases let you CHOOSE per operation        │
│  • MongoDB: w:"majority" + r:"majority" = strong consistency    │
│  • Cassandra: ALL/QUORUM/ONE — tune per query                   │
│  • DynamoDB: Strongly consistent reads (2x cost) available      │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Tunable Consistency Examples

```sql
-- MongoDB: Tune per write operation
db.orders.insertOne(
  { item: "widget", qty: 10 },
  { writeConcern: { w: "majority", j: true } }  -- Strong durability
);

-- Cassandra: Tune per query
-- QUORUM = majority of replicas must respond
SELECT * FROM orders WHERE id = 42
  USING CONSISTENCY QUORUM;

-- DynamoDB: Choose per read
aws dynamodb get-item \
  --table-name Orders \
  --key '{"id": {"N": "42"}}' \
  --consistent-read  // Strongly consistent (2x RCU cost)
```

---

## 7. When to Choose What?

### Decision Framework

```
CHOOSE ACID WHEN:
├── Money is involved (banking, payments, billing)
├── Legal/regulatory requirements (healthcare, finance)
├── Data correctness is non-negotiable
├── Inventory management (can't oversell!)
├── User authentication & authorization
├── Single-region deployment
└── Moderate scale (single server or small cluster)

CHOOSE BASE WHEN:
├── Massive scale (millions of ops/sec globally)
├── High availability is more important than instant consistency
├── Data can be eventually correct (social feeds, analytics)
├── Multi-region / multi-datacenter deployment
├── IoT sensor data (slight delay is fine)
├── Content delivery & caching
├── Recommendation engines (approximate is OK)
└── Event logging & metrics

CHOOSE BOTH (Hybrid) — MOST REAL SYSTEMS:
├── ACID for payment processing (PostgreSQL)
├── BASE for product catalog search (Elasticsearch)
├── ACID for order placement (PostgreSQL)
├── BASE for analytics dashboard (Cassandra)
├── ACID for user accounts (PostgreSQL)
└── BASE for session caching (Redis)
```

### Real-World Examples

| Company | ACID Usage | BASE Usage |
|---------|-----------|-----------|
| **Amazon** | Order payments (RDBMS) | Product recommendations (DynamoDB) |
| **Netflix** | Billing (MySQL) | Viewing history (Cassandra) |
| **Uber** | Payment processing (PostgreSQL) | Ride tracking (Redis + Cassandra) |
| **Facebook** | User account (MySQL) | News feed (TAO, custom) |
| **Twitter** | DM (encrypted, strong) | Like/retweet counts (eventual) |

---

## 8. Common Misconceptions Debunked

```
┌────────────────────────────────────────────────────────────────────┐
│                    MYTH vs REALITY                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  MYTH: "NoSQL databases don't support transactions"               │
│  REALITY: MongoDB has multi-document ACID since 4.0 (2018)       │
│           DynamoDB has transactions since 2018                    │
│           FaunaDB is fully ACID                                   │
│                                                                    │
│  MYTH: "ACID means slow"                                          │
│  REALITY: PostgreSQL handles 100K+ transactions/sec               │
│           VoltDB does millions of ACID transactions/sec           │
│           Bottleneck is usually BAD queries, not ACID             │
│                                                                    │
│  MYTH: "Eventually consistent = data loss"                        │
│  REALITY: Data is NOT lost — it just takes time to propagate     │
│           Usually milliseconds to seconds                         │
│                                                                    │
│  MYTH: "You must choose one or the other"                         │
│  REALITY: Most systems use BOTH! (Polyglot persistence)          │
│           Use ACID where correctness matters                     │
│           Use BASE where availability/scale matters               │
│                                                                    │
│  MYTH: "BASE is better because it scales"                         │
│  REALITY: NewSQL (CockroachDB, Spanner, TiDB) gives you         │
│           ACID + horizontal scaling. The trade-off is narrowing. │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 9. The Connection to CAP Theorem (Preview)

ACID and BASE are directly related to the **CAP theorem** (covered in detail in Chapter 1.6):

```
CAP THEOREM: In a distributed system, you can only guarantee
2 out of 3 properties:

  C = Consistency (every read returns the latest write)
  A = Availability (every request gets a response)
  P = Partition Tolerance (system works despite network failures)

Since network partitions are INEVITABLE in distributed systems,
the real choice is between C and A:

  ACID databases → Choose CP (Consistency + Partition Tolerance)
    → During network partition: become unavailable to stay consistent
    → PostgreSQL, MySQL, Oracle (single-node: all three!)

  BASE databases → Choose AP (Availability + Partition Tolerance)
    → During network partition: stay available but may be inconsistent
    → Cassandra, DynamoDB, CouchDB

  NewSQL databases → Try to maximize all three!
    → CockroachDB, Spanner (CP with high availability design)
```

---

## 🧠 Quick Recall — Chapter Summary

| Concept | One-Line Summary |
|---------|-----------------|
| **Atomicity** | Transaction is all-or-nothing — no partial changes |
| **Consistency** | Database moves from one valid state to another |
| **Isolation** | Concurrent transactions don't interfere with each other |
| **Durability** | Once committed, data survives any failure |
| **ACID** | Gold standard for data integrity (SQL databases) |
| **BASE** | Alternative for distributed systems — availability over consistency |
| **Basically Available** | System always responds, even with stale data |
| **Soft State** | Data may change over time as it propagates |
| **Eventually Consistent** | All nodes will converge to same value — eventually |
| **Isolation Levels** | Read Uncommitted → Read Committed → Repeatable Read → Serializable |
| **Dirty Read** | Reading uncommitted data from another transaction |
| **Phantom Read** | New rows appearing between two reads in same transaction |
| **Tunable Consistency** | Many databases let you choose consistency level per operation |

---

## ❓ Self-Check Questions

1. Explain each letter of ACID with a real-world example.
2. What happens during a bank transfer if the database crashes after debiting but before crediting? How does atomicity help?
3. What are the four SQL isolation levels? Which problems does each prevent?
4. What is a "dirty read"? Give an example.
5. When would you choose BASE over ACID?
6. Is "eventual consistency" the same as "data loss"? Explain.
7. Name a scenario where SERIALIZABLE isolation is absolutely necessary.
8. How does WAL (Write-Ahead Logging) enable durability?
9. Can a NoSQL database be ACID-compliant? Give examples.
10. A social media app shows a post's like count as 99 on one server and 101 on another. Which property is this demonstrating — and is it a problem?

---

> **Next Chapter** → [1.6 — CAP Theorem & Distributed Database Theory](./06-CAP-Theorem.md)
