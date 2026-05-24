# DNS-Based Load Balancing & Failover

> **What you'll learn**: How DNS is used to distribute traffic across multiple servers, implement geographic routing, and automatically failover when servers die — the simplest form of load balancing that requires zero special infrastructure.

---

## Real-Life Analogy

Imagine you run a pizza chain with 5 locations across a city. When someone calls your central phone number asking "where's the nearest pizza place?", the operator:

1. **Looks at where the caller is** → routes them to the closest location
2. **Checks if that location is open** → if closed, suggests the next closest
3. **Balances the load** → if one location is swamped, suggests another

DNS-based load balancing works exactly like this operator. Instead of giving everyone the same server IP, DNS gives different IPs to different people based on various strategies — distributing load, reducing latency, and providing failover.

---

## How DNS Load Balancing Works — The Basic Idea

```
┌─────────────────────────────────────────────────────────────────────┐
│          Traditional (Single Server) vs DNS Load Balanced            │
│                                                                     │
│  TRADITIONAL:                                                       │
│  ┌──────────────┐                    ┌──────────────┐               │
│  │ All users    │─── api.example.com ──▶│ Server 1   │               │
│  │              │    → 10.0.0.1        │ (overloaded)│               │
│  └──────────────┘                    └──────────────┘               │
│                                                                     │
│  DNS LOAD BALANCED:                                                 │
│  ┌──────────────┐                    ┌──────────────┐               │
│  │ User A       │─── → 10.0.0.1 ──────▶│ Server 1   │               │
│  │ User B       │─── → 10.0.0.2 ──────▶│ Server 2   │               │
│  │ User C       │─── → 10.0.0.3 ──────▶│ Server 3   │               │
│  │ User D       │─── → 10.0.0.1 ──────▶│ Server 1   │               │
│  └──────────────┘                    └──────────────┘               │
│                                                                     │
│  Same domain name → Different IPs → Traffic is distributed!         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## DNS Load Balancing Techniques

### 1. Round-Robin DNS (Simplest)

The DNS server returns **multiple A records**, rotating their order for each query:

```
┌──────────────────────────────────────────────────────────────────┐
│              Round-Robin DNS                                       │
│                                                                  │
│  Query 1: "A record for api.example.com?"                        │
│  Answer:  10.0.0.1, 10.0.0.2, 10.0.0.3  (Server 1 first)       │
│                                                                  │
│  Query 2: "A record for api.example.com?"                        │
│  Answer:  10.0.0.2, 10.0.0.3, 10.0.0.1  (Server 2 first)       │
│                                                                  │
│  Query 3: "A record for api.example.com?"                        │
│  Answer:  10.0.0.3, 10.0.0.1, 10.0.0.2  (Server 3 first)       │
│                                                                  │
│  Clients typically use the FIRST IP in the list.                 │
│  By rotating order, traffic gets roughly evenly distributed.     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Zone file for Round-Robin:**
```
api.example.com.    300    IN    A    10.0.0.1
api.example.com.    300    IN    A    10.0.0.2
api.example.com.    300    IN    A    10.0.0.3
```

**Pros & Cons:**
| Pros | Cons |
|------|------|
| Dead simple — no special software | No health checking |
| No single point of failure | Uneven distribution (caching) |
| Free — built into DNS | Can't consider server load |
| Works with any DNS provider | Client may cache a dead server |

---

### 2. Weighted Round-Robin DNS

Send more traffic to more powerful servers by duplicating records or using weights:

