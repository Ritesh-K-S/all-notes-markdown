# Column-Family Stores (Cassandra, HBase)

> **What you'll learn**: How column-family databases store data in wide columns optimized for massive write throughput, how Cassandra achieves planet-scale distribution with no single point of failure, and when to choose them over relational or document databases.

---

## Real-Life Analogy

Imagine you're tracking **weather data** for every city in the world, every hour, for 50 years.

With a **traditional filing cabinet** (relational DB):
- One drawer per city → each drawer has millions of hourly records → finding "all temperatures for Mumbai in January 2024" means opening the drawer and flipping through thousands of pages.

With a **wide spreadsheet system** (column-family store):
- Each city gets a **row**, and each hour is a **column**: `Mumbai | 2024-01-01T00:00: 22°C | 2024-01-01T01:00: 21°C | ...`
- You can have **millions of columns per row** (one per hour for 50 years = 438,000 columns)
- Reading a time range = reading adjacent columns (super fast!)
- Adding new data = just append a new column (never modify existing data)

This is perfect for **time-series data, event logs, and IoT sensor data** where you write constantly and read by time ranges.

---

## Core Concept Explained Step-by-Step

### What is a Column-Family Store?

Unlike relational databases (row-oriented), column-family stores organize data by **columns grouped into families**:

```
ROW-ORIENTED (Traditional SQL):
┌────────────────────────────────────────────────┐
│ Row 1: | id: 1 | name: Alice | age: 28 | city: Mumbai |
│ Row 2: | id: 2 | name: Bob   | age: 35 | city: Delhi  |
│ Row 3: | id: 3 | name: Carol | age: 22 | city: Pune   |
└────────────────────────────────────────────────┘
All columns stored together per row → great for SELECT *

COLUMN-FAMILY (Cassandra):
┌─────────────────────────────────────────────────────────────────┐
│ Row Key: "sensor_42"                                             │
│   Column Family: "readings"                                     │
│     │ 2024-01-01T00:00 │ 2024-01-01T01:00 │ 2024-01-01T02:00 │
│     │     22.5°C       │     21.8°C       │     21.2°C       │
│                                                                  │
│ Row Key: "sensor_43"                                             │
│   Column Family: "readings"                                     │
│     │ 2024-01-01T00:00 │ 2024-01-01T00:30 │ 2024-01-01T01:00 │
│     │     19.1°C       │     18.9°C       │     19.3°C       │
└─────────────────────────────────────────────────────────────────┘
Different rows can have DIFFERENT columns!
```

### Cassandra Data Model

```
KEYSPACE (like a database)
└── TABLE (like a table, but distributed)
    ├── Partition Key → determines which node stores the data
    ├── Clustering Key → determines sort order within a partition
    └── Regular Columns → the actual data

Example: Messages table
┌────────────────────────────────────────────────────────────────────┐
│ Partition Key: chat_id    Clustering Key: message_time             │
├────────────────────────────────────────────────────────────────────┤
│ chat_id = "alice_bob"                                              │
│   ├── 2024-01-15T10:00 | sender: Alice | text: "Hey!"            │
│   ├── 2024-01-15T10:01 | sender: Bob   | text: "Hi there!"       │
│   └── 2024-01-15T10:02 | sender: Alice | text: "How are you?"    │
│                                                                    │
│ chat_id = "alice_carol"                                            │
│   ├── 2024-01-15T09:00 | sender: Carol | text: "Meeting at 3?"   │
│   └── 2024-01-15T09:05 | sender: Alice | text: "Sure!"           │
└────────────────────────────────────────────────────────────────────┘
```

### Why Cassandra is Different from SQL

