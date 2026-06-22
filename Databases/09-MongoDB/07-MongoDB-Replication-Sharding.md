# 🔄 Chapter 3B.7 — MongoDB Replication & Sharding

> **Level:** 🔴 Advanced | 🔥 High Demand
> **Time to Master:** ~5-6 hours
> **Prerequisites:** Chapter 3B.5 (Indexing), Chapter 3B.6 (Schema Design), Chapter 1.6 (CAP Theorem)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **Replica Sets** — how MongoDB achieves high availability and data durability
- Know how **elections** work when a primary fails (and why you need an odd number of nodes)
- Configure **Read Preferences** and **Write Concerns** for the exact consistency you need
- Understand **Sharding** — how MongoDB distributes data across machines for horizontal scaling
- Choose the **right shard key** (the most important decision in a sharded cluster)
- Know how the **Balancer**, **Chunks**, and **Zones** work together
- Design systems that handle **millions of operations per second**

---

## 🏗️ Part 1: Replica Sets — High Availability

### What Is a Replica Set?

A **Replica Set** is a group of MongoDB servers (called **members**) that maintain the **same copy of data**. If one server dies, another takes over automatically.

```
    MongoDB Replica Set (3 members — minimum for production)

    ┌─────────────────────────────────────────────────────┐
    │                                                     │
    │    ┌─────────────┐                                  │
    │    │   PRIMARY    │ ← All writes go here            │
    │    │  (mongod)    │ ← Reads go here (by default)    │
    │    └──────┬───────┘                                  │
    │           │                                          │
    │           │  Replication (oplog)                     │
    │           │                                          │
    │     ┌─────┴──────┐                                  │
    │     │            │                                  │
    │  ┌──▼────────┐  ┌▼──────────┐                      │
    │  │ SECONDARY  │  │ SECONDARY │ ← Copies of primary  │
    │  │ (mongod)   │  │ (mongod)  │ ← Can serve reads    │
    │  └────────────┘  └───────────┘                      │
    │                                                     │
    │  If PRIMARY dies → Election → One SECONDARY becomes │
    │  the new PRIMARY automatically! (< 12 seconds)      │
    └─────────────────────────────────────────────────────┘
```

### Key Concepts

```
┌──────────────────┬──────────────────────────────────────────────────┐
│ Concept          │ Description                                      │
├──────────────────┼──────────────────────────────────────────────────┤
│ Primary          │ The only member that accepts WRITES              │
│ Secondary        │ Read-only copies, replicate from primary         │
│ Arbiter          │ Votes in elections but holds NO data             │
│ Oplog            │ Operations Log — capped collection of all writes │
│ Heartbeat        │ Members ping each other every 2 seconds          │
│ Election         │ Process of choosing a new primary                │
│ Majority         │ More than half the voting members                │
└──────────────────┴──────────────────────────────────────────────────┘
```

---

### Setting Up a Replica Set

```javascript
// Step 1: Start 3 mongod instances on different ports (or servers)
// mongod --replSet myReplicaSet --port 27017 --dbpath /data/rs0
// mongod --replSet myReplicaSet --port 27018 --dbpath /data/rs1
// mongod --replSet myReplicaSet --port 27019 --dbpath /data/rs2

// Step 2: Connect to one member and initiate
mongosh --port 27017

rs.initiate({
  _id: "myReplicaSet",
  members: [
    { _id: 0, host: "server1:27017" },
    { _id: 1, host: "server2:27018" },
    { _id: 2, host: "server3:27019" }
  ]
})

// Step 3: Check replica set status
rs.status()

// Output shows:
// member 0: PRIMARY
// member 1: SECONDARY
// member 2: SECONDARY

// Step 4: Check who is primary
rs.isMaster()
// or
db.hello()
```

### Useful Replica Set Commands

