# Chapter 1.3: DNS — How Your Browser Finds a Website

> **Level**: ⭐ Beginner  
> **Goal**: Understand DNS (Domain Name System) — the "phone book" of the internet — and how your browser turns `google.com` into an IP address.

---

## 🧠 The Problem — Humans vs Computers

**Computers** communicate using IP addresses: `142.250.190.46`

**Humans** remember names: `google.com`

Imagine if you had to memorize IP addresses for every website:
- Google → `142.250.190.46`
- Facebook → `157.240.1.35`
- Amazon → `205.251.242.103`

That would be **impossible**! 

> **DNS solves this problem**. It's like a **phone book** — you look up a name, and it gives you the number (IP address).

```
    You say:  "google.com"
                  │
                  ▼
            ┌──────────┐
            │   DNS    │
            │  Server  │   ← The Phone Book of the Internet
            └─────┬────┘
                  │
                  ▼
            "142.250.190.46"
                  │
                  ▼
            Browser connects to this IP address
```

---

## 📖 What is a Domain Name?

Let's break down a domain name:

```
         https://www.mail.google.com/inbox
         ─────   ─── ──── ────── ─── ─────
           │      │    │     │    │     │
           │      │    │     │    │     └─── Path (page on the website)
           │      │    │     │    │
           │      │    │     │    └────────── TLD (Top Level Domain)
           │      │    │     │                .com, .org, .net, .in, .io
           │      │    │     │
           │      │    │     └─────────────── Domain Name (google)
           │      │    │                      This is what you buy/register
           │      │    │
           │      │    └───────────────────── Subdomain (mail)
           │      │                           Can have multiple: mail, docs, drive
           │      │
           │      └────────────────────────── Subdomain (www)
           │                                  "www" is just a subdomain!
           │
           └───────────────────────────────── Protocol (https)
```

### Domain Hierarchy — It's Like an Address System

```
    Think of it like a physical address (read RIGHT to LEFT):
    
    "mail.google.com"
    
    .com     = Country (like "India")
    google   = City (like "Mumbai")
    mail     = Street/Building (like "MG Road")
    
    
    The DNS hierarchy (read from top to bottom):
    
                         . (Root)
                         │
              ┌──────────┼──────────┐
              │          │          │
            .com       .org      .in
              │          │          │
         ┌────┼────┐     │     ┌───┼───┐
         │    │    │     │     │       │
      google amazon │  wikipedia  flipkart  gov
         │    │    │              │
      ┌──┼──┐ │  facebook     ┌──┼──┐
      │  │  │ │               │     │
    www mail drive           www   seller
    
    
    So "mail.google.com" is actually read as:
    . (root) → .com → google → mail
```

---

## 🔍 DNS Resolution — Step by Step (The Full Journey)

When you type `www.google.com` in your browser, here's **exactly** what happens:

