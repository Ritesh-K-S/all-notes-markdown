# DNS Caching, TTL & Propagation

> **What you'll learn**: How DNS caching works at every level, what TTL really means, why DNS changes don't take effect immediately, how propagation works (and why the term is misleading), and how to manage TTL for different scenarios like migrations, failovers, and zero-downtime changes.

---

## Real-Life Analogy

Imagine your friend changes their phone number. You still have the OLD number saved in your phone. You'll keep calling the old number until:

1. You **notice it doesn't work** and look up the new one
2. Your phone's "auto-update contacts" feature kicks in after some time
3. Someone **tells you** the number changed

Now imagine this delay happening across **millions** of phone books simultaneously — your friend's new number needs to "propagate" to everyone who saved the old one.

DNS caching works the same way. When a domain's IP changes, every device and server that cached the old IP will keep using it until their cache expires. The **TTL (Time To Live)** is like telling everyone: "This information is valid for X seconds. After that, look it up again."

---

## What is DNS Caching?

DNS caching stores the results of DNS lookups so the same query doesn't need to travel the entire hierarchy again. Caching happens at EVERY level:

```
┌─────────────────────────────────────────────────────────────────────────┐
│              DNS Caching Layers (All Have Their Own Cache)                │
│                                                                         │
│  Layer 1: Browser Cache                                                  │
│  ├── Chrome, Firefox, Safari each maintain their own                    │
│  ├── Typically 100-1000 entries                                          │
│  └── Cleared on browser restart                                          │
│           │                                                             │
│  Layer 2: Operating System Cache                                         │
│  ├── Windows: DNS Client Service                                        │
│  ├── Linux: systemd-resolved, nscd                                      │
│  ├── macOS: mDNSResponder                                               │
│  └── Persists across browser restarts                                    │
│           │                                                             │
│  Layer 3: Router/Gateway Cache                                           │
│  ├── Home routers (dnsmasq)                                             │
│  ├── Corporate DNS appliances                                            │
│  └── Serves all devices on the network                                   │
│           │                                                             │
│  Layer 4: ISP/Recursive Resolver Cache                                   │
│  ├── Google (8.8.8.8), Cloudflare (1.1.1.1)                            │
│  ├── ISP resolvers (Jio, Airtel, Comcast)                               │
│  ├── MASSIVE cache — millions of entries                                 │
│  └── Serves thousands/millions of users                                  │
│           │                                                             │
│  Layer 5: TLD/Root Server Responses (cached by resolver)                 │
│  ├── ".com NS records" cached for 172800 seconds (2 days!)              │
│  └── Root hints cached for 518400 seconds (6 days!)                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What is TTL (Time To Live)?

**TTL is a number (in seconds) that tells every cache: "You can keep this answer for this many seconds. After that, you MUST ask again."**

```
┌─────────────────────────────────────────────────────────────────┐
│              TTL Countdown Example                                │
│                                                                 │
│  DNS Response: example.com → 93.184.216.34, TTL=300             │
│                                                                 │
│  t=0s:   Cache stores answer. TTL remaining = 300               │
│  t=60s:  TTL remaining = 240. Still serving cached answer.      │
│  t=150s: TTL remaining = 150. Still serving cached answer.      │
│  t=300s: TTL remaining = 0. CACHE EXPIRED!                      │
│  t=301s: Next query → must ask upstream DNS server again.       │
│                                                                 │
│  ┌────────────────────────────────────────────┐                 │
│  │ 0s         150s         300s    301s       │                 │
│  │ |───────────|───────────|       |          │                 │
│  │ ◄── Serving from cache ──►  ◄── Re-query ─►│                 │
│  │     (fast, no network)       (network hop)  │                 │
│  └────────────────────────────────────────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Common TTL Values and Their Use Cases

| TTL Value | Seconds | Use Case |
|-----------|---------|----------|
| 60 | 1 minute | Failover-critical services, during migrations |
| 300 | 5 minutes | Standard web applications (good balance) |
| 900 | 15 minutes | Semi-static services |
| 3600 | 1 hour | Email records (MX), relatively stable |
| 86400 | 1 day | NS records, very stable infrastructure |
| 172800 | 2 days | TLD delegation records |

---

## How TTL Works at Each Cache Level

