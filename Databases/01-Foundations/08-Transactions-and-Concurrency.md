# 🔒 Chapter 1.8 — Transactions & Concurrency Control

> **Level:** 🔴 Advanced | ⭐ Must-Know
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 1.5 (ACID vs BASE), Chapter 1.7 (Indexing)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand what a **transaction** really is — beyond textbook definitions
- Master all **4 isolation levels** and know exactly when to use each
- Know how **MVCC** works (used by PostgreSQL, Oracle, MySQL InnoDB, MongoDB)
- Understand **pessimistic vs optimistic** locking — and pick the right one
- Diagnose and **prevent deadlocks** like a production DBA
- Grasp **Two-Phase Commit (2PC)** for distributed transactions

---

## 🧠 Why Transactions Exist — The Horror Story

Imagine online banking **without** transactions:

```
User: Transfer $500 from Account A → Account B

Step 1: Deduct $500 from Account A    ✅ Done! (A: $1000 → $500)
Step 2: 💥 SERVER CRASHES 💥
Step 3: Add $500 to Account B         ❌ Never happened!

Result: $500 has VANISHED from the universe. 💀
        Account A: $500 (lost $500)
        Account B: $0   (never received)
```

> **A transaction wraps multiple operations into a single "all-or-nothing" unit. Either EVERYTHING succeeds, or NOTHING happens.**

---

## 📦 Transaction Basics

### The Transaction Lifecycle

```
                Transaction Lifecycle

    ┌──────────┐     ┌─────────────────────────┐     ┌──────────┐
    │  BEGIN    │────►│   Active State           │────►│  COMMIT  │
    │TRANSACTION│     │                         │     │  (Save)  │
    └──────────┘     │  - Read data            │     └──────────┘
                     │  - Modify data          │          │
                     │  - All in memory/WAL    │          ▼
                     │                         │     Changes are
                     │  If error occurs:       │     PERMANENT ✅
                     │        │                │
                     │        ▼                │
                     │   ┌──────────┐          │
                     │   │ ROLLBACK │          │
                     │   │ (Undo)   │          │
                     │   └──────────┘          │
                     │        │                │
                     │        ▼                │
                     │   Everything            │
                     │   UNDONE ❌              │
                     └─────────────────────────┘
```

### Syntax Across Databases

```sql
-- Standard SQL / PostgreSQL / SQL Server
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 500 WHERE id = 'A';
    UPDATE accounts SET balance = balance + 500 WHERE id = 'B';
COMMIT;

-- If something goes wrong:
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 500 WHERE id = 'A';
    -- Oops, Account B doesn't exist!
ROLLBACK;  -- Undo everything, Account A gets its $500 back

-- Oracle (implicit begin — every statement starts a transaction)
UPDATE accounts SET balance = balance - 500 WHERE id = 'A';
UPDATE accounts SET balance = balance + 500 WHERE id = 'B';
COMMIT;

-- MySQL (InnoDB — autocommit is ON by default)
START TRANSACTION;
    UPDATE accounts SET balance = balance - 500 WHERE id = 'A';
    UPDATE accounts SET balance = balance + 500 WHERE id = 'B';
COMMIT;

-- MongoDB (v4.0+ multi-document transactions)
session = client.startSession();
session.startTransaction();
    db.accounts.updateOne({ _id: "A" }, { $inc: { balance: -500 } }, { session });
    db.accounts.updateOne({ _id: "B" }, { $inc: { balance: +500 } }, { session });
session.commitTransaction();
```

### Savepoints — Partial Rollback

```sql
BEGIN TRANSACTION;

UPDATE inventory SET qty = qty - 1 WHERE product = 'Laptop';
SAVEPOINT after_inventory;

UPDATE shipping SET status = 'dispatched' WHERE order_id = 1001;
-- Oops! Shipping system error

ROLLBACK TO SAVEPOINT after_inventory;
-- Inventory change is KEPT, shipping change is UNDONE

UPDATE shipping SET status = 'pending' WHERE order_id = 1001;  -- retry with different status
COMMIT;
```

---

## 👻 The Concurrency Nightmares

