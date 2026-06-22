# 3D.2 — CQL: Cassandra Query Language 🟡⭐

> **"CQL looks like SQL, feels like SQL, but think like SQL and Cassandra will punish you. CQL is SQL's disciplined cousin — it only lets you do what's efficient."**

---

## 📌 What You'll Master in This Chapter

- **CQL syntax** — creating keyspaces, tables, and queries
- **Partition Keys vs Clustering Keys** — the most important concept in Cassandra
- **Primary Key design** — how it determines EVERYTHING about performance
- **Data Types** — from simple to collections to User-Defined Types (UDTs)
- **Collections** — Lists, Sets, and Maps (and when to use each)
- **CRUD Operations** — INSERT, SELECT, UPDATE, DELETE in CQL
- **Materialized Views** — auto-maintained denormalized tables
- **Secondary Indexes** — when they work and when they're a disaster
- **Batches** — CQL batches are NOT what you think!
- **TTL and Timestamps** — auto-expiring data
- **Lightweight Transactions (LWT)** — Cassandra's version of compare-and-swap

---

## 🛠️ Getting Started with CQL

### Connecting to Cassandra

```bash
# Using cqlsh (Cassandra's interactive shell)
$ cqlsh                          # Connect to localhost
$ cqlsh 192.168.1.100            # Connect to specific node
$ cqlsh 192.168.1.100 9042       # Specify port
$ cqlsh --username admin --password secret  # With auth

# Inside cqlsh:
cqlsh> DESCRIBE CLUSTER;         -- Show cluster name & partitioner
cqlsh> DESCRIBE KEYSPACES;       -- List all keyspaces
cqlsh> DESCRIBE TABLES;          -- List tables in current keyspace
cqlsh> CONSISTENCY;              -- Show current consistency level
cqlsh> CONSISTENCY QUORUM;       -- Set consistency level
cqlsh> TRACING ON;               -- Enable query tracing (debug)
cqlsh> EXPAND ON;                -- Vertical output format
```

---

## 🔥 1. Keyspaces — Cassandra's "Database"

A **keyspace** is the top-level container — equivalent to a "database" in SQL world. It defines the **replication strategy**.

```sql
-- ✅ Production keyspace (always use NetworkTopologyStrategy!)
CREATE KEYSPACE IF NOT EXISTS ecommerce
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'us-east': 3,
    'eu-west': 3
}
AND durable_writes = true;    -- true by default (use commit log)

-- 🧪 Development keyspace (SimpleStrategy OK for local dev)
CREATE KEYSPACE IF NOT EXISTS dev_app
WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 1
};

-- Switch to keyspace
USE ecommerce;

-- Alter replication (e.g., add a new data center)
ALTER KEYSPACE ecommerce
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'us-east': 3,
    'eu-west': 3,
    'ap-south': 3    -- Added new DC!
};

-- Drop keyspace (⚠️ DELETES ALL DATA!)
DROP KEYSPACE IF EXISTS dev_app;
```

> ⭐ **Rule:** NEVER use `SimpleStrategy` in production. It doesn't understand data centers or racks, so replicas might all end up on the same rack — defeating the purpose of replication.

---

## 🔥 2. The Primary Key — THE Most Important Concept

The **Primary Key** in Cassandra is not just a unique identifier — it's the **entire foundation** of how data is stored, distributed, and queried. Get this wrong, and everything falls apart.

### Primary Key Components

```
PRIMARY KEY (partition_key, clustering_col1, clustering_col2)
              ──────┬─────  ──────────────┬──────────────────
                    │                      │
                    ▼                      ▼
         Determines WHICH NODE      Determines SORT ORDER
         stores this data           within the partition
         (distribution)             (organization)
```

### The Four Types of Primary Keys

```sql
-- Type 1: Simple Primary Key (single partition key, no clustering)
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name TEXT,
    email TEXT
);
-- Each user_id → ONE partition → ONE row per partition
-- Good for: Key-value lookups


-- Type 2: Composite Partition Key (multiple columns = partition key)
CREATE TABLE sensor_data (
    sensor_id TEXT,
    date TEXT,
    reading_time TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY ((sensor_id, date), reading_time)
);
--              ^^^^^^^^^^^^^^^^   ^^^^^^^^^^^^
--              Partition Key       Clustering Key
--              (both columns!)
-- sensor_id + date TOGETHER determine the partition
-- Good for: Bucketing data (e.g., one partition per sensor per day)


-- Type 3: Compound Primary Key (partition key + clustering keys)
CREATE TABLE messages (
    chat_id UUID,
    sent_at TIMESTAMP,
    sender TEXT,
    body TEXT,
    PRIMARY KEY (chat_id, sent_at)
);
--              ^^^^^^^  ^^^^^^^
--              Partition Clustering
-- All messages for a chat → same partition (same node)
-- Sorted by sent_at within the partition
-- Good for: Time-ordered data within a group


-- Type 4: Multiple Clustering Keys
CREATE TABLE posts_by_user (
    user_id UUID,
    year INT,
    created_at TIMESTAMP,
    post_id UUID,
    title TEXT,
    PRIMARY KEY (user_id, year, created_at)
) WITH CLUSTERING ORDER BY (year DESC, created_at DESC);
-- Partition: user_id
-- Within partition: sorted by year DESC, then created_at DESC
-- Latest posts first!
```

