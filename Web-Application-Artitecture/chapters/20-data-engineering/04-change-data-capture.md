# Change Data Capture (CDC) — Debezium

> **What you'll learn**: How to detect and stream every INSERT, UPDATE, and DELETE from your production database to downstream systems in real-time — without modifying your application code or impacting database performance.

---

## Real-Life Analogy

Imagine you're a **journalist** covering a city council.

### The Old Way (Batch/Polling)

Every night at midnight, you go to City Hall, photocopy ALL the official records, compare them to yesterday's copies, and figure out what changed. This is slow, wasteful, and you miss changes that happened and were reversed in the same day.

### The CDC Way (Transaction Log)

Instead, you sit in the council chamber with a live stenographer's feed. Every time a council member proposes a change, votes, or signs a document — you get a real-time notification:

```
12:03 PM — INSERTED: New building permit #4521 for 123 Main St
12:15 PM — UPDATED:  Permit #4520 status changed: "pending" → "approved"
12:30 PM — DELETED:  Temporary parking restriction on Oak Ave removed
```

You never miss a change. You get it the moment it happens. And you didn't have to compare entire filing cabinets.

**That's CDC** — reading the database's own transaction log to capture every change as it happens.

---

## Core Concept Explained Step-by-Step

### Step 1: The Problem CDC Solves

In [Chapter 20.2](./02-data-pipelines.md), we learned about ETL/ELT pipelines. But how does the pipeline know **what changed** since the last run?

```
Naive approach (full table scan every night):
─────────────────────────────────────────────
  Every night: SELECT * FROM orders  → Copy ALL 500 million rows
  
  Problems:
  • Scans entire table (takes hours)
  • Wastes bandwidth (99% of rows didn't change)
  • Creates load on production database
  • Misses intra-day changes (insert + delete before midnight)
  • Hours of latency (data is always stale)


CDC approach (stream changes in real-time):
─────────────────────────────────────────────
  Continuously: Read transaction log → Stream only CHANGES
  
  Benefits:
  • Only sends changed rows (efficient)
  • Near-zero load on production database
  • Milliseconds of latency (near real-time)
  • Captures ALL changes including intermediate states
  • No full table scans needed
```

### Step 2: How Databases Store Changes (The Transaction Log)

Every serious database maintains a **write-ahead log (WAL)** or **transaction log** — a sequential record of every change before it's applied:

```
┌─────────────────────────────────────────────────────────────────┐
│              DATABASE TRANSACTION LOG (WAL)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PostgreSQL: WAL (Write-Ahead Log)                                │
│  MySQL:      Binlog (Binary Log)                                  │
│  MongoDB:    Oplog (Operations Log)                               │
│  SQL Server: Transaction Log                                      │
│  Oracle:     Redo Log                                             │
│                                                                   │
│  Example WAL entries:                                             │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ LSN: 000000010000000000000042                             │    │
│  │ Time: 2024-06-15 14:23:01.456                            │    │
│  │ Operation: INSERT                                         │    │
│  │ Table: public.orders                                      │    │
│  │ Data: {id:9001, user:42, amount:150.00, status:"new"}    │    │
│  ├──────────────────────────────────────────────────────────┤    │
│  │ LSN: 000000010000000000000043                             │    │
│  │ Time: 2024-06-15 14:23:02.100                            │    │
│  │ Operation: UPDATE                                         │    │
│  │ Table: public.orders                                      │    │
│  │ Before: {id:9001, status:"new"}                          │    │
│  │ After:  {id:9001, status:"processing"}                   │    │
│  ├──────────────────────────────────────────────────────────┤    │
│  │ LSN: 000000010000000000000044                             │    │
│  │ Time: 2024-06-15 14:23:05.789                            │    │
│  │ Operation: DELETE                                         │    │
│  │ Table: public.temp_cache                                  │    │
│  │ Data: {id:500, key:"session_xyz"}                        │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                   │
│  CDC reads this log → streams changes to Kafka/warehouse          │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: CDC Architecture with Debezium

**Debezium** is the most popular open-source CDC platform. It reads database transaction logs and publishes change events to Apache Kafka.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   CDC ARCHITECTURE (DEBEZIUM + KAFKA)                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Production DB                 Debezium              Kafka               │
│  ────────────                 ────────              ─────                │
│                                                                          │
│  ┌──────────────┐         ┌──────────────┐     ┌──────────────────┐     │
│  │  PostgreSQL   │         │   Debezium    │     │  Kafka Topic:     │     │
│  │              │         │   Connector   │     │  "db.orders"      │     │
│  │  orders table│ ──WAL──▶│              │────▶│                   │     │
│  │  users table │         │  Reads WAL    │     │  Change events:   │     │
│  │  products   │         │  No queries!  │     │  {op:"c", ...}   │     │
│  └──────────────┘         └──────────────┘     │  {op:"u", ...}   │     │
│                                                │  {op:"d", ...}   │     │
│  Zero impact on                                └────────┬─────────┘     │
│  production queries!                                    │               │
│                                                         │               │
│                              ┌───────────────────────────┼────────┐      │
│                              │                           │        │      │
│                              ▼                           ▼        ▼      │
│                    ┌──────────────┐          ┌────────┐ ┌──────────┐    │
│                    │ Data Warehouse│          │ Search │ │ Cache    │    │
│                    │ (BigQuery)    │          │(Elastic│ │(Redis)   │    │
│                    └──────────────┘          │Search) │ └──────────┘    │
│                                             └────────┘                  │
│                                                                          │
│  Every downstream system stays in sync with production — automatically!  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Step 4: CDC Methods Compared

```
┌──────────────────────┬─────────────────────────────────────────────────┐
│ Method               │ Description                                      │
├──────────────────────┼─────────────────────────────────────────────────┤
│ Log-Based CDC        │ Read database transaction log (WAL/Binlog)       │
│ (Debezium)           │ ✅ No impact on DB, captures ALL changes         │
│                      │ ✅ Milliseconds latency, ordered, complete       │
├──────────────────────┼─────────────────────────────────────────────────┤
│ Trigger-Based CDC    │ Database triggers fire on INSERT/UPDATE/DELETE   │
│                      │ ❌ Adds overhead to every write                  │
│                      │ ❌ Complex to maintain, slows production         │
├──────────────────────┼─────────────────────────────────────────────────┤
│ Timestamp-Based      │ Query WHERE updated_at > last_sync_time         │
│ (Polling)            │ ❌ Misses deletes, requires timestamp column     │
│                      │ ❌ Creates load on production, minutes of delay  │
├──────────────────────┼─────────────────────────────────────────────────┤
│ Diff-Based           │ Compare current snapshot to previous snapshot    │
│ (Full comparison)    │ ❌ Very slow for large tables                    │
│                      │ ❌ Misses intermediate states                    │
└──────────────────────┴─────────────────────────────────────────────────┘