When multiple users access the **same data** simultaneously, bad things happen. These are the **read phenomena**:

### 1. Dirty Read 🦠

> Reading data that **another transaction hasn't committed yet**.

```
Timeline:
  ────────────────────────────────────────────────►
  
  Transaction A (Alice):              Transaction B (Bob):
  ─────────────────────               ─────────────────────
  BEGIN;                              BEGIN;
  UPDATE accounts                     
    SET balance = 0                   
    WHERE id = 'X';                   
  -- balance is now 0                 SELECT balance FROM accounts
  -- (NOT committed yet!)               WHERE id = 'X';
                                      -- Reads 0 (dirty data!) 🦠
  ROLLBACK;                           
  -- balance is back to $1000!        -- Bob thinks balance is $0
                                      -- but it's actually $1000!
                                      COMMIT;

  💀 Bob made decisions based on data that NEVER EXISTED.
```

---

### 2. Non-Repeatable Read 🎭

> Reading the **same row twice** and getting **different results**.

```
Timeline:
  ────────────────────────────────────────────────►
  
  Transaction A:                      Transaction B:
  ─────────────                       ─────────────
  BEGIN;                              
  SELECT price FROM products          
    WHERE id = 42;                    
  -- Result: $100                     
                                      BEGIN;
                                      UPDATE products 
                                        SET price = $150 
                                        WHERE id = 42;
                                      COMMIT;
  SELECT price FROM products          
    WHERE id = 42;                    
  -- Result: $150  ← DIFFERENT! 🎭
  
  💀 Same query, same transaction, different results.
     If A was calculating a discount based on the first read... chaos.
```

---

### 3. Phantom Read 👻

> Running the **same query twice** and getting **different ROWS** (new rows appeared or disappeared).

```
Timeline:
  ────────────────────────────────────────────────►
  
  Transaction A:                      Transaction B:
  ─────────────                       ─────────────
  BEGIN;                              
  SELECT COUNT(*) FROM orders         
    WHERE status = 'pending';         
  -- Result: 5 orders                 
                                      BEGIN;
                                      INSERT INTO orders 
                                        (status) VALUES ('pending');
                                      COMMIT;
  SELECT COUNT(*) FROM orders         
    WHERE status = 'pending';         
  -- Result: 6 orders ← PHANTOM! 👻
  
  💀 A "ghost" row appeared between two identical queries.
     Critical for reports: "We had 5 pending orders... wait, now 6?"
```

---

### 4. Lost Update 💸

> Two transactions **read the same data**, then **both update it** — one update is **silently lost**.

```
Timeline:
  ────────────────────────────────────────────────►
  
  Transaction A (Alice):              Transaction B (Bob):
  ─────────────────────               ─────────────────────
  BEGIN;                              BEGIN;
  SELECT qty FROM inventory           SELECT qty FROM inventory
    WHERE product = 'iPhone';           WHERE product = 'iPhone';
  -- qty = 10                         -- qty = 10
  
  -- Alice sells 3                    -- Bob sells 2
  UPDATE inventory                    
    SET qty = 10 - 3 = 7              UPDATE inventory
    WHERE product = 'iPhone';           SET qty = 10 - 2 = 8
                                        WHERE product = 'iPhone';
  COMMIT;                             COMMIT;
  
  Final qty: 8  (Bob's update wins)
  Expected:  5  (10 - 3 - 2)
  
  💀 Alice's sale of 3 units is LOST. Inventory is wrong.
```

---

## 🛡️ Isolation Levels — The Defense System

