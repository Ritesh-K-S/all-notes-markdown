# Distributed File Systems (HDFS, GFS)

> **What you'll learn**: How distributed file systems like Google File System (GFS) and Hadoop Distributed File System (HDFS) store petabytes of data across thousands of commodity machines, providing fault tolerance, massive throughput, and the foundation for big data processing.

---

## Real-Life Analogy

Imagine you're running a **library so enormous** that no single building can hold all the books. So you do something clever:

1. You build **hundreds of small library branches** across the city (commodity servers)
2. Each book is **torn into chapters** (split into blocks/chunks)
3. Each chapter is **photocopied 3 times** and stored in **3 different branches** (replication)
4. There's ONE **central catalog office** (master/NameNode) that knows exactly which chapters are in which branch
5. When someone wants a book, they ask the catalog office → get the list of branches → go directly to those branches to pick up chapters in parallel

If one branch burns down? No problem — every chapter has copies in other branches. The catalog office notices the loss and tells other branches to make new copies.

That's a distributed file system. It's designed for **HUGE files** (GBs to TBs), **sequential reads** (reading entire datasets), and **surviving hardware failures** as a daily occurrence.

---

## Core Concept Explained Step-by-Step

### Why Do We Need Distributed File Systems?

```
THE PROBLEM:
─────────────────────────────────────────────────────
You have 10 PETABYTES of data (10,000 TB).

Single server capacity:   ~16 TB (large disk)
Single server bandwidth:  ~500 MB/s read speed

To read 10 PB at 500 MB/s = 231 DAYS

DISTRIBUTED SOLUTION:
─────────────────────────────────────────────────────
Spread data across 1,000 servers.

Each server stores:       ~10 TB
Read in PARALLEL:         1,000 × 500 MB/s = 500 GB/s

To read 10 PB at 500 GB/s = ~5.5 HOURS

That's 1,000x faster! Plus:
  ✓ No single point of storage failure
  ✓ Data survives disk/machine crashes
  ✓ Can grow by adding more machines
```

### The Architecture (Master-Worker Pattern)

```
┌─────────────────────────────────────────────────────────────────┐
│              DISTRIBUTED FILE SYSTEM ARCHITECTURE                │
│                                                                 │
│          ┌─────────────────────────────────┐                   │
│          │       MASTER / NAME NODE        │                   │
│          │                                 │                   │
│          │  • File → Chunk mapping         │                   │
│          │  • Chunk → Server mapping       │                   │
│          │  • Namespace (directory tree)   │                   │
│          │  • Replication management       │                   │
│          │  • Heartbeat monitoring         │                   │
│          └────────────────┬────────────────┘                   │
│                           │                                     │
│              Metadata     │    Data transfers happen            │
│              operations   │    DIRECTLY between client          │
│              only         │    and data nodes (not              │
│                           │    through master!)                  │
│                           │                                     │
│  ┌────────────────────────┼────────────────────────────┐       │
│  │                        │                            │       │
│  ▼                        ▼                            ▼       │
│ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│ │  DATA NODE 1 │  │  DATA NODE 2 │  │  DATA NODE 3 │  ...    │
│ │              │  │              │  │              │         │
│ │ ┌──┐┌──┐┌──┐│  │ ┌──┐┌──┐┌──┐│  │ ┌──┐┌──┐┌──┐│         │
│ │ │C1││C4││C7││  │ │C1││C2││C5││  │ │C2││C3││C6││         │
│ │ └──┘└──┘└──┘│  │ └──┘└──┘└──┘│  │ └──┘└──┘└──┘│         │
│ │ ┌──┐┌──┐    │  │ ┌──┐┌──┐    │  │ ┌──┐┌──┐    │         │
│ │ │C3││C9│    │  │ │C8││C9│    │  │ │C4││C7│    │         │
│ │ └──┘└──┘    │  │ └──┘└──┘    │  │ └──┘└──┘    │         │
│ └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                 │
│  C = Chunk (64MB in GFS, 128MB in HDFS)                        │
│  Each chunk replicated to 3 nodes (default)                    │
└─────────────────────────────────────────────────────────────────┘
```

### Key Design Principles

| Principle | Explanation |
|-----------|-------------|
| **Hardware failures are normal** | With 1000s of disks, failures happen DAILY. Design for it. |
| **Files are huge** | Multi-GB files are common. Optimize for large sequential I/O, not small random access. |
| **Write-once, read-many** | Files are appended to, rarely modified in place. Once written, mostly read. |
| **Co-locate computation with data** | Move the program TO the data, not data to the program ("data locality"). |
| **Throughput over latency** | Batch processing cares about GB/s, not ms per request. |

---

## Google File System (GFS) — The Pioneer

