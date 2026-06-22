# 🛋️ Chapter 3G.2 — CouchDB & PouchDB — Offline-First Databases

> **Level:** 🟡 Intermediate
> **Time to Master:** ~3–4 hours
> **Prerequisites:** Chapter 3A.1 (NoSQL Overview), Chapter 1.5 (ACID vs BASE)

> **"The network is NOT reliable. The best apps work perfectly even when it disappears."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand the **Offline-First philosophy** and why it matters
- Know **CouchDB's architecture** — HTTP-native, document-oriented, replicate-everything
- Master **MVCC (Multi-Version Concurrency Control)** — how CouchDB handles conflicts
- Understand the **CouchDB Replication Protocol** — the magic behind sync
- Know **PouchDB** — CouchDB that lives in your browser/phone
- Build mental models for **conflict resolution** in distributed systems
- Know exactly **when to choose CouchDB/PouchDB** over MongoDB or DynamoDB

---

## 1. The Problem — Why Offline-First Matters

### The Painful Reality of Modern Apps

```
Scenario 1: Field Worker
────────────────────────
  A healthcare worker in rural India is recording patient data on a tablet.
  
  WiFi? 😂 Barely exists.
  Cell signal? Comes and goes every 30 minutes.
  
  Traditional App:                    Offline-First App:
  ┌─────────────┐                    ┌─────────────┐
  │  "Saving... │                    │  ✅ Saved!   │
  │   ⏳⏳⏳    │                    │  (locally)   │
  │  ❌ Network │                    │  Will sync   │
  │    Error!   │                    │  when online │
  │  Data LOST!"│                    │  💯 No data  │
  └─────────────┘                    │    loss ever │
                                     └─────────────┘

Scenario 2: Subway Commuter
────────────────────────────
  You're editing a note on your phone in the subway.
  Signal drops every tunnel. Traditional app loses your edits.
  Offline-first app? Works like nothing happened.

Scenario 3: IoT / Edge Computing
──────────────────────────────────
  Smart factory sensors collecting data.
  Network to cloud goes down for 2 hours.
  Without offline-first → 2 hours of data LOST.
  With CouchDB/PouchDB → syncs everything when connection returns.
```

### The Offline-First Manifesto

```
┌─────────────────────────────────────────────────────────────┐
│              OFFLINE-FIRST PRINCIPLES                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. The app works WITHOUT any network connection            │
│  2. Data is stored LOCALLY first, synced to server later    │
│  3. Conflicts are inevitable — handle them gracefully       │
│  4. The user should NEVER see a loading spinner for reads   │
│  5. Sync happens automatically when connectivity returns    │
│  6. The app should be indistinguishable from an online app  │
│                                                             │
│  "Treat the network as an enhancement, not a requirement."  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. CouchDB — The Original Offline-First Database

### What Is CouchDB?

```
┌─────────────────────────────────────────────────────────────────┐
│                    Apache CouchDB                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  • Document database (JSON documents)                           │
│  • Written in Erlang (built for fault tolerance!)               │
│  • HTTP/REST API — every operation is an HTTP request           │
│  • Built-in replication — peer-to-peer, master-master           │
│  • MVCC — never overwrites data, creates new versions           │
│  • MapReduce views for queries                                  │
│  • Mango queries (JSON-based query language)                    │
│  • Created by Damien Katz (ex-Lotus Notes developer)           │
│  • Apache Foundation project since 2008                        │
│                                                                 │
│  Core Philosophy: "Relax." — CouchDB's motto                  │
│  (Because you relax knowing your data is safe and synced)      │
└─────────────────────────────────────────────────────────────────┘
```

### CouchDB vs MongoDB — The Key Differences

```
┌──────────────────────┬──────────────────┬──────────────────┐
│  Feature             │  CouchDB 🛋️      │  MongoDB 🍃      │
├──────────────────────┼──────────────────┼──────────────────┤
│  Protocol            │  HTTP/REST       │  Binary (Wire)   │
│  Query Language      │  Mango + MR      │  MQL (Rich)      │
│  Replication         │  Built-in P2P    │  Replica Sets    │
│  Conflict Handling   │  First-class     │  Last-write-wins │
│  Offline Support     │  Core feature    │  Not built-in    │
│  API Style           │  RESTful         │  Driver-based    │
│  Storage Engine      │  B+Tree          │  WiredTiger      │
│  Scaling Model       │  Replication     │  Sharding        │
│  Ad-hoc Queries      │  Limited         │  Excellent       │
│  Aggregation         │  MapReduce       │  Pipeline        │
│  Use Case            │  Sync/Offline    │  General Purpose │
│  Community           │  Smaller         │  Massive         │
│  Learning Curve      │  🟢 Easy API     │  🟢 Easy         │
└──────────────────────┴──────────────────┴──────────────────┘
```

> 💡 **Key Insight**: MongoDB optimizes for **query flexibility and performance**. CouchDB optimizes for **reliability, replication, and offline access**. They solve different problems.

---

## 3. CouchDB Architecture — Everything Is HTTP

### The HTTP API — Your Database Is a Web Server

This is CouchDB's most distinctive feature. **Every single operation** — from creating databases to querying documents — is a standard HTTP request.

```bash
# ── Create a database ──
curl -X PUT http://localhost:5984/myapp
# Response: {"ok": true}