### Visual: How Primary Key Affects Storage

```
┌─────────────────────────────────────────────────────────────────┐
│  TABLE: messages                                                 │
│  PRIMARY KEY (chat_id, sent_at)                                 │
│                                                                   │
│  PARTITION 1 (chat_id = 'abc'):     Stored on Node A            │
│  ┌────────────────────────────────────────────────────┐         │
│  │ sent_at              │ sender  │ body               │         │
│  ├──────────────────────┼─────────┼────────────────────┤         │
│  │ 2024-01-15 10:00:00  │ Alice   │ Hey!               │  ← Row │
│  │ 2024-01-15 10:01:00  │ Bob     │ Hi Alice!          │  ← Row │
│  │ 2024-01-15 10:02:00  │ Alice   │ How are you?       │  ← Row │
│  └────────────────────────────────────────────────────┘         │
│  ↑ All rows with same chat_id are in the SAME partition         │
│  ↑ Sorted by sent_at (clustering key)                           │
│                                                                   │
│  PARTITION 2 (chat_id = 'xyz'):     Stored on Node C            │
│  ┌────────────────────────────────────────────────────┐         │
│  │ sent_at              │ sender  │ body               │         │
│  ├──────────────────────┼─────────┼────────────────────┤         │
│  │ 2024-01-15 09:00:00  │ Charlie │ Meeting at 3?      │         │
│  │ 2024-01-15 09:05:00  │ Dana    │ Sure!              │         │
│  └────────────────────────────────────────────────────┘         │
│                                                                   │
│  💡 Partition = unit of distribution AND unit of read efficiency │
│  Reading one partition = reading from ONE node = FAST            │
│  Reading across partitions = reading from MANY nodes = SLOW     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

> ⭐ **The Golden Rule of CQL:** You can **only query efficiently by partition key**. Cassandra is designed so that every query hits **one partition** (one node). If your query requires scanning multiple partitions, it's a **bad query** (and Cassandra will let you know by requiring `ALLOW FILTERING`).

---

## 🔥 3. Data Types — Complete Reference

### Basic Data Types

| Type | Description | Example | Size |
|------|-------------|---------|------|
| `TEXT` / `VARCHAR` | UTF-8 string | `'Hello World'` | Variable |
| `INT` | 32-bit signed integer | `42` | 4 bytes |
| `BIGINT` | 64-bit signed integer | `9223372036854775807` | 8 bytes |
| `SMALLINT` | 16-bit signed integer | `32767` | 2 bytes |
| `TINYINT` | 8-bit signed integer | `127` | 1 byte |
| `FLOAT` | 32-bit IEEE-754 floating point | `3.14` | 4 bytes |
| `DOUBLE` | 64-bit IEEE-754 floating point | `3.14159265358979` | 8 bytes |
| `DECIMAL` | Variable-precision decimal | `99.99` | Variable |
| `VARINT` | Arbitrary-precision integer | `12345678901234567890` | Variable |
| `BOOLEAN` | True or false | `true` | 1 byte |
| `UUID` | Type 4 UUID (random) | `uuid()` | 16 bytes |
| `TIMEUUID` | Type 1 UUID (time-based) | `now()` | 16 bytes |
| `TIMESTAMP` | Date + time (ms since epoch) | `'2024-01-15 10:30:00'` | 8 bytes |
| `DATE` | Date only | `'2024-01-15'` | 4 bytes |
| `TIME` | Time only (nanosecond precision) | `'10:30:00.000'` | 8 bytes |
| `DURATION` | Duration (months, days, nanos) | `1h30m` | Variable |
| `BLOB` | Arbitrary bytes | `0xDEADBEEF` | Variable |
| `INET` | IP address (v4 or v6) | `'192.168.1.1'` | 4/16 bytes |
| `COUNTER` | Distributed counter | Can only increment/decrement | 8 bytes |
| `ASCII` | ASCII string | `'Hello'` | Variable |

### UUID vs TIMEUUID — When to Use Which

```
UUID (Type 4):
  - Random, no ordering
  - Use for: Primary keys where order doesn't matter
  - Generate: uuid()

