# Global Server Load Balancing (GSLB) & GeoDNS

> **What you'll learn**: How to route users to the nearest data center across the globe using DNS-level load balancing, how GeoDNS resolves domain names to region-specific IPs, failover strategies across continents, and the architecture behind sub-100ms global response times that companies like Netflix, Google, and Cloudflare achieve.

---

## Real-Life Analogy — International Pizza Chain

Imagine you call "Dominos Pizza" from Tokyo. The phone system doesn't connect you to a Dominos in New York — it connects you to the **nearest Dominos in Tokyo**. The phone number is the same globally, but the actual store you reach depends on WHERE you're calling from.

Now imagine:
- If the Tokyo store is closed (server down) → route to Osaka store (nearest alternative)
- If there's a natural disaster in Japan (entire region down) → route to Korea
- If it's peak dinner time in Tokyo → route some overflow to a less busy nearby store

**GSLB is this "smart phone system" for the internet.** Same domain name, different server depending on your location, server health, and load.

```
Traditional LB (single region):          GSLB (multi-region):
                                          
User (Tokyo) ──▶ LB ──▶ Servers         User (Tokyo) ──▶ DNS ──▶ Tokyo servers
User (NYC)   ──▶ LB ──▶ Servers         User (NYC)   ──▶ DNS ──▶ NYC servers
User (London)──▶ LB ──▶ Servers         User (London)──▶ DNS ──▶ London servers
                                          
Everyone hits same region!                Each user hits NEAREST region!
Tokyo user: 200ms latency                Tokyo user: 10ms latency
```

---

## Core Concept Explained Step-by-Step

### Step 1: The Latency Problem

```
SPEED OF LIGHT LIMITATION:

Light travels through fiber optic cable at ~200,000 km/s.
Round-trip time (RTT) = physical distance × 2 / speed

┌─────────────────────────────────────────────────────────────────┐
│  User Location → Server Location     │  Distance  │  Min RTT   │
├──────────────────────────────────────┼────────────┼────────────┤
│  Tokyo → Tokyo (same city)           │    20 km   │   <1 ms    │
│  Tokyo → Singapore                   │  5,300 km  │   ~53 ms   │
│  Tokyo → US West (California)        │  8,500 km  │   ~85 ms   │
│  Tokyo → US East (Virginia)          │ 11,000 km  │  ~110 ms   │
│  Tokyo → London                      │  9,500 km  │   ~95 ms   │
│  Sydney → London                     │ 17,000 km  │  ~170 ms   │
└──────────────────────────────────────┴────────────┴────────────┘

These are MINIMUMS — actual latency is 2-3x higher due to routing,
congestion, and processing.

If ALL your servers are in US-East:
- US users: 20-40ms (great!)
- European users: 80-120ms (okay)
- Asian users: 200-300ms (terrible!)
- Australian users: 250-350ms (awful!)
```

### Step 2: How GeoDNS Works

```
NORMAL DNS (without geo-awareness):
www.myapp.com → always returns 52.23.186.100 (US-East server)
Everyone hits the same IP regardless of location.

GEODNS (geo-aware DNS):
www.myapp.com → DEPENDS on where you ask from:
  - Asked from Japan?   → 13.231.45.67   (Tokyo server)
  - Asked from Germany? → 3.121.78.90    (Frankfurt server)
  - Asked from Brazil?  → 54.207.12.34   (São Paulo server)
  - Asked from US?      → 52.23.186.100  (US-East server)

HOW? The DNS server knows the client's approximate location
from their IP address (GeoIP database).

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Step 1: User in Tokyo types www.myapp.com                      │
│                                                                  │
│  Step 2: DNS query reaches GeoDNS server                        │
│          Query source IP: 203.0.113.50 (Tokyo IP range)         │
│                                                                  │
│  Step 3: GeoDNS looks up IP → Location mapping                  │
│          "203.0.113.50 is in Tokyo, Japan, Asia-Pacific"        │
│                                                                  │
│  Step 4: GeoDNS returns nearest server IP                       │
│          "For Asia-Pacific, return 13.231.45.67 (Tokyo DC)"     │
│                                                                  │
│  Step 5: User connects to Tokyo server → 10ms latency!          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Step 3: GSLB — More Than Just GeoDNS

```
GSLB = GeoDNS + Health Monitoring + Traffic Management