### GFS Architecture (2003 Paper)

```
┌──────────────────────────────────────────────────────────────┐
│                    GOOGLE FILE SYSTEM                         │
│                                                              │
│  ┌───────────────────────────────────────┐                  │
│  │           GFS MASTER                   │                  │
│  │                                       │                  │
│  │  Metadata (in RAM for speed):         │                  │
│  │  ┌─────────────────────────────────┐  │                  │
│  │  │ /bigdata/webindex.dat           │  │                  │
│  │  │   Chunk 1 → [Server A, B, D]   │  │                  │
│  │  │   Chunk 2 → [Server C, E, F]   │  │                  │
│  │  │   Chunk 3 → [Server A, C, G]   │  │                  │
│  │  │                                 │  │                  │
│  │  │ /logs/2024/access.log           │  │                  │
│  │  │   Chunk 1 → [Server B, D, F]   │  │                  │
│  │  └─────────────────────────────────┘  │                  │
│  │                                       │                  │
│  │  Operation Log (WAL on disk)          │                  │
│  │  Checkpoints (periodic snapshots)     │                  │
│  └───────────────────────────────────────┘                  │
│                                                              │
│  Chunk Size: 64 MB (large to reduce metadata overhead)       │
│  Replication Factor: 3 (configurable)                        │
│  Lease mechanism for write ordering                          │
└──────────────────────────────────────────────────────────────┘
```

### GFS Read Flow

```
Client                    GFS Master              Chunk Servers
  │                          │                         │
  │ 1. "Read /data/file.dat  │                         │
  │     bytes 100M-200M"     │                         │
  │─────────────────────────▶│                         │
  │                          │                         │
  │ 2. Master calculates:    │                         │
  │    offset 100M ÷ 64MB    │                         │
  │    = chunk index 1        │                         │
  │    (chunk handle: 0x7F2A) │                         │
  │                          │                         │
  │ 3. Returns chunk handle  │                         │
  │    + server locations    │                         │
  │◀─────────────────────────│                         │
  │    "Chunk 0x7F2A is on   │                         │
  │     Server A, B, D"      │                         │
  │                          │                         │
  │ 4. Client contacts NEAREST chunk server            │
  │    (picks Server A)                                │
  │──────────────────────────────────────────────────▶│ Server A
  │    "Give me chunk 0x7F2A, offset 36M, len 64M"    │
  │                          │                         │
  │ 5. Server reads from local disk & returns data     │
  │◀──────────────────────────────────────────────────│
  │                          │                         │
  │ (Client caches chunk locations for future reads)   │
```

### GFS Write Flow (Record Append)

```
Client                    GFS Master    Primary       Secondary Replicas
  │                          │          (Server A)    (Server B, D)
  │                          │              │              │
  │ 1. "I want to write"    │              │              │
  │─────────────────────────▶│              │              │
  │                          │              │              │
  │ 2. Master grants LEASE   │              │              │
  │    to one replica        │              │              │
  │    (makes it Primary)    │              │              │
  │                          │──lease──────▶│              │
  │                          │              │              │
  │ 3. Master returns        │              │              │
  │    primary + secondaries │              │              │
  │◀─────────────────────────│              │              │
  │                          │              │              │
  │ 4. Client pushes data to ALL replicas (pipelined)     │
  │──────────────────────────────data──────▶│              │
  │──────────────────────────────data─────────────────────▶│
  │                          │              │              │
  │ 5. Client sends WRITE request to Primary              │
  │─────────────────────────────────────────▶│              │
  │                          │              │              │
  │ 6. Primary assigns serial number, applies write       │
  │                          │              │              │
  │ 7. Primary forwards write order to secondaries        │
  │                          │              │──────────────▶│
  │                          │              │              │
  │ 8. Secondaries apply same write in same order         │
  │                          │              │◀─────ACK─────│
  │                          │              │              │
  │ 9. Primary confirms to client                         │
  │◀────────────────────────────────────────│              │
  │    "Write successful"    │              │              │
```

---

## Hadoop Distributed File System (HDFS)

### HDFS Architecture

HDFS is the open-source implementation inspired by GFS. It's the storage layer of the **Hadoop ecosystem**.