```
    ┌──────────┐
    │ BROWSER  │  "I need the IP for www.google.com"
    └────┬─────┘
         │
         │ Step 1: Check BROWSER CACHE
         │ "Have I looked this up recently?"
         │
         │ Cache HIT? ──▶ Use cached IP ──▶ Done! ✅
         │ Cache MISS? ──▶ Continue ↓
         │
         │ Step 2: Check OS CACHE
         │ "Has any app on this computer looked this up?"
         │
         │ Cache HIT? ──▶ Use cached IP ──▶ Done! ✅
         │ Cache MISS? ──▶ Continue ↓
         │
         │ Step 3: Check HOSTS FILE
         │ (A local file on your computer with manual entries)
         │ Windows: C:\Windows\System32\drivers\etc\hosts
         │ Linux/Mac: /etc/hosts
         │
         │ Found? ──▶ Use that IP ──▶ Done! ✅
         │ Not found? ──▶ Continue ↓
         │
         ▼
    ┌──────────────────┐
    │  RESOLVER         │  (Usually your ISP's DNS server)
    │  (Recursive DNS)  │  "Let me find this for you!"
    │                   │
    │  Also checks its  │  Cache HIT? ──▶ Done! ✅
    │  own cache first  │  Cache MISS? ──▶ Continue ↓
    └────────┬─────────┘
             │
             │ Step 4: Ask ROOT NAME SERVER
             ▼
    ┌────────────────┐
    │  ROOT SERVER   │  "I don't know google.com, but I know
    │  (. )          │   who handles .com domains"
    │                │
    │  There are 13  │──▶ "Go ask the .com TLD server"
    │  root servers  │    (Returns IP of .com TLD server)
    │  worldwide     │
    └────────────────┘
             │
             │ Step 5: Ask TLD NAME SERVER
             ▼
    ┌────────────────┐
    │  TLD SERVER    │  "I don't know www.google.com, but I know
    │  (.com)        │   who handles google.com"
    │                │
    │  Handles ALL   │──▶ "Go ask Google's nameserver"
    │  .com domains  │    (Returns IP of Google's NS)
    └────────────────┘
             │
             │ Step 6: Ask AUTHORITATIVE NAME SERVER
             ▼
    ┌────────────────────┐
    │  AUTHORITATIVE NS  │  "Yes! I am Google's official DNS server."
    │  (ns1.google.com)  │  "www.google.com = 142.250.190.46"
    │                    │
    │  This server has   │──▶ Returns: 142.250.190.46
    │  the FINAL answer  │
    └────────────────────┘
             │
             │ Step 7: Response travels back
             ▼
    ┌──────────────────┐
    │  RESOLVER         │  Caches the result for next time
    │                   │  (TTL: Time To Live = e.g., 300 seconds)
    └────────┬─────────┘
             │
             ▼
    ┌──────────┐
    │ BROWSER  │  Now knows: www.google.com = 142.250.190.46
    │          │  Caches it too!
    │          │  Connects to 142.250.190.46
    └──────────┘
```

---

## ⏱️ How Fast is DNS Resolution?

```
    ┌─────────────────────────────────────────────────┐
    │                                                 │
    │  Browser Cache Hit:        ~0 ms   (instant)    │
    │  OS Cache Hit:             ~1 ms                │
    │  Resolver Cache Hit:       ~5-10 ms             │
    │  Full DNS Resolution:      ~50-200 ms           │
    │                                                 │
    │  That's why caching is SO important!            │
    │  Most DNS lookups are served from cache.        │
    │                                                 │
    └─────────────────────────────────────────────────┘
```

---

## 📝 DNS Record Types — What Information DNS Stores

DNS doesn't just store IP addresses. It stores multiple **types of records**:

```
    ┌──────────┬────────────────────────────────────────────────────────────┐
    │  Record  │  What it does                                             │
    │  Type    │                                                           │
    ├──────────┼────────────────────────────────────────────────────────────┤
    │          │                                                           │
    │  A       │  Maps domain → IPv4 address                               │
    │          │  google.com → 142.250.190.46                              │
    │          │                                                           │
    ├──────────┼────────────────────────────────────────────────────────────┤
    │          │                                                           │
    │  AAAA    │  Maps domain → IPv6 address                               │
    │          │  google.com → 2607:f8b0:4004:800::200e                    │
    │          │                                                           │
    ├──────────┼────────────────────────────────────────────────────────────┤
    │          │                                                           │
    │  CNAME   │  Alias — points one domain to another domain              │
    │          │  www.google.com → google.com                              │
    │          │  (Like a nickname pointing to the real name)              │
    │          │                                                           │
    ├──────────┼────────────────────────────────────────────────────────────┤
    │          │                                                           │
    │  MX      │  Mail server — where to send emails                       │
    │          │  google.com → smtp.google.com (priority: 10)              │
    │          │                                                           │
    ├──────────┼────────────────────────────────────────────────────────────┤
    │          │                                                           │
    │  NS      │  Name server — which DNS server is authoritative          │
    │          │  google.com → ns1.google.com                              │
    │          │                                                           │
    ├──────────┼────────────────────────────────────────────────────────────┤
    │          │                                                           │
    │  TXT     │  Text record — stores any text (used for verification)    │
    │          │  google.com → "v=spf1 include:_spf.google.com ~all"      │
    │          │                                                           │
    ├──────────┼────────────────────────────────────────────────────────────┤
    │          │                                                           │
    │  SOA     │  Start of Authority — master info about the domain        │
    │          │  Who manages it, when it was last updated, etc.           │
    │          │                                                           │
    └──────────┴────────────────────────────────────────────────────────────┘
```

