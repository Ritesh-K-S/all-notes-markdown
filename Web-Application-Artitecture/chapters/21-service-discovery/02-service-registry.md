# Service Registry — Consul, Eureka, etcd

> **What you'll learn**: How service registries work internally — the databases that keep track of every running service instance, their health, and their metadata. You'll understand Consul, Eureka, and etcd deeply enough to choose the right one and operate it in production.

---

## Real-Life Analogy

Think of a service registry like the **reception desk at a massive office building**:

- When a new company (service) moves in, they register at the front desk: "We're Acme Corp, room 405, open 9-5."
- The front desk maintains a **live directory** of all tenants.
- When someone walks in and says "I need to find the accounting firm," the desk says "Floor 4, room 405 — and they're currently open."
- If a company leaves without telling the desk, security (health checks) notices the office is empty and removes them from the directory.
- The desk has **multiple staff** (replicas) so it's never unavailable — if one person is on break, another can answer.

A service registry is exactly this: a **living, breathing directory** of all services, constantly updated as services come and go.

---

## What is a Service Registry?

A service registry is a **database of service instances**. It stores:

```
┌────────────────────────────────────────────────────────────┐
│              Service Registry — Internal State              │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  Service: "payment-service"                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Instance 1:                                          │    │
│  │   ID: payment-abc123                                 │    │
│  │   Host: 10.0.2.15                                    │    │
│  │   Port: 8080                                         │    │
│  │   Status: UP                                         │    │
│  │   Health: Passing                                    │    │
│  │   Metadata: {version: "2.3.1", region: "us-east-1"} │    │
│  │   Last Heartbeat: 2024-01-15T10:30:05Z              │    │
│  ├─────────────────────────────────────────────────────┤    │
│  │ Instance 2:                                          │    │
│  │   ID: payment-def456                                 │    │
│  │   Host: 10.0.2.16                                    │    │
│  │   Port: 8080                                         │    │
│  │   Status: UP                                         │    │
│  │   Health: Passing                                    │    │
│  │   Metadata: {version: "2.3.1", region: "us-east-1"} │    │
│  │   Last Heartbeat: 2024-01-15T10:30:03Z              │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Service: "order-service"                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Instance 1: ...                                      │    │
│  │ Instance 2: ...                                      │    │
│  │ Instance 3: ...                                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

---

## The Three Major Service Registries

```
┌────────────────────────────────────────────────────────────────────┐
│                     Service Registry Landscape                       │
├────────────────────┬──────────────────────┬────────────────────────┤
│   Netflix Eureka   │  HashiCorp Consul    │       etcd             │
├────────────────────┼──────────────────────┼────────────────────────┤
│ • Java ecosystem   │ • Multi-purpose      │ • Key-value store      │
│ • AP system        │ • CP system (Raft)   │ • CP system (Raft)     │
│ • Peer-to-peer     │ • Service mesh       │ • Kubernetes native    │
│ • Self-preservation│ • Health checks      │ • Strong consistency   │
│ • Spring Cloud     │ • Multi-datacenter   │ • Watch mechanism      │
│ • Netflix OSS      │ • DNS + HTTP API     │ • CNCF project         │
└────────────────────┴──────────────────────┴────────────────────────┘
```

---

## Deep Dive: Netflix Eureka

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Eureka Architecture                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Zone A                           Zone B                      │
│  ┌───────────────┐               ┌───────────────┐          │
│  │ Eureka Server │◄─── peer ────▶│ Eureka Server │          │
│  │    (node 1)   │  replication  │    (node 2)   │          │
│  └───────┬───────┘               └───────┬───────┘          │
│          │                               │                    │
│   ┌──────┼──────┐                 ┌──────┼──────┐           │
│   │      │      │                 │      │      │           │
│   ▼      ▼      ▼                 ▼      ▼      ▼           │
│ ┌───┐  ┌───┐  ┌───┐           ┌───┐  ┌───┐  ┌───┐        │
│ │Svc│  │Svc│  │Svc│           │Svc│  │Svc│  │Svc│        │
│ │ A │  │ B │  │ C │           │ A │  │ D │  │ E │        │
│ └───┘  └───┘  └───┘           └───┘  └───┘  └───┘        │
│                                                               │
│  • Each service registers with its zone's Eureka             │
│  • Eureka nodes replicate to each other (eventually)         │
│  • Clients prefer same-zone instances                         │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### How Eureka Works Internally

**Consistency Model: AP (Availability + Partition Tolerance)**

Eureka is an **AP system** (in CAP theorem terms). This means:
- It prioritizes **availability** over consistency
- During a network partition, all nodes continue serving (potentially stale) data
- Replication is **eventual** — nodes sync with peers asynchronously

**Key Mechanisms:**

1. **Registration**: Service sends a POST with instance info. Eureka stores it in a `ConcurrentHashMap` in memory.

2. **Heartbeats (Renewal)**: Every 30 seconds (configurable), services send a heartbeat. If missed for 90 seconds (3 intervals), Eureka evicts the instance.

3. **Self-Preservation Mode**: If Eureka detects that more than 15% of registered instances are missing heartbeats, it assumes a **network issue** (not mass failure) and stops evicting. This prevents cascading failures during network partitions.

4. **Peer Replication**: Each registration/heartbeat is replicated to peer Eureka nodes asynchronously. If replication fails, it retries.

5. **Client Cache**: Clients fetch the full registry on first call, then get **deltas** every 30 seconds. This means clients can still route even if Eureka goes down temporarily.

```
Timeline: Eureka Registration + Heartbeat Flow
═══════════════════════════════════════════════