```
┌────────────────────────────────────────────────────────────────┐
│                        HDFS CLUSTER                             │
│                                                                │
│  ┌────────────────────────────────────────────────────┐       │
│  │                  NAME NODE (Master)                 │       │
│  │                                                    │       │
│  │  Namespace:     /user/alice/data.csv               │       │
│  │  Block Map:     Block B1 → [DN1, DN3, DN5]        │       │
│  │                 Block B2 → [DN2, DN4, DN6]        │       │
│  │  EditLog:       Records every metadata change     │       │
│  │  FsImage:       Periodic checkpoint of namespace  │       │
│  │                                                    │       │
│  │  RAM Usage:     ~150 bytes per block per replica  │       │
│  │  (1M blocks × 150B = 150MB RAM needed)            │       │
│  └─────────────────────────┬──────────────────────────┘       │
│                            │                                   │
│         ┌──────────────────┼──────────────────────┐           │
│         │ Heartbeat (3s)   │  Block Reports (6h)  │           │
│         │                  │                      │           │
│  ┌──────▼──────┐   ┌──────▼──────┐   ┌───────────▼───┐      │
│  │ DataNode 1  │   │ DataNode 2  │   │  DataNode 3   │ ...  │
│  │             │   │             │   │               │      │
│  │ Blocks:     │   │ Blocks:     │   │ Blocks:       │      │
│  │  B1, B4, B7 │   │  B2, B5, B8 │   │  B1, B3, B6   │      │
│  │             │   │             │   │               │      │
│  │ Local disks │   │ Local disks │   │ Local disks   │      │
│  │ (JBOD, no  │   │ (JBOD, no  │   │ (JBOD, no    │      │
│  │  RAID!)     │   │  RAID!)     │   │  RAID!)       │      │
│  └─────────────┘   └─────────────┘   └───────────────┘      │
│                                                                │
│  Block Size: 128 MB (default, configurable)                   │
│  Replication Factor: 3 (default)                              │
│  Rack Awareness: 2 replicas in one rack, 1 in another         │
└────────────────────────────────────────────────────────────────┘
```

### HDFS Block Placement Strategy (Rack Awareness)

```
┌─────── RACK 1 ───────┐     ┌─────── RACK 2 ───────┐
│                       │     │                       │
│  ┌─────────────────┐  │     │  ┌─────────────────┐  │
│  │   DataNode A    │  │     │  │   DataNode C    │  │
│  │                 │  │     │  │                 │  │
│  │  ┌───────────┐  │  │     │  │  ┌───────────┐  │  │
│  │  │ Block B1  │  │  │     │  │  │ Block B1  │  │  │
│  │  │ (replica1)│  │  │     │  │  │ (replica3)│  │  │
│  │  └───────────┘  │  │     │  │  └───────────┘  │  │
│  └─────────────────┘  │     │  └─────────────────┘  │
│                       │     │                       │
│  ┌─────────────────┐  │     │  ┌─────────────────┐  │
│  │   DataNode B    │  │     │  │   DataNode D    │  │
│  │                 │  │     │  │                 │  │
│  │  ┌───────────┐  │  │     │  │                 │  │
│  │  │ Block B1  │  │  │     │  │                 │  │
│  │  │ (replica2)│  │  │     │  │                 │  │
│  │  └───────────┘  │  │     │  │                 │  │
│  └─────────────────┘  │     │  └─────────────────┘  │
│                       │     │                       │
└───────────────────────┘     └───────────────────────┘
         ▲                              ▲
         │                              │
    2 replicas in                  1 replica in
    writer's rack                  different rack
    (fast writes)                  (survives rack failure)
```

**Why this placement?**
- 2 replicas in same rack = fast writes (same-rack bandwidth is 10× inter-rack)
- 1 replica in different rack = survives entire rack failure (power, switch failure)
- Trade-off: write bandwidth vs. fault tolerance

### HDFS NameNode High Availability