Winner: Log-Based CDC (Debezium) — used by 90%+ of production systems
```

---

## How It Works Internally

### Debezium's Internal Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     DEBEZIUM INTERNALS                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                    KAFKA CONNECT FRAMEWORK                      │     │
│  │                                                                │     │
│  │  ┌────────────────────────────────────────────────────────┐   │     │
│  │  │              DEBEZIUM SOURCE CONNECTOR                  │   │     │
│  │  │                                                        │   │     │
│  │  │  1. SNAPSHOT PHASE (initial):                          │   │     │
│  │  │     • Locks table briefly                              │   │     │
│  │  │     • Records current WAL position (LSN)               │   │     │
│  │  │     • Reads all existing rows                          │   │     │
│  │  │     • Publishes as "read" events (op:"r")             │   │     │
│  │  │                                                        │   │     │
│  │  │  2. STREAMING PHASE (ongoing):                         │   │     │
│  │  │     • Connects to WAL replication slot                 │   │     │
│  │  │     • Reads WAL entries from recorded LSN              │   │     │
│  │  │     • Decodes binary WAL → structured events           │   │     │
│  │  │     • Publishes to Kafka topics                        │   │     │
│  │  │     • Commits offset (tracks progress)                 │   │     │
│  │  │                                                        │   │     │
│  │  │  ┌──────────────────────────────────────────────┐     │   │     │
│  │  │  │  OFFSET STORAGE                               │     │   │     │
│  │  │  │  Tracks: LSN position, txn ID                 │     │   │     │
│  │  │  │  If connector restarts → resumes from here    │     │   │     │
│  │  │  └──────────────────────────────────────────────┘     │   │     │
│  │  └────────────────────────────────────────────────────────┘   │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Change Event Structure (Debezium Format)

When Debezium captures a change, it produces a structured JSON event:

```json
{
  "schema": { ... },
  "payload": {
    "before": {                          // Row BEFORE the change (null for INSERT)
      "id": 9001,
      "user_id": 42,
      "amount": 150.00,
      "status": "new"
    },
    "after": {                           // Row AFTER the change (null for DELETE)
      "id": 9001,
      "user_id": 42,
      "amount": 150.00,
      "status": "processing"
    },
    "source": {                          // Metadata about where this came from
      "version": "2.5.0",
      "connector": "postgresql",
      "name": "prod-db",
      "ts_ms": 1718458982100,            // Timestamp of change in DB
      "db": "myapp",
      "schema": "public",
      "table": "orders",
      "lsn": 234567890,                  // WAL position
      "txId": 12345                      // Transaction ID
    },
    "op": "u",                           // c=create, u=update, d=delete, r=read(snapshot)
    "ts_ms": 1718458982200               // Timestamp Debezium processed it
  }
}
```

### PostgreSQL WAL Replication Slot

```
┌─────────────────────────────────────────────────────────────────┐
│              POSTGRESQL LOGICAL REPLICATION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PostgreSQL                                                       │
│  ┌───────────────────────────────────────────────────┐           │
│  │  WAL (Write-Ahead Log)                             │           │
│  │  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐    │           │
│  │  │ LSN │ LSN │ LSN │ LSN │ LSN │ LSN │ LSN │    │           │
│  │  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │    │           │
│  │  └──┬──┴─────┴─────┴──┬──┴─────┴─────┴─────┘    │           │
│  │     │                  │                           │           │
│  │     │   Replication    │                           │           │
│  │     │   Slot Bookmark  │                           │           │
│  │     ▼                  ▼                           │           │
│  │  "debezium_slot"     "Currently reading LSN 4"    │           │
│  │                                                    │           │
│  │  The slot ensures:                                │           │
│  │  • WAL entries aren't deleted until consumed      │           │
│  │  • Debezium can resume from exact position        │           │
│  │  • No data loss even if connector is down         │           │
│  └───────────────────────────────────────────────────┘           │
│         │                                                         │
│         │  Logical Decoding Plugin (pgoutput / wal2json)          │
│         │  Converts binary WAL → readable change events           │
│         ▼                                                         │
│  ┌───────────────────┐                                           │
│  │   Debezium         │                                           │
│  │   Connector        │                                           │
│  └───────────────────┘                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Setting Up Debezium with Docker Compose

```yaml
# docker-compose.yml — Full CDC stack
version: '3.8'

services:
  # Source database
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: secret
    command:
      - "postgres"
      - "-c" 
      - "wal_level=logical"            # REQUIRED for CDC!
      - "-c"
      - "max_replication_slots=4"      # Allow replication slots
      - "-c"
      - "max_wal_senders=4"            # Allow WAL streaming
    ports:
      - "5432:5432"

  # Kafka (message broker for change events)
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    ports:
      - "9092:9092"

  # Kafka Connect + Debezium
  connect:
    image: debezium/connect:2.5
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: debezium-connect
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_statuses
    ports:
      - "8083:8083"
    depends_on:
      - kafka
      - postgres
```