# ── Create a document ──
curl -X POST http://localhost:5984/myapp \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Ritesh",
    "email": "ritesh@example.com",
    "skills": ["Python", "CouchDB"],
    "joined": "2024-01-15"
  }'
# Response: {
#   "ok": true,
#   "id": "a1b2c3d4...",
#   "rev": "1-abc123..."        ← Revision number!
# }

# ── Read a document ──
curl http://localhost:5984/myapp/a1b2c3d4
# Response: Full JSON document with _id and _rev

# ── Update a document (MUST include _rev!) ──
curl -X PUT http://localhost:5984/myapp/a1b2c3d4 \
  -H "Content-Type: application/json" \
  -d '{
    "_rev": "1-abc123...",     ← REQUIRED! Proves you saw the latest version
    "name": "Ritesh Singh",
    "email": "ritesh@example.com",
    "skills": ["Python", "CouchDB", "PouchDB"],
    "joined": "2024-01-15"
  }'
# Response: {"ok": true, "rev": "2-def456..."}   ← New revision!

# ── Delete a document ──
curl -X DELETE http://localhost:5984/myapp/a1b2c3d4?rev=2-def456

# ── List all databases ──
curl http://localhost:5984/_all_dbs

# ── Get database info ──
curl http://localhost:5984/myapp
# Response: {"db_name":"myapp","doc_count":1,"update_seq":"1-..."}
```

### Why HTTP Matters

```
Traditional Database:                    CouchDB:
  App → Special Driver → DB             App → HTTP Request → DB
  
  Need drivers for:                      Works with:
  • Python driver                        • curl
  • Java driver                          • Any HTTP library
  • Node.js driver                       • Web browser (directly!)
  • Go driver                            • Postman
  • Ruby driver                          • wget
  • ... install, configure,              • fetch() in JavaScript
    manage versions                      • ANY language that speaks HTTP
  
  💡 CouchDB can be used by any device that can make HTTP requests.
     No special drivers needed. No binary protocols to implement.
     Your database IS a web API.
```

### CouchDB Internal Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                CouchDB Architecture                         │
│                                                             │
│  ┌───────────────────────────────────┐                     │
│  │         HTTP Request Handler       │  ← Erlang/OTP      │
│  │  (Built on Erlang's massive       │     (designed for    │
│  │   concurrency model)              │      telecom-grade   │
│  └─────────────┬─────────────────────┘      reliability)   │
│                │                                            │
│      ┌─────────┼──────────┐                                │
│      ▼         ▼          ▼                                │
│  ┌──────┐ ┌──────────┐ ┌──────────────┐                   │
│  │Query │ │  MVCC    │ │ Replication  │                    │
│  │Engine│ │  Engine  │ │   Engine     │                    │
│  └──┬───┘ └────┬─────┘ └──────┬───────┘                   │
│     │          │               │                            │
│     ▼          ▼               ▼                            │
│  ┌─────────────────────────────────────┐                   │
│  │        B+Tree Storage Engine        │                   │
│  │                                     │                   │
│  │  ┌────────┐  ┌────────┐  ┌──────┐  │                   │
│  │  │  Docs  │  │ Views  │  │ Meta │  │                   │
│  │  │ B+Tree │  │ B+Tree │  │ data │  │                   │
│  │  └────────┘  └────────┘  └──────┘  │                   │
│  │                                     │                   │
│  │  Append-Only! Never overwrites.    │                   │
│  │  (Compaction reclaims space later)  │                   │
│  └─────────────────────────────────────┘                   │
│                                                             │
│  Key Properties:                                           │
│  • Append-only B+Tree → Crash-proof (no partial writes)   │
│  • MVCC → Readers never block writers                      │
│  • Erlang OTP → Millions of concurrent connections        │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. MVCC — Multi-Version Concurrency Control

### The Core Idea

In most databases, when you update a document, the old version is **overwritten**. In CouchDB, the old version is **kept**, and a new version is **appended**.

```
Traditional DB (Overwrite):
  Version 1: { name: "Ritesh" }
       ↓ UPDATE
  Version 1: { name: "Ritesh Singh" }    ← Old data GONE forever
  

