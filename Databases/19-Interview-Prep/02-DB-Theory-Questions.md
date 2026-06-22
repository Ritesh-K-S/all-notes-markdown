# 🧠 Chapter 8.2 — Database Interview Questions — Concepts & Theory

> **"Anyone can write a query. But when they ask you WHY the database does what it does — that's where heroes are made."** — Principal Engineer, Amazon RDS

---

## 📌 Metadata

| Field | Value |
|-------|-------|
| **Level** | 🟡 Intermediate → 🔴 Advanced |
| **Time to Master** | ~5-6 hours |
| **Prerequisites** | Foundations (Part 1), SQL Mastery (Part 2A) |
| **Interview Coverage** | System Design rounds, DBA interviews, Backend Sr. Engineer |

---

## 🎯 What You'll Master

- ✅ 80+ database theory questions — the stuff textbooks don't teach well
- ✅ Deep conceptual understanding (ACID, CAP, indexing internals, replication)
- ✅ Architectural thinking — sharding, partitioning, scaling trade-offs
- ✅ Real-world scenario questions (how companies actually ask them)
- ✅ The "WHY" behind every answer — not just definitions
- ✅ Questions spanning SQL, NoSQL, and distributed systems

---

## 📋 Question Categories

```
┌─────────────────────────────────────────────────────────────────────┐
│              DATABASE THEORY — QUESTION MAP                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SECTION 1: Data Modeling & Schema Design      (Q1 – Q15)          │
│  SECTION 2: ACID, Transactions & Concurrency   (Q16 – Q30)        │
│  SECTION 3: Indexing & Query Optimization       (Q31 – Q45)        │
│  SECTION 4: CAP Theorem & Distributed Systems   (Q46 – Q55)       │
│  SECTION 5: Replication, Sharding & Scaling     (Q56 – Q70)       │
│  SECTION 6: Database Comparisons & Architecture (Q71 – Q80)       │
│  BONUS: Rapid-Fire "Would You Rather" Questions                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

# 📐 SECTION 1: Data Modeling & Schema Design (Q1–Q15)

---

### Q1: What is the difference between a Database and a DBMS?

```
┌──────────────────────────────────────────────────────────────┐
│  DATABASE = The actual DATA (files on disk)                  │
│  DBMS     = The SOFTWARE that manages the data              │
│                                                              │
│  Analogy:                                                    │
│  Database = Books in a library                               │
│  DBMS     = The librarian + catalog system                   │
│                                                              │
│  Examples:                                                   │
│  DBMS: Oracle, PostgreSQL, MySQL, MongoDB                    │
│  Database: The "sales_db" inside PostgreSQL                  │
└──────────────────────────────────────────────────────────────┘
```

---

### Q2: What is the difference between a Schema and a Database?

```
Database Level Hierarchy:

  PostgreSQL:                    MySQL:
  ┌─────────────────────┐       ┌─────────────────────┐
  │ Cluster (Instance)  │       │ Server (Instance)   │
  │  ├── Database 1     │       │  ├── Database 1     │
  │  │    ├── Schema A  │       │  │    └── Tables    │
  │  │    │   └── Tables│       │  │        (schema = │
  │  │    └── Schema B  │       │  │         database)│
  │  │        └── Tables│       │  └── Database 2     │
  │  └── Database 2     │       └─────────────────────┘
  └─────────────────────┘
  
  Oracle:                        SQL Server:
  ┌─────────────────────┐       ┌─────────────────────┐
  │ Instance            │       │ Instance            │
  │  └── Database       │       │  ├── Database 1     │
  │      ├── Schema A   │       │  │    ├── Schema A  │
  │      │   (= user)   │       │  │    └── Schema B  │
  │      └── Schema B   │       │  └── Database 2     │
  └─────────────────────┘       └─────────────────────┘
```

> 💡 **Pro Tip**: In MySQL, "database" and "schema" are synonyms. In PostgreSQL/SQL Server, a schema is a namespace WITHIN a database. In Oracle, a schema equals a user.

---

### Q3: What is Normalization? Why do we normalize?

**Normalization** = Organizing data to **eliminate redundancy** and **ensure data integrity**.

```
UNNORMALIZED (Messy):
┌────────┬──────────┬──────────────────────────────────────┐
│ OrderID│ Customer │ Products                             │
├────────┼──────────┼──────────────────────────────────────┤
│ 1001   │ Alice    │ Laptop, Mouse, Keyboard              │
│ 1002   │ Alice    │ Monitor                              │
└────────┴──────────┴──────────────────────────────────────┘
Problems: Repeating groups, update anomalies, delete anomalies

NORMALIZED (1NF → 3NF):
Orders:    OrderID, CustomerID, OrderDate
Customers: CustomerID, Name, Email  
Items:     ItemID, OrderID, ProductID, Quantity
Products:  ProductID, Name, Price
```

**The Normal Forms — Quick Reference:**

```
┌──────┬──────────────────────────────┬──────────────────────────────────┐
│ NF   │ Rule                         │ Eliminates                       │
├──────┼──────────────────────────────┼──────────────────────────────────┤
│ 1NF  │ Atomic values, no repeating  │ Repeating groups                 │
│      │ groups                        │                                  │
│ 2NF  │ No partial dependencies      │ Partial dependency on composite  │
│      │ (on composite key)           │ key                              │
│ 3NF  │ No transitive dependencies   │ Non-key → non-key dependencies  │
│      │ (non-key depends on non-key) │                                  │
│ BCNF │ Every determinant is a       │ Anomalies in 3NF edge cases     │
│      │ candidate key                │                                  │
│ 4NF  │ No multi-valued dependencies │ Independent multi-valued facts  │
│ 5NF  │ No join dependencies         │ Complex decomposition anomalies │
└──────┴──────────────────────────────┴──────────────────────────────────┘
```

**🎯 Interview Answer**: "We normalize to 3NF/BCNF in OLTP to prevent anomalies. We denormalize for OLAP/reporting for read performance. The right level depends on the workload."

---

### Q4: What are the three types of Data Anomalies?

```
UPDATE ANOMALY:
  If "Alice" changes address, must update EVERY order row.
  Miss one → inconsistent data!

INSERT ANOMALY:
  Can't add a new product without an order.
  The product has nowhere to live!

DELETE ANOMALY:
  Delete the last order for "Bob" → lose Bob's customer info too.
  Data destroyed unintentionally!
```

---

### Q5: What is an ER Diagram? Explain the components.

```
ENTITY-RELATIONSHIP DIAGRAM:

  ┌────────────┐         ┌────────────────────┐         ┌────────────┐
  │  CUSTOMER  │ 1     M │       ORDER        │ M     1 │  PRODUCT   │
  │────────────│─────────│────────────────────│─────────│────────────│
  │ PK: cust_id│         │ PK: order_id       │         │ PK: prod_id│
  │ name       │         │ FK: cust_id        │         │ name       │
  │ email      │         │ FK: prod_id        │         │ price      │
  │ phone      │         │ quantity           │         │ category   │
  └────────────┘         │ order_date         │         └────────────┘
                         └────────────────────┘

  Components:
  ├── Entity (Rectangle): A thing → Customer, Order, Product
  ├── Attribute (Oval/Column): Properties → name, email, price
  ├── Relationship (Diamond/Line): Connection → "places", "contains"
  ├── Cardinality: 1:1, 1:M, M:M → How many of each
  └── Primary/Foreign Keys: Links between entities
```

**Cardinality Patterns:**
```
1:1  → One user has one profile (rare, consider merging)
1:M  → One customer has many orders (most common)
M:M  → Many students take many courses (junction table needed)
```

---

### Q6: What is a Star Schema vs Snowflake Schema?

```
STAR SCHEMA:                         SNOWFLAKE SCHEMA:
                                     
     ┌──────┐                             ┌───────┐
     │ Dim1 │                             │ Dim1a │
     └──┬───┘                             └──┬────┘
        │                                    │
  ┌─────┼──────┐                       ┌─────┼──────┐
  │Dim2 │ FACT │ Dim3               Dim1│     │ FACT │ Dim3
  └─────┼──────┘                    ────┼─────┼──────┘
        │                              │     │
     ┌──┴───┐                       ┌──┴──┐  │
     │ Dim4 │                       │Dim2 │ Dim3a
     └──────┘                       └─────┘

STAR:                                SNOWFLAKE:
✅ Simple queries (fewer JOINs)      ✅ Less redundancy
✅ Faster reads                      ✅ Less storage
❌ Some data redundancy              ❌ More JOINs (slower)
⭐ Preferred for most DW/BI         🟡 Use when storage matters
```

---

### Q7: What is a Surrogate Key vs Natural Key?

```
NATURAL KEY: Business-meaningful value
  Examples: SSN, email, ISBN, phone number
  ✅ Business meaning
  ❌ Can change (email, phone)
  ❌ May be long (composite keys)
  ❌ Privacy concerns (SSN as PK?)

SURROGATE KEY: System-generated, no business meaning
  Examples: Auto-increment INT, UUID, Snowflake ID
  ✅ Never changes
  ✅ Small (efficient JOINs)
  ✅ Simple
  ❌ No business meaning
  ❌ Extra index for business lookups
```

**🎯 Best Practice**: Use surrogate keys as PRIMARY KEY + unique constraint on the natural key.

```sql
CREATE TABLE users (
    user_id   SERIAL PRIMARY KEY,         -- Surrogate (system)
    email     VARCHAR(255) UNIQUE NOT NULL -- Natural (business)
);
```

---

### Q8: What is a Junction Table (Association Table)?

```
MANY-TO-MANY: Students ↔ Courses

Students:        Junction:           Courses:
┌──────────┐    ┌────────────────┐  ┌──────────┐
│ student_id│───→│ student_id(FK) │←─│ course_id │
│ name      │    │ course_id(FK)  │  │ name      │
└──────────┘    │ enrolled_date  │  └──────────┘
                │ grade          │
                └────────────────┘
                (student_courses)
```

```sql
CREATE TABLE student_courses (
    student_id INT REFERENCES students(student_id),
    course_id  INT REFERENCES courses(course_id),
    enrolled_date DATE DEFAULT CURRENT_DATE,
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id)  -- Composite PK
);
```

---

### Q9: When should you use UUID vs Auto-Increment for Primary Keys?

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ Auto-Increment       │ UUID                 │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Size             │ 4-8 bytes            │ 16 bytes             │
│ Ordering         │ ✅ Sequential        │ ❌ Random (v4)       │
│ Index friendly   │ ✅ Append-only       │ ❌ Random inserts    │
│ Distributed      │ ❌ Needs coordinator │ ✅ No coordination   │
│ Guessable        │ ⚠️ Yes (security)    │ ✅ No                │
│ Merge-friendly   │ ❌ Conflicts         │ ✅ Globally unique   │
│ Read performance │ ✅ Better            │ 🟡 Slightly worse    │
│ Write at scale   │ 🟡 Bottleneck        │ ✅ Distributed       │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Best for         │ Single-server OLTP   │ Distributed, APIs    │
└──────────────────┴──────────────────────┴──────────────────────┘
```

