# 🪶 Chapter 2F.1 — SQLite: The Embedded Powerhouse

> **Level:** 🟢 Beginner Friendly | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 2.1 (SQL Basics)

> *"SQLite is not competing with Oracle. It's competing with `fopen()`."*
> — D. Richard Hipp (Creator of SQLite)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why SQLite is the most deployed database engine on the planet**
- Know exactly **when to use SQLite** and **when NOT to**
- Master SQLite's unique architecture — **zero-config, serverless, self-contained**
- Work with WAL mode, PRAGMA statements, and SQLite internals
- Build real projects using SQLite with Python, Node.js, C, and mobile platforms
- Handle concurrency, locking, and performance like a pro
- Understand why **every single smartphone on Earth** runs SQLite right now

---

## 🧠 The Big Question

> *"Why is a 'small' database the most used database in the entire world?"*

Right now, **as you read this**, SQLite is running on:

```
📱 Every iPhone ever made              → SQLite
📱 Every Android phone ever made       → SQLite
💻 Every Mac, Windows, Linux machine   → SQLite (via browsers)
🌐 Chrome, Firefox, Safari, Edge       → SQLite (Web SQL, cookies, history)
✈️ Airbus A350 flight software         → SQLite
📺 Every smart TV                      → SQLite
🚗 Most car infotainment systems       → SQLite
📡 Every copy of Windows 10/11         → SQLite
🎮 Many game engines                   → SQLite

Estimated active deployments: ~1 TRILLION+ SQLite databases worldwide
```

That's not a typo. **One trillion.** More than all other databases **combined** — by orders of magnitude.

> 💡 **Mind-Blowing Fact**: There are more SQLite databases on Earth than there are people. Your phone alone has 100+ SQLite databases running right now.

---

## 1. What IS SQLite? — The Unconventional Database

### The Definition That Changes Everything

> **SQLite** is a **self-contained, serverless, zero-configuration, transactional** SQL database engine. It's not a client-server database — it's a **library** that gets linked directly into your application.

### How Traditional Databases Work vs SQLite

```
═══════════════════════════════════════════════════════════════════

  TRADITIONAL DATABASE (MySQL, PostgreSQL, Oracle, SQL Server)
  ──────────────────────────────────────────────────────────────

    ┌──────────┐          ┌──────────────────────┐
    │ Your App │ ───TCP──→│  DATABASE SERVER      │──→ 📁 Data Files
    └──────────┘  Network │  (Separate Process)   │
                  Socket  │  • Listens on port    │
    ┌──────────┐          │  • Auth/Permissions   │
    │ Other App│ ───TCP──→│  • Process management │
    └──────────┘          │  • Memory management  │
                          └──────────────────────┘
    
    Need: Install server → Configure → Create users → Open ports → Connect

═══════════════════════════════════════════════════════════════════

  SQLite (The Rebel)
  ──────────────────

    ┌───────────────────────────────────────────┐
    │             YOUR APPLICATION              │
    │                                           │
    │   ┌───────────────┐                       │
    │   │  SQLite       │                       │
    │   │  Library      │───→ 📄 mydata.db      │
    │   │  (linked in)  │    (single file!)     │
    │   └───────────────┘                       │
    │                                           │
    └───────────────────────────────────────────┘

    Need: Nothing. Just use it. The database IS a file.

═══════════════════════════════════════════════════════════════════
```

### The 5 Pillars of SQLite

```
╔══════════════════════════════════════════════════════════════╗
║                   5 PILLARS OF SQLite                        ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  1️⃣  SERVERLESS                                              ║
║     No separate server process. No installation.             ║
║     No configuration. No admin needed.                       ║
║                                                              ║
║  2️⃣  SELF-CONTAINED                                          ║
║     Single C library (~150KB). Zero external dependencies.   ║
║     Everything is baked in.                                  ║
║                                                              ║
║  3️⃣  ZERO-CONFIGURATION                                      ║
║     No setup, no tuning, no management.                      ║
║     Just open a file and start querying.                     ║
║                                                              ║
║  4️⃣  TRANSACTIONAL (ACID-Compliant)                          ║
║     Full ACID support. Atomic commits.                       ║
║     Survives crashes, power failures, OS crashes.            ║
║                                                              ║
║  5️⃣  SINGLE-FILE DATABASE                                    ║
║     Entire database = one cross-platform file.               ║
║     Copy it, email it, put it on USB. It just works.         ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 2. SQLite Architecture — How It Works Inside

### The Overall Architecture

Unlike client-server databases with massive architectures, SQLite is elegantly simple:

```
┌──────────────────────────────────────────────────────────────┐
│                    SQLite ARCHITECTURE                        │
│                                                              │
│  ┌─────────────────────────────────────────────┐             │
│  │              SQL INTERFACE                   │  ← You     │
│  │    (SQL statements go in, results come out)  │    talk     │
│  └───────────────────┬─────────────────────────┘    here     │
│                      │                                       │
│  ┌───────────────────▼─────────────────────────┐             │
│  │            SQL COMPILER                      │             │
│  │  ┌──────────┐ ┌───────────┐ ┌────────────┐  │             │
│  │  │ Tokenizer│→│  Parser   │→│Code        │  │             │
│  │  │          │ │           │ │Generator   │  │             │
│  │  └──────────┘ └───────────┘ └────────────┘  │             │
│  └───────────────────┬─────────────────────────┘             │
│                      │  Bytecode (VDBE program)              │
│  ┌───────────────────▼─────────────────────────┐             │
│  │     VIRTUAL DATABASE ENGINE (VDBE)           │             │
│  │     (Executes the bytecode program)          │             │
│  └───────────────────┬─────────────────────────┘             │
│                      │                                       │
│  ┌───────────────────▼─────────────────────────┐             │
│  │            B-TREE MODULE                     │             │
│  │  (Manages data storage in B-tree structures) │             │
│  │  • Table B-trees (data)                      │             │
│  │  • Index B-trees (indexes)                   │             │
│  └───────────────────┬─────────────────────────┘             │
│                      │                                       │
│  ┌───────────────────▼─────────────────────────┐             │
│  │             PAGER MODULE                     │             │
│  │  (Page cache + Transaction management)       │             │
│  │  • Reads/writes fixed-size pages             │             │
│  │  • Manages journal/WAL for crash recovery    │             │
│  │  • Handles locking                           │             │
│  └───────────────────┬─────────────────────────┘             │
│                      │                                       │
│  ┌───────────────────▼─────────────────────────┐             │
│  │        OS INTERFACE (VFS Layer)              │             │
│  │  (Abstracts OS-specific file I/O & locking)  │             │
│  └───────────────────┬─────────────────────────┘             │
│                      │                                       │
│                  📄 database.db (single file on disk)        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Components Explained

| Component | What It Does | Analogy |
|-----------|-------------|---------|
| **SQL Interface** | Accepts SQL strings from your app | The front door |
| **Tokenizer** | Breaks SQL into tokens (`SELECT`, `FROM`, `WHERE`...) | Breaking a sentence into words |
| **Parser** | Builds a parse tree, validates syntax | Understanding the grammar |
| **Code Generator** | Converts parse tree to VDBE bytecode | Compiling code |
| **VDBE** | Executes bytecode instructions step-by-step | The CPU of SQLite |
| **B-Tree** | Stores data in balanced tree structures | The filing cabinet |
| **Pager** | Manages fixed-size pages, caching, crash recovery | The warehouse manager |
| **VFS** | Abstracts OS file operations (portability layer) | The building foundation |

### The VDBE — SQLite's Secret Weapon

SQLite compiles every SQL statement into **bytecode** and runs it on a virtual machine (the VDBE). You can actually SEE this bytecode:

```sql
-- Show the bytecode for a query
EXPLAIN SELECT name, age FROM users WHERE age > 25;
```

```
addr  opcode         p1    p2    p3    p4             p5
----  -------------  ----  ----  ----  -------------  --
0     Init           0     12    0                    0
1     OpenRead       0     2     0     3              0
2     Rewind         0     10    0                    0
3     Column         0     2     0                    0
4     Le             1     9     0     (BINARY)       0
5     Column         0     1     0                    0
6     Column         0     2     0                    0
7     ResultRow      0     2     0                    0
8     Next           0     3     0                    0
9     Goto           0     11    0                    0
10    Close          0     0     0                    0
11    Halt           0     0     0                    0
12    Integer        25    1     0                    0
13    Transaction    0     0     0                    0
14    Goto           0     1     0                    0
```

> 💡 **Pro Tip**: `EXPLAIN` (bytecode) and `EXPLAIN QUERY PLAN` (execution strategy) are your two best debugging tools in SQLite.

---

## 3. The SQLite File Format — One File to Rule Them All

### What's Inside a `.db` File?