### Python: Register Debezium Connector

```python
import requests
import json

# ═══════════════════════════════════════════════════════
# Register a Debezium PostgreSQL connector via REST API
# This tells Debezium to start capturing changes from our DB
# ═══════════════════════════════════════════════════════

def register_cdc_connector():
    """Register Debezium connector to capture changes from PostgreSQL."""
    
    connector_config = {
        "name": "orders-connector",
        "config": {
            # Connector class (PostgreSQL)
            "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
            
            # Database connection
            "database.hostname": "postgres",
            "database.port": "5432",
            "database.user": "debezium",
            "database.password": "secret",
            "database.dbname": "myapp",
            
            # What to capture
            "table.include.list": "public.orders,public.users,public.products",
            
            # Kafka topic prefix (topics: "myapp.public.orders", etc.)
            "topic.prefix": "myapp",
            
            # Replication slot name
            "slot.name": "debezium_orders",
            "plugin.name": "pgoutput",       # Logical decoding plugin
            
            # Snapshot mode (initial = full snapshot first, then streaming)
            "snapshot.mode": "initial",
            
            # Transforms: route events to specific topics
            "transforms": "route",
            "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
            "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
            "transforms.route.replacement": "cdc.$3",   # Topic: cdc.orders
            
            # Heartbeat to prevent WAL buildup during quiet periods
            "heartbeat.interval.ms": "10000",
        }
    }
    
    response = requests.post(
        "http://localhost:8083/connectors",
        headers={"Content-Type": "application/json"},
        data=json.dumps(connector_config)
    )
    
    if response.status_code == 201:
        print("✅ CDC connector registered successfully!")
    else:
        print(f"❌ Failed: {response.text}")
    
    return response.json()


def check_connector_status():
    """Check if the connector is running and healthy."""
    response = requests.get("http://localhost:8083/connectors/orders-connector/status")
    status = response.json()
    
    connector_state = status['connector']['state']
    task_states = [t['state'] for t in status['tasks']]
    
    print(f"Connector: {connector_state}")
    print(f"Tasks: {task_states}")
    # Expected: Connector: RUNNING, Tasks: ['RUNNING']
```

### Python: Consuming CDC Events from Kafka

```python
from kafka import KafkaConsumer
import json

# ═══════════════════════════════════════════════════════
# Consume CDC events from Kafka and sync to data warehouse
# Each message represents one row change in PostgreSQL
# ═══════════════════════════════════════════════════════

def consume_cdc_events():
    """Consume Debezium CDC events and process them."""
    
    consumer = KafkaConsumer(
        'cdc.orders',                     # Kafka topic with order changes
        bootstrap_servers=['localhost:9092'],
        group_id='warehouse-sync',
        value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        auto_offset_reset='earliest',     # Start from beginning if new consumer
        enable_auto_commit=False,         # Manual commit for exactly-once
    )
    
    for message in consumer:
        event = message.value
        payload = event['payload']
        
        op = payload['op']           # c=create, u=update, d=delete
        table = payload['source']['table']
        
        if op == 'c':   # INSERT
            row = payload['after']
            print(f"INSERT into {table}: {row}")
            upsert_to_warehouse(table, row)
            
        elif op == 'u':  # UPDATE
            before = payload['before']
            after = payload['after']
            print(f"UPDATE {table}: {before} → {after}")
            upsert_to_warehouse(table, after)
            
        elif op == 'd':  # DELETE
            row = payload['before']
            print(f"DELETE from {table}: {row}")
            soft_delete_in_warehouse(table, row['id'])
            
        elif op == 'r':  # SNAPSHOT READ (initial load)
            row = payload['after']
            upsert_to_warehouse(table, row)
        
        # Commit offset after processing
        consumer.commit()


def upsert_to_warehouse(table: str, row: dict):
    """Insert or update row in the data warehouse."""
    # In production: batch these and bulk-load every N seconds
    # Use MERGE/UPSERT statement in your warehouse
    pass
```

