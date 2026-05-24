# DNS Resolution Flow — Step by Step

> **What you'll learn**: The exact journey a DNS query takes from the moment you type a URL until your browser gets an IP address — every hop, every cache check, every server involved, explained in painstaking detail.

---

## Real-Life Analogy

Imagine you want to find the phone number of a specific person named "John Smith who works at Acme Corp in New York."

Here's how you'd search:

1. **Check your own phone contacts** (your memory) — fastest!
2. **Ask the person sitting next to you** — they might know
3. **Call the country's main directory** — "Who handles New York businesses?"
4. **Call New York's business directory** — "Who handles Acme Corp?"
5. **Call Acme Corp's reception** — "What's John Smith's number?"
6. **Write it down** in your contacts so you don't have to do this again!

DNS resolution works EXACTLY this way — it's a hierarchical lookup that goes from the most local cache to progressively higher authorities, until someone gives you the definitive answer.

---

## The Big Picture — DNS Resolution Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DNS Resolution: Full Journey                           │
│                                                                         │
│  You type: www.example.com                                              │
│                                                                         │
│  ① Browser Cache ─── Hit? → Done! (< 0ms)                              │
│         │ Miss                                                          │
│         ▼                                                               │
│  ② OS Cache (Stub Resolver) ─── Hit? → Done! (< 1ms)                   │
│         │ Miss                                                          │
│         ▼                                                               │
│  ③ Router Cache ─── Hit? → Done! (< 2ms)                               │
│         │ Miss                                                          │
│         ▼                                                               │
│  ④ ISP Recursive Resolver ─── Hit? → Done! (5-20ms)                    │
│         │ Miss                                                          │
│         ▼                                                               │
│  ⑤ Root Name Server ─── "Ask .com TLD" (50-100ms)                      │
│         │                                                               │
│         ▼                                                               │
│  ⑥ TLD Name Server (.com) ─── "Ask ns1.example.com" (50-100ms)         │
│         │                                                               │
│         ▼                                                               │
│  ⑦ Authoritative Name Server ─── "IP is 93.184.216.34" (50-100ms)      │
│         │                                                               │
│         ▼                                                               │
│  Answer bubbles back up, cached at every level                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Deep Dive

### Step 1: Browser DNS Cache

The VERY first thing that happens when you type `www.example.com`:

```
┌────────────────────────────────────────────────┐
│  Browser Internal Cache                         │
│                                                │
│  www.example.com → 93.184.216.34  (TTL: 245s) │
│  google.com      → 142.250.190.78 (TTL: 120s) │
│  api.github.com  → 140.82.112.5   (TTL: 30s)  │
│                                                │
│  Cache Hit? → Use this IP immediately          │
│  Cache Miss? → Go to Step 2                    │
└────────────────────────────────────────────────┘
```

**Details:**
- Chrome stores DNS cache in memory (view it at `chrome://net-internals/#dns`)
- Firefox, Safari, Edge all have their own caches
- Typical cache sizes: 100-1000 entries
- Respects TTL from DNS response
- Cleared when you close the browser or flush manually

**View Chrome DNS cache:**
```
chrome://net-internals/#dns
```

---

### Step 2: Operating System Stub Resolver

If the browser cache misses, it calls the OS-level DNS resolver (the "stub resolver").

