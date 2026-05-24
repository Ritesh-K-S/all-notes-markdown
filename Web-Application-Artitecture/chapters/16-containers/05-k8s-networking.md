# Kubernetes Networking & Service Discovery

> **What you'll learn**: How pods communicate with each other across nodes, how services route traffic, how DNS-based service discovery works, how Ingress exposes apps to the internet, and the network policies that secure it all.

---

## Real-Life Analogy: A Corporate Campus Phone System

Imagine a massive corporate campus with thousands of employees (pods) spread across multiple buildings (nodes):

- Every employee has a **desk phone with a unique extension** (Pod IP) — but extensions change when people move desks
- The **company directory** (DNS/CoreDNS) lets you find anyone by name: "Call the Sales team" → routes to an available person
- A **receptionist at each building** (kube-proxy) routes incoming calls to the right extension
- The **front gate security** (Ingress controller) handles external visitors and directs them to the right building
- **Phone policies** (Network Policies) control who can call whom — Finance can't call Engineering directly

```
External World ──▶ Front Gate (Ingress) ──▶ Building Receptionist (Service)
                                                      │
                                            ┌─────────┼─────────┐
                                            ▼         ▼         ▼
                                        Employee  Employee  Employee
                                        (Pod 1)   (Pod 2)   (Pod 3)
```

---

## Kubernetes Networking Model — The Four Rules

Kubernetes enforces a **flat networking model** with these guarantees:

```
┌───────────────────────────────────────────────────────────────────┐
│              KUBERNETES NETWORKING RULES                            │
│                                                                    │
│  1. Every Pod gets its own unique IP address                       │
│  2. Pods on ANY node can communicate with pods on ANY other node  │
│     WITHOUT NAT (Network Address Translation)                     │
│  3. Agents on a node (kubelet, etc.) can talk to all pods         │
│     on that node                                                  │
│  4. Pods in host network can reach all pods on all nodes          │
│                                                                    │
│  Result: Pods behave as if they're on the SAME flat network       │
│          regardless of which physical machine they're on           │
└───────────────────────────────────────────────────────────────────┘
```

---

## The Four Types of Kubernetes Communication

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                    │
│  1. Container ↔ Container (same Pod)                              │
│     └─ Via localhost (share network namespace)                     │
│                                                                    │
│  2. Pod ↔ Pod (same node)                                         │
│     └─ Via virtual bridge (cbr0/cni0)                             │
│                                                                    │
│  3. Pod ↔ Pod (different nodes)                                   │
│     └─ Via overlay network (VXLAN, IPinIP) or native routing      │
│                                                                    │
│  4. External ↔ Pod                                                │
│     └─ Via Service (NodePort/LoadBalancer) or Ingress             │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## How Pod-to-Pod Communication Works

### Same Node Communication

```
┌──────────────────── Worker Node 1 ────────────────────────────┐
│                                                                │
│  ┌─── Pod A ────────┐       ┌─── Pod B ────────┐             │
│  │ eth0: 10.244.1.2 │       │ eth0: 10.244.1.3 │             │
│  └────────┬─────────┘       └────────┬─────────┘             │
│           │ veth pair                 │ veth pair              │
│           ▼                           ▼                        │
│  ┌────────────────────────────────────────────────────┐       │
│  │            Virtual Bridge (cni0/cbr0)               │       │
│  │            Subnet: 10.244.1.0/24                    │       │
│  └────────────────────────────────────────────────────┘       │
│                                                                │
│  Pod A sends to 10.244.1.3:                                    │
│  → Packet goes to bridge → bridge sees Pod B is local         │
│  → Forwards directly to Pod B's veth                          │
└────────────────────────────────────────────────────────────────┘
```

### Cross-Node Communication

