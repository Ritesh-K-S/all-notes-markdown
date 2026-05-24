# Sidecar Pattern, Ambassador Pattern & Adapter Pattern

> **What you'll learn**: Three powerful infrastructure patterns that offload cross-cutting concerns (networking, logging, monitoring, protocol translation) from your application into separate companion processes — enabling polyglot architectures and keeping your services focused on business logic.

---

## Real-Life Analogy

### Sidecar
Think of a motorcycle with a sidecar. The motorcycle (your app) focuses on moving forward. The sidecar (companion process) carries extra passengers, luggage, or equipment without slowing down the motorcycle. They travel together, share the same journey, but have different responsibilities.

### Ambassador
An ambassador represents their country in a foreign land. They handle all the protocol, translation, and diplomacy so the president (your app) doesn't need to learn every foreign language or custom. Your app talks to the ambassador; the ambassador handles the complex external world.

### Adapter
A power adapter converts one plug shape to another. Your laptop (app) has one plug type. Different countries (external services) have different outlet types. The adapter translates between them — your laptop doesn't need to change.

---

## The Sidecar Pattern

A **sidecar** is a helper process deployed alongside your main application in the same host/pod. It extends your app's capabilities without modifying the app itself.

```
┌─────────────────────────────────────────────────────┐
│                  Pod / Host                           │
│                                                       │
│  ┌───────────────────┐    ┌───────────────────────┐  │
│  │   Main Container   │    │   Sidecar Container   │  │
│  │                     │    │                       │  │
│  │   Your Application  │◄──▶│   Cross-cutting      │  │
│  │   (Business Logic)  │    │   Concern:            │  │
│  │                     │    │   • Logging agent     │  │
│  │   Python / Java /   │    │   • Proxy (Envoy)    │  │
│  │   Go / whatever     │    │   • Config watcher   │  │
│  │                     │    │   • Metrics collector │  │
│  └───────────────────┘    └───────────────────────┘  │
│                                                       │
│  Shared: network (localhost), filesystem (volumes)    │
└─────────────────────────────────────────────────────┘
```

### What Sidecars Do:

```
┌─────────────────────────────────────────────────────────────┐
│                 COMMON SIDECAR USE CASES                      │
├──────────────────────┬──────────────────────────────────────┤
│  Use Case            │  Example                              │
├──────────────────────┼──────────────────────────────────────┤
│  Service mesh proxy  │  Envoy/Istio sidecar                 │
│  Log collection      │  Fluentd/Filebeat sidecar            │
│  Monitoring          │  Prometheus exporter sidecar         │
│  Configuration       │  Config watcher that reloads config  │
│  Security            │  OAuth2 proxy, mTLS termination      │
│  Health checking     │  Custom health check sidecar         │
│  Data sync           │  Database proxy, connection pooling  │
└──────────────────────┴──────────────────────────────────────┘
```

### Kubernetes Sidecar Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  # Main application container
  - name: myapp
    image: myapp:v1.0
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app
  
  # Sidecar: Log shipping
  - name: log-shipper
    image: fluentd:latest
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app
      readOnly: true
    env:
    - name: ELASTICSEARCH_HOST
      value: "elasticsearch:9200"
  
  # Sidecar: Envoy proxy (service mesh)
  - name: envoy-proxy
    image: envoyproxy/envoy:v1.28
    ports:
    - containerPort: 9901  # Admin
    - containerPort: 15001 # Outbound traffic

  volumes:
  - name: log-volume
    emptyDir: {}
```

---

## The Ambassador Pattern

An **ambassador** is a specific type of sidecar that acts as a **proxy for outbound connections** — handling retries, circuit breaking, routing, and protocol translation on behalf of your application.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Pod / Host                                │
│                                                                   │
│  ┌──────────────────┐       ┌──────────────────────┐            │
│  │  Main App         │       │  Ambassador Proxy     │            │
│  │                    │       │                       │            │
│  │  Connects to:     │──────▶│  Handles:             │            │
│  │  localhost:6379    │       │  • Retries            │            │
│  │  (thinks it's     │       │  • Circuit breaking   │            │
│  │   talking to      │       │  • Load balancing     │            │
│  │   local Redis)    │       │  • TLS encryption     │            │
│  │                    │       │  • Service discovery  │            │
│  └──────────────────┘       └───────────┬───────────┘            │
│                                          │                        │
└──────────────────────────────────────────┼────────────────────────┘
                                           │
                                           ▼
                            ┌─────────────────────────┐
                            │   Actual Remote Service   │
                            │   Redis Cluster (3 nodes) │
                            │   redis-1, redis-2,       │
                            │   redis-3                 │
                            └─────────────────────────┘
```

