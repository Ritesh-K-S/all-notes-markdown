# Service Discovery — How Services Find Each Other

> **What you'll learn**: How microservices locate and communicate with each other dynamically, without hardcoding IP addresses and ports — the backbone of any distributed system.

---

## Real-Life Analogy

Imagine you move to a new city and need to find a doctor, a grocery store, and a gym. You have three options:

1. **Hardcode the address** — Someone gives you a piece of paper with addresses. But what if the doctor moves? Your paper is useless.
2. **Ask a friend every time** — "Hey, where's the nearest grocery store?" Your friend always knows the latest location. This is like **client-side service discovery**.
3. **Call a central directory (like Google Maps)** — You tell it what you need, and it routes you to the right place automatically. This is like **server-side service discovery**.

In a microservices world, services come and go constantly — they scale up, scale down, crash, restart on different machines. **Service discovery** is the mechanism that lets services find each other despite this chaos.

---

## Why Do We Need Service Discovery?

In a monolithic application, everything is in one process — calling a function is trivial. But in microservices:

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE PROBLEM                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Order Service needs to call:                                     │
│    • Payment Service  → Where is it? 10.0.2.15:8080? 10.0.3.7?  │
│    • Inventory Service → Which instance? There are 5 of them!    │
│    • User Service     → It just restarted on a new IP!           │
│                                                                   │
│  Services are DYNAMIC:                                            │
│    • IPs change on restart                                        │
│    • Instances scale up/down                                      │
│    • Containers get rescheduled                                   │
│    • Deployments happen constantly                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Without service discovery**, you'd have to:
- Hardcode IP addresses in config files
- Manually update configs every time a service moves
- Restart dependent services when addresses change
- Pray nothing crashes (it will)

---

## The Two Models of Service Discovery

### Model 1: Client-Side Discovery

The **client** (calling service) is responsible for finding the target service.

```
                    ┌──────────────────┐
                    │  Service Registry │
                    │  (Eureka / Consul)│
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │ 1. Query     │              │
              │   "Where is  │              │
              │   Payment?"  │              │
              ▼              │              │
┌──────────────────┐         │              │
│   Order Service  │         │              │
│   (Client)       │         │              │
└────────┬─────────┘         │              │
         │                   │              │
         │ 2. Gets list:     │              │
         │  [10.0.2.15:8080, │              │
         │   10.0.2.16:8080, │              │
         │   10.0.2.17:8080] │              │
         │                   │              │
         │ 3. Client picks   │              │
         │    one (load      │              │
         │    balancing)     │              │
         ▼                   │              │
┌──────────────────┐  ┌──────────────┐  ┌──────────────┐
│ Payment Service  │  │ Payment Svc  │  │ Payment Svc  │
│ 10.0.2.15:8080   │  │ 10.0.2.16    │  │ 10.0.2.17    │
└──────────────────┘  └──────────────┘  └──────────────┘
```

**How it works:**
1. Service registers itself with the registry on startup
2. Client queries the registry: "Where is Payment Service?"
3. Registry returns a list of healthy instances
4. Client picks one using a load-balancing algorithm (round-robin, random, etc.)
5. Client calls the chosen instance directly

**Examples:** Netflix Eureka + Ribbon, Spring Cloud LoadBalancer

---

### Model 2: Server-Side Discovery

The **client** sends requests to a **router/load balancer**, which handles finding the right service.

```
┌──────────────────┐         ┌──────────────────┐
│   Order Service  │────────▶│   Load Balancer  │
│   (Client)       │         │   / Router       │
└──────────────────┘         └────────┬─────────┘
                                      │
                         ┌────────────┼────────────┐
                         │            │            │
                         ▼            ▼            ▼
                  ┌───────────┐ ┌───────────┐ ┌───────────┐
                  │ Payment   │ │ Payment   │ │ Payment   │
                  │ Svc #1    │ │ Svc #2    │ │ Svc #3    │
                  └───────────┘ └───────────┘ └───────────┘
                         ▲            ▲            ▲
                         │            │            │
                         └────────────┼────────────┘
                                      │
                              ┌───────┴────────┐
                              │Service Registry│
                              └────────────────┘
```

**How it works:**
1. Services register with the registry
2. Client sends request to a well-known load balancer address
3. Load balancer queries the registry (or is updated by it)
4. Load balancer forwards request to a healthy instance
5. Client doesn't need to know anything about service locations

**Examples:** AWS ALB, Kubernetes Services, Nginx + Consul

---

## Comparison: Client-Side vs Server-Side