```javascript
// Add a new member
rs.add("server4:27020")

// Remove a member
rs.remove("server4:27020")

// Add an arbiter (lightweight vote-only member)
rs.addArb("arbiter-server:27021")

// Force a member to step down as primary
rs.stepDown()           // Current primary steps down, election happens
rs.stepDown(300)        // Step down for 300 seconds (won't re-elect self)

// View replica set configuration
rs.conf()

// Reconfigure replica set
var config = rs.conf()
config.members[1].priority = 2    // Higher priority = more likely to become primary
rs.reconfig(config)
```

---

### The Oplog — The Replication Engine

```
    How Replication Works:

    ┌─────────────────────────────────────────────────────────┐
    │                                                         │
    │  PRIMARY                                                │
    │  ┌─────────────────────────────────────────┐            │
    │  │ Write: db.users.insertOne({name:"John"})│            │
    │  └──────────────────┬──────────────────────┘            │
    │                     │                                   │
    │                     ▼                                   │
    │  ┌─────────────────────────────────────────┐            │
    │  │           Oplog (local.oplog.rs)         │            │
    │  │ Capped collection — fixed size, wraps    │            │
    │  │                                          │            │
    │  │ { ts: Timestamp(1710456000, 1),          │            │
    │  │   op: "i",            // insert          │            │
    │  │   ns: "mydb.users",   // namespace       │            │
    │  │   o: {name: "John"}   // the document    │            │
    │  │ }                                        │            │
    │  └──────────┬──────────────────┬────────────┘            │
    │             │                  │                         │
    │     ┌───────▼──────┐   ┌──────▼───────┐                 │
    │     │  SECONDARY 1 │   │  SECONDARY 2 │                 │
    │     │              │   │              │                 │
    │     │  Tails the   │   │  Tails the   │                 │
    │     │  oplog and   │   │  oplog and   │                 │
    │     │  replays ops │   │  replays ops │                 │
    │     └──────────────┘   └──────────────┘                 │
    │                                                         │
    └─────────────────────────────────────────────────────────┘

    Oplog operations:
    "i" = insert
    "u" = update
    "d" = delete
    "c" = command (createCollection, dropDatabase, etc.)
    "n" = no-op (used for heartbeats)
```

```javascript
// View the oplog
use local
db.oplog.rs.find().sort({ $natural: -1 }).limit(5)

// Check oplog size and window
rs.printReplicationInfo()
// Output:
// configured oplog size:   990 MB
// log length start to end: 48532 secs (13.48 hours)
// oplog first event time:  Mon Mar 11 2024 08:00:00
// oplog last event time:   Mon Mar 11 2024 21:28:52

// ⚠️ If a secondary is offline longer than the oplog window,
//    it CAN'T catch up → needs FULL RESYNC (hours of downtime!)
//    Rule: Size your oplog for at least 24-72 hours of operations.
```

---

### Elections — How a New Primary Is Chosen

```
    When PRIMARY Fails:

    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │   PRIMARY    │    │  SECONDARY   │    │  SECONDARY   │
    │   (down!)    │    │   Member 1   │    │   Member 2   │
    │      ☠️       │    │              │    │              │
    └──────────────┘    └──────┬───────┘    └──────┬───────┘
                               │                   │
                     Heartbeat │ fails!             │
                               │                   │
                          ┌────▼────────────────────▼────┐
                          │        ELECTION               │
                          │                               │
                          │  1. Detect primary is down    │
                          │     (no heartbeat for 10s)    │
                          │                               │
                          │  2. Eligible members          │
                          │     nominate themselves       │
                          │                               │
                          │  3. Members VOTE              │
                          │     (need MAJORITY of votes)  │
                          │                               │
                          │  3 members → need 2 votes     │
                          │  5 members → need 3 votes     │
                          │  7 members → need 4 votes     │
                          │                               │
                          │  4. Winner = new PRIMARY      │
                          │     (typically < 12 seconds)  │
                          └───────────────────────────────┘

    Election factors (who wins?):
    ┌────────────────────┬──────────────────────────────────────┐
    │ Factor             │ Details                              │
    ├────────────────────┼──────────────────────────────────────┤
    │ Priority           │ Higher priority preferred (default 1)│
    │ Oplog freshness    │ Most up-to-date data preferred       │
    │ Network partition  │ Must be in the MAJORITY partition    │
    │ Votes              │ Must have votes: 1 (default)         │
    └────────────────────┴──────────────────────────────────────┘
```