```
┌─── Worker Node 1 ───────────┐         ┌─── Worker Node 2 ───────────┐
│                              │         │                              │
│  ┌─── Pod A ──────┐         │         │         ┌─── Pod C ──────┐  │
│  │ 10.244.1.2     │         │         │         │ 10.244.2.5     │  │
│  └───────┬────────┘         │         │         └───────┬────────┘  │
│          │                   │         │                 │           │
│  ┌───────┴─────────┐        │         │        ┌───────┴─────────┐ │
│  │ Bridge 10.244.1.0│        │         │        │ Bridge 10.244.2.0│ │
│  └───────┬──────────┘        │         │        └───────┬──────────┘│
│          │                   │         │                 │           │
│  ┌───────┴──────────┐        │         │        ┌───────┴──────────┐│
│  │  Node Network     │        │         │        │  Node Network     ││
│  │  eth0: 192.168.1.1│◄───────┼─────────┼───────▶│  eth0: 192.168.1.2││
│  └───────────────────┘        │         │        └───────────────────┘│
│                              │         │                              │
└──────────────────────────────┘         └──────────────────────────────┘

Pod A (10.244.1.2) → Pod C (10.244.2.5):
1. Packet leaves Pod A to bridge
2. Bridge doesn't know 10.244.2.x → forwards to node network
3. Overlay/routing sends to Node 2 (encapsulated or routed)
4. Node 2 receives → forwards to its bridge → delivers to Pod C
```

### CNI (Container Network Interface) Plugins

The actual networking is implemented by **CNI plugins**:

| Plugin | How it Works | Best For |
|--------|-------------|----------|
| **Calico** | BGP routing + IP-in-IP/VXLAN | Production, network policies |
| **Cilium** | eBPF-based (kernel-level) | Performance, observability |
| **Flannel** | Simple VXLAN overlay | Learning, simple clusters |
| **Weave** | Mesh overlay with encryption | Multi-cloud, encryption |
| **AWS VPC CNI** | Native VPC networking | AWS EKS (pod IPs from VPC) |

```
┌──────────────── CNI Plugin Comparison ─────────────────────────┐
│                                                                  │
│  Flannel (Simple)        Calico (Production)    Cilium (Modern) │
│  ─────────────────       ──────────────────     ──────────────  │
│  VXLAN overlay           BGP + IPinIP           eBPF in kernel  │
│  Easy setup              Network Policies       L7 visibility   │
│  No network policies     High performance       Best performance│
│  Good for learning       Industry standard      Newer, growing  │
│                                                                  │
│  Performance: ★★☆        Performance: ★★★       Performance: ★★★★│
└──────────────────────────────────────────────────────────────────┘
```

---

## Services Deep Dive — How Traffic Gets Routed

### kube-proxy and iptables

When you create a Service, kube-proxy programs network rules:

```
┌──────────── How a Service Request is Routed ─────────────────────┐
│                                                                    │
│  Client Pod                                                        │
│  "Connect to webapp-service:80"                                   │
│       │                                                            │
│       ▼                                                            │
│  DNS Lookup: webapp-service → 10.96.45.12 (ClusterIP)            │
│       │                                                            │
│       ▼                                                            │
│  iptables rules (programmed by kube-proxy):                       │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  IF dst = 10.96.45.12:80 THEN                           │     │
│  │    DNAT to one of:                                       │     │
│  │      - 10.244.1.5:8080  (Pod 1)  ← 33% probability    │     │
│  │      - 10.244.1.6:8080  (Pod 2)  ← 33% probability    │     │
│  │      - 10.244.2.3:8080  (Pod 3)  ← 33% probability    │     │
│  └─────────────────────────────────────────────────────────┘     │
│       │                                                            │
│       ▼                                                            │
│  Packet is rewritten: dst = 10.244.1.5:8080                      │
│  Routed to Pod 1                                                  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### kube-proxy Modes

```
┌─── iptables mode (default) ──┐  ┌─── IPVS mode (better performance) ──┐
│                               │  │                                       │
│  Uses iptables rules          │  │  Uses Linux IPVS (IP Virtual Server)  │
│  Random load balancing        │  │  Multiple LB algorithms               │
│  O(n) rules per service       │  │  O(1) lookup (hash table)            │
│  OK for < 1000 services       │  │  Better for > 1000 services          │
│                               │  │  Round-robin, least-conn, etc.        │
└───────────────────────────────┘  └───────────────────────────────────────┘
```

---

## DNS-Based Service Discovery (CoreDNS)

Every Service automatically gets a DNS entry:

```
┌─────────────────── DNS Names in Kubernetes ──────────────────────┐
│                                                                    │
│  Service: webapp-service in namespace "production"                │
│                                                                    │
│  Full DNS:    webapp-service.production.svc.cluster.local         │
│  Short (same ns): webapp-service                                  │
│  Short (diff ns): webapp-service.production                      │
│                                                                    │
│  ┌────────────────────────────────────────────────────────┐      │
│  │  <service>.<namespace>.svc.<cluster-domain>            │      │
│  │      │          │       │        │                     │      │
│  │   webapp    production  svc   cluster.local            │      │
│  │   -service                                             │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                    │
│  Example lookups:                                                 │
│  webapp-service              → 10.96.45.12 (same namespace)      │
│  redis.caching               → 10.96.12.8  (caching namespace)   │
│  kafka.messaging.svc.cluster.local → 10.96.88.3 (full path)     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Headless Services — Direct Pod Discovery