```
┌─────────────────────────────────────────────────────────────────────┐
│         TTL Decrement Through Cache Layers                           │
│                                                                     │
│  Authoritative Server responds: TTL = 300                           │
│       │                                                             │
│       ▼                                                             │
│  Recursive Resolver receives at t=0                                 │
│  Caches with TTL=300. Starts countdown.                             │
│       │                                                             │
│       │  (20 seconds pass while processing)                         │
│       ▼                                                             │
│  Responds to client at t=20                                         │
│  Sends TTL=280 (300 - 20 = remaining time)                         │
│       │                                                             │
│       │  (5 seconds network transit)                                │
│       ▼                                                             │
│  OS receives at t=25                                                │
│  Caches with TTL=275                                                │
│       │                                                             │
│       ▼                                                             │
│  Browser receives at t=25                                           │
│  Caches with TTL=275                                                │
│                                                                     │
│  KEY INSIGHT: Each layer gets a DECREMENTED TTL                     │
│  (not the original). The clock is already ticking!                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### What Happens When You Change a DNS Record

```
┌─────────────────────────────────────────────────────────────────────┐
│         DNS Change Propagation Timeline (TTL=300)                    │
│                                                                     │
│  t=0:    You change A record from 10.0.0.1 → 10.0.0.2              │
│          Authoritative server immediately returns new IP            │
│                                                                     │
│  t=0-300s: CHAOS PERIOD                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Users with fresh cache → get OLD IP (10.0.0.1)            │    │
│  │  Users making new queries → get NEW IP (10.0.0.2)          │    │
│  │  Both IPs are being used simultaneously!                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  t=300s: All caches with TTL=300 have expired                       │
│          NEW queries now get 10.0.0.2                               │
│                                                                     │
│  BUT! Some resolvers may not respect TTL:                           │
│  - Some ISPs set MINIMUM TTL of 300-3600 regardless of yours        │
│  - Some corporate resolvers override TTL                            │
│  - Some devices have buggy DNS implementations                      │
│                                                                     │
│  WORST CASE: Up to 48-72 hours for complete global propagation      │
│  (because some resolvers clamp TTL to very high values)             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## "DNS Propagation" — Why the Term is Misleading

People often say "DNS is propagating" — but this is technically wrong. DNS doesn't **push** changes to anyone. What actually happens:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  MYTH: "DNS propagates changes from server to server"           │
│         (like a wave spreading outward)                          │
│                                                                 │
│  REALITY: "Caches expire independently based on their TTL"      │
│           (each cache discovers the change on its own)           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │  Authoritative: Changed at t=0                          │    │
│  │                                                         │    │
│  │  Resolver A (cached 290s ago): Expires in 10s  ✓ FAST   │    │
│  │  Resolver B (cached 5s ago):   Expires in 295s ✗ SLOW   │    │
│  │  Resolver C (cached 150s ago): Expires in 150s ~ MID    │    │
│  │                                                         │    │
│  │  Each resolver will see the change at a DIFFERENT time   │    │
│  │  depending on when they last cached the old answer.      │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## TTL Strategy for Different Scenarios

### Scenario 1: Normal Operations

```
Situation: Stable service, rarely changes IP
Recommended TTL: 300-3600 seconds (5 min to 1 hour)
Reason: Reduces DNS query volume, lowers costs, faster for users
```

### Scenario 2: Preparing for a Migration