CouchDB (Append-Only MVCC):
  Rev 1-abc: { name: "Ritesh" }          ← Still exists!
  Rev 2-def: { name: "Ritesh Singh" }    ← New version appended
  Rev 3-ghi: { name: "Ritesh S." }       ← Latest version
  
  ┌──────────────────────────────────────────────────────────┐
  │  The _rev field is a VERSION NUMBER:                      │
  │                                                          │
  │  "1-abc123"  →  Revision 1 (generation-hash)            │
  │   ↑     ↑                                                │
  │   │     └── Hash of document content                     │
  │   └── Generation number (increments each update)         │
  │                                                          │
  │  "2-def456"  →  Revision 2                              │
  │  "3-ghi789"  →  Revision 3 (current/winning)            │
  └──────────────────────────────────────────────────────────┘
```

### Why MVCC Matters

```
Scenario: Two users editing the same document simultaneously

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  1. Alice reads document (rev: 1-abc)                        │
│  2. Bob reads document (rev: 1-abc)                          │
│  3. Alice updates → sends rev 1-abc → SUCCESS → rev 2-def   │
│  4. Bob updates → sends rev 1-abc → CONFLICT! ❌             │
│     (Because current rev is now 2-def, not 1-abc)            │
│                                                              │
│  CouchDB says: "Hey Bob, someone else changed this.         │
│  Here's the latest version. Merge your changes."             │
│                                                              │
│  This is called OPTIMISTIC CONCURRENCY CONTROL:              │
│  • No locks                                                  │
│  • No waiting                                                │
│  • Conflicts detected, not prevented                         │
│  • Application decides how to resolve                        │
└──────────────────────────────────────────────────────────────┘
```

### Handling the 409 Conflict

```bash
# Bob's update attempt returns HTTP 409 Conflict
curl -X PUT http://localhost:5984/myapp/doc1 \
  -d '{"_rev": "1-abc", "name": "Bob's version"}'
# Response: {"error": "conflict", "reason": "Document update conflict."}

# Bob must:
# 1. GET the latest version
curl http://localhost:5984/myapp/doc1
# Response: {"_rev": "2-def", "name": "Alice's version"}

# 2. Merge changes and retry with latest _rev
curl -X PUT http://localhost:5984/myapp/doc1 \
  -d '{"_rev": "2-def", "name": "Merged version"}'
# Response: {"ok": true, "rev": "3-ghi"}
```

### Compaction — Cleaning Up Old Revisions

```
Over time, old revisions pile up:
  Rev 1 → Rev 2 → Rev 3 → Rev 4 → ... → Rev 500

Compaction removes old revision BODIES (keeps the rev history tree):
  Before: [Body1][Body2][Body3]...[Body500]  → Large file
  After:  [     ][     ][     ]...[Body500]  → Reclaimed space!

# Trigger compaction
curl -X POST http://localhost:5984/myapp/_compact
```

> 💡 **Pro Tip**: CouchDB stores data in an **append-only** file. This means it never corrupts data during crashes (no partial writes), but the file grows indefinitely. Compaction is essential for production use.

---

## 5. Querying CouchDB — Views & Mango

### Method 1: MapReduce Views (The Original Way)

MapReduce views are **pre-computed indexes** defined as JavaScript functions.

```javascript
// ── Design Document (stored IN the database) ──
// PUT http://localhost:5984/myapp/_design/users

{
  "_id": "_design/users",
  "views": {
    "by_email": {
      "map": "function(doc) { if (doc.email) { emit(doc.email, doc.name); } }"
    },
    "by_city": {
      "map": "function(doc) { if (doc.address && doc.address.city) { emit(doc.address.city, 1); } }",
      "reduce": "_count"
    },
    "by_skill": {
      "map": "function(doc) { if (doc.skills) { doc.skills.forEach(function(skill) { emit(skill, doc.name); }); } }"
    }
  }
}
```

```bash
# ── Query the view ──

# All users by email (sorted)
curl http://localhost:5984/myapp/_design/users/_view/by_email

# Specific email
curl http://localhost:5984/myapp/_design/users/_view/by_email?key="ritesh@example.com"

# Range of emails
curl 'http://localhost:5984/myapp/_design/users/_view/by_email?startkey="a"&endkey="m"'

# Count users per city (reduce)
curl http://localhost:5984/myapp/_design/users/_view/by_city?group=true
# Response: {"rows":[{"key":"Delhi","value":15},{"key":"Mumbai","value":8}]}

# All users with skill "Python"
curl http://localhost:5984/myapp/_design/users/_view/by_skill?key="Python"
```

### How MapReduce Views Work Internally

```
Step 1: MAP function runs on every document
─────────────────────────────────────────────
  doc1 { name:"Ritesh", city:"Delhi" }  →  emit("Delhi", 1)
  doc2 { name:"Priya",  city:"Mumbai"} →  emit("Mumbai", 1)
  doc3 { name:"Amit",   city:"Delhi" }  →  emit("Delhi", 1)