```
┌──────────────────────────────────────────────────────────────┐
│                   myapp.db (SQLite Database File)            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Page 1 (Header + Schema Table)                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Magic String: "SQLite format 3\000"                    │  │
│  │ Page Size: 4096 bytes (default)                        │  │
│  │ File Format Versions                                   │  │
│  │ Database Size (in pages)                               │  │
│  │ Free Page List                                         │  │
│  │ Schema Cookie                                          │  │
│  │ sqlite_master table (stores all CREATE statements)     │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Page 2..N (B-tree pages — your actual data)                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Table B-trees: Store actual row data                   │  │
│  │ Index B-trees: Store index entries                     │  │
│  │ Overflow pages: For large BLOBs/text                   │  │
│  │ Free pages: Reclaimed space                            │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Facts About the File Format

```
╔══════════════════════════════════════════════════════════════╗
║  SQLITE FILE FORMAT FACTS                                    ║
╠══════════════════════════════════════════════════════════════╣
║  • Default page size: 4096 bytes (configurable: 512-65536)  ║
║  • Max database size: 281 terabytes (2^47 pages × 65536)    ║
║  • Max row size: ~1 billion bytes                            ║
║  • Max columns per table: 2000 (default limit: 32767)       ║
║  • File is cross-platform (copy from Windows → Linux → Mac) ║
║  • Backward compatible since 2004 (version 3.0.0)           ║
║  • File format is STABLE — guaranteed until 2050            ║
║  • Used as US Library of Congress archival format            ║
╚══════════════════════════════════════════════════════════════╝
```

> 💡 **Why This Matters**: The US Library of Congress chose SQLite as a recommended storage format for datasets. That's how stable and trustworthy this format is.

---

## 4. When to Use SQLite — The Decision Matrix

This is **the most important section**. Using SQLite in the wrong scenario is a disaster. Using it in the right scenario is pure genius.

### ✅ PERFECT Use Cases for SQLite

```
╔══════════════════════════════════════════════════════════════════╗
║  USE SQLite WHEN...                                              ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  📱 MOBILE APPS (iOS / Android)                                  ║
║     → Every major app uses SQLite internally                     ║
║     → WhatsApp, Instagram, Uber — all use SQLite on device       ║
║                                                                  ║
║  🖥️ DESKTOP APPLICATIONS                                        ║
║     → Application data, user preferences, local cache            ║
║     → VS Code, Photoshop, Skype — all use SQLite                ║
║                                                                  ║
║  🧪 TESTING & PROTOTYPING                                       ║
║     → Instant database for unit tests (no server needed)         ║
║     → Prototype your schema before moving to PostgreSQL          ║
║                                                                  ║
║  📊 DATA ANALYSIS & SCIENCE                                     ║
║     → Load CSVs into SQLite, run SQL queries                     ║
║     → Better than pandas for large datasets that fit on disk     ║
║                                                                  ║
║  🌐 SMALL-TO-MEDIUM WEBSITES                                    ║
║     → Sites with <100K visitors/day                              ║
║     → Blogs, documentation sites, internal tools                 ║
║     → sqlite.org itself runs on SQLite!                          ║
║                                                                  ║
║  📦 EMBEDDED / IoT DEVICES                                      ║
║     → Routers, TVs, cameras, wearables                           ║
║     → Tiny footprint, no admin needed                            ║
║                                                                  ║
║  📁 APPLICATION FILE FORMAT                                      ║
║     → Instead of XML/JSON config → use SQLite as the file format ║
║     → Example: Adobe Lightroom catalogs are SQLite               ║
║                                                                  ║
║  ✈️ EDGE COMPUTING & OFFLINE-FIRST APPS                          ║
║     → Local database synced with server when online              ║
║     → POS systems, field data collection                         ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### ❌ DO NOT Use SQLite When...

```
╔══════════════════════════════════════════════════════════════════╗
║  AVOID SQLite WHEN...                                            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  🔴 HIGH-WRITE CONCURRENCY                                      ║
║     → Multiple processes/threads writing simultaneously          ║
║     → SQLite allows only ONE writer at a time (even with WAL)   ║
║     → Use PostgreSQL/MySQL instead                               ║
║                                                                  ║
║  🔴 CLIENT-SERVER ARCHITECTURE                                   ║
║     → Multiple apps on different machines accessing same DB      ║
║     → SQLite is NOT a client-server database                     ║
║     → Network file systems + SQLite = corruption risk ⚠️        ║
║                                                                  ║
║  🔴 VERY LARGE DATASETS (Terabytes+)                             ║
║     → While technically possible (281 TB max), not practical     ║
║     → No query parallelism, no partitioning                      ║
║     → Use PostgreSQL, Cassandra, or a data warehouse             ║
║                                                                  ║
║  🔴 HIGH-TRAFFIC WEB APPLICATIONS                                ║
║     → 10,000+ concurrent users writing data                      ║
║     → Write throughput bottleneck                                ║
║                                                                  ║
║  🔴 REPLICATION / HIGH AVAILABILITY                              ║
║     → No built-in replication                                    ║
║     → No failover, no clustering                                 ║
║     → (Though Litestream & LiteFS are changing this — see §12) ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### The Decision Flowchart

```
                    Do you need a database?
                           │
                          YES
                           │
              ┌────────────▼────────────┐
              │ Multiple machines       │
              │ accessing the same DB?  │
              └────────┬────────┬───────┘
                      YES       NO
                       │         │
              Use Client─Server  │
              (PostgreSQL/MySQL)  │
                                 │
              ┌──────────────────▼──────────────┐
              │ High-write concurrency needed?   │
              │ (>100 writes/sec from many       │
              │  threads/processes)              │
              └────────┬─────────┬──────────────┘
                      YES        NO
                       │          │
              Use Client─Server   │
                                  │
              ┌───────────────────▼─────────────┐
              │ Data fits on a single machine?   │
              └────────┬──────────┬─────────────┘
                       NO         YES
                       │           │
              Use Distributed DB   │
                                   │
                          ┌────────▼────────┐
                          │  ✅ USE SQLite!  │
                          └─────────────────┘
```

---

## 5. Getting Started — Zero to Querying in 60 Seconds

### Installation (Spoiler: You Probably Already Have It)

**Check if SQLite is already installed:**

```bash
# Linux/macOS — almost always pre-installed
sqlite3 --version

# Windows — may need to download
# Download from: https://sqlite.org/download.html
# Get the "sqlite-tools" bundle → extract → add to PATH
```

**Language-specific (you may already have it):**

```
Python:    Built-in! → import sqlite3  (ships with Python stdlib)
Node.js:   npm install better-sqlite3  (or npm install sqlite3)
Java:      Add JDBC driver (org.xerial:sqlite-jdbc)
C/C++:     Download sqlite3.c and sqlite3.h (amalgamation)
PHP:       Built-in! → new SQLite3('mydb.db')
Go:        go get github.com/mattn/go-sqlite3
Rust:      cargo add rusqlite
Swift/iOS: Built-in! (Core Data uses SQLite underneath)
Android:   Built-in! (android.database.sqlite)
```

### Your First SQLite Database — 60-Second Quickstart

```bash
# Create (or open) a database
sqlite3 myapp.db