```
┌─────────────────────────────────────────────────────────────────┐
│         Pre-Migration TTL Lowering Strategy                       │
│                                                                 │
│  STEP 1: 48 hours BEFORE migration                              │
│  Lower TTL from 3600 → 60 seconds                               │
│  (Wait for old high-TTL caches to expire)                       │
│                                                                 │
│  STEP 2: Perform the migration                                  │
│  Change A record from old_ip → new_ip                           │
│  (Now worst-case propagation = ~60 seconds)                     │
│                                                                 │
│  STEP 3: After migration is stable (24 hours later)             │
│  Raise TTL back from 60 → 3600 seconds                          │
│                                                                 │
│  Timeline:                                                      │
│  ──────────────────────────────────────────────────────────     │
│  Day -2       Day 0         Day +1        Day +2                │
│  TTL=60     MIGRATE!      Verify OK      TTL=3600              │
│  (prepare)  (switch IP)   (monitoring)   (stabilize)           │
│  ──────────────────────────────────────────────────────────     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Scenario 3: Failover/HA Setup

```
Situation: Need fast failover when primary server dies
Recommended TTL: 30-60 seconds
Trade-off: More DNS queries (higher cost) but faster failover
```

### Scenario 4: CDN/Static Content

```
Situation: CNAME to CDN (CloudFront, Cloudflare)
Recommended TTL: 300-3600 seconds
Reason: CDN handles failover internally, DNS just points to CDN edge
```

---

## How It Works Internally

### Negative Caching (NXDOMAIN TTL)

When a domain does NOT exist, the "non-existence" is ALSO cached:

```
┌─────────────────────────────────────────────────────────────────┐
│         Negative Caching (RFC 2308)                              │
│                                                                 │
│  Query: "What's the IP of typo.example.com?"                    │
│  Response: NXDOMAIN (doesn't exist)                             │
│                                                                 │
│  The SOA record's MINIMUM field defines negative cache TTL:     │
│  example.com. SOA ns1.example.com. admin.example.com. (         │
│      2024010101  ; serial                                       │
│      3600        ; refresh                                      │
│      900         ; retry                                        │
│      604800      ; expire                                       │
│      86400       ; ← THIS: Negative caching TTL (1 day!)       │
│  )                                                              │
│                                                                 │
│  Problem: If you create typo.example.com AFTER someone queries  │
│  it, they won't see it for up to 86400 seconds (1 day)!        │
│                                                                 │
│  Fix: Set SOA minimum to 300-3600 for new domains               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### How Recursive Resolvers Handle TTL

```
┌─────────────────────────────────────────────────────────────────┐
│         Resolver TTL Policies                                    │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Google Public DNS (8.8.8.8)                            │     │
│  │ - Respects TTL as-is                                   │     │
│  │ - Minimum TTL: None (can be 0)                         │     │
│  │ - Maximum TTL: 86400 (1 day)                           │     │
│  │ - Supports prefetching near-expired records            │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Cloudflare (1.1.1.1)                                   │     │
│  │ - Minimum TTL: 1 second (very compliant)               │     │
│  │ - Maximum TTL: 86400                                    │     │
│  │ - Prefetches 10% before TTL expiry                     │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Some ISP Resolvers (problematic!)                      │     │
│  │ - May enforce MINIMUM TTL of 300-3600                  │     │
│  │ - May ignore low TTLs entirely                          │     │
│  │ - May cache for longer than specified                   │     │
│  │ - This is why "propagation takes 24-48 hours"          │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Prefetching (Proactive Cache Refresh)

Modern resolvers don't wait for TTL to expire — they refresh BEFORE it expires:

```
┌─────────────────────────────────────────────────────────────────┐
│         Prefetching Strategy                                     │
│                                                                 │
│  Record: example.com A 93.184.216.34, TTL=300                   │
│                                                                 │
│  Without prefetching:                                           │
│  t=300s: Cache expires → Next query is SLOW (cold lookup)       │
│                                                                 │
│  With prefetching:                                              │
│  t=270s (90% of TTL): Resolver proactively re-queries           │
│  t=300s: New answer already cached → Next query is FAST!        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  |════════════════════════|═══|════════════════════| │        │
│  │  0                     270  300                  600 │        │
│  │  ◄── Serving from cache ─►│◄─►│◄── New cached ────► │        │
│  │                       Prefetch                       │        │
│  │                       happens                        │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  Result: Users NEVER experience cold DNS lookups                │
│  for popular domains.                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### DNS Cache Poisoning (Security Concern)

```
┌─────────────────────────────────────────────────────────────────┐
│         Cache Poisoning Attack                                    │
│                                                                 │
│  NORMAL:                                                        │
│  Resolver asks: "What's bank.com?"                              │
│  Authoritative answers: "192.168.1.100" (legitimate)            │
│                                                                 │
│  POISONING ATTACK:                                              │
│  Attacker sends FAKE response before real one arrives:          │
│  Fake answer: "bank.com → 10.66.6.6" (attacker's server!)      │
│                                                                 │
│  Resolver caches the FAKE answer!                               │
│  All users get sent to attacker's server for TTL duration!      │
│                                                                 │
│  DEFENSES:                                                      │
│  1. Source port randomization (16-bit entropy)                  │
│  2. Transaction ID randomization (16-bit)                       │
│  3. 0x20 encoding (mixed case in query = more entropy)          │
│  4. DNSSEC (cryptographic signatures on records)                │
│  5. DNS-over-HTTPS / DNS-over-TLS (encrypted transport)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Check TTL of Any Domain

```python
import dns.resolver
import time

def check_ttl(domain, record_type='A'):
    """
    Query a domain and show the TTL at different levels.
    Run this multiple times to see TTL counting down.
    """
    resolver = dns.resolver.Resolver()
    resolver.nameservers = ['8.8.8.8']  # Use Google DNS
    
    try:
        answer = resolver.resolve(domain, record_type)
        
        print(f"\n  Domain: {domain} ({record_type} record)")
        print(f"  {'─' * 50}")
        
        for rdata in answer:
            print(f"  Value: {rdata}")
        
        print(f"  TTL: {answer.rrset.ttl} seconds remaining")
        print(f"  (Original TTL may have been higher — this is remaining)")
        
        # Query authoritative server for original TTL
        ns_answer = resolver.resolve(domain, 'NS')
        ns_name = str(ns_answer[0])
        
        try:
            ns_ip = resolver.resolve(ns_name, 'A')
            auth_resolver = dns.resolver.Resolver()
            auth_resolver.nameservers = [str(ns_ip[0])]
            auth_answer = auth_resolver.resolve(domain, record_type)
            print(f"  Original TTL (from authoritative): {auth_answer.rrset.ttl}s")
        except:
            pass
            
    except dns.resolver.NXDOMAIN:
        print(f"  {domain} does not exist (NXDOMAIN)")
    except Exception as e:
        print(f"  Error: {e}")

def watch_ttl_countdown(domain, interval=10, count=5):
    """Watch TTL counting down over time (educational demo)."""
    print(f"\n  Watching TTL countdown for {domain}:")
    print(f"  (Querying every {interval} seconds, {count} times)")
    print()
    
    for i in range(count):
        resolver = dns.resolver.Resolver()
        resolver.nameservers = ['8.8.8.8']
        answer = resolver.resolve(domain, 'A')
        ttl = answer.rrset.ttl
        
        bar = '█' * (ttl // 10)
        print(f"  t={i*interval:3d}s: TTL={ttl:4d}s {bar}")
        
        if i < count - 1:
            time.sleep(interval)

# Usage
check_ttl("google.com")
check_ttl("example.com")
check_ttl("google.com", "MX")

# Uncomment to watch countdown:
# watch_ttl_countdown("example.com", interval=30, count=10)
```

### Python — Pre-Migration TTL Lowering Script

```python
import requests
import time

# Cloudflare API example for managing TTL during migration
CLOUDFLARE_API = "https://api.cloudflare.com/client/v4"
ZONE_ID = "your_zone_id"
HEADERS = {
    "Authorization": "Bearer YOUR_API_TOKEN",
    "Content-Type": "application/json"
}

def get_record_id(domain, record_type='A'):
    """Find the DNS record ID for a domain."""
    response = requests.get(
        f"{CLOUDFLARE_API}/zones/{ZONE_ID}/dns_records",
        headers=HEADERS,
        params={"name": domain, "type": record_type}
    )
    records = response.json()["result"]
    return records[0]["id"] if records else None

def update_ttl(domain, new_ttl, record_type='A'):
    """Update TTL for a DNS record (pre-migration step)."""
    record_id = get_record_id(domain, record_type)
    if not record_id:
        print(f"  Record not found: {domain}")
        return
    
    # Get current record
    response = requests.get(
        f"{CLOUDFLARE_API}/zones/{ZONE_ID}/dns_records/{record_id}",
        headers=HEADERS
    )
    record = response.json()["result"]
    
    # Update just the TTL
    response = requests.put(
        f"{CLOUDFLARE_API}/zones/{ZONE_ID}/dns_records/{record_id}",
        headers=HEADERS,
        json={
            "type": record["type"],
            "name": record["name"],
            "content": record["content"],
            "ttl": new_ttl
        }
    )
    
    if response.json()["success"]:
        print(f"  ✓ TTL updated: {domain} → {new_ttl}s")
    else:
        print(f"  ✗ Error: {response.json()['errors']}")

def migration_workflow(domain, new_ip):
    """
    Safe DNS migration workflow:
    1. Lower TTL (wait for old caches to expire)
    2. Switch IP
    3. Verify
    4. Raise TTL back
    """
    print(f"\n  === DNS Migration Workflow for {domain} ===")
    
    # Step 1: Lower TTL
    print("\n  [Step 1] Lowering TTL to 60 seconds...")
    update_ttl(domain, 60)
    print("  ⏳ Waiting for old caches to expire (need to wait old_ttl seconds)")
    print("  In production: wait 2x the old TTL before proceeding")
    
    # Step 2: Change IP (in production, wait old_ttl time here)
    print(f"\n  [Step 2] Changing IP to {new_ip}...")
    record_id = get_record_id(domain)
    # ... update record content to new_ip
    
    # Step 3: Verify
    print("\n  [Step 3] Verifying new record resolves correctly...")
    # ... verify DNS resolution
    
    # Step 4: Raise TTL back
    print("\n  [Step 4] Raising TTL back to 300 seconds...")
    update_ttl(domain, 300)
    
    print("\n  ✓ Migration complete!")

# Usage
# migration_workflow("api.example.com", "10.0.0.2")
```

### Java — DNS Cache Management

```java
import java.net.InetAddress;
import java.security.Security;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class DnsCacheManager {
    
    public static void main(String[] args) throws Exception {
        // Java has its OWN DNS cache (separate from OS!)
        // By default, successful lookups cached FOREVER in JVM!
        // This is often a problem in cloud environments.
        
        demonstrateJavaDnsCache();
    }
    
    static void demonstrateJavaDnsCache() throws Exception {
        // Check current JVM DNS cache settings
        String positiveTtl = Security.getProperty("networkaddress.cache.ttl");
        String negativeTtl = Security.getProperty("networkaddress.cache.negative.ttl");
        
        System.out.println("JVM DNS Cache Settings:");
        System.out.println("  Positive TTL: " + 
            (positiveTtl == null ? "FOREVER (default!)" : positiveTtl + "s"));
        System.out.println("  Negative TTL: " + 
            (negativeTtl == null ? "10s (default)" : negativeTtl + "s"));
        
        // Fix: Set reasonable TTL for cloud/container environments
        // Must be set BEFORE any DNS lookups happen
        Security.setProperty("networkaddress.cache.ttl", "60");
        Security.setProperty("networkaddress.cache.negative.ttl", "10");
        
        System.out.println("\nAfter fix:");
        System.out.println("  Positive TTL: 60s");
        System.out.println("  Negative TTL: 10s");
        
        // Alternative: Set via JVM flags
        // -Dsun.net.inetaddr.ttl=60
        // -Dsun.net.inetaddr.negative.ttl=10
        
        // Demonstrate resolution with timing
        long start = System.nanoTime();
        InetAddress addr1 = InetAddress.getByName("www.google.com");
        long first = (System.nanoTime() - start) / 1_000_000;
        
        start = System.nanoTime();
        InetAddress addr2 = InetAddress.getByName("www.google.com");
        long second = (System.nanoTime() - start) / 1_000_000;
        
        System.out.println("\nResolution timing:");
        System.out.println("  First:  " + first + "ms (network lookup)");
        System.out.println("  Second: " + second + "ms (JVM cache hit)");
    }
}
```

### Java — Spring Boot DNS TTL Configuration

```java
import org.springframework.context.annotation.Configuration;
import javax.annotation.PostConstruct;
import java.security.Security;

@Configuration
public class DnsConfiguration {
    
    /**
     * CRITICAL for cloud-native applications!
     * 
     * By default, JVM caches DNS forever. This means:
     * - Load balancer IP changes? JVM uses old IP.
     * - Service scales down? JVM sends to dead instance.
     * - Database failover? JVM can't find new primary.
     * 
     * Fix: Set TTL to 60 seconds for cloud environments.
     */
    @PostConstruct
    public void configureDnsTtl() {
        // Cache successful lookups for 60 seconds
        Security.setProperty("networkaddress.cache.ttl", "60");
        
        // Cache failed lookups for 10 seconds
        Security.setProperty("networkaddress.cache.negative.ttl", "10");
    }
}
```

---

## Infrastructure Examples

### Checking DNS Propagation Globally

```bash
# Query different resolvers to check propagation status
echo "=== Checking DNS propagation for example.com ==="

# Google DNS
dig @8.8.8.8 example.com A +short
echo "Google DNS (8.8.8.8): $(dig @8.8.8.8 example.com A +short)"

# Cloudflare DNS
echo "Cloudflare (1.1.1.1): $(dig @1.1.1.1 example.com A +short)"

# OpenDNS
echo "OpenDNS (208.67.222.222): $(dig @208.67.222.222 example.com A +short)"

# Quad9
echo "Quad9 (9.9.9.9): $(dig @9.9.9.9 example.com A +short)"

# Local ISP
echo "Local ISP: $(dig example.com A +short)"

# Check remaining TTL
echo "\nTTL remaining:"
dig @8.8.8.8 example.com A | grep -A1 "ANSWER SECTION"
```

### Flush DNS Cache (All Platforms)

```bash
# === Linux ===
# systemd-resolved
sudo systemd-resolve --flush-caches
sudo resolvectl flush-caches

# nscd
sudo systemctl restart nscd

# dnsmasq
sudo systemctl restart dnsmasq

# === macOS ===
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# === Windows (PowerShell) ===
# ipconfig /flushdns
# Clear-DnsClientCache
```

### Nginx DNS Caching Behavior

```nginx
# PROBLEM: Nginx caches DNS at startup and NEVER re-resolves!
# This is dangerous when upstream IPs change (e.g., AWS ALB)

# BAD - DNS cached forever:
upstream backend {
    server api.internal.example.com;  # Resolved once at startup!
}

# GOOD - Force periodic re-resolution:
resolver 10.0.0.2 valid=30s;  # Use internal DNS, re-resolve every 30s

server {
    location /api/ {
        set $backend "api.internal.example.com";
        proxy_pass http://$backend;  # Variable forces re-resolution
    }
}
```

### Kubernetes DNS (CoreDNS) TTL Configuration

```yaml
# CoreDNS ConfigMap - controls DNS TTL for cluster services
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        
        # Kubernetes service discovery
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30    # TTL for service records (default: 5)
        }
        
        # Cache settings
        cache 30 {
            success 2048 300  # Cache hits: max 2048 entries, 300s max TTL
            denial 256 60     # Cache misses: max 256 entries, 60s
            prefetch 10 60s 25%  # Prefetch if >10 queries in 60s
        }
        
        # Forward external queries
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        
        prometheus :9153
        loop
        reload
        loadbalance
    }