Step 2: Results sorted by key → stored in B+Tree
──────────────────────────────────────────────────
  ┌─────────┬───────┐
  │   Key   │ Value │
  │ "Delhi" │   1   │
  │ "Delhi" │   1   │
  │ "Mumbai"│   1   │
  └─────────┴───────┘

Step 3: REDUCE aggregates (optional)
─────────────────────────────────────
  "Delhi"  → _count → 2
  "Mumbai" → _count → 1

💡 Views are computed ONCE, then incrementally updated.
   Subsequent queries are FAST (B+Tree lookup, not re-computation).
```

### Method 2: Mango Queries (The Modern Way)

Mango is CouchDB's **MongoDB-inspired** JSON query language. Much easier than MapReduce for simple queries.

```bash
# ── Create an index ──
curl -X POST http://localhost:5984/myapp/_index \
  -H "Content-Type: application/json" \
  -d '{
    "index": {
      "fields": ["age", "city"]
    },
    "name": "age-city-index",
    "type": "json"
  }'

# ── Query using Mango ──
curl -X POST http://localhost:5984/myapp/_find \
  -H "Content-Type: application/json" \
  -d '{
    "selector": {
      "age": { "$gte": 25 },
      "city": "Delhi"
    },
    "fields": ["name", "email", "age"],
    "sort": [{ "age": "asc" }],
    "limit": 10
  }'

# ── Mango Selector Operators ──
# $eq, $ne        → Equal, Not equal
# $gt, $gte       → Greater than (or equal)
# $lt, $lte       → Less than (or equal)
# $in, $nin       → In array, Not in array
# $exists         → Field exists
# $type           → Field type check
# $and, $or, $not → Logical operators
# $regex          → Regular expression match
# $elemMatch      → Array element matching
```

### MapReduce vs Mango — When to Use Which

| Feature | MapReduce Views | Mango Queries |
|---------|----------------|---------------|
| Flexibility | Very high (custom JS) | Standard operators |
| Aggregations | ✅ (reduce functions) | ❌ Not supported |
| Setup | Design documents | Create index + query |
| Learning Curve | Steep | Easy (like MongoDB) |
| Performance | Excellent (pre-computed) | Good (with indexes) |
| Use When | Complex transformations, counts, sums | Simple CRUD queries |

---

## 6. The Replication Protocol — CouchDB's Superpower

### What Makes CouchDB Replication Special

```
┌────────────────────────────────────────────────────────────────┐
│          CouchDB Replication — The Magic                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Most databases: Master → Slave (one-way)                      │
│  CouchDB:        Peer ↔ Peer (bidirectional, multi-master!)   │
│                                                                │
│  Any node can accept writes. Changes sync to all others.       │
│  Network goes down? No problem. Sync when it's back.           │
│                                                                │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐      │
│  │  Couch   │◄───────►│  Couch   │◄───────►│  Couch   │      │
│  │  Node A  │         │  Node B  │         │  Node C  │      │
│  │ (Delhi)  │         │ (Mumbai) │         │ (London) │      │
│  └──────────┘         └──────────┘         └──────────┘      │
│       ▲                    ▲                    ▲              │
│       │                    │                    │              │
│    Writes OK            Writes OK            Writes OK        │
│                                                                │
│  Even if Node B goes offline for 3 days:                       │
│  • A and C continue working normally                           │
│  • When B comes back: automatic sync, no data loss!           │
│  • Conflicts detected and stored for resolution               │
└────────────────────────────────────────────────────────────────┘
```

### Replication Modes

```bash
# ── One-Time (Pull) Replication ──
curl -X POST http://localhost:5984/_replicate \
  -H "Content-Type: application/json" \
  -d '{
    "source": "http://remote-server:5984/myapp",
    "target": "myapp"
  }'

# ── One-Time (Push) Replication ──
curl -X POST http://localhost:5984/_replicate \
  -H "Content-Type: application/json" \
  -d '{
    "source": "myapp",
    "target": "http://remote-server:5984/myapp"
  }'

# ── Continuous Replication (Real-Time Sync!) ──
curl -X POST http://localhost:5984/_replicate \
  -H "Content-Type: application/json" \
  -d '{
    "source": "myapp",
    "target": "http://remote-server:5984/myapp",
    "continuous": true
  }'
# This keeps running! Every change syncs in near-real-time.

# ── Filtered Replication (Sync only specific docs) ──
curl -X POST http://localhost:5984/_replicate \
  -H "Content-Type: application/json" \
  -d '{
    "source": "myapp",
    "target": "http://remote-server:5984/myapp",
    "filter": "mydesign/by_type",
    "query_params": { "type": "order" }
  }'
```

### How Replication Works Internally

```
Step 1: Compare sequence numbers
──────────────────────────────────
  Source: "I'm at sequence 150"
  Target: "I'm at sequence 120"
  → Need to sync changes 121 → 150