# You're now inside the SQLite shell!
```

```sql
-- Create a table
CREATE TABLE users (
    id    INTEGER PRIMARY KEY AUTOINCREMENT,
    name  TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    age   INTEGER CHECK(age >= 0 AND age <= 150),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Insert data
INSERT INTO users (name, email, age) VALUES ('Ritesh', 'ritesh@example.com', 28);
INSERT INTO users (name, email, age) VALUES ('Priya', 'priya@example.com', 25);
INSERT INTO users (name, email, age) VALUES ('John', 'john@example.com', 32);

-- Query data
SELECT * FROM users;
-- id | name   | email              | age | created_at
-- 1  | Ritesh | ritesh@example.com | 28  | 2026-06-02 10:30:00
-- 2  | Priya  | priya@example.com  | 25  | 2026-06-02 10:30:01
-- 3  | John   | john@example.com   | 32  | 2026-06-02 10:30:02

-- Done! Your database is now saved as "myapp.db" — a single file.
```

> 💡 **That's it.** No server. No configuration. No users/passwords. No ports. Just... a file.

---

## 6. SQLite Data Types — The Flexible Type System

### SQLite's Type Affinity System (Unique!)

Unlike MySQL/PostgreSQL where column types are **strict**, SQLite uses **type affinity** — a flexible type system that will surprise you:

```
╔══════════════════════════════════════════════════════════════════╗
║  SQLite has only 5 STORAGE CLASSES (not "data types"):          ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. NULL      → The value is NULL                                ║
║  2. INTEGER   → Signed integer (1, 2, 3, 4, 6, or 8 bytes)     ║
║  3. REAL      → Floating-point (8-byte IEEE float)              ║
║  4. TEXT      → Text string (UTF-8, UTF-16BE, or UTF-16LE)     ║
║  5. BLOB      → Binary data (stored exactly as input)           ║
║                                                                  ║
║  That's it. No DATE, no BOOLEAN, no VARCHAR(255).               ║
║  Everything maps to one of these 5.                              ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Type Affinity — How SQLite Interprets Your Types

When you write `VARCHAR(100)` or `BOOLEAN`, SQLite maps it:

```
╔═════════════════════════════════╦════════════════════╗
║  You Write (in CREATE TABLE)    ║  SQLite Stores As  ║
╠═════════════════════════════════╬════════════════════╣
║  INT, INTEGER, TINYINT,         ║                    ║
║  SMALLINT, MEDIUMINT, BIGINT,   ║  INTEGER affinity  ║
║  INT2, INT8, BOOLEAN            ║                    ║
╠═════════════════════════════════╬════════════════════╣
║  CHAR(20), VARCHAR(255),        ║                    ║
║  TEXT, CLOB, VARYING CHARACTER  ║  TEXT affinity      ║
╠═════════════════════════════════╬════════════════════╣
║  REAL, DOUBLE, FLOAT,           ║                    ║
║  DOUBLE PRECISION               ║  REAL affinity      ║
╠═════════════════════════════════╬════════════════════╣
║  BLOB                           ║  BLOB (no affinity) ║
║  (or no type specified)         ║                    ║
╠═════════════════════════════════╬════════════════════╣
║  NUMERIC, DECIMAL(10,5),        ║                    ║
║  DATE, DATETIME                 ║  NUMERIC affinity   ║
╚═════════════════════════════════╩════════════════════╝
```

### The Controversial Flexibility

```sql
-- This is VALID in SQLite (but would fail in PostgreSQL/MySQL):
CREATE TABLE crazy (x);              -- No type at all!
INSERT INTO crazy VALUES (42);       -- Integer
INSERT INTO crazy VALUES ('hello');  -- Text in the SAME column
INSERT INTO crazy VALUES (3.14);     -- Real in the SAME column
INSERT INTO crazy VALUES (NULL);     -- Null
INSERT INTO crazy VALUES (X'BEEF');  -- Blob

SELECT x, typeof(x) FROM crazy;
-- 42      | integer
-- hello   | text
-- 3.14    | real
-- NULL    | null
-- ¾ï      | blob
```

> ⚠️ **Warning**: This flexibility is a double-edged sword. It's convenient for rapid prototyping but can lead to data integrity issues if you're not careful. Use `CHECK` constraints and `STRICT` tables (SQLite 3.37+) to enforce types.

### STRICT Tables (SQLite 3.37+, 2021)

```sql
-- Enforce strict typing (like traditional RDBMS)
CREATE TABLE users (
    id    INTEGER PRIMARY KEY,
    name  TEXT NOT NULL,
    age   INTEGER NOT NULL,
    score REAL
) STRICT;

INSERT INTO users VALUES (1, 'Ritesh', 'twenty-eight', 95.5);
-- ERROR: cannot store TEXT value in INTEGER column (age)  ✅ Caught!
```

### Handling Dates & Booleans (No Native Types!)

```sql
-- DATES: Store as TEXT (ISO 8601), INTEGER (Unix epoch), or REAL (Julian day)

-- Approach 1: ISO 8601 string (recommended)
INSERT INTO events (name, event_date) VALUES ('Launch', '2026-06-02 14:30:00');

-- Approach 2: Unix timestamp
INSERT INTO events (name, event_date) VALUES ('Launch', 1748871000);

-- Date functions work beautifully:
SELECT date('now');                           -- 2026-06-02
SELECT datetime('now', 'localtime');          -- 2026-06-02 20:00:00
SELECT strftime('%Y-%m-%d %H:%M', 'now');    -- 2026-06-02 14:30
SELECT date('2026-06-02', '+7 days');         -- 2026-06-09
SELECT julianday('now') - julianday('2026-01-01');  -- Days since Jan 1


-- BOOLEANS: Use INTEGER (0 = false, 1 = true)
CREATE TABLE tasks (
    id      INTEGER PRIMARY KEY,
    title   TEXT NOT NULL,
    is_done INTEGER DEFAULT 0 CHECK(is_done IN (0, 1))
);

INSERT INTO tasks (title, is_done) VALUES ('Learn SQLite', 1);
SELECT * FROM tasks WHERE is_done;  -- Truthy check works!
```

---

## 7. SQLite-Specific SQL Features & Syntax

### AUTOINCREMENT vs INTEGER PRIMARY KEY

```sql
-- Method 1: INTEGER PRIMARY KEY (recommended — faster)
CREATE TABLE users (
    id INTEGER PRIMARY KEY,  -- rowid alias, auto-assigns, can REUSE deleted IDs
    name TEXT
);

-- Method 2: AUTOINCREMENT (prevents ID reuse — slightly slower)
CREATE TABLE orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,  -- IDs NEVER reused (even after DELETE)
    product TEXT
);
```

```
╔══════════════════════════════════════════════════════════════╗
║  INTEGER PRIMARY KEY vs AUTOINCREMENT                        ║
╠═════════════════════════════════╦════════════════════════════╣
║  INTEGER PRIMARY KEY            ║  AUTOINCREMENT             ║
╠═════════════════════════════════╬════════════════════════════╣
║  IDs may be reused after DELETE ║  IDs are NEVER reused      ║
║  Slightly faster                ║  Slightly slower (tracks   ║
║                                 ║  max ID in sqlite_sequence)║
║  Use for: most cases            ║  Use for: audit logs,      ║
║                                 ║  where ID reuse = bad      ║
╚═════════════════════════════════╩════════════════════════════╝
```

### UPSERT (INSERT OR ... )

```sql
-- INSERT OR REPLACE (replaces entire row on conflict)
INSERT OR REPLACE INTO users (id, name, email)
VALUES (1, 'Ritesh Singh', 'ritesh@new.com');

-- INSERT OR IGNORE (silently skip on conflict)
INSERT OR IGNORE INTO users (name, email) VALUES ('Duplicate', 'ritesh@example.com');

-- ON CONFLICT (true UPSERT — SQLite 3.24+)
INSERT INTO users (name, email, age) 
VALUES ('Ritesh', 'ritesh@example.com', 29)
ON CONFLICT(email) DO UPDATE SET 
    name = excluded.name,
    age = excluded.age;
-- If email exists → update name and age
-- If email doesn't exist → insert new row
```

### RETURNING Clause (SQLite 3.35+)

```sql
-- Get back the inserted/updated/deleted rows
INSERT INTO users (name, email, age) 
VALUES ('Neha', 'neha@example.com', 27)
RETURNING id, name, created_at;
-- Returns: 4 | Neha | 2026-06-02 14:30:00

UPDATE users SET age = 30 WHERE name = 'Ritesh'
RETURNING id, name, age;
-- Returns: 1 | Ritesh | 30

DELETE FROM users WHERE age > 50
RETURNING *;
-- Returns all deleted rows
```

### Generated (Computed) Columns (SQLite 3.31+)

```sql
CREATE TABLE products (
    id        INTEGER PRIMARY KEY,
    name      TEXT NOT NULL,
    price     REAL NOT NULL,
    tax_rate  REAL NOT NULL DEFAULT 0.18,
    
    -- Virtual: computed on-the-fly (no storage used)
    total_price REAL GENERATED ALWAYS AS (price * (1 + tax_rate)) VIRTUAL,
    
    -- Stored: computed and saved to disk (can be indexed!)
    search_name TEXT GENERATED ALWAYS AS (lower(name)) STORED
);

INSERT INTO products (name, price) VALUES ('Laptop', 50000);
SELECT name, price, total_price FROM products;
-- Laptop | 50000 | 59000.0
```

### Common Table Expressions (CTE) & Window Functions

```sql
-- CTE
WITH active_users AS (
    SELECT * FROM users WHERE last_login > date('now', '-30 days')
)
SELECT name, email FROM active_users ORDER BY name;

-- Recursive CTE (e.g., generate a series)
WITH RECURSIVE cnt(x) AS (
    SELECT 1
    UNION ALL
    SELECT x + 1 FROM cnt WHERE x < 10
)
SELECT x FROM cnt;
-- Returns: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10

-- Window Functions (SQLite 3.25+)
SELECT 
    name,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dept_rank,
    AVG(salary) OVER (PARTITION BY department) as dept_avg,
    salary - AVG(salary) OVER (PARTITION BY department) as diff_from_avg
FROM employees;
```

### ATTACH — Multi-Database Queries!

```sql
-- Open another database and query across both!
ATTACH DATABASE 'analytics.db' AS analytics;
ATTACH DATABASE 'archive.db' AS archive;

-- Query across databases
SELECT u.name, a.page_views
FROM main.users u
JOIN analytics.visits a ON u.id = a.user_id;

-- You can attach up to 10 databases simultaneously (default limit)
DETACH DATABASE analytics;
```

> 💡 **Pro Tip**: `ATTACH` is incredibly useful for merging datasets, archiving old data, or separating concerns without any server configuration.

---

## 8. PRAGMA Statements — SQLite's Control Panel

PRAGMAs are SQLite's configuration commands. They control everything from performance to safety.

### Essential PRAGMAs — The "Copy-Paste This Into Every Project" List

```sql
-- ═══════════════════════════════════════════════════════════
-- 🔥 THE MUST-HAVE PRAGMA CONFIGURATION
-- Run these right after opening any SQLite connection!
-- ═══════════════════════════════════════════════════════════

-- 1. Enable WAL mode (10x faster for most workloads)
PRAGMA journal_mode = WAL;

-- 2. Normal synchronous (safe enough, much faster)
PRAGMA synchronous = NORMAL;

-- 3. Increase cache size (default is tiny — use more RAM!)
PRAGMA cache_size = -64000;  -- 64MB (negative = KB)

-- 4. Enable foreign keys (OFF by default — yes, really!)
PRAGMA foreign_keys = ON;

-- 5. Set busy timeout (wait instead of failing on lock)
PRAGMA busy_timeout = 5000;  -- Wait up to 5 seconds

-- 6. Enable memory-mapped I/O (faster reads)
PRAGMA mmap_size = 268435456;  -- 256MB

-- 7. Store temp tables in memory
PRAGMA temp_store = MEMORY;
```

> ⚠️ **Critical**: `PRAGMA foreign_keys = ON` must be set **on every connection**. By default, SQLite does NOT enforce foreign keys. This catches 90% of "why isn't my FK working?!" questions.

### PRAGMA Reference Table

```
╔═══════════════════════════════╦════════════════════╦══════════════════════════════╗
║  PRAGMA                       ║  Default           ║  Recommended                 ║
╠═══════════════════════════════╬════════════════════╬══════════════════════════════╣
║  journal_mode                 ║  DELETE            ║  WAL (always!)               ║
║  synchronous                  ║  FULL              ║  NORMAL (with WAL)           ║
║  cache_size                   ║  -2000 (2MB)       ║  -64000 (64MB)               ║
║  foreign_keys                 ║  OFF (!)           ║  ON (always!)                ║
║  busy_timeout                 ║  0 (fail instantly)║  5000 (5 seconds)            ║
║  mmap_size                    ║  0 (disabled)      ║  256MB-1GB                   ║
║  temp_store                   ║  DEFAULT (file)    ║  MEMORY                      ║
║  page_size                    ║  4096              ║  4096 (leave default)        ║
║  auto_vacuum                  ║  NONE              ║  INCREMENTAL (for big DBs)   ║
║  wal_autocheckpoint           ║  1000 (pages)      ║  1000 (leave default)        ║
╚═══════════════════════════════╩════════════════════╩══════════════════════════════╝
```

### Useful Diagnostic PRAGMAs

```sql
-- Check database integrity
PRAGMA integrity_check;       -- Full scan (slow but thorough)
PRAGMA quick_check;           -- Faster, less thorough

-- Database info
PRAGMA database_list;         -- All attached databases
PRAGMA table_info(users);     -- Column info for a table
PRAGMA table_xinfo(users);    -- Extended column info (includes hidden columns)
PRAGMA index_list(users);     -- All indexes on a table
PRAGMA index_info(idx_name);  -- Columns in an index
PRAGMA foreign_key_list(orders);  -- FK relationships

-- Performance stats
PRAGMA page_count;            -- Total pages in database
PRAGMA page_size;             -- Size of each page
PRAGMA freelist_count;        -- Unused pages (fragmentation)
PRAGMA cache_size;            -- Current cache size
PRAGMA compile_options;       -- How SQLite was compiled
```

---

## 9. WAL Mode — The Performance Game-Changer

### Journal Modes: DELETE vs WAL

This is the **single most impactful** performance setting in SQLite.

```
═══════════════════════════════════════════════════════════════════
  DELETE Mode (Default) — "The Old Way"
═══════════════════════════════════════════════════════════════════

  WRITE Transaction:
  1. Copy original page to rollback journal     ← Extra I/O!
  2. Modify page in database file
  3. Delete journal on commit                   ← Extra I/O!

  ┌───────────┐    ┌──────────────────┐
  │ database  │←──→│ rollback journal │
  │   .db     │    │ (temp file)      │
  └───────────┘    └──────────────────┘

  Problem: Writers BLOCK readers. Readers BLOCK writers.
           Only one writer OR multiple readers at a time.

═══════════════════════════════════════════════════════════════════
  WAL Mode — "The Game-Changer"
═══════════════════════════════════════════════════════════════════

  WRITE Transaction:
  1. Append changes to WAL file                 ← Sequential write (fast!)
  2. Original database stays untouched
  3. Checkpoint merges WAL → database (background)

  ┌───────────┐    ┌─────────────────┐
  │ database  │←───│  WAL file       │
  │   .db     │    │  .db-wal        │
  └───────────┘    └─────────────────┘
                   ┌─────────────────┐
                   │  SHM file       │
                   │  .db-shm        │
                   └─────────────────┘

  Magic: Writers DON'T block readers! Readers DON'T block writers!
         One writer + multiple readers — simultaneously! 🎉

═══════════════════════════════════════════════════════════════════
```

### DELETE vs WAL — The Comparison

```
╔══════════════════════════╦══════════════════╦══════════════════╗
║  Feature                 ║  DELETE Mode     ║  WAL Mode        ║
╠══════════════════════════╬══════════════════╬══════════════════╣
║  Concurrent readers      ║  Yes             ║  Yes             ║
║  Read during write       ║  ❌ Blocked      ║  ✅ Works!       ║
║  Write performance       ║  Slower          ║  Much faster     ║
║  Read performance        ║  Good            ║  Good            ║
║  Crash safety            ║  ✅ Safe         ║  ✅ Safe         ║
║  Extra files             ║  .db-journal     ║  .db-wal, .db-shm║
║  Network filesystem      ║  ⚠️ Risky       ║  ❌ Don't!       ║
║  Large transactions      ║  Better          ║  WAL grows large ║
║  Checkpoint overhead     ║  None            ║  Periodic        ║
╚══════════════════════════╩══════════════════╩══════════════════╝
```

### How to Enable WAL Mode

```sql
-- Enable WAL (persists across connections — set it once!)
PRAGMA journal_mode = WAL;
-- Returns: wal

-- Check current mode
PRAGMA journal_mode;
-- Returns: wal

-- Manual checkpoint (usually automatic)
PRAGMA wal_checkpoint(TRUNCATE);
```

### WAL Internals — How It Actually Works

```
  Time →
  ═══════════════════════════════════════════════════════════

  Transaction 1 (WRITE):
  ┌────────────────────────────────────────────────┐
  │  Appends pages to WAL: [Page 5'] [Page 12']   │
  └────────────────────────────────────────────────┘

  Transaction 2 (READ — happens simultaneously!):
  ┌────────────────────────────────────────────────┐
  │  Reads from main database file                 │
  │  For pages NOT in WAL → read from .db          │
  │  For pages IN WAL → read latest version        │
  │  Uses WAL-index (.db-shm) for fast lookup      │
  └────────────────────────────────────────────────┘

  Checkpoint (background):
  ┌────────────────────────────────────────────────┐
  │  Merges WAL pages back into main database      │
  │  Resets WAL file                               │
  │  Default: every 1000 pages                     │
  └────────────────────────────────────────────────┘
```

> 💡 **Pro Tip**: WAL mode is so important that if you're using SQLite and NOT in WAL mode, you're leaving massive performance on the table. Always enable it (unless using network filesystems).

---

## 10. Concurrency & Locking — SQLite's Biggest Limitation (And How to Handle It)

### The Fundamental Rule

```
╔══════════════════════════════════════════════════════════════════╗
║                                                                  ║
║   SQLite allows:                                                 ║
║     ✅ Multiple READERS at the same time                        ║
║     ✅ ONE WRITER at a time (even with WAL mode)                ║
║     ❌ Multiple WRITERS never (this is the core limitation)     ║
║                                                                  ║
║   If another writer is active → your write WAITS (busy_timeout) ║
║   If busy_timeout expires → SQLITE_BUSY error                   ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### SQLite Lock States

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  UNLOCKED ──→ SHARED ──→ RESERVED ──→ PENDING ──→ EXCLUSIVE
  │                 │                                    │
  │            (Reading)                           (Writing)
  │                 │                                    │
  │           Multiple readers OK              Only ONE at a time
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  DELETE mode lock progression:
  ┌───────────┬──────────────────────────────────────────────┐
  │ Lock      │ What it means                                │
  ├───────────┼──────────────────────────────────────────────┤
  │ UNLOCKED  │ No one is using the database                 │
  │ SHARED    │ Reading. Multiple SHARED locks allowed.      │
  │ RESERVED  │ Planning to write. Only ONE allowed.         │
  │           │ Readers can still get SHARED locks.          │
  │ PENDING   │ Waiting for existing readers to finish.      │
  │           │ No NEW readers allowed.                      │
  │ EXCLUSIVE │ Writing. No one else can do anything.        │
  └───────────┴──────────────────────────────────────────────┘

  WAL mode (simpler):
  ┌───────────┬──────────────────────────────────────────────┐
  │ SHARED    │ Reading (unlimited concurrent readers)       │
  │ EXCLUSIVE │ Writing (only one writer, readers NOT blocked)│
  └───────────┴──────────────────────────────────────────────┘
```

### Handling SQLITE_BUSY (The #1 Concurrency Error)

```python
# Python — The WRONG way
import sqlite3

conn = sqlite3.connect('myapp.db')
cursor = conn.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
# ❌ If another process is writing → sqlite3.OperationalError: database is locked

# Python — The RIGHT way
conn = sqlite3.connect('myapp.db')
conn.execute("PRAGMA busy_timeout = 5000")  # Wait up to 5 seconds
conn.execute("PRAGMA journal_mode = WAL")    # Enable WAL

# Now it waits gracefully instead of crashing
cursor = conn.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
conn.commit()
```

### Concurrency Best Practices

```
╔══════════════════════════════════════════════════════════════════╗
║  SQLITE CONCURRENCY BEST PRACTICES                              ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. ALWAYS enable WAL mode                                       ║
║     → Readers never block writers, writers never block readers   ║
║                                                                  ║
║  2. ALWAYS set busy_timeout (e.g., 5000ms)                      ║
║     → Prevents immediate SQLITE_BUSY errors                     ║
║                                                                  ║
║  3. Keep write transactions SHORT                                ║
║     → Don't hold transactions open while waiting for user input ║
║                                                                  ║
║  4. Use BEGIN IMMEDIATE for write transactions                   ║
║     → Acquires write lock immediately (fail fast)                ║
║     → Instead of: BEGIN → read → try to write → BUSY!           ║
║                                                                  ║
║  5. Use a SINGLE connection for writes (connection pooling)      ║
║     → Serialize writes through one connection                    ║
║     → Multiple connections for reads                             ║
║                                                                  ║
║  6. Batch writes in transactions                                 ║
║     → 1000 individual INSERTs: ~25 seconds                      ║
║     → 1000 INSERTs in one transaction: ~0.05 seconds             ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Transaction Types

```sql
-- Default: DEFERRED (read lock first, upgrade to write later)
BEGIN DEFERRED TRANSACTION;
-- Gets SHARED lock → if you try to write → tries to upgrade → might BUSY

-- IMMEDIATE: Get write lock right away (recommended for writes)
BEGIN IMMEDIATE TRANSACTION;
-- Gets RESERVED lock immediately → fails fast if another writer exists

-- EXCLUSIVE: Full exclusive lock (rarely needed with WAL)
BEGIN EXCLUSIVE TRANSACTION;
-- Gets EXCLUSIVE lock → no one else can even read (DELETE mode)
```

---

## 11. Performance Optimization — Making SQLite Fly

### The Performance Baseline

```
╔══════════════════════════════════════════════════════════════════╗
║  SQLITE PERFORMANCE (modern hardware, SSD, WAL mode)            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  INSERT (batched in transaction):  ~100,000 - 500,000 rows/sec  ║
║  INSERT (individual):             ~50 - 100 rows/sec (DON'T!)   ║
║  SELECT (indexed):                ~1,000,000+ rows/sec          ║
║  SELECT (full scan):              ~10,000,000+ rows/sec         ║
║  UPDATE (indexed):                ~100,000+ rows/sec            ║
║                                                                  ║
║  ⚡ These numbers are INSANE for an embedded database.          ║
║  Most apps will NEVER hit SQLite's read performance limit.      ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Rule #1: Batch Your Writes in Transactions

This is the **single biggest performance improvement** you can make:

```sql
-- ❌ TERRIBLE: 10,000 individual inserts (~200 seconds!)
INSERT INTO logs (message) VALUES ('event 1');
INSERT INTO logs (message) VALUES ('event 2');
-- ... 9,998 more ...

-- ✅ AMAZING: Same 10,000 inserts in a transaction (~0.05 seconds!)
BEGIN TRANSACTION;
INSERT INTO logs (message) VALUES ('event 1');
INSERT INTO logs (message) VALUES ('event 2');
-- ... 9,998 more ...
COMMIT;

-- 💡 Why? Each standalone INSERT is its own transaction:
--    open journal → write → fsync → close journal → delete journal
--    × 10,000 = 10,000 fsyncs (each ~10ms on HDD)
--    In a batch: only 1 fsync at COMMIT
```

### Rule #2: Use Indexes Wisely

```sql
-- Create indexes for columns you frequently filter/sort/join on
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Check if your query uses the index
EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = 'ritesh@example.com';
-- SEARCH users USING INDEX idx_users_email (email=?)  ✅ Good!

-- vs
EXPLAIN QUERY PLAN SELECT * FROM users WHERE name LIKE '%rit%';
-- SCAN users  ❌ Full table scan! (LIKE with leading % can't use index)

-- Covering index (includes all needed columns — avoids table lookup)
CREATE INDEX idx_users_covering ON users(email, name, age);
-- Now this query never touches the table, only the index:
EXPLAIN QUERY PLAN SELECT name, age FROM users WHERE email = 'ritesh@example.com';
-- SEARCH users USING COVERING INDEX idx_users_covering (email=?)  ⚡ Fastest!
```

### Rule #3: Analyze & Optimize

```sql
-- Generate statistics for the query planner
ANALYZE;

-- Check index usage
EXPLAIN QUERY PLAN <your_query>;

-- Vacuum to reclaim space and defragment
VACUUM;

-- Incremental vacuum (for large databases — less blocking)
PRAGMA auto_vacuum = INCREMENTAL;
PRAGMA incremental_vacuum(100);  -- Free up to 100 pages

-- Check database size and fragmentation
PRAGMA page_count;        -- Total pages
PRAGMA freelist_count;    -- Unused pages (wasted space)
```

### Performance Cheat Sheet

```
╔═══════════════════════════════════════════════════════════════╗
║  ACTION                              │  IMPACT               ║
╠══════════════════════════════════════╪═══════════════════════╣
║  Enable WAL mode                     │  🔥🔥🔥 (10x writes) ║
║  Batch inserts in transactions       │  🔥🔥🔥🔥 (4000x!)   ║
║  Set PRAGMA synchronous = NORMAL     │  🔥🔥 (2x faster)    ║
║  Increase cache_size to 64MB         │  🔥🔥 (less disk I/O)║
║  Add proper indexes                  │  🔥🔥🔥 (100x+ reads)║
║  Use prepared statements             │  🔥 (avoid re-parsing)║
║  Enable mmap_size                    │  🔥 (faster reads)   ║
║  Use IMMEDIATE transactions          │  🔥 (less contention)║
║  Run ANALYZE periodically            │  🔥 (better plans)   ║
║  Use covering indexes                │  🔥🔥 (skip table)   ║
║  Use temp_store = MEMORY             │  🔥 (faster temp ops)║
╚══════════════════════════════════════╧═══════════════════════╝
```

---

## 12. Using SQLite in Real Applications

### Python (Built-in — No Install Needed!)

```python
import sqlite3
from contextlib import closing

# ── Connection Setup ──────────────────────────────────────────
def get_connection(db_path='myapp.db'):
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row  # Access columns by name
    conn.execute("PRAGMA journal_mode = WAL")
    conn.execute("PRAGMA synchronous = NORMAL")
    conn.execute("PRAGMA cache_size = -64000")
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA busy_timeout = 5000")
    conn.execute("PRAGMA temp_store = MEMORY")
    conn.execute("PRAGMA mmap_size = 268435456")
    return conn

# ── Create Tables ─────────────────────────────────────────────
def init_db():
    with closing(get_connection()) as conn:
        conn.executescript("""
            CREATE TABLE IF NOT EXISTS users (
                id         INTEGER PRIMARY KEY AUTOINCREMENT,
                name       TEXT NOT NULL,
                email      TEXT UNIQUE NOT NULL,
                created_at TEXT DEFAULT (datetime('now'))
            );
            
            CREATE TABLE IF NOT EXISTS posts (
                id         INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id    INTEGER NOT NULL REFERENCES users(id),
                title      TEXT NOT NULL,
                body       TEXT,
                created_at TEXT DEFAULT (datetime('now'))
            );
            
            CREATE INDEX IF NOT EXISTS idx_posts_user ON posts(user_id);
        """)

# ── CRUD Operations ───────────────────────────────────────────
def create_user(name, email):
    with closing(get_connection()) as conn:
        cursor = conn.execute(
            "INSERT INTO users (name, email) VALUES (?, ?) RETURNING id, name",
            (name, email)  # ← Always use parameterized queries! (SQL injection prevention)
        )
        user = cursor.fetchone()
        conn.commit()
        return dict(user)

def get_users():
    with closing(get_connection()) as conn:
        rows = conn.execute("SELECT * FROM users ORDER BY created_at DESC").fetchall()
        return [dict(row) for row in rows]

def search_users(query):
    with closing(get_connection()) as conn:
        # ✅ SAFE: parameterized query
        rows = conn.execute(
            "SELECT * FROM users WHERE name LIKE ?",
            (f'%{query}%',)
        ).fetchall()
        return [dict(row) for row in rows]

# ── Batch Insert (The Right Way) ─────────────────────────────
def bulk_insert_users(users_list):
    """Insert thousands of users efficiently."""
    with closing(get_connection()) as conn:
        conn.executemany(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            users_list  # [(name, email), (name, email), ...]
        )
        conn.commit()
        # 10,000 users → ~50ms (vs 200+ seconds without transaction)

# ── Usage ─────────────────────────────────────────────────────
init_db()
user = create_user("Ritesh", "ritesh@example.com")
print(f"Created user: {user}")
# {'id': 1, 'name': 'Ritesh'}
```

> ⚠️ **Security**: ALWAYS use `?` placeholders for parameters. NEVER use f-strings or .format() to build SQL — that's SQL injection waiting to happen.

### Node.js (better-sqlite3 — Synchronous, Fast)

```javascript
// npm install better-sqlite3
const Database = require('better-sqlite3');

// Open database with recommended options
const db = new Database('myapp.db');
db.pragma('journal_mode = WAL');
db.pragma('synchronous = NORMAL');
db.pragma('cache_size = -64000');
db.pragma('foreign_keys = ON');
db.pragma('busy_timeout = 5000');
db.pragma('temp_store = MEMORY');
db.pragma('mmap_size = 268435456');

// Create tables
db.exec(`
    CREATE TABLE IF NOT EXISTS users (
        id    INTEGER PRIMARY KEY AUTOINCREMENT,
        name  TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        created_at TEXT DEFAULT (datetime('now'))
    );
`);

// Prepared statements (faster for repeated queries)
const insertUser = db.prepare(
    'INSERT INTO users (name, email) VALUES (@name, @email) RETURNING *'
);
const getUser = db.prepare('SELECT * FROM users WHERE id = ?');
const getAllUsers = db.prepare('SELECT * FROM users ORDER BY id');
const searchUsers = db.prepare('SELECT * FROM users WHERE name LIKE ?');

// Insert
const user = insertUser.get({ name: 'Ritesh', email: 'ritesh@example.com' });
console.log(user); // { id: 1, name: 'Ritesh', email: 'ritesh@example.com', ... }

// Batch insert using transaction (FAST!)
const insertMany = db.transaction((users) => {
    for (const u of users) insertUser.run(u);
});

insertMany([
    { name: 'Priya', email: 'priya@example.com' },
    { name: 'John',  email: 'john@example.com' },
    { name: 'Neha',  email: 'neha@example.com' },
]);
// All or nothing — if one fails, all roll back!

// Query
const users = getAllUsers.all();
const found = searchUsers.all('%rit%');

// Always close when done
process.on('exit', () => db.close());
```

### Mobile (Android — Kotlin with Room)

```kotlin
// build.gradle: implementation "androidx.room:room-runtime:2.6.1"
// build.gradle: kapt "androidx.room:room-compiler:2.6.1"

// 1. Define Entity (Table)
@Entity(tableName = "users")
data class User(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    @ColumnInfo(name = "name") val name: String,
    @ColumnInfo(name = "email") val email: String,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)

// 2. Define DAO (Data Access Object)
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY created_at DESC")
    fun getAll(): Flow<List<User>>  // Reactive — auto-updates UI!

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)

    @Delete
    suspend fun delete(user: User)

    @Query("SELECT * FROM users WHERE name LIKE '%' || :query || '%'")
    fun search(query: String): Flow<List<User>>
}

// 3. Define Database
@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

// 4. Usage in ViewModel
class UserViewModel(application: Application) : AndroidViewModel(application) {
    private val db = Room.databaseBuilder(
        application, AppDatabase::class.java, "myapp.db"
    ).build()

    val users = db.userDao().getAll()  // LiveData/Flow!
}
```

### Mobile (iOS — Swift with GRDB)

```swift
// SPM: https://github.com/groue/GRDB.swift
import GRDB

// Define your model
struct User: Codable, FetchableRecord, PersistableRecord {
    var id: Int64?
    var name: String
    var email: String
    var createdAt: Date
    
    // Auto-generate table name: "user"
    static let databaseTableName = "users"
}

// Setup database
let dbQueue = try DatabaseQueue(path: dbPath)

try dbQueue.write { db in
    try db.create(table: "users", ifNotExists: true) { t in
        t.autoIncrementedPrimaryKey("id")
        t.column("name", .text).notNull()
        t.column("email", .text).notNull().unique()
        t.column("createdAt", .datetime).notNull()
            .defaults(to: Date())
    }
}

// Insert
try dbQueue.write { db in
    var user = User(id: nil, name: "Ritesh", email: "ritesh@example.com", createdAt: Date())
    try user.insert(db)
    print("Inserted user with ID: \(user.id!)")
}

// Query
let users = try dbQueue.read { db in
    try User.order(Column("createdAt").desc).fetchAll(db)
}
```

---

## 13. Full-Text Search (FTS5) — Google-Like Search in SQLite

### What is FTS5?

FTS5 (Full-Text Search version 5) turns SQLite into a **search engine**. Instead of slow `LIKE '%keyword%'` queries, you get instant, ranked search results.

```sql
-- ── Create an FTS5 Virtual Table ─────────────────────────────
CREATE VIRTUAL TABLE articles_fts USING fts5(
    title,
    body,
    content='articles',        -- Link to real table (optional)
    content_rowid='id'         -- Map to real table's rowid
);

-- ── Populate the FTS index ───────────────────────────────────
INSERT INTO articles_fts(rowid, title, body)
SELECT id, title, body FROM articles;

-- ── Search! ──────────────────────────────────────────────────
-- Simple search
SELECT * FROM articles_fts WHERE articles_fts MATCH 'database';

-- Boolean operators
SELECT * FROM articles_fts WHERE articles_fts MATCH 'sqlite AND performance';
SELECT * FROM articles_fts WHERE articles_fts MATCH 'sqlite OR postgres';
SELECT * FROM articles_fts WHERE articles_fts MATCH 'sqlite NOT mysql';

-- Phrase search (exact phrase)
SELECT * FROM articles_fts WHERE articles_fts MATCH '"full text search"';

-- Column-specific search
SELECT * FROM articles_fts WHERE articles_fts MATCH 'title:sqlite';

-- Prefix search
SELECT * FROM articles_fts WHERE articles_fts MATCH 'data*';  -- database, datastore, etc.

-- NEAR search (words near each other)
SELECT * FROM articles_fts WHERE articles_fts MATCH 'NEAR(sqlite performance, 5)';

-- ── Ranked Results (BM25) ────────────────────────────────────
SELECT 
    rowid,
    title,
    rank  -- BM25 relevance score (lower = more relevant)
FROM articles_fts 
WHERE articles_fts MATCH 'sqlite performance'
ORDER BY rank;  -- Best matches first

-- ── Highlight & Snippet ──────────────────────────────────────
SELECT 
    highlight(articles_fts, 0, '<b>', '</b>') as title,
    snippet(articles_fts, 1, '<b>', '</b>', '...', 32) as body_preview
FROM articles_fts 
WHERE articles_fts MATCH 'sqlite';
-- Returns: <b>SQLite</b> Performance Guide
--          ...dramatically improve <b>SQLite</b> read performance...
```

### Keeping FTS in Sync with Your Real Table (Triggers)

```sql
-- Auto-sync when data changes in the main table
CREATE TRIGGER articles_ai AFTER INSERT ON articles BEGIN
    INSERT INTO articles_fts(rowid, title, body)
    VALUES (new.id, new.title, new.body);
END;

CREATE TRIGGER articles_ad AFTER DELETE ON articles BEGIN
    INSERT INTO articles_fts(articles_fts, rowid, title, body)
    VALUES ('delete', old.id, old.title, old.body);
END;

CREATE TRIGGER articles_au AFTER UPDATE ON articles BEGIN
    INSERT INTO articles_fts(articles_fts, rowid, title, body)
    VALUES ('delete', old.id, old.title, old.body);
    INSERT INTO articles_fts(rowid, title, body)
    VALUES (new.id, new.title, new.body);
END;
```

> 💡 **Pro Tip**: FTS5 is built into SQLite (no extension needed in most distributions). It's perfect for in-app search, autocomplete, and filtering.

---

## 14. JSON Support (JSON1) — Document Database Inside SQLite

### SQLite as a Document Store

Since SQLite 3.38+ (2022), JSON functions are **built-in** (previously required loading an extension). This effectively gives you MongoDB-like capabilities inside SQLite.

```sql
-- ── Store JSON Data ──────────────────────────────────────────
CREATE TABLE events (
    id   INTEGER PRIMARY KEY,
    data TEXT NOT NULL  -- Store JSON as TEXT
);

INSERT INTO events (data) VALUES ('
{
    "type": "purchase",
    "user": { "id": 42, "name": "Ritesh" },
    "items": [
        { "product": "Laptop", "price": 50000 },
        { "product": "Mouse", "price": 500 }
    ],
    "timestamp": "2026-06-02T14:30:00Z"
}
');

-- ── Extract JSON Values ──────────────────────────────────────
-- json_extract() or the -> and ->> operators (3.38+)

SELECT json_extract(data, '$.type') FROM events;
-- "purchase"

SELECT data ->> '$.user.name' FROM events;
-- Ritesh (unquoted — ->> removes quotes)

SELECT data -> '$.user.name' FROM events;
-- "Ritesh" (quoted — -> keeps JSON formatting)

SELECT data ->> '$.items[0].product' FROM events;
-- Laptop

SELECT data ->> '$.items[1].price' FROM events;
-- 500

-- ── Query JSON Arrays ────────────────────────────────────────
-- Expand array into rows with json_each()
SELECT 
    e.id,
    j.value ->> '$.product' as product,
    j.value ->> '$.price' as price
FROM events e, json_each(e.data, '$.items') j;
-- 1 | Laptop | 50000
-- 1 | Mouse  | 500

-- ── Filter by JSON Values ────────────────────────────────────
SELECT * FROM events 
WHERE data ->> '$.type' = 'purchase'
AND data ->> '$.user.id' = 42;

-- ── Aggregate JSON ───────────────────────────────────────────
SELECT 
    data ->> '$.user.name' as user_name,
    SUM(j.value ->> '$.price') as total_spent
FROM events e, json_each(e.data, '$.items') j
GROUP BY user_name;
-- Ritesh | 50500

-- ── Modify JSON ──────────────────────────────────────────────
UPDATE events 
SET data = json_set(data, '$.status', 'completed')
WHERE id = 1;

UPDATE events 
SET data = json_insert(data, '$.notes', 'Delivered on time')
WHERE id = 1;

UPDATE events
SET data = json_remove(data, '$.items[1]')  -- Remove second item
WHERE id = 1;

-- ── Index JSON for Performance ───────────────────────────────
-- Create an index on a JSON path (generated column + index)
CREATE INDEX idx_events_type ON events(data ->> '$.type');
CREATE INDEX idx_events_user_id ON events(CAST(data ->> '$.user.id' AS INTEGER));

-- Now this query is fast:
SELECT * FROM events WHERE data ->> '$.type' = 'purchase';
-- SEARCH events USING INDEX idx_events_type  ✅
```

### JSON Function Reference

```
╔══════════════════════════════════╦═════════════════════════════════════════╗
║  Function                        ║  Purpose                               ║
╠══════════════════════════════════╬═════════════════════════════════════════╣
║  json(value)                     ║  Validate & minify JSON                ║
║  json_extract(json, path)        ║  Extract value at path                 ║
║  json ->> '$.path'               ║  Extract (unquoted) — 3.38+           ║
║  json -> '$.path'                ║  Extract (JSON-formatted) — 3.38+     ║
║  json_set(json, path, value)     ║  Set value (overwrite if exists)       ║
║  json_insert(json, path, value)  ║  Insert (skip if exists)               ║
║  json_replace(json, path, value) ║  Replace (skip if NOT exists)          ║
║  json_remove(json, path)         ║  Remove value at path                  ║
║  json_type(json, path)           ║  Get JSON type at path                 ║
║  json_valid(json)                ║  Check if string is valid JSON         ║
║  json_array(v1, v2, ...)         ║  Create JSON array                     ║
║  json_object(k1,v1, k2,v2, ...) ║  Create JSON object                    ║
║  json_group_array(value)         ║  Aggregate into JSON array             ║
║  json_group_object(key, value)   ║  Aggregate into JSON object            ║
║  json_each(json, path)           ║  Table-valued: iterate array/object    ║
║  json_tree(json, path)           ║  Table-valued: recursive traverse      ║
╚══════════════════════════════════╩═════════════════════════════════════════╝
```

---

## 15. Advanced Features You Should Know

### R-Tree Index (Spatial Data)

```sql
-- Perfect for geospatial queries, bounding-box searches, range queries
CREATE VIRTUAL TABLE spatial_index USING rtree(
    id,
    min_x, max_x,    -- Longitude range
    min_y, max_y     -- Latitude range
);

-- Insert bounding boxes for locations
INSERT INTO spatial_index VALUES (1, 77.0, 77.5, 28.4, 28.8);  -- Delhi
INSERT INTO spatial_index VALUES (2, 72.7, 73.1, 18.8, 19.3);  -- Mumbai

-- Find locations within a region (e.g., "what's near me?")
SELECT * FROM spatial_index
WHERE min_x >= 77.0 AND max_x <= 78.0
AND min_y >= 28.0 AND max_y <= 29.0;
-- Returns: Delhi (id=1)
```

### Partial Indexes (Save Space, Boost Speed)

```sql
-- Index only active users (ignore millions of inactive ones)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = 1;

-- Index only recent orders
CREATE INDEX idx_recent_orders ON orders(user_id, total)
WHERE order_date > '2026-01-01';

-- The query planner uses partial indexes automatically
SELECT * FROM users WHERE email = 'ritesh@example.com' AND is_active = 1;
-- Uses idx_active_users (much smaller than full index!)
```

### Expression Indexes

```sql
-- Index on an expression (computed value)
CREATE INDEX idx_users_lower_email ON users(lower(email));

-- Now case-insensitive search is fast:
SELECT * FROM users WHERE lower(email) = 'ritesh@example.com';
-- SEARCH users USING INDEX idx_users_lower_email  ✅
```

### SAVEPOINT (Nested Transactions)

```sql
BEGIN TRANSACTION;

INSERT INTO orders (product, qty) VALUES ('Laptop', 1);

SAVEPOINT before_accessories;
INSERT INTO orders (product, qty) VALUES ('Mouse', 2);
INSERT INTO orders (product, qty) VALUES ('Keyboard', 1);

-- Oops, roll back just the accessories
ROLLBACK TO SAVEPOINT before_accessories;

-- Laptop order is still intact!
COMMIT;
```

### In-Memory Databases

```sql
-- Create a database entirely in RAM (ultra-fast, lost when connection closes)
-- From CLI:
sqlite3 :memory:

-- From Python:
conn = sqlite3.connect(':memory:')

-- Perfect for:
-- • Unit testing (fresh DB for each test)
-- • Temporary data processing
-- • Caching layers
```

### Virtual Tables (Extensibility)

```sql
-- CSV virtual table (query CSV files with SQL!)
-- Requires: .load csv
CREATE VIRTUAL TABLE temp.csv_data USING csv(
    filename='data.csv',
    header=yes
);
SELECT * FROM csv_data WHERE age > 25;

-- Or use the built-in .import command:
.mode csv
.import data.csv my_table
SELECT COUNT(*) FROM my_table;
```

---

## 16. The SQLite CLI — Power User Commands

### Essential Dot Commands

```
╔═══════════════════════════════╦═════════════════════════════════════════╗
║  Command                      ║  What It Does                          ║
╠═══════════════════════════════╬═════════════════════════════════════════╣
║  .help                        ║  Show all dot commands                  ║
║  .tables                      ║  List all tables                        ║
║  .schema                      ║  Show CREATE statements for everything  ║
║  .schema users                ║  Show CREATE statement for 'users'      ║
║  .headers on                  ║  Show column headers in output          ║
║  .mode column                 ║  Pretty column-aligned output           ║
║  .mode csv                    ║  Output as CSV                          ║
║  .mode json                   ║  Output as JSON (3.33+)                ║
║  .mode markdown               ║  Output as Markdown table              ║
║  .output results.csv          ║  Redirect output to file                ║
║  .output stdout               ║  Reset output to terminal               ║
║  .import data.csv tablename   ║  Import CSV into table                  ║
║  .dump                        ║  Export entire DB as SQL                 ║
║  .dump users                  ║  Export one table as SQL                 ║
║  .read script.sql             ║  Execute SQL from file                  ║
║  .backup backup.db            ║  Create backup of database              ║
║  .databases                   ║  List all attached databases             ║
║  .indexes                     ║  List all indexes                        ║
║  .timer on                    ║  Show execution time for queries         ║
║  .eqp on                     ║  Show EXPLAIN QUERY PLAN for every query ║
║  .quit                        ║  Exit SQLite shell                       ║
╚═══════════════════════════════╩═════════════════════════════════════════╝
```

### Practical CLI Workflows

```bash
# ── Export database to SQL ────────────────────────────────────
sqlite3 myapp.db .dump > backup.sql

# ── Restore from SQL dump ─────────────────────────────────────
sqlite3 newdb.db < backup.sql

# ── Import CSV ────────────────────────────────────────────────
sqlite3 analytics.db <<EOF
.mode csv
.import sales_data.csv sales
SELECT COUNT(*) FROM sales;
EOF

# ── One-liner queries ─────────────────────────────────────────
sqlite3 myapp.db "SELECT COUNT(*) FROM users;"
sqlite3 -header -column myapp.db "SELECT name, email FROM users LIMIT 10;"
sqlite3 -json myapp.db "SELECT * FROM users;" > users.json

# ── Pretty output setup ──────────────────────────────────────
# Create ~/.sqliterc (auto-runs on startup)
echo ".headers on
.mode column
.timer on
PRAGMA foreign_keys = ON;" > ~/.sqliterc
```

---

## 17. SQLite in the Modern Era — Litestream, LiteFS & Turso

### The Renaissance of SQLite

SQLite is experiencing a massive resurgence thanks to new tools that solve its traditional limitations:

```
╔══════════════════════════════════════════════════════════════════╗
║  THE SQLITE RENAISSANCE (2020s)                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  🔄 Litestream — Streaming replication to S3/Azure/GCS          ║
║     → Continuous backup of SQLite to cloud storage               ║
║     → Disaster recovery in seconds                               ║
║     → No more "what if the server dies?"                         ║
║                                                                  ║
║  📁 LiteFS — Distributed SQLite with FUSE filesystem            ║
║     → Replicate SQLite across multiple servers                   ║
║     → Built by the Fly.io team                                   ║
║     → Read replicas across regions                               ║
║                                                                  ║
║  🌐 Turso (libSQL) — SQLite for the edge                       ║
║     → SQLite fork with built-in replication                      ║
║     → Embedded replicas — database at the edge                   ║
║     → Used by: Vercel, Netlify, Fly.io apps                     ║
║                                                                  ║
║  ⚡ SQLite on the server is now viable for MANY web apps        ║
║     → "Just use SQLite" is no longer crazy advice                ║
║     → Simpler ops, lower costs, blazing speed                    ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Why "SQLite on the Server" Is a Movement

```
Traditional Architecture:
┌──────────┐          ┌────────────────┐
│  App     │───TCP───→│  PostgreSQL    │  ← Network latency on EVERY query
│  Server  │          │  Server        │  ← Separate process to manage
└──────────┘          └────────────────┘  ← Connection pooling needed
                                          ← Scaling = complex

SQLite-on-Server Architecture:
┌────────────────────────────────┐
│  App Server                    │
│  ┌──────────────────────────┐  │
│  │  SQLite (embedded)       │  │  ← ZERO network latency
│  │  📄 app.db               │  │  ← No connection pooling
│  └──────────────────────────┘  │  ← No separate process
│       │                        │  ← Simpler operations
│       ↓                        │
│  Litestream → S3 (backup)      │  ← Continuous disaster recovery
└────────────────────────────────┘

When this works:
✅ Single-server apps
✅ Read-heavy workloads  
✅ Low-to-medium write load
✅ Edge/serverless functions
✅ You want simplicity over scalability
```

---

## 18. SQLite vs Other Databases — When to Choose What

### The Honest Comparison

```
╔══════════════════════╦══════════════╦═══════════════╦═════════════╦══════════════╗
║ Feature              ║ SQLite       ║ PostgreSQL    ║ MySQL       ║ MongoDB      ║
╠══════════════════════╬══════════════╬═══════════════╬═════════════╬══════════════╣
║ Setup time           ║ 0 seconds    ║ 5-30 min      ║ 5-15 min    ║ 5-15 min     ║
║ Admin needed         ║ None         ║ Yes           ║ Yes         ║ Yes          ║
║ Concurrent writers   ║ 1 ❌        ║ Many ✅       ║ Many ✅     ║ Many ✅      ║
║ Concurrent readers   ║ Many ✅     ║ Many ✅       ║ Many ✅     ║ Many ✅      ║
║ Max practical size   ║ ~100 GB      ║ Terabytes     ║ Terabytes   ║ Petabytes    ║
║ Replication          ║ Litestream   ║ Built-in      ║ Built-in    ║ Built-in     ║
║ Full-text search     ║ FTS5 ✅     ║ tsvector ✅   ║ FULLTEXT ✅ ║ Atlas Search ║
║ JSON support         ║ Good ✅     ║ Best (JSONB)  ║ Good ✅     ║ Native       ║
║ Stored procedures    ║ ❌ No       ║ ✅ PL/pgSQL   ║ ✅ Yes      ║ ❌ No        ║
║ Cost                 ║ Free (PD)    ║ Free (OSS)    ║ Free (GPL)  ║ Free (SSPL)  ║
║ Deployment           ║ Copy a file  ║ Server        ║ Server      ║ Server       ║
║ Best for             ║ Embedded,    ║ Complex apps, ║ Web apps,   ║ Flexible     ║
║                      ║ mobile, edge ║ analytics     ║ CMS, SaaS   ║ schema apps  ║
╚══════════════════════╩══════════════╩═══════════════╩═════════════╩══════════════╝
```

### Decision Matrix

```
"Which database should I use?"

┌─────────────────────────────────────────────────────────────┐
│  Building a mobile app?             → SQLite (on device)    │
│  Building a desktop app?            → SQLite                │
│  Building a small web app?          → SQLite (with backups) │
│  Building a medium web app?         → PostgreSQL or MySQL   │
│  Building a large-scale web app?    → PostgreSQL + Redis    │
│  Need flexible schema?              → MongoDB               │
│  IoT / embedded device?             → SQLite                │
│  Data analysis on a laptop?         → SQLite (+ DuckDB)     │
│  Microservices with many writers?   → PostgreSQL             │
│  Learning SQL for the first time?   → SQLite (easiest start)│
│  Need to embed DB in your binary?   → SQLite (the ONLY opt) │
└─────────────────────────────────────────────────────────────┘
```

---

## 19. Common Pitfalls & How to Avoid Them

```
╔══════════════════════════════════════════════════════════════════╗
║  PITFALL                            │  SOLUTION                 ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  1. Foreign keys not enforced       │  PRAGMA foreign_keys = ON ║
║     (default is OFF!)               │  Set on EVERY connection  ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  2. "database is locked" errors     │  PRAGMA busy_timeout=5000 ║
║                                     │  Use WAL mode             ║
║                                     │  Use BEGIN IMMEDIATE      ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  3. Slow bulk inserts               │  Wrap in BEGIN/COMMIT     ║
║                                     │  Use executemany()        ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  4. Wrong data types accepted       │  Use STRICT tables        ║
║     ("hello" in INTEGER column)     │  Use CHECK constraints    ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  5. Database corruption on NFS      │  NEVER use SQLite on      ║
║                                     │  network filesystems      ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  6. Database growing forever        │  Run VACUUM periodically  ║
║     (even after DELETEs)            │  Or use auto_vacuum       ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  7. Queries getting slow            │  Run ANALYZE              ║
║                                     │  Check EXPLAIN QUERY PLAN ║
║                                     │  Add appropriate indexes  ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  8. SQL injection                   │  ALWAYS use ? parameters  ║
║                                     │  NEVER concatenate SQL    ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  9. Not closing connections         │  Use context managers     ║
║                                     │  (with/using/defer)       ║
╠═════════════════════════════════════╪═══════════════════════════╣
║  10. Using SQLite when you          │  If >100 write ops/sec    ║
║      shouldn't                      │  from multiple processes  ║
║                                     │  → use PostgreSQL/MySQL   ║
╚═════════════════════════════════════╧═══════════════════════════╝
```

---

## 20. Interview-Ready Knowledge

### "Tell me about SQLite" (30-Second Answer)

> *"SQLite is the most deployed database engine in the world — estimated over a trillion active databases. It's serverless, zero-configuration, and stores the entire database in a single cross-platform file. Unlike client-server databases, SQLite is linked directly into the application as a library. It's ACID-compliant, supports full SQL, and is used in every smartphone, every browser, and countless embedded devices. It's ideal for mobile apps, desktop apps, embedded systems, and small-to-medium web applications, but not for high-concurrency write-heavy workloads."*

### Common Interview Questions

| Question | Key Points |
|----------|-----------|
| "When would you NOT use SQLite?" | High write concurrency, client-server apps, very large datasets, replication needs |
| "How does SQLite handle concurrency?" | Single writer + multiple readers (WAL mode). No multi-writer support |
| "What is WAL mode?" | Write-Ahead Logging — writers don't block readers, 10x write performance, changes go to separate WAL file |
| "How is data stored?" | Single file, B-tree pages, fixed page size (default 4KB) |
| "Does SQLite support ACID?" | Yes, full ACID. Uses journal/WAL for atomicity and durability |
| "Max database size?" | 281 TB theoretical, ~100 GB practical |
| "Why is SQLite public domain?" | Creator D. Richard Hipp intentionally released it with no license restrictions |
| "What's the type system?" | Type affinity (5 storage classes), not strict types (unless STRICT tables) |
| "FTS5?" | Full-text search with BM25 ranking, phrase/boolean queries, highlight/snippet |
| "SQLite vs MySQL for web app?" | MySQL for multi-user web apps; SQLite for single-server, read-heavy, or embedded |

---

## 🔑 Key Takeaways

```
✅ SQLite = the most deployed database on Earth (~1 trillion+ instances)
✅ Serverless, zero-config — just a single file
✅ Perfect for: mobile, desktop, embedded, IoT, testing, small web apps
✅ NOT for: high write concurrency, client-server, multi-server setups
✅ ALWAYS enable: WAL mode, foreign_keys, busy_timeout
✅ ALWAYS batch writes in transactions (4000x speedup!)
✅ FTS5 gives you Google-like search, JSON1 gives you MongoDB-like flexibility
✅ The SQLite Renaissance: Litestream, LiteFS, Turso making it viable for servers
✅ Type affinity is unique — use STRICT tables for safety
✅ Public domain — truly free, no license worries, used forever
```

---

## 🔗 What's Next?

**Chapter 2F.2 → [SQLite Advanced Usage & Optimization](./02-SQLite-Advanced.md)**
Where we go deeper into FTS5 tuning, R-Tree mastery, advanced PRAGMA tuning, and building production-grade applications with SQLite.

---

> *"Think of SQLite not as a replacement for Oracle, but as a replacement for fopen()."*
> — D. Richard Hipp, Creator of SQLite

---

> *"SQLite doesn't compete with client-server databases. SQLite competes with custom file formats. And it wins — every single time."*