```
┌──────────────────────────────────────────────────────────────────┐
│              Weighted DNS Distribution                             │
│                                                                  │
│  Server A (8 CPUs):  Weight 60%  → returned 6 out of 10 times   │
│  Server B (4 CPUs):  Weight 30%  → returned 3 out of 10 times   │
│  Server C (2 CPUs):  Weight 10%  → returned 1 out of 10 times   │
│                                                                  │
│  ┌─────────┐                                                    │
│  │Requests │ ══╦══════════════════╗                              │
│  │         │   ║   60% → Server A ║ (powerful)                   │
│  │         │   ╠══════════════╗   ║                              │
│  │         │   ║  30% → B    ║   ║                              │
│  │         │   ╠════════╗    ║   ║                              │
│  │         │   ║ 10%→ C ║    ║   ║                              │
│  └─────────┘   ╚════════╩════╩═══╝                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**AWS Route 53 Weighted Routing Example:**
```
api.example.com.  300  IN  A  10.0.0.1   ; Weight: 70 (powerful server)
api.example.com.  300  IN  A  10.0.0.2   ; Weight: 20 (medium server)
api.example.com.  300  IN  A  10.0.0.3   ; Weight: 10 (small server)
```

---

### 3. GeoDNS (Geographic Load Balancing)

Return different IPs based on where the user is located:

```
┌─────────────────────────────────────────────────────────────────────┐
│              GeoDNS — Location-Based Routing                         │
│                                                                     │
│                         DNS Server                                   │
│                            │                                         │
│          ┌─────────────────┼─────────────────┐                      │
│          │                 │                 │                       │
│          ▼                 ▼                 ▼                       │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│   │  US User    │  │  EU User    │  │ Asia User   │               │
│   │  → 10.0.1.x│  │  → 10.0.2.x│  │  → 10.0.3.x│               │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘               │
│          │                 │                 │                       │
│          ▼                 ▼                 ▼                       │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│   │ US-East DC  │  │ EU-West DC  │  │ Asia-Pac DC │               │
│   │ Virginia    │  │ Ireland     │  │ Singapore   │               │
│   └─────────────┘  └─────────────┘  └─────────────┘               │
│                                                                     │
│   User in India queries api.example.com → gets Singapore IP         │
│   User in Germany queries api.example.com → gets Ireland IP         │
│   User in Texas queries api.example.com → gets Virginia IP          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**How does DNS know the user's location?**
- **EDNS Client Subnet (ECS)**: Recursive resolver sends the client's subnet to the authoritative server
- **Resolver IP geolocation**: If no ECS, use the resolver's location as a proxy for the user's location

---

### 4. Latency-Based DNS Routing

Return the IP of the server with the **lowest measured latency** to the user:

```
┌─────────────────────────────────────────────────────────────────┐
│              Latency-Based Routing                                │
│                                                                 │
│  User in Mumbai                                                 │
│       │                                                         │
│       │  DNS query: "api.example.com"                           │
│       ▼                                                         │
│  Route 53 checks latency measurements:                          │
│       │                                                         │
│       ├── US-East:   180ms latency from Mumbai                  │
│       ├── EU-West:   150ms latency from Mumbai                  │
│       ├── Singapore: 45ms  latency from Mumbai  ← WINNER!      │
│       │                                                         │
│       ▼                                                         │
│  Returns: Singapore server IP (10.0.3.1)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

This is different from GeoDNS because geographic closeness doesn't always mean lowest latency (network topology matters).

---

### 5. DNS Failover (Health-Checked DNS)

Automatically remove dead servers from DNS responses:

```
┌─────────────────────────────────────────────────────────────────────┐
│              DNS Failover with Health Checks                          │
│                                                                     │
│  NORMAL STATE:                                                      │
│  ┌──────────────┐                                                   │
│  │ DNS Server   │──── api.example.com ──▶ 10.0.0.1 (✓ healthy)     │
│  │              │──── api.example.com ──▶ 10.0.0.2 (✓ healthy)     │
│  │              │──── api.example.com ──▶ 10.0.0.3 (✓ healthy)     │
│  └──────────────┘                                                   │
│                                                                     │
│  Health checker pings all servers every 10 seconds                  │
│                                                                     │
│  FAILURE STATE (Server 2 dies):                                     │
│  ┌──────────────┐                                                   │
│  │ DNS Server   │──── api.example.com ──▶ 10.0.0.1 (✓ healthy)     │
│  │              │──── api.example.com ──▶ 10.0.0.2 (✗ REMOVED!)    │
│  │              │──── api.example.com ──▶ 10.0.0.3 (✓ healthy)     │
│  └──────────────┘                                                   │
│                                                                     │
│  Users now only get 10.0.0.1 or 10.0.0.3                           │
│  Server 2 is removed until health check passes again                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Health check types:**
- **HTTP health check**: GET `/health` → expects 200 OK
- **TCP health check**: Can establish TCP connection?
- **HTTPS health check**: Valid TLS handshake + 200?