Step 2: Get the changes feed
─────────────────────────────
  Source sends: [
    { seq: 121, id: "doc1", rev: "3-xyz" },
    { seq: 122, id: "doc5", rev: "2-abc" },
    ...
    { seq: 150, id: "doc9", rev: "1-def" }
  ]

Step 3: Check which docs target needs
──────────────────────────────────────
  Target checks its rev tree:
    doc1: I have rev 2-uvw → NEED rev 3-xyz ✅
    doc5: I have rev 2-abc → Already up to date ⏭️
    doc9: I don't have this → NEED it ✅

Step 4: Transfer only needed documents
───────────────────────────────────────
  Only doc1 (updated) and doc9 (new) are transferred.
  Minimal bandwidth! Even over slow connections.

Step 5: Handle conflicts
─────────────────────────
  If both sides modified the same doc → CONFLICT
  Both versions are kept! App decides which wins.
```

### The Changes Feed — Real-Time Event Stream

```bash
# ── Get all changes since the beginning ──
curl http://localhost:5984/myapp/_changes

# ── Get changes since a specific sequence ──
curl http://localhost:5984/myapp/_changes?since=120

# ── Long-polling (wait for new changes) ──
curl http://localhost:5984/myapp/_changes?feed=longpoll&since=now

# ── Continuous feed (Server-Sent Events style) ──
curl http://localhost:5984/myapp/_changes?feed=continuous&since=now
# Returns: {"seq":"151-...","id":"doc10","changes":[{"rev":"1-aaa"}]}
#          {"seq":"152-...","id":"doc11","changes":[{"rev":"1-bbb"}]}
#          ... (keeps streaming forever)