> 💡 **Pro Tip**: Use UUIDv7 (time-sortable) for the best of both worlds — globally unique AND sequential for index performance.

---

### Q10: What is Database Partitioning? Types?

```
PARTITIONING: Split ONE table into multiple physical pieces.
(Different from Sharding: Partitioning = same server, Sharding = different servers)

┌───────────────────────────────────────────────────────────────┐
│                    PARTITIONING TYPES                         │
├──────────────────┬────────────────────────────────────────────┤
│ Range            │ By date: Jan data, Feb data, Mar data     │
│                  │ Best for: Time-series, logs, events       │
├──────────────────┼────────────────────────────────────────────┤
│ List             │ By value: US orders, EU orders, APAC      │
│                  │ Best for: Geographic, categorical data    │
├──────────────────┼────────────────────────────────────────────┤
│ Hash             │ By hash(key): Evenly distribute rows      │
│                  │ Best for: Even distribution, no hot spots │
├──────────────────┼────────────────────────────────────────────┤
│ Composite        │ Range + Hash together                     │
│                  │ Best for: Large multi-dimensional data    │
└──────────────────┴────────────────────────────────────────────┘
```

```sql
-- PostgreSQL: Range partitioning by date
CREATE TABLE orders (
    order_id    BIGSERIAL,
    order_date  DATE NOT NULL,
    amount      DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

---

### Q11: What is the difference between Partitioning and Sharding?

```
┌──────────────────┬────────────────────────┬────────────────────────┐
│ Feature          │ Partitioning           │ Sharding               │
├──────────────────┼────────────────────────┼────────────────────────┤
│ Where            │ Same server            │ Different servers      │
│ Managed by       │ Database engine        │ Application / proxy    │
│ Complexity       │ 🟡 Moderate            │ 🔴 High               │
│ Cross-partition  │ ✅ Easy (same DB)      │ ❌ Hard (cross-network)│
│ queries          │                        │                        │
│ Scale limit      │ Single server limit    │ Unlimited              │
│ Availability     │ Single point failure   │ Shard-level failure    │
│ Use case         │ Manage large tables    │ Scale beyond 1 server  │
└──────────────────┴────────────────────────┴────────────────────────┘
```

---

### Q12: What is Referential Integrity?

```
REFERENTIAL INTEGRITY: Every FK value MUST reference an existing PK.

  orders.customer_id → customers.customer_id
  
  ✅ INSERT order with customer_id = 5 → customer 5 exists → OK
  ❌ INSERT order with customer_id = 999 → customer 999 missing → REJECTED
  ❌ DELETE customer 5 → has orders → REJECTED (or CASCADE)

  ON DELETE options:
    RESTRICT   → Prevent deletion (default)
    CASCADE    → Delete child rows too (dangerous!)
    SET NULL   → Set FK to NULL
    SET DEFAULT→ Set FK to default value
    NO ACTION  → Same as RESTRICT (checked at end of statement)
```

---

### Q13: What is the difference between OLTP and OLAP databases?

```
┌────────────────────┬──────────────────────┬─────────────────────────┐
│ Characteristic     │ OLTP                 │ OLAP                    │
├────────────────────┼──────────────────────┼─────────────────────────┤
│ Full name          │ Online Transaction   │ Online Analytical       │
│                    │ Processing           │ Processing              │
│ Operations         │ INSERT, UPDATE, DEL  │ SELECT (complex)        │
│ Query complexity   │ Simple, short        │ Complex aggregations    │
│ Response time      │ Milliseconds         │ Seconds to minutes      │
│ Data model         │ 3NF (normalized)     │ Star/Snowflake          │
│ Users              │ Thousands+           │ Analysts (dozens)       │
│ Data freshness     │ Real-time            │ Periodic refresh        │
│ Storage engine     │ Row-oriented         │ Column-oriented         │
│ Examples           │ PostgreSQL, MySQL    │ Redshift, BigQuery      │
│                    │ Oracle, SQL Server   │ Snowflake, ClickHouse   │
│ Operations         │ "Add item to cart"   │ "Revenue by region      │
│                    │                      │  last 5 years"          │
└────────────────────┴──────────────────────┴─────────────────────────┘
```

---

### Q14: What is Row-Oriented vs Column-Oriented Storage?

```
ROW-ORIENTED (OLTP):                 COLUMN-ORIENTED (OLAP):
Stores: Row1, Row2, Row3...         Stores: Col1 values, Col2 values...

  id | name  | salary               id:     [1, 2, 3, 4, 5]
  ---|-------|--------               name:   [Alice, Bob, Charlie...]
  1  | Alice | 120K   → on disk     salary: [120K, 110K, 90K...]
  2  | Bob   | 110K   → together    
  3  | Char  | 90K    →             

SELECT * FROM emp WHERE id=1;       SELECT AVG(salary) FROM emp;
  → Read 1 row (fast!) ✅            → Read only salary column ✅
  → Row-oriented wins               → Column-oriented wins

SELECT AVG(salary) FROM emp;        SELECT * FROM emp WHERE id=1;
  → Must read ALL columns 💀         → Must assemble from columns 🐌
  → Row-oriented loses               → Column-oriented loses

Row-Oriented: PostgreSQL, MySQL, Oracle, SQL Server
Column-Oriented: Redshift, BigQuery, ClickHouse, Cassandra, Parquet
```

---

### Q15: What is a Data Lake vs Data Warehouse vs Data Lakehouse?

```
┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Feature          │ Data Warehouse   │ Data Lake        │ Data Lakehouse   │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Data type        │ Structured       │ Any (raw)        │ Any (raw + curated)
│ Schema           │ Schema-on-write  │ Schema-on-read   │ Both             │
│ Processing       │ ETL              │ ELT              │ Both             │
│ Quality          │ High (curated)   │ Variable         │ Governed         │
│ Users            │ BI Analysts      │ Data Scientists  │ Both             │
│ Cost             │ $$$              │ $                │ $$               │
│ ACID             │ ✅               │ ❌               │ ✅ (Delta/Iceberg)
│ Examples         │ Snowflake,       │ S3, HDFS,        │ Databricks,      │
│                  │ Redshift         │ Azure Data Lake  │ Delta Lake       │
└──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

# 🔒 SECTION 2: ACID, Transactions & Concurrency (Q16–Q30)

---

### Q16: Explain ACID with a real-world example.

```
SCENARIO: Online Flight Booking

A = ATOMICITY (All or Nothing)
  Reserve seat + charge card + send confirmation
  If card fails → unreserve seat → no email
  All 3 happen, or NONE happen.

C = CONSISTENCY  
  Seat count never < 0. Business rules always valid.
  If 1 seat left and 2 people book → only 1 succeeds.

I = ISOLATION
  Two people booking same seat simultaneously:
  Each sees a consistent view. Winner gets seat, loser gets error.
  No "double booking."

D = DURABILITY
  Booking confirmed + server crashes = booking STILL exists after restart.
  Written to disk/WAL before confirmation.
```

---

### Q17: What is BASE? When would you choose it over ACID?

```
BASE:
  BA = Basically Available  → System always responds (maybe stale data)
  S  = Soft State           → Data may change over time without input
  E  = Eventually Consistent→ Given time, all nodes converge

CHOOSE BASE WHEN:
  ✅ Social media likes (exact count doesn't matter instantly)
  ✅ Product catalog (stale data for seconds is OK)
  ✅ Shopping cart (availability > consistency)
  ✅ DNS resolution (cached, eventually propagates)
  ✅ Analytics dashboards (near-real-time is fine)

CHOOSE ACID WHEN:
  ✅ Banking/Financial transactions
  ✅ Healthcare records
  ✅ Inventory management (overselling = $$$)
  ✅ Booking systems (no double-booking)
```

---

### Q18: What is a Dirty Read, Non-Repeatable Read, and Phantom Read?

```
DIRTY READ: 👻
  T1: UPDATE salary SET amount = 200K (not committed yet)
  T2: SELECT salary → sees 200K ← READING UNCOMMITTED DATA!
  T1: ROLLBACK → salary is back to 100K
  T2: Used 200K which NEVER existed! 💀

NON-REPEATABLE READ: 🔄
  T1: SELECT salary → 100K
  T2: UPDATE salary = 120K; COMMIT;
  T1: SELECT salary → 120K ← DIFFERENT VALUE on same query!

PHANTOM READ: 👻👻
  T1: SELECT COUNT(*) WHERE dept='Eng' → 5 employees
  T2: INSERT new employee in Eng; COMMIT;
  T1: SELECT COUNT(*) WHERE dept='Eng' → 6 employees ← NEW ROW appeared!

                  Dirty    Non-Repeatable    Phantom
READ UNCOMMITTED:  ✅          ✅               ✅      ← Never use this
READ COMMITTED:    ❌          ✅               ✅      ← Most common default
REPEATABLE READ:   ❌          ❌               ✅      ← MySQL default
SERIALIZABLE:      ❌          ❌               ❌      ← Safest, slowest
```

---

### Q19: What is MVCC? How do different databases implement it?

```
MVCC: Multi-Version Concurrency Control
"Readers don't block writers. Writers don't block readers."

HOW: Keep multiple versions of each row. Each transaction sees
a SNAPSHOT — a consistent view of data at a point in time.

┌─────────────────┬───────────────────────────────────────────┐
│ Database        │ MVCC Implementation                       │
├─────────────────┼───────────────────────────────────────────┤
│ PostgreSQL      │ Old versions stored IN the table itself   │
│                 │ → Needs VACUUM to clean dead tuples       │
│                 │ → xmin/xmax system columns track versions │
├─────────────────┼───────────────────────────────────────────┤
│ Oracle          │ Old versions stored in UNDO tablespace    │
│                 │ → "Consistent Read" from undo data        │
│                 │ → ORA-01555 if undo space exhausted       │
├─────────────────┼───────────────────────────────────────────┤
│ MySQL (InnoDB)  │ Old versions in undo log                  │
│                 │ → Purge thread cleans old versions        │
│                 │ → Read view created per transaction/stmt  │
├─────────────────┼───────────────────────────────────────────┤
│ SQL Server      │ Uses tempdb for version store (when RCSI) │
│                 │ → Row Versioning Isolation (RCSI/SI)      │
│                 │ → Default: locking (not MVCC!)            │
└─────────────────┴───────────────────────────────────────────┘
```

---

### Q20: What is a Write-Ahead Log (WAL)?