### Java: CDC Event Consumer with Kafka Streams

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.common.serialization.Serdes;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.util.Properties;

public class CDCEventProcessor {
    
    private static final ObjectMapper mapper = new ObjectMapper();
    
    /**
     * Process CDC events using Kafka Streams.
     * Routes changes to different sinks based on operation type.
     */
    public void startProcessing() {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "cdc-processor");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Read CDC events from Debezium topic
        KStream<String, String> cdcEvents = builder.stream("cdc.orders");
        
        // Branch based on operation type
        cdcEvents
            .mapValues(this::parseEvent)
            .filter((key, event) -> event != null)
            .foreach((key, event) -> {
                String operation = event.get("payload").get("op").asText();
                JsonNode after = event.get("payload").get("after");
                JsonNode before = event.get("payload").get("before");
                
                switch (operation) {
                    case "c" -> handleInsert(after);
                    case "u" -> handleUpdate(before, after);
                    case "d" -> handleDelete(before);
                    case "r" -> handleSnapshot(after);
                }
            });
        
        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
        
        // Graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
    
    private JsonNode parseEvent(String raw) {
        try {
            return mapper.readTree(raw);
        } catch (Exception e) {
            System.err.println("Failed to parse CDC event: " + e.getMessage());
            return null;
        }
    }
    
    private void handleInsert(JsonNode row) {
        long orderId = row.get("id").asLong();
        double amount = row.get("amount").asDouble();
        System.out.printf("New order #%d — $%.2f%n", orderId, amount);
        
        // Sync to warehouse, update cache, trigger notifications, etc.
        syncToWarehouse("INSERT", row);
        invalidateCache("order:" + orderId);
    }
    
    private void handleUpdate(JsonNode before, JsonNode after) {
        String oldStatus = before.get("status").asText();
        String newStatus = after.get("status").asText();
        System.out.printf("Order #%d: %s → %s%n",
            after.get("id").asLong(), oldStatus, newStatus);
        
        syncToWarehouse("UPSERT", after);
        invalidateCache("order:" + after.get("id").asLong());
    }
    
    private void handleDelete(JsonNode row) {
        System.out.printf("Order #%d deleted%n", row.get("id").asLong());
        syncToWarehouse("DELETE", row);
        invalidateCache("order:" + row.get("id").asLong());
    }
    
    private void handleSnapshot(JsonNode row) {
        // Initial load — just insert into warehouse
        syncToWarehouse("UPSERT", row);
    }
    
    private void syncToWarehouse(String op, JsonNode row) { /* ... */ }
    private void invalidateCache(String key) { /* ... */ }
}
```

---

## Infrastructure Examples

### PostgreSQL Configuration for CDC

```sql
-- Enable logical replication (required for Debezium)
ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM SET max_replication_slots = 4;
ALTER SYSTEM SET max_wal_senders = 4;

-- Create a dedicated user for Debezium (principle of least privilege)
CREATE ROLE debezium WITH LOGIN PASSWORD 'secure_password' REPLICATION;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO debezium;

-- Create a publication for the tables you want to capture
CREATE PUBLICATION debezium_pub FOR TABLE orders, users, products;

-- Verify replication slot (created automatically by Debezium)
SELECT * FROM pg_replication_slots;
-- slot_name: debezium_orders
-- plugin: pgoutput
-- active: t
-- restart_lsn: 0/15A2340

-- Monitor WAL lag (how far behind the connector is)
SELECT 
    slot_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots;