# ── EventSource in JavaScript ──
# const changes = new EventSource('/myapp/_changes?feed=eventsource&since=now');
# changes.onmessage = (event) => {
#     const change = JSON.parse(event.data);
#     console.log('Document changed:', change.id);
# };
```

---

## 7. Conflict Resolution — The Hard Problem

### Why Conflicts Happen

```
Timeline:
  t0: Doc "user1" exists with rev 1-aaa on ALL nodes

  t1: Network partition occurs! Node A and B can't communicate.

  t2: Alice (on Node A) updates user1 → rev 2-bbb
      Bob (on Node B) updates user1   → rev 2-ccc

  t3: Network heals. Replication runs.
      
      Node A has: rev 2-bbb (Alice's version)
      Node B has: rev 2-ccc (Bob's version)
      
      CONFLICT! Same generation (2), different content.

  How CouchDB handles this:
  ┌─────────────────────────────────────────────────────────┐
  │  CouchDB picks a DETERMINISTIC WINNER:                  │
  │  • Compares rev hashes alphabetically                   │
  │  • Same winner on ALL nodes (no coordination needed!)   │
  │  • Losing revision is NOT deleted — it's preserved!    │
  │                                                         │
  │  Rev Tree:                                              │
  │       1-aaa                                             │
  │      /     \                                            │
  │   2-bbb   2-ccc ← conflict                            │
  │    (win)   (kept but marked as conflict)               │
  └─────────────────────────────────────────────────────────┘
```

### Detecting and Resolving Conflicts

```bash
# ── Check for conflicts ──
curl http://localhost:5984/myapp/user1?conflicts=true
# Response: {
#   "_id": "user1",
#   "_rev": "2-ccc",           ← Winning revision
#   "name": "Bob's version",
#   "_conflicts": ["2-bbb"]    ← Conflicting revisions
# }

# ── Get the conflicting version ──
curl http://localhost:5984/myapp/user1?rev=2-bbb
# Response: { "name": "Alice's version" }

# ── Resolve: Merge and delete the losing rev ──
# Step 1: Create merged version
curl -X PUT http://localhost:5984/myapp/user1 \
  -d '{
    "_rev": "2-ccc",
    "name": "Merged: Alice + Bob changes",
    "resolved_at": "2024-06-15T10:00:00Z"
  }'

# Step 2: Delete the conflicting revision
curl -X DELETE http://localhost:5984/myapp/user1?rev=2-bbb
```

### Conflict Resolution Strategies

```
┌────────────────────────────────────────────────────────────────┐
│           Conflict Resolution Strategies                       │
├──────────────────┬─────────────────────────────────────────────┤
│  Strategy        │  How It Works                               │
├──────────────────┼─────────────────────────────────────────────┤
│  Last Write Wins │  Use a timestamp field, keep the latest    │
│  (Simple)        │  ⚠️ Clock skew can cause issues            │
│                  │                                             │
│  Merge Fields    │  Combine non-overlapping field changes:    │
│  (Smart)         │  A changed "name", B changed "email"       │
│                  │  → Keep both changes                       │
│                  │                                             │
│  User Decides    │  Show both versions to the user:           │
│  (Safe)          │  "Two people edited this. Which to keep?"  │
│                  │                                             │
│  Domain Logic    │  Apply business rules:                     │
│  (Best)          │  "Higher quantity wins" for inventory      │
│                  │  "Union of items" for shopping cart         │
│                  │                                             │
│  CRDTs           │  Conflict-free Replicated Data Types:      │
│  (Advanced)      │  Mathematical structures that auto-merge   │
│                  │  (counters, sets, maps)                     │
└──────────────────┴─────────────────────────────────────────────┘
```

---

## 8. PouchDB — CouchDB in Your Browser & Phone

### What Is PouchDB?

```
┌─────────────────────────────────────────────────────────────────┐
│                        PouchDB                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PouchDB is an open-source JavaScript database that:           │
│                                                                 │
│  • Runs IN the browser (IndexedDB/WebSQL under the hood)       │
│  • Runs in Node.js                                             │
│  • Runs in Electron/React Native/Cordova                       │
│  • Speaks the CouchDB replication protocol                     │
│  • Syncs automatically with CouchDB/Cloudant/PouchDB Server   │
│                                                                 │
│  Think of it as: "CouchDB that fits in a <script> tag"        │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐      │
│  │                                                      │      │
│  │   Browser/Phone          Server                     │      │
│  │  ┌──────────┐    Sync   ┌──────────┐               │      │
│  │  │ PouchDB  │◄────────►│ CouchDB  │               │      │
│  │  │ (local)  │  (auto)  │ (remote) │               │      │
│  │  └──────────┘           └──────────┘               │      │
│  │      ▲                                              │      │
│  │      │                                              │      │
│  │   Your App                                          │      │
│  │   (works offline!)                                  │      │
│  │                                                      │      │
│  └──────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### PouchDB CRUD — The Complete API

```javascript
// ── Install ──
// npm install pouchdb
// OR: <script src="https://cdn.jsdelivr.net/npm/pouchdb/dist/pouchdb.min.js"></script>

// ── Create a local database ──
const db = new PouchDB('my_local_db');

// ── Create a document ──
const response = await db.put({
  _id: 'user:ritesh',             // You provide the _id
  name: 'Ritesh',
  email: 'ritesh@example.com',
  skills: ['JavaScript', 'CouchDB'],
  created_at: new Date().toISOString()
});
console.log(response);
// { ok: true, id: 'user:ritesh', rev: '1-abc123' }

// Or let PouchDB generate the _id:
const response2 = await db.post({
  name: 'Priya',
  email: 'priya@example.com'
});
// { ok: true, id: 'auto-generated-uuid', rev: '1-xyz789' }

// ── Read a document ──
const doc = await db.get('user:ritesh');
console.log(doc);
// { _id: 'user:ritesh', _rev: '1-abc123', name: 'Ritesh', ... }

// ── Update a document ──
doc.name = 'Ritesh Singh';
doc.skills.push('PouchDB');
const updateResponse = await db.put(doc);
// _rev is automatically included because we fetched the doc first
// { ok: true, id: 'user:ritesh', rev: '2-def456' }

// ── Delete a document ──
const docToDelete = await db.get('user:ritesh');
await db.remove(docToDelete);
// OR: await db.remove('user:ritesh', docToDelete._rev);

// ── Get all documents ──
const allDocs = await db.allDocs({
  include_docs: true,           // Include full document bodies
  startkey: 'user:',            // Only docs starting with "user:"
  endkey: 'user:\ufff0'         // Unicode trick for prefix matching
});
console.log(allDocs.rows);

// ── Bulk operations ──
await db.bulkDocs([
  { _id: 'user:a', name: 'Alice' },
  { _id: 'user:b', name: 'Bob' },
  { _id: 'user:c', name: 'Charlie' }
]);
```

### PouchDB Querying

```javascript
// ── Method 1: allDocs with key ranges (fastest!) ──
const orders = await db.allDocs({
  include_docs: true,
  startkey: 'order:2024-01',
  endkey: 'order:2024-12\ufff0'
});

// ── Method 2: find() with Mango-style queries ──
// Requires pouchdb-find plugin
// npm install pouchdb-find
PouchDB.plugin(require('pouchdb-find'));

// Create an index
await db.createIndex({
  index: { fields: ['type', 'created_at'] }
});

// Query
const result = await db.find({
  selector: {
    type: 'order',
    created_at: { $gte: '2024-01-01' }
  },
  sort: [{ created_at: 'desc' }],
  limit: 20
});

// ── Method 3: MapReduce views ──
const ddoc = {
  _id: '_design/my_views',
  views: {
    by_type: {
      map: function(doc) {
        emit(doc.type, null);
      }.toString()
    }
  }
};
await db.put(ddoc);

const result2 = await db.query('my_views/by_type', {
  key: 'order',
  include_docs: true
});
```

### The Magic — Sync Between PouchDB and CouchDB

```javascript
// ── One-time replication ──

// Pull from remote to local
db.replicate.from('http://localhost:5984/myapp');

// Push from local to remote
db.replicate.to('http://localhost:5984/myapp');

// ── Live Sync (Continuous bidirectional replication!) ──
const sync = db.sync('http://localhost:5984/myapp', {
  live: true,       // Keep syncing (don't stop after one pass)
  retry: true       // Retry on network failure!
});

// ── Listen to sync events ──
sync.on('change', (info) => {
  console.log('Data changed:', info.direction, info.change.docs);
  // Update your UI here!
});

sync.on('paused', (err) => {
  if (err) {
    console.log('Sync paused — offline');
    // Show "offline" indicator
  } else {
    console.log('Sync paused — caught up');
    // Show "synced" indicator ✅
  }
});

sync.on('active', () => {
  console.log('Sync resumed — back online!');
  // Show "syncing..." indicator
});

sync.on('denied', (err) => {
  console.log('Sync denied:', err);
  // Authentication or permission issue
});

sync.on('error', (err) => {
  console.log('Sync error:', err);
  // Handle fatal errors
});

// ── Cancel sync ──
sync.cancel();
```

### Real-World Architecture: Offline-First App

```
┌─────────────────────────────────────────────────────────────────┐
│            Offline-First Architecture with PouchDB              │
│                                                                 │
│  ┌─────────────────────────────────────────┐                   │
│  │            User's Device                 │                   │
│  │                                          │                   │
│  │  ┌────────────┐     ┌──────────────┐    │                   │
│  │  │  React /   │────►│   PouchDB    │    │                   │
│  │  │  Vue /     │◄────│   (Local)    │    │                   │
│  │  │  Angular   │     │              │    │                   │
│  │  └────────────┘     │  IndexedDB   │    │                   │
│  │                     │  under hood  │    │                   │
│  │                     └──────┬───────┘    │                   │
│  └────────────────────────────│─────────────┘                   │
│                               │                                 │
│                          Sync │ (when online)                   │
│                          ─ ─ ─│─ ─ ─ ─ ─ ─                    │
│                               │                                 │
│  ┌────────────────────────────▼─────────────┐                   │
│  │            Server                         │                   │
│  │                                           │                   │
│  │  ┌──────────────┐    ┌───────────────┐   │                   │
│  │  │   CouchDB    │───►│  App Server   │   │                   │
│  │  │   (Central)  │    │  (optional)   │   │                   │
│  │  └──────────────┘    └───────────────┘   │                   │
│  │                                           │                   │
│  │  CouchDB also replicates to:             │                   │
│  │  • Other CouchDB nodes (HA)              │                   │
│  │  • Analytics pipeline                     │                   │
│  │  • Backup servers                        │                   │
│  └───────────────────────────────────────────┘                   │
│                                                                 │
│  ✅ App works 100% offline                                      │
│  ✅ Changes sync when back online                               │
│  ✅ Conflicts handled gracefully                                │
│  ✅ No loading spinners for local data                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Real-World Use Cases

### When CouchDB/PouchDB Shines

```
┌────────────────────────────────────────────────────────────────┐
│  Use Case                │  Why CouchDB/PouchDB               │
├──────────────────────────┼─────────────────────────────────────┤
│  Field data collection   │  Offline-first, sync when back     │
│  (healthcare, census)    │  in range                           │
│                          │                                     │
│  Mobile apps with sync   │  PouchDB in app, CouchDB on        │
│  (notes, tasks, CRM)     │  server, automatic sync            │
│                          │                                     │
│  Multi-site retail       │  Each store has local CouchDB,     │
│  (POS systems)           │  syncs to HQ periodically          │
│                          │                                     │
│  IoT / Edge computing    │  Sensors store locally, batch      │
│                          │  sync to cloud when connected      │
│                          │                                     │
│  Collaborative editing   │  MVCC handles concurrent edits,    │
│  (wikis, shared docs)    │  conflict resolution built-in      │
│                          │                                     │
│  Progressive Web Apps    │  PouchDB in browser = offline      │
│  (PWAs)                  │  capable web app with sync         │
│                          │                                     │
│  Disaster-resilient      │  Multi-master replication means    │
│  systems                 │  no single point of failure        │
└──────────────────────────┴─────────────────────────────────────┘
```

### When NOT to Use CouchDB/PouchDB

```
❌ High-throughput analytics (use ClickHouse, BigQuery)
❌ Complex joins across many entity types (use PostgreSQL)
❌ Real-time aggregations at scale (use Redis, Elasticsearch)
❌ Simple key-value caching (use Redis)
❌ Massive write throughput with single consistency (use Cassandra)
❌ Full-text search (use Elasticsearch)
❌ When you don't need offline support or replication (simpler options exist)
```

---

## 10. CouchDB Ecosystem & Related Tools

```
┌─────────────────────────────────────────────────────────────────┐
│                  CouchDB Ecosystem                              │
├──────────────────┬──────────────────────────────────────────────┤
│  IBM Cloudant    │  CouchDB-as-a-Service (managed cloud)       │
│                  │  Global distribution, full API compatibility │
│                  │                                              │
│  CouchDB 3.x    │  Latest version with clustering (BigCouch)  │
│                  │  Multiple nodes, automatic sharding          │
│                  │                                              │
│  PouchDB Server  │  CouchDB-compatible server written in       │
│                  │  Node.js (lightweight, easy to deploy)       │
│                  │                                              │
│  Fauxton        │  CouchDB's built-in web admin UI            │
│                  │  (like MongoDB Compass for CouchDB)          │
│                  │                                              │
│  Nano           │  Official Node.js client library for         │
│                  │  CouchDB (if you prefer a driver over curl)  │
│                  │                                              │
│  Hoodie         │  Full offline-first framework built on       │
│                  │  CouchDB + PouchDB (like Firebase, but open) │
│                  │                                              │
│  RxDB           │  Reactive database built on PouchDB          │
│                  │  (great for React/Angular real-time UIs)     │
└──────────────────┴──────────────────────────────────────────────┘
```

---

## 11. CouchDB Administration Essentials

```bash
# ── Installation (Ubuntu) ──
sudo apt update
sudo apt install -y couchdb
# Access Fauxton UI: http://localhost:5984/_utils

# ── Configuration ──
# Main config file: /opt/couchdb/etc/local.ini

# ── Security: Create admin user ──
curl -X PUT http://localhost:5984/_node/_local/_config/admins/admin \
  -d '"my-secure-password"'

# ── Create a database ──
curl -X PUT http://admin:password@localhost:5984/production_db

# ── Database-level security ──
curl -X PUT http://admin:password@localhost:5984/production_db/_security \
  -d '{
    "admins": { "names": ["admin"], "roles": ["db_admin"] },
    "members": { "names": ["app_user"], "roles": ["reader"] }
  }'