### Why Odd Number of Members?

```
    3 members → majority = 2 → can survive 1 failure    ✅
    4 members → majority = 3 → can survive 1 failure    ⚠️ (same as 3!)
    5 members → majority = 3 → can survive 2 failures   ✅
    6 members → majority = 4 → can survive 2 failures   ⚠️ (same as 5!)
    7 members → majority = 4 → can survive 3 failures   ✅

    Even numbers give you NO extra fault tolerance!
    Always use ODD numbers: 3, 5, or 7 members.
    Max voting members: 7 (MongoDB limit).
    Max total members: 50 (non-voting allowed).
```

---

### Read Preference — Where to Read From

```javascript
// Read preference tells the driver WHERE to send read queries

// PRIMARY (default) — always read from primary
// Strongest consistency, but primary handles all load
db.collection.find().readPref("primary")

// PRIMARY_PREFERRED — prefer primary, fall back to secondary
db.collection.find().readPref("primaryPreferred")

// SECONDARY — always read from a secondary
// ⚠️ Data might be slightly stale (replication lag)
db.collection.find().readPref("secondary")

// SECONDARY_PREFERRED — prefer secondary, fall back to primary
// Great for offloading read traffic from primary
db.collection.find().readPref("secondaryPreferred")

// NEAREST — read from the member with lowest network latency
// Great for geographically distributed replica sets
db.collection.find().readPref("nearest")
```

```
    Read Preference Decision Guide:

    ┌─────────────────────────────┬────────────────────────────────────┐
    │ Read Preference             │ Use When                          │
    ├─────────────────────────────┼────────────────────────────────────┤
    │ primary                     │ Must have latest data (default)   │
    │ primaryPreferred            │ Want latest, but OK with stale    │
    │                             │ if primary is down                │
    │ secondary                   │ Analytics, reporting, dashboards  │
    │                             │ (stale data is OK)                │
    │ secondaryPreferred          │ Offload reads, tolerate staleness │
    │ nearest                     │ Multi-region, minimize latency    │
    └─────────────────────────────┴────────────────────────────────────┘

    ⚠️ Replication lag is typically < 1 second,
       but CAN be minutes under heavy write load.
```

---

### Write Concern — How Durable Are Your Writes?

```javascript
// Write concern = "How many members must acknowledge the write before success?"

// w: 1 (default) — acknowledged by PRIMARY only
// Fastest, but data could be lost if primary dies before replication
db.orders.insertOne(
  { item: "laptop", qty: 1 },
  { writeConcern: { w: 1 } }
)

// w: "majority" — acknowledged by majority of members
// RECOMMENDED for important data
db.orders.insertOne(
  { item: "laptop", qty: 1 },
  { writeConcern: { w: "majority" } }
)

// w: 0 — "fire and forget" — no acknowledgment at all!
// Fastest possible, but you don't know if it succeeded
db.logs.insertOne(
  { event: "page_view" },
  { writeConcern: { w: 0 } }
)

// w: <number> — acknowledged by exactly N members
db.orders.insertOne(
  { item: "laptop" },
  { writeConcern: { w: 3 } }  // Wait for all 3 members
)

// j: true — write must be committed to journal (on-disk) before acknowledging
db.orders.insertOne(
  { item: "laptop" },
  { writeConcern: { w: "majority", j: true } }  // Safest possible!
)

// wtimeout — max time to wait for write concern (in ms)
db.orders.insertOne(
  { item: "laptop" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }  // 5 second timeout
)
```