Sometimes you need to discover **individual pod IPs** (e.g., for databases, Kafka brokers):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-brokers
spec:
  clusterIP: None    # ← Headless! No ClusterIP assigned
  selector:
    app: kafka
  ports:
    - port: 9092
```

```
Normal Service DNS:     kafka-service → 10.96.x.x (one VIP)
Headless Service DNS:   kafka-brokers → 10.244.1.5, 10.244.1.6, 10.244.2.3
                        (returns ALL pod IPs!)

Individual pod DNS (with StatefulSet):
  kafka-0.kafka-brokers.default.svc.cluster.local → 10.244.1.5
  kafka-1.kafka-brokers.default.svc.cluster.local → 10.244.1.6
  kafka-2.kafka-brokers.default.svc.cluster.local → 10.244.2.3
```

---

## Ingress — Exposing Apps to the Internet

An **Ingress** is an API object that manages external HTTP/HTTPS access:

```
┌──────────────────── The Internet ────────────────────────────────┐
│                                                                    │
│  User → https://api.mycompany.com/users                          │
│  User → https://app.mycompany.com                                 │
│  User → https://api.mycompany.com/products                       │
│                                                                    │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────── Ingress Controller (Nginx/Traefik) ──────────────────┐
│                                                                    │
│  Rules:                                                           │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Host: api.mycompany.com                                      │ │
│  │   /users    → user-service:80                                │ │
│  │   /products → product-service:80                             │ │
│  │                                                               │ │
│  │ Host: app.mycompany.com                                      │ │
│  │   /         → frontend-service:80                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  Also handles: TLS termination, rate limiting, rewrites           │
└──────────┬────────────────────────────────┬───────────────────────┘
           │                                │
           ▼                                ▼
    ┌──────────────┐                ┌──────────────┐
    │ user-service │                │product-service│
    │   (Pods)     │                │   (Pods)      │
    └──────────────┘                └──────────────┘
```

```yaml
# ingress.yaml — Route external traffic
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.mycompany.com
        - app.mycompany.com
      secretName: mycompany-tls
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 80
    - host: app.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### Popular Ingress Controllers

| Controller | Best For | Key Feature |
|-----------|---------|-------------|
| **Nginx Ingress** | General purpose | Most widely used |
| **Traefik** | Auto-discovery | Middleware chain, dashboard |
| **HAProxy** | High performance | TCP/UDP support |
| **AWS ALB Ingress** | AWS EKS | Native ALB integration |
| **Istio Gateway** | Service mesh | Advanced traffic management |
| **Contour** | Envoy-based | HTTPProxy CRD, delegation |

---

## Network Policies — Kubernetes Firewall

By default, all pods can talk to all other pods. **Network Policies** restrict this:

```
Without Network Policy:           With Network Policy:
┌───┐  ┌───┐  ┌───┐              ┌───┐  ┌───┐  ┌───┐
│ A │◄▶│ B │◄▶│ C │              │ A │──▶│ B │   │ C │
└───┘  └───┘  └───┘              └───┘  └───┘   └───┘
Everyone talks to everyone         A→B only, C is isolated
```

```yaml
# network-policy.yaml — Allow only specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server          # This policy applies to api-server pods
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Only allow traffic FROM ingress controller and frontend
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-system
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Only allow traffic TO database and redis
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # Allow DNS (required for service discovery!)
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

**Visualization:**
```
┌─────── production namespace ─────────────────────────────────────┐
│                                                                    │
│  ┌── ingress-nginx ──┐                                            │
│  │  (namespace:       │                                            │
│  │   ingress-system)  │───TCP:8080──▶┌──────────────┐             │
│  └────────────────────┘              │  api-server  │             │
│                                      │              │             │
│  ┌── frontend ────────┐──TCP:8080──▶ │  (protected  │             │
│  │  app: frontend     │              │   by policy) │             │
│  └────────────────────┘              └──────┬───────┘             │
│                                             │                      │
│                              ┌──────────────┼──────────────┐      │
│                              ▼ TCP:5432     ▼ TCP:6379     │      │
│                        ┌──────────┐   ┌──────────┐         │      │
│                        │ postgres │   │  redis   │         │      │
│                        └──────────┘   └──────────┘         │      │
│                                                                    │
│  ┌── attacker-pod ────┐                                           │
│  │  BLOCKED! ❌        │──TCP:8080──✖                              │
│  └────────────────────┘                                           │
└────────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Service Discovery in Action