# ── Compaction (reclaim disk space) ──
curl -X POST http://admin:password@localhost:5984/production_db/_compact

# ── View compaction ──
curl -X POST http://admin:password@localhost:5984/production_db/_compact/my_design_doc

# ── Backup (replicate to another CouchDB) ──
curl -X POST http://admin:password@localhost:5984/_replicate \
  -d '{
    "source": "production_db",
    "target": "http://backup-server:5984/production_db_backup",
    "create_target": true
  }'
```

---

## 🧠 Chapter Summary — Key Takeaways

```
┌─────────────────────────────────────────────────────────────────┐
│        CouchDB & PouchDB — What You Must Remember              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. OFFLINE-FIRST: Data lives locally first, syncs later.      │
│     The app works perfectly without any network.               │
│                                                                 │
│  2. EVERYTHING IS HTTP: No special drivers needed. Your        │
│     database is a REST API. curl is your best friend.          │
│                                                                 │
│  3. MVCC: Documents have revisions (_rev). Updates require     │
│     the current _rev. Conflicts are detected, not prevented.   │
│                                                                 │
│  4. REPLICATION IS THE SUPERPOWER: Bidirectional, peer-to-     │
│     peer, works across continents, handles network failures.   │
│                                                                 │
│  5. POUCHDB = COUCH IN THE BROWSER: Same API, same protocol,  │
│     runs in browser/phone, syncs with CouchDB automatically.  │
│                                                                 │
│  6. CONFLICTS ARE NORMAL: In distributed systems, conflicts    │
│     happen. CouchDB preserves both versions. You decide how    │
│     to merge them.                                             │
│                                                                 │
│  7. NOT FOR EVERYTHING: CouchDB excels at sync/offline use     │
│     cases. For complex queries or analytics, look elsewhere.   │
│                                                                 │
│  8. APPEND-ONLY STORAGE: Never corrupts on crash, but needs    │
│     compaction to reclaim space.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|-------------|-------|
| [3G.3 — InfluxDB & Time-Series Databases](./03-TimeSeries-InfluxDB.md) | Time-series data for IoT, monitoring & analytics |

---

> 💡 **Final Thought**: CouchDB and PouchDB solve a problem most databases don't even acknowledge — **the network is unreliable**. In a world of mobile apps, IoT devices, and edge computing, offline-first isn't a nice-to-have. It's a **survival strategy**. Master this, and you'll build apps that work when nothing else does.

---

*Chapter 3G.2 — Complete. You now understand the offline-first philosophy and the replication protocol that powers it.* 🛋️