```
SQL DATABASE:                              CASSANDRA:
┌─────────────────────────┐               ┌─────────────────────────────┐
│ Single Master           │               │ No Master (Peer-to-Peer)    │
│ (Single point of failure)│              │ (Every node is equal)       │
├─────────────────────────┤               ├─────────────────────────────┤
│ Read-optimized          │               │ Write-optimized             │
│ (Indexes, B-Trees)      │               │ (Append-only, LSM-Trees)   │
├─────────────────────────┤               ├─────────────────────────────┤
│ Strong consistency      │               │ Tunable consistency         │
│ (ACID always)           │               │ (Choose per query)          │
├─────────────────────────┤               ├─────────────────────────────┤
│ Vertical scaling        │               │ Linear horizontal scaling   │
│ (Bigger server)         │               │ (Add more nodes)            │
├─────────────────────────┤               ├─────────────────────────────┤
│ Schema enforced         │               │ Schema defined but flexible │
└─────────────────────────┘               └─────────────────────────────┘
```

---

## How It Works Internally

### Cassandra Ring Architecture

```
                    ┌───────────────────┐
                    │    Node A         │
                    │  Tokens: 0-255   │
                    └────────┬──────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
   ┌──────────▼───┐    ┌────▼────┐    ┌───▼──────────┐
   │   Node F     │    │  Client │    │   Node B     │
   │ Tokens:      │    │         │    │ Tokens:      │
   │ 1280-1535    │    └─────────┘    │ 256-511      │
   └──────────┬───┘                    └───┬──────────┘
              │                            │
   ┌──────────▼───┐                    ┌───▼──────────┐
   │   Node E     │                    │   Node C     │
   │ Tokens:      │                    │ Tokens:      │
   │ 1024-1279    │                    │ 512-767      │
   └──────────┬───┘                    └───┬──────────┘
              │                            │
              └──────────────┬─────────────┘
                             │
                    ┌────────▼──────────┐
                    │    Node D         │
                    │ Tokens: 768-1023  │
                    └───────────────────┘

Data placement: hash(partition_key) → token → node
Replication: Data copied to N next nodes in the ring (RF=3 means 3 copies)
```

### Write Path (LSM-Tree)

```
WRITE REQUEST
      │
      ▼
┌─────────────────┐
│ 1. Commit Log   │ ← Written FIRST (for durability)
│    (on disk)    │    Sequential write = FAST
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. Memtable     │ ← In-memory sorted structure
│    (in RAM)     │    Sorted by partition key + clustering key
└────────┬────────┘
         │ When Memtable is full...
         ▼
┌─────────────────┐
│ 3. SSTable      │ ← Flushed to disk (immutable!)
│    (on disk)    │    Sorted Strings Table
└────────┬────────┘
         │ Background process...
         ▼
┌─────────────────┐
│ 4. Compaction   │ ← Merges SSTables, removes tombstones
│    (background) │    Reclaims space from deleted data
└─────────────────┘

Why writes are fast:
- Commit log: Sequential append (no random I/O)
- Memtable: In-memory (RAM speed)
- No "update in place" — always append new data
- Total write latency: ~1-2ms
```

### Read Path

```
READ REQUEST (partition_key + clustering_key)
      │
      ▼
┌─────────────────┐
│ 1. Memtable     │ ← Check in-memory first (newest data)
└────────┬────────┘
         │ Not found? ↓
         ▼
┌─────────────────┐
│ 2. Row Cache    │ ← Recently read rows (if enabled)
└────────┬────────┘
         │ Not found? ↓
         ▼
┌─────────────────┐
│ 3. Bloom Filter │ ← Quickly tells "definitely NOT in this SSTable"
│   (per SSTable) │    Avoids unnecessary disk reads
└────────┬────────┘
         │ Maybe here? ↓
         ▼
┌─────────────────┐
│ 4. Partition    │ ← Index to find exact position in SSTable
│    Index        │
└────────┬────────┘
         │ Found position ↓
         ▼
┌─────────────────┐
│ 5. SSTable      │ ← Read from disk (or OS page cache)
│    (disk)       │
└─────────────────┘
```

### Consistency Levels