---

### 6. Active-Passive DNS Failover

Primary server handles all traffic. Secondary only activates when primary dies:

```
┌─────────────────────────────────────────────────────────────────┐
│              Active-Passive Failover                              │
│                                                                 │
│  NORMAL:                                                        │
│  api.example.com → 10.0.0.1 (Primary - Active)                 │
│                    10.0.0.2 (Secondary - Standing by)            │
│                                                                 │
│  PRIMARY FAILS:                                                 │
│  ┌─────────┐        ┌─────────┐                                │
│  │Primary  │ ✗ DEAD │Secondary│                                 │
│  │10.0.0.1 │        │10.0.0.2 │ ← NOW ACTIVE                   │
│  └─────────┘        └─────────┘                                 │
│                                                                 │
│  DNS automatically switches:                                    │
│  api.example.com → 10.0.0.2 (Secondary - Now Active)           │
│                                                                 │
│  Time to failover = Health check interval + TTL                 │
│  (e.g., 10s check + 60s TTL = 70 seconds max)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### How DNS Resolvers Handle Multiple A Records

When a DNS response contains multiple A records, clients have different behaviors:

```
┌─────────────────────────────────────────────────────────────────┐
│              Client Behavior with Multiple A Records              │
│                                                                 │
│  DNS Response: [10.0.0.1, 10.0.0.2, 10.0.0.3]                  │
│                                                                 │
│  Browser behavior:                                              │
│  ┌────────────────────────────────────────────┐                 │
│  │ Chrome/Firefox/Safari:                      │                 │
│  │ 1. Try first IP (10.0.0.1)                │                 │
│  │ 2. If connection fails → try 10.0.0.2     │                 │
│  │ 3. If that fails → try 10.0.0.3           │                 │
│  │ (Called "Happy Eyeballs" / RFC 8305)       │                 │
│  └────────────────────────────────────────────┘                 │
│                                                                 │
│  curl behavior:                                                 │
│  ┌────────────────────────────────────────────┐                 │
│  │ Uses first IP only (unless --retry)        │                 │
│  └────────────────────────────────────────────┘                 │
│                                                                 │
│  Java HttpClient:                                               │
│  ┌────────────────────────────────────────────┐                 │
│  │ Depends on implementation                  │                 │
│  │ Many try first IP only                     │                 │
│  │ Some implement retry on connection failure │                 │
│  └────────────────────────────────────────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Happy Eyeballs Algorithm (RFC 8305)

Modern browsers implement "Happy Eyeballs" for faster connection establishment:

```
┌─────────────────────────────────────────────────────────────────┐
│              Happy Eyeballs (Dual-Stack Connection Racing)        │
│                                                                 │
│  DNS returns: IPv4 [10.0.0.1] + IPv6 [2001:db8::1]             │
│                                                                 │
│  t=0ms:   Start IPv6 connection to [2001:db8::1]               │
│  t=250ms: Start IPv4 connection to 10.0.0.1 (if IPv6 pending)  │
│  t=???:   Use whichever connects first!                         │
│                                                                 │
│  If multiple A records:                                         │
│  t=0ms:   Try first IP                                          │
│  t=250ms: Try second IP (if first hasn't connected)             │
│  t=500ms: Try third IP (if nothing yet)                         │
│  Winner = first successful connection                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### EDNS Client Subnet (How GeoDNS Works)

```
┌─────────────────────────────────────────────────────────────────┐
│              EDNS Client Subnet (ECS) Flow                        │
│                                                                 │
│  User (IP: 49.207.x.x — India)                                 │
│       │                                                         │
│       │ DNS query to resolver (8.8.8.8)                         │
│       ▼                                                         │
│  Recursive Resolver (8.8.8.8)                                   │
│       │                                                         │
│       │ Adds EDNS option: CLIENT-SUBNET=49.207.0.0/16           │
│       │ (Reveals user's approximate location to authoritative)  │
│       ▼                                                         │
│  Authoritative DNS (Route 53)                                   │
│       │                                                         │
│       │ Looks up 49.207.0.0/16 → India                         │
│       │ Returns: Singapore server (closest to India)             │
│       ▼                                                         │
│  Response: 10.0.3.1 (Singapore DC)                              │
│  + SCOPE: 49.207.0.0/16 (cache this answer for this subnet)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### DNS Failover Timing Math

```
Total failover time = Detection time + Propagation time

Detection time:
  = Health check interval × failure threshold
  = 10 seconds × 3 consecutive failures
  = 30 seconds

Propagation time:
  = DNS TTL (time for caches to expire)
  = 60 seconds (if TTL is 60)

TOTAL WORST CASE = 30 + 60 = 90 seconds
TOTAL BEST CASE  = 10 + 0  = 10 seconds (if TTL just expired)

┌──────────────────────────────────────────────────┐
│  Lower TTL = Faster failover BUT more DNS queries │
│  Higher TTL = Slower failover BUT less DNS load   │
│                                                  │
│  Production recommendation:                       │
│  - Normal operations: TTL = 300 seconds          │
│  - During migrations: TTL = 60 seconds           │
│  - Critical services: TTL = 30-60 seconds        │
└──────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — DNS Round-Robin Implementation

```python
import dns.resolver
import random
from collections import Counter

def demonstrate_round_robin(domain, num_queries=20):
    """
    Show how DNS round-robin distributes queries.
    Each query may return IPs in different order.
    """
    ip_counter = Counter()
    
    resolver = dns.resolver.Resolver()
    resolver.nameservers = ['8.8.8.8']
    resolver.cache = dns.resolver.LRUCache(0)  # Disable cache to see rotation
    
    for i in range(num_queries):
        try:
            answers = resolver.resolve(domain, 'A')
            first_ip = str(answers[0])  # Client typically uses first IP
            ip_counter[first_ip] += 1
            
            all_ips = [str(rr) for rr in answers]
            print(f"  Query {i+1}: {all_ips} → using {first_ip}")
        except Exception as e:
            print(f"  Query {i+1}: Error - {e}")
    
    print(f"\n  Distribution after {num_queries} queries:")
    for ip, count in ip_counter.most_common():
        bar = '█' * count
        print(f"    {ip}: {count} times {bar}")

demonstrate_round_robin("google.com")
```

### Python — Health-Checked DNS Failover System

```python
import time
import threading
import requests
from typing import Dict, List

class DnsFailoverManager:
    """
    A simple health-checked DNS failover system.
    In production, use Route 53, Cloudflare, or similar.
    """
    
    def __init__(self, domain: str, servers: List[Dict]):
        """
        servers: [{"ip": "10.0.0.1", "health_url": "http://10.0.0.1/health"}]
        """
        self.domain = domain
        self.servers = servers
        self.healthy_servers = set(s["ip"] for s in servers)
        self.check_interval = 10  # seconds
        self.failure_threshold = 3  # consecutive failures before removal
        self.failure_counts: Dict[str, int] = {s["ip"]: 0 for s in servers}
        self._running = False
    
    def start_health_checks(self):
        """Start background health checking thread."""
        self._running = True
        thread = threading.Thread(target=self._health_check_loop, daemon=True)
        thread.start()
        print(f"  Health checker started for {self.domain}")
    
    def _health_check_loop(self):
        while self._running:
            for server in self.servers:
                self._check_server(server)
            time.sleep(self.check_interval)
    
    def _check_server(self, server: Dict):
        ip = server["ip"]
        try:
            resp = requests.get(server["health_url"], timeout=5)
            if resp.status_code == 200:
                # Server is healthy
                self.failure_counts[ip] = 0
                if ip not in self.healthy_servers:
                    self.healthy_servers.add(ip)
                    print(f"  ✓ Server {ip} recovered — added back to pool")
            else:
                self._record_failure(ip)
        except requests.RequestException:
            self._record_failure(ip)
    
    def _record_failure(self, ip: str):
        self.failure_counts[ip] += 1
        if self.failure_counts[ip] >= self.failure_threshold:
            if ip in self.healthy_servers:
                self.healthy_servers.discard(ip)
                print(f"  ✗ Server {ip} REMOVED — {self.failure_threshold} consecutive failures")
    
    def resolve(self) -> List[str]:
        """Return list of healthy server IPs (simulates DNS response)."""
        healthy = list(self.healthy_servers)
        random.shuffle(healthy)  # Simple round-robin
        return healthy
    
    def stop(self):
        self._running = False

# Usage
manager = DnsFailoverManager("api.example.com", [
    {"ip": "10.0.0.1", "health_url": "http://10.0.0.1:8080/health"},
    {"ip": "10.0.0.2", "health_url": "http://10.0.0.2:8080/health"},
    {"ip": "10.0.0.3", "health_url": "http://10.0.0.3:8080/health"},
])
manager.start_health_checks()
```

### Java — Weighted DNS Resolution

```java
import java.util.*;
import java.util.concurrent.ThreadLocalRandom;

public class WeightedDnsResolver {
    
    // Simulates weighted DNS resolution
    private final List<WeightedServer> servers;
    private final int totalWeight;
    
    public WeightedDnsResolver(List<WeightedServer> servers) {
        this.servers = servers;
        this.totalWeight = servers.stream()
            .mapToInt(WeightedServer::getWeight)
            .sum();
    }
    
    /**
     * Returns a server IP based on weighted random selection.
     * Higher weight = more likely to be selected.
     */
    public String resolve() {
        int random = ThreadLocalRandom.current().nextInt(totalWeight);
        int cumulative = 0;
        
        for (WeightedServer server : servers) {
            cumulative += server.getWeight();
            if (random < cumulative) {
                return server.getIp();
            }
        }
        return servers.get(0).getIp(); // Fallback
    }
    
    public static void main(String[] args) {
        // Configure: US-East gets 60%, EU-West 30%, Asia 10%
        List<WeightedServer> servers = List.of(
            new WeightedServer("10.0.1.1", 60, "us-east"),
            new WeightedServer("10.0.2.1", 30, "eu-west"),
            new WeightedServer("10.0.3.1", 10, "ap-south")
        );
        
        WeightedDnsResolver resolver = new WeightedDnsResolver(servers);
        
        // Simulate 1000 queries and show distribution
        Map<String, Integer> distribution = new HashMap<>();
        for (int i = 0; i < 1000; i++) {
            String ip = resolver.resolve();
            distribution.merge(ip, 1, Integer::sum);
        }
        
        System.out.println("Distribution over 1000 queries:");
        distribution.forEach((ip, count) -> 
            System.out.printf("  %s: %d (%.1f%%)%n", ip, count, count / 10.0));
    }
}

record WeightedServer(String ip, int weight, String region) {
    public String getIp() { return ip; }
    public int getWeight() { return weight; }
}
```

### Java — GeoDNS Resolver Simulation

```java
import java.util.*;

public class GeoDnsResolver {
    
    // Region → Server IPs mapping
    private final Map<String, List<String>> regionServers;
    // IP range → Region mapping (simplified)
    private final Map<String, String> ipToRegion;
    
    public GeoDnsResolver() {
        regionServers = Map.of(
            "us-east", List.of("10.0.1.1", "10.0.1.2"),
            "eu-west", List.of("10.0.2.1", "10.0.2.2"),
            "ap-south", List.of("10.0.3.1", "10.0.3.2")
        );
        
        // Simplified IP-to-region mapping
        // In production: MaxMind GeoIP database
        ipToRegion = Map.of(
            "203.0.113", "ap-south",   // India
            "198.51.100", "us-east",   // US
            "192.0.2", "eu-west"       // Europe
        );
    }
    
    /**
     * Resolve domain based on client's IP address (geographic routing).
     */
    public List<String> resolve(String domain, String clientIp) {
        String region = detectRegion(clientIp);
        List<String> servers = regionServers.getOrDefault(region, 
            regionServers.get("us-east")); // Default to US
        
        System.out.printf("  Client %s → Region: %s → Servers: %s%n",
            clientIp, region, servers);
        return servers;
    }
    
    private String detectRegion(String clientIp) {
        String prefix = clientIp.substring(0, clientIp.lastIndexOf('.'));
        return ipToRegion.getOrDefault(prefix, "us-east");
    }
    
    public static void main(String[] args) {
        GeoDnsResolver resolver = new GeoDnsResolver();
        
        // Simulate queries from different regions
        resolver.resolve("api.example.com", "203.0.113.50");   // India
        resolver.resolve("api.example.com", "198.51.100.25");  // US
        resolver.resolve("api.example.com", "192.0.2.100");    // Europe
    }
}
```

---

## Infrastructure Examples

### AWS Route 53 — Weighted Routing Policy

```hcl
# Terraform: Route 53 weighted routing

# 70% traffic to us-east-1
resource "aws_route53_record" "api_us_east" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "us-east-1"
  
  weighted_routing_policy {
    weight = 70
  }
  
  alias {
    name                   = aws_lb.us_east.dns_name
    zone_id                = aws_lb.us_east.zone_id
    evaluate_target_health = true
  }
}