TIMEUUID (Type 1):
  - Contains embedded timestamp
  - Naturally sortable by time!
  - Use for: Clustering keys where time-ordering matters
  - Generate: now()
  - Extract time: toTimestamp(timeuuid_column)

💡 TIMEUUID is perfect for "give me the latest N items" queries
   because it sorts by creation time automatically.
```

---

## 🔥 4. Collections — Lists, Sets, and Maps

CQL supports three collection types for storing multiple values in a single column.

```sql
-- SET: Unordered unique values (stored sorted internally)
CREATE TABLE user_tags (
    user_id UUID PRIMARY KEY,
    tags SET<TEXT>,          -- {'java', 'python', 'go'}
    emails SET<TEXT>
);

INSERT INTO user_tags (user_id, tags, emails)
VALUES (uuid(), {'java', 'python'}, {'a@b.com', 'c@d.com'});

-- Add to set
UPDATE user_tags SET tags = tags + {'go'} WHERE user_id = ?;
-- Remove from set
UPDATE user_tags SET tags = tags - {'python'} WHERE user_id = ?;


-- LIST: Ordered, allows duplicates
CREATE TABLE user_timeline (
    user_id UUID PRIMARY KEY,
    posts LIST<TEXT>         -- ['post1', 'post2', 'post1']
);

-- Append to list
UPDATE user_timeline SET posts = posts + ['new post'] WHERE user_id = ?;
-- Prepend to list
UPDATE user_timeline SET posts = ['first post'] + posts WHERE user_id = ?;
-- Set by index (⚠️ causes a read-before-write!)
UPDATE user_timeline SET posts[0] = 'updated' WHERE user_id = ?;


-- MAP: Key-value pairs
CREATE TABLE user_preferences (
    user_id UUID PRIMARY KEY,
    preferences MAP<TEXT, TEXT>   -- {'theme': 'dark', 'lang': 'en'}
);

-- Set/update a key
UPDATE user_preferences
SET preferences['theme'] = 'light'
WHERE user_id = ?;

-- Delete a key
DELETE preferences['theme']
FROM user_preferences
WHERE user_id = ?;
```

### Collection Gotchas ⚠️

```
┌─────────────────────────────────────────────────────────────────┐
│  COLLECTION RULES & WARNINGS:                                    │
│                                                                   │
│  1. Collections are meant for SMALL amounts of data              │
│     (ideally < 100 elements, max ~64KB)                          │
│                                                                   │
│  2. Cassandra reads the ENTIRE collection to return ANY part     │
│     → 10,000 element list? ALL 10K read every time!             │
│                                                                   │
│  3. Collections CANNOT be used in WHERE clauses                  │
│     (unless you create a secondary index on them)                │
│                                                                   │
│  4. Frozen collections: stored as a single blob, cannot be       │
│     partially updated — must replace entire collection           │
│     → Required for collections inside collections                │
│     → Required for UDTs in older versions                        │
│                                                                   │
│  5. If you need a "collection" with millions of items:           │
│     USE A CLUSTERING KEY instead!                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```sql
-- FROZEN collections (immutable — must replace entirely)
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    metadata FROZEN<MAP<TEXT, TEXT>>,       -- Stored as single blob
    nested FROZEN<LIST<FROZEN<MAP<TEXT, TEXT>>>>  -- Nested needs FROZEN
);

-- With frozen, you can't do partial updates:
-- ❌ UPDATE events SET metadata['key'] = 'val' WHERE event_id = ?;
-- ✅ UPDATE events SET metadata = {'key':'val', 'key2':'val2'} WHERE ...;
```

---

## 🔥 5. User-Defined Types (UDTs)

UDTs let you create custom composite types — like structs or objects.

```sql
-- Create a UDT
CREATE TYPE address (
    street TEXT,
    city TEXT,
    state TEXT,
    zip TEXT,
    country TEXT
);

-- Use UDT in a table (must be FROZEN in many versions)
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    name TEXT,
    home_address FROZEN<address>,
    work_address FROZEN<address>
);

-- Insert with UDT
INSERT INTO customers (customer_id, name, home_address)
VALUES (
    uuid(),
    'John Doe',
    {street: '123 Main St', city: 'NYC', state: 'NY', zip: '10001', country: 'US'}
);

-- Query UDT fields
SELECT name, home_address.city, home_address.state
FROM customers
WHERE customer_id = ?;
```