```
┌─────────────────────────────────────────────────────────────┐
│              HDFS HA (High Availability) Setup               │
│                                                             │
│  ┌─────────────────┐         ┌─────────────────┐           │
│  │  Active NameNode│         │ Standby NameNode│           │
│  │                 │         │                 │           │
│  │  Serves all     │         │  Hot standby    │           │
│  │  client requests│         │  (ready to take │           │
│  │                 │         │   over instantly)│           │
│  └────────┬────────┘         └────────┬────────┘           │
│           │                           │                     │
│           │    Shared EditLog         │                     │
│           │    (via JournalNodes)     │                     │
│           ▼                           ▼                     │
│  ┌──────────────────────────────────────────┐              │
│  │        JOURNAL NODES (Quorum)            │              │
│  │   JN1        JN2         JN3            │              │
│  │                                          │              │
│  │  Store EditLog segments                  │              │
│  │  Majority must ACK a write               │              │
│  │  (2 out of 3 = quorum)                  │              │
│  └──────────────────────────────────────────┘              │
│                                                             │
│  ┌──────────────────────────────────────────┐              │
│  │        ZOOKEEPER (Failover Controller)   │              │
│  │                                          │              │
│  │  Monitors health of Active NameNode      │              │
│  │  Triggers automatic failover             │              │
│  │  Prevents split-brain (fencing)          │              │
│  └──────────────────────────────────────────┘              │
│                                                             │
│  Failover time: ~30 seconds (automatic)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Data Write Pipeline

When a client writes a file to HDFS, data flows through a **pipeline** of DataNodes:

```
Client              DataNode 1         DataNode 2         DataNode 3
  │                    │                   │                   │
  │  1. Request block locations from NameNode                  │
  │                    │                   │                   │
  │  2. NameNode returns: [DN1, DN2, DN3]                      │
  │                    │                   │                   │
  │  3. Client connects to DN1 (first in pipeline)             │
  │────packet 1───────▶│                   │                   │
  │                    │──packet 1────────▶│                   │
  │                    │                   │──packet 1────────▶│
  │                    │                   │                   │
  │                    │                   │◀───ACK────────────│
  │                    │◀───ACK────────────│                   │
  │◀───ACK────────────│                   │                   │
  │                    │                   │                   │
  │────packet 2───────▶│                   │                   │
  │                    │──packet 2────────▶│                   │
  │                    │                   │──packet 2────────▶│
  │                    │                   │                   │
  │                    │                   │◀───ACK────────────│
  │                    │◀───ACK────────────│                   │
  │◀───ACK────────────│                   │                   │
  │                    │                   │                   │

PACKET SIZE: 64 KB (within a 128 MB block)
Pipeline reduces CLIENT'S upload bandwidth by 3x
(Client sends once, pipeline handles replication)
```

### Checksums and Data Integrity

```
┌─────────────────────────────────────────────────────┐
│          DATA INTEGRITY IN HDFS                     │
│                                                     │
│  For every 512 bytes of data, HDFS stores a        │
│  4-byte CRC-32 checksum.                           │
│                                                     │
│  Block on disk:                                    │
│  ┌─────────┬──────┬─────────┬──────┬─────────┐   │
│  │ Data    │ CRC  │ Data    │ CRC  │ Data    │   │
│  │ 512B    │ 4B   │ 512B    │ 4B   │ 512B    │   │
│  └─────────┴──────┴─────────┴──────┴─────────┘   │
│                                                     │
│  VERIFICATION HAPPENS:                              │
│  1. On every READ (client verifies)               │
│  2. Periodic background scan (DataNode Scanner)    │
│  3. During block replication                       │
│                                                     │
│  IF CHECKSUM FAILS:                                │
│  1. Client reports corrupted block to NameNode     │
│  2. NameNode marks that replica as corrupt         │
│  3. NameNode schedules re-replication from         │
│     a healthy replica                              │
│  4. Corrupted replica is deleted                   │
└─────────────────────────────────────────────────────┘
```

### How NameNode Manages Metadata

```
┌─────────────────────────────────────────────────────┐
│          NAMENODE MEMORY STRUCTURE                   │
│                                                     │
│  NAMESPACE (like a file system tree):               │
│  /                                                  │
│  ├── user/                                         │
│  │   ├── alice/                                    │
│  │   │   └── data.csv  [blocks: B1, B2, B3]      │
│  │   └── bob/                                      │
│  │       └── model.bin [blocks: B4, B5]           │
│  └── tmp/                                          │
│      └── job_output/  [blocks: B6]                │
│                                                     │
│  BLOCK MAP (in memory):                            │
│  B1 → {DN1:disk3, DN3:disk1, DN5:disk2}          │
│  B2 → {DN2:disk1, DN4:disk3, DN6:disk1}          │
│  B3 → {DN1:disk2, DN2:disk4, DN4:disk1}          │
│  ...                                               │
│                                                     │
│  MEMORY USAGE:                                      │
│  ~150 bytes per block reference                    │
│  100 million blocks × 150B = ~15 GB RAM needed    │
│  (This limits cluster size!)                       │
│                                                     │
│  PERSISTENCE:                                       │
│  - FsImage: Complete namespace snapshot (on disk)  │
│  - EditLog: Journal of all changes since snapshot  │
│  - On restart: Load FsImage + replay EditLog       │
└─────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — HDFS Operations with hdfs Library