```

### MySQL Binlog Configuration for CDC

```ini
# my.cnf — Enable binlog for CDC
[mysqld]
server-id         = 1
log_bin           = mysql-bin
binlog_format     = ROW          # REQUIRED: must be ROW, not STATEMENT
binlog_row_image  = FULL         # Capture full row before + after
expire_logs_days  = 3            # Keep binlogs for 3 days
```

### Kubernetes Deployment (Production)

```yaml
# Kafka Connect with Debezium on Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debezium-connect
spec:
  replicas: 2  # HA: multiple workers
  selector:
    matchLabels:
      app: debezium-connect
  template:
    metadata:
      labels:
        app: debezium-connect
    spec:
      containers:
        - name: connect
          image: debezium/connect:2.5
          resources:
            requests:
              memory: "2Gi"
              cpu: "1000m"
            limits:
              memory: "4Gi"
              cpu: "2000m"
          env:
            - name: BOOTSTRAP_SERVERS
              value: "kafka-0.kafka:9092,kafka-1.kafka:9092,kafka-2.kafka:9092"
            - name: GROUP_ID
              value: "debezium-cluster"
            - name: CONFIG_STORAGE_TOPIC
              value: "debezium_configs"
            - name: OFFSET_STORAGE_TOPIC
              value: "debezium_offsets"
            - name: STATUS_STORAGE_TOPIC
              value: "debezium_statuses"
            - name: CONFIG_STORAGE_REPLICATION_FACTOR
              value: "3"
            - name: OFFSET_STORAGE_REPLICATION_FACTOR
              value: "3"
          ports:
            - containerPort: 8083
          livenessProbe:
            httpGet:
              path: /
              port: 8083
            initialDelaySeconds: 30
            periodSeconds: 10