> 💡 **Pro Tip:** UDTs are great for grouping related fields, but they have limitations. They must be `FROZEN` (which means you can't update individual fields — you replace the whole UDT). Use them for data that's always read/written together.

---

## 🔥 6. CRUD Operations — The Complete Guide

### CREATE (INSERT)

```sql
-- Basic INSERT (always an UPSERT in Cassandra!)
INSERT INTO users (user_id, name, email, created_at)
VALUES (uuid(), 'Alice', 'alice@example.com', toTimestamp(now()));

-- ⚠️ INSERT = UPSERT in Cassandra!
-- If the primary key already exists, it OVERWRITES (no error, no warning)

-- INSERT with TTL (auto-delete after N seconds)
INSERT INTO sessions (session_id, user_id, token)
VALUES (uuid(), ?, 'abc123')
USING TTL 3600;    -- Expires in 1 hour (3600 seconds)

-- INSERT with explicit TIMESTAMP (microseconds)
INSERT INTO users (user_id, name)
VALUES (?, 'Bob')
USING TIMESTAMP 1705312200000000;  -- Custom write timestamp

-- INSERT IF NOT EXISTS (Lightweight Transaction — uses Paxos!)
INSERT INTO users (user_id, name, email)
VALUES (uuid(), 'Alice', 'alice@example.com')
IF NOT EXISTS;    -- Only inserts if PK doesn't exist
                  -- ⚠️ 4x slower than normal INSERT (Paxos consensus)
```

### READ (SELECT)

```sql
-- Select all (⚠️ NEVER do this in production — full table scan!)
SELECT * FROM users;

-- Select by partition key (THE correct way)
SELECT * FROM users WHERE user_id = 550e8400-e29b-41d4-a716-446655440000;

-- Select with clustering key range
SELECT * FROM messages
WHERE chat_id = ?
AND sent_at > '2024-01-01'
AND sent_at < '2024-02-01';

-- Select with LIMIT
SELECT * FROM messages
WHERE chat_id = ?
ORDER BY sent_at DESC    -- Only works with clustering key!
LIMIT 20;

-- Select specific columns
SELECT name, email FROM users WHERE user_id = ?;

-- Count (⚠️ expensive — scans all partitions!)
SELECT COUNT(*) FROM users;

-- Token range query (for pagination/analytics)
SELECT * FROM users
WHERE TOKEN(user_id) > TOKEN(?)
LIMIT 100;

-- ❌ WHAT YOU CANNOT DO (without ALLOW FILTERING):
-- SELECT * FROM users WHERE name = 'Alice';          -- ❌ name is not partition key
-- SELECT * FROM users WHERE email LIKE '%@gmail%';   -- ❌ no LIKE support
-- SELECT * FROM messages WHERE body = 'hello';       -- ❌ body is not in PK
```

### ALLOW FILTERING — The Dangerous Keyword

```sql
-- ⚠️ ALLOW FILTERING = "I know this is inefficient, do it anyway"
SELECT * FROM users WHERE name = 'Alice' ALLOW FILTERING;

-- What ALLOW FILTERING actually does:
-- 1. Reads EVERY partition in the table
-- 2. Checks each row against your WHERE clause
-- 3. Returns matches
-- This is a FULL TABLE SCAN — O(n) on ENTIRE dataset!

-- 💀 In production with 1 billion rows = death of your cluster
-- ✅ Use case: dev/debugging on small datasets ONLY
```

### UPDATE

```sql
-- Basic UPDATE (also an UPSERT!)
UPDATE users
SET name = 'Alice Smith', email = 'alice.smith@example.com'
WHERE user_id = 550e8400-e29b-41d4-a716-446655440000;

-- ⚠️ UPDATE = UPSERT too!
-- If the row doesn't exist, it CREATES it!
-- (only sets the specified columns + primary key, other columns = null)

-- UPDATE with TTL
UPDATE users USING TTL 86400     -- These columns expire in 24 hours
SET session_token = 'xyz'
WHERE user_id = ?;

-- Conditional UPDATE (LWT)
UPDATE users
SET email = 'new@email.com'
WHERE user_id = ?
IF email = 'old@email.com';     -- Only updates if current value matches
                                 -- Returns [applied] = true/false
```

### DELETE

```sql
-- Delete entire row
DELETE FROM users WHERE user_id = ?;

-- Delete specific columns
DELETE email, phone FROM users WHERE user_id = ?;

-- Delete with clustering key range
DELETE FROM messages
WHERE chat_id = ?
AND sent_at < '2024-01-01';    -- Delete old messages

-- Conditional DELETE (LWT)
DELETE FROM users
WHERE user_id = ?
IF EXISTS;                      -- Only delete if row exists

-- ⚠️ Remember: DELETE creates a TOMBSTONE, not an actual deletion!
-- Actual removal happens during compaction after gc_grace_seconds
```

---

## 🔥 7. COUNTER Columns — Distributed Counting

Counters are special columns designed for distributed counting (likes, views, etc).

```sql
-- Counter tables have STRICT rules:
-- 1. Only counter columns + primary key columns allowed
-- 2. Cannot INSERT — only UPDATE (increment/decrement)
-- 3. Cannot set a specific value
-- 4. Cannot mix counter and non-counter columns

CREATE TABLE page_views (
    page_url TEXT,
    date TEXT,
    views COUNTER,
    unique_visitors COUNTER,
    PRIMARY KEY ((page_url, date))
);

-- Increment
UPDATE page_views SET views = views + 1
WHERE page_url = '/home' AND date = '2024-01-15';

-- Decrement
UPDATE page_views SET views = views - 1
WHERE page_url = '/home' AND date = '2024-01-15';

-- Read
SELECT * FROM page_views
WHERE page_url = '/home' AND date = '2024-01-15';
```

> ⚠️ **Counter Warning:** Counters are **eventually consistent** and may have temporary inaccuracies during network issues. They're great for "approximately correct" counts (page views, likes) but NOT for financial transactions.

---

## 🔥 8. Secondary Indexes — Handle with Care

Secondary indexes let you query by non-primary-key columns. But they come with serious trade-offs.

```sql
-- Create a secondary index
CREATE INDEX ON users (email);

-- Now you can query by email
SELECT * FROM users WHERE email = 'alice@example.com';

-- Create a custom-named index
CREATE INDEX idx_users_country ON users (country);

-- Index on collection elements
CREATE INDEX ON user_tags (tags);          -- Index on SET elements
SELECT * FROM user_tags WHERE tags CONTAINS 'java';

CREATE INDEX ON user_prefs (KEYS(preferences));    -- Index on MAP keys
CREATE INDEX ON user_prefs (VALUES(preferences));  -- Index on MAP values
CREATE INDEX ON user_prefs (ENTRIES(preferences)); -- Index on MAP entries
```

### Why Secondary Indexes Are Dangerous in Production

```
┌─────────────────────────────────────────────────────────────────┐
│          SECONDARY INDEX — THE HIDDEN PROBLEM                    │
│                                                                   │
│  When you query by partition key:                                │
│  ┌──────┐                                                       │
│  │Node A│ ← Query goes to 1 node. FAST! ✅                     │
│  └──────┘                                                       │
│                                                                   │
│  When you query by secondary index:                              │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                 │
│  │Node A│ │Node B│ │Node C│ │Node D│ │Node E│                  │
│  │ scan │ │ scan │ │ scan │ │ scan │ │ scan │                  │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘                 │
│  ↑ Query goes to ALL nodes! Each node checks its local index.  │
│  ↑ Then results are merged. SLOW on large clusters! ❌          │
│                                                                   │
│  WHEN SECONDARY INDEXES WORK:                                    │
│  ✅ Low cardinality columns (country, status, category)         │
│  ✅ Always combined with partition key in WHERE clause          │
│  ✅ Small clusters (< 10 nodes)                                 │
│                                                                   │
│  WHEN TO AVOID:                                                  │
│  ❌ High cardinality columns (email, user_id, timestamps)       │
│  ❌ Frequently updated columns                                  │
│  ❌ Large clusters (query hits EVERY node)                      │
│  ❌ As a replacement for proper data modeling                   │
│                                                                   │
│  BETTER ALTERNATIVE:                                             │
│  → Create a DENORMALIZED TABLE with the query's access pattern  │
│  → Use MATERIALIZED VIEWS (with caveats)                        │
│  → Use SASI or SAI indexes (newer, more efficient)              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### SAI (Storage-Attached Indexes) — The Modern Alternative

```sql
-- SAI indexes (Cassandra 4.0+) — much better than legacy secondary indexes
CREATE CUSTOM INDEX ON users (email)
USING 'StorageAttachedIndex';

CREATE CUSTOM INDEX ON users (age)
USING 'StorageAttachedIndex';

-- SAI supports more query types:
SELECT * FROM users WHERE email = 'alice@example.com';
SELECT * FROM users WHERE age > 25 AND age < 50;   -- Range queries!

-- Still not a replacement for good data modeling, but MUCH better
-- than legacy secondary indexes for ad-hoc queries.
```

---

## 🔥 9. Materialized Views — Auto-Maintained Denormalization

Materialized Views (MVs) let Cassandra automatically maintain a denormalized copy of your data with a different primary key.

```sql
-- Base table
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name TEXT,
    email TEXT,
    country TEXT
);