# 30% traffic to eu-west-1
resource "aws_route53_record" "api_eu_west" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "eu-west-1"
  
  weighted_routing_policy {
    weight = 30
  }
  
  alias {
    name                   = aws_lb.eu_west.dns_name
    zone_id                = aws_lb.eu_west.zone_id
    evaluate_target_health = true
  }
}

# Health check for failover
resource "aws_route53_health_check" "api_us_east" {
  fqdn              = "api-us-east.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10
  
  tags = {
    Name = "api-us-east-health-check"
  }
}
```

### AWS Route 53 — Latency-Based Routing

```hcl
# Route based on lowest latency to user
resource "aws_route53_record" "api_latency_us" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "us-east-1"
  
  latency_routing_policy {
    region = "us-east-1"
  }
  
  alias {
    name                   = aws_lb.us_east.dns_name
    zone_id                = aws_lb.us_east.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_latency_eu" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "eu-west-1"
  
  latency_routing_policy {
    region = "eu-west-1"
  }
  
  alias {
    name                   = aws_lb.eu_west.dns_name
    zone_id                = aws_lb.eu_west.zone_id
    evaluate_target_health = true
  }
}
```

### AWS Route 53 — Failover Routing

```hcl
# Primary-Secondary failover
resource "aws_route53_record" "api_primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "primary"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
  
  health_check_id = aws_route53_health_check.primary.id
}