```
┌───────────────────────────────────────────────────────────────────┐
│  CASSANDRA CONSISTENCY LEVELS (Replication Factor = 3)            │
├──────────────┬────────────────────────────────────────────────────┤
│ Level        │ What it means                                      │
├──────────────┼────────────────────────────────────────────────────┤
│ ONE          │ Acknowledge after 1 replica responds               │
│              │ Fastest, but might read stale data                 │
├──────────────┼────────────────────────────────────────────────────┤
│ QUORUM       │ Acknowledge after majority (2/3) responds          │
│              │ Good balance of speed + consistency                │
├──────────────┼────────────────────────────────────────────────────┤
│ ALL          │ Acknowledge after ALL replicas respond             │
│              │ Slowest, but guaranteed consistency                │
│              │ If 1 node is down, write FAILS                    │
├──────────────┼────────────────────────────────────────────────────┤
│ LOCAL_QUORUM │ Majority in LOCAL datacenter only                  │
│              │ Best for multi-datacenter deployments              │
└──────────────┴────────────────────────────────────────────────────┘

Strong consistency formula:
  READ_CL + WRITE_CL > Replication Factor
  Example: QUORUM(2) + QUORUM(2) > RF(3) ✅ = Strong consistency
```

---

## Code Examples

### Python (cassandra-driver)

```python
from cassandra.cluster import Cluster
from cassandra.query import SimpleStatement, ConsistencyLevel
from cassandra import WriteTimeout
from datetime import datetime, timedelta
import uuid

# Connect to Cassandra cluster
cluster = Cluster(['node1', 'node2', 'node3'])
session = cluster.connect()

# Create keyspace with replication strategy
session.execute("""
    CREATE KEYSPACE IF NOT EXISTS messaging
    WITH replication = {
        'class': 'NetworkTopologyStrategy',
        'datacenter1': 3,
        'datacenter2': 3
    }
""")
session.set_keyspace('messaging')

# Create table optimized for "get messages in a chat"
session.execute("""
    CREATE TABLE IF NOT EXISTS messages (
        chat_id TEXT,
        message_time TIMESTAMP,
        message_id UUID,
        sender TEXT,
        content TEXT,
        PRIMARY KEY (chat_id, message_time, message_id)
    ) WITH CLUSTERING ORDER BY (message_time DESC)
""")

# INSERT message (write-optimized, append-only)
session.execute("""
    INSERT INTO messages (chat_id, message_time, message_id, sender, content)
    VALUES (%s, %s, %s, %s, %s)
""", ("alice_bob", datetime.now(), uuid.uuid4(), "Alice", "Hey, how are you?"))

# QUERY with consistency level
stmt = SimpleStatement(
    "SELECT sender, content, message_time FROM messages WHERE chat_id = %s LIMIT 50",
    consistency_level=ConsistencyLevel.LOCAL_QUORUM
)
rows = session.execute(stmt, ("alice_bob",))
for row in rows:
    print(f"[{row.message_time}] {row.sender}: {row.content}")

# TIME-RANGE QUERY (super efficient due to clustering key!)
stmt = SimpleStatement("""
    SELECT * FROM messages 
    WHERE chat_id = %s 
    AND message_time >= %s 
    AND message_time <= %s
""")
yesterday = datetime.now() - timedelta(days=1)
rows = session.execute(stmt, ("alice_bob", yesterday, datetime.now()))

# BATCH for atomic writes within same partition
from cassandra.query import BatchStatement
batch = BatchStatement()
batch.add(SimpleStatement(
    "INSERT INTO messages (chat_id, message_time, message_id, sender, content) VALUES (%s, %s, %s, %s, %s)"),
    ("group_chat", datetime.now(), uuid.uuid4(), "Alice", "Meeting at 3"))
batch.add(SimpleStatement(
    "UPDATE chat_metadata SET last_message = %s WHERE chat_id = %s"),
    (datetime.now(), "group_chat"))
session.execute(batch)
```

### Java (DataStax Driver)