-- Materialized View: query users by email
CREATE MATERIALIZED VIEW users_by_email AS
    SELECT * FROM users
    WHERE email IS NOT NULL AND user_id IS NOT NULL
    PRIMARY KEY (email, user_id);

-- Materialized View: query users by country
CREATE MATERIALIZED VIEW users_by_country AS
    SELECT * FROM users
    WHERE country IS NOT NULL AND user_id IS NOT NULL
    PRIMARY KEY (country, user_id);

-- Now you can query by email or country directly!
SELECT * FROM users_by_email WHERE email = 'alice@example.com';
SELECT * FROM users_by_country WHERE country = 'US';

-- Cassandra automatically keeps MVs in sync with the base table.
-- INSERT/UPDATE/DELETE on base table → auto-updated in MVs
```

### Materialized View Rules & Limitations

```
┌─────────────────────────────────────────────────────────────────┐
│  MATERIALIZED VIEW RULES:                                        │
│                                                                   │
│  1. MV primary key must include ALL columns of base table's PK  │
│  2. MV can add exactly ONE new column to the partition key      │
│  3. All primary key columns need IS NOT NULL in WHERE clause    │
│  4. Cannot include non-PK base columns in MV's PK              │
│     (except the one new column)                                  │
│  5. No INSERT/UPDATE/DELETE directly on MVs                     │
│                                                                   │
│  ⚠️  MVs are EVENTUALLY CONSISTENT with the base table          │
│  ⚠️  MVs add write latency (every base write → MV write)       │
│  ⚠️  MVs can get out of sync in edge cases (repairs needed)    │
│  ⚠️  Many teams prefer MANUAL denormalization over MVs          │
│      (more control, more predictable behavior)                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