```
PRINCIPLE: Write the INTENT to a log BEFORE modifying actual data.

Why? Data pages are written at random locations (slow, crash-risky).
     Log writes are sequential (fast, crash-safe).

FLOW:
  1. Transaction: UPDATE salary = 120K
  2. Write to WAL: "Change row 42, column salary, from 100K to 120K"
  3. Return COMMIT OK to client
  4. (Later) Background process writes actual data page
  5. If crash before step 4 → REPLAY WAL on restart → data recovered ✅

NAMES ACROSS DATABASES:
  PostgreSQL → WAL (Write-Ahead Log) / pg_wal directory
  Oracle     → Redo Log (+ Archive Log for recovery)
  MySQL      → Redo Log (InnoDB) + Binary Log (replication)
  SQL Server → Transaction Log (.ldf file)
```

---

### Q21: Explain Pessimistic vs Optimistic Concurrency Control.

```
PESSIMISTIC: "Assume conflicts WILL happen. Lock preemptively."
  BEGIN;
  SELECT * FROM inventory WHERE id=1 FOR UPDATE;  -- LOCK row
  -- Calculate new stock...
  UPDATE inventory SET stock = stock - 1 WHERE id=1;
  COMMIT;  -- Release lock
  
  ✅ Best for: High contention (many writers on same rows)
  ❌ Risk: Deadlocks, reduced concurrency

OPTIMISTIC: "Assume conflicts are RARE. Check at commit time."
  SELECT *, version FROM inventory WHERE id=1;  -- version=5
  -- Calculate new stock...
  UPDATE inventory SET stock=stock-1, version=6 
  WHERE id=1 AND version=5;
  -- If 0 rows affected → conflict detected → RETRY
  
  ✅ Best for: Low contention (reads >> writes)
  ❌ Risk: Retries under high contention (starvation)
```

---

### Q22: What is a Deadlock? How do databases detect and resolve them?

```
DEADLOCK:
  T1: Locks Row A → Waits for Row B
  T2: Locks Row B → Waits for Row A
  → Circular dependency → NEITHER can proceed!

DETECTION:
  • Wait-for graph: Database builds a directed graph of "who waits for whom"
  • If the graph has a CYCLE → deadlock detected

RESOLUTION:
  • Choose a VICTIM (lowest cost transaction)
  • ROLLBACK the victim
  • The other transaction proceeds
  • Victim can retry

PREVENTION STRATEGIES:
  1. Lock resources in consistent ORDER (alphabetical, by ID)
  2. Keep transactions SHORT
  3. Use lower isolation levels when possible
  4. Use SKIP LOCKED / NOWAIT
  5. Reduce hot spots (batch updates vs row-by-row)
```

```sql
-- PostgreSQL: Check for deadlocks
SELECT * FROM pg_stat_activity WHERE wait_event_type = 'Lock';

-- SQL Server: Deadlock graph
-- Enable trace flag 1222 or use Extended Events

-- MySQL: Show engine status for latest deadlock
SHOW ENGINE INNODB STATUS;
```

---

### Q23: What is Two-Phase Commit (2PC)?

```
2PC: Ensures ALL participants in a distributed transaction 
     either ALL commit or ALL abort.

Phase 1 — PREPARE (Voting):
  Coordinator → "Can you commit?"
  Participant A → "YES, I can commit"
  Participant B → "YES, I can commit"
  
Phase 2 — COMMIT (Decision):
  Coordinator → "Everyone said YES → COMMIT"
  Participant A → COMMIT ✅
  Participant B → COMMIT ✅

  If ANY participant says NO:
  Coordinator → "ABORT ALL"
  Everyone → ROLLBACK ❌

PROBLEMS:
  • Coordinator failure = all participants BLOCKED (holding locks)
  • Slow (2 round-trip network calls minimum)
  • Not partition-tolerant

ALTERNATIVES:
  • Saga pattern (compensating transactions)
  • 3PC (Three-Phase Commit) — less blocking
  • Consensus protocols (Raft, Paxos)
```

---

### Q24: What is the difference between Shared Lock and Exclusive Lock?

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Lock Type        │ Shared (S) Lock      │ Exclusive (X) Lock   │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Purpose          │ Reading data         │ Writing/modifying    │
│ Multiple holders │ ✅ Yes               │ ❌ No (only one)     │
│ Blocks readers   │ ❌ No                │ ✅ Yes               │
│ Blocks writers   │ ✅ Yes               │ ✅ Yes               │
│ SQL              │ SELECT ... FOR SHARE │ SELECT ... FOR UPDATE│
│ Analogy          │ Reading a book       │ Editing a document   │
│                  │ (others can read too)│ (exclusive access)   │
└──────────────────┴──────────────────────┴──────────────────────┘

Compatibility Matrix:
          | Shared (S) | Exclusive (X)
Shared    |  ✅ OK     |  ❌ WAIT
Exclusive |  ❌ WAIT   |  ❌ WAIT
```

---

### Q25: What is Lock Escalation?

```
LOCK ESCALATION: Database automatically converts many fine-grained locks
into fewer coarse-grained locks to save memory.

Row Locks → Page Locks → Table Locks

Example (SQL Server):
  UPDATE top 6000 rows → 6000 row locks
  Memory for 6000 locks is expensive!
  SQL Server: "Let me just lock the whole table instead" → 1 table lock
  
  ✅ Saves memory
  ❌ Reduces concurrency (other transactions blocked)

Threshold (SQL Server): ~5000 locks on single table → escalate
Threshold (Oracle): Oracle does NOT do lock escalation! Always row-level.
```

---

### Q26: Explain Snapshot Isolation (SI) and Read Committed Snapshot Isolation (RCSI).

```
SNAPSHOT ISOLATION (SI):
  Transaction sees a SNAPSHOT of data as of transaction START.
  All reads return data from that snapshot, even if others commit changes.
  
  PostgreSQL: SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SQL Server: SET TRANSACTION ISOLATION LEVEL SNAPSHOT;

RCSI (Read Committed Snapshot Isolation):
  Each STATEMENT sees a snapshot as of statement START.
  Different statements in same transaction may see different data.
  
  SQL Server: ALTER DATABASE mydb SET READ_COMMITTED_SNAPSHOT ON;
  PostgreSQL: This is the DEFAULT behavior!

  SI:   Snapshot at TX start   → consistent across statements
  RCSI: Snapshot at STMT start → fresher but may vary within TX
```

---

### Q27: What are Savepoints?

```sql
BEGIN TRANSACTION;

INSERT INTO orders VALUES (1001, 'Product A');

SAVEPOINT sp1;  -- Mark a point

INSERT INTO orders VALUES (1002, 'Product B');

-- Oops, something wrong with 1002
ROLLBACK TO SAVEPOINT sp1;  -- Undo only 1002, keep 1001!

INSERT INTO orders VALUES (1003, 'Product C');