t=0s    Service starts → POST /eureka/apps/PAYMENT
t=0s    Eureka stores in memory, replicates to peers
t=30s   Service sends heartbeat → PUT /eureka/apps/PAYMENT/instance-id
t=60s   Service sends heartbeat
t=90s   Service sends heartbeat
...
t=120s  Service crashes (no more heartbeats)
t=150s  Missed 1 heartbeat
t=180s  Missed 2 heartbeats
t=210s  Missed 3 heartbeats → Eureka evicts instance
        (unless self-preservation is active)
```

---

## Deep Dive: HashiCorp Consul

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Consul Architecture                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Datacenter 1 (dc1)                  Datacenter 2 (dc2)          │
│  ┌─────────────────────────────┐     ┌────────────────────────┐  │
│  │  Consul Server Cluster      │     │  Consul Server Cluster │  │
│  │  ┌────┐  ┌────┐  ┌────┐    │ WAN │  ┌────┐  ┌────┐       │  │
│  │  │Ldr │  │Flwr│  │Flwr│◄───┼─────┼─▶│Ldr │  │Flwr│       │  │
│  │  └──┬─┘  └──┬─┘  └──┬─┘    │     │  └──┬─┘  └──┬─┘       │  │
│  │     │       │       │       │     │     │       │          │  │
│  │     └───────┼───────┘       │     │     └───────┘          │  │
│  │             │ Raft          │     │           Raft          │  │
│  └─────────────┼───────────────┘     └───────────┼────────────┘  │
│                │                                  │               │
│         ┌──────┼──────┐                   ┌──────┼──────┐        │
│         │      │      │                   │      │      │        │
│  ┌──────┴─┐ ┌─┴──────┐ ┌────────┐  ┌────┴───┐ ┌┴──────┐       │
│  │ Agent  │ │ Agent   │ │ Agent  │  │ Agent  │ │ Agent  │       │
│  │(client)│ │(client) │ │(client)│  │(client)│ │(client)│       │
│  │        │ │         │ │        │  │        │ │        │       │
│  │ Svc A  │ │ Svc B   │ │ Svc C  │  │ Svc A  │ │ Svc D  │       │
│  └────────┘ └─────────┘ └────────┘  └────────┘ └────────┘       │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### How Consul Works Internally

**Consistency Model: CP (Consistency + Partition Tolerance)**

Consul uses the **Raft consensus algorithm** to maintain strong consistency:

1. **Server Cluster**: 3 or 5 server nodes form a Raft cluster. One is elected **leader**.
2. **Writes go to leader**: All registrations must go through the leader, which replicates to followers before acknowledging.
3. **Reads can be stale or consistent**: You choose per-query. `?stale` for fast reads, default for consistent.

**Consul's Unique Features:**

| Feature | How It Works |
|---------|-------------|
| **Multi-datacenter** | WAN gossip connects DCs; each DC has its own Raft cluster |
| **DNS Interface** | Query `payment.service.consul` via DNS (port 8600) |
| **Health Checks** | Agent on each node runs checks locally (HTTP, TCP, script, TTL) |
| **KV Store** | Built-in key-value store for configuration |
| **Service Mesh** | Connect feature provides mTLS between services |
| **Prepared Queries** | Templates for failover across datacenters |

**Agent Architecture:**

Every node runs a Consul **agent** in either client or server mode:
- **Client agents**: Lightweight, run on every service host, forward requests to servers
- **Server agents**: Participate in Raft, store state, heavy lifting

```
Service Registration Flow in Consul:
═════════════════════════════════════