```
    Write Concern Spectrum:

    FAST ◄──────────────────────────────────────────────► SAFE
    
    w: 0          w: 1           w: "majority"      w: all members
    No ack        Primary ack    Majority ack        Everyone ack
    (fire&forget) (default)      (recommended)       (slowest)
    
    Data loss     Primary dies   Very unlikely       Nearly
    very likely   before repl    data loss           impossible
    if crash      = data lost                        data loss

    ┌────────────────────────────────────────────────────────────┐
    │  💡 Production recommendation:                             │
    │     w: "majority", j: true                                 │
    │     (majority acknowledged + journaled)                    │
    │                                                            │
    │  Set at the database level:                                │
    │  db.adminCommand({                                         │
    │    setDefaultRWConcern: 1,                                 │
    │    defaultWriteConcern: { w: "majority", j: true }         │
    │  })                                                        │
    └────────────────────────────────────────────────────────────┘
```

---

### Read Concern — How Fresh Is Your Read?

```javascript
// Read concern = "What version of data am I guaranteed to see?"

// "local" (default) — returns the latest data on this node
// Might read data that hasn't been replicated (could be rolled back!)
db.orders.find().readConcern("local")

// "available" — like "local" but doesn't wait for anything
// Fastest possible read
db.orders.find().readConcern("available")

// "majority" — returns data acknowledged by majority
// Data will NOT be rolled back
db.orders.find().readConcern("majority")

// "snapshot" — used in multi-document transactions
// Provides a consistent point-in-time snapshot
session.startTransaction({ readConcern: { level: "snapshot" } })

// "linearizable" — strongest guarantee
// Reflects ALL writes that completed before the read started
// ⚠️ Very slow — must confirm with majority
db.orders.find({ _id: orderId }).readConcern("linearizable")
```

```
    Read Concern Comparison:

    ┌──────────────────┬────────┬─────────────┬──────────────────────────┐
    │ Read Concern     │ Speed  │ Consistency │ Rollback Risk?           │
    ├──────────────────┼────────┼─────────────┼──────────────────────────┤
    │ local (default)  │ ⚡ Fast│ Eventual    │ ⚠️ Yes (can read         │
    │                  │        │             │    unreplicated data)     │
    │ available        │ ⚡⚡   │ Weakest     │ ⚠️ Yes                    │
    │ majority         │ 🟡 Med│ Strong      │ ✅ No                     │
    │ snapshot         │ 🟡 Med│ Point-in-   │ ✅ No (transactions)      │
    │                  │        │ time        │                          │
    │ linearizable     │ 🐌 Slow│ Strongest  │ ✅ No (sequential reads)  │
    └──────────────────┴────────┴─────────────┴──────────────────────────┘
```

---

## 🏗️ Part 2: Sharding — Horizontal Scaling

### Why Shard?

```
    Single Server Limits:
    ┌─────────────────────────────────────┐
    │  Storage:    Limited by disk size   │
    │  RAM:        Limited by server RAM  │
    │  CPU:        Limited by cores       │
    │  Throughput:  Limited by I/O        │
    │                                     │
    │  At some point, you CAN'T buy a     │
    │  bigger server. That's when you     │
    │  go HORIZONTAL — add more servers.  │
    └─────────────────────────────────────┘

    ┌─────────────────┐      ┌─────────────────┐
    │ Vertical Scaling│      │Horizontal Scaling│
    │  (Scale UP)     │      │  (Scale OUT)     │
    │                 │      │                  │
    │  Bigger Server  │      │  More Servers    │
    │  ┌───────────┐  │      │  ┌──┐┌──┐┌──┐   │
    │  │           │  │      │  │  ││  ││  │   │
    │  │  BIGGER   │  │      │  │S1││S2││S3│   │
    │  │  SERVER   │  │      │  │  ││  ││  │   │
    │  │           │  │      │  └──┘└──┘└──┘   │
    │  └───────────┘  │      │                  │
    │  $$$$$$ 💰💰💰   │      │  $$ per server   │
    │  Has a ceiling  │      │  Nearly limitless│
    └─────────────────┘      └─────────────────┘
```