```
┌─────────────────────────────────────────────────────────────┐
│  Operating System DNS Stack                                   │
│                                                             │
│  Application (Browser)                                      │
│       │                                                     │
│       │ getaddrinfo("www.example.com")                      │
│       ▼                                                     │
│  ┌─────────────────────────┐                                │
│  │   /etc/hosts check      │  ← Checked FIRST!             │
│  │   127.0.0.1  localhost   │                                │
│  │   203.0.113.5 myapp.dev  │                                │
│  └─────────────────────────┘                                │
│       │ Not found in hosts                                  │
│       ▼                                                     │
│  ┌─────────────────────────┐                                │
│  │   OS DNS Cache           │  ← nscd/systemd-resolved/     │
│  │   (nscd on Linux,        │     Windows DNS Client Service │
│  │    dnsmasq, systemd)     │                                │
│  └─────────────────────────┘                                │
│       │ Cache miss                                          │
│       ▼                                                     │
│  Send query to configured DNS resolver                      │
│  (from /etc/resolv.conf or DHCP)                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Key points:**
- On Linux: `getaddrinfo()` syscall → checks `/etc/hosts` → checks NSS → queries resolver in `/etc/resolv.conf`
- On Windows: `GetAddrInfoW()` → checks `C:\Windows\System32\drivers\etc\hosts` → checks DNS Client cache → queries configured DNS
- On macOS: `getaddrinfo()` → checks `/etc/hosts` → queries `mDNSResponder`

**Check OS cache:**
```bash
# Linux (systemd-resolved)
resolvectl statistics

# Windows
ipconfig /displaydns