```

---

## CDC Use Cases Beyond Data Warehousing

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     CDC USE CASES                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. DATA WAREHOUSE SYNC (most common)                                    │
│     PostgreSQL ──CDC──▶ Kafka ──▶ Snowflake/BigQuery                     │
│                                                                          │
│  2. CACHE INVALIDATION                                                   │
│     PostgreSQL ──CDC──▶ Kafka ──▶ Redis (invalidate stale cache entries) │
│                                                                          │
│  3. SEARCH INDEX SYNC                                                    │
│     PostgreSQL ──CDC──▶ Kafka ──▶ Elasticsearch (keep search updated)   │
│                                                                          │
│  4. MICROSERVICE SYNC                                                    │
│     Order Service DB ──CDC──▶ Kafka ──▶ Shipping Service                 │
│     (avoids distributed transactions — see Chapter 13.6)                 │
│                                                                          │
│  5. AUDIT LOGGING                                                        │
│     Any DB ──CDC──▶ Kafka ──▶ Immutable Audit Store                      │
│     (who changed what, when — for compliance)                            │
│                                                                          │
│  6. REAL-TIME MATERIALIZED VIEWS                                         │
│     Source tables ──CDC──▶ Stream Processor ──▶ Aggregated views         │
│                                                                          │
│  7. DATABASE MIGRATION                                                   │
│     Old DB ──CDC──▶ Kafka ──▶ New DB (zero-downtime migration)           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Uber's CDC at Scale

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      UBER's CDC ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Uber has 1000+ microservices, each with its own database.               │
│  CDC is the backbone connecting them all.                                │
│                                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                               │
│  │ Rides DB  │  │ Payments │  │  Users    │  ... 1000+ databases          │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘                               │
│        │              │              │                                    │
│        ▼              ▼              ▼                                    │
│  ┌──────────────────────────────────────────┐                            │
│  │        Debezium / Custom CDC Layer        │                            │
│  │        (Reads MySQL binlogs at scale)     │                            │
│  └──────────────────────┬───────────────────┘                            │
│                         │                                                │
│                         ▼                                                │
│  ┌──────────────────────────────────────────┐                            │
│  │           Apache Kafka                    │                            │
│  │     (Trillions of messages/day)           │                            │
│  └──────────────────────┬───────────────────┘                            │
│                         │                                                │
│            ┌────────────┼────────────────────┐                           │
│            ▼            ▼                    ▼                            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐                 │
│  │ Data Lake     │ │ Elasticsearch│ │ Real-Time ML      │                 │
│  │ (Apache Hudi) │ │ (Search)     │ │ (Fraud Detection) │                 │
│  └──────────────┘ └──────────────┘ └──────────────────┘                 │
│                                                                          │
│  Scale: Processes 10+ million events/second through CDC                  │
│  Latency: Sub-second from DB write to downstream availability            │
│  Innovation: Built custom CDC tool when Debezium couldn't scale enough   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Airbnb

- Uses Debezium CDC to stream MySQL changes to Apache Kafka
- Powers their real-time search index (when a host updates availability → Elasticsearch updates in seconds)
- Feeds data into their "Minerva" metrics platform for real-time dashboards

### WePay (acquired by JPMorgan Chase)

- Pioneered production use of Debezium for CDC
- Streams all payment transaction changes to BigQuery in real-time
- Used for compliance reporting and fraud detection with sub-minute latency

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Forgetting `wal_level=logical` | CDC connector can't read WAL | Set before deploying Debezium; requires DB restart |
| Not monitoring WAL/replication lag | If connector falls behind, WAL grows → disk fills up → DB crashes | Alert on lag > 100MB, monitor `pg_replication_slots` |
| Not handling schema changes | Adding/removing columns breaks downstream consumers | Use schema registry (Confluent/Apicurio), plan migrations |
| Processing events out of order | Applying UPDATE before INSERT causes failures | Use Kafka partition key = primary key (ensures ordering per entity) |
| No dead letter queue | Poison messages block entire pipeline | Configure DLQ for unparseable/failed events |
| Running initial snapshot during peak hours | Snapshot locks tables, creates massive load | Schedule snapshots during maintenance windows |
| Not cleaning up old replication slots | Inactive slots prevent WAL cleanup → disk fills | Monitor and drop inactive slots |

---

## When to Use / When NOT to Use

### Use CDC When:
- ✅ You need real-time or near-real-time data sync (seconds, not hours)
- ✅ You want zero impact on production database performance
- ✅ You need to capture ALL changes including intermediate states and deletes
- ✅ Multiple downstream systems need the same change events
- ✅ You're migrating databases and need zero-downtime cutover
- ✅ Microservices need to stay in sync without distributed transactions

### When NOT to Use CDC:
- ❌ Daily batch sync is perfectly fine for your use case (simpler approaches exist)
- ❌ Your source database doesn't support logical replication (some managed DBs restrict it)
- ❌ You only have a few thousand rows total (overhead not worth it)
- ❌ You don't have Kafka infrastructure (and don't want to manage it)

### Alternatives to Consider:
- **Simple cases**: Timestamp-based polling (WHERE updated_at > last_sync)
- **AWS-native**: DynamoDB Streams (built-in CDC for DynamoDB)
- **Managed CDC**: AWS DMS, GCP Datastream, Fivetran (no self-hosting)

---

## Key Takeaways

1. **CDC reads the database's transaction log** (WAL/Binlog) to capture every INSERT, UPDATE, and DELETE in real-time — without querying the database.

2. **Debezium** is the industry-standard open-source CDC tool. It runs as a Kafka Connect connector and publishes change events to Kafka topics.

3. **Log-based CDC** is superior to triggers, polling, or diffing because it has zero impact on production queries and captures ALL changes with correct ordering.

4. CDC events contain **before** and **after** images of each row, plus metadata (table, timestamp, transaction ID) — enabling full auditability.

5. **Initial snapshot** + **continuous streaming** ensures you get all existing data plus all future changes — with exactly-once semantics when configured properly.

6. CDC is not just for data warehousing — it powers **cache invalidation**, **search sync**, **microservice integration**, **audit logging**, and **zero-downtime migrations**.

7. **Monitor WAL lag** religiously. If CDC falls behind and WAL accumulates, your database disk can fill up and cause an outage.

---

## What's Next?

Now that you understand how to capture changes in real-time from your production databases, the next step is putting it all together: [Chapter 20.5: Real-Time Analytics Architecture](./05-real-time-analytics.md) covers how to build end-to-end systems that process millions of events per second and serve live dashboards with sub-second latency.