> 💡 **Pro Tip:** Many experienced Cassandra teams **avoid Materialized Views** and instead maintain denormalized tables manually in application code (or using Cassandra batches). MVs can cause operational surprises. Use them cautiously.

---

## 🔥 10. Batches — They're NOT What You Think!

In SQL, batches are for **performance** (bulk operations). In Cassandra, batches are for **atomicity** — and misusing them is a classic mistake.

```sql
-- ✅ CORRECT USE: Atomic writes to SAME partition (logged batch)
BEGIN BATCH
    INSERT INTO messages (chat_id, sent_at, sender, body)
    VALUES ('chat1', toTimestamp(now()), 'Alice', 'Hello!');
    
    UPDATE chat_stats SET message_count = message_count + 1
    WHERE chat_id = 'chat1';
APPLY BATCH;

-- ✅ CORRECT USE: Denormalization — write to multiple tables atomically
BEGIN BATCH
    INSERT INTO users (user_id, name, email)
    VALUES (?, 'Alice', 'alice@example.com');
    
    INSERT INTO users_by_email (email, user_id, name)
    VALUES ('alice@example.com', ?, 'Alice');
APPLY BATCH;
-- Ensures both tables are updated together (no partial writes)

-- ❌ WRONG USE: Batching for "performance" (bulk loading)
-- DON'T DO THIS:
BEGIN BATCH
    INSERT INTO users (user_id, ...) VALUES (uuid(), ...);  -- partition 1
    INSERT INTO users (user_id, ...) VALUES (uuid(), ...);  -- partition 2
    INSERT INTO users (user_id, ...) VALUES (uuid(), ...);  -- partition 3
    -- ... 1000 more inserts to different partitions ...
APPLY BATCH;
-- ❌ This is SLOWER than individual inserts!
-- Why? Coordinator must track all partitions, use Paxos-like protocol,
-- hold everything in memory, and ALL must succeed or ALL fail.
```

### Batch Types

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  LOGGED BATCH (default — BEGIN BATCH ... APPLY BATCH):           │
│  → Atomic: all succeed or none do                                │
│  → Uses a batch log for recovery                                │
│  → Use for: keeping denormalized tables in sync                 │
│  → Small batches only (same partition preferred)                │
│                                                                   │
│  UNLOGGED BATCH (BEGIN UNLOGGED BATCH ... APPLY BATCH):         │
│  → No atomicity guarantee                                       │
│  → Slightly faster (no batch log)                               │
│  → Use for: multiple writes to SAME partition                   │
│  → Never for cross-partition writes (no atomicity!)             │
│                                                                   │
│  COUNTER BATCH (BEGIN COUNTER BATCH ... APPLY BATCH):           │
│  → For counter updates only                                     │
│  → Cannot mix counter and non-counter operations                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 11. TTL (Time To Live) — Auto-Expiring Data