1. Service starts on Node X
2. Local Consul agent (client) on Node X receives registration
3. Agent forwards to a Consul server
4. Server leader writes to Raft log
5. Raft replicates to majority of servers
6. Registration acknowledged
7. Agent starts health checking the local service
8. Health status propagated via gossip protocol (Serf)
```

---

## Deep Dive: etcd

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     etcd Cluster                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐             │
│  │  etcd    │     │  etcd    │     │  etcd    │             │
│  │  Node 1  │◄───▶│  Node 2  │◄───▶│  Node 3  │             │
│  │ (Leader) │     │(Follower)│     │(Follower)│             │
│  └────┬─────┘     └────┬─────┘     └────┬─────┘             │
│       │                │                │                     │
│       └────────────────┼────────────────┘                     │
│                        │                                      │
│              Raft Consensus Protocol                           │
│                                                                │
│  Data Model: Flat key-value with prefix-based hierarchy      │
│                                                                │
│  /services/payment/instance-1 → {"host":"10.0.2.15","port":8080}│
│  /services/payment/instance-2 → {"host":"10.0.2.16","port":8080}│
│  /services/order/instance-1   → {"host":"10.0.3.10","port":9090}│
│                                                                │
│  Key features:                                                │
│  • Watch API: Get notified of changes in real-time           │
│  • Lease: Keys expire if lease not renewed (like heartbeat)  │
│  • Transactions: Compare-and-swap for atomic operations      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### How etcd Works Internally

**etcd is the backbone of Kubernetes** — it stores ALL cluster state including service endpoints.

**Key Mechanisms:**

1. **Lease-based Registration**: A service creates a lease (e.g., 10-second TTL), attaches its key to the lease, and keeps the lease alive with keep-alives. If the service dies, the lease expires, and the key is deleted.

2. **Watch API**: Clients can watch a key prefix (e.g., `/services/payment/`) and get notified instantly when instances are added or removed.

3. **MVCC (Multi-Version Concurrency Control)**: etcd keeps a history of all changes. You can read "as of revision 1542" for consistency.

4. **Raft Consensus**: Same as Consul — leader handles writes, replicates to majority.

```
etcd Service Discovery Flow:
═════════════════════════════

Register:
  1. Create lease with 10s TTL
  2. PUT /services/payment/instance-1 with lease attached
  3. Start goroutine to send keep-alive every 3s

Discover:
  1. GET /services/payment/ (prefix query)
  2. Returns all instances under that prefix
  3. Watch /services/payment/ for changes
  4. On change event → update local routing table

Failure:
  1. Service crashes → no more keep-alives
  2. After 10s, lease expires
  3. etcd deletes key
  4. Watchers get DELETE event
  5. Clients remove instance from routing
```

---

## Comparison Table

| Feature | Eureka | Consul | etcd |
|---------|--------|--------|------|
| **Consistency** | AP (eventual) | CP (Raft) | CP (Raft) |
| **Language** | Java | Go | Go |
| **Health Checks** | Heartbeat only | HTTP, TCP, Script, gRPC, TTL | Lease-based TTL |
| **Multi-DC** | Limited (zones) | Native (WAN gossip) | Requires custom setup |
| **DNS Interface** | No | Yes | No (use coredns plugin) |
| **KV Store** | No | Yes (built-in) | Yes (primary purpose) |
| **Service Mesh** | No | Yes (Connect) | No (but K8s uses it) |
| **Watch/Push** | Client polling (delta) | Blocking queries / Watch | gRPC Watch stream |
| **Self-Preservation** | Yes | No (uses Raft guarantees) | No |
| **Kubernetes Integration** | Via Spring Cloud K8s | Native (Helm chart) | IS Kubernetes' store |
| **Best For** | Spring/Java microservices | Multi-purpose, multi-DC | Kubernetes, simple KV |
| **Scalability** | 1000s of services | 10,000s of services | 1000s of keys/watches |

---

## Code Examples

### Python: Service Registry with etcd

```python
import etcd3
import json
import threading
import time
import uuid