```python
from hdfs import InsecureClient
import os

# Connect to HDFS (via WebHDFS REST API)
client = InsecureClient('http://namenode:9870', user='alice')

# --- CREATE DIRECTORIES ---
client.makedirs('/user/alice/datasets/2024')

# --- UPLOAD A FILE ---
# Uploads local file, HDFS splits into 128MB blocks automatically
local_file = '/tmp/large_dataset.csv'  # 500 MB file
client.upload(
    hdfs_path='/user/alice/datasets/2024/sales.csv',
    local_path=local_file,
    overwrite=True
)
# HDFS will create: Block 1 (128MB), Block 2 (128MB), 
#                   Block 3 (128MB), Block 4 (116MB)

# --- READ A FILE ---
with client.read('/user/alice/datasets/2024/sales.csv') as reader:
    for line in reader:
        # Process line by line (streaming, not loading all into memory)
        process_record(line.decode('utf-8'))

# --- LIST DIRECTORY ---
files = client.list('/user/alice/datasets/2024/')
for f in files:
    status = client.status(f'/user/alice/datasets/2024/{f}')
    print(f"  {f}: {status['length'] / 1e6:.1f} MB, "
          f"replication={status['replication']}, "
          f"blockSize={status['blockSize'] / 1e6:.0f} MB")

# --- CHECK BLOCK LOCATIONS (for data locality) ---
# Shows which DataNodes hold each block of a file
import requests
resp = requests.get(
    'http://namenode:9870/webhdfs/v1/user/alice/datasets/2024/sales.csv',
    params={'op': 'GET_BLOCK_LOCATIONS'}
)
blocks = resp.json()['LocatedBlocks']['locatedBlocks']
for i, block in enumerate(blocks):
    locations = [loc['name'] for loc in block['locations']]
    print(f"  Block {i}: {block['block']['numBytes']/1e6:.0f} MB "
          f"on nodes: {locations}")

# --- DELETE ---
client.delete('/user/alice/datasets/2024/old_data.csv')
```

### Java — HDFS Operations with Hadoop FileSystem API

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.*;
import org.apache.hadoop.hdfs.DistributedFileSystem;
import org.apache.hadoop.hdfs.protocol.DatanodeInfo;

import java.io.*;
import java.net.URI;

public class HDFSExample {
    public static void main(String[] args) throws Exception {
        // Configure HDFS connection
        Configuration conf = new Configuration();
        conf.set("fs.defaultFS", "hdfs://namenode:9000");
        conf.set("dfs.replication", "3");
        conf.set("dfs.blocksize", "134217728"); // 128 MB

        FileSystem fs = FileSystem.get(URI.create("hdfs://namenode:9000"), conf);

        // --- WRITE A FILE ---
        Path hdfsPath = new Path("/user/alice/output/result.csv");
        try (FSDataOutputStream out = fs.create(hdfsPath, (short) 3)) {
            // Write data — HDFS handles blocking & replication
            out.writeUTF("id,name,value\n");
            out.writeUTF("1,Alice,100\n");
            out.writeUTF("2,Bob,200\n");
            out.hflush(); // Flush to DataNodes (visible to readers)
        }

        // --- READ A FILE ---
        try (FSDataInputStream in = fs.open(hdfsPath)) {
            BufferedReader reader = new BufferedReader(new InputStreamReader(in));
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }
        }

        // --- GET FILE STATUS (metadata) ---
        FileStatus status = fs.getFileStatus(hdfsPath);
        System.out.printf("File: %s%n", status.getPath());
        System.out.printf("Size: %d bytes%n", status.getLen());
        System.out.printf("Replication: %d%n", status.getReplication());
        System.out.printf("Block Size: %d MB%n", status.getBlockSize() / (1024*1024));

        // --- GET BLOCK LOCATIONS (data locality info) ---
        BlockLocation[] blocks = fs.getFileBlockLocations(status, 0, status.getLen());
        for (int i = 0; i < blocks.length; i++) {
            System.out.printf("Block %d: hosts=%s, offset=%d, length=%d%n",
                i,
                String.join(",", blocks[i].getHosts()),
                blocks[i].getOffset(),
                blocks[i].getLength());
        }

        // --- LIST DIRECTORY ---
        FileStatus[] files = fs.listStatus(new Path("/user/alice/output/"));
        for (FileStatus f : files) {
            System.out.printf("  %s [%s] %d bytes%n",
                f.getPath().getName(),
                f.isDirectory() ? "DIR" : "FILE",
                f.getLen());
        }

        fs.close();
    }
}
```

### MapReduce Example — Processing Data Where It Lives

```python
# This is the KEY insight of HDFS: move computation TO the data
# MapReduce leverages HDFS block locations for data locality

from mrjob.job import MRJob
import re

class WordCount(MRJob):
    """
    When this runs on a Hadoop cluster:
    1. HDFS tells MapReduce which nodes have each block
    2. Map tasks are scheduled ON the same nodes as the data blocks
    3. Data doesn't move over the network — computation goes to data!
    """

    def mapper(self, _, line):
        # This runs on the DataNode that HAS the data block
        # No network transfer needed to read the input!
        for word in re.findall(r'\w+', line.lower()):
            yield word, 1

    def reducer(self, word, counts):
        # Aggregation happens after shuffle
        yield word, sum(counts)