---

### Sharded Cluster Architecture

```
    ┌───────────────────────────────────────────────────────────────┐
    │                    SHARDED CLUSTER                             │
    │                                                               │
    │   Application / Driver                                        │
    │        │                                                      │
    │        ▼                                                      │
    │   ┌─────────────────────────────────────┐                     │
    │   │          mongos (Router)            │                     │
    │   │  Routes queries to correct shard(s) │                     │
    │   │  Does NOT store data                │                     │
    │   └──────┬──────────┬──────────┬────────┘                     │
    │          │          │          │                               │
    │     ┌────▼───┐ ┌────▼───┐ ┌────▼───┐                         │
    │     │ Shard 1│ │ Shard 2│ │ Shard 3│   ← Each shard is a     │
    │     │        │ │        │ │        │     REPLICA SET!          │
    │     │Users   │ │Users   │ │Users   │                         │
    │     │A - G   │ │H - O   │ │P - Z   │   ← Data split by       │
    │     └────────┘ └────────┘ └────────┘     shard key range      │
    │                                                               │
    │   ┌──────────────────────────────────────┐                    │
    │   │     Config Servers (Replica Set)      │                    │
    │   │  Stores metadata:                     │                    │
    │   │  • Which chunks are on which shard    │                    │
    │   │  • Shard key ranges                   │                    │
    │   │  • Cluster configuration              │                    │
    │   └──────────────────────────────────────┘                    │
    └───────────────────────────────────────────────────────────────┘
```

### Components Explained

```
┌──────────────────┬──────────────────────────────────────────────────────┐
│ Component        │ Description                                          │
├──────────────────┼──────────────────────────────────────────────────────┤
│ mongos           │ Query router. App connects here, not to shards.      │
│                  │ Stateless — can run multiple for load balancing.      │
│ Shard            │ Stores a subset of the data. Each shard = replica set│
│ Config Servers   │ 3-member replica set storing cluster metadata.        │
│ Shard Key        │ The field(s) used to distribute data across shards.  │
│ Chunk            │ Contiguous range of shard key values (~128MB default)│
│ Balancer         │ Background process that moves chunks between shards  │
│                  │ to ensure even distribution.                         │
└──────────────────┴──────────────────────────────────────────────────────┘
```

---

### Setting Up Sharding

```javascript
// Step 1: Enable sharding on a database
sh.enableSharding("mydb")

// Step 2: Create an index on the shard key
db.orders.createIndex({ customerId: 1 })

// Step 3: Shard the collection
sh.shardCollection("mydb.orders", { customerId: 1 })

// Step 4: Check sharding status
sh.status()

// Output:
// --- Sharding Status ---
// shards:
//   shard0: server1:27018, server2:27019, server3:27020
//   shard1: server4:27021, server5:27022, server6:27023
//   shard2: server7:27024, server8:27025, server9:27026
// databases:
//   mydb:  partitioned: true
//     orders:
//       shard key: { customerId: 1 }
//       chunks:
//         shard0: 3
//         shard1: 4
//         shard2: 3
```

---

### Shard Key Strategies

#### Range-Based Sharding (Default)

```
    Shard key: { customerId: 1 }  — Range-based

    ┌────────────────────┬───────────────────────┬────────────────────┐
    │     Shard 0        │      Shard 1          │      Shard 2       │
    │  customerId:       │  customerId:          │  customerId:       │
    │  "A000" → "F999"   │  "G000" → "M999"      │  "N000" → "Z999"  │
    └────────────────────┴───────────────────────┴────────────────────┘

    ✅ Range queries are efficient:
       find({ customerId: { $gte: "A", $lt: "D" } })
       → Only hits Shard 0 (targeted query!)

    ❌ Hotspot risk:
       If most new customers start with "A" → Shard 0 gets ALL writes!
```

#### Hashed Sharding