> SQL Standard defines **4 isolation levels** — each prevents more phenomena but costs more performance.

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    ISOLATION LEVELS COMPARISON                        ║
╠══════════════════╦════════════╦══════════════╦═════════════╦═════════╣
║                  ║ Dirty      ║ Non-         ║ Phantom     ║ Lost    ║
║ Isolation Level  ║ Read       ║ Repeatable   ║ Read        ║ Update  ║
║                  ║            ║ Read         ║             ║         ║
╠══════════════════╬════════════╬══════════════╬═════════════╬═════════╣
║ READ UNCOMMITTED ║ ⚠️ Possible ║ ⚠️ Possible   ║ ⚠️ Possible  ║ ⚠️ Poss. ║
║ (Chaos mode)     ║            ║              ║             ║         ║
╠══════════════════╬════════════╬══════════════╬═════════════╬═════════╣
║ READ COMMITTED   ║ ✅ Blocked ║ ⚠️ Possible   ║ ⚠️ Possible  ║ ⚠️ Poss. ║
║ (Most common)    ║            ║              ║             ║         ║
╠══════════════════╬════════════╬══════════════╬═════════════╬═════════╣
║ REPEATABLE READ  ║ ✅ Blocked ║ ✅ Blocked   ║ ⚠️ Possible  ║ ✅ Block ║
║                  ║            ║              ║             ║         ║
╠══════════════════╬════════════╬══════════════╬═════════════╬═════════╣
║ SERIALIZABLE     ║ ✅ Blocked ║ ✅ Blocked   ║ ✅ Blocked  ║ ✅ Block ║
║ (Safest, slowest)║            ║              ║             ║         ║
╚══════════════════╩════════════╩══════════════╩═════════════╩═════════╝

                ◄──────────────────────────────────────────────►
                Less Safe / Faster         More Safe / Slower
```

### Default Isolation Levels by Database

```
┌─────────────────┬────────────────────┬──────────────────────────┐
│ Database        │ Default Level      │ Implementation           │
├─────────────────┼────────────────────┼──────────────────────────┤
│ Oracle          │ READ COMMITTED     │ MVCC (Undo segments)     │
│ PostgreSQL      │ READ COMMITTED     │ MVCC (Tuple versioning)  │
│ MySQL (InnoDB)  │ REPEATABLE READ    │ MVCC + Gap locks         │
│ SQL Server      │ READ COMMITTED     │ Locks (or RCSI with MVCC)│
│ MongoDB         │ Snapshot           │ WiredTiger MVCC          │
│ SQLite          │ SERIALIZABLE       │ File-level locking       │
└─────────────────┴────────────────────┴──────────────────────────┘
```

### Setting Isolation Levels

```sql
-- PostgreSQL
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
    -- Your queries here
COMMIT;

-- Or per session:
SET default_transaction_isolation = 'repeatable read';

-- SQL Server
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
    -- Your queries here
COMMIT;

-- MySQL
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
    -- Your queries here
COMMIT;

-- Oracle (only supports READ COMMITTED and SERIALIZABLE)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Your queries here
COMMIT;
```

---

## 🔄 MVCC — Multi-Version Concurrency Control

> **The magic behind modern databases.** Instead of locking, keep **multiple versions** of each row.

### The Core Idea

```
Traditional Locking:
  Writer BLOCKS readers. Reader BLOCKS writers. 😤
  
  Writer: "I'm updating row 5, everyone WAIT!"
  Reader: "I just want to read... *sigh*"

MVCC (Multi-Version Concurrency Control):
  Writers NEVER block readers. Readers NEVER block writers. 🎉
  
  Writer: "I'm creating version 2 of row 5"
  Reader: "Cool, I'll keep reading version 1"
  Both work simultaneously!
```

### How MVCC Works (PostgreSQL Example)

```
Table: accounts

 Physical Storage (heap):
 ┌─────┬─────────┬─────────┬─────────┬─────────────┐
 │ Row │ xmin    │ xmax    │ balance │ Visible to?  │
 │     │(created)│(deleted)│         │              │
 ├─────┼─────────┼─────────┼─────────┼─────────────┤
 │  1  │  100    │  105    │  $1000  │ Txn < 105    │  ← Old version
 │  1' │  105    │  -      │  $500   │ Txn >= 105   │  ← New version
 └─────┴─────────┴─────────┴─────────┴─────────────┘

 Transaction 103 (started before update):
   → Sees row with xmin=100, xmax=105
   → 103 < 105, so sees OLD version → $1000 ✅

 Transaction 110 (started after update):
   → Sees row with xmin=105, xmax=null
   → 110 >= 105, so sees NEW version → $500 ✅

 Neither transaction blocks the other! 🎉