COMMIT;  -- 1001 and 1003 are committed. 1002 was rolled back.
```

**Use cases**: Partial rollbacks in complex transactions, error handling in stored procedures.

---

### Q28: What is Transaction Log Truncation? Why does the log grow?

```
TRANSACTION LOG GROWTH CAUSES:
  1. Long-running transactions (log can't be reused until TX completes)
  2. No log backups (SQL Server: FULL recovery model requires log backups)
  3. Replication lag (log must be kept until subscriber reads it)
  4. Uncommitted transactions
  5. Database mirroring/AG synchronization

FIX:
  SQL Server:
    BACKUP LOG mydb TO DISK = 'path';  -- Truncates inactive log
    
  PostgreSQL:
    CHECKPOINT;  -- Allows WAL recycling
    Archive old WAL files
    
  Oracle:
    ALTER SYSTEM SWITCH LOGFILE;  -- Force log switch
    Archive redo logs
```

---

### Q29: What is the difference between Implicit and Explicit Transactions?

```sql
-- IMPLICIT TRANSACTION (Auto-commit mode)
INSERT INTO orders VALUES (1, 'Product');
-- ↑ Automatically committed immediately!

-- EXPLICIT TRANSACTION (Manual control)
BEGIN TRANSACTION;
INSERT INTO orders VALUES (1, 'Product');
INSERT INTO order_items VALUES (1, 101, 2);
-- ↑ Nothing committed yet — we're in control
COMMIT;  -- NOW both are committed atomically

-- Auto-commit settings:
-- PostgreSQL: Default ON (each statement is its own TX)
-- MySQL:      Default ON (autocommit=1)
-- SQL Server: Default ON (implicit_transactions OFF)
-- Oracle:     Default OFF! (must explicitly COMMIT) ← Different!
```

> ⚠️ **Critical Oracle Gotcha**: Oracle doesn't auto-commit by default! If you close SQL*Plus without COMMIT, your changes are LOST. Every Oracle DBA has learned this the hard way.

---

### Q30: What are Advisory Locks?

```sql
-- PostgreSQL Advisory Locks: Application-level locking
-- (Not tied to any table or row — you define what they protect)

-- Session-level lock (released at session end)
SELECT pg_advisory_lock(12345);  -- Lock ID 12345
-- ... do critical section work ...
SELECT pg_advisory_unlock(12345);

-- Transaction-level lock (released at COMMIT/ROLLBACK)
SELECT pg_advisory_xact_lock(12345);

-- Non-blocking (try to acquire, return true/false)
SELECT pg_try_advisory_lock(12345);

-- Use case: Prevent duplicate cron jobs
-- Job runner: IF pg_try_advisory_lock(job_id) THEN run job ELSE skip
```

---

# ⚡ SECTION 3: Indexing & Query Optimization (Q31–Q45)

---

### Q31: How does a B-Tree index work internally?

```
B-TREE (Balanced Tree): The DEFAULT index in almost every database.

                    ┌─────────────┐
                    │  50 | 100   │          ← Root Node
                    └──┬──┬──┬───┘
                   /   │  │  │   \
         ┌────────┐ ┌──┴──┐ ┌┴────────┐
         │10|20|30│ │60|70│ │110|120   │    ← Branch Nodes
         └┬┬┬┬───┘ └┬┬┬──┘ └┬┬┬──────┘
          ││││       │││     │││
         Leaf Nodes (actual data pointers)   ← Leaf Nodes

PROPERTIES:
  • Self-balancing: All leaf nodes at same depth
  • Sorted: Enables range queries (BETWEEN, <, >, ORDER BY)
  • Branching factor: Each node has many children (100s)
  • Depth: Even 100M rows only need ~3-4 levels
  • O(log n) for search, insert, delete

WHY B-TREE WINS:
  100M rows → log₁₀₀(100,000,000) ≈ 4 disk reads
  vs Full Scan: 100,000,000 disk reads
```

---

### Q32: What is a B+Tree? How is it different from B-Tree?

```
B-TREE:                              B+TREE:
┌───────────────┐                    ┌───────────────┐
│ Keys + Data   │ ← all nodes       │ Keys only     │ ← internal nodes
│ at every level│   have data        │ (no data)     │
└───────────────┘                    └───────────────┘
                                     Leaf nodes:
                                     ┌──────────────────────────────┐
                                     │ Keys + Data + NEXT pointer  │
                                     │ [10]→[20]→[30]→[40]→[50]→  │
                                     └──────────────────────────────┘
                                     Linked list! Range scans ⚡

B+TREE ADVANTAGES:
  ✅ Leaves are linked → range queries just follow pointers
  ✅ Internal nodes are smaller (no data) → more keys per page
  ✅ More branching → shallower tree → fewer disk reads
  ✅ ALL data at leaf level → consistent access time
  
Most databases actually use B+Trees (even when they say "B-Tree"):
  PostgreSQL, MySQL, Oracle, SQL Server → all use B+Trees
```

---

### Q33: What is a Hash Index? When is it better than B-Tree?

```
HASH INDEX: Compute hash(key) → directly find the bucket.

  hash('Alice') → bucket 7 → Row pointer

  ✅ Exact match: WHERE name = 'Alice'  → O(1) lookup! ⚡
  ❌ Range queries: WHERE age > 25       → USELESS (no ordering)
  ❌ Sorting: ORDER BY name             → USELESS
  ❌ Prefix matching: LIKE 'Ali%'       → USELESS

USE HASH WHEN:
  • Only do equality lookups (=, IN)
  • Never need ORDER BY, range, or LIKE on that column
  • PostgreSQL: CREATE INDEX idx ON table USING hash(column);
  • MySQL: MEMORY engine supports hash indexes
  • Oracle: Does not support standalone hash indexes
```

---

### Q34: What is a Bitmap Index? When should you use it?

```
BITMAP INDEX: One bitmap (bit array) per distinct value.

Column "Gender" with values M, F:
  M: 1 0 1 1 0 1 0 1 0 1  (1 = row has 'M')
  F: 0 1 0 0 1 0 1 0 1 0  (1 = row has 'F')

WHERE gender = 'M' AND status = 'active':
  M bitmap:      1 0 1 1 0 1 0 1 0 1
  Active bitmap: 1 1 0 1 0 0 1 1 0 1
  AND result:    1 0 0 1 0 0 0 1 0 1  ← Rows 1, 4, 8, 10

✅ Low cardinality columns (gender, status, country)
✅ Complex AND/OR/NOT combinations (data warehousing)
✅ Read-heavy workloads
❌ High cardinality (name, email — too many bitmaps)
❌ Write-heavy (updating bitmap = expensive locking)

Supported: Oracle (extensively), PostgreSQL (BitmapAnd/Or in plans)
Not native: MySQL, SQL Server (but query optimizer creates internal bitmaps)
```

---

### Q35: What is a Covering Index?

```
COVERING INDEX: An index that contains ALL columns needed by a query.
The database never needs to access the actual table → "Index-Only Scan" ⚡

Query: SELECT name, salary FROM emp WHERE department = 'Engineering';

Regular index on (department):
  1. Index lookup → find row IDs for 'Engineering'
  2. Table access → fetch name, salary from actual rows (RANDOM I/O!)

Covering index on (department) INCLUDE (name, salary):
  1. Index lookup → find entries with all needed data
  2. Return directly from index → NO table access! ⚡

Performance boost: 10x-100x for covered queries
```

```sql
-- PostgreSQL (INCLUDE syntax, 11+)
CREATE INDEX idx_cover ON employees(department) INCLUDE (emp_name, salary);

-- SQL Server (INCLUDE syntax)
CREATE INDEX idx_cover ON employees(department) INCLUDE (emp_name, salary);

-- MySQL (include columns in index directly)
CREATE INDEX idx_cover ON employees(department, emp_name, salary);
```

---

### Q36: What is Index Selectivity? Why does it matter?

```
SELECTIVITY = Number of DISTINCT values / Total rows

High Selectivity (GOOD for indexing):
  email column: 1,000,000 distinct / 1,000,000 rows = 1.0 ← EXCELLENT
  → Index on email is very useful

Low Selectivity (BAD for indexing):
  gender column: 2 distinct / 1,000,000 rows = 0.000002 ← TERRIBLE
  → Index on gender is useless (50% of table matches each value)

RULE OF THUMB:
  Selectivity > 10-15% → Index may not be used (full scan is cheaper)
  Selectivity < 5% → Index is very effective

  The optimizer looks at column STATISTICS to decide.
```

---

### Q37: What is an Execution Plan? How do you read one?

```
EXECUTION PLAN: The database's STEP-BY-STEP strategy to execute your query.

PostgreSQL:
  EXPLAIN ANALYZE SELECT * FROM employees WHERE department = 'Eng';

  Seq Scan on employees  (cost=0.00..25.00 rows=5 width=68) (actual time=0.015..0.020 rows=4)
    Filter: (department = 'Eng'::text)
    Rows Removed by Filter: 6

READING THE PLAN (inside-out, bottom-up):
  ┌──────────────────────────────────────────────────────────────┐
  │ KEY METRICS:                                                 │
  │                                                              │
  │ cost=0.00..25.00                                             │
  │   ↑ startup   ↑ total  (arbitrary units, not ms)            │
  │                                                              │
  │ rows=5        → Estimated rows (from statistics)             │
  │ actual rows=4 → Real rows (only with ANALYZE)               │
  │   If estimated ≠ actual → STALE STATISTICS! 🚨               │
  │                                                              │
  │ width=68      → Average row size in bytes                    │
  │                                                              │
  │ OPERATION TYPES:                                             │
  │   Seq Scan       → Full table scan (no index) 🔴             │
  │   Index Scan     → Using index + table lookup 🟡             │
  │   Index Only Scan→ All data from index alone 🟢 ⚡           │
  │   Bitmap Scan    → Build bitmap, then heap access 🟡        │
  │   Nested Loop    → For each row in A, scan B 🔴 on large    │
  │   Hash Join      → Build hash table, probe 🟢 on large     │
  │   Merge Join     → Merge sorted inputs 🟢 on sorted data   │
  │   Sort           → In-memory or disk sort 🟡                │
  └──────────────────────────────────────────────────────────────┘
```

---

### Q38: What makes a query non-SARGable?

```
SARGable = Search ARGument able → Can the WHERE clause use an index?

❌ NON-SARGABLE (kills index usage):
  WHERE YEAR(hire_date) = 2024        ← Function on column
  WHERE UPPER(name) = 'ALICE'         ← Function on column  
  WHERE salary * 1.1 > 100000         ← Arithmetic on column
  WHERE name LIKE '%smith'            ← Leading wildcard
  WHERE CAST(id AS VARCHAR) = '100'   ← Type conversion on column
  WHERE col1 + col2 > 100             ← Expression on columns
  WHERE col IS NOT NULL               ← Depends on optimizer

✅ SARGABLE (index-friendly):
  WHERE hire_date >= '2024-01-01' AND hire_date < '2025-01-01'
  WHERE name = 'Alice'
  WHERE salary > 90909                ← Move arithmetic to right side
  WHERE name LIKE 'Smith%'            ← Trailing wildcard OK
  WHERE id = 100                      ← No conversion needed
```

> 💡 **Golden Rule**: Keep the indexed column NAKED on the left side. Move all functions, arithmetic, and conversions to the right side.

---

### Q39: What is Index Fragmentation? How do you fix it?

```
FRAGMENTATION: Over time, INSERTs/UPDATEs/DELETEs cause index pages
to become out-of-order or partially empty.

Types:
  Internal Fragmentation: Pages are partially empty (wasted space)
  External Fragmentation: Logical order ≠ physical order on disk

Impact:
  More I/O → More disk reads → Slower range scans

FIX:
  SQL Server:
    ALTER INDEX idx_name ON table REBUILD;      -- >30% fragmentation
    ALTER INDEX idx_name ON table REORGANIZE;   -- 10-30% fragmentation

  PostgreSQL:
    REINDEX INDEX idx_name;                      -- Rebuilds index
    -- Or let autovacuum handle it

  Oracle:
    ALTER INDEX idx_name REBUILD;

  MySQL:
    ALTER TABLE table ENGINE=InnoDB;             -- Rebuilds table + indexes
    OPTIMIZE TABLE table;
```

---

### Q40: What is the difference between Clustered and Non-Clustered Index?

```
CLUSTERED:
  • The table DATA is physically sorted by the index key
  • Only ONE per table (data can only be sorted one way)
  • The leaf level of the index IS the data
  • Primary Key → usually Clustered Index (auto in SQL Server/MySQL)

NON-CLUSTERED:
  • Separate structure pointing to the data
  • MANY per table (SQL Server: up to 999)
  • Leaf level contains pointers (row locators)
  • Like a book's back-of-book index

PostgreSQL:
  • No clustered index concept (heap tables)
  • CLUSTER command sorts table once (doesn't maintain order)
  • All indexes are non-clustered

MySQL (InnoDB):
  • Primary Key IS the clustered index (always!)
  • Secondary indexes point to Primary Key (not row pointer)
  • → Secondary index lookup = 2 lookups (index → PK → data)
```

---

### Q41: How do you decide which columns to index?

```
INDEX DECISION FRAMEWORK:

✅ INDEX THESE:
  1. Primary Key columns (automatic)
  2. Foreign Key columns (JOIN performance!)
  3. Columns in WHERE clauses (frequent filters)
  4. Columns in ORDER BY (avoid sort operations)
  5. Columns in GROUP BY (aggregation performance)
  6. Columns with HIGH selectivity (email, user_id)

❌ DON'T INDEX THESE:
  1. Low cardinality columns alone (gender, boolean)
  2. Columns in small tables (< 1000 rows → full scan is fine)
  3. Heavily updated columns (index maintenance overhead)
  4. Wide columns (TEXT, BLOB → index too large)
  5. Columns never used in queries

⚠️ INDEX ANTI-PATTERNS:
  • Too many indexes: Slows down INSERTs/UPDATEs
  • Unused indexes: Waste storage + write overhead
  • Redundant indexes: (A,B) already covers queries on (A)
  • Wrong column order in composite index
```

---

### Q42: What is a Partial (Filtered) Index?

```sql
-- Only index rows matching a condition
-- Index active orders (ignore 95% completed orders)

-- PostgreSQL
CREATE INDEX idx_active ON orders(customer_id, order_date)
WHERE status = 'active';

-- SQL Server  
CREATE INDEX idx_active ON orders(customer_id, order_date)
WHERE status = 'active';

-- Oracle: Function-based index with CASE
CREATE INDEX idx_active ON orders(
    CASE WHEN status = 'active' THEN customer_id END,
    CASE WHEN status = 'active' THEN order_date END
);

BENEFITS:
  ✅ Much smaller index (only active rows)
  ✅ Fits in memory → faster lookups
  ✅ Less write overhead (only updates active rows)
  ✅ Perfect for: soft deletes, status columns, recent data
```

---

### Q43: What is Statistics? Why should you update them?

```
STATISTICS: Metadata about data DISTRIBUTION in tables and indexes.
The query optimizer uses statistics to choose the best execution plan.

What statistics contain:
  • Number of rows in table
  • Number of distinct values per column
  • Data distribution (histogram of values)
  • Average row width
  • Correlation between physical and logical ordering

OUTDATED STATISTICS = BAD PLANS:
  Statistics say: "department 'Engineering' has 10 rows" → Index Seek
  Reality:        "department 'Engineering' now has 10,000 rows" → Should be Full Scan!
  Result:         10,000 individual index lookups → SLOWER than full scan 💀

UPDATE STATISTICS:
  PostgreSQL: ANALYZE;                    -- All tables
              ANALYZE employees;          -- Specific table
              -- autovacuum does this automatically

  SQL Server: UPDATE STATISTICS employees; -- Specific table
              sp_updatestats;              -- All tables

  Oracle:     EXEC DBMS_STATS.GATHER_TABLE_STATS('schema', 'employees');

  MySQL:      ANALYZE TABLE employees;
```

---

### Q44: What is Query Hint? Should you use them?

```sql
-- Query hints FORCE the optimizer to use a specific strategy.

-- SQL Server
SELECT * FROM employees WITH (INDEX(idx_dept))  -- Force specific index
WHERE department = 'Engineering';

SELECT * FROM orders OPTION (HASH JOIN);  -- Force hash join

-- Oracle
SELECT /*+ INDEX(e idx_dept) */ * FROM employees e  -- Force index
WHERE department = 'Engineering';

SELECT /*+ PARALLEL(e, 4) */ * FROM employees e;  -- Force parallelism

-- PostgreSQL (limited hints, use pg_hint_plan extension)
SET enable_seqscan = off;  -- Discourage sequential scans

-- MySQL
SELECT * FROM employees FORCE INDEX (idx_dept)
WHERE department = 'Engineering';
```

**Should you use hints?**
```
❌ Usually NO:
  • Hints bypass the optimizer's intelligence
  • Data changes → hint may become wrong
  • Creates maintenance burden
  
✅ Sometimes YES:
  • Optimizer consistently chooses bad plan
  • Parameter sniffing issues
  • Known data distribution quirks
  • Temporary fix while investigating root cause
```

---

### Q45: What is the difference between EXPLAIN and EXPLAIN ANALYZE?

```
EXPLAIN:  Shows the PLANNED execution strategy (estimates only)
          Fast — doesn't actually run the query
          
EXPLAIN ANALYZE: Runs the query AND shows actual execution metrics
                 Slower — must execute the query completely!

PostgreSQL Example:

  EXPLAIN SELECT * FROM employees WHERE salary > 100000;
  ┌─────────────────────────────────────────────────────────────┐
  │ Seq Scan on employees (cost=0.00..1.12 rows=4 width=68)   │
  │   Filter: (salary > 100000)                                │
  │ Estimated only! ☝️                                          │
  └─────────────────────────────────────────────────────────────┘

  EXPLAIN ANALYZE SELECT * FROM employees WHERE salary > 100000;
  ┌─────────────────────────────────────────────────────────────┐
  │ Seq Scan on employees (cost=0.00..1.12 rows=4 width=68)   │
  │   (actual time=0.013..0.015 rows=3 loops=1)               │
  │   Filter: (salary > 100000)                                │
  │   Rows Removed by Filter: 7                                │
  │ Planning Time: 0.048 ms                                    │
  │ Execution Time: 0.032 ms                                   │
  │ Real metrics! ☝️                                            │
  └─────────────────────────────────────────────────────────────┘

⚠️ WARNING: EXPLAIN ANALYZE on UPDATE/DELETE actually MODIFIES data!
   Use: BEGIN; EXPLAIN ANALYZE UPDATE ...; ROLLBACK;
```

---

# 🌐 SECTION 4: CAP Theorem & Distributed Systems (Q46–Q55)

---

### Q46: Explain the CAP Theorem in simple terms.

```
CAP THEOREM: In a distributed system, you can only guarantee 
TWO of these THREE properties:

  C = Consistency:  Every read gets the most recent write
  A = Availability: Every request gets a response (no errors)
  P = Partition Tolerance: System works despite network failures

IN REALITY:
  Network partitions WILL happen (you can't avoid P).
  So the REAL choice is: CP or AP?

  CP (Consistency + Partition Tolerance):
    "I'd rather return an error than stale data"
    Examples: MongoDB (strong reads), HBase, Redis (single), PostgreSQL

  AP (Availability + Partition Tolerance):
    "I'd rather return stale data than an error"
    Examples: Cassandra, DynamoDB, CouchDB, DNS

  CA (Consistency + Availability):
    "Only possible if no network partitions" → Single-node databases
    Examples: Single PostgreSQL, Single MySQL (not distributed!)
```

---

### Q47: What is Eventual Consistency? Give real-world examples.

```
EVENTUAL CONSISTENCY: If no new updates are made, eventually ALL replicas
will converge to the same value.

"How eventual is eventually?"
  DNS:         24-48 hours (TTL-based)
  Cassandra:   Milliseconds to seconds (anti-entropy repair)
  DynamoDB:    Typically < 1 second
  S3:          Milliseconds (now strongly consistent for overwrite PUTs!)

REAL-WORLD EXAMPLES:
  ✅ Social media "likes" count → 1,340 vs 1,342 → who cares?
  ✅ Product review count → "~4,500 reviews" → approximate is fine
  ✅ Search engine index → Results may be slightly stale → acceptable
  ✅ CDN cache → Serve slightly old page → better than slow/unavailable
  
  ❌ Bank balance → "You had $500... or was it $0?" → UNACCEPTABLE
  ❌ Inventory count → "In stock!" → Actually sold out → BAD
  ❌ Flight seat → "Available!" → Actually taken → VERY BAD
```

---

### Q48: What is Quorum in distributed databases?

```
QUORUM: The minimum number of nodes that must agree for an operation 
to be considered successful.

Formula: N = Total replicas, W = Write quorum, R = Read quorum

STRONG CONSISTENCY when: W + R > N

Example with N=3 replicas:
  W=2, R=2: W+R=4 > 3 → Strong consistency ✅
  W=1, R=3: W+R=4 > 3 → Strong consistency ✅ (fast writes, slow reads)
  W=3, R=1: W+R=4 > 3 → Strong consistency ✅ (slow writes, fast reads)
  W=1, R=1: W+R=2 < 3 → Eventual consistency ⚠️ (but fastest!)

COMMON PATTERNS:
  Write: QUORUM (2/3)  + Read: QUORUM (2/3) → Strong + balanced
  Write: ALL (3/3)     + Read: ONE (1/3)     → Strongest write, fast read
  Write: ONE (1/3)     + Read: ONE (1/3)     → Fastest, eventual consistency

  Cassandra, DynamoDB, Riak all use this model.
```

---

### Q49: What is the PACELC Theorem?

```
PACELC extends CAP to address the normal (non-partitioned) case:

  IF there's a Partition (P):
    Choose between Availability (A) and Consistency (C)
  ELSE (normal operation):
    Choose between Latency (L) and Consistency (C)

┌──────────────────────┬────────────────────┬────────────────────┐
│ Database             │ During Partition   │ Normal Operation   │
├──────────────────────┼────────────────────┼────────────────────┤
│ Cassandra            │ PA (Available)     │ EL (Low latency)   │
│ DynamoDB             │ PA (Available)     │ EL (Low latency)   │
│ MongoDB              │ PC (Consistent)    │ EC (Consistent)    │
│ PostgreSQL (single)  │ PC (Consistent)    │ EC (Consistent)    │
│ CockroachDB          │ PC (Consistent)    │ EC (Consistent)    │
│ Cosmos DB            │ PA or PC (tunable) │ EL or EC (tunable) │
└──────────────────────┴────────────────────┴────────────────────┘

This is more practical than CAP because most of the time there's NO partition,
and you're choosing between fast vs correct.
```

---

### Q50: What is a Split Brain problem?

```
SPLIT BRAIN: Network partition causes TWO nodes to both think 
they're the PRIMARY/LEADER → both accept writes → DATA DIVERGENCE!

  Normal:
  ┌──────┐        ┌──────┐
  │Primary│←─────→│Replica│    Healthy replication
  └──────┘        └──────┘
  
  Split Brain:
  ┌──────┐   ✂️   ┌──────┐
  │"I'm   │  NET  │"I'm   │   Both accept writes!
  │primary"│ DOWN  │primary"│   Data diverges! 💀
  └──────┘        └──────┘

PREVENTION:
  • Quorum-based election: Need majority to become primary
    (3 nodes → need 2 votes, 5 nodes → need 3 votes)
  • Fencing tokens: Old primary's writes rejected by storage
  • STONITH (Shoot The Other Node In The Head): Force-kill the old primary
  • Consensus protocols: Raft, Paxos (used by etcd, CockroachDB)
```

---

### Q51: What is Consistent Hashing?

```
CONSISTENT HASHING: Distribute data across nodes in a ring.
Adding/removing a node only affects its neighbors, not ALL data.

Traditional Hashing: hash(key) % N_nodes
  Problem: Change N → ALL keys move to different nodes! 💀

Consistent Hashing: 
  Place nodes on a circle (0 to 2^32)
  hash(key) → find next node clockwise

  Adding Node D:
  ┌─────────────────────────────┐
  │         A                   │
  │       ╱   ╲                 │  Only keys between C and D
  │     ╱       ╲               │  move to D.
  │   D    ●     B              │  Keys for A and B: unchanged!
  │     ╲       ╱               │
  │       ╲   ╱                 │
  │         C                   │
  └─────────────────────────────┘

Used by: Cassandra, DynamoDB, Memcached, Redis Cluster
Virtual nodes: Each physical node gets multiple positions → even distribution
```

---

### Q52: What is a Vector Clock?

```
VECTOR CLOCK: Track causality in distributed systems.
Each node maintains a counter for EVERY node.

Example (3 nodes: A, B, C):

  A: [A:1, B:0, C:0]  → A writes first
  A syncs to B:
  B: [A:1, B:1, C:0]  → B acknowledges + does own work
  
  Meanwhile:
  C: [A:0, B:0, C:1]  → C writes independently

  Now comparing B and C:
  B: [A:1, B:1, C:0]
  C: [A:0, B:0, C:1]
  → CONFLICT! Neither "happened before" the other.
  → Application must resolve (last-write-wins, merge, prompt user)

Used by: DynamoDB, Riak, Voldemort
Alternative: Lamport timestamps (simpler but less info)
```

---

### Q53: What is Gossip Protocol?

```
GOSSIP PROTOCOL: Nodes randomly share information with peers.
Like how rumors spread in a social group! 🗣️

Round 1: Node A tells Node B: "Node C is alive"
Round 2: Node B tells Node D: "Node A and C are alive"
Round 3: Node D tells Node E: "A, B, C alive"
...eventually ALL nodes know about everyone.

Properties:
  ✅ Scalable (no central coordinator)
  ✅ Fault-tolerant (no single point of failure)
  ✅ Eventually consistent (converges over time)
  ❌ Not immediate (takes O(log N) rounds)

Used by: Cassandra, DynamoDB, Consul, Serf
Purpose: Membership detection, failure detection, metadata propagation
```

---

### Q54: What is a Consensus Algorithm? Compare Raft vs Paxos.

```
CONSENSUS: How distributed nodes agree on a single value.

┌────────────────┬──────────────────────┬──────────────────────┐
│ Feature        │ Paxos                │ Raft                 │
├────────────────┼──────────────────────┼──────────────────────┤
│ Complexity     │ 🔴 Very complex      │ 🟡 Understandable    │
│ Inventor       │ Leslie Lamport (1989)│ Ongaro & Ousterhout  │
│ Leader         │ Not required         │ Strong leader        │
│ Phases         │ Prepare → Accept     │ Election → Replication│
│ Practical use  │ Google Chubby        │ etcd, CockroachDB,   │
│                │ (theoretical)        │ TiDB, Consul         │
│ Understandable │ "Only 3 people in    │ Designed for         │
│                │  world understand it"│ understandability     │
└────────────────┴──────────────────────┴──────────────────────┘

RAFT IN SIMPLE TERMS:
  1. ELECTION: If no heartbeat from leader → start election
  2. Candidate requests votes from majority
  3. Majority votes → becomes LEADER
  4. Leader replicates log entries to followers
  5. Entry committed when MAJORITY acknowledges
  6. If leader fails → new election
```

---

### Q55: What is Linearizability vs Serializability?

```
LINEARIZABILITY (Single-operation guarantee):
  "Once a write is acknowledged, ALL subsequent reads see it."
  "Operations appear to happen at a single instant in time."
  → About individual reads/writes, external time ordering
  → Relevant for: Distributed systems (replicas)

SERIALIZABILITY (Transaction guarantee):
  "Concurrent transactions produce the same result as 
   SOME serial (one-at-a-time) execution."
  → About groups of operations (transactions)
  → Relevant for: Database isolation levels

STRICT SERIALIZABILITY = Both combined
  → The gold standard. Used by CockroachDB, Google Spanner.
  → Very expensive but guarantees correctness.
```

---

# 📊 SECTION 5: Replication, Sharding & Scaling (Q56–Q70)

---

### Q56: What is Database Replication? Types?

```
REPLICATION: Copying data from one database to others.

┌────────────────────────────────────────────────────────────────┐
│                REPLICATION TYPES                                │
├──────────────────┬─────────────────────────────────────────────┤
│ Single-Leader    │ One primary (read/write)                    │
│ (Master-Slave)   │ Multiple replicas (read-only)               │
│                  │ ✅ Simple, consistent writes                │
│                  │ ❌ Single write bottleneck                  │
│                  │ Used by: PostgreSQL, MySQL, MongoDB         │
├──────────────────┼─────────────────────────────────────────────┤
│ Multi-Leader     │ Multiple primaries (each accepts writes)    │
│ (Master-Master)  │ ✅ Write availability in multiple regions   │
│                  │ ❌ Conflict resolution needed               │
│                  │ Used by: CockroachDB, MySQL Group Repl.    │
├──────────────────┼─────────────────────────────────────────────┤
│ Leaderless       │ Any node accepts reads AND writes           │
│                  │ Quorum-based consistency                    │
│                  │ ✅ High availability, no leader election    │
│                  │ ❌ Complex conflict resolution              │
│                  │ Used by: Cassandra, DynamoDB, Riak          │
└──────────────────┴─────────────────────────────────────────────┘
```

---

### Q57: What is Synchronous vs Asynchronous Replication?

```
SYNCHRONOUS:
  Primary: WRITE data → Wait for replica ACK → Return to client
  ✅ Zero data loss (replica always up-to-date)
  ❌ Slower (must wait for network round-trip)
  ❌ If replica is down → primary blocks
  Used for: Financial systems, critical data

ASYNCHRONOUS:
  Primary: WRITE data → Return to client → (later) send to replica
  ✅ Fast (no waiting)
  ❌ Data loss possible if primary crashes before replicating
  ❌ Replication lag (replica may be seconds/minutes behind)
  Used for: Read replicas, analytics, most web apps

SEMI-SYNCHRONOUS:
  At least ONE replica must ACK (others async)
  ✅ Balance of safety and speed
  Used by: MySQL semi-sync, PostgreSQL synchronous_commit
```

---

### Q58: What is Replication Lag? How do you handle it?

```
REPLICATION LAG: Time delay between primary write and replica visibility.

PROBLEMS:
  User writes → reads from replica → doesn't see own write! 🤯

  User: "I updated my profile to Seattle"
  App: SELECT city FROM users WHERE id=1;  (hits replica)
  Result: "New York" ← Still old data! (replica hasn't caught up)

SOLUTIONS:
  1. Read-after-write consistency:
     → Route user's reads to PRIMARY for data they just wrote
     → For X seconds after a write, read from primary
     
  2. Monotonic reads:
     → Always route same user to same replica (sticky sessions)
     → Prevents reading from a LESS updated replica
     
  3. Causal consistency:
     → Track dependencies, ensure reads respect write ordering
     
  4. Synchronous replication:
     → No lag, but slower writes
```

---

### Q59: What is Database Sharding? Compare strategies.

```
SHARDING: Horizontally split data across MULTIPLE database instances.

            ┌─────────────────────────────────────────┐
            │          100M Users                      │
            │    (too much for one server)             │
            └──────┬──────────┬──────────┬────────────┘
                   │          │          │
              ┌────┴────┐ ┌──┴────┐ ┌──┴────┐
              │ Shard 1 │ │Shard 2│ │Shard 3│
              │ A-H     │ │ I-P   │ │ Q-Z   │
              │ users   │ │ users │ │ users │
              └─────────┘ └───────┘ └───────┘

STRATEGIES:
┌──────────────┬─────────────────────────┬────────────────────┐
│ Strategy     │ How                     │ Trade-offs         │
├──────────────┼─────────────────────────┼────────────────────┤
│ Hash-based   │ shard = hash(key) % N   │ Even dist, hard to │
│              │                         │ add shards          │
│ Range-based  │ key ranges per shard    │ Hot spots possible │
│ Directory    │ Lookup table maps →shard│ Flexible, extra hop│
│ Geo-based    │ By region/location      │ Low latency,       │
│              │                         │ uneven sizes        │
└──────────────┴─────────────────────────┴────────────────────┘

CHALLENGES:
  ❌ Cross-shard queries (JOINs across shards)
  ❌ Distributed transactions
  ❌ Resharding when adding/removing shards
  ❌ Hotspots (one shard gets more traffic)
  ❌ Schema changes must be applied to ALL shards
```

---

### Q60: What is a Shard Key? How do you choose one?

```
SHARD KEY: The column(s) used to determine which shard a row lives on.

GOOD SHARD KEYS:
  ✅ High cardinality (many distinct values → even distribution)
  ✅ Used in most queries (avoid cross-shard queries)
  ✅ Even distribution (no hot spots)
  ✅ Immutable (changing shard key = moving data between shards!)

BAD SHARD KEYS:
  ❌ Timestamps → all new writes go to SAME shard (hot spot!)
  ❌ Low cardinality → uneven distribution (country: 50% = US)
  ❌ Monotonically increasing → always hits last shard
  ❌ Frequently changed → expensive data migration

EXAMPLES:
  E-commerce: user_id (queries usually scoped to one user)
  Social media: user_id (each user's posts on same shard)
  Multi-tenant SaaS: tenant_id (tenant data co-located)
  IoT: device_id + date (compound key for distribution + time queries)
```

---

### Q61: What is Read Replica? How does it help scaling?

```
READ REPLICA: A copy of the primary database that handles ONLY read queries.

  ┌──────────┐         ┌──────────────┐
  │  App     │──Write──→│   Primary    │
  │  Server  │         │   (Leader)   │
  │          │         └──────┬───────┘
  │          │                │ Replication
  │          │         ┌──────┴──────────────────┐
  │          │──Read──→│ Replica 1  │  Replica 2 │
  └──────────┘         └─────────────────────────┘

BENEFITS:
  ✅ Scale reads horizontally (add more replicas)
  ✅ Offload analytics/reports from primary
  ✅ Geographic distribution (replica in each region)
  ✅ Hot standby for failover

LIMITATIONS:
  ❌ Replication lag (reads may be stale)
  ❌ Doesn't help with write scaling
  ❌ More infrastructure to manage
```

---

### Q62: What is Connection Pooling? Why is it essential?

```
WITHOUT POOLING:                     WITH POOLING:
Every request → new connection       Connections reused from pool

  Request 1 → Open → Query → Close    Request 1 → Get from pool → Query → Return
  Request 2 → Open → Query → Close    Request 2 → Get from pool → Query → Return
  Request 3 → Open → Query → Close    Request 3 → Get from pool → Query → Return
  
  Each "Open" = TCP handshake +        Pool maintains long-lived connections
  Authentication + Process setup       No setup overhead per request!
  ~50-200ms overhead per connection    ~0ms overhead

POOL SIZING (PostgreSQL community formula):
  pool_size = (core_count * 2) + spindle_count
  Example: 4-core server, SSD → (4*2) + 1 = 9 connections
  
  Common mistake: Setting pool too LARGE
  More connections ≠ more throughput
  Hundreds of connections → context switching → SLOWER! 📉
```

---

### Q63: What is a Proxy in database architecture?

```
DATABASE PROXY: Sits between app and database. Routes queries intelligently.

  ┌──────┐         ┌─────────┐         ┌──────────┐
  │ App  │────────→│  Proxy  │────────→│ Primary  │ (writes)
  │      │         │         │────────→│ Replica  │ (reads)
  └──────┘         │ Routes  │────────→│ Replica  │ (reads)
                   │ queries │         └──────────┘
                   └─────────┘

FEATURES:
  • Read/Write splitting (SELECT → replica, INSERT → primary)
  • Connection multiplexing (1000 app conns → 20 DB conns)
  • Query caching
  • Load balancing across replicas
  • Failover (detect dead primary, promote replica)
  • Query routing (shard-aware routing)

POPULAR PROXIES:
  PostgreSQL: PgBouncer, Pgpool-II, Odyssey
  MySQL:      ProxySQL, MySQL Router, MaxScale
  MongoDB:    mongos (built-in shard router)
  General:    HAProxy, Vitess (YouTube's MySQL scaling layer)
```

---

### Q64: What is the difference between Vertical and Horizontal Scaling?

```
VERTICAL (Scale Up):                 HORIZONTAL (Scale Out):
  Add more RAM, CPU, disk            Add more servers
  to the SAME machine                
                                     
  ┌─────────────────┐               ┌───┐ ┌───┐ ┌───┐
  │                 │               │ S1│ │ S2│ │ S3│
  │   BIG SERVER    │               └───┘ └───┘ └───┘
  │   128 GB RAM    │               
  │   64 cores      │               ✅ Theoretically unlimited
  │                 │               ✅ Fault tolerant
  └─────────────────┘               ❌ Complex (distributed queries)
                                    ❌ Cross-node transactions hard
  ✅ Simple                         
  ✅ No code changes                
  ❌ Physical limits                 
  ❌ Single point of failure         
  ❌ Gets expensive fast             

STRATEGY:
  Start vertical → when you hit limits → go horizontal
  Most apps never need horizontal scaling (vertical handles up to ~1TB RAM)
```

---

### Q65: What is a Hot Spot in database systems?

```
HOT SPOT: One shard/partition/row gets disproportionately more traffic.

EXAMPLES:
  Celebrity tweet → millions reading ONE row
  Black Friday → everyone buying from ONE category
  Timestamp shard key → all new data hits LATEST shard
  
SOLUTIONS:
  • Add random prefix to shard key (spread writes)
    user_42 → shard_7_user_42 (random 0-9 prefix → 10x spread)
  • Salting: hash(key + salt) → different shards for hot key
  • Rate limiting: Protect the hot partition
  • Caching: Cache hot data in Redis/Memcached
  • Separate tables: Move hot data to dedicated infrastructure
  
EXAMPLE (Bad → Good):
  Bad shard key:  created_at → all inserts hit latest partition
  Good shard key: user_id → writes distributed across users
  Compound key:   (user_id, created_at) → good distribution + time queries
```

---

### Q66: What is Change Data Capture (CDC)?

```
CDC: Capture every change (INSERT/UPDATE/DELETE) from a database
and stream it to other systems in real-time.

  ┌──────────┐     CDC        ┌────────────────────┐
  │ Database │───────────────→│ Kafka / Event Bus  │
  │ (Source) │  Transaction   │                    │
  └──────────┘  Log Mining    └────────┬───────────┘
                                       │
                              ┌────────┼──────────┐
                              ↓        ↓          ↓
                         ┌────────┐ ┌───────┐ ┌───────┐
                         │ Search │ │ Cache │ │ DW    │
                         │ Index  │ │ Redis │ │ BQ    │
                         └────────┘ └───────┘ └───────┘

HOW IT WORKS:
  • Log-based: Read transaction log (WAL/binlog/redo)
    → Most efficient, no impact on source DB
  • Trigger-based: Database triggers write to changelog table
    → Simple but adds overhead to every write
  • Timestamp-based: Poll for rows modified since last check
    → Simple but misses deletes, has lag

TOOLS: Debezium (most popular), Oracle GoldenGate, AWS DMS,
       Fivetran, Airbyte
```

---

### Q67: What is Database Failover? Explain the process.

```
FAILOVER: Switching from a failed primary to a standby replica.

  AUTOMATIC FAILOVER PROCESS:
  
  1. DETECTION: Health check fails (heartbeat timeout)
     Primary: ❌ No response for 30 seconds
  
  2. ELECTION: Choose new primary
     Replicas vote → most up-to-date wins
  
  3. PROMOTION: Standby becomes primary
     New primary: "I'm now accepting writes!"
  
  4. REROUTING: Update connection strings / DNS
     Application → points to new primary
  
  5. FENCING: Prevent old primary from accepting writes
     Old primary (if it comes back): "You're no longer primary!"

FAILOVER METRICS:
  RTO (Recovery Time Objective): How long until system is back
    → Typical: 30 seconds to 5 minutes
  
  RPO (Recovery Point Objective): How much data can you lose
    → Sync replication: 0 (zero data loss)
    → Async replication: Seconds of data loss possible

TOOLS:
  PostgreSQL: Patroni, pg_auto_failover, repmgr
  MySQL:      MHA, Orchestrator, InnoDB Cluster
  SQL Server: Always On Availability Groups
  Oracle:     Data Guard
```

---

### Q68: What is Multi-Region Database Deployment?

```
MULTI-REGION: Database replicas in multiple geographic regions.

  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
  │  US-EAST     │──────→│  EU-WEST     │──────→│  APAC        │
  │  Primary     │       │  Replica     │       │  Replica     │
  │  (writes)    │       │  (reads)     │       │  (reads)     │
  └──────────────┘       └──────────────┘       └──────────────┘
         ↑                      ↑                      ↑
    US Users              EU Users               APAC Users
    <10ms latency         <10ms latency          <10ms latency

APPROACHES:
  1. Single Primary + Read Replicas: Writes go to one region
  2. Multi-Primary: Each region can write (CockroachDB, Spanner)
  3. Active-Active: Full read/write in each region (conflict resolution needed)

CHALLENGES:
  • Cross-region latency (100-300ms per replica write)
  • Data sovereignty (GDPR: EU data stays in EU)
  • Conflict resolution in multi-primary setups
  • Cost (egress charges between regions)
```

---

### Q69: What is the Outbox Pattern?

```
PROBLEM: Update database AND publish event — atomically.

  ❌ WRONG:
  BEGIN;
  INSERT INTO orders (...);          -- DB operation
  COMMIT;
  publish_to_kafka('order_created'); -- This can FAIL! 💀

  ❌ Also WRONG:
  publish_to_kafka('order_created'); -- What if DB insert fails?
  BEGIN;
  INSERT INTO orders (...);
  COMMIT;

OUTBOX PATTERN ✅:
  BEGIN;
  INSERT INTO orders (...);
  INSERT INTO outbox (event_type, payload, created_at, processed);
  COMMIT;  -- BOTH in same transaction → ACID guarantees!

  -- Separate process reads outbox → publishes to Kafka
  -- After successful publish → mark as processed
  
  Outbox table:
  ┌──────┬──────────────┬────────────────────────┬───────────┐
  │ id   │ event_type   │ payload                │ processed │
  ├──────┼──────────────┼────────────────────────┼───────────┤
  │ 1    │ order_created│ {"order_id": 123, ...} │ true      │
  │ 2    │ order_shipped│ {"order_id": 124, ...} │ false     │
  └──────┴──────────────┴────────────────────────┴───────────┘

  Tools: Debezium (reads WAL instead of polling outbox → even better!)
```

---

### Q70: What is a Database Proxy vs Load Balancer?

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ Load Balancer        │ Database Proxy       │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Protocol aware   │ ❌ TCP/HTTP level    │ ✅ SQL protocol      │
│ Query parsing    │ ❌ No               │ ✅ Yes               │
│ Read/write split │ ❌ No               │ ✅ Yes               │
│ Connection pool  │ ❌ No               │ ✅ Yes               │
│ Query caching    │ ❌ No               │ ✅ Yes (some)        │
│ Failover         │ ✅ Health checks    │ ✅ + query rerouting │
│ Examples         │ HAProxy, ELB, NLB   │ PgBouncer, ProxySQL  │
│ Best for         │ General TCP traffic │ Database-specific    │
└──────────────────┴──────────────────────┴──────────────────────┘
```

---

# 🏗️ SECTION 6: Database Comparisons & Architecture (Q71–Q80)

---

### Q71: SQL vs NoSQL — When to use which?

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│ Factor               │ Choose SQL           │ Choose NoSQL         │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Data structure       │ Structured, relations│ Semi/unstructured    │
│ Schema               │ Fixed, planned       │ Flexible, evolving   │
│ Consistency          │ ACID required        │ Eventual is OK       │
│ Query complexity     │ Complex JOINs needed │ Simple lookups       │
│ Scale                │ Vertical (mostly)    │ Horizontal           │
│ Transactions         │ Multi-table ACID     │ Single-doc/eventual  │
│ Team expertise       │ SQL knowledge        │ Document/KV patterns │
│ Data volume          │ TB scale             │ PB scale             │
│ Examples             │ Banking, ERP, CRM    │ Social media, IoT    │
│                      │ Healthcare, Finance  │ Gaming, Real-time    │
└──────────────────────┴──────────────────────┴──────────────────────┘

🎯 MODERN REALITY:
  • PostgreSQL has JSONB (document features in SQL!)
  • MongoDB has multi-document ACID transactions
  • The line between SQL and NoSQL is BLURRING
  • Many systems use BOTH (polyglot persistence)
```

---

### Q72: Compare document vs key-value vs wide-column vs graph databases.

```
┌──────────────────┬────────────────┬──────────────┬───────────────┬───────────────┐
│ Type             │ Document       │ Key-Value    │ Wide-Column   │ Graph         │
├──────────────────┼────────────────┼──────────────┼───────────────┼───────────────┤
│ Data model       │ JSON/BSON docs │ Key → Value  │ Row key →     │ Nodes +       │
│                  │                │              │ Column families│ Edges         │
│ Query            │ Rich (MQL,     │ GET/SET only │ CQL (limited) │ Cypher, SPARQL│
│ flexibility      │ aggregation)   │ (by key)     │               │ (traversal)   │
│ Schema           │ Flexible       │ None         │ Semi-flexible │ Property-based│
│ Scaling          │ Horizontal     │ Horizontal   │ Horizontal    │ Vertical*     │
│ Best for         │ General purpose│ Caching,     │ IoT, time-    │ Social nets,  │
│                  │ content mgmt   │ sessions     │ series, logs  │ fraud, knowl. │
│ Example          │ MongoDB,       │ Redis,       │ Cassandra,    │ Neo4j,        │
│                  │ CouchDB        │ DynamoDB     │ HBase, ScyllaDB│ Amazon Neptune│
│ When to avoid    │ Heavy JOINs    │ Complex      │ Complex       │ Simple CRUD   │
│                  │                │ queries      │ transactions  │               │
└──────────────────┴────────────────┴──────────────┴───────────────┴───────────────┘
```

---

### Q73: What is Polyglot Persistence?

```
POLYGLOT PERSISTENCE: Use DIFFERENT databases for DIFFERENT parts 
of the same application, based on each use case's needs.

E-COMMERCE EXAMPLE:
┌─────────────────────────────────────────────────────────────┐
│                      E-Commerce App                         │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ User Profiles│  │   Product    │  │   Orders &   │     │
│  │   MongoDB    │  │   Catalog    │  │   Payments   │     │
│  │ (flexible    │  │ Elasticsearch│  │  PostgreSQL  │     │
│  │  schema)     │  │ (search)     │  │  (ACID)      │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Sessions &  │  │  Analytics   │  │ Social Graph │     │
│  │    Cache     │  │ ClickHouse   │  │    Neo4j     │     │
│  │    Redis     │  │ (columnar)   │  │  (friends,   │     │
│  │  (fast!)     │  │              │  │   recomm.)   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘

✅ Right tool for each job
❌ Operational complexity (manage 6 databases!)
❌ Data consistency across databases
❌ Team must know multiple technologies
```

---

### Q74: What is the difference between Embedded and Client-Server databases?

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ Embedded             │ Client-Server        │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Architecture     │ Runs in app process  │ Separate server      │
│ Network          │ No network needed    │ Network required     │
│ Concurrency      │ Usually single-user  │ Multi-user           │
│ Administration   │ Zero config          │ DBA needed           │
│ Scaling          │ Limited              │ Scalable             │
│ Size             │ Small (MB-few GB)    │ Large (GB-PB)        │
│ Examples         │ SQLite, LevelDB,     │ PostgreSQL, MySQL,   │
│                  │ Berkeley DB, H2      │ Oracle, MongoDB      │
│ Use cases        │ Mobile apps, desktop,│ Web apps, enterprise,│
│                  │ IoT, testing         │ microservices        │
└──────────────────┴──────────────────────┴──────────────────────┘
```

---

### Q75: When would you choose PostgreSQL over MySQL (and vice versa)?

```
CHOOSE POSTGRESQL WHEN:
  ✅ Complex queries with many JOINs
  ✅ Need JSONB (document-like features in SQL)
  ✅ Geospatial queries (PostGIS is incredible)
  ✅ Full-text search (built-in, good enough for many cases)
  ✅ Advanced data types (arrays, hstore, ranges, inet)
  ✅ Need custom types / extensions ecosystem
  ✅ Standards compliance matters
  ✅ Write correctness critical (stricter by default)

CHOOSE MYSQL WHEN:
  ✅ Simple read-heavy web application
  ✅ Team already knows MySQL
  ✅ Need very fast simple queries
  ✅ WordPress / PHP / Laravel ecosystem
  ✅ MySQL-specific features you rely on
  ✅ Simpler replication setup
  ✅ Wide hosting support (cheaper)
  ✅ Need managed service (AWS RDS MySQL, PlanetScale)

BOTH ARE EXCELLENT — the "best" depends on YOUR use case, not blogs.
```

---

### Q76: What is a Time-Series Database? When to use one?

```
TIME-SERIES DB: Optimized for timestamped data points.

Regular DB:                          Time-Series DB:
  Store any data shape               Optimized for:
  General-purpose                    ┌─────────────────────────┐
  Not optimized for time             │ timestamp | metric | val│
                                     │ 10:00:01  | cpu    | 72 │
                                     │ 10:00:02  | cpu    | 75 │
                                     │ 10:00:03  | cpu    | 73 │
                                     └─────────────────────────┘

OPTIMIZATIONS:
  • Columnar compression (90%+ compression ratio)
  • Auto-partitioning by time
  • Downsampling (aggregate old data: 1s → 1m → 1h)
  • Built-in retention policies (auto-delete old data)
  • Time-based aggregation functions

EXAMPLES: InfluxDB, TimescaleDB (PostgreSQL extension), 
          QuestDB, Prometheus, ClickHouse

USE CASES:
  ✅ Server monitoring (CPU, memory, disk)
  ✅ IoT sensor data (temperature, humidity)
  ✅ Financial market data (stock prices)
  ✅ Application metrics (response times, error rates)
  ✅ Log analytics
```

---

### Q77: What is a Vector Database? Why is it important now?

```
VECTOR DATABASE: Stores and searches HIGH-DIMENSIONAL vectors (embeddings).

Traditional DB:                     Vector DB:
  Search by exact values            Search by SIMILARITY
  WHERE name = 'Alice'              "Find items SIMILAR to this image"
  Exact match                       Approximate nearest neighbor (ANN)

HOW IT WORKS:
  1. ML Model converts data → vector embedding
     "cute fluffy cat" → [0.2, 0.8, 0.1, 0.9, ...]  (768 dimensions)
  
  2. Vector DB indexes these embeddings
  
  3. Query: "adorable kitten" → [0.3, 0.7, 0.2, 0.8, ...]
     → Find closest vectors → semantically similar results!

ALGORITHMS: HNSW, IVF, Product Quantization
EXAMPLES: Pinecone, Milvus, Weaviate, Qdrant, pgvector (PostgreSQL)
USE CASES: RAG for LLMs, image search, recommendation engines,
           semantic search, anomaly detection
```

---

### Q78: What is the difference between a Data Warehouse and a Database?

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ Database (OLTP)      │ Data Warehouse (OLAP)│
├──────────────────┼──────────────────────┼──────────────────────┤
│ Purpose          │ Run the business     │ Analyze the business │
│ Data source      │ Application writes   │ Multiple sources     │
│ Data freshness   │ Real-time            │ Periodic loads       │
│ Schema           │ 3NF (normalized)     │ Star/Snowflake       │
│ Query type       │ Short, simple        │ Long, complex        │
│ Optimization     │ Write performance    │ Read/scan performance│
│ History          │ Current state        │ Historical trends    │
│ Users            │ Applications         │ Analysts, BI tools   │
│ Size             │ GB to low TB         │ TB to PB             │
│ Storage          │ Row-oriented         │ Column-oriented      │
│ Examples         │ PostgreSQL, MySQL    │ Snowflake, Redshift  │
└──────────────────┴──────────────────────┴──────────────────────┘
```

---

### Q79: What is Database-per-Service in Microservices?

```
SHARED DATABASE (Monolith):          DATABASE PER SERVICE (Microservices):
┌──────────────────────┐             ┌───────┐  ┌───────┐  ┌───────┐
│ Service A            │             │Svc A  │  │Svc B  │  │Svc C  │
│ Service B  →→→ DB ←←←│             │  ↓    │  │  ↓    │  │  ↓    │
│ Service C            │             │ DB_A  │  │ DB_B  │  │ DB_C  │
└──────────────────────┘             └───────┘  └───────┘  └───────┘

Database per Service:
  ✅ Independent scaling per service
  ✅ Independent schema evolution
  ✅ Technology freedom (SQL + NoSQL mix)
  ✅ Failure isolation
  
  ❌ Cross-service queries are hard (no JOINs!)
  ❌ Distributed transactions needed (Saga pattern)
  ❌ Data duplication
  ❌ Eventual consistency between services
```

---

### Q80: What is NewSQL? How is it different from SQL and NoSQL?

```
┌──────────────────┬──────────────┬──────────────┬──────────────────┐
│ Feature          │ SQL          │ NoSQL        │ NewSQL           │
├──────────────────┼──────────────┼──────────────┼──────────────────┤
│ Schema           │ Fixed        │ Flexible     │ Fixed (SQL)      │
│ Scaling          │ Vertical     │ Horizontal   │ Horizontal       │
│ Consistency      │ ACID         │ BASE         │ ACID             │
│ Query language   │ SQL          │ Various      │ SQL              │
│ Distributed      │ ❌ (mostly)  │ ✅           │ ✅               │
│ Examples         │ PostgreSQL,  │ MongoDB,     │ CockroachDB,     │
│                  │ Oracle       │ Cassandra    │ TiDB, Spanner    │
│ Philosophy       │ Correct but  │ Available    │ Correct AND      │
│                  │ limited scale│ at scale     │ scalable         │
└──────────────────┴──────────────┴──────────────┴──────────────────┘

NewSQL = "SQL guarantees at NoSQL scale"
  The best of both worlds — but relatively new and less battle-tested.
```

---

# 🔥 BONUS: Rapid-Fire "Would You Rather" Questions

These are often asked in system design interviews:

```
Q: SQL or NoSQL for a banking system?
A: SQL — ACID transactions are non-negotiable for financial data.

Q: PostgreSQL or MongoDB for a startup MVP?
A: Either works. PostgreSQL if relational, MongoDB if schema is truly unknown.
   PostgreSQL with JSONB gives you both.

Q: Cassandra or DynamoDB for IoT data?
A: DynamoDB if on AWS (managed, less ops). Cassandra if multi-cloud.

Q: Redis or Memcached for caching?
A: Redis — richer data structures, persistence, pub/sub. Memcached only for 
   simple key-value with multi-threaded performance.

Q: Elasticsearch or PostgreSQL for search?
A: Elasticsearch for full-text search at scale. PostgreSQL's built-in FTS 
   for simpler needs without adding infrastructure.

Q: Synchronous or Asynchronous replication?
A: Sync for zero data loss (banking). Async for performance (web apps).
   Semi-sync as a middle ground.

Q: Normalize or Denormalize?
A: Normalize for OLTP (data integrity). Denormalize for OLAP (read speed).
   Often: both (normalized source + denormalized views/materialized views).

Q: UUID or Auto-Increment?
A: Auto-increment for single-server. UUID for distributed/microservices.
   UUIDv7 for both benefits (time-sortable + globally unique).
```

---

## 🔑 Key Takeaways

```
┌─────────────────────────────────────────────────────────────────────┐
│             DATABASE THEORY INTERVIEW SURVIVAL KIT                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. ACID vs BASE → Know when to use each                          │
│  2. CAP Theorem → Understand the REAL choice is CP vs AP          │
│  3. Indexing → B-Tree internals, covering indexes, selectivity    │
│  4. MVCC → How concurrent access works without locking            │
│  5. Replication → Sync vs Async, lag handling                     │
│  6. Sharding → Strategies, shard key selection, trade-offs        │
│  7. Execution Plans → Read them, identify red flags               │
│  8. Normalization → Know all forms, when to denormalize           │
│  9. SQL vs NoSQL → It's not "which is better" — it's "when"      │
│  10. Distributed consensus → Raft, quorum, eventual consistency   │
│                                                                     │
│  The #1 differentiator: Understanding TRADE-OFFS, not just        │
│  definitions. Every answer should include "it depends on..."      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Link |
|-------------|------|
| MongoDB Interview Questions | [Chapter 8.3 — MongoDB Interview](./03-MongoDB-Interview.md) |
| SQL Practice Problems | [Chapter 8.4 — 50 Must-Solve Problems](./04-SQL-Practice-Problems.md) |
| Quick Cheat Sheets | [Chapter 8.5 — Cheat Sheets](./05-Cheat-Sheets.md) |

---

> **"A great database engineer doesn't memorize — they understand trade-offs."** 🧠