```javascript
// Creates a hash of the shard key value → even distribution
sh.shardCollection("mydb.orders", { customerId: "hashed" })
```

```
    Shard key: { customerId: "hashed" }

    hash("Alice")  = 7291842...  → Shard 2
    hash("Bob")    = 1938274...  → Shard 0
    hash("Charlie")= 5829301...  → Shard 1
    hash("Dave")   = 8472910...  → Shard 2

    ┌────────────────────┬───────────────────────┬────────────────────┐
    │     Shard 0        │      Shard 1          │      Shard 2       │
    │  Bob, Frank, ...   │  Charlie, Eve, ...    │  Alice, Dave, ...  │
    │  (hash range 0-33%)│  (hash range 34-66%)  │  (hash range 67%+)│
    └────────────────────┴───────────────────────┴────────────────────┘

    ✅ Perfectly even distribution (no hotspots!)
    ❌ Range queries become scatter-gather (hit ALL shards)
       find({ customerId: { $gte: "A", $lt: "D" } })
       → Must query ALL shards → merge results → slow!
```

---

### Choosing the Right Shard Key — The Most Critical Decision

```
    ┌────────────────────────────────────────────────────────────────────┐
    │              THE 3 PROPERTIES OF A GOOD SHARD KEY                 │
    │                                                                    │
    │  1. HIGH CARDINALITY                                              │
    │     Many distinct values → data can spread across many chunks      │
    │     ✅ Good: email, orderId, customerId, UUID                     │
    │     ❌ Bad:  status ("active"/"inactive"), country (only ~200)     │
    │                                                                    │
    │  2. LOW FREQUENCY                                                  │
    │     No single value appears excessively often                      │
    │     ✅ Good: orderId (each unique), customerId (few orders each)   │
    │     ❌ Bad:  country (if 80% of users are in USA → one huge chunk) │
    │                                                                    │
    │  3. NON-MONOTONIC (avoids hotspots)                               │
    │     Values don't always increase/decrease                          │
    │     ✅ Good: hash(customerId), UUID                               │
    │     ❌ Bad:  ObjectId, timestamp, autoincrement                   │
    │              (all inserts go to LAST shard!)                       │
    └────────────────────────────────────────────────────────────────────┘
```

### Shard Key Examples — Good vs Bad

```
    ┌──────────────────────┬──────┬────────────────────────────────────┐
    │ Shard Key            │ Good?│ Why                                │
    ├──────────────────────┼──────┼────────────────────────────────────┤
    │ { _id: 1 }           │ ❌   │ ObjectId is monotonic → hotspot    │
    │ { _id: "hashed" }    │ 🟡   │ Even writes, but scatter-gather    │
    │ { email: 1 }         │ ✅   │ High cardinality, good distribution│
    │ { status: 1 }        │ ❌   │ Only 3-5 values → can't split!     │
    │ { country: 1 }       │ ❌   │ Low cardinality, skewed frequency  │
    │ { country:1, id:1 }  │ ✅   │ Compound key → good cardinality    │
    │ { tenantId:1, ts:1 } │ ✅✅ │ Multi-tenant: tenant-targeted reads│
    │ { userId: "hashed" } │ ✅   │ Even writes for user-based data    │
    │ { timestamp: 1 }     │ ❌   │ Monotonic → ALL writes to 1 shard  │
    └──────────────────────┴──────┴────────────────────────────────────┘
```

### ⚠️ Shard Key Is (Almost) Immutable!

```
    BEFORE MongoDB 5.0:
    ❌ You CANNOT change the shard key after sharding a collection.
       If you chose badly → drop collection and re-shard!

    MongoDB 5.0+:
    ✅ You CAN refine the shard key (add suffix fields)
       { customerId: 1 } → { customerId: 1, orderId: 1 }
       (Can only ADD fields, not change existing ones)

    MongoDB 7.0+:
    ✅ You CAN reshard a collection with a completely new shard key
       (Online resharding — but it's resource-intensive!)

    💡 Choose your shard key VERY carefully from the start.
```

