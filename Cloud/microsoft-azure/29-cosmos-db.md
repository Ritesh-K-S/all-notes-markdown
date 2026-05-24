# Chapter 29: Azure Cosmos DB

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cosmos DB Fundamentals](#part-1-cosmos-db-fundamentals)
- [Part 2: Creating a Cosmos DB Account (Full Portal Walkthrough)](#part-2-creating-a-cosmos-db-account-full-portal-walkthrough)
- [Part 3: Request Units (RUs) & Throughput](#part-3-request-units-rus--throughput)
- [Part 4: Partitioning](#part-4-partitioning)
- [Part 5: Consistency Levels](#part-5-consistency-levels)
- [Part 6: Global Distribution](#part-6-global-distribution)
- [Part 7: APIs (NoSQL, MongoDB, Cassandra, Gremlin, Table)](#part-7-apis)
- [Part 8: Working with Data (Portal)](#part-8-working-with-data-portal)
- [Part 9: Terraform & Bicep](#part-9-terraform--bicep)
- [Part 10: az CLI Reference](#part-10-az-cli-reference)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Cosmos DB is a globally distributed, multi-model NoSQL database service. It guarantees single-digit millisecond response times anywhere in the world. Think of it as a super-fast, globally replicated database that can handle millions of requests per second — perfect for apps that need to work fast everywhere.

```
What you'll learn:
├── Cosmos DB Fundamentals
│   ├── What is Cosmos DB (globally distributed NoSQL)
│   ├── Multi-model APIs (NoSQL, MongoDB, Cassandra, etc.)
│   ├── Request Units (RU) — how you pay for throughput
│   └── Cosmos DB vs Azure SQL vs other databases
├── Creating a Cosmos DB Account (Portal Walkthrough)
├── Request Units (RUs) — understanding throughput
├── Partitioning (how Cosmos scales)
├── Consistency Levels (5 levels explained simply)
├── Global Distribution (multi-region writes)
├── APIs (NoSQL, MongoDB, Cassandra, Gremlin, Table)
├── Working with Data (create/query in Portal)
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: Cosmos DB Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           COSMOS DB CONCEPT                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Cosmos DB?                                                  │
│ ├── Globally distributed NoSQL database                           │
│ ├── Single-digit millisecond reads AND writes                    │
│ ├── 99.999% availability SLA (five nines!)                       │
│ ├── Automatic scaling (serverless or provisioned)                │
│ ├── Multiple data models (document, key-value, graph, column)   │
│ ├── Turn-key global distribution (replicate to any Azure region)│
│ └── Schema-free (store any JSON document)                        │
│                                                                       │
│ Hierarchy:                                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Cosmos DB Account: cosmos-myapp-prod                        │  │
│ │ ├── API: NoSQL (or MongoDB, Cassandra, Gremlin, Table)    │  │
│ │ ├── Regions: Central India (write), West Europe (read)    │  │
│ │ │                                                           │  │
│ │ ├── Database: myapp-db                                     │  │
│ │ │   ├── Container: users (partition key: /userId)         │  │
│ │ │   │   ├── Item: { "userId": "U1", "name": "John" }    │  │
│ │ │   │   └── Item: { "userId": "U2", "name": "Jane" }    │  │
│ │ │   │                                                       │  │
│ │ │   ├── Container: orders (partition key: /customerId)   │  │
│ │ │   │   └── Item: { "orderId": "O1", "customerId": "C1"}│  │
│ │ │   │                                                       │  │
│ │ │   └── Container: products (partition key: /category)   │  │
│ │ │                                                           │  │
│ │ └── Throughput: 4000 RU/s (shared or per container)       │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Terminology mapping:                                                 │
│ ┌──────────────┬──────────────┬──────────────┬──────────────┐    │
│ │ Cosmos DB    │ SQL Database │ MongoDB       │ DynamoDB     │    │
│ ├──────────────┼──────────────┼──────────────┼──────────────┤    │
│ │ Account      │ Server       │ Cluster       │ Account      │    │
│ │ Database     │ Database     │ Database      │ (implicit)   │    │
│ │ Container    │ Table        │ Collection    │ Table        │    │
│ │ Item         │ Row          │ Document      │ Item         │    │
│ │ Partition key│ (no equiv.)  │ Shard key    │ Partition key│    │
│ └──────────────┴──────────────┴──────────────┴──────────────┘    │
│                                                                       │
│ ⚡ AWS equivalent: DynamoDB (NoSQL) + DocumentDB (MongoDB)         │
│ ⚡ GCP equivalent: Firestore + Bigtable                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Cosmos DB Account (Full Portal Walkthrough)

```
Console → Azure Cosmos DB → Create → API for NoSQL

┌─────────────────────────────────────────────────────────────────────┐
│           BASICS                                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-cosmosdb-prod ▼]                               │
│                                                                       │
│ Account name: [cosmos-myapp-prod]                                  │
│ ⚡ Globally unique! Creates: cosmos-myapp-prod.documents.azure.com│
│                                                                       │
│ Location: [Central India ▼]                                        │
│ ⚡ Primary (write) region. Add read regions later.                │
│                                                                       │
│ Capacity mode:                                                       │
│ ● Provisioned throughput (predictable workloads)                  │
│   Set specific RU/s, pay whether you use them or not.            │
│ ○ Serverless (unpredictable/sporadic workloads)                  │
│   Pay per RU consumed. Max 5000 RU/s. Single region only.       │
│                                                                       │
│ Apply Free Tier Discount: ● Apply  ○ Do not apply               │
│ ⚡ Free tier: 1000 RU/s + 25 GB storage FREE!                    │
│   Only 1 free tier account per Azure subscription.               │
│                                                                       │
│ [Next: Global Distribution >]                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           GLOBAL DISTRIBUTION                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Geo-Redundancy: ● Enable  ○ Disable                               │
│ ⚡ Adds a read replica in paired region automatically.            │
│                                                                       │
│ Multi-region Writes: ○ Enable  ● Disable                          │
│ ⚡ Enable = Write to ANY region (active-active).                  │
│   Disable = Write only to primary region.                        │
│   Enable costs more (charged per region with RU/s).              │
│                                                                       │
│ Availability Zones: ☑ Enable                                      │
│ ⚡ Spreads replicas across AZs in each region.                    │
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           NETWORKING                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Connectivity method:                                                 │
│ ○ All networks                                                     │
│ ● Private endpoint ← recommended for production                 │
│ ○ Public endpoint (selected networks)                             │
│                                                                       │
│ [+ Add private endpoint]                                           │
│ Name: pe-cosmos-myapp                                              │
│ Sub-resource: Sql (for NoSQL API)                                  │
│ VNet: vnet-prod, Subnet: subnet-endpoints                         │
│                                                                       │
│ [Next: Backup Policy >]                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           BACKUP POLICY                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Backup policy:                                                       │
│ ● Periodic (default — automatic every 4 hours)                   │
│   Backup interval: [240] minutes (4 hours)                       │
│   Retention: [8] hours (minimum = 2 × interval)                 │
│   Copies: [2]                                                     │
│ ○ Continuous (30 days or 7 days PITR)                            │
│   ⚡ Point-in-time restore to any second!                        │
│   More expensive but much better for production.                 │
│                                                                       │
│ Backup storage redundancy:                                           │
│ ○ Locally-redundant                                               │
│ ● Geo-redundant (recommended)                                     │
│ ○ Zone-redundant                                                  │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Request Units (RUs) & Throughput

```
┌─────────────────────────────────────────────────────────────────────┐
│           REQUEST UNITS (RUs)                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is an RU?                                                       │
│ ├── A single unit of database throughput                           │
│ ├── 1 RU = Reading 1 item (1 KB) by its ID (point read)        │
│ ├── Writing costs ~5x more than reading                          │
│ ├── Queries cost depends on complexity                            │
│ └── You provision RU/s (per second) for your container           │
│                                                                       │
│ Cost examples:                                                       │
│ ┌──────────────────────────────────────┬─────────────────┐        │
│ │ Operation                            │ Approximate RU  │        │
│ ├──────────────────────────────────────┼─────────────────┤        │
│ │ Point read (1 KB item by id)         │ 1 RU            │        │
│ │ Point read (10 KB item)              │ ~3 RU           │        │
│ │ Insert (1 KB item)                   │ ~5 RU           │        │
│ │ Update (1 KB item)                   │ ~10 RU          │        │
│ │ Simple query (10 results, indexed)   │ ~10-30 RU       │        │
│ │ Complex query (cross-partition)      │ ~100+ RU        │        │
│ └──────────────────────────────────────┴─────────────────┘        │
│                                                                       │
│ Throughput options:                                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ 1. Manual provisioned: Set exact RU/s (400-unlimited)      │  │
│ │    Example: 4000 RU/s → ~$230/month                        │  │
│ │    ⚡ Cheapest if workload is predictable.                  │  │
│ │                                                              │  │
│ │ 2. Autoscale: Set max RU/s, scales between 10%-100%       │  │
│ │    Example: Max 4000 → scales between 400-4000 RU/s       │  │
│ │    ⚡ Pay for actual usage. Good for variable workloads.   │  │
│ │                                                              │  │
│ │ 3. Serverless: No provisioning, pay per RU consumed        │  │
│ │    Max 5000 RU/s. Single region only.                      │  │
│ │    ⚡ Cheapest for low/sporadic traffic.                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Throughput scope:                                                     │
│ ├── Database-level: Shared across all containers (cost-effective)│
│ └── Container-level: Dedicated to one container (isolated perf) │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Partitioning

```
┌─────────────────────────────────────────────────────────────────────┐
│           PARTITIONING                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Partition key = THE most important decision in Cosmos DB!           │
│                                                                       │
│ How it works:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Container: orders (partition key: /customerId)              │  │
│ │                                                              │  │
│ │ Logical Partition "C1":       Logical Partition "C2":       │  │
│ │ ┌─────────────────────┐     ┌─────────────────────┐       │  │
│ │ │ customerId: "C1"    │     │ customerId: "C2"    │       │  │
│ │ │ order1, order2, ... │     │ order3, order4, ... │       │  │
│ │ └─────────────────────┘     └─────────────────────┘       │  │
│ │                                                              │  │
│ │ Physical Partition 1:        Physical Partition 2:          │  │
│ │ ┌─────────────────────┐     ┌─────────────────────┐       │  │
│ │ │ Contains: C1-C500   │     │ Contains: C501-C1000│       │  │
│ │ │ Max 20 GB per LP    │     │ Max 20 GB per LP    │       │  │
│ │ │ Max 50 GB per PP    │     │ Max 50 GB per PP    │       │  │
│ │ └─────────────────────┘     └─────────────────────┘       │  │
│ │                                                              │  │
│ │ ⚡ Cosmos splits physical partitions as data grows.         │  │
│ │ ⚡ Queries on partition key are FAST (single partition).    │  │
│ │ ⚡ Cross-partition queries are SLOW (scan all partitions).  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Choosing a good partition key:                                       │
│ ├── High cardinality (many distinct values) ✅                   │
│ ├── Even data distribution (no "hot" partitions) ✅              │
│ ├── Used in most queries (WHERE clause) ✅                      │
│ ├── Examples:                                                     │
│ │   Users container → /userId ✅                               │
│ │   Orders container → /customerId ✅                          │
│ │   Products container → /category ✅                          │
│ │   Logs container → /date ⚠️ (could be uneven)               │
│ │   ALL items → /status ❌ (very few values = hot partition)  │
│ └── ⚠️ Partition key CANNOT be changed after creation!          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Consistency Levels

```
┌─────────────────────────────────────────────────────────────────────┐
│           5 CONSISTENCY LEVELS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cosmos DB offers 5 consistency levels (unique feature!):            │
│                                                                       │
│ Strongest ←──────────────────────────────────────→ Weakest         │
│ Strong     Bounded    Session    Consistent    Eventual             │
│            Staleness  (default)  Prefix                             │
│                                                                       │
│ ┌──────────────────┬────────────────────────────────────────────┐  │
│ │ Level            │ Simple Explanation                          │  │
│ ├──────────────────┼────────────────────────────────────────────┤  │
│ │ Strong           │ Everyone always sees the latest write.     │  │
│ │                  │ Like a single database. Highest latency.   │  │
│ │                  │ Use: Financial transactions, inventory.    │  │
│ │                  │                                            │  │
│ │ Bounded Staleness│ Reads might be behind writes by max N     │  │
│ │                  │ seconds or N operations. Predictable lag. │  │
│ │                  │ Use: Leaderboards, counters.              │  │
│ │                  │                                            │  │
│ │ Session (default)│ Within ONE session, reads see your writes.│  │
│ │                  │ Other sessions might see older data.      │  │
│ │                  │ Use: Most apps! (shopping cart, profile). │  │
│ │                  │ ⚡ RECOMMENDED for most applications.     │  │
│ │                  │                                            │  │
│ │ Consistent Prefix│ Reads never see out-of-order writes.      │  │
│ │                  │ If writes are A→B→C, reads see A,AB,ABC. │  │
│ │                  │ Use: Social media feeds.                   │  │
│ │                  │                                            │  │
│ │ Eventual         │ Eventually all replicas converge.         │  │
│ │                  │ Reads might see old data temporarily.     │  │
│ │                  │ Cheapest (lowest RU cost). Highest speed. │  │
│ │                  │ Use: Counters, likes, non-critical data.  │  │
│ └──────────────────┴────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ Default: Session consistency (best for most apps!)              │
│ ⚡ Stronger = Higher latency + Higher RU cost                      │
│ ⚡ Weaker = Lower latency + Lower RU cost                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Global Distribution

```
┌─────────────────────────────────────────────────────────────────────┐
│           GLOBAL DISTRIBUTION                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Add/remove regions with a single click!                              │
│                                                                       │
│ Portal → Cosmos DB Account → Replicate data globally             │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ [World Map showing regions]                                  │  │
│ │                                                              │  │
│ │ Write region: Central India ★                               │  │
│ │ Read regions:                                                │  │
│ │ ├── West Europe (added) ●                                  │  │
│ │ ├── East US (added) ●                                      │  │
│ │ └── Southeast Asia (added) ●                               │  │
│ │                                                              │  │
│ │ [+ Add region]  [Save]                                      │  │
│ │                                                              │  │
│ │ ⚡ Each read region adds cost (same RU/s replicated).      │  │
│ │   If you provision 4000 RU/s in 4 regions = charged for    │  │
│ │   16,000 RU/s total.                                        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Multi-region writes (active-active):                                 │
│ ├── Write to ANY region (closest to user)                        │
│ ├── Conflict resolution: Last Writer Wins (default) or custom   │
│ ├── Even lower write latency globally                            │
│ └── Higher cost and complexity                                   │
│                                                                       │
│ Automatic failover:                                                  │
│ ├── If write region fails → auto-promotes read region           │
│ ├── Priority-based (you set the order)                           │
│ ├── Transparent to application (connection retries)             │
│ └── Enable: Portal → Account → Automatic failover → Enable   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: APIs

```
┌─────────────────────────────────────────────────────────────────────┐
│           COSMOS DB APIs                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Choose API when creating the account (cannot change after!):        │
│                                                                       │
│ ┌──────────────┬─────────────────────────────────────────────────┐ │
│ │ API          │ Description                                     │ │
│ ├──────────────┼─────────────────────────────────────────────────┤ │
│ │ NoSQL        │ Native Cosmos API. JSON documents + SQL-like   │ │
│ │ (recommended)│ queries. Best performance & features.          │ │
│ │              │ Query: SELECT * FROM c WHERE c.name = 'John'  │ │
│ │              │                                                 │ │
│ │ MongoDB      │ MongoDB-compatible. Use existing MongoDB       │ │
│ │              │ drivers/tools. Good for MongoDB migrations.   │ │
│ │              │                                                 │ │
│ │ Cassandra    │ Cassandra-compatible (CQL). Wide-column store.│ │
│ │              │ Good for Cassandra migrations.                 │ │
│ │              │                                                 │ │
│ │ Gremlin      │ Graph database. For relationships/networks.   │ │
│ │              │ Social graphs, recommendations, fraud detection│ │
│ │              │                                                 │ │
│ │ Table        │ Azure Table Storage compatible (improved).     │ │
│ │              │ Simple key-value. Table Storage migration.     │ │
│ └──────────────┴─────────────────────────────────────────────────┘ │
│                                                                       │
│ ⚡ For new projects: Always use NoSQL API (best features, best perf)│
│ ⚡ For migrations: Use the API matching your existing database.    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Working with Data (Portal)

```
Portal → Cosmos DB Account → Data Explorer

┌─────────────────────────────────────────────────────────────────────┐
│           DATA EXPLORER                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Create Database:                                                     │
│ [New Database]                                                       │
│ Database id: [myapp-db]                                             │
│ ☑ Share throughput across containers                              │
│ Database throughput (autoscale): Max RU/s [4000]                  │
│                                                                       │
│ Create Container:                                                    │
│ [New Container]                                                      │
│ Database id: [myapp-db]                                             │
│ Container id: [users]                                               │
│ Partition key: [/userId]                                           │
│ ⚡ Choose carefully! Cannot change after creation.                 │
│                                                                       │
│ Add Item (document):                                                 │
│ [New Item]                                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ {                                                            │  │
│ │   "id": "user-001",                                        │  │
│ │   "userId": "U001",                                        │  │
│ │   "name": "John Doe",                                      │  │
│ │   "email": "john@example.com",                             │  │
│ │   "age": 30,                                                │  │
│ │   "address": {                                              │  │
│ │     "city": "Mumbai",                                      │  │
│ │     "country": "India"                                     │  │
│ │   },                                                        │  │
│ │   "tags": ["premium", "verified"]                          │  │
│ │ }                                                            │  │
│ │                                                              │  │
│ │ ⚡ "id" = unique within partition (required!)               │  │
│ │ ⚡ Partition key value must be present (/userId = "U001")  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│ [Save]                                                               │
│                                                                       │
│ Query Items:                                                         │
│ [New SQL Query]                                                      │
│ SELECT * FROM c WHERE c.address.city = 'Mumbai'                    │
│ [Execute Query] → Shows matching documents                        │
│                                                                       │
│ ⚡ Query charges: See "Request Charge" in results (RU cost).     │
│                                                                       │
│ Delete Item:                                                         │
│ Click item → [...] → Delete                                      │
│                                                                       │
│ Delete Container:                                                    │
│ Right-click container → Delete Container                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_cosmosdb_account" "main" {
  name                = "cosmos-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"  # NoSQL API

  consistency_policy {
    consistency_level = "Session"
  }

  geo_location {
    location          = "centralindia"
    failover_priority = 0  # Primary
  }

  geo_location {
    location          = "westeurope"
    failover_priority = 1  # Secondary
  }
}

resource "azurerm_cosmosdb_sql_database" "main" {
  name                = "myapp-db"
  resource_group_name = azurerm_resource_group.main.name
  account_name        = azurerm_cosmosdb_account.main.name

  autoscale_settings {
    max_throughput = 4000
  }
}

resource "azurerm_cosmosdb_sql_container" "users" {
  name                = "users"
  resource_group_name = azurerm_resource_group.main.name
  account_name        = azurerm_cosmosdb_account.main.name
  database_name       = azurerm_cosmosdb_sql_database.main.name
  partition_key_paths = ["/userId"]
}
```

### Bicep

```bicep
// Cosmos DB Account
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2023-11-15' = {
  name: 'cosmos-myapp-prod'
  location: resourceGroup().location
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
    }
    locations: [
      { locationName: 'centralindia', failoverPriority: 0, isZoneRedundant: true }
      { locationName: 'westeurope', failoverPriority: 1, isZoneRedundant: false }
    ]
    enableAutomaticFailover: true
  }
}

// SQL Database
resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2023-11-15' = {
  parent: cosmosAccount
  name: 'myapp-db'
  properties: {
    resource: { id: 'myapp-db' }
    options: { autoscaleSettings: { maxThroughput: 4000 } }
  }
}

// SQL Container
resource container 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2023-11-15' = {
  parent: cosmosDb
  name: 'users'
  properties: {
    resource: {
      id: 'users'
      partitionKey: { paths: ['/userId'], kind: 'Hash' }
      indexingPolicy: {
        automatic: true
        indexingMode: 'consistent'
      }
    }
  }
}
```
# Create Cosmos DB account
az cosmosdb create \
  --name cosmos-myapp-prod \
  --resource-group rg-cosmosdb-prod \
  --locations regionName=centralindia failoverPriority=0 \
  --locations regionName=westeurope failoverPriority=1 \
  --default-consistency-level Session \
  --enable-automatic-failover true

# Create database
az cosmosdb sql database create \
  --account-name cosmos-myapp-prod \
  --resource-group rg-cosmosdb-prod \
  --name myapp-db \
  --max-throughput 4000

# Create container
az cosmosdb sql container create \
  --account-name cosmos-myapp-prod \
  --resource-group rg-cosmosdb-prod \
  --database-name myapp-db \
  --name users \
  --partition-key-path "/userId"

# List databases
az cosmosdb sql database list \
  --account-name cosmos-myapp-prod \
  --resource-group rg-cosmosdb-prod \
  --output table

# Get connection keys
az cosmosdb keys list \
  --name cosmos-myapp-prod \
  --resource-group rg-cosmosdb-prod

# Update throughput
az cosmosdb sql database throughput update \
  --account-name cosmos-myapp-prod \
  --resource-group rg-cosmosdb-prod \
  --name myapp-db \
  --max-throughput 8000

# Add region
az cosmosdb update \
  --name cosmos-myapp-prod \
  --resource-group rg-cosmosdb-prod \
  --locations regionName=centralindia failoverPriority=0 \
  --locations regionName=westeurope failoverPriority=1 \
  --locations regionName=eastus failoverPriority=2

# Delete container
az cosmosdb sql container delete \
  --account-name cosmos-myapp-prod \
  --resource-group rg-cosmosdb-prod \
  --database-name myapp-db \
  --name users --yes

# Delete account
az cosmosdb delete \
  --name cosmos-myapp-prod \
  --resource-group rg-cosmosdb-prod --yes
```

---

## Part 11: Real-World Patterns

```
Pattern 1: E-Commerce (Session Cart + User Profiles)
├── Account: cosmos-ecom-prod (NoSQL API, Session consistency)
├── Database: ecom-db (autoscale 4000 RU/s)
├── Container: users (partition key: /userId)
├── Container: cart (partition key: /userId, TTL: 7 days)
├── Container: orders (partition key: /customerId)
├── Regions: Central India (write) + West Europe (read)
└── Access: App Service managed identity

Pattern 2: IoT Telemetry (High Ingestion)
├── Account: cosmos-iot-prod (NoSQL API, Eventual consistency)
├── Container: telemetry (partition key: /deviceId)
├── Throughput: Autoscale max 40,000 RU/s
├── TTL: 30 days (auto-delete old data)
├── Change feed → Azure Functions → Process in real-time
└── Archive: Change feed → ADLS Gen2 (cold storage)
```

---

## Quick Reference

```
Cosmos DB = Globally distributed NoSQL (single-digit ms latency)
Account → Database → Container → Item (document)
Partition Key = Most important decision! Cannot change after creation.

Throughput: Manual RU/s | Autoscale RU/s | Serverless
1 RU = 1 point read of 1 KB item
Consistency: Strong > Bounded > Session (default) > Prefix > Eventual

APIs: NoSQL (best) | MongoDB | Cassandra | Gremlin | Table
Global: Click to add/remove regions
Failover: Automatic (priority-based)

Free tier: 1000 RU/s + 25 GB (1 per subscription)
```

---

## What's Next?

Next chapter: [Chapter 30: Azure Cache for Redis](30-azure-cache-redis.md) — In-memory caching for ultra-fast data access.