resource "aws_route53_record" "api_secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "secondary"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
  
  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

### Cloudflare DNS Load Balancing Configuration

```json
{
  "description": "API Load Balancer",
  "name": "api.example.com",
  "fallback_pool": "pool-eu-west",
  "default_pools": ["pool-us-east", "pool-eu-west", "pool-ap-south"],
  "region_pools": {
    "WNAM": ["pool-us-east"],
    "ENAM": ["pool-us-east"],
    "WEU": ["pool-eu-west"],
    "EEU": ["pool-eu-west"],
    "SAS": ["pool-ap-south"],
    "SEAS": ["pool-ap-south"]
  },
  "proxied": true,
  "steering_policy": "geo",
  "session_affinity": "cookie",
  "pools": [
    {
      "id": "pool-us-east",
      "name": "US East Pool",
      "origins": [
        {"name": "us-east-1", "address": "10.0.1.1", "weight": 1},
        {"name": "us-east-2", "address": "10.0.1.2", "weight": 1}
      ],
      "monitor": "health-check-https"
    }
  ]
}
```

---

## Real-World Example

### How Netflix Uses DNS for Global Load Balancing

```
┌─────────────────────────────────────────────────────────────────────┐
│            Netflix's Multi-Tier DNS Strategy                          │
│                                                                     │
│  User opens Netflix app                                             │
│       │                                                             │
│       ▼                                                             │
│  DNS Query: api.netflix.com                                         │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────┐                        │
│  │  Tier 1: Route 53 GeoDNS               │                        │
│  │  Routes to nearest AWS region           │                        │
│  │  (US-East, EU-West, AP-Southeast, etc.) │                        │
│  └──────────────────┬──────────────────────┘                        │
│                     │                                               │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                        │
│  │  Tier 2: Regional Load Balancer (ALB)    │                        │
│  │  Distributes across AZs within region   │                        │
│  └──────────────────┬──────────────────────┘                        │
│                     │                                               │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                        │
│  │  Tier 3: Zuul API Gateway               │                        │
│  │  Routes to specific microservice        │                        │
│  └─────────────────────────────────────────┘                        │
│                                                                     │
│  FAILOVER: If US-East health check fails:                           │
│  Route 53 stops returning US-East IPs                               │
│  Users get routed to US-West (next closest)                         │
│  Failover time: ~30-90 seconds (TTL=60)                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### How GitHub Uses DNS Failover

GitHub uses Fastly (CDN) with DNS-based failover:
- Primary: Fastly edge servers (Anycast)
- Fallback: Direct to GitHub's origin servers
- Health checks every 10 seconds
- TTL of 60 seconds for fast failover
- During major Fastly outage (June 2021): failover kicked in within ~1 minute

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| TTL too high for failover | Dead server served for hours | Use TTL ≤ 60s for critical failover |
| No health checks | Dead servers stay in DNS | Always pair DNS LB with health checks |
| Relying on DNS alone for LB | Uneven distribution due to caching | Use DNS + application-layer LB together |
| Not accounting for resolver caching | Some ISPs ignore low TTLs | Accept that DNS failover isn't instant |
| CNAME at apex for CDN | Breaks other records | Use ALIAS/ANAME or provider-specific solution |
| Not testing failover | Discover bugs during actual outage | Regular failover drills |
| Ignoring EDNS Client Subnet | GeoDNS routes based on resolver, not user | Enable ECS at authoritative DNS |
| Round-robin without health checks | 1/N users hit dead server | Add health-check DNS or client retry |

---

## DNS LB vs Application-Layer LB — When to Use Each

```
┌──────────────────────────────────────────────────────────────────────┐
│              Comparison: DNS LB vs L4/L7 Load Balancer                 │
├──────────────────────┬──────────────────────┬────────────────────────┤
│ Feature              │ DNS Load Balancing    │ L4/L7 Load Balancer    │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Granularity          │ Per DNS query         │ Per connection/request │
│ Health checks        │ Slow (10-60s)         │ Fast (1-5s)            │
│ Failover speed       │ Slow (TTL-dependent)  │ Instant                │
│ Session affinity     │ Difficult             │ Easy (cookies/headers) │
│ SSL termination      │ No                    │ Yes                    │
│ Cost                 │ Very low              │ Moderate-High          │
│ Geographic routing   │ Excellent             │ Limited (single region)│
│ Infrastructure       │ None (just DNS)       │ Need LB instances      │
│ Scale                │ Global                │ Regional               │
│ Algorithm options    │ Limited               │ Many                   │
└──────────────────────┴──────────────────────┴────────────────────────┘