```

---

## Real-World Example

### Cloudflare's 1.1.1.1 — How They Achieve Sub-5ms Response Times

```
┌─────────────────────────────────────────────────────────────────────┐
│            Cloudflare's DNS Caching Architecture                      │
│                                                                     │
│  ┌─────────────────────────────────────────────────┐                │
│  │  Every Cloudflare PoP (300+ worldwide)          │                │
│  │                                                 │                │
│  │  ┌───────────────────────────────────────────┐  │                │
│  │  │  Hot Cache (in-memory, per-server)        │  │                │
│  │  │  - Millions of entries                     │  │                │
│  │  │  - Respects TTL from authoritative        │  │                │
│  │  │  - Hit rate: ~80% of queries              │  │                │
│  │  └───────────────────────────────────────────┘  │                │
│  │                                                 │                │
│  │  ┌───────────────────────────────────────────┐  │                │
│  │  │  Shared Cache (distributed within PoP)    │  │                │
│  │  │  - Shared across all servers in the PoP   │  │                │
│  │  │  - Hit rate: ~15% of queries              │  │                │
│  │  └───────────────────────────────────────────┘  │                │
│  │                                                 │                │
│  │  Cache Miss (~5%): Full recursive resolution    │                │
│  │                                                 │                │
│  └─────────────────────────────────────────────────┘                │
│                                                                     │
│  Optimizations:                                                     │
│  1. Prefetching: Re-resolve popular records before TTL expires      │
│  2. Stale-while-revalidate: Serve stale data while refreshing      │
│  3. Query minimization (RFC 7816): Send minimal info to upstreams  │
│  4. Aggressive NSEC caching: Cache negative answers efficiently     │
│                                                                     │
│  Result: Average response time < 11ms globally                      │
│  (With cache hit: < 1ms!)                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### The Famous "48-Hour Propagation" Myth