```

### MVCC Across Databases

```
╔══════════════════════════════════════════════════════════════════════╗
║ Database    │ MVCC Implementation                                   ║
╠═════════════╪══════════════════════════════════════════════════════╣
║ PostgreSQL  │ Old versions stored IN the table (heap)              ║
║             │ VACUUM cleans up dead tuples                         ║
║             │ Pro: Simple. Con: Table bloat needs VACUUM           ║
╠═════════════╪══════════════════════════════════════════════════════╣
║ Oracle      │ Old versions stored in UNDO tablespace               ║
║             │ Current version in table, old in undo                ║
║             │ Pro: No table bloat. Con: "ORA-01555 snapshot too old"║
╠═════════════╪══════════════════════════════════════════════════════╣
║ MySQL       │ Old versions in UNDO log (InnoDB)                    ║
║ (InnoDB)    │ Purge thread cleans old versions                     ║
║             │ Pro: Similar to Oracle. Con: Long txns = undo bloat  ║
╠═════════════╪══════════════════════════════════════════════════════╣
║ SQL Server  │ Two modes:                                           ║
║             │ 1. Locking (default) — no MVCC                      ║
║             │ 2. RCSI/Snapshot — versions in tempdb                ║
║             │ Pro: Flexible. Con: tempdb can become bottleneck     ║
╠═════════════╪══════════════════════════════════════════════════════╣
║ MongoDB     │ WiredTiger engine uses MVCC internally               ║
║             │ Snapshot isolation for multi-document transactions   ║
╚═════════════╧══════════════════════════════════════════════════════╝
```

---

## 🔐 Locking Mechanisms

### Lock Types

```
╔═══════════════════════════════════════════════════════════════╗
║                    LOCK COMPATIBILITY MATRIX                  ║
╠═════════════════════╦═══════════╦═══════════╦════════════════╣
║                     ║ Shared(S) ║Exclusive(X)║ Update (U)    ║
╠═════════════════════╬═══════════╬═══════════╬════════════════╣
║ Shared (S)          ║  ✅ Yes   ║  ❌ No    ║  ✅ Yes        ║
║ (Read Lock)         ║           ║           ║                ║
╠═════════════════════╬═══════════╬═══════════╬════════════════╣
║ Exclusive (X)       ║  ❌ No    ║  ❌ No    ║  ❌ No         ║
║ (Write Lock)        ║           ║           ║                ║
╠═════════════════════╬═══════════╬═══════════╬════════════════╣
║ Update (U)          ║  ✅ Yes   ║  ❌ No    ║  ❌ No         ║
║ (Read→Write intent) ║           ║           ║                ║
╚═════════════════════╩═══════════╩═══════════╩════════════════╝

  S + S = ✅  Multiple readers can coexist
  S + X = ❌  Writer blocks readers (in locking systems)
  X + X = ❌  Only one writer at a time
```

### Lock Granularity

```
                Lock Granularity Spectrum

  Coarse ◄─────────────────────────────────────► Fine
  
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ DATABASE │  │  TABLE   │  │  PAGE    │  │   ROW    │
  │  LOCK    │  │  LOCK    │  │  LOCK    │  │  LOCK    │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
       │             │              │             │
   One writer    Lock entire    Lock a disk   Lock single
   at a time     table         page (~8KB)    row
       │             │              │             │
   SQLite       MySQL MyISAM   SQL Server    PostgreSQL
   (default)    (legacy)       (internal)    Oracle
                                             MySQL InnoDB

  Coarse locks → Less overhead, less concurrency
  Fine locks   → More overhead, more concurrency
```

---

## ⚔️ Pessimistic vs Optimistic Locking

### Pessimistic Locking — "Lock First, Ask Questions Later"

```sql
-- "I KNOW there will be conflicts, so I'll lock immediately"

-- PostgreSQL / MySQL
BEGIN;
SELECT * FROM products WHERE id = 42 FOR UPDATE;  -- LOCK the row!
-- Nobody else can modify this row until I commit
UPDATE products SET stock = stock - 1 WHERE id = 42;
COMMIT;  -- Lock released