class EtcdServiceRegistry:
    """Service registry implementation using etcd3."""
    
    def __init__(self, etcd_host="localhost", etcd_port=2379):
        self.client = etcd3.client(host=etcd_host, port=etcd_port)
        self.instance_id = str(uuid.uuid4())[:8]
        self._keep_alive_thread = None
        self._running = False
    
    def register(self, service_name, host, port, ttl=10):
        """Register a service instance with a lease (auto-expires)."""
        # Create a lease — key expires if not renewed
        lease = self.client.lease(ttl=ttl)
        
        key = f"/services/{service_name}/{self.instance_id}"
        value = json.dumps({
            "host": host,
            "port": port,
            "instance_id": self.instance_id,
            "registered_at": time.time()
        })
        
        # Put with lease — auto-deleted if lease expires
        self.client.put(key, value, lease=lease)
        print(f"Registered: {key} → {host}:{port} (TTL={ttl}s)")
        
        # Start background thread to keep lease alive
        self._running = True
        self._keep_alive_thread = threading.Thread(
            target=self._keep_alive, args=(lease,), daemon=True
        )
        self._keep_alive_thread.start()
        
        return lease
    
    def _keep_alive(self, lease):
        """Send keep-alive to prevent lease expiry."""
        while self._running:
            try:
                lease.refresh()
                time.sleep(lease.ttl // 3)  # Refresh at 1/3 of TTL
            except Exception as e:
                print(f"Keep-alive failed: {e}")
                break
    
    def discover(self, service_name):
        """Get all healthy instances of a service."""
        prefix = f"/services/{service_name}/"
        instances = []
        
        for value, metadata in self.client.get_prefix(prefix):
            instance = json.loads(value.decode())
            instances.append(instance)
        
        return instances
    
    def watch(self, service_name, callback):
        """Watch for changes to a service's instances."""
        prefix = f"/services/{service_name}/"
        
        events_iterator, cancel = self.client.watch_prefix(prefix)
        
        for event in events_iterator:
            if isinstance(event, etcd3.events.PutEvent):
                instance = json.loads(event.value.decode())
                callback("ADDED", instance)
            elif isinstance(event, etcd3.events.DeleteEvent):
                callback("REMOVED", event.key.decode())
    
    def deregister(self, service_name):
        """Gracefully deregister on shutdown."""
        self._running = False
        key = f"/services/{service_name}/{self.instance_id}"
        self.client.delete(key)
        print(f"Deregistered: {key}")


# --- Usage ---
registry = EtcdServiceRegistry()

# Register this service
registry.register("payment-service", "10.0.2.15", 8080, ttl=10)

# Discover another service
instances = registry.discover("order-service")
for inst in instances:
    print(f"  Found: {inst['host']}:{inst['port']}")

# Watch for changes (in production, run in background thread)
def on_change(event_type, data):
    print(f"Service change: {event_type} → {data}")

# registry.watch("payment-service", on_change)
```

### Java: Service Registry with Consul

```java
import com.orbitz.consul.Consul;
import com.orbitz.consul.HealthClient;
import com.orbitz.consul.AgentClient;
import com.orbitz.consul.model.agent.ImmutableRegistration;
import com.orbitz.consul.model.agent.Registration;
import com.orbitz.consul.model.health.ServiceHealth;
import java.util.List;
import java.util.stream.Collectors;

public class ConsulServiceRegistry {
    
    private final Consul consul;
    private final AgentClient agentClient;
    private final HealthClient healthClient;
    
    public ConsulServiceRegistry(String consulHost, int consulPort) {
        this.consul = Consul.builder()
            .withUrl(String.format("http://%s:%d", consulHost, consulPort))
            .build();
        this.agentClient = consul.agentClient();
        this.healthClient = consul.healthClient();
    }
    
    /**
     * Register a service with Consul including health check.
     */
    public void register(String serviceName, String host, int port) {
        String serviceId = serviceName + "-" + host + "-" + port;
        
        Registration registration = ImmutableRegistration.builder()
            .id(serviceId)
            .name(serviceName)
            .address(host)
            .port(port)
            .check(Registration.RegCheck.http(
                String.format("http://%s:%d/health", host, port),
                10L,    // Check every 10 seconds
                5L      // Deregister after 5 failed checks (critical for 30s)
            ))
            .putMeta("version", "2.3.1")
            .putMeta("region", "us-east-1")
            .build();
        
        agentClient.register(registration);
        System.out.printf("Registered %s at %s:%d%n", serviceName, host, port);
        
        // Add shutdown hook for graceful deregistration
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            agentClient.deregister(serviceId);
            System.out.printf("Deregistered %s%n", serviceId);
        }));
    }
    
    /**
     * Discover all healthy instances of a service.
     */
    public List<ServiceInstance> discover(String serviceName) {
        List<ServiceHealth> healthyServices = 
            healthClient.getHealthyServiceInstances(serviceName)
                .getResponse();
        
        return healthyServices.stream()
            .map(sh -> new ServiceInstance(
                sh.getService().getAddress(),
                sh.getService().getPort(),
                sh.getService().getMeta()
            ))
            .collect(Collectors.toList());
    }
    
    /**
     * Watch for service changes using blocking queries (long polling).
     */
    public void watch(String serviceName, ServiceChangeListener listener) {
        // Consul blocking queries: request blocks until data changes
        // or timeout (default 5 min). Very efficient.
        new Thread(() -> {
            long index = 0;
            while (true) {
                try {
                    var response = healthClient
                        .getHealthyServiceInstances(serviceName,
                            QueryOptions.blockMinutes(5, index).build());
                    
                    long newIndex = response.getIndex();
                    if (newIndex != index) {
                        index = newIndex;
                        listener.onServiceChange(serviceName, 
                            response.getResponse());
                    }
                } catch (Exception e) {
                    System.err.println("Watch error: " + e.getMessage());
                    Thread.sleep(1000); // Backoff on error
                }
            }
        }, "consul-watcher-" + serviceName).start();
    }
}

