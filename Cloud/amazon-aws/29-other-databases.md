# Chapter 29: DocumentDB, Neptune & Other Databases

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Amazon DocumentDB](#part-1-amazon-documentdb)
- [Part 2: Amazon Neptune](#part-2-amazon-neptune)
- [Part 3: Amazon Timestream](#part-3-amazon-timestream)
- [Part 4: Amazon QLDB](#part-4-amazon-qldb)
- [Part 5: Amazon Keyspaces](#part-5-amazon-keyspaces)
- [Part 6: Amazon MemoryDB for Redis](#part-6-amazon-memorydb-for-redis)
- [Part 7: Database Decision Guide](#part-7-database-decision-guide)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### Why Purpose-Built Databases?

In the early days of computing, companies used one database for everything — usually Oracle or MySQL. But trying to use a relational database for every use case is like using a hammer for every job — it works, but a screwdriver is much better for screws.

**AWS offers purpose-built databases** — each optimized for a specific data model and access pattern:
- **Graph data** (social networks, fraud detection) → **Neptune** — like mapping friendships on a whiteboard
- **Document data** (content management, user profiles) → **DocumentDB** — MongoDB-compatible
- **Time-series data** (IoT sensors, stock prices) → **Timestream** — optimized for data that changes over time
- **Ledger data** (supply chain, financial records) → **QLDB** — immutable, verifiable history
- **Wide-column data** (ad-tech, IoT at massive scale) → **Keyspaces** — Apache Cassandra-compatible

**Rule of thumb:** Start with RDS/Aurora for relational data or DynamoDB for key-value data. Only use a specialized database when your use case specifically demands it.

AWS offers purpose-built databases for specific workloads beyond RDS, Aurora, DynamoDB, and ElastiCache. This chapter covers specialized databases and when to choose each.

```
What you'll learn:
├── DocumentDB (MongoDB compatible)
├── Neptune (Graph database)
├── Timestream (Time-series database)
├── QLDB (Ledger/immutable database)
├── Keyspaces (Cassandra compatible)
├── MemoryDB for Redis (durable in-memory)
└── Decision guide: Which database for which use case
```

---

## Part 1: Amazon DocumentDB

```
Console → DocumentDB → Clusters → Create

┌─────────────────────────────────────────────────────────────────┐
│           AMAZON DOCUMENTDB                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Managed document database, MongoDB 4.0/5.0 compatible.  │
│                                                                   │
│ Cluster identifier: [prod-docdb]                               │
│ Engine version: [5.0.0 ▼]                                     │
│ Instance class: [db.r6g.large ▼]                               │
│ Number of instances: [3] (1 primary + 2 replicas)             │
│                                                                   │
│ VPC: [vpc-main ▼]                                              │
│ Subnet group: [docdb-private ▼]                               │
│ Security group: [sg-docdb ▼] (port 27017)                     │
│                                                                   │
│ Authentication: Username/password                              │
│ Encryption at rest: ☑  Encryption in transit: ☑              │
│ Backup retention: [7] days                                    │
│                                                                   │
│ Key features:                                                   │
│ ├── MongoDB API compatible (use MongoDB drivers/tools)       │
│ ├── Shared storage across instances (like Aurora)            │
│ ├── Up to 15 read replicas                                    │
│ ├── Auto-scales storage: 10 GB to 128 TB                     │
│ ├── ACID transactions                                         │
│ ├── Automatic failover (<30 seconds)                         │
│ └── ⚠️ Not 100% MongoDB compatible (check compatibility)     │
│                                                                   │
│ When to use:                                                    │
│ ├── Migrating from MongoDB to managed AWS                    │
│ ├── Document/JSON data model                                  │
│ ├── Content management, catalogs, user profiles              │
│ └── Need MongoDB API but want managed infrastructure         │
│                                                                   │
│ When NOT to use:                                                │
│ ├── Need full MongoDB feature parity → use MongoDB Atlas     │
│ ├── Simple key-value → DynamoDB                               │
│ ├── Relational data → RDS/Aurora                              │
│ └── Need MongoDB change streams → limited in DocumentDB     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Amazon Neptune

```
Console → Neptune → Databases → Create database

┌─────────────────────────────────────────────────────────────────┐
│           AMAZON NEPTUNE                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Managed graph database. Supports property graph          │
│ (Gremlin/openCypher) and RDF (SPARQL) query languages.        │
│                                                                   │
│ DB cluster identifier: [prod-neptune]                          │
│ Engine version: [1.3.1.0 ▼]                                   │
│                                                                   │
│ Instance class: [db.r6g.large ▼]                               │
│ Number of replicas: [2]                                        │
│                                                                   │
│ Serverless: ☑ Enable Neptune Serverless                       │
│   Min capacity: [1] NCU  Max capacity: [32] NCU               │
│ → Auto-scales compute based on workload                       │
│                                                                   │
│ Key features:                                                   │
│ ├── Property graph (Gremlin, openCypher)                     │
│ ├── RDF graph (SPARQL)                                        │
│ ├── Up to 15 read replicas                                    │
│ ├── Shared storage (like Aurora): 10 GB to 128 TB            │
│ ├── ACID transactions                                         │
│ ├── Neptune ML: ML predictions on graph data                 │
│ ├── Neptune Analytics: Analytical queries on graphs           │
│ └── Full-text search integration                              │
│                                                                   │
│ When to use:                                                    │
│ ├── Social networks (friends, followers, recommendations)    │
│ ├── Fraud detection (transaction patterns)                   │
│ ├── Knowledge graphs (entities + relationships)              │
│ ├── Network management (IT infrastructure relationships)    │
│ ├── Identity graphs (user identity resolution)               │
│ └── Recommendation engines                                    │
│                                                                   │
│ Example Gremlin query:                                         │
│ // Find friends of friends who like "AWS"                     │
│ g.V('user-001').out('friends').out('friends')                 │
│  .has('interests', 'AWS').dedup().limit(10)                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Amazon Timestream

```
┌─────────────────────────────────────────────────────────────────────┐
│           AMAZON TIMESTREAM                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Serverless time-series database. Optimized for              │
│ timestamped data (IoT sensors, metrics, app events).              │
│                                                                       │
│ Console → Timestream → Databases → Create database                │
│ Database name: [iot-metrics]                                       │
│                                                                       │
│ Create table:                                                       │
│ Table name: [sensor-readings]                                      │
│ Memory store retention: [24] hours (hot data, fast queries)       │
│ Magnetic store retention: [365] days (warm data, cheaper)         │
│                                                                       │
│ Key features:                                                        │
│ ├── Serverless (no capacity provisioning)                        │
│ ├── Automatic data tiering (memory → magnetic)                  │
│ ├── Built-in time-series functions                               │
│ ├── SQL-compatible query language                                │
│ ├── 1000x faster, 1/10th cost vs relational DB for time-series │
│ ├── Trillions of events per day                                  │
│ ├── Built-in interpolation, aggregation, smoothing              │
│ └── Integrates with Grafana for dashboards                       │
│                                                                       │
│ When to use:                                                         │
│ ├── IoT sensor data                                               │
│ ├── Application/infrastructure monitoring                        │
│ ├── DevOps metrics                                                │
│ ├── Industrial telemetry                                          │
│ └── Real-time analytics on time-stamped data                     │
│                                                                       │
│ Example query:                                                       │
│ SELECT device_id, AVG(temperature)                                │
│ FROM sensor_readings                                               │
│ WHERE time BETWEEN ago(1h) AND now()                              │
│ GROUP BY device_id                                                 │
│ HAVING AVG(temperature) > 80                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Amazon QLDB

```
┌─────────────────────────────────────────────────────────────────────┐
│           AMAZON QLDB (Quantum Ledger Database)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Immutable, transparent, cryptographically verifiable        │
│ ledger database. Think "blockchain without decentralization."     │
│                                                                       │
│ ⚠️ Note: QLDB is no longer accepting new customers (as of 2024). │
│ Consider alternatives: DynamoDB + Streams, Aurora for audit logs.│
│                                                                       │
│ Key concepts:                                                        │
│ ├── Immutable: Data can never be deleted or modified              │
│ ├── Cryptographic verification: Hash chain proves no tampering   │
│ ├── Complete audit trail: Full history of every change           │
│ ├── PartiQL query language (SQL-like)                            │
│ ├── Document data model (JSON)                                    │
│ ├── ACID transactions                                             │
│ └── Serverless (pay-per-use)                                     │
│                                                                       │
│ Was used for:                                                        │
│ ├── Financial transactions (audit trail)                         │
│ ├── Supply chain tracking                                        │
│ ├── Insurance claims history                                      │
│ ├── HR/payroll records                                            │
│ └── Any use case requiring provable data integrity               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Amazon Keyspaces

```
┌─────────────────────────────────────────────────────────────────────┐
│           AMAZON KEYSPACES (for Apache Cassandra)                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed Cassandra-compatible database. CQL (Cassandra      │
│ Query Language) compatible. Serverless.                            │
│                                                                       │
│ Console → Keyspaces → Create keyspace → Create table              │
│                                                                       │
│ Keyspace name: [my_application]                                    │
│ Table name: [user_events]                                          │
│ Columns: user_id (text, partition key), event_time (timestamp,   │
│          clustering key), event_type (text), details (text)       │
│                                                                       │
│ Capacity mode: ● On-demand ○ Provisioned                         │
│                                                                       │
│ Key features:                                                        │
│ ├── CQL compatible (use existing Cassandra drivers/tools)       │
│ ├── Serverless (no cluster management)                           │
│ ├── On-demand or provisioned capacity                            │
│ ├── Encryption at rest + in transit                              │
│ ├── Point-in-time recovery                                        │
│ ├── Multi-region replication                                     │
│ └── 99.999% availability SLA                                     │
│                                                                       │
│ When to use:                                                         │
│ ├── Migrating from Cassandra to managed AWS                     │
│ ├── Wide-column data model                                        │
│ ├── High write throughput                                         │
│ ├── Large-scale distributed data                                 │
│ └── ⚠️ Not 100% Cassandra compatible (check features)            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Amazon MemoryDB for Redis

```
┌─────────────────────────────────────────────────────────────────────┐
│           AMAZON MEMORYDB FOR REDIS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Redis-compatible, durable, in-memory database.              │
│ Unlike ElastiCache (cache), MemoryDB is a PRIMARY database.      │
│                                                                       │
│ ElastiCache vs MemoryDB:                                            │
│ ┌───────────────────┬──────────────────┬──────────────────────────┐│
│ │                   │ ElastiCache      │ MemoryDB               ││
│ ├───────────────────┼──────────────────┼──────────────────────────┤│
│ │ Purpose           │ Cache layer      │ Primary database        ││
│ │ Durability        │ Best-effort      │ Durable (transaction    ││
│ │                   │                  │ log across AZs)         ││
│ │ Data loss on fail │ Possible         │ No data loss (RPO=0)    ││
│ │ Use as sole DB    │ No (need source) │ Yes (durable)           ││
│ │ Latency           │ Microseconds     │ Microseconds (reads),   ││
│ │                   │                  │ low ms (writes)          ││
│ │ Pricing           │ Lower            │ ~20% more than EC       ││
│ └───────────────────┴──────────────────┴──────────────────────────┘│
│                                                                       │
│ When to use MemoryDB:                                               │
│ ├── Redis as your primary database (not just cache)              │
│ ├── Need durability + microsecond reads                          │
│ ├── Session stores that can't lose data                          │
│ └── Real-time applications needing Redis data structures         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Database Decision Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│           AWS DATABASE DECISION GUIDE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ "I need a relational database"                                      │
│ ├── Standard SQL, managed → RDS (MySQL, PostgreSQL, etc.)       │
│ ├── High performance, auto-scaling → Aurora                     │
│ └── Serverless relational → Aurora Serverless v2                │
│                                                                       │
│ "I need a NoSQL database"                                           │
│ ├── Key-value / document, serverless → DynamoDB                 │
│ ├── MongoDB compatible → DocumentDB                              │
│ └── Cassandra compatible → Keyspaces                             │
│                                                                       │
│ "I need in-memory / caching"                                        │
│ ├── Cache layer (not primary DB) → ElastiCache Redis            │
│ ├── Redis as primary DB (durable) → MemoryDB for Redis         │
│ └── DynamoDB read cache → DAX                                    │
│                                                                       │
│ "I need a specialized database"                                     │
│ ├── Graph data (relationships) → Neptune                        │
│ ├── Time-series (IoT, metrics) → Timestream                    │
│ ├── Immutable ledger → QLDB (deprecated) / DynamoDB + audit    │
│ └── Data warehouse → Redshift (Chapter 54)                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
AWS Specialized Databases:
├── DocumentDB: MongoDB compatible, managed, shared storage
├── Neptune: Graph DB (Gremlin, openCypher, SPARQL)
├── Timestream: Time-series, serverless, IoT/metrics
├── QLDB: Immutable ledger (deprecated for new customers)
├── Keyspaces: Cassandra compatible, serverless
├── MemoryDB: Durable Redis (primary DB, not just cache)
└── Decision: Match database type to data model + access pattern
```

---

## What's Next?

In **Chapter 30: CodeCommit**, we'll begin the CI/CD & Developer Tools section — covering AWS's managed Git repositories, branches, pull requests, and integration with the AWS CI/CD pipeline.