-- Oracle
SELECT * FROM products WHERE id = 42 FOR UPDATE NOWAIT;  -- Don't wait, error if locked
SELECT * FROM products WHERE id = 42 FOR UPDATE WAIT 5;  -- Wait max 5 seconds

-- SQL Server
BEGIN TRANSACTION;
SELECT * FROM products WITH (UPDLOCK, ROWLOCK) WHERE id = 42;
UPDATE products SET stock = stock - 1 WHERE id = 42;
COMMIT;
```

```
When to use Pessimistic Locking:
  ✅ High contention (many users editing same data)
  ✅ Short transactions
  ✅ Banking / financial systems
  ✅ Inventory management (prevent overselling)
  
  ❌ Avoid when: Long-running transactions, distributed systems
```

### Optimistic Locking — "Hope for the Best, Check Before Saving"

```sql
-- "Conflicts are RARE, so I'll just check at the end"

-- Step 1: Read the row with its version
SELECT id, name, stock, version FROM products WHERE id = 42;
-- Result: id=42, name='iPhone', stock=10, version=5

-- Step 2: User modifies data in the application (no lock held!)

-- Step 3: Update ONLY IF version hasn't changed
UPDATE products 
SET stock = 9, version = version + 1 
WHERE id = 42 AND version = 5;  -- Check version!

-- If rows_affected = 1 → Success! ✅
-- If rows_affected = 0 → Someone else modified it! → Retry or error ❌
```

```
Optimistic Locking Implementations:

  1. VERSION COLUMN (most common)
     ┌─────┬────────┬───────┬─────────┐
     │ id  │ name   │ stock │ version │
     ├─────┼────────┼───────┼─────────┤
     │ 42  │ iPhone │ 10    │ 5       │
     └─────┴────────┴───────┴─────────┘
     
  2. TIMESTAMP COLUMN
     WHERE id = 42 AND updated_at = '2024-01-15 10:30:00'
     
  3. HASH OF ROW DATA
     WHERE id = 42 AND MD5(name||stock||price) = 'abc123...'
     
  4. MongoDB: Built-in with __v field (Mongoose) or conditional updates
     db.products.updateOne(
       { _id: 42, version: 5 },
       { $set: { stock: 9 }, $inc: { version: 1 } }
     );

When to use Optimistic Locking:
  ✅ Low contention (conflicts are rare)
  ✅ Long-running transactions (user editing a form)
  ✅ Distributed systems / microservices
  ✅ Web applications (user thinks for minutes)
  
  ❌ Avoid when: High contention (constant retries = worse performance)
```

### Comparison

```
╔════════════════════╦══════════════════════╦═══════════════════════╗
║                    ║ PESSIMISTIC          ║ OPTIMISTIC            ║
╠════════════════════╬══════════════════════╬═══════════════════════╣
║ Lock timing        ║ Lock BEFORE access   ║ Check BEFORE commit   ║
║ Conflict handling  ║ Block/wait           ║ Retry/fail            ║
║ Concurrency        ║ Lower                ║ Higher                ║
║ Deadlock risk      ║ ⚠️ Yes               ║ ✅ None               ║
║ Best for           ║ High contention      ║ Low contention        ║
║ Cost when no       ║ High (unnecessary    ║ Low (no locks)        ║
║   conflict         ║  lock overhead)      ║                       ║
║ Cost when          ║ Low (already locked) ║ High (retry overhead) ║
║   conflict         ║                      ║                       ║
╚════════════════════╩══════════════════════╩═══════════════════════╝
```

---

## 💀 Deadlocks — The Database's Worst Enemy

### What Is a Deadlock?

```
  Transaction A                    Transaction B
  ─────────────                    ─────────────
  Lock Row 1 ✅                    Lock Row 2 ✅
       │                                │
       │                                │
       ▼                                ▼
  Want Row 2 ⏳                    Want Row 1 ⏳
  (Held by B!)                     (Held by A!)
       │                                │
       └─── WAITING ◄──── WAITING ──────┘
             
             💀 DEADLOCK! Both wait forever!
             
  The database DETECTS this and kills one transaction:
  → Transaction B: "ERROR: deadlock detected" 
  → Transaction A: Gets Row 2, completes successfully