```java
import com.datastax.oss.driver.api.core.CqlSession;
import com.datastax.oss.driver.api.core.cql.*;
import com.datastax.oss.driver.api.core.DefaultConsistencyLevel;
import java.time.Instant;
import java.util.UUID;

public class CassandraExample {
    public static void main(String[] args) {
        try (CqlSession session = CqlSession.builder()
                .addContactPoint(new InetSocketAddress("localhost", 9042))
                .withLocalDatacenter("datacenter1")
                .withKeyspace("messaging")
                .build()) {

            // Prepared statement (reusable, efficient)
            PreparedStatement insertMsg = session.prepare(
                "INSERT INTO messages (chat_id, message_time, message_id, sender, content) " +
                "VALUES (?, ?, ?, ?, ?)");

            // Insert with QUORUM consistency
            BoundStatement bound = insertMsg.bind(
                "alice_bob",
                Instant.now(),
                UUID.randomUUID(),
                "Bob",
                "Hi Alice! I'm good, thanks!"
            ).setConsistencyLevel(DefaultConsistencyLevel.LOCAL_QUORUM);

            session.execute(bound);

            // Query recent messages
            ResultSet rs = session.execute(
                SimpleStatement.newInstance(
                    "SELECT sender, content, message_time FROM messages " +
                    "WHERE chat_id = ? LIMIT 20",
                    "alice_bob"
                ).setConsistencyLevel(DefaultConsistencyLevel.ONE)  // Fast reads
            );

            for (Row row : rs) {
                System.out.printf("[%s] %s: %s%n",
                    row.getInstant("message_time"),
                    row.getString("sender"),
                    row.getString("content"));
            }
        }
    }
}
```

---

## Infrastructure Examples

### Cassandra Cluster (Docker Compose)

```yaml
version: '3.8'
services:
  cassandra-seed:
    image: cassandra:4.1
    environment:
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_DC=dc1
      - CASSANDRA_RACK=rack1
      - HEAP_NEWSIZE=256M
      - MAX_HEAP_SIZE=1G
    ports:
      - "9042:9042"
    volumes:
      - cassandra-seed-data:/var/lib/cassandra

  cassandra-node2:
    image: cassandra:4.1
    environment:
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_SEEDS=cassandra-seed
      - CASSANDRA_DC=dc1
    depends_on:
      - cassandra-seed
    volumes:
      - cassandra-node2-data:/var/lib/cassandra

  cassandra-node3:
    image: cassandra:4.1
    environment:
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_SEEDS=cassandra-seed
      - CASSANDRA_DC=dc1
    depends_on:
      - cassandra-seed
    volumes:
      - cassandra-node3-data:/var/lib/cassandra

volumes:
  cassandra-seed-data:
  cassandra-node2-data:
  cassandra-node3-data:
```

### Production Configuration (cassandra.yaml highlights)

```yaml
cluster_name: 'ProductionCluster'
num_tokens: 256
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "10.0.1.1,10.0.1.2"

# Performance tuning
concurrent_reads: 32
concurrent_writes: 64
memtable_allocation_type: offheap_objects
memtable_flush_writers: 4
compaction_throughput_mb_per_sec: 64

# Consistency
endpoint_snitch: GossipingPropertyFileSnitch
```

---

## Real-World Example

### Discord — Cassandra for Message Storage

Discord stores **billions of messages** in Cassandra:

```
┌─────────────────────────────────────────────────────────────────┐
│              Discord Message Storage Architecture                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Table Design:                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ PRIMARY KEY: (channel_id, bucket) + message_id (clustering)│ │
│  │                                                            │ │
│  │ Bucket = message_id / 10 days of messages                 │ │
│  │ (Prevents partitions from growing unbounded)              │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  Access Pattern:                                                 │
│  "Load last 50 messages in channel X"                           │
│  → Single partition read, reverse clustering order              │
│  → Sub-10ms latency for any channel                            │
│                                                                   │
│  Scale:                                                          │
│  • 1 Trillion messages stored                                    │
│  • Millions of messages per second written                       │
│  • Cassandra cluster: 177 nodes                                  │
│  • Data: ~10 PB                                                  │
│                                                                   │
│  Challenge: Hot partitions (popular channels)                    │
│  Solution: Moved hottest data to ScyllaDB (faster Cassandra)    │
└─────────────────────────────────────────────────────────────────┘
```