| Aspect | Client-Side | Server-Side |
|--------|-------------|-------------|
| **Complexity in client** | High (needs discovery logic) | Low (just call the LB address) |
| **Extra hop** | No (direct call) | Yes (through load balancer) |
| **Language coupling** | Needs library per language | Language-agnostic |
| **Load balancing logic** | In the client | In the router |
| **Single point of failure** | Registry only | Registry + Load Balancer |
| **Latency** | Lower (direct) | Slightly higher (extra hop) |
| **Used by** | Netflix (Eureka+Ribbon) | Kubernetes, AWS ELB |

---

## How It Works Internally

### Service Registration

When a service starts up, it must tell the registry "I exist!" This can happen two ways:

#### Self-Registration (Service registers itself)

```
┌─────────────────────────────────────────────────────────────┐
│                  Service Lifecycle                            │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Service starts up                                         │
│     │                                                         │
│     ▼                                                         │
│  2. POST /register {name: "payment", host: "10.0.2.15",      │
│                      port: 8080, health: "/health"}           │
│     │                                                         │
│     ▼                                                         │
│  3. Registry stores the entry                                 │
│     │                                                         │
│     ▼                                                         │
│  4. Service sends heartbeats every 30s                        │
│     │                                                         │
│     ▼                                                         │
│  5. If heartbeat missed 3 times → registry removes instance   │
│     │                                                         │
│     ▼                                                         │
│  6. On shutdown → POST /deregister                            │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

#### Third-Party Registration (A separate component registers services)

```
┌─────────────────┐     watches     ┌─────────────────┐
│   Registrar /   │ ◄────────────── │  Docker / K8s   │
│   Sidecar       │                 │  (Orchestrator) │
└────────┬────────┘                 └─────────────────┘
         │
         │ Detects new container started
         │
         ▼
┌─────────────────┐
│ Service Registry │
│ (Consul / etcd) │
└─────────────────┘
```

**Examples:** Kubernetes uses this model — the kubelet and kube-proxy handle registration automatically.

---

### Health Checking

The registry needs to know if a service is actually alive and able to handle requests:

```
Registry ──── HTTP GET /health ────▶ Service Instance
         ◀── 200 OK ──────────────── (Healthy)

Registry ──── HTTP GET /health ────▶ Service Instance
         ◀── 503 / Timeout ────────── (Unhealthy → Remove from pool)
```

**Types of health checks:**

| Type | How | When to Use |
|------|-----|-------------|
| **TTL (Heartbeat)** | Service sends "I'm alive" periodically | Simple, low overhead |
| **HTTP Check** | Registry pings `/health` endpoint | Most common for web services |
| **TCP Check** | Registry tries to open a TCP connection | For non-HTTP services |
| **Script Check** | Registry runs a custom script | Complex health logic |
| **gRPC Check** | Uses gRPC Health Checking Protocol | gRPC services |

---

### DNS-Based Service Discovery

The simplest form — services are discovered via DNS:

```
┌──────────────┐                    ┌──────────┐
│Order Service │── DNS query ──────▶│ DNS      │
│              │   "payment.svc"    │ Server   │
│              │◀── A records ──────│          │
│              │   10.0.2.15        │          │
│              │   10.0.2.16        └──────────┘
│              │   10.0.2.17
│              │
│              │── HTTP call ──────▶ 10.0.2.15:8080
└──────────────┘
```

**Kubernetes does this natively:**
- Each Service gets a DNS name: `payment-service.namespace.svc.cluster.local`
- CoreDNS resolves it to the current Pod IPs

**Limitations of DNS-based discovery:**
- DNS caching can return stale IPs
- No built-in load balancing (client gets all IPs)
- No health checking at DNS level
- TTL-based refresh is slow

---

## Code Examples

### Python: Client-Side Discovery with Consul

```python
import consul
import random
import requests

class ServiceDiscovery:
    """Client-side service discovery using HashiCorp Consul."""
    
    def __init__(self, consul_host="localhost", consul_port=8500):
        self.consul = consul.Consul(host=consul_host, port=consul_port)
    
    def register(self, name, host, port, health_endpoint="/health"):
        """Register this service with Consul."""
        self.consul.agent.service.register(
            name=name,
            service_id=f"{name}-{host}-{port}",
            address=host,
            port=port,
            check=consul.Check.http(
                url=f"http://{host}:{port}{health_endpoint}",
                interval="10s",       # Check every 10 seconds
                timeout="5s",         # Timeout after 5 seconds
                deregister="30s"      # Remove if unhealthy for 30s
            )
        )
        print(f"Registered {name} at {host}:{port}")
    
    def discover(self, service_name):
        """Find healthy instances of a service."""
        # Query Consul for healthy instances only
        _, services = self.consul.health.service(
            service_name, passing=True  # Only return healthy instances
        )
        
        instances = []
        for svc in services:
            address = svc["Service"]["Address"]
            port = svc["Service"]["Port"]
            instances.append(f"http://{address}:{port}")
        
        return instances
    
    def call_service(self, service_name, path, method="GET"):
        """Discover a service and call it (with client-side load balancing)."""
        instances = self.discover(service_name)
        
        if not instances:
            raise Exception(f"No healthy instances of {service_name}")
        
        # Simple random load balancing
        target = random.choice(instances)
        url = f"{target}{path}"
        
        response = requests.request(method, url, timeout=5)
        return response.json()