```

### Deadlock Prevention Strategies

```sql
-- Strategy 1: Always access tables/rows in the SAME ORDER
-- ❌ Bad:
--   Txn A: Lock users → Lock orders
--   Txn B: Lock orders → Lock users    ← Opposite order = deadlock risk!
-- ✅ Good:
--   Txn A: Lock orders → Lock users
--   Txn B: Lock orders → Lock users    ← Same order = no deadlock!

-- Strategy 2: Keep transactions SHORT
-- ❌ Bad:
BEGIN;
    SELECT * FROM orders FOR UPDATE;
    -- Call external API (takes 5 seconds)... 💀
    UPDATE orders SET status = 'shipped';
COMMIT;
-- ✅ Good:
-- Call API first, THEN do the transaction
BEGIN;
    UPDATE orders SET status = 'shipped' WHERE id = 1001;
COMMIT;

-- Strategy 3: Use lock timeouts
-- PostgreSQL:
SET lock_timeout = '5s';  -- Don't wait forever

-- Oracle:
SELECT * FROM orders WHERE id = 1001 FOR UPDATE WAIT 5;

-- SQL Server:
SET LOCK_TIMEOUT 5000;  -- 5000 milliseconds

-- Strategy 4: Use optimistic locking (no locks = no deadlocks)

-- Strategy 5: Reduce lock scope
-- ❌ Bad: SELECT * FROM orders FOR UPDATE;  (locks ALL orders)
-- ✅ Good: SELECT * FROM orders WHERE id = 1001 FOR UPDATE;  (lock ONE)
```

### Monitoring Deadlocks

```sql
-- PostgreSQL: Check for locks
SELECT blocked_locks.pid     AS blocked_pid,
       blocking_locks.pid    AS blocking_pid,
       blocked_activity.query AS blocked_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
WHERE NOT blocked_locks.granted;

-- MySQL: Show current locks
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
SHOW ENGINE INNODB STATUS;  -- Deadlock section

-- SQL Server: Deadlock graph
-- Enable trace flag 1222 for deadlock logging
DBCC TRACEON(1222, -1);
-- Or use Extended Events / SQL Server Profiler

-- Oracle: 
SELECT * FROM V$LOCK WHERE BLOCK > 0;
```

---

## 🌐 Distributed Transactions — Two-Phase Commit (2PC)

> When a transaction spans **multiple databases or services**.

### The Problem

```
E-Commerce Order:
  Database 1 (Orders): Create order record
  Database 2 (Inventory): Deduct stock
  Database 3 (Payments): Charge credit card

  What if Orders succeeds, Inventory succeeds, but Payment fails?
  → Must undo EVERYTHING across ALL databases!
```

### Two-Phase Commit Protocol

```
                    Coordinator
                    (Transaction Manager)
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
     │ DB 1    │   │ DB 2    │   │ DB 3    │
     │ Orders  │   │Inventory│   │Payments │
     └─────────┘   └─────────┘   └─────────┘

  PHASE 1: PREPARE (Voting Phase)
  ─────────────────────────────────────────
  Coordinator → DB1: "Can you commit?"  → DB1: "YES" ✅
  Coordinator → DB2: "Can you commit?"  → DB2: "YES" ✅
  Coordinator → DB3: "Can you commit?"  → DB3: "YES" ✅

  PHASE 2: COMMIT (Decision Phase)
  ─────────────────────────────────────────
  All said YES? → Coordinator: "COMMIT ALL!"
  
  Coordinator → DB1: "COMMIT!"  → DB1: COMMITTED ✅
  Coordinator → DB2: "COMMIT!"  → DB2: COMMITTED ✅
  Coordinator → DB3: "COMMIT!"  → DB3: COMMITTED ✅

  ────────────────────────────────────────────
  WHAT IF DB3 SAYS "NO" IN PHASE 1?
  ────────────────────────────────────────────
  Coordinator → ALL: "ABORT! ROLLBACK EVERYTHING!"
  
  DB1: ROLLED BACK ↩️
  DB2: ROLLED BACK ↩️
  DB3: ROLLED BACK ↩️