### Netflix — Cassandra for Everything

- **Viewing history**: 500+ billion rows
- **Bookmarks**: Where you paused (per user per show)
- **A/B test data**: Massive write throughput for experiment events
- **30 PB** across thousands of Cassandra nodes globally

---

## Cassandra vs HBase

| Feature | Cassandra | HBase |
|---------|-----------|-------|
| **Architecture** | Peer-to-peer (no master) | Master-slave (HMaster) |
| **Consistency** | Tunable (AP by default) | Strong (CP) |
| **Write Speed** | Excellent | Good |
| **Read Speed** | Good | Excellent (with BlockCache) |
| **Ecosystem** | Standalone | Requires Hadoop/HDFS |
| **Operations** | Simpler (no ZooKeeper) | Complex (ZK + HDFS + HBase) |
| **Best For** | Write-heavy, globally distributed | Analytics on Hadoop ecosystem |
| **Used By** | Netflix, Discord, Instagram | Facebook (Messages), Yahoo |

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using Cassandra like SQL | No JOINs, no subqueries, no GROUP BY | Design tables around queries (denormalize) |
| Wrong partition key | Huge partitions (>100MB) or hot spots | Choose high-cardinality keys |
| Too many tombstones | Deleted data still occupies space until compaction | Use TTL, avoid frequent deletes |
| Secondary indexes on high-cardinality columns | Scatter-gather across all nodes | Denormalize or use materialized views |
| Reading before writing | Cassandra is designed for "write without reading" | Use UPSERT patterns, not read-modify-write |
| Not setting TTL | Data grows forever | Always set TTL for time-series data |
| Small clusters with RF=3 | All data on all nodes (no benefit) | Need at least 3+ nodes per DC |

---

## When to Use / When NOT to Use

### ✅ Use Column-Family Stores When:
- **Write throughput** is the primary concern (100K+ writes/sec)
- Data is **time-series** or **event-based** (logs, metrics, IoT)
- You need **multi-datacenter** replication with no downtime
- **Availability** matters more than strong consistency
- Data has a **natural partition key** (user_id, device_id, chat_id)
- You're storing **immutable append-only data**
- Scale is massive (TBs to PBs of data)

### ❌ Do NOT Use Column-Family Stores When:
- You need **complex queries** (JOINs, ad-hoc aggregations)
- **Strong consistency** is required for every operation
- Data model requires **frequent updates** to the same rows
- You need **transactions** across multiple partitions
- Dataset is small (< 100GB) — overhead isn't worth it
- Access patterns require **full table scans** (use Spark/SQL instead)
- Your team isn't ready for the **data modeling discipline** required

---

## Key Takeaways

1. **Column-family stores** (Cassandra, HBase) are optimized for **massive write throughput** and **time-range reads** on distributed clusters.
2. **Cassandra's ring architecture** has no single point of failure — every node is equal, and the system survives node failures gracefully.
3. **LSM-Trees** make writes fast by appending to memory first, then flushing sorted data to disk — no random I/O on writes.
4. **Design tables around your queries** — in Cassandra, you create one table per query pattern (denormalization is expected).
5. **Tunable consistency** lets you choose per-query: fast-but-stale (ONE) vs strong-but-slower (QUORUM/ALL).
6. **Partition key choice** is the single most important modeling decision — it determines data distribution and query performance.
7. **Cassandra excels at**: messaging, IoT, time-series, user activity feeds, and any write-heavy workload that needs global distribution.

---

## What's Next?

Next, we'll explore **Graph Databases (Neo4j, Amazon Neptune)** (Chapter 9.6), databases designed for data with complex, interconnected relationships like social networks, recommendation engines, and fraud detection.