When domain registrars say "allow 24-48 hours for DNS propagation," here's what they really mean:

```
Why it CAN take 48 hours:
┌──────────────────────────────────────────────────────────────────┐
│  1. NS record TTL at TLD (.com) level: 172800 seconds (2 days!) │
│     If you CHANGE name servers, TLD caches old NS for 2 days.   │
│                                                                  │
│  2. Some ISP resolvers enforce minimum TTL of hours/days         │
│     They ignore your low TTL and cache longer anyway.            │
│                                                                  │
│  3. Some corporate resolvers cache aggressively                  │
│     Enterprise proxies may override TTL policies.                │
│                                                                  │
│  4. Client-side caching (OS, browser, apps)                      │
│     Users may need to restart browser/flush cache.               │
└──────────────────────────────────────────────────────────────────┘

Why it's USUALLY much faster:
┌──────────────────────────────────────────────────────────────────┐
│  If you're just changing an A record (not NS records):           │
│  - With TTL=300: Full propagation in ~5-10 minutes              │
│  - With TTL=60: Full propagation in ~2-5 minutes                │
│  - Major resolvers (Google, Cloudflare): Nearly instant         │
│                                                                  │
│  The "48 hours" only applies to:                                │
│  - Changing nameservers (NS records)                             │
│  - New domain registrations                                      │
│  - Transferring domains between registrars                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| TTL=0 thinking it means "no cache" | Some resolvers treat 0 as "use minimum" | Use TTL=1 minimum |
| Not lowering TTL BEFORE migration | Old high-TTL caches persist during switch | Lower TTL 2× old TTL before changes |
| Java's default DNS cache = forever | Stale IPs served after failover | Set `networkaddress.cache.ttl=60` |
| Nginx not re-resolving DNS | Sends traffic to dead upstream | Use `resolver` directive + variable in proxy_pass |
| Ignoring negative caching | New subdomain invisible for hours | Set SOA minimum to reasonable value (300-3600) |
| Expecting instant propagation | Users report site down after DNS change | Plan for TTL duration of mixed traffic |
| Not having both old and new servers running | Some users hit old IP, which is dead | Keep old server running for at least 2× TTL |
| Testing only with `dig`/`nslookup` | These bypass OS/browser cache | Also test in browser, check from different networks |

---

## When to Use / When NOT to Use

### Use Low TTL (30-60s) When:
- ✅ Active failover configuration
- ✅ During planned DNS migrations
- ✅ Load balancing with health checks
- ✅ Blue/green deployments via DNS
- ✅ Dynamic/frequently changing infrastructure

### Use High TTL (3600-86400s) When:
- ✅ Stable infrastructure that rarely changes
- ✅ Cost optimization (reduce DNS query volume)
- ✅ Reducing DNS lookup latency for users
- ✅ NS records (delegation rarely changes)
- ✅ MX records (mail servers are stable)

### Never Use:
- ❌ TTL=0 for production (unreliable behavior across resolvers)
- ❌ TTL > 86400 for records that might need changing
- ❌ High TTL during active infrastructure changes
- ❌ Different TTLs for the same record across multiple NS (inconsistent)

---

## Key Takeaways

- **TTL is the single most important DNS setting** — it controls how fast changes take effect and how much DNS load your infrastructure generates.
- **DNS doesn't "propagate"** — caches independently expire based on TTL. Different users see changes at different times.
- **Lower TTL before making changes** — the golden rule of DNS migrations. Wait 2× the old TTL before switching.
- **Java's JVM caches DNS forever by default** — this is the #1 DNS gotcha in cloud-native Java apps. Always set `networkaddress.cache.ttl`.
- **Nginx doesn't re-resolve DNS** unless you use the `resolver` directive with variables — critical for dynamic upstreams.
- **Negative caching exists** — NXDOMAIN responses are cached too, controlled by the SOA record's minimum field.
- **"48-hour propagation" is mostly myth** — A record changes with low TTL propagate in minutes; only NS changes truly take up to 48 hours.

---

## What's Next?

Now that you understand caching and TTL, let's look at managed DNS services that handle all this complexity for you at scale — with built-in health checks, failover, and global distribution. See [05-managed-dns.md](./05-managed-dns.md).