# --- Usage ---
discovery = ServiceDiscovery()

# On service startup: register ourselves
discovery.register("order-service", "10.0.2.10", 8080)

# When we need to call another service
result = discovery.call_service("payment-service", "/api/v1/charge")
```

### Java: Client-Side Discovery with Spring Cloud + Eureka

```java
// --- Service Registration (application.yml) ---
// spring:
//   application:
//     name: payment-service
// eureka:
//   client:
//     serviceUrl:
//       defaultZone: http://eureka-server:8761/eureka/
//   instance:
//     preferIpAddress: true
//     leaseRenewalIntervalInSeconds: 10
//     leaseExpirationDurationInSeconds: 30

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.client.RestTemplate;
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

public class OrderService {
    
    private final DiscoveryClient discoveryClient;
    private final RestTemplate restTemplate;
    
    public OrderService(DiscoveryClient discoveryClient) {
        this.discoveryClient = discoveryClient;
        this.restTemplate = new RestTemplate();
    }
    
    /**
     * Discover payment service and make a call.
     * This demonstrates client-side discovery.
     */
    public PaymentResponse chargeCustomer(String orderId, double amount) {
        // 1. Query the registry for healthy instances
        List<ServiceInstance> instances = 
            discoveryClient.getInstances("payment-service");
        
        if (instances.isEmpty()) {
            throw new RuntimeException("No payment-service instances available!");
        }
        
        // 2. Client-side load balancing (random selection)
        ServiceInstance instance = instances.get(
            ThreadLocalRandom.current().nextInt(instances.size())
        );
        
        // 3. Build URL and call
        String url = String.format("%s/api/v1/charge", instance.getUri());
        
        ChargeRequest request = new ChargeRequest(orderId, amount);
        return restTemplate.postForObject(url, request, PaymentResponse.class);
    }
}

// --- With Spring Cloud LoadBalancer (recommended approach) ---
// Uses @LoadBalanced RestTemplate - discovery + LB is automatic