```python
import os
import requests
import dns.resolver  # pip install dnspython

# Method 1: Environment Variables (automatically injected)
# K8s creates env vars for every service in the same namespace
redis_host = os.environ.get('REDIS_SERVICE_HOST', 'redis')
redis_port = os.environ.get('REDIS_SERVICE_PORT', '6379')
print(f"Redis endpoint: {redis_host}:{redis_port}")

# Method 2: DNS-based discovery (preferred)
def discover_service(service_name, namespace='default'):
    """Discover a service using Kubernetes DNS"""
    fqdn = f"{service_name}.{namespace}.svc.cluster.local"
    
    # A record → ClusterIP
    try:
        answers = dns.resolver.resolve(fqdn, 'A')
        for rdata in answers:
            print(f"  {fqdn} → {rdata.address}")
        return str(answers[0].address)
    except dns.resolver.NXDOMAIN:
        print(f"  Service {fqdn} not found!")
        return None

# Discover services
print("Discovering services:")
api_ip = discover_service('api-service', 'production')
db_ip = discover_service('postgres', 'databases')

# Method 3: SRV records (for port discovery)
def discover_service_with_port(service_name, namespace='default'):
    """Get both host and port using SRV records"""
    srv_name = f"_http._tcp.{service_name}.{namespace}.svc.cluster.local"
    try:
        answers = dns.resolver.resolve(srv_name, 'SRV')
        for rdata in answers:
            print(f"  {service_name}: {rdata.target}:{rdata.port}")
    except Exception as e:
        print(f"  SRV lookup failed: {e}")

# In-app usage: just use the service name as hostname!
response = requests.get('http://user-service:8080/api/users')
# K8s DNS resolves "user-service" → ClusterIP → routed to a pod
```

### Java — Spring Boot with Kubernetes Service Discovery

```java
// application.yml — Using K8s DNS for service communication
// spring:
//   datasource:
//     url: jdbc:postgresql://postgres.databases:5432/mydb
//   redis:
//     host: redis.caching
//     port: 6379

@Service
public class UserClient {
    
    private final RestTemplate restTemplate;
    
    // K8s DNS resolves "user-service" to the Service ClusterIP
    private static final String USER_SERVICE_URL = "http://user-service:8080";
    
    public UserClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
    
    public User getUser(Long id) {
        // This works because K8s DNS resolves the service name
        return restTemplate.getForObject(
            USER_SERVICE_URL + "/api/users/" + id,
            User.class
        );
    }
    
    public List<User> getAllUsers() {
        // Service discovery is transparent — just use the service name!
        ResponseEntity<List<User>> response = restTemplate.exchange(
            USER_SERVICE_URL + "/api/users",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<List<User>>() {}
        );
        return response.getBody();
    }
}

// Health check endpoint for K8s probes
@RestController
public class HealthController {
    
    @GetMapping("/health/live")
    public ResponseEntity<String> liveness() {
        return ResponseEntity.ok("alive");
    }
    
    @GetMapping("/health/ready")
    public ResponseEntity<String> readiness(
            @Autowired DataSource dataSource,
            @Autowired RedisTemplate<String, String> redis) {
        try {
            // Check database connectivity
            dataSource.getConnection().isValid(2);
            // Check Redis connectivity
            redis.getConnectionFactory().getConnection().ping();
            return ResponseEntity.ok("ready");
        } catch (Exception e) {
            return ResponseEntity.status(503).body("not ready: " + e.getMessage());
        }
    }
}
```

---

## Infrastructure Example: Complete Networking Setup