if __name__ == '__main__':
    WordCount.run()

# Execution flow with data locality:
#
# NameNode says: "Block B1 of input.txt is on DN1, DN3, DN5"
#
# YARN ResourceManager: "Schedule Map task on DN1" (data-local!)
#
# DN1 reads block B1 from LOCAL DISK (no network!)
# DN1 runs mapper on that block
# Result: minimal data movement, maximum throughput
```

---

## Infrastructure Examples

### Setting Up HDFS Cluster (Docker Compose)

```yaml
version: '3'
services:
  namenode:
    image: apache/hadoop:3.3.6
    hostname: namenode
    command: ["hdfs", "namenode"]
    ports:
      - "9870:9870"   # Web UI
      - "9000:9000"   # RPC
    environment:
      HADOOP_CONF_DIR: /etc/hadoop
    volumes:
      - namenode_data:/hadoop/dfs/name
      - ./config:/etc/hadoop
    
  datanode1:
    image: apache/hadoop:3.3.6
    hostname: datanode1
    command: ["hdfs", "datanode"]
    environment:
      HADOOP_CONF_DIR: /etc/hadoop
    volumes:
      - datanode1_data:/hadoop/dfs/data
      - ./config:/etc/hadoop
    depends_on: [namenode]

  datanode2:
    image: apache/hadoop:3.3.6
    hostname: datanode2
    command: ["hdfs", "datanode"]
    volumes:
      - datanode2_data:/hadoop/dfs/data
      - ./config:/etc/hadoop
    depends_on: [namenode]

  datanode3:
    image: apache/hadoop:3.3.6
    hostname: datanode3
    command: ["hdfs", "datanode"]
    volumes:
      - datanode3_data:/hadoop/dfs/data
      - ./config:/etc/hadoop
    depends_on: [namenode]

volumes:
  namenode_data:
  datanode1_data:
  datanode2_data:
  datanode3_data:
```

### HDFS Configuration (hdfs-site.xml)

```xml
<configuration>
    <!-- Block size: 128 MB (good for large files) -->
    <property>
        <name>dfs.blocksize</name>
        <value>134217728</value>
    </property>

    <!-- Replication factor: 3 copies of each block -->
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>

    <!-- NameNode data directory -->
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///hadoop/dfs/name</value>
    </property>

    <!-- DataNode data directory (can specify multiple disks!) -->
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///disk1/hdfs/data,file:///disk2/hdfs/data,file:///disk3/hdfs/data</value>
    </property>

    <!-- Enable rack awareness -->
    <property>
        <name>net.topology.script.file.name</name>
        <value>/etc/hadoop/rack-topology.sh</value>
    </property>

    <!-- NameNode HA (High Availability) -->
    <property>
        <name>dfs.nameservices</name>
        <value>mycluster</value>
    </property>
    <property>
        <name>dfs.ha.namenodes.mycluster</name>
        <value>nn1,nn2</value>
    </property>
</configuration>
```

---

## GFS vs HDFS Comparison

```
┌────────────────────────┬──────────────────────┬──────────────────────┐
│       Feature          │        GFS           │        HDFS          │
├────────────────────────┼──────────────────────┼──────────────────────┤
│ Organization           │ Google (proprietary) │ Apache (open-source) │
│ Chunk/Block size       │ 64 MB                │ 128 MB (default)     │
│ Master name            │ GFS Master           │ NameNode             │
│ Worker name            │ Chunkserver          │ DataNode             │
│ Replication            │ 3 (default)          │ 3 (default)          │
│ Consistency            │ Relaxed (append)     │ Strong (write-once)  │
│ Append support         │ Concurrent appends   │ Single writer only   │
│ Master HA              │ Shadow masters       │ Standby NameNode     │
│ Successor              │ Colossus (GFS2)      │ HDFS 3.x + Ozone    │
│ Erasure coding         │ Colossus yes         │ HDFS 3.x yes         │
│ Typical use            │ Google internal      │ Hadoop ecosystem     │
└────────────────────────┴──────────────────────┴──────────────────────┘
```

---

## Modern Evolution: Beyond HDFS

### Cloud Object Storage Has Replaced HDFS for Many Use Cases

```
2005-2015: HDFS Era                    2015-Present: Cloud Era
─────────────────────                  ──────────────────────