### Try It Yourself! (See Real DNS Records)

```bash
# On Windows (Command Prompt or PowerShell):
nslookup google.com
nslookup -type=MX google.com
nslookup -type=NS google.com

# On Mac/Linux:
dig google.com
dig google.com MX
dig google.com NS

# Online tool:
# Visit: https://mxtoolbox.com/DNSLookup.aspx
```

**Example output of `nslookup google.com`:**
```
Server:    8.8.8.8         ← DNS resolver used (Google's public DNS)
Address:   8.8.8.8#53      ← Port 53 (DNS always uses port 53)

Name:      google.com
Address:   142.250.190.46  ← The A record (IPv4 address)
```

---

## 🕐 TTL — Time To Live (How Long to Cache?)

When a DNS server responds, it includes a **TTL** value — telling the requester **how long to cache** this result:

```
    DNS Response:
    ┌─────────────────────────────────────┐
    │  Domain:   google.com               │
    │  Type:     A                        │
    │  Value:    142.250.190.46           │
    │  TTL:      300 seconds (5 minutes)  │  ← Cache this for 5 minutes!
    └─────────────────────────────────────┘
    
    What this means:
    ────────────────
    
    Time 0:00  →  First lookup: goes through full DNS resolution
    Time 0:01  →  Second lookup: uses CACHE (instant!) ⚡
    Time 2:30  →  Another lookup: still uses CACHE ⚡
    Time 5:01  →  Cache EXPIRED → full DNS resolution again
    
    
    Common TTL values:
    ┌────────────────┬──────────────────────────────────────┐
    │  TTL           │  Used for                            │
    ├────────────────┼──────────────────────────────────────┤
    │  60 seconds    │  Frequently changing services        │
    │  300 seconds   │  Standard websites                   │
    │  3600 seconds  │  Stable services                     │
    │  86400 seconds │  Very stable (1 day)                 │
    └────────────────┴──────────────────────────────────────┘
```

---

## 🔄 DNS in the Real World — How Companies Use It

### 1. Multiple IPs for One Domain (Load Distribution)

Google doesn't have just ONE server. DNS can return **multiple IPs**:

```
    nslookup google.com
    
    Address: 142.250.190.46
    Address: 142.250.190.78
    Address: 142.250.190.110
    Address: 142.250.190.14
    
    Each IP points to a different server!
    Your browser picks one (usually the first).
    
    DNS rotates the order each time (Round Robin DNS):
    
    Request 1 → [46, 78, 110, 14]  → User goes to server 46
    Request 2 → [78, 110, 14, 46]  → User goes to server 78
    Request 3 → [110, 14, 46, 78]  → User goes to server 110
    
    This distributes traffic across multiple servers!
```

### 2. GeoDNS — Different IP Based on Location

```
    User in India:
    ┌──────┐                    ┌──────────────────┐
    │ User │── google.com? ──▶ │ DNS returns:      │
    │ 🇮🇳   │                    │ 142.250.182.206  │ ← Server in Mumbai
    └──────┘                    └──────────────────┘
    
    User in USA:
    ┌──────┐                    ┌──────────────────┐
    │ User │── google.com? ──▶ │ DNS returns:      │
    │ 🇺🇸   │                    │ 142.250.190.46   │ ← Server in Oregon
    └──────┘                    └──────────────────┘
    
    This way, users are directed to the NEAREST server = faster response!
```

### 3. DNS Failover — If a Server Goes Down

```
    Normal:
    api.myapp.com → 10.0.1.100 (Primary Server ✅)
    
    Server goes down:
    api.myapp.com → 10.0.1.100 (Primary Server ❌ DEAD!)
                        │
                        ▼
    Health check detects failure
                        │
                        ▼
    DNS automatically updates:
    api.myapp.com → 10.0.2.200 (Backup Server ✅)
    
    Users are now directed to the backup server!
```

---

## 🏗️ DNS Infrastructure — The Root Servers