### Why Ambassador Pattern?

Your app just connects to `localhost:6379`. The ambassador handles:
- Finding the actual Redis cluster nodes
- Routing reads to replicas, writes to primary
- Retrying on transient failures
- Encrypting the connection with TLS
- Reporting metrics

**Your app doesn't need ANY of this logic.** It stays simple.

### Real Example — Connecting to a sharded database:

```
Without Ambassador:
─────────────────────────────────────────
App must know:
  • All shard addresses
  • Sharding algorithm
  • Health of each shard
  • Retry logic
  • Connection pooling per shard

With Ambassador:
─────────────────────────────────────────
App connects to: localhost:5432
Ambassador handles everything else.

┌──────┐  localhost:5432  ┌───────────┐  ┌─────────────┐
│ App  │─────────────────▶│Ambassador │──▶│ Shard 1     │
│      │                  │           │──▶│ Shard 2     │
│      │                  │           │──▶│ Shard 3     │
└──────┘                  └───────────┘  └─────────────┘
```

---

## The Adapter Pattern

An **adapter** translates between your application's interface and an external system's incompatible interface. It's like a plug converter.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Pod / Host                                 │
│                                                                   │
│  ┌──────────────────┐       ┌──────────────────────┐            │
│  │  Main App         │       │  Adapter Container    │            │
│  │                    │       │                       │            │
│  │  Outputs:          │──────▶│  Transforms:          │            │
│  │  Custom metrics    │       │  Custom format        │            │
│  │  in app-specific   │       │  → Prometheus format  │            │
│  │  format            │       │                       │            │
│  │                    │       │  Custom logs          │            │
│  │                    │       │  → ELK format         │            │
│  └──────────────────┘       └───────────┬───────────┘            │
│                                          │                        │
└──────────────────────────────────────────┼────────────────────────┘
                                           │  standard format
                                           ▼
                            ┌─────────────────────────┐
                            │  Prometheus / ELK Stack   │
                            │  (Expects standard format)│
                            └─────────────────────────┘
```

### Common Adapter Use Cases:

```
┌─────────────────────────────────────────────────────────────┐
│                  ADAPTER EXAMPLES                             │
├────────────────────────────────────────────────────────────  │
│                                                               │
│  1. Metrics Adapter                                          │
│     App outputs: { "cpu_usage": 0.75, "mem_mb": 512 }       │
│     Prometheus expects: cpu_usage_ratio 0.75                 │
│     Adapter translates between formats                       │
│                                                               │
│  2. Logging Adapter                                          │
│     App outputs: Custom binary log format                    │
│     ELK expects: JSON with @timestamp, level, message        │
│     Adapter converts the format                              │
│                                                               │
│  3. Protocol Adapter                                         │
│     Legacy service speaks: SOAP/XML                          │
│     New services expect: REST/JSON                           │
│     Adapter translates between protocols                     │
│                                                               │
│  4. Authentication Adapter                                   │
│     App uses: API keys                                       │
│     External system expects: OAuth2 tokens                   │
│     Adapter handles token management                         │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Comparison: Sidecar vs Ambassador vs Adapter

```
┌──────────────┬───────────────────┬──────────────────┬──────────────────┐
│              │     SIDECAR       │    AMBASSADOR    │     ADAPTER      │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ Direction    │ General companion │ Outbound proxy   │ Inbound/Outbound │
│              │                   │ for remote calls │ format translator│
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ Purpose      │ Extend app        │ Simplify remote  │ Translate        │
│              │ capabilities      │ connections      │ interfaces       │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ Examples     │ Log shipping,     │ Redis proxy,     │ Metrics exporter,│
│              │ config reload,    │ DB proxy,        │ protocol bridge, │
│              │ cert rotation     │ service mesh     │ format converter │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ App knows?   │ May or may not    │ App thinks it's  │ App outputs its  │
│              │ know about sidecar│ connecting local  │ native format    │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ Relationship │ Parent pattern    │ Specialization   │ Specialization   │
│              │                   │ of Sidecar       │ of Sidecar       │
└──────────────┴───────────────────┴──────────────────┴──────────────────┘

Note: Ambassador and Adapter are SPECIFIC TYPES of the Sidecar pattern.
```