```

### 2PC Limitations & Alternatives

```
╔══════════════════════════════════════════════════════════════════╗
║ 2PC Problems:                                                    ║
║   • Blocking: If coordinator crashes, participants are stuck     ║
║   • Performance: Holding locks across network = slow             ║
║   • Single point of failure: Coordinator crash = disaster        ║
║                                                                  ║
║ Modern Alternatives:                                             ║
║ ┌────────────────────┬───────────────────────────────────────┐   ║
║ │ Pattern            │ Description                           │   ║
║ ├────────────────────┼───────────────────────────────────────┤   ║
║ │ Saga Pattern       │ Chain of local transactions with      │   ║
║ │                    │ compensating actions for rollback     │   ║
║ │                    │ (Used by: Uber, Netflix)              │   ║
║ ├────────────────────┼───────────────────────────────────────┤   ║
║ │ Outbox Pattern     │ Write event to local DB + outbox      │   ║
║ │                    │ table, async publish to message queue  │   ║
║ ├────────────────────┼───────────────────────────────────────┤   ║
║ │ Event Sourcing     │ Store events, not state. Replay to    │   ║
║ │                    │ rebuild state. Natural consistency.    │   ║
║ ├────────────────────┼───────────────────────────────────────┤   ║
║ │ 3PC (Three-Phase)  │ Adds a "pre-commit" phase to reduce  │   ║
║ │                    │ blocking. Rarely used in practice.    │   ║
║ └────────────────────┴───────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🧪 Practical Scenarios — Choose Your Weapon

```
╔══════════════════════════════════════════════════════════════════╗
║ Scenario                  │ Isolation     │ Locking     │ Why  ║
╠═══════════════════════════╪═══════════════╪═════════════╪══════╣
║ Bank transfer             │ SERIALIZABLE  │ Pessimistic │ $$$  ║
║ Read dashboard stats      │ READ COMMITTED│ None/MVCC   │ Perf ║
║ E-commerce checkout       │ REPEATABLE RD │ Pessimistic │ Stock║
║ User editing a wiki page  │ READ COMMITTED│ Optimistic  │ Long ║
║ Analytics / reporting     │ READ COMMITTED│ None (MVCC) │ Read ║
║ Booking a flight seat     │ SERIALIZABLE  │ Pessimistic │ Seat ║
║ Social media "like" count │ READ COMMITTED│ Optimistic  │ Perf ║
║ Medical records update    │ SERIALIZABLE  │ Pessimistic │ Life ║
╚═══════════════════════════╧═══════════════╧═════════════╧══════╝
```

---

## 🔑 Key Takeaways

```
✅ Transactions = all-or-nothing (COMMIT or ROLLBACK, no middle ground)
✅ 4 Isolation Levels: READ UNCOMMITTED → READ COMMITTED → REPEATABLE READ → SERIALIZABLE
✅ MVCC = writers don't block readers (PostgreSQL, Oracle, MySQL InnoDB, MongoDB)
✅ Pessimistic locking = lock first, high contention scenarios
✅ Optimistic locking = version check at commit, low contention / long transactions
✅ Deadlocks: always access resources in the same order, keep transactions short
✅ 2PC for distributed transactions, but prefer Saga pattern in microservices
✅ Default isolation level varies: PostgreSQL/Oracle/SQL Server = READ COMMITTED, MySQL = REPEATABLE READ
✅ Most apps are fine with READ COMMITTED + row-level locking
✅ SERIALIZABLE when money, health, or legal compliance is at stake
```

---

## 🔗 What's Next?

**Chapter 1.9 → [Database Security Fundamentals](./09-Database-Security.md)**
Where we learn to protect your data from the bad guys — SQL injection, encryption, access control, and audit trails.

---

> *"The hardest part of concurrency isn't writing correct code — it's writing code that stays correct when a thousand users hit it at the same time."*