---

### Chunks and the Balancer

```
    How Data Is Organized:

    Collection with shard key: { customerId: 1 }

    ┌──────────────────────────────────────────────────────────┐
    │                        CHUNKS                            │
    │                                                          │
    │  Chunk 1: customerId [MinKey → "C999"]  → Shard 0       │
    │  Chunk 2: customerId ["D000" → "G999"]  → Shard 0       │
    │  Chunk 3: customerId ["H000" → "L999"]  → Shard 1       │
    │  Chunk 4: customerId ["M000" → "Q999"]  → Shard 1       │
    │  Chunk 5: customerId ["R000" → "V999"]  → Shard 2       │
    │  Chunk 6: customerId ["W000" → MaxKey]  → Shard 2       │
    │                                                          │
    │  Default chunk size: 128 MB (configurable: 1-1024 MB)    │
    └──────────────────────────────────────────────────────────┘

    When a chunk exceeds 128 MB → it SPLITS into two chunks:

    Chunk 2: ["D000" → "G999"] (150 MB — too big!)
        ↓ SPLIT
    Chunk 2a: ["D000" → "E999"] (75 MB)
    Chunk 2b: ["F000" → "G999"] (75 MB)

    If shards become unbalanced:
    Shard 0: 10 chunks   ← Too many!
    Shard 1: 4 chunks
    Shard 2: 4 chunks

    Balancer kicks in → moves chunks from Shard 0 to others:
    Shard 0: 6 chunks   ← Balanced!
    Shard 1: 6 chunks
    Shard 2: 6 chunks
```

```javascript
// Check balancer status
sh.getBalancerState()          // true/false
sh.isBalancerRunning()         // Is it currently migrating?

// Enable/disable balancer
sh.startBalancer()
sh.stopBalancer()

// Set balancer window (only run during off-peak hours)
db.settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "02:00", stop: "06:00" } } },
  { upsert: true }
)

// Change chunk size (in MB)
db.settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 64 } },   // Smaller chunks = more even distribution
  { upsert: true }
)
```

---

### Zones (Tag-Aware Sharding)

> Assign **ranges of shard key values** to specific shards. Perfect for **data locality** and **compliance**.

```javascript
// Scenario: GDPR requires European user data stays in EU data centers

// Step 1: Tag shards with zones
sh.addShardTag("shard-eu-1", "EU")
sh.addShardTag("shard-eu-2", "EU")
sh.addShardTag("shard-us-1", "US")
sh.addShardTag("shard-us-2", "US")

// Step 2: Shard key must include the zone discriminator
sh.shardCollection("mydb.users", { region: 1, email: 1 })

// Step 3: Define zone ranges
sh.addTagRange(
  "mydb.users",
  { region: "EU", email: MinKey },   // From
  { region: "EU", email: MaxKey },   // To
  "EU"                                // Zone tag
)

sh.addTagRange(
  "mydb.users",
  { region: "US", email: MinKey },
  { region: "US", email: MaxKey },
  "US"
)

// Result:
// All documents with region: "EU" → stored on shard-eu-1 or shard-eu-2
// All documents with region: "US" → stored on shard-us-1 or shard-us-2
// GDPR compliance achieved! ✅
```

---

### Query Routing in a Sharded Cluster