// Simple data class
record ServiceInstance(String host, int port, Map<String, String> metadata) {
    public String url() {
        return String.format("http://%s:%d", host, port);
    }
}
```

---

## Infrastructure: Running a Production Consul Cluster

### Docker Compose — 3-Node Consul Cluster

```yaml
version: '3.8'

services:
  consul-server-1:
    image: hashicorp/consul:1.17
    container_name: consul-server-1
    command: >
      agent -server
      -bootstrap-expect=3
      -node=consul-1
      -bind=0.0.0.0
      -client=0.0.0.0
      -retry-join=consul-server-2
      -retry-join=consul-server-3
      -ui
    ports:
      - "8500:8500"    # HTTP API + UI
      - "8600:8600/udp"  # DNS
    volumes:
      - consul-data-1:/consul/data

  consul-server-2:
    image: hashicorp/consul:1.17
    container_name: consul-server-2
    command: >
      agent -server
      -bootstrap-expect=3
      -node=consul-2
      -bind=0.0.0.0
      -client=0.0.0.0
      -retry-join=consul-server-1
      -retry-join=consul-server-3

  consul-server-3:
    image: hashicorp/consul:1.17
    container_name: consul-server-3
    command: >
      agent -server
      -bootstrap-expect=3
      -node=consul-3
      -bind=0.0.0.0
      -client=0.0.0.0
      -retry-join=consul-server-1
      -retry-join=consul-server-2

volumes:
  consul-data-1:
```

### Kubernetes: etcd Cluster (What Powers K8s Itself)

```yaml
# This is what K8s uses internally — you rarely manage this yourself
# unless running K8s the hard way or using etcd standalone
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  containers:
    - name: etcd
      image: registry.k8s.io/etcd:3.5.9
      command:
        - etcd
        - --name=etcd-0
        - --data-dir=/var/lib/etcd
        - --listen-client-urls=https://0.0.0.0:2379
        - --advertise-client-urls=https://etcd-0:2379
        - --listen-peer-urls=https://0.0.0.0:2380
        - --initial-cluster=etcd-0=https://etcd-0:2380,etcd-1=https://etcd-1:2380,etcd-2=https://etcd-2:2380
        - --initial-cluster-state=new
        - --cert-file=/etc/etcd/tls/server.crt
        - --key-file=/etc/etcd/tls/server.key
        - --trusted-ca-file=/etc/etcd/tls/ca.crt
      volumeMounts:
        - name: etcd-data
          mountPath: /var/lib/etcd