TTL automatically deletes data after a specified number of seconds. Perfect for sessions, caches, temporary data.

```sql
-- Set TTL on INSERT (entire row expires)
INSERT INTO sessions (session_id, user_id, token)
VALUES (uuid(), ?, 'abc123')
USING TTL 86400;    -- Expires in 24 hours

-- Set TTL on UPDATE (specific columns expire)
UPDATE users USING TTL 3600
SET reset_token = 'xyz789'
WHERE user_id = ?;
-- Only reset_token expires — other columns remain!

-- Check remaining TTL
SELECT TTL(reset_token) FROM users WHERE user_id = ?;
-- Returns seconds remaining (or null if no TTL)

-- Check write timestamp
SELECT WRITETIME(name) FROM users WHERE user_id = ?;
-- Returns microsecond timestamp of when the column was written

-- ⚠️ TTL on a row = TTL on each non-PK column individually
-- When ALL columns expire, the row "disappears" (only PK remains briefly)
-- A tombstone is created when TTL expires → same gc_grace_seconds rules apply
```

### TTL Anti-Pattern Warning

```
⚠️  If you set TTL on individual columns (not the whole row),
    the row itself persists as a "ghost row" — primary key columns
    remain, but all other columns are null.
    
    This can cause phantom rows in your results.
    
    💡 Best Practice: Set TTL on ALL non-PK columns together,
    or use TTL at INSERT time (applies to all columns in that insert).
```

---

## 🔥 12. Lightweight Transactions (LWT) — Compare-and-Swap

LWTs use the **Paxos consensus protocol** to provide linearizable consistency — true "if-then" conditional operations.

```sql
-- Insert only if not exists
INSERT INTO users (user_id, email, name)
VALUES (uuid(), 'alice@example.com', 'Alice')
IF NOT EXISTS;

-- Returns: [applied] | user_id | email              | name
--           True     |         |                    |
-- or:       False    | <existing_uuid> | alice@... | Alice

-- Update only if condition is met
UPDATE users SET email = 'newemail@example.com'
WHERE user_id = ?
IF email = 'oldemail@example.com';

-- Delete only if exists
DELETE FROM users WHERE user_id = ?
IF EXISTS;

-- Multiple conditions
UPDATE inventory SET quantity = 5
WHERE product_id = ?
IF quantity > 0 AND reserved = false;
```

### LWT Performance Impact

```
┌─────────────────────────────────────────────────────────────────┐
│                LWT PERFORMANCE IMPACT                            │
│                                                                   │
│  Normal write:     1 round-trip to replicas                     │
│  LWT write:        4 round-trips (Paxos protocol!)             │
│                                                                   │
│  ┌─────────────────────────────────────────────┐                │
│  │ 1. PREPARE → Propose a ballot number         │                │
│  │ 2. PROMISE → Replicas promise to accept      │                │
│  │ 3. PROPOSE → Send the actual value           │                │
│  │ 4. COMMIT  → Confirm the write               │                │
│  └─────────────────────────────────────────────┘                │
│                                                                   │
│  ⚠️ LWTs are ~4x slower than normal writes                     │
│  ⚠️ LWT-heavy workloads can create contention                  │
│  ⚠️ Use sparingly — only when you truly need CAS semantics     │
│                                                                   │
│  Good use cases:                                                 │
│  ✅ User registration (unique email check)                      │
│  ✅ Distributed locks                                           │
│  ✅ Account balance checks                                      │
│  ✅ Inventory reservation                                       │
│                                                                   │
│  Bad use cases:                                                  │
│  ❌ Every write in a high-throughput pipeline                   │
│  ❌ Counter-like operations (use COUNTER type instead)          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 13. Static Columns — Shared Data Within a Partition

Static columns store data that's shared across ALL rows in a partition — stored once, returned with every row.

```sql
CREATE TABLE courses (
    department TEXT,
    course_id UUID,
    course_name TEXT,
    department_head TEXT STATIC,    -- Same for all courses in department
    department_budget DECIMAL STATIC,
    professor TEXT,
    PRIMARY KEY (department, course_id)
);

-- Insert/update static column
INSERT INTO courses (department, department_head, department_budget)
VALUES ('CS', 'Dr. Smith', 500000.00);

-- Insert rows
INSERT INTO courses (department, course_id, course_name, professor)
VALUES ('CS', uuid(), 'Algorithms', 'Dr. Jones');