There are only **13 root server addresses** in the world (named A through M), but each "address" actually has **hundreds of physical servers** spread globally using a technique called **Anycast**:

```
    The 13 Root Servers:
    ┌──────────────────────────────────────────┐
    │  A  - Verisign (USA)                     │
    │  B  - USC-ISI (USA)                      │
    │  C  - Cogent Communications (USA)        │
    │  D  - University of Maryland (USA)       │
    │  E  - NASA (USA)                         │
    │  F  - Internet Systems Consortium (USA)  │
    │  G  - US DoD (USA)                       │
    │  H  - US Army Research Lab (USA)         │
    │  I  - Netnod (Sweden)                    │
    │  J  - Verisign (USA)                     │
    │  K  - RIPE NCC (Netherlands)             │
    │  L  - ICANN (USA)                        │
    │  M  - WIDE Project (Japan)               │
    └──────────────────────────────────────────┘
    
    Total physical root servers: 1,500+ worldwide
    (Using Anycast, queries go to the nearest one)
```

---

## 🛡️ DNS Security Concerns

```
    ┌───────────────────────────────────────────────────────┐
    │  THREAT: DNS Spoofing / Cache Poisoning               │
    │                                                       │
    │  Normal:                                              │
    │  User → "bank.com?" → DNS → 93.184.216.34 (Real)     │
    │                                                       │
    │  Attack:                                              │
    │  Attacker poisons the DNS cache with a FAKE IP:       │
    │  User → "bank.com?" → DNS → 66.66.66.66 (FAKE!) ☠️   │
    │  User thinks they're on their bank's website,         │
    │  but they're on the attacker's site!                  │
    │                                                       │
    │  SOLUTION: DNSSEC (DNS Security Extensions)           │
    │  Adds digital signatures to DNS records               │
    │  so clients can verify the response is authentic.     │
    └───────────────────────────────────────────────────────┘
```

---

## 🧪 DNS in Python & Java — Programmatic Lookups

### Python
```python
import socket

# Simple DNS lookup
ip_address = socket.gethostbyname('google.com')
print(f"google.com → {ip_address}")
# Output: google.com → 142.250.190.46

# Get all IPs for a domain
results = socket.getaddrinfo('google.com', 80)
for result in results:
    print(f"  IP: {result[4][0]}")

# Reverse DNS lookup (IP → Domain)
domain = socket.gethostbyaddr('142.250.190.46')
print(f"142.250.190.46 → {domain[0]}")
```

### Java
```java
import java.net.InetAddress;

public class DNSLookup {
    public static void main(String[] args) throws Exception {
        // Simple DNS lookup
        InetAddress address = InetAddress.getByName("google.com");
        System.out.println("google.com → " + address.getHostAddress());
        
        // Get all IPs
        InetAddress[] allAddresses = InetAddress.getAllByName("google.com");
        for (InetAddress addr : allAddresses) {
            System.out.println("  IP: " + addr.getHostAddress());
        }
    }
}
```

---

## 🔑 Key Takeaways

```
╔════════════════════════════════════════════════════════════════════╗
║                                                                    ║
║  1. DNS = Translates human-readable names (google.com) to         ║
║     machine-readable IP addresses (142.250.190.46)                 ║
║                                                                    ║
║  2. DNS resolution checks in order:                                ║
║     Browser Cache → OS Cache → Hosts File → Resolver →            ║
║     Root Server → TLD Server → Authoritative Server                ║
║                                                                    ║
║  3. DNS records: A (IPv4), AAAA (IPv6), CNAME (alias),            ║
║     MX (email), NS (name server), TXT (text/verification)         ║
║                                                                    ║
║  4. TTL controls how long DNS results are cached                   ║
║                                                                    ║
║  5. Real companies use DNS for load balancing (Round Robin),       ║
║     geographic routing (GeoDNS), and failover                      ║
║                                                                    ║
║  6. There are 13 root server addresses (1,500+ physical servers)   ║
║                                                                    ║
╚════════════════════════════════════════════════════════════════════╝
```

---

[⬅️ Previous: How the Internet Works](./02-how-the-internet-works.md) | [Next: HTTP & HTTPS ➡️](./04-http-and-https.md)