---

## How It Works Internally — Envoy as a Sidecar Proxy

The most common sidecar in production is **Envoy** (used by Istio, AWS App Mesh, etc.):

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENVOY SIDECAR FLOW                             │
│                                                                   │
│  Incoming Request                                                │
│       │                                                           │
│       ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  ENVOY SIDECAR                            │    │
│  │                                                           │    │
│  │  1. TLS termination (decrypt incoming HTTPS)              │    │
│  │  2. Authentication (verify JWT)                           │    │
│  │  3. Rate limiting (check quota)                           │    │
│  │  4. Load balancing (choose backend)                       │    │
│  │  5. Retry policy (retry on 503)                           │    │
│  │  6. Circuit breaking (stop if target is failing)          │    │
│  │  7. Metrics collection (latency, error rate)              │    │
│  │  8. Distributed tracing (inject trace headers)            │    │
│  │                                                           │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                          │ forward to localhost:8080              │
│                          ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  YOUR APPLICATION                         │    │
│  │                                                           │    │
│  │  • Receives plain HTTP on localhost:8080                   │    │
│  │  • No TLS, auth, retry, or metrics code needed!           │    │
│  │  • Just pure business logic                               │    │
│  │                                                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: Ambassador Pattern — Redis Connection Proxy

```python
# ─── Without Ambassador: App must handle complexity ───────
# Your app needs to know about Redis Cluster, retries, failover
from redis.cluster import RedisCluster

# Complex configuration IN your app
redis = RedisCluster(
    startup_nodes=[
        {"host": "redis-node-1", "port": 6379},
        {"host": "redis-node-2", "port": 6379},
        {"host": "redis-node-3", "port": 6379},
    ],
    retry_on_timeout=True,
    socket_timeout=5,
    # ... lots of configuration
)


# ─── With Ambassador: App connects to localhost ───────────
# Ambassador container handles the cluster complexity
import redis

# Simple! App just connects to localhost
redis_client = redis.Redis(host="localhost", port=6379)

# App doesn't know (or care) about:
# - Cluster topology
# - Failover handling  
# - Connection pooling
# - Retry logic
# The ambassador sidecar handles ALL of this!


# ─── Ambassador implementation (separate container) ───────
# ambassador/proxy.py (runs as sidecar)
import socket
import threading
from redis.cluster import RedisCluster

class RedisAmbassador:
    """Proxies local Redis connections to a Redis Cluster"""
    
    def __init__(self, listen_port=6379, cluster_nodes=None):
        self.listen_port = listen_port
        self.cluster = RedisCluster(startup_nodes=cluster_nodes)
        self.server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server.bind(("127.0.0.1", listen_port))
        self.server.listen(100)
    
    def run(self):
        print(f"Ambassador listening on localhost:{self.listen_port}")
        while True:
            client_conn, addr = self.server.accept()
            thread = threading.Thread(
                target=self._handle_connection, args=(client_conn,))
            thread.start()
    
    def _handle_connection(self, client_conn):
        """Proxy commands from local app to Redis Cluster"""
        # Parse Redis protocol, forward to cluster
        # Handle retries, circuit breaking, etc.
        pass

# Run the ambassador
ambassador = RedisAmbassador(
    cluster_nodes=[
        {"host": "redis-1.prod", "port": 6379},
        {"host": "redis-2.prod", "port": 6379},
        {"host": "redis-3.prod", "port": 6379},
    ]
)
ambassador.run()
```

### Java: Adapter Pattern — Metrics Format Translation