BEST PRACTICE: Use BOTH together!
- DNS for global/geographic routing (between regions)
- L4/L7 LB for local distribution (within a region)
```

---

## When to Use / When NOT to Use

### Use DNS Load Balancing When:
- ✅ Distributing traffic across geographic regions
- ✅ Simple round-robin for non-critical services
- ✅ Primary/backup failover across data centers
- ✅ You need global load distribution without infrastructure cost
- ✅ As the first tier in a multi-tier LB strategy

### Do NOT Use DNS LB Alone When:
- ❌ You need instant failover (< 10 seconds)
- ❌ You need per-request load balancing
- ❌ You need session stickiness
- ❌ You need to balance based on server load/capacity
- ❌ You need SSL offloading or request inspection
- ❌ You have only servers in one data center

---

## Key Takeaways

- **DNS load balancing is the simplest form of traffic distribution** — just return multiple IPs or different IPs based on strategy.
- **Round-Robin DNS** rotates through IPs but offers no health awareness or load awareness.
- **GeoDNS** routes users to the nearest server based on their geographic location (using EDNS Client Subnet).
- **DNS failover** removes dead servers from responses, but is limited by TTL (not instant like L4/L7 LB).
- **Failover time = health check detection + DNS TTL** — typically 30-120 seconds.
- **Best practice: use DNS for global routing + L7 load balancer for local distribution** — they complement each other.
- **Never rely on DNS load balancing alone** for production systems — always pair with application-layer load balancing.

---

## What's Next?

Understanding DNS caching and TTL is crucial for controlling how fast DNS changes propagate. See [04-dns-caching-ttl.md](./04-dns-caching-ttl.md).