```yaml
# === Namespace isolation ===
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production

---
# === Default deny all traffic in namespace ===
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}    # Applies to ALL pods
  policyTypes:
    - Ingress
    - Egress

---
# === Allow DNS for all pods ===
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## Real-World Example: How Shopify Networks 900K+ Stores

```
┌──────────── Shopify Kubernetes Networking ──────────────────────┐
│                                                                   │
│  Scale: 900,000+ stores, 150+ microservices                     │
│                                                                   │
│  External Traffic Flow:                                          │
│                                                                   │
│  shopowner.myshopify.com                                         │
│       │                                                           │
│       ▼                                                           │
│  ┌──────────────┐                                                │
│  │  Cloudflare  │  ← DDoS protection, edge caching             │
│  │  (CDN/WAF)   │                                                │
│  └──────┬───────┘                                                │
│         ▼                                                         │
│  ┌──────────────┐                                                │
│  │   Nginx      │  ← Ingress Controller (custom)                │
│  │   OpenResty  │     Rate limiting, routing                     │
│  └──────┬───────┘                                                │
│         │                                                         │
│    ┌────┴─────┬──────────────┐                                   │
│    ▼          ▼              ▼                                    │
│ ┌──────┐  ┌──────┐   ┌──────────────┐                           │
│ │Storefront│  │Checkout│  │Admin API    │                           │
│ │Service │  │Service│   │Service      │                           │
│ └──────┘  └──────┘   └──────────────┘                           │
│                                                                   │
│  Internal: Services discover each other via DNS                  │
│  Security: Network Policies isolate payment services             │
│  Observability: Cilium for eBPF-based network visibility         │
└───────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Problem | Solution |
|---------|---------|----------|
| Not using Network Policies | All pods can talk to all pods | Start with default-deny, whitelist |
| Forgetting to allow DNS egress | Pods can't resolve service names | Always allow UDP/TCP port 53 to kube-dns |
| Using NodePort in production | Exposes random high ports, no TLS | Use Ingress + LoadBalancer |
| Hardcoding pod IPs | Pods get new IPs on restart | Always use Service DNS names |
| No readiness probes | Traffic sent to unready pods | Always configure readiness checks |
| Ignoring DNS caching | Stale service resolution after scaling | Set appropriate `ndots` and TTL |
| Not considering DNS ndots | Excessive DNS queries for external names | Set `ndots: 2` in pod DNS config |
| Using ClusterIP: None without understanding | All pod IPs exposed, no load balancing | Use headless only for StatefulSets |

---

## When to Use / When NOT to Use

### Ingress vs LoadBalancer vs NodePort

```
┌────────────────────────────────────────────────────────────────┐
│  Use Case                          │  Solution                  │
├────────────────────────────────────┼────────────────────────────┤
│  HTTP(S) routing, TLS, virtual     │  Ingress                  │
│  hosts, path-based routing         │                           │
│                                     │                           │
│  Single service exposed externally  │  LoadBalancer Service     │
│  (non-HTTP: databases, gRPC)       │                           │
│                                     │                           │
│  Development/testing only           │  NodePort                 │
│                                     │                           │
│  Internal service-to-service        │  ClusterIP (default)     │
│                                     │                           │
│  Direct pod access (databases,      │  Headless Service        │
│  Kafka brokers)                    │  (ClusterIP: None)       │
└────────────────────────────────────┴────────────────────────────┘
```

### Network Policy Decision

| Scenario | Need Network Policies? |
|----------|----------------------|
| Development cluster, single team | Optional (adds complexity) |
| Production, single tenant | Yes (defense in depth) |
| Multi-tenant cluster | **Mandatory** (isolation required) |
| PCI/HIPAA compliance | **Mandatory** (regulatory) |

---

## Key Takeaways

1. **Every pod gets a unique, routable IP** — no NAT between pods, regardless of which node they're on
2. **Services provide stable DNS names** that route to healthy pods — never hardcode pod IPs
3. **CoreDNS powers service discovery** — use `service-name.namespace` for cross-namespace calls
4. **Ingress controllers** handle external HTTP/HTTPS traffic with routing, TLS, and more
5. **Network Policies are your firewall** — start with deny-all, then whitelist needed communication
6. **Headless services** expose individual pod IPs for stateful workloads (databases, message brokers)
7. **CNI plugins** (Calico, Cilium) implement the actual networking — choose based on your needs

---

## What's Next?

Now that you understand how pods communicate, let's learn how to persist data and manage configuration in [Chapter 16.6: Kubernetes Storage, ConfigMaps & Secrets](./06-k8s-storage-config.md).