# macOS
sudo dscacheutil -statistics
```

---

### Step 3: Router/Gateway Cache

Your home router or corporate gateway often runs a small DNS cache:

```
┌──────────────────────────────────────────────┐
│  Home Router (e.g., 192.168.1.1)             │
│                                              │
│  Built-in DNS forwarder (dnsmasq typically)  │
│                                              │
│  Cache: 100-1000 entries                     │
│  Serves all devices on the network           │
│                                              │
│  Cache Hit? → Return to OS immediately       │
│  Cache Miss? → Forward to upstream resolver  │
└──────────────────────────────────────────────┘
```

This step is often invisible but adds a caching layer for all devices on your network.

---

### Step 4: Recursive Resolver (The Heavy Lifter)

This is where the REAL work happens. The recursive resolver (usually your ISP's, or `8.8.8.8`, `1.1.1.1`) does the full lookup on your behalf.

```
┌────────────────────────────────────────────────────────────────────┐
│            Recursive Resolver (e.g., 8.8.8.8)                       │
│                                                                    │
│  ┌──────────────────────────────────────────┐                      │
│  │  HUGE CACHE (millions of entries)        │                      │
│  │                                          │                      │
│  │  www.example.com → 93.184.216.34 (180s)  │                      │
│  │  .com NS → a.gtld-servers.net (172800s)  │                      │
│  │  example.com NS → ns1.example.com (3600) │                      │
│  │                                          │                      │
│  │  Cache Hit? → Return immediately         │                      │
│  │  Cache Miss? → Start iterative lookup    │                      │
│  └──────────────────────────────────────────┘                      │
│                                                                    │
│  If cache miss, the resolver does Steps 5-7 on your behalf         │
│  (This is why it's called "recursive" — it recurses for you)       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

**Recursive vs Iterative:**
```
RECURSIVE (what your computer does):
  "Please give me the final answer. I'll wait."

ITERATIVE (what the recursive resolver does internally):
  "Give me whoever I should ask next. I'll keep asking until I get the answer."
```

---

### Step 5: Root Name Servers

If the recursive resolver has no cached answer, it starts at the TOP of the DNS hierarchy.

```
┌─────────────────────────────────────────────────────────────────┐
│              Root Name Servers (13 clusters)                       │
│                                                                 │
│  a.root-servers.net  (Verisign)        198.41.0.4              │
│  b.root-servers.net  (USC-ISI)         199.9.14.201            │
│  c.root-servers.net  (Cogent)          192.33.4.12             │
│  d.root-servers.net  (U of Maryland)   199.7.91.13             │
│  e.root-servers.net  (NASA)            192.203.230.10          │
│  f.root-servers.net  (ISC)             192.5.5.241             │
│  g.root-servers.net  (US DOD)          192.112.36.4            │
│  h.root-servers.net  (US Army)         198.97.190.53           │
│  i.root-servers.net  (Netnod)          192.36.148.17           │
│  j.root-servers.net  (Verisign)        192.58.128.30           │
│  k.root-servers.net  (RIPE NCC)        193.0.14.129            │
│  l.root-servers.net  (ICANN)           199.7.83.42             │
│  m.root-servers.net  (WIDE)            202.12.27.33            │
│                                                                 │
│  Total: 13 logical servers, 1500+ physical instances worldwide  │
│  (Using Anycast — same IP, many locations)                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The conversation:**
```
Resolver → Root: "Where is www.example.com?"
Root → Resolver: "I don't know, but .com is handled by these TLD servers:
                   a.gtld-servers.net (192.5.6.30)
                   b.gtld-servers.net (192.33.14.30)
                   ... (13 servers)"
```

**Important:** Root servers DON'T know the answer — they only know which TLD servers handle `.com`, `.org`, `.net`, etc.

---

### Step 6: TLD (Top-Level Domain) Name Servers

The resolver now asks the `.com` TLD servers:

```
┌─────────────────────────────────────────────────────────────────┐
│              .com TLD Servers (managed by Verisign)               │
│                                                                 │
│  a.gtld-servers.net  through  m.gtld-servers.net                │
│                                                                 │
│  These know EVERY .com domain's name servers                    │
│  (Over 150 million .com registrations!)                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The conversation:**
```
Resolver → .com TLD: "Where is www.example.com?"
.com TLD → Resolver: "I don't know the IP, but example.com is managed by:
                       ns1.example.com (198.51.100.1)
                       ns2.example.com (198.51.100.2)"
```

**The TLD server returns NS records + "glue records"** (A records for the name servers, so you don't get stuck in a circular dependency).

---

### Step 7: Authoritative Name Server

Finally, the resolver asks the authoritative server that OWNS `example.com`:

```
┌─────────────────────────────────────────────────────────────────┐
│         Authoritative Name Server (ns1.example.com)              │
│                                                                 │
│  This server has the ACTUAL zone file for example.com           │
│  It is the source of truth.                                     │
│                                                                 │
│  Zone file contains:                                            │
│    www.example.com.  300  IN  A  93.184.216.34                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The conversation:**
```
Resolver → ns1.example.com: "What's the A record for www.example.com?"
ns1.example.com → Resolver: "93.184.216.34, TTL=300 seconds"
```

---

### Step 8: The Answer Travels Back

```
┌─────────────────────────────────────────────────────────────────────┐
│              Answer Propagation (Bottom → Top)                        │
│                                                                     │
│  Authoritative (93.184.216.34, TTL=300)                             │
│       │                                                             │
│       ▼                                                             │
│  Recursive Resolver: Caches for 300 seconds                         │
│       │                                                             │
│       ▼                                                             │
│  Router: Caches for 300 seconds                                     │
│       │                                                             │
│       ▼                                                             │
│  OS: Caches for 300 seconds                                         │
│       │                                                             │
│       ▼                                                             │
│  Browser: Caches for 300 seconds                                    │
│       │                                                             │
│       ▼                                                             │
│  Browser connects to 93.184.216.34 via TCP → TLS → HTTP             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Complete Timing Breakdown

```
┌──────────────────────────────────────────────────────────────────┐
│              Typical DNS Resolution Times                          │
├──────────────────────────┬───────────────────────────────────────┤
│ Step                     │ Typical Latency                       │
├──────────────────────────┼───────────────────────────────────────┤
│ Browser cache hit        │ < 0.1 ms                              │
│ OS cache hit             │ < 1 ms                                │
│ Router cache hit         │ 1-5 ms                                │
│ Recursive resolver cache │ 5-30 ms (network round trip)          │
│ Full resolution (cold):  │                                       │
│   → Root server          │ +20-100 ms                            │
│   → TLD server           │ +20-100 ms                            │
│   → Authoritative server │ +20-100 ms                            │
├──────────────────────────┼───────────────────────────────────────┤
│ TOTAL (cold, worst case) │ 100-500 ms                            │
│ TOTAL (warm cache)       │ < 5 ms                                │
└──────────────────────────┴───────────────────────────────────────┘
```

---

## How It Works Internally

### DNS Packet Structure

Every DNS query and response uses the same packet format:

```
┌─────────────────────────────────────────────────────┐
│              DNS Message Format                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌───────────────────────────────────────────┐      │
│  │  HEADER (12 bytes)                         │      │
│  │  - Transaction ID (16 bits)               │      │
│  │  - Flags (QR, Opcode, AA, TC, RD, RA)    │      │
│  │  - Question Count                          │      │
│  │  - Answer Count                            │      │
│  │  - Authority Count                         │      │
│  │  - Additional Count                        │      │
│  └───────────────────────────────────────────┘      │
│                                                     │
│  ┌───────────────────────────────────────────┐      │
│  │  QUESTION SECTION                          │      │
│  │  - Name: www.example.com                  │      │
│  │  - Type: A (1)                            │      │
│  │  - Class: IN (1)                          │      │
│  └───────────────────────────────────────────┘      │
│                                                     │
│  ┌───────────────────────────────────────────┐      │
│  │  ANSWER SECTION                            │      │
│  │  - Name: www.example.com                  │      │
│  │  - Type: A                                │      │
│  │  - Class: IN                              │      │
│  │  - TTL: 300                               │      │
│  │  - Data: 93.184.216.34                    │      │
│  └───────────────────────────────────────────┘      │
│                                                     │
│  ┌───────────────────────────────────────────┐      │
│  │  AUTHORITY SECTION (NS records)            │      │
│  └───────────────────────────────────────────┘      │
│                                                     │
│  ┌───────────────────────────────────────────┐      │
│  │  ADDITIONAL SECTION (glue records)         │      │
│  └───────────────────────────────────────────┘      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### DNS Header Flags Explained

```
┌──┬──────────┬──┬──┬──┬──┬────────┬──┬──┬──┬────────┐
│QR│  Opcode  │AA│TC│RD│RA│   Z    │AD│CD│ RCODE    │
│1 │  4 bits  │1 │1 │1 │1 │ 1 bit  │1 │1 │ 4 bits  │
└──┴──────────┴──┴──┴──┴──┴────────┴──┴──┴──┴────────┘

QR = 0 (Query) or 1 (Response)
AA = Authoritative Answer
TC = Truncated (response too big for UDP, retry with TCP)
RD = Recursion Desired (client wants recursive resolution)
RA = Recursion Available (server supports recursion)
AD = Authenticated Data (DNSSEC validated)
RCODE = Response Code (0=OK, 3=NXDOMAIN, 2=SERVFAIL)
```

### Transport: UDP vs TCP

```
┌──────────────────────────────────────────────────────────────────┐
│              DNS Transport Decision                                │
│                                                                  │
│  Query Size ≤ 512 bytes?                                         │
│       │                                                          │
│       ├── YES → Use UDP (faster, no handshake)                   │
│       │         Port 53                                          │
│       │                                                          │
│       └── NO → Use TCP (reliable, for large responses)           │
│                Port 53                                            │
│                                                                  │
│  Also use TCP when:                                              │
│  - Response has TC (Truncated) flag set                          │
│  - Zone transfers (AXFR/IXFR)                                   │
│  - EDNS0 allows up to 4096 bytes over UDP                       │
│                                                                  │
│  Modern DNS also supports:                                       │
│  - DNS over HTTPS (DoH) → Port 443                              │
│  - DNS over TLS (DoT) → Port 853                                │
│  - DNS over QUIC (DoQ) → Port 853 (UDP)                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### EDNS (Extension Mechanisms for DNS)

Traditional DNS has a 512-byte UDP limit. EDNS0 (RFC 6891) extends this:

```
Additional OPT Record in query:
- Advertises client's UDP buffer size (typically 4096)
- Carries DNSSEC flags (DO bit)
- Carries EDNS Client Subnet (for GeoDNS)

Without EDNS: Max 512 bytes → often truncated → fall back to TCP
With EDNS:    Max 4096 bytes → most responses fit in UDP
```

### Negative Caching (NXDOMAIN)

When a domain doesn't exist, the answer is ALSO cached:

```
Query: "What is nonexistent.example.com?"
Response: NXDOMAIN (this domain does not exist)
          SOA record in Authority section → Minimum TTL = cache time

The resolver caches: "nonexistent.example.com = NXDOMAIN for 86400 seconds"
```

This prevents repeated queries for domains that don't exist.

---

## Code Examples

### Python — Trace Full DNS Resolution Path

```python
import dns.resolver
import dns.name
import dns.rdatatype
import time

def trace_dns_resolution(domain):
    """
    Manually trace DNS resolution from root servers down,
    mimicking what a recursive resolver does internally.
    """
    domain_name = dns.name.from_text(domain)
    print(f"\n{'='*60}")
    print(f"  Tracing DNS resolution for: {domain}")
    print(f"{'='*60}")
    
    # Start from root hints
    # In practice, resolvers have root server IPs hardcoded
    current_nameservers = ['198.41.0.4']  # a.root-servers.net
    
    # Walk down the hierarchy
    labels = str(domain_name).rstrip('.').split('.')
    
    for i in range(len(labels)):
        query_name = '.'.join(labels[-(i+1):]) + '.'
        print(f"\n  Step {i+1}: Querying for '{query_name}'")
        print(f"  Asking: {current_nameservers[0]}")
        
        try:
            # Create a query that doesn't use recursion (like iterative)
            resolver = dns.resolver.Resolver()
            resolver.nameservers = current_nameservers
            
            # Try to get NS records for delegation
            try:
                answer = resolver.resolve(query_name, 'NS')
                ns_names = [str(rr) for rr in answer]
                print(f"  Got NS: {ns_names[:2]}...")
                
                # Resolve NS to IP for next iteration
                for ns in ns_names[:1]:
                    try:
                        ns_ip = resolver.resolve(ns, 'A')
                        current_nameservers = [str(ns_ip[0])]
                    except:
                        pass
            except:
                pass
                
        except Exception as e:
            print(f"  Error: {e}")
    
    # Final query for the actual record
    print(f"\n  Final Step: Querying A record for {domain}")
    resolver = dns.resolver.Resolver()
    answer = resolver.resolve(domain, 'A')
    for rr in answer:
        print(f"  ✓ ANSWER: {domain} → {rr}")
        print(f"  TTL: {answer.rrset.ttl} seconds")

# Simple resolution with timing
def timed_resolution(domain):
    """Resolve a domain and measure how long it takes."""
    start = time.time()
    resolver = dns.resolver.Resolver()
    resolver.nameservers = ['8.8.8.8']  # Use Google DNS
    
    answer = resolver.resolve(domain, 'A')
    elapsed = (time.time() - start) * 1000  # Convert to ms
    
    print(f"{domain} → {answer[0]} ({elapsed:.2f} ms)")
    return answer

# Usage
trace_dns_resolution("www.google.com")
timed_resolution("www.example.com")
```

### Python — Build a Simple DNS Resolver (Educational)

```python
import socket
import struct
import random

def build_dns_query(domain, query_type=1):
    """
    Build a raw DNS query packet from scratch.
    query_type: 1=A, 28=AAAA, 5=CNAME, 15=MX
    """
    # Transaction ID (random 16-bit)
    transaction_id = random.randint(0, 65535)
    
    # Flags: Standard query, recursion desired
    flags = 0x0100  # RD=1
    
    # Header: ID, Flags, QDCount=1, ANCount=0, NSCount=0, ARCount=0
    header = struct.pack('!HHHHHH', 
                         transaction_id, flags, 1, 0, 0, 0)
    
    # Encode domain name (e.g., "www.example.com" → \x03www\x07example\x03com\x00)
    question = b''
    for label in domain.split('.'):
        question += struct.pack('!B', len(label)) + label.encode()
    question += b'\x00'  # End of name
    
    # Query type and class
    question += struct.pack('!HH', query_type, 1)  # Type=A, Class=IN
    
    return header + question, transaction_id

def parse_dns_response(data):
    """Parse a DNS response packet and extract answers."""
    # Parse header
    (tx_id, flags, qd_count, an_count, 
     ns_count, ar_count) = struct.unpack('!HHHHHH', data[:12])
    
    rcode = flags & 0x000F
    if rcode == 3:
        print("  NXDOMAIN - Domain does not exist")
        return []
    
    # Skip question section
    offset = 12
    for _ in range(qd_count):
        while data[offset] != 0:
            offset += data[offset] + 1
        offset += 5  # null byte + type (2) + class (2)
    
    # Parse answer section
    answers = []
    for _ in range(an_count):
        # Handle name compression (pointer = 0xC0xx)
        if data[offset] & 0xC0 == 0xC0:
            offset += 2
        else:
            while data[offset] != 0:
                offset += data[offset] + 1
            offset += 1
        
        rtype, rclass, ttl, rdlength = struct.unpack(
            '!HHIH', data[offset:offset+10])
        offset += 10
        
        if rtype == 1:  # A record
            ip = '.'.join(str(b) for b in data[offset:offset+4])
            answers.append({'type': 'A', 'ip': ip, 'ttl': ttl})
        
        offset += rdlength
    
    return answers

def resolve(domain, dns_server='8.8.8.8'):
    """Send DNS query and get response."""
    query, tx_id = build_dns_query(domain)
    
    # Send via UDP
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(5)
    sock.sendto(query, (dns_server, 53))
    
    response, _ = sock.recvfrom(4096)
    sock.close()
    
    answers = parse_dns_response(response)
    for ans in answers:
        print(f"  {domain} → {ans['ip']} (TTL: {ans['ttl']}s)")
    return answers

# Usage
print("Resolving www.example.com:")
resolve("www.example.com")
```

### Java — DNS Resolution with Timing

```java
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Arrays;

public class DnsResolutionTracer {
    
    public static void main(String[] args) {
        String[] domains = {
            "www.google.com",
            "www.amazon.com",
            "api.github.com",
            "nonexistent.invalid"
        };
        
        for (String domain : domains) {
            resolveWithTiming(domain);
        }
    }
    
    static void resolveWithTiming(String domain) {
        long start = System.nanoTime();
        
        try {
            // InetAddress.getAllByName triggers DNS resolution
            InetAddress[] addresses = InetAddress.getAllByName(domain);
            
            long elapsed = (System.nanoTime() - start) / 1_000_000; // ms
            
            System.out.println("\n" + domain + " (" + elapsed + " ms):");
            for (InetAddress addr : addresses) {
                System.out.println("  → " + addr.getHostAddress());
            }
            
            // Check if result was cached (resolve again)
            long start2 = System.nanoTime();
            InetAddress.getAllByName(domain);
            long elapsed2 = (System.nanoTime() - start2) / 1_000_000;
            System.out.println("  2nd lookup: " + elapsed2 + " ms (cached)");
            
        } catch (UnknownHostException e) {
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            System.out.println("\n" + domain + " → NXDOMAIN (" + elapsed + " ms)");
        }
    }
}
```

### Java — Custom DNS Resolver with Netty (Production-Grade)

```java
import io.netty.resolver.dns.DnsNameResolver;
import io.netty.resolver.dns.DnsNameResolverBuilder;
import io.netty.resolver.dns.DnsServerAddressStreamProviders;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioDatagramChannel;

import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.util.List;
import java.util.concurrent.Future;

public class NettyDnsResolver {
    
    public static void main(String[] args) throws Exception {
        NioEventLoopGroup group = new NioEventLoopGroup(1);
        
        // Build a non-blocking DNS resolver
        DnsNameResolver resolver = new DnsNameResolverBuilder(group.next())
            .channelType(NioDatagramChannel.class)
            .nameServerProvider(
                DnsServerAddressStreamProviders.platformDefault())
            .queryTimeoutMillis(5000)
            .maxQueriesPerResolve(3)  // Max retries
            .build();
        
        try {
            // Resolve asynchronously
            Future<InetAddress> future = resolver.resolve("www.google.com");
            InetAddress address = future.get();
            System.out.println("Resolved: " + address.getHostAddress());
            
            // Resolve all addresses
            Future<List<InetAddress>> allFuture = 
                resolver.resolveAll("www.google.com");
            List<InetAddress> addresses = allFuture.get();
            addresses.forEach(a -> 
                System.out.println("  → " + a.getHostAddress()));
            
        } finally {
            resolver.close();
            group.shutdownGracefully();
        }
    }
}
```

---

## Infrastructure Examples

### Observing DNS Resolution with dig

```bash
# Full trace from root servers (mimics recursive resolution)
dig +trace www.example.com

# Output shows each step:
# .                  518400  IN  NS  a.root-servers.net.
# com.               172800  IN  NS  a.gtld-servers.net.
# example.com.       172800  IN  NS  ns1.example.com.
# www.example.com.   300     IN  A   93.184.216.34

# Query specific DNS server
dig @8.8.8.8 www.example.com A

# See full response with all sections
dig +noall +answer +authority +additional www.example.com

# Check query time
dig www.example.com | grep "Query time"
# ;; Query time: 23 msec

# Query over TCP (for large responses)
dig +tcp www.example.com

# Query with DNSSEC validation
dig +dnssec www.example.com
```

### Configuring a Recursive Resolver (Unbound)

```yaml
# /etc/unbound/unbound.conf
# High-performance recursive resolver

server:
    interface: 0.0.0.0
    port: 53
    
    # Performance tuning
    num-threads: 4
    msg-cache-size: 256m
    rrset-cache-size: 512m
    cache-max-ttl: 86400
    cache-min-ttl: 60
    
    # Security
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    
    # DNSSEC validation
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
    
    # Prefetch almost-expired records
    prefetch: yes
    prefetch-key: yes
    
    # Access control
    access-control: 10.0.0.0/8 allow
    access-control: 127.0.0.0/8 allow
    access-control: 0.0.0.0/0 refuse

# Forward to upstream (if not doing full recursion)
forward-zone:
    name: "."
    forward-addr: 1.1.1.1
    forward-addr: 8.8.8.8
```

### Docker — Running Your Own DNS Resolver

```dockerfile
# Dockerfile for a local DNS resolver with caching
FROM alpine:3.18

RUN apk add --no-cache unbound

COPY unbound.conf /etc/unbound/unbound.conf

# Download root hints
RUN wget -O /etc/unbound/root.hints \
    https://www.internic.net/domain/named.root

EXPOSE 53/udp 53/tcp

CMD ["unbound", "-d", "-c", "/etc/unbound/unbound.conf"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  dns-resolver:
    build: .
    ports:
      - "5353:53/udp"
      - "5353:53/tcp"
    restart: unless-stopped
    volumes:
      - ./unbound.conf:/etc/unbound/unbound.conf:ro
```

---

## Real-World Example

### How Google Public DNS (8.8.8.8) Handles 1 Trillion Queries/Day

```
┌─────────────────────────────────────────────────────────────────────┐
│            Google Public DNS Architecture                             │
│                                                                     │
│  User Query (from anywhere in the world)                            │
│       │                                                             │
│       ▼                                                             │
│  Anycast Routing → Nearest Google PoP                               │
│  (8.8.8.8 is announced from 20+ locations)                          │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────┐                            │
│  │  Layer 1: Frontend Servers           │                            │
│  │  - Parse query                       │                            │
│  │  - Rate limit abusive clients        │                            │
│  │  - Check local cache (millions       │                            │
│  │    of entries in shared memory)      │                            │
│  └──────────────┬──────────────────────┘                            │
│                 │ Cache miss                                         │
│                 ▼                                                    │
│  ┌─────────────────────────────────────┐                            │
│  │  Layer 2: Recursive Resolution       │                            │
│  │  - Full iterative resolution          │                            │
│  │  - DNSSEC validation                  │                            │
│  │  - Prefetching about-to-expire        │                            │
│  │  - Randomized source ports            │                            │
│  │  - 0x20 encoding (case randomization)│                            │
│  └──────────────┬──────────────────────┘                            │
│                 │                                                    │
│                 ▼                                                    │
│  ┌─────────────────────────────────────┐                            │
│  │  Layer 3: Shared Cache (Distributed) │                            │
│  │  - Distributed across all nodes       │                            │
│  │  - One resolution serves all users   │                            │
│  │  - Massive hit rate (>95%)           │                            │
│  └─────────────────────────────────────┘                            │
│                                                                     │
│  Security measures:                                                 │
│  - Source port randomization (prevent Kaminsky attack)               │
│  - DNSSEC validation                                                │
│  - DNS-over-HTTPS (8.8.8.8/dns-query)                              │
│  - DNS-over-TLS (dns.google port 853)                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Netflix's DNS Resolution Strategy

Netflix uses DNS creatively for global routing:

1. Client queries `api.netflix.com`
2. Netflix's Route 53 returns different IPs based on:
   - Client's geographic location (GeoDNS)
   - Current health of backend servers
   - Current load levels
3. Short TTLs (60s) allow rapid failover
4. If US-East fails → DNS returns US-West IPs within 60 seconds

---

## Common Mistakes / Pitfalls

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Hardcoding DNS server | Breaks in different networks | Use system resolver or well-known (8.8.8.8) |
| Not handling NXDOMAIN | App crashes on non-existent domains | Always handle `UnknownHostException` |
| Ignoring TTL | Stale data served after IP changes | Respect TTL in your application cache |
| Not having fallback DNS | ISP DNS goes down → everything fails | Configure multiple resolvers |
| Caching forever | IP changes but app uses old IP | Re-resolve periodically based on TTL |
| Blocking on DNS | Whole app hangs during resolution | Use async DNS resolution |
| Testing with production DNS | Slow, unreliable tests | Use local DNS or mock for tests |
| Ignoring IPv6 (AAAA) | Fails on IPv6-only networks | Query both A and AAAA |

---

## When to Use / When NOT to Use

### Use Custom/Direct DNS Resolution When:
- ✅ Building a CDN or anycast service
- ✅ Implementing service discovery
- ✅ Need fine-grained control over resolution
- ✅ Building security tools (DNS monitoring)
- ✅ Need to query specific record types (TXT, SRV)

### Use System DNS Resolution When:
- ✅ Normal web applications (let the OS handle it)
- ✅ Standard HTTP clients
- ✅ Don't need record-type-specific queries

### Run Your Own Recursive Resolver When:
- ✅ Privacy requirements (don't want ISP to see queries)
- ✅ Need custom DNS filtering (block ads, malware)
- ✅ Low-latency environments (local cache reduces latency)
- ✅ Enterprise networks (central DNS policy)

---

## Key Takeaways

- **DNS resolution is hierarchical**: Browser → OS → Router → Recursive Resolver → Root → TLD → Authoritative.
- **Caching happens at EVERY level** — most DNS queries never reach the authoritative server.
- **Recursive resolvers do the heavy lifting** — they iterate through the hierarchy on your behalf.
- **There are only 13 root server clusters** but 1500+ physical instances worldwide using Anycast.
- **DNS primarily uses UDP** (port 53) for speed, falling back to TCP for large responses or zone transfers.
- **Cold resolution takes 100-500ms** but cached resolution takes < 5ms — this is why TTL matters.
- **Modern DNS adds encryption**: DNS-over-HTTPS (DoH) and DNS-over-TLS (DoT) prevent eavesdropping.

---

## What's Next?

Now that you understand how DNS resolution works, let's see how it's used for something powerful: distributing traffic across multiple servers. See [03-dns-load-balancing.md](./03-dns-load-balancing.md).