┌─────────────────────┐               ┌─────────────────────┐
│  On-Premise Cluster │               │  Cloud Architecture │
│                     │               │                     │
│  Compute + Storage  │               │  Compute (separate) │
│  tightly coupled    │               │    ┌───────────┐    │
│                     │               │    │ Spark/EMR │    │
│  ┌───────────────┐  │               │    └─────┬─────┘    │
│  │HDFS DataNodes │  │               │          │          │
│  │  + MapReduce  │  │               │          ▼          │
│  │  on SAME node │  │               │  Storage (separate) │
│  └───────────────┘  │               │    ┌───────────┐    │
│                     │               │    │  S3 / GCS │    │
│  Pro: Data locality │               │    └───────────┘    │
│  Con: Can't scale   │               │                     │
│       independently │               │  Pro: Scale compute │
│                     │               │       and storage   │
└─────────────────────┘               │       independently │
                                      │  Con: Network hop   │
                                      │       for all reads │
                                      └─────────────────────┘

Modern compromise: Use S3 as primary storage, 
cache hot data on local NVMe SSDs (like Apache Spark 
with "data caching" or Alluxio)
```

### When HDFS Is Still Relevant

| Scenario | Use HDFS | Use Cloud Object Storage |
|----------|----------|------------------------|
| On-premise big data | ✅ | ❌ |
| Data locality critical | ✅ | ❌ |
| Low-latency iterative processing | ✅ | ❌ |
| Cost-sensitive at petabyte scale (own hardware) | ✅ | ❌ |
| Cloud-native applications | ❌ | ✅ |
| Serverless analytics (Athena, BigQuery) | ❌ | ✅ |
| Multi-framework access (Spark, Presto, Flink) | ❌ | ✅ |
| Elastic compute (scale up/down) | ❌ | ✅ |

---

## Real-World Example

### How Google Uses Distributed File Systems

```
┌─────────────────────────────────────────────────────────────────┐
│              GOOGLE'S STORAGE EVOLUTION                          │
│                                                                 │
│  2003: GFS (Google File System)                                │
│  ├── Designed for web crawl data                               │
│  ├── 64 MB chunks, 3x replication                             │
│  ├── Single master (bottleneck at scale)                       │
│  └── Optimized for large sequential reads                      │
│                                                                 │
│  2010+: COLOSSUS (GFS2)                                        │
│  ├── Eliminated single-master bottleneck                       │
│  │   (distributed metadata with Bigtable)                      │
│  ├── Smaller chunk size (1 MB) → better for small files        │
│  ├── Erasure coding (1.5x overhead vs 3x with replication)    │
│  ├── Reed-Solomon encoding: 6 data + 3 parity chunks          │
│  ├── Client-side encoding (less server work)                   │
│  └── Powers: Gmail, YouTube, Google Drive, BigQuery,           │
│       Cloud Storage, ALL Google services                        │
│                                                                 │
│  COLOSSUS SCALE:                                               │
│  • Stores EXABYTES (thousands of petabytes)                    │
│  • Millions of servers                                         │
│  • Handles billions of operations per second                   │
│  • 99.999999999% durability                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### How Facebook/Meta Uses HDFS

```
FACEBOOK'S HDFS DEPLOYMENT (largest known):
─────────────────────────────────────────────
• 300+ PB stored in HDFS (by 2014, much larger now)
• Multiple clusters, each with 1000s of nodes
• Used for: analytics, ML training, ad ranking data
• Custom modifications:
  - HDFS RAID (reduce 3x replication → ~1.4x with XOR coding)
  - AvatarNode (fast NameNode failover before official HA)
  - Federation (multiple NameNodes for different namespaces)
```

### How Yahoo/Hadoop Ecosystem Uses HDFS

```
┌─────────────────────────────────────────────────┐
│         HADOOP ECOSYSTEM ON HDFS                │
│                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │  Hive    │ │  Spark   │ │  Flink   │       │
│  │ (SQL on  │ │ (Fast    │ │ (Stream  │       │
│  │  Hadoop) │ │  batch)  │ │  process)│       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘       │
│       │             │            │              │
│       └─────────────┼────────────┘              │
│                     │                           │
│                     ▼                           │
│  ┌──────────────────────────────────────┐      │
│  │          YARN (Resource Manager)     │      │
│  │     Schedules computation on nodes   │      │
│  └──────────────────┬───────────────────┘      │
│                     │                           │
│                     ▼                           │
│  ┌──────────────────────────────────────┐      │
│  │              HDFS                    │      │
│  │     Stores all the data             │      │
│  │     Provides data locality info     │      │
│  │     to YARN for task scheduling     │      │
│  └──────────────────────────────────────┘      │
│                                                 │
│  Data locality = #1 performance optimization    │
│  YARN tries to run tasks on nodes that HAVE    │
│  the required HDFS blocks                       │
└─────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Storing millions of small files** | Each file = 1 block = 150B of NameNode RAM. 100M small files = NameNode OOM | Combine small files into SequenceFiles, Avro, or Parquet |
| **Single NameNode without HA** | NameNode crash = entire cluster goes offline | Always deploy Standby NameNode + ZooKeeper |
| **Default replication for cold data** | 3x replication on 10 PB = 30 PB stored | Use erasure coding (HDFS 3.x) for cold data: 1.5x overhead |
| **Not enabling rack awareness** | All 3 replicas might end up in same rack → rack failure = data loss | Configure topology scripts properly |
| **Using HDFS for random access** | HDFS is optimized for sequential reads of large files | Use HBase (on HDFS) for random access, or a proper database |
| **Ignoring NameNode memory limits** | NameNode keeps ALL metadata in RAM | Monitor heap usage; consider HDFS Federation for scaling |
| **Not monitoring DataNode health** | Dead nodes mean under-replicated blocks | Set up alerts for under-replication and disk failures |

### The Small Files Problem (Critical!)

```
PROBLEM:
─────────────────────────────────────
1 million files × 1 KB each = 1 GB total data