INSERT INTO courses (department, course_id, course_name, professor)
VALUES ('CS', uuid(), 'Databases', 'Dr. Lee');

-- Query — static columns appear with every row:
-- department | course_id | course_name | department_head | department_budget | professor
-- CS         | uuid1     | Algorithms  | Dr. Smith       | 500000.00         | Dr. Jones
-- CS         | uuid2     | Databases   | Dr. Smith       | 500000.00         | Dr. Lee
--                                        ↑ Same value repeated (stored once!)
```

---

## 🔥 14. Prepared Statements — Always Use Them

```sql
-- In application code (Java driver example):
-- 1. Prepare once
PreparedStatement ps = session.prepare(
    "INSERT INTO users (user_id, name, email) VALUES (?, ?, ?)"
);

-- 2. Bind and execute many times
BoundStatement bs = ps.bind(uuid, "Alice", "alice@example.com");
session.execute(bs);

-- Benefits:
-- ✅ Query parsed and planned ONCE, not every time
-- ✅ Avoids CQL injection (parameterized!)
-- ✅ Reduces network overhead (sends query ID, not full text)
-- ✅ 3-5x faster than raw CQL strings
```

> ⭐ **Always use prepared statements** in production code. Never concatenate user input into CQL strings — it's the same SQL injection risk as in relational databases.

---

## 📋 CQL vs SQL — Quick Comparison

| Feature | SQL (MySQL/PG) | CQL (Cassandra) |
|---------|---------------|-----------------|
| `SELECT *` | ✅ Fine (with limits) | ⚠️ Full cluster scan without PK |
| `WHERE` on any column | ✅ Works (may be slow) | ❌ Must include partition key |
| `JOIN` | ✅ Full support | ❌ Not supported |
| `GROUP BY` | ✅ Full support | ⚠️ Limited (partition-level only) |
| `ORDER BY` | ✅ Any column | ⚠️ Clustering columns only |
| `LIKE` / `ILIKE` | ✅ Supported | ❌ Not supported |
| `INSERT` duplicate PK | ❌ Error / Conflict | ✅ Silently overwrites (UPSERT) |
| `UPDATE` non-existent row | ❌ 0 rows affected | ✅ Creates the row (UPSERT) |
| `NULL` semantics | NULL = unknown | NULL = not set / absent |
| Aggregations | ✅ Full (SUM, AVG, etc.) | ⚠️ Basic (within partition) |
| Subqueries | ✅ Supported | ❌ Not supported |
| Stored Procedures | ✅ Supported | ❌ Not supported |
| Transactions | ✅ Full ACID | ⚠️ LWT only (Paxos) |
| Schema changes | Lock table | Online (rolling, no downtime) |
| `ALTER TABLE ADD` | ✅ Supported | ✅ Supported (instant, no rebuild) |
| `ALTER TABLE DROP` | ✅ Supported | ✅ Supported (marks as tombstone) |

---

## 🎯 Chapter Summary — What You Now Know

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  ✅ Keyspace = top-level container (defines replication)         │
│  ✅ Primary Key = Partition Key + Clustering Key(s)              │
│  ✅ Partition Key determines WHERE data lives (which node)      │
│  ✅ Clustering Key determines HOW data is sorted within         │
│  ✅ Collections (SET, LIST, MAP) — keep them small!             │
│  ✅ UDTs for structured nested data (must be FROZEN)            │
│  ✅ INSERT = UPDATE = UPSERT (no errors for duplicates)         │
│  ✅ ALLOW FILTERING = full table scan = production danger       │
│  ✅ Secondary Indexes → use SAI, avoid legacy indexes           │
│  ✅ Materialized Views → auto-maintained but use cautiously     │
│  ✅ Batches are for ATOMICITY, not performance                  │
│  ✅ TTL auto-expires data (creates tombstones)                  │
│  ✅ LWTs use Paxos — 4x slower, use sparingly                  │
│  ✅ Static columns share data across partition rows             │
│  ✅ ALWAYS use prepared statements                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⏭️ What's Next?

| Next Chapter | What You'll Learn |
|-------------|-------------------|
| [3D.3 — Cassandra Data Modeling](./03-Cassandra-Data-Modeling.md) | Query-first design, partition sizing, denormalization patterns, real-world models |
| [3D.4 — Cassandra Operations & Tuning](./04-Cassandra-Operations.md) | nodetool, repair, monitoring, compaction tuning, production best practices |

---

> **"In CQL, you don't query the data you have — you model the data for the queries you need."** 🎯