┌─────────────────────────────────────────────────────────────────────┐
│                         GSLB CAPABILITIES                            │
│                                                                     │
│  1. GEOGRAPHIC ROUTING (GeoDNS)                                     │
│     Route users to nearest healthy region                           │
│                                                                     │
│  2. HEALTH MONITORING                                               │
│     Continuously check if each region is alive and responsive       │
│                                                                     │
│  3. FAILOVER                                                        │
│     If Tokyo DC goes down, route Tokyo users to Singapore           │
│                                                                     │
│  4. LOAD-BASED ROUTING                                             │
│     If US-East is overloaded (85% CPU), shift traffic to US-West   │
│                                                                     │
│  5. LATENCY-BASED ROUTING                                          │
│     Measure actual latency to each DC, pick the fastest            │
│     (geography ≠ lowest latency due to network topology)           │
│                                                                     │
│  6. WEIGHTED ROUTING                                                │
│     Send 80% to primary, 20% to canary region                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 4: GSLB Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                                                                       │
│   User (Tokyo)                                                        │
│      │                                                                │
│      │ DNS Query: "What's the IP for www.myapp.com?"                 │
│      ▼                                                                │
│   ┌──────────────────┐                                               │
│   │  Local DNS       │  (ISP's recursive resolver)                    │
│   │  Resolver        │                                                │
│   └────────┬─────────┘                                               │
│            │                                                          │
│            ▼                                                          │
│   ┌──────────────────┐                                               │
│   │  GSLB / GeoDNS   │  (AWS Route 53, Cloudflare DNS, NS1)         │
│   │  Authoritative   │                                                │
│   │  Name Server     │                                                │
│   │                  │                                                │
│   │  Decision Logic: │                                                │
│   │  ├── Client location: Tokyo, Japan                               │
│   │  ├── Tokyo DC health: ✅ HEALTHY                                  │
│   │  ├── Tokyo DC load: 45% (fine)                                   │
│   │  └── Decision: Return Tokyo DC IP                                │
│   └────────┬─────────┘                                               │
│            │                                                          │
│            │ Response: A Record → 13.231.45.67 (Tokyo)               │
│            ▼                                                          │
│   ┌──────────────────┐       ┌──────────────────┐                   │
│   │  Tokyo DC        │       │  US-East DC      │  (not chosen)      │
│   │  ┌────────────┐  │       │  ┌────────────┐  │                   │
│   │  │  Local LB  │  │       │  │  Local LB  │  │                   │
│   │  └─────┬──────┘  │       │  └────────────┘  │                   │
│   │    ┌───┼───┐     │       │                  │                   │
│   │    ▼   ▼   ▼     │       └──────────────────┘                   │
│   │  ┌─┐ ┌─┐ ┌─┐    │       ┌──────────────────┐                   │
│   │  │S│ │S│ │S│    │       │  London DC       │  (not chosen)      │
│   │  │1│ │2│ │3│    │       │                  │                   │
│   │  └─┘ └─┘ └─┘    │       └──────────────────┘                   │
│   └──────────────────┘                                               │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Step 5: Failover Across Regions

```
SCENARIO: Tokyo data center goes completely offline

BEFORE FAILURE:
┌────────────────────────────────────────────┐
│ GSLB routing table:                        │
│  Asia users     → Tokyo DC     (primary)   │
│  Europe users   → London DC    (primary)   │
│  Americas users → US-East DC   (primary)   │
└────────────────────────────────────────────┘

TOKYO DC FAILS: Health checks detect failure (30-60 seconds)

AFTER FAILOVER:
┌────────────────────────────────────────────┐
│ GSLB routing table (updated):              │
│  Asia users     → Singapore DC (failover!) │
│  Europe users   → London DC    (primary)   │
│  Americas users → US-East DC   (primary)   │
└────────────────────────────────────────────┘

User experience:
- Tokyo users: latency goes from 10ms → 50ms (Singapore)
- Still much better than 200ms to US-East!
- Service continues without interruption

RECOVERY:
When Tokyo DC is back online:
- GSLB detects health recovery
- Gradually shifts traffic back (not all at once!)
- 10% → 25% → 50% → 100% over ~10 minutes
```

---

## How It Works Internally

### DNS TTL and Propagation

```
CRITICAL CONCEPT: DNS Time-To-Live (TTL)

When GSLB returns an IP, it also sets a TTL:
"This answer is valid for X seconds. After that, ask again."

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Short TTL (30-60 seconds):                                 │
│  ✅ Fast failover (clients re-query quickly)                 │
│  ❌ More DNS traffic (higher load on DNS servers)            │
│  ❌ Slightly higher latency (DNS lookup more often)          │
│                                                              │
│  Long TTL (300-3600 seconds):                               │
│  ✅ Less DNS traffic, cached locally                         │
│  ✅ Lower latency (fewer lookups)                            │
│  ❌ Slow failover (clients keep using stale IP for minutes!) │
│                                                              │
│  RECOMMENDATION:                                             │
│  Normal: TTL = 60s (balance of speed and caching)           │
│  During incidents: Drop TTL to 30s beforehand               │
│  Static sites: TTL = 300s (infrequent changes)              │
│                                                              │
└──────────────────────────────────────────────────────────────┘

FAILOVER TIMELINE (with 60s TTL):

T=0:    Tokyo DC goes down
T=30s:  GSLB health check detects failure
T=30s:  GSLB updates DNS to point to Singapore
T=30-90s: Clients' cached DNS expires, they re-query
T=90s:  Most traffic has shifted to Singapore

Total failover time: ~90 seconds (acceptable for most apps)
```

### Anycast vs Unicast for GSLB

```
UNICAST (traditional):
Each server has a unique IP. DNS returns different IPs per region.
  Tokyo DC: 13.231.45.67
  London DC: 3.121.78.90
  US-East:   52.23.186.100

ANYCAST (modern, used by Cloudflare/Google):
ALL locations advertise the SAME IP via BGP routing.
Network automatically routes to nearest location.
  All DCs: 1.2.3.4 (same IP, but reaches different physical servers!)

┌─────────────────────────────────────────────────────────────────────┐
│                       ANYCAST ROUTING                                │
│                                                                     │
│  User in Tokyo                                                      │
│    └── Network says: "1.2.3.4 is 2 hops away via Tokyo"           │
│    └── Routes to Tokyo DC ✅                                        │
│                                                                     │
│  User in London                                                     │
│    └── Network says: "1.2.3.4 is 3 hops away via London"          │
│    └── Routes to London DC ✅                                       │
│                                                                     │
│  If Tokyo DC goes down:                                             │
│    └── Network: "1.2.3.4 no longer reachable via Tokyo"            │
│    └── Next best: "1.2.3.4 is 5 hops away via Singapore"          │
│    └── Automatic failover! No DNS changes needed! ✅                │
│                                                                     │
│  BENEFIT: Failover in seconds (BGP convergence)                     │
│  vs minutes (DNS TTL expiry)                                        │
└─────────────────────────────────────────────────────────────────────┘
```

### GSLB Routing Policies

```
POLICY 1: GEOLOCATION (most common)
─────────────────────────────────────
Map regions to data centers:
  Asia-Pacific  → ap-northeast-1 (Tokyo)
  Europe        → eu-west-1 (Ireland)
  Americas      → us-east-1 (Virginia)
  Default       → us-east-1

POLICY 2: LATENCY-BASED
─────────────────────────────────────
Measure actual network latency from DNS resolver locations.
Return the DC with lowest measured latency.
(More accurate than geography alone!)

POLICY 3: WEIGHTED
─────────────────────────────────────
Distribute traffic by percentage:
  us-east-1: 60%  (main DC)
  us-west-2: 30%  (secondary)
  eu-west-1: 10%  (canary)

POLICY 4: FAILOVER (active-passive)
─────────────────────────────────────
Primary: us-east-1
Secondary: us-west-2
Return secondary ONLY if primary is unhealthy.

POLICY 5: MULTI-VALUE (simple round-robin)
─────────────────────────────────────
Return multiple IPs, let client pick.
DNS: [52.23.186.100, 13.231.45.67, 3.121.78.90]
```

---

## Code Examples

### Python — GSLB Routing Simulator

```python
# gslb_router.py — Simulates GeoDNS routing decisions
import math
from dataclasses import dataclass

@dataclass
class DataCenter:
    name: str
    region: str
    ip: str
    lat: float
    lon: float
    healthy: bool = True
    load_percent: int = 0

# Global data centers
DATA_CENTERS = [
    DataCenter("Tokyo", "ap-northeast-1", "13.231.45.67", 35.68, 139.69),
    DataCenter("Singapore", "ap-southeast-1", "54.169.12.34", 1.35, 103.82),
    DataCenter("London", "eu-west-2", "3.121.78.90", 51.51, -0.13),
    DataCenter("Virginia", "us-east-1", "52.23.186.100", 37.43, -79.46),
    DataCenter("California", "us-west-1", "54.183.22.33", 37.77, -122.42),
]

def haversine_distance(lat1, lon1, lat2, lon2) -> float:
    """Calculate distance between two points on Earth (km)."""
    R = 6371  # Earth's radius in km
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = math.sin(dlat/2)**2 + math.cos(math.radians(lat1)) * \
        math.cos(math.radians(lat2)) * math.sin(dlon/2)**2
    return R * 2 * math.asin(math.sqrt(a))

def resolve_gslb(client_lat: float, client_lon: float, policy="geo") -> DataCenter:
    """Simulate GSLB resolution based on client location."""
    healthy_dcs = [dc for dc in DATA_CENTERS if dc.healthy]
    
    if policy == "geo":
        # Pick nearest healthy DC by geographic distance
        return min(healthy_dcs, key=lambda dc: 
            haversine_distance(client_lat, client_lon, dc.lat, dc.lon))
    
    elif policy == "least_load":
        # Pick healthy DC with lowest load
        return min(healthy_dcs, key=lambda dc: dc.load_percent)
    
    elif policy == "failover":
        # Always return primary (Virginia) unless unhealthy
        primary = next((dc for dc in healthy_dcs if dc.region == "us-east-1"), None)
        return primary or healthy_dcs[0]

# Simulate DNS resolution for users worldwide
users = [
    ("User in Tokyo", 35.68, 139.69),
    ("User in Berlin", 52.52, 13.40),
    ("User in New York", 40.71, -74.01),
]

for name, lat, lon in users:
    dc = resolve_gslb(lat, lon)
    dist = haversine_distance(lat, lon, dc.lat, dc.lon)
    print(f"{name} → {dc.name} ({dc.ip}) [{dist:.0f}km away]")
```

### Java — GSLB Health Monitor

```java
// GSLBHealthMonitor.java — Monitors data centers and triggers failover
import java.net.http.*;
import java.net.URI;
import java.time.Duration;
import java.util.*;
import java.util.concurrent.*;

public class GSLBHealthMonitor {
    
    record DataCenter(String name, String region, String healthUrl, boolean primary) {}
    
    private final List<DataCenter> dataCenters;
    private final Map<String, Boolean> healthStatus = new ConcurrentHashMap<>();
    private final Map<String, String> routingTable = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);
    
    public GSLBHealthMonitor(List<DataCenter> dataCenters) {
        this.dataCenters = dataCenters;
        dataCenters.forEach(dc -> healthStatus.put(dc.name(), true));
    }
    
    // Start periodic health checks for all data centers
    public void startMonitoring() {
        for (DataCenter dc : dataCenters) {
            scheduler.scheduleAtFixedRate(
                () -> checkHealth(dc),
                0, 10, TimeUnit.SECONDS  // Every 10 seconds
            );
        }
    }
    
    private void checkHealth(DataCenter dc) {
        try {
            HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(5))
                .build();
            HttpResponse<String> response = client.send(
                HttpRequest.newBuilder().uri(URI.create(dc.healthUrl())).build(),
                HttpResponse.BodyHandlers.ofString()
            );
            
            boolean healthy = response.statusCode() == 200;
            boolean wasHealthy = healthStatus.getOrDefault(dc.name(), true);
            healthStatus.put(dc.name(), healthy);
            
            if (wasHealthy && !healthy) {
                System.out.printf("ALERT: %s (%s) is DOWN! Triggering failover...%n",
                    dc.name(), dc.region());
                triggerFailover(dc);
            } else if (!wasHealthy && healthy) {
                System.out.printf("RECOVERY: %s (%s) is back UP%n", dc.name(), dc.region());
                triggerRecovery(dc);
            }
        } catch (Exception e) {
            healthStatus.put(dc.name(), false);
            System.out.printf("ALERT: %s unreachable: %s%n", dc.name(), e.getMessage());
        }
    }
    
    private void triggerFailover(DataCenter failedDc) {
        // Update routing: redirect traffic to nearest healthy DC
        System.out.printf("Failover: %s traffic → nearest healthy DC%n", failedDc.region());
    }
    
    private void triggerRecovery(DataCenter recoveredDc) {
        // Gradually shift traffic back (not all at once)
        System.out.printf("Recovery: Gradually restoring traffic to %s%n", recoveredDc.name());
    }
}
```

---

## Infrastructure Examples

### AWS Route 53 — GeoDNS + Health Checks

```bash
# Create health check for Tokyo DC
aws route53 create-health-check --caller-reference tokyo-$(date +%s) \
  --health-check-config '{
    "IPAddress": "13.231.45.67",
    "Port": 443,
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "RequestInterval": 10,
    "FailureThreshold": 3
  }'

# Create geolocation routing policy
# Asia → Tokyo
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.myapp.com",
        "Type": "A",
        "SetIdentifier": "tokyo",
        "GeoLocation": {"ContinentCode": "AS"},
        "TTL": 60,
        "ResourceRecords": [{"Value": "13.231.45.67"}],
        "HealthCheckId": "health-check-tokyo-id"
      }
    }]
  }'

# Europe → London
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.myapp.com",
        "Type": "A",
        "SetIdentifier": "london",
        "GeoLocation": {"ContinentCode": "EU"},
        "TTL": 60,
        "ResourceRecords": [{"Value": "3.121.78.90"}],
        "HealthCheckId": "health-check-london-id"
      }
    }]
  }'

# Default (everyone else) → US-East
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.myapp.com",
        "Type": "A",
        "SetIdentifier": "default",
        "GeoLocation": {"CountryCode": "*"},
        "TTL": 60,
        "ResourceRecords": [{"Value": "52.23.186.100"}],
        "HealthCheckId": "health-check-us-east-id"
      }
    }]
  }'
```

### Cloudflare DNS — Built-in GeoDNS

```
Cloudflare's approach (Anycast — simpler than traditional GeoDNS):

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Cloudflare has 300+ data centers worldwide.                   │
│  They use ANYCAST: same IP, nearest location automatically.     │
│                                                                 │
│  You just point DNS to Cloudflare:                             │
│  www.myapp.com → CNAME → myapp.cdn.cloudflare.net              │
│                                                                 │
│  Cloudflare handles:                                            │
│  ✅ GeoDNS routing (automatic, no config needed)                │
│  ✅ Health monitoring of your origin servers                    │
│  ✅ Failover across regions                                     │
│  ✅ DDoS protection at the edge                                │
│  ✅ Caching static content at edge locations                   │
│                                                                 │
│  Configuration (Cloudflare dashboard / API):                   │
│  Load Balancer:                                                 │
│    Pool "asia":                                                 │
│      Origins: [tokyo-server-1, tokyo-server-2]                 │
│      Health check: /health, interval 15s                       │
│    Pool "europe":                                               │
│      Origins: [london-server-1, london-server-2]               │
│    Pool "americas":                                             │
│      Origins: [virginia-server-1, virginia-server-2]           │
│    Steering policy: "geo" (or "dynamic_latency")              │
│    Fallback pool: "americas"                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### How Netflix Routes 250M+ Users Globally

```
Netflix Global Traffic Architecture:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  LAYER 1: DNS (AWS Route 53 + custom GSLB)                          │
│  ──────────────────────────────────────────                          │
│  User queries: netflix.com                                           │
│  Route 53 returns nearest AWS region IP based on:                    │
│    - User's geographic location                                      │
│    - Current region health                                           │
│    - Region capacity (avoids overloaded regions)                     │
│                                                                      │
│  LAYER 2: Edge / CDN (Open Connect)                                 │
│  ──────────────────────────────────────────                          │
│  Netflix operates 17,000+ servers in ISP data centers worldwide.    │
│  Video streaming comes from these edge servers — NOT from AWS.      │
│  GSLB routes video requests to the nearest Open Connect appliance.  │
│                                                                      │
│  LAYER 3: Regional (AWS regions)                                     │
│  ──────────────────────────────────────────                          │
│  API calls (search, browse, auth) go to regional AWS deployments.   │
│  3 active regions: us-east-1, us-west-2, eu-west-1                  │
│  If a region fails: traffic automatically shifts to other 2.        │
│                                                                      │
│  FAILOVER TESTED REGULARLY:                                          │
│  Netflix runs "Chaos Engineering" — they INTENTIONALLY              │
│  take down entire regions to verify GSLB failover works.            │
│                                                                      │
│  RESULT:                                                             │
│  - 99.99% availability (minutes of downtime per year)               │
│  - Video starts in < 2 seconds globally                             │
│  - API responses < 50ms from nearest region                         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| High DNS TTL (3600s) with failover | Users hit dead DC for up to 1 hour! | Use 60s TTL (or Anycast for instant failover) |
| GeoDNS without health checks | Route to regions that are DOWN | Always pair GeoDNS with health monitoring |
| Only using geography (not latency) | Sometimes a "nearby" DC has terrible network path | Use latency-based routing when possible |
| Not testing failover | Works in theory, breaks in practice | Regularly test by taking regions offline |
| Data consistency across regions | User writes in Tokyo, reads stale data in Singapore | Implement cross-region replication or route writes to primary |
| No "default" catch-all rule | Users from unmapped regions get DNS errors | Always configure a default/fallback region |
| Session state not replicated | User fails over to new region, loses session | Use globally replicated session store (DynamoDB Global Tables) |

---

## When to Use / When NOT to Use

### ✅ Use GSLB / GeoDNS When:

| Scenario | Why |
|----------|-----|
| Global user base (multiple continents) | Reduce latency for everyone |
| High availability requirement (99.99%+) | Survive entire region failures |
| Compliance (data residency) | EU data must stay in EU servers |
| Large-scale traffic (100K+ req/s) | Distribute load across regions |
| Competitive latency requirement | Gaming, trading, real-time apps |

### ❌ You Don't Need GSLB When:

| Scenario | Why Not |
|----------|---------|
| Users in one country/region | Single-region LB is sufficient |
| Small traffic (< 1000 req/s) | Cost/complexity not justified |
| Non-latency-sensitive app (email, batch) | 200ms vs 50ms doesn't matter |
| Early-stage startup | Premature optimization; scale regionally first |

### Progression Path:

```
Stage 1: Single server, single region
    ↓
Stage 2: Multiple servers, single region (regular LB)
    ↓
Stage 3: Multiple servers, two regions (active-passive GSLB)
    ↓
Stage 4: Multiple servers, 3+ regions (active-active GSLB)
    ↓
Stage 5: Edge computing + CDN + multi-region (Netflix/Google scale)
```

---

## Key Takeaways

1. **GSLB routes users to the nearest healthy data center** — combining GeoDNS, health monitoring, and failover to deliver global low-latency access.

2. **GeoDNS returns different IPs based on the client's geographic location** — the DNS server uses GeoIP databases to determine where the query originated.

3. **DNS TTL controls failover speed** — shorter TTL = faster failover but more DNS queries. 60 seconds is a good default.

4. **Anycast is the modern alternative to GeoDNS** — same IP announced from multiple locations, network routing handles the rest. Used by Cloudflare, Google, and most CDNs.

5. **GSLB requires data replication across regions** — routing users to different DCs means those DCs need synchronized data (databases, sessions, files).

6. **Always test failover** — Netflix takes down entire regions intentionally (Chaos Engineering) to verify their GSLB works under real conditions.

7. **Start simple** — most apps don't need GSLB until they have a truly global user base or require 99.99%+ availability. A single region with good CDN covers most cases.

---

## What's Next?

You now understand how to load balance at every level — from single servers to global data centers. In **Chapter 6.7: Load Balancing Tools**, we'll compare the actual software and services you'll use: Nginx, HAProxy, AWS ALB/NLB, and Envoy — their strengths, configurations, and when to choose each.