```java
// ─── Adapter: Converts custom app metrics to Prometheus format ─

// Your app outputs metrics in its own format
public class AppMetrics {
    private final Map<String, Double> gauges = new ConcurrentHashMap<>();
    private final Map<String, Long> counters = new ConcurrentHashMap<>();
    
    public void setGauge(String name, double value) {
        gauges.put(name, value);
    }
    
    public void incrementCounter(String name) {
        counters.merge(name, 1L, Long::sum);
    }
    
    // App-specific format
    public Map<String, Object> export() {
        return Map.of("gauges", gauges, "counters", counters);
    }
}

// ─── Adapter: Translates to Prometheus exposition format ─────
public class PrometheusAdapter {
    
    private final AppMetrics appMetrics;
    
    public PrometheusAdapter(AppMetrics appMetrics) {
        this.appMetrics = appMetrics;
    }
    
    /**
     * Exposes metrics in Prometheus text format on /metrics endpoint.
     * 
     * Converts:
     *   {"gauges": {"cpu_usage": 0.75}, "counters": {"requests": 1234}}
     * To:
     *   # TYPE cpu_usage gauge
     *   cpu_usage 0.75
     *   # TYPE requests_total counter
     *   requests_total 1234
     */
    @GetMapping("/metrics")
    public String prometheusMetrics() {
        StringBuilder sb = new StringBuilder();
        Map<String, Object> raw = appMetrics.export();
        
        // Convert gauges
        Map<String, Double> gauges = (Map<String, Double>) raw.get("gauges");
        for (var entry : gauges.entrySet()) {
            sb.append("# TYPE ").append(entry.getKey()).append(" gauge\n");
            sb.append(entry.getKey()).append(" ")
              .append(entry.getValue()).append("\n");
        }
        
        // Convert counters (Prometheus convention: suffix with _total)
        Map<String, Long> counters = (Map<String, Long>) raw.get("counters");
        for (var entry : counters.entrySet()) {
            String name = entry.getKey() + "_total";
            sb.append("# TYPE ").append(name).append(" counter\n");
            sb.append(name).append(" ")
              .append(entry.getValue()).append("\n");
        }
        
        return sb.toString();
    }
}
```

### Kubernetes: Full Sidecar Deployment

```yaml
# Complete example: App + logging sidecar + metrics adapter
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      # ─── Main Application ────────────────────────────
      - name: order-service
        image: order-service:v2.1
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: logs
          mountPath: /app/logs
        - name: tmp
          mountPath: /tmp
        resources:
          requests: { cpu: "500m", memory: "512Mi" }
          limits:   { cpu: "1000m", memory: "1Gi" }
      
      # ─── Sidecar: Log Shipping (Fluentd) ─────────────
      - name: log-shipper
        image: fluent/fluentd:v1.16
        volumeMounts:
        - name: logs
          mountPath: /app/logs
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc
        resources:
          requests: { cpu: "100m", memory: "128Mi" }
          limits:   { cpu: "200m", memory: "256Mi" }
      
      # ─── Adapter: Prometheus Metrics Exporter ─────────
      - name: metrics-adapter
        image: prom/statsd-exporter:v0.25
        ports:
        - containerPort: 9102  # Prometheus scrapes this
        args:
        - --statsd.listen-udp=:8125
        - --web.listen-address=:9102
        resources:
          requests: { cpu: "50m", memory: "64Mi" }
          limits:   { cpu: "100m", memory: "128Mi" }
      
      # ─── Ambassador: Envoy Proxy ─────────────────────
      - name: envoy-proxy
        image: envoyproxy/envoy:v1.28
        ports:
        - containerPort: 9901
        volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
        resources:
          requests: { cpu: "100m", memory: "128Mi" }
          limits:   { cpu: "200m", memory: "256Mi" }
      
      volumes:
      - name: logs
        emptyDir: {}
      - name: tmp
        emptyDir: {}
      - name: fluentd-config
        configMap:
          name: fluentd-config
      - name: envoy-config
        configMap:
          name: envoy-config
```

---

## Service Mesh — Sidecar Pattern at Scale

When EVERY service has an Envoy sidecar, you get a **service mesh**:

```
┌─────────────────────────────────────────────────────────────────┐
│                     SERVICE MESH (Istio)                          │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   Control Plane (Istiod)                    │  │
│  │  • Pushes config to all Envoy sidecars                     │  │
│  │  • Manages certificates (mTLS)                             │  │
│  │  • Defines traffic policies                                │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │ config push                        │
│              ┌───────────────┼───────────────┐                   │
│              ▼               ▼               ▼                   │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐│
│  │┌──────┐┌───────┐│ │┌──────┐┌───────┐│ │┌──────┐┌───────┐││
│  ││App A ││Envoy  ││ ││App B ││Envoy  ││ ││App C ││Envoy  │││
│  │└──────┘└───┬───┘│ │└──────┘└───┬───┘│ │└──────┘└───┬───┘││
│  │    Pod A   │     │ │    Pod B   │     │ │    Pod C   │     ││
│  └────────────┼─────┘ └────────────┼─────┘ └────────────┼─────┘│
│               │                    │                     │       │
│               └────── mTLS ────────┴──── mTLS ──────────┘       │
│                    (encrypted traffic between all services)       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Istio at Google/Lyft

- **Google** uses a service mesh internally for all inter-service communication (Envoy was created at Lyft)
- Every service gets an Envoy sidecar automatically
- Provides: mTLS encryption, traffic management, observability — WITHOUT changing any application code
- Services are written in Go, Java, Python, C++ — the sidecar makes them all behave consistently

### Netflix — Sidecar for Polyglot Support

Netflix built **Prana** (a sidecar) so non-JVM services could use their Java-based infrastructure:
```
JVM services: Direct access to Eureka, Ribbon, Hystrix
Non-JVM services: Prana sidecar provides same capabilities via HTTP
```

### Kubernetes + Fluentd (Logging Sidecar)

Almost every Kubernetes deployment uses a logging sidecar:
- App writes logs to a shared volume or stdout
- Fluentd/Filebeat sidecar ships logs to Elasticsearch
- App has ZERO dependency on the logging infrastructure

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Sidecar adds too much latency | Extra network hop for every request | Profile; ensure sidecar overhead is <5ms |
| Sidecar consumes too many resources | Doubles your pod resource usage | Set appropriate resource limits; monitor |
| Not handling sidecar failures | If sidecar dies, app is blind/deaf | Set up health checks and restart policies |
| Over-using sidecars | 4+ sidecars per pod = complexity | Consolidate; use multi-purpose sidecars |
| Sidecar and app lifecycle mismatch | Sidecar starts after app → app fails | Use init containers; handle gracefully |
| Ambassador that's a single point of failure | Ambassador crash = app can't talk to anything | Run ambassador with restart policy |
| Tight coupling to sidecar | App fails if specific sidecar is missing | App should degrade gracefully |

---

## When to Use / When NOT to Use

### Use Sidecar/Ambassador/Adapter When:
- Cross-cutting concerns (logging, metrics, security) affect ALL services
- You have a polyglot architecture (different languages/frameworks)
- You want consistent behavior WITHOUT modifying application code
- Network-level concerns (mTLS, retries, circuit breaking)
- Legacy apps that can't be easily modified
- You're on Kubernetes (native sidecar support)

### Do NOT Use When:
- Simple monolith (just use libraries/middleware)
- The overhead is not justified (single service, low traffic)
- You can use a shared library in a mono-language environment
- Inter-process communication latency is unacceptable (ultra-low-latency trading)
- Your team doesn't have Kubernetes expertise

> **Rule of thumb**: If you have 3+ services in different languages that all need the same cross-cutting behavior, sidecars are the answer. If everything is in one language, a shared library might be simpler.

---

## Key Takeaways

1. **Sidecar = companion process** that extends your app's capabilities without code changes. It's the parent pattern; Ambassador and Adapter are specializations.

2. **Ambassador = outbound proxy** that simplifies how your app connects to external/remote services. App connects to localhost; ambassador handles the complexity.

3. **Adapter = format translator** that converts between your app's interface and an external system's incompatible interface.

4. **Service meshes ARE the sidecar pattern at scale** — Istio/Linkerd inject an Envoy sidecar into every pod, providing consistent networking behavior across all services.

5. **Language-agnostic** — The biggest benefit is that these patterns work regardless of what language your app is written in. A Python app gets the same mTLS, retries, and metrics as a Java app.

6. **Resource overhead is real** — Each sidecar consumes CPU and memory. In a 100-pod cluster with 3 sidecars each, you're running 300 extra containers. Plan accordingly.

7. **Kubernetes native support** — Kubernetes 1.28+ has native sidecar containers with proper lifecycle management (start before app, stop after app).

---

## What's Next?

Next, we'll explore **Data Partitioning Strategies for Planet-Scale Apps** — how to split data across thousands of machines while maintaining query performance, consistency, and operability.

See: [07-data-partitioning-strategies.md](./07-data-partitioning-strategies.md)