@Configuration
public class AppConfig {
    @Bean
    @LoadBalanced  // Enables client-side load balancing
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class OrderServiceSimplified {
    private final RestTemplate restTemplate;
    
    public PaymentResponse chargeCustomer(String orderId, double amount) {
        // Spring resolves "payment-service" to an actual instance automatically
        String url = "http://payment-service/api/v1/charge";
        return restTemplate.postForObject(url, 
            new ChargeRequest(orderId, amount), PaymentResponse.class);
    }
}
```

---

## Service Discovery in Kubernetes

Kubernetes has **built-in service discovery** — you don't need Eureka or Consul:

```
┌──────────────────────────────────────────────────────────────┐
│                  Kubernetes Service Discovery                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌─────────────┐                                              │
│  │ Order Pod   │                                              │
│  │             │── GET http://payment-svc:8080/charge         │
│  └──────┬──────┘                                              │
│         │                                                      │
│         ▼ (DNS resolves "payment-svc" to ClusterIP)           │
│  ┌─────────────────────┐                                      │
│  │ Service: payment-svc│                                      │
│  │ ClusterIP: 10.96.1.5│                                      │
│  └──────────┬──────────┘                                      │
│             │ (kube-proxy routes via iptables/IPVS)           │
│             │                                                  │
│    ┌────────┼─────────┐                                       │
│    ▼        ▼         ▼                                       │
│ ┌──────┐ ┌──────┐ ┌──────┐                                   │
│ │Pod 1 │ │Pod 2 │ │Pod 3 │  (Endpoints selected by labels)   │
│ └──────┘ └──────┘ └──────┘                                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```yaml
# Kubernetes Service definition
apiVersion: v1
kind: Service
metadata:
  name: payment-svc
  namespace: production
spec:
  selector:
    app: payment         # Finds all pods with this label
  ports:
    - protocol: TCP
      port: 8080         # Service port
      targetPort: 8080   # Container port
  type: ClusterIP        # Internal only (default)
```

**How Kubernetes DNS works:**
- Service: `payment-svc.production.svc.cluster.local`
- Short form within same namespace: `payment-svc`
- CoreDNS automatically updates when pods change
- `kube-proxy` uses iptables/IPVS for load balancing across pods

---

## Infrastructure Examples

### Docker Compose with Consul

```yaml
version: '3.8'
services:
  consul:
    image: consul:1.15
    ports:
      - "8500:8500"      # UI + API
      - "8600:8600/udp"  # DNS
    command: agent -server -bootstrap -ui -client=0.0.0.0
    
  payment-service:
    image: payment-service:latest
    environment:
      - CONSUL_HOST=consul
      - SERVICE_NAME=payment
      - SERVICE_PORT=8080
    depends_on:
      - consul
    deploy:
      replicas: 3        # 3 instances registered

  order-service:
    image: order-service:latest
    environment:
      - CONSUL_HOST=consul
    depends_on:
      - consul
      - payment-service
```

### Nginx with Consul Template (Server-Side Discovery)

```nginx
# This template is auto-generated by consul-template
# whenever service instances change

upstream payment_backend {
    {{range service "payment"}}
    server {{.Address}}:{{.Port}} max_fails=3 fail_timeout=30s;
    {{end}}
}

server {
    listen 80;
    
    location /api/payment/ {
        proxy_pass http://payment_backend;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }
}
```

---

## Real-World Examples

### Netflix — Eureka

Netflix pioneered client-side service discovery with **Eureka**:
- 600+ microservices all register with Eureka
- Eureka servers are replicated across availability zones
- Clients cache the registry locally (resilient to Eureka outages)
- Combined with **Ribbon** for client-side load balancing
- Each service sends heartbeats every 30 seconds
- Self-preservation mode: if too many services disappear at once, Eureka assumes it's a network partition (not mass failure) and keeps entries

### Uber — Hyperbahn + TChannel

Uber built custom service discovery on top of:
- **Hyperbahn**: A routing mesh where every service registers
- **Ringpop**: Consistent hashing for routing to the right instance
- Designed for extremely low latency at massive scale

### Kubernetes (Google Borg Heritage)

Google's Borg (precursor to K8s) pioneered:
- Built-in DNS-based discovery
- Label-based service selection
- No external registry needed
- Endpoints automatically updated as pods come/go

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | How to Fix |
|---------|-------------|------------|
| **Hardcoding service URLs** | Breaks when services move | Use discovery or DNS names |
| **No health checks** | Traffic goes to dead instances | Always configure health endpoints |
| **Not caching registry results** | Registry becomes single point of failure | Cache locally, refresh periodically |
| **Ignoring DNS TTL** | Stale DNS entries cause 5xx errors | Use short TTLs or real-time discovery |
| **No circuit breaker** | One bad instance takes down caller | Combine with circuit breaker (Ch 12.3) |
| **Relying solely on DNS caching** | Can't react fast to instance failures | Use active health checking |
| **Not handling deregistration** | Zombie entries cause errors | Graceful shutdown + TTL-based expiry |

---

## When to Use / When NOT to Use

### Use Service Discovery When:
- ✅ You have **more than 2-3 services** communicating
- ✅ Services **scale dynamically** (auto-scaling, containers)
- ✅ You're using **containers or Kubernetes**
- ✅ Services are deployed **independently** and frequently
- ✅ You need **zero-downtime deployments**

### Don't Need It When:
- ❌ You have a **monolithic application**
- ❌ Services have **fixed, known addresses** (rare in cloud)
- ❌ You have **only 2-3 services** behind a simple load balancer
- ❌ You're running on a **single server**

### Client-Side vs Server-Side Decision:

| Choose Client-Side When | Choose Server-Side When |
|------------------------|------------------------|
| You want lower latency (no extra hop) | You want simplicity for clients |
| All services use the same language/framework | Services use many different languages |
| You need fine-grained LB control | You want centralized LB management |
| Netflix-style architecture | Kubernetes-style architecture |

---

## Key Takeaways

1. **Service discovery solves the "where is it?" problem** — in dynamic environments, you can't hardcode addresses.

2. **Two main models**: Client-side (client queries registry + load balances) vs Server-side (load balancer handles everything).

3. **Health checking is essential** — discovering a dead service is worse than no discovery at all.

4. **Kubernetes has built-in discovery** — via DNS and Services. You don't need Eureka/Consul in K8s unless you have cross-cluster needs.

5. **Always have a fallback** — cache registry data locally so you survive registry outages.

6. **Combine with other patterns** — service discovery works best with circuit breakers (Chapter 12.3), retries (Chapter 12.2), and load balancing (Chapter 6).

7. **Registration must be automatic** — if humans need to update a registry, the system is fragile.

---

## What's Next?

Now that you understand how services find each other, the next question is: **where do they register, and how is that registry built?** In the next chapter, [02-service-registry.md](./02-service-registry.md), we'll deep-dive into **Service Registries** — Consul, Eureka, etcd — how they work internally, how they replicate, and how they handle failures.