But in HDFS:
- 1 million blocks on NameNode = 150 MB RAM just for metadata
- 1 million blocks × 3 replicas = 3 million block reports
- Each file uses minimum 1 block (128 MB allocated, 1 KB used)
  → MASSIVE waste of disk space and NameNode memory

SOLUTION:
─────────────────────────────────────
Combine small files into large container formats:

┌─────────────────────────────────────┐
│       SEQUENCEFILE / PARQUET        │
│                                     │
│  ┌─────┐┌─────┐┌─────┐┌─────┐    │
│  │rec 1││rec 2││rec 3││rec 4│... │  One large file
│  └─────┘└─────┘└─────┘└─────┘    │  containing
│                                     │  millions of
│  Size: 128 MB+ (fills blocks)      │  records
│  NameNode overhead: 1 block entry  │
└─────────────────────────────────────┘

Also consider:
- HAR (Hadoop Archive) files
- HBase for small random-access records
- Apache Ozone (object store on top of HDFS)
```

---

## When to Use / When NOT to Use

### ✅ Use Distributed File Systems (HDFS) When:

- Processing **petabytes** of data in batch (MapReduce, Spark, Hive)
- **Data locality** is critical (minimize network I/O)
- You have an **on-premise cluster** with commodity hardware
- Workload is **write-once, read-many** (append-only)
- You need a **single namespace** across thousands of machines
- Running **Hadoop ecosystem** tools (Hive, Spark, HBase, Flink)

### ❌ Do NOT Use HDFS When:

- You're in the **cloud** and can use S3/GCS (usually cheaper & simpler)
- You need **low-latency random access** (< 10ms) → Use a database or HBase
- You have **millions of small files** (< 1 MB each) → Use object storage
- You need **strong consistency** for concurrent writers → Use a database
- Your data is **< 1 TB** (overkill; just use regular storage)
- You need **elastic scaling** (HDFS clusters are hard to resize quickly)

### Decision Flow

```
"Do I need HDFS?"

Is your data > 10 TB?
├── NO → Regular file storage / database is fine
└── YES ↓

Are you on-premise?
├── YES → HDFS is a great choice
└── NO (cloud) ↓

Do you need data locality for iterative ML/analytics?
├── YES → Consider HDFS on cloud VMs (or Alluxio cache layer)
└── NO → Use S3/GCS + Spark/EMR (simpler, cheaper)
```

---

## Key Takeaways

- **Distributed file systems** (GFS, HDFS) store massive files across thousands of commodity servers by splitting them into large blocks (64-128 MB) and replicating each block 3 times.
- The **master node** (NameNode in HDFS) stores all metadata in RAM for fast lookups, while **worker nodes** (DataNodes) store the actual data blocks on local disks.
- **Data locality** is the key optimization: move computation TO the data rather than moving data to computation. This eliminates network bottlenecks.
- **Rack awareness** ensures replicas are spread across different failure domains (racks), surviving both disk failures and entire rack failures.
- **The small files problem** is the #1 pitfall — each file consumes NameNode memory regardless of size. Combine small files into large container formats.
- **Modern trend**: Cloud object storage (S3, GCS) has replaced HDFS for many workloads, but HDFS remains relevant for on-premise big data and latency-sensitive iterative processing.
- **Google evolved** from GFS to Colossus, replacing replication with erasure coding and distributing the master to eliminate the single-master bottleneck.

---

## What's Next?

Next, we'll explore how companies like Netflix, YouTube, and Instagram handle the complete pipeline from upload to delivery with [Content Delivery & Media Processing Pipelines](./04-media-processing.md) — covering transcoding, thumbnail generation, adaptive streaming, and serving billions of media files globally.