```

---

## Real-World Examples

### Netflix — Eureka at Scale

- **Scale**: 600+ microservices, thousands of instances
- **Setup**: Eureka servers in each AWS region/AZ, peer-replicated
- **Client caching**: Each service caches the full registry locally → survives Eureka outages
- **Self-preservation**: Prevents mass evictions during network issues (saved Netflix during AWS outages)
- **Custom extensions**: Netflix built custom health indicators beyond simple heartbeats

### HashiCorp Consul at Stripe

- **Use case**: Service discovery + service mesh (Consul Connect)
- **Multi-DC**: Consul clusters in multiple regions with WAN federation
- **mTLS**: All service-to-service communication encrypted via Consul Connect
- **Prepared Queries**: Automatic failover to another DC if local services are unhealthy

### Kubernetes + etcd (Google, Every K8s User)

- **Google Borg → Kubernetes**: etcd stores ALL cluster state
- **Scale**: Large clusters have 10,000+ pods, all endpoints stored in etcd
- **Performance**: etcd handles 10,000+ write requests/sec in production K8s clusters
- **Backup critical**: If etcd is lost, your entire cluster state is gone

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | How to Fix |
|---------|-------------|------------|
| **Single registry node** | Registry itself is a SPOF | Always run 3-5 nodes (odd number for Raft) |
| **No health checks** | Dead services stay registered | Configure HTTP/TCP health checks |
| **Ignoring self-preservation (Eureka)** | Mass evictions during network blips | Understand and tune self-preservation thresholds |
| **etcd too many watchers** | Memory exhaustion on etcd cluster | Use shared informers / cache watches |
| **Not monitoring the registry** | Registry dies silently | Alert on leader elections, peer count, request latency |
| **Storing too much in etcd** | etcd has 8GB default limit | Only store small service metadata, not configs |
| **Missing graceful deregistration** | Stale entries for TTL duration | Add shutdown hooks to deregister |
| **Cross-DC reads without stale flag** | High latency for reads | Use `?stale` for cross-DC reads in Consul |

---

## When to Use / When NOT to Use

### Choose Eureka When:
- ✅ You're in the **Java/Spring Cloud** ecosystem
- ✅ You want **AP behavior** (availability over consistency)
- ✅ You need **self-preservation** (protect against network issues)
- ✅ You're building Netflix-style microservices

### Choose Consul When:
- ✅ You need **multi-datacenter** support
- ✅ You want **service mesh** (mTLS) built-in
- ✅ You need **DNS-based discovery** for non-cloud-native apps
- ✅ You want a **KV store + discovery** in one tool
- ✅ You're polyglot (multiple languages)

### Choose etcd When:
- ✅ You're running **Kubernetes** (it's already there!)
- ✅ You need **strong consistency** guarantees
- ✅ You want **watch-based** real-time notifications
- ✅ You need a **simple, reliable KV store**

### Don't Use a Separate Registry When:
- ❌ You're **already on Kubernetes** — use native Service + DNS
- ❌ You have a **simple monolith** — no services to discover
- ❌ You're using **AWS ECS/Fargate** — use AWS Cloud Map instead
- ❌ You have **< 5 services** — a simple config file or DNS might suffice

---

## Key Takeaways

1. **A service registry is the source of truth** for which services are running, where, and whether they're healthy.

2. **Eureka = AP, Consul/etcd = CP** — choose based on your consistency needs. AP gives availability during partitions; CP gives correctness.

3. **Always run in a cluster** — 3 or 5 nodes. A single registry node is a single point of failure that defeats the purpose.

4. **Health checking is non-negotiable** — a registry full of dead entries is worse than no registry.

5. **Kubernetes has this built-in** — etcd + CoreDNS + Services = native discovery. Don't add Eureka/Consul unless you have cross-cluster needs.

6. **Client-side caching saves you** — services should cache registry data locally to survive registry outages.

7. **Monitor your registry** — it's critical infrastructure. Alert on leader loss, high latency, and node failures.

---

## What's Next?

Service registries store service locations, but services also need **configuration** — database URLs, feature flags, API keys, timeouts. In the next chapter, [03-configuration-management.md](./03-configuration-management.md), we'll explore **Configuration Management** — how to manage, distribute, and update configuration across hundreds of services without redeploying.
