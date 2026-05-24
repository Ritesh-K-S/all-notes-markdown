# Chapter 55: Azure Service Fabric

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Service Fabric Fundamentals](#part-1-service-fabric-fundamentals)
- [Part 2: Creating a Service Fabric Cluster (Portal Walkthrough)](#part-2-creating-a-service-fabric-cluster-portal-walkthrough)
- [Part 3: Application Model](#part-3-application-model)
- [Part 4: Stateful vs Stateless Services](#part-4-stateful-vs-stateless-services)
- [Part 5: Reliable Collections](#part-5-reliable-collections)
- [Part 6: Service Fabric vs AKS](#part-6-service-fabric-vs-aks)
- [Part 7: Terraform & az CLI Reference](#part-7-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Service Fabric is a distributed systems platform for building and managing microservices. It's the technology that powers many Azure services internally (Cosmos DB, SQL Database, Event Hubs, etc.). Its unique strength is built-in support for stateful services.

```
What you'll learn:
├── Service Fabric Fundamentals
│   ├── Cluster architecture (nodes, fault/upgrade domains)
│   ├── When to use Service Fabric
│   └── Managed vs unmanaged clusters
├── Creating a Cluster (Portal)
├── Application Model (apps, services, partitions, replicas)
├── Stateful vs Stateless Services
├── Reliable Collections (built-in state management)
├── Service Fabric vs AKS comparison
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Service Fabric Fundamentals

```
Service Fabric = Microservices platform with built-in state

Cluster architecture:
┌──────────────────────────────────────────────────────┐
│ Service Fabric Cluster                                │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│ │ Node 1   │ │ Node 2   │ │ Node 3   │             │
│ │ (VM)     │ │ (VM)     │ │ (VM)     │             │
│ │ FD0/UD0  │ │ FD1/UD1  │ │ FD2/UD2  │             │
│ │          │ │          │ │          │             │
│ │ [Svc A]  │ │ [Svc A]  │ │ [Svc B]  │             │
│ │ [Svc B]  │ │ [Svc C]  │ │ [Svc C]  │             │
│ └──────────┘ └──────────┘ └──────────┘             │
│                                                       │
│ FD = Fault Domain (hardware isolation)              │
│ UD = Upgrade Domain (rolling upgrades)              │
│ System places replicas across FDs for HA            │
└──────────────────────────────────────────────────────┘

Cluster types:
├── Service Fabric managed clusters (recommended)
│   └── Azure manages infrastructure, patching, scaling
├── Classic clusters (self-managed)
│   └── You manage VM scale sets, networking
└── Standalone (on-prem or other clouds)

When to use Service Fabric:
├── Need stateful microservices (built-in state, no external DB)
├── Low-latency, high-throughput scenarios
├── Actor model programming (virtual actors)
├── Legacy .NET Framework migration
└── Already using Service Fabric internally
```

---

## Part 2: Creating a Service Fabric Cluster (Portal Walkthrough)

```
Console → Service Fabric managed clusters → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE SERVICE FABRIC MANAGED CLUSTER                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-servicefabric ▼]                               │
│ Cluster name: [sf-myapp-prod]                                      │
│ Location: [Central India ▼]                                        │
│                                                                       │
│ SKU:                                                                  │
│ ○ Basic (dev/test, 3 nodes)                                       │
│ ● Standard (production, 5+ nodes)                                 │
│                                                                       │
│ Username: [sfadmin]                                                 │
│ Password: [••••••••••••]                                           │
│                                                                       │
│ Node Types:                                                          │
│ ├── Primary node type (system services)                           │
│ │   VM Size: [Standard_D2s_v3]                                   │
│ │   Instance count: [5]                                           │
│ └── Secondary node type (optional, for specific workloads)       │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Application Model

```
Application hierarchy:
├── Application Type (e.g., "OrderApp")
│   ├── Service Type 1 (e.g., "WebFrontend" — stateless)
│   │   ├── Instance 1 (Node 1)
│   │   ├── Instance 2 (Node 2)
│   │   └── Instance 3 (Node 3)
│   │
│   └── Service Type 2 (e.g., "OrderProcessor" — stateful)
│       ├── Partition 1:
│       │   ├── Primary Replica (Node 1) ← Writes here
│       │   ├── Secondary Replica (Node 2)
│       │   └── Secondary Replica (Node 3)
│       └── Partition 2:
│           ├── Primary Replica (Node 2)
│           ├── Secondary Replica (Node 1)
│           └── Secondary Replica (Node 3)

Partitioning:
├── Splits data/load across multiple replicas
├── Partition schemes:
│   ├── Singleton: One partition (simple services)
│   ├── UniformInt64Range: Key range (e.g., 0-1000)
│   └── Named: Named partitions (e.g., by region)
└── Each partition has its own primary + secondary replicas
```

---

## Part 4: Stateful vs Stateless Services

```
Stateless services:
├── No local state (like a typical web server)
├── State stored externally (database, cache)
├── Scale out by adding instances
├── Any instance can handle any request
└── Examples: Web API, gateway, proxy

Stateful services:
├── State stored WITH the service (in-process)
├── No external database needed!
├── Primary replica handles reads/writes
├── Secondary replicas replicate state automatically
├── Ultra-low latency (no network hop to database)
├── Partitioned for scalability
└── Examples: Shopping cart, session store, leaderboard

Comparison:
┌──────────────────┬─────────────────┬─────────────────┐
│                  │ Stateless       │ Stateful        │
├──────────────────┼─────────────────┼─────────────────┤
│ State storage    │ External DB     │ Built-in (local)│
│ Latency          │ DB round-trip   │ In-memory ⚡    │
│ Scaling          │ Add instances   │ Add partitions  │
│ Complexity       │ Simple          │ More complex    │
│ Data durability  │ DB handles it   │ Replication     │
│ Use case         │ Web APIs        │ Hot data, cache │
└──────────────────┴─────────────────┴─────────────────┘
```

---

## Part 5: Reliable Collections

```
Reliable Collections = Distributed, replicated data structures

Available collections:
├── ReliableDictionary<TKey, TValue>
│   Like a distributed ConcurrentDictionary
│   Replicated across nodes automatically
│
└── ReliableQueue<T>
    Like a distributed queue
    FIFO ordering guaranteed

Example (C#):
  var dictionary = await StateManager
    .GetOrAddAsync<IReliableDictionary<string, int>>("myDictionary");

  using (var tx = StateManager.CreateTransaction())
  {
      // Add/update value
      await dictionary.AddOrUpdateAsync(tx, "counter", 1, (key, val) => val + 1);
      await tx.CommitAsync(); // Replicated to all replicas!
  }

Properties:
├── Transactional: ACID transactions across collections
├── Replicated: Auto-copied to secondary replicas
├── Persisted: Survives node restarts (checkpointed to disk)
├── Partitioned: Each partition has its own collections
└── Consistent: Strong consistency within partition
```

---

## Part 6: Service Fabric vs AKS

```
┌──────────────────┬─────────────────────┬─────────────────────┐
│                  │ Service Fabric      │ AKS (Kubernetes)    │
├──────────────────┼─────────────────────┼─────────────────────┤
│ Container support│ Yes                 │ Yes (primary)       │
│ Stateful svc     │ Built-in ✅         │ External (DB/Redis) │
│ Programming model│ .NET SDK, actors    │ Any language/runtime│
│ Ecosystem        │ Microsoft           │ CNCF (huge)         │
│ Portability      │ Azure + on-prem     │ Any cloud/on-prem   │
│ Community        │ Smaller             │ Massive             │
│ Learning curve   │ Moderate            │ Steep               │
│ Managed offering │ SF Managed Clusters │ AKS (fully managed) │
│ When to choose   │ Stateful .NET, low  │ Container-first,    │
│                  │ latency, actors     │ multi-cloud, CNCF   │
└──────────────────┴─────────────────────┴─────────────────────┘

⚡ For new projects, AKS is generally recommended
⚡ Service Fabric is best for stateful .NET workloads
```

---

## Part 7: Terraform & az CLI Reference

### Terraform

```hcl
resource "azurerm_service_fabric_managed_cluster" "main" {
  name                = "sf-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard"
  username            = "sfadmin"
  password            = var.admin_password

  node_type {
    name                 = "primary"
    vm_size              = "Standard_D2s_v3"
    vm_instance_count    = 5
    primary              = true
    data_disk_size_gb    = 128
  }
}
```

### Bicep

```bicep
// Service Fabric Managed Cluster
resource sfCluster 'Microsoft.ServiceFabric/managedClusters@2023-11-01-preview' = {
  name: 'sf-myapp-prod'
  location: resourceGroup().location
  sku: { name: 'Standard' }
  properties: {
    adminUserName: 'sfadmin'
    adminPassword: adminPassword
    dnsName: 'sf-myapp-prod'
    clientConnectionPort: 19000
    httpGatewayConnectionPort: 19080
  }
}

// Node Type
resource nodeType 'Microsoft.ServiceFabric/managedClusters/nodeTypes@2023-11-01-preview' = {
  parent: sfCluster
  name: 'primary'
  properties: {
    isPrimary: true
    vmSize: 'Standard_D2s_v3'
    vmInstanceCount: 5
    dataDiskSizeGB: 128
  }
}
```

### az CLI

```bash
# Create managed cluster
az sf managed-cluster create \
  --cluster-name sf-myapp-prod \
  --resource-group rg-servicefabric \
  --location centralindia \
  --sku Standard \
  --admin-password "SecureP@ss123!"

# Add node type
az sf managed-node-type create \
  --cluster-name sf-myapp-prod \
  --resource-group rg-servicefabric \
  --node-type-name primary \
  --instance-count 5 \
  --vm-size Standard_D2s_v3 \
  --is-primary true

# List clusters
az sf managed-cluster list --resource-group rg-servicefabric -o table

# Show cluster details
az sf managed-cluster show --cluster-name sf-myapp-prod --resource-group rg-servicefabric

# Scale node type
az sf managed-node-type update \
  --cluster-name sf-myapp-prod \
  --resource-group rg-servicefabric \
  --node-type-name primary \
  --instance-count 7

# Deploy application
az sf managed-cluster application create \
  --cluster-name sf-myapp-prod \
  --resource-group rg-servicefabric \
  --application-name OrderApp \
  --application-type-name OrderAppType \
  --application-type-version 1.0

# Delete cluster
az sf managed-cluster delete \
  --cluster-name sf-myapp-prod \
  --resource-group rg-servicefabric --yes
```

---

## Real-World Patterns

### Pattern 1: Stateful Microservices for Gaming

```
┌─────────────────────────────────────────────────┐
│  Gaming Session Management                      │
├─────────────────────────────────────────────────┤
│                                                 │
│  Player connects ─→ Stateful Service            │
│                      (player state in memory)   │
│                      (replicated to 3 nodes)    │
│                                                 │
│  Benefits over external DB:                     │
│  - Sub-millisecond reads (local state)          │
│  - No external DB dependency                    │
│  - Automatic failover with state preserved      │
│  - Partition by player ID for scale             │
│                                                 │
│  Use Case: Real-time leaderboards, game state,  │
│  matchmaking, inventory management              │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Service Fabric = Microservices platform with built-in state
Powers Azure internally (Cosmos DB, SQL Database, Event Hubs)

Cluster: Nodes → Fault Domains + Upgrade Domains
App Model: Application → Services → Partitions → Replicas

Stateless: No local state, external DB (like regular web API)
Stateful: Built-in state via Reliable Collections (ultra-low latency)
Reliable Collections: ReliableDictionary, ReliableQueue (replicated, ACID)

Managed clusters: Azure handles infra, patching, scaling
SKUs: Basic (dev/test, 3 nodes) | Standard (prod, 5+ nodes)

vs AKS: Service Fabric for stateful .NET; AKS for everything else
```

---

## What's Next?

Next chapter: [Chapter 56: Azure Synapse Analytics](56-synapse-analytics.md) — Unified analytics platform combining data warehousing, big data, and data integration.