```
    ┌───────────────────────────────────────────────────────────────┐
    │            How mongos Routes Queries                          │
    │                                                               │
    │  TARGETED QUERY (Best — hits only 1 shard):                   │
    │  ┌─────────────────────────────────────────┐                  │
    │  │ db.orders.find({ customerId: "C123" })  │                  │
    │  │                                          │                  │
    │  │ Shard key: { customerId: 1 }            │                  │
    │  │ mongos knows: C123 is on Shard 0        │                  │
    │  │ → Routes ONLY to Shard 0                │                  │
    │  │ → Fast! ⚡                               │                  │
    │  └─────────────────────────────────────────┘                  │
    │                                                               │
    │  SCATTER-GATHER QUERY (Slow — hits ALL shards):              │
    │  ┌─────────────────────────────────────────┐                  │
    │  │ db.orders.find({ status: "shipped" })   │                  │
    │  │                                          │                  │
    │  │ "status" is NOT the shard key            │                  │
    │  │ mongos doesn't know which shard has it   │                  │
    │  │ → Sends query to ALL shards              │                  │
    │  │ → Merges results                         │                  │
    │  │ → Slow! 🐌 (especially with many shards) │                  │
    │  └─────────────────────────────────────────┘                  │
    │                                                               │
    │  💡 Design queries to include the shard key whenever possible! │
    └───────────────────────────────────────────────────────────────┘
```

---

## 📊 Replication vs Sharding — When to Use What

```
┌──────────────────┬─────────────────────────┬────────────────────────────┐
│                  │ Replication (Replica Set)│ Sharding (Sharded Cluster) │
├──────────────────┼─────────────────────────┼────────────────────────────┤
│ Purpose          │ High availability       │ Horizontal scaling         │
│ Data copies      │ Same data on all members│ Different data per shard   │
│ Writes           │ All go to ONE primary   │ Distributed across shards  │
│ Reads            │ Can spread to secondaries│ Distributed across shards │
│ Scaling          │ Read scaling            │ Read AND write scaling     │
│ Complexity       │ Low                     │ High                       │
│ When to use      │ Always (even without    │ When one server can't      │
│                  │ sharding)               │ handle the load/data       │
│ Typical setup    │ 3 members              │ 3+ shards × 3 replicas     │
│ Storage capacity │ Limited by 1 server     │ Sum of all shards          │
└──────────────────┴─────────────────────────┴────────────────────────────┘

    Real-World Architecture:

    Small app:     1 Replica Set (3 members)
    Medium app:    1 Replica Set (3-5 members) + read from secondaries
    Large app:     Sharded cluster (3 shards × 3 replicas = 9+ servers)
    Massive app:   Sharded cluster (100+ shards × 3 replicas = 300+ servers)
```

---

## 💡 Production Best Practices

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │             REPLICATION & SHARDING CHECKLIST                        │
    │                                                                     │
    │  Replica Sets:                                                      │
    │  □ Use 3 or 5 members (odd number!)                                │
    │  □ Members across different availability zones / data centers       │
    │  □ Set write concern to "majority" for important data              │
    │  □ Size oplog for at least 24-72 hours of operations               │
    │  □ Monitor replication lag (should be < 1 second normally)          │
    │  □ Test failover regularly (rs.stepDown())                         │
    │  □ Never put primary + all secondaries in same rack/zone           │
    │                                                                     │
    │  Sharding:                                                          │
    │  □ Only shard when you NEED to (adds complexity!)                   │
    │  □ Choose shard key carefully (high cardinality, non-monotonic)     │
    │  □ Design queries to include the shard key (targeted queries)       │
    │  □ Monitor chunk distribution (sh.status())                        │
    │  □ Set balancer window to off-peak hours                           │
    │  □ Each shard should be a 3-member replica set                     │
    │  □ Run 2+ mongos routers behind a load balancer                    │
    │  □ Config servers: always 3-member replica set                     │
    └─────────────────────────────────────────────────────────────────────┘
```

---

## 🧭 What's Next?

Your data is replicated and sharded. Now let's learn how to keep it **safe and consistent** with transactions and security:

**Next Chapter → [3B.8 MongoDB Transactions & Security](./08-MongoDB-Transactions-Security.md)** 🔴

> _"Replication gives you availability. Sharding gives you scale. Together, they give you a system that never sleeps."_

---

[← Previous: MongoDB Schema Design Patterns](./06-MongoDB-Schema-Design.md) | [Index](../INDEX.md) | [Next: MongoDB Transactions & Security →](./08-MongoDB-Transactions-Security.md)
