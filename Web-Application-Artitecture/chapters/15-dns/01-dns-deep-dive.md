# DNS Deep Dive — Records (A, AAAA, CNAME, MX, TXT, NS)

> **What you'll learn**: Every type of DNS record that exists, what it does, how it's structured internally, and when to use each one — explained so clearly that you'll never be confused by DNS again.

---

## Real-Life Analogy

Imagine you have a contact card for your friend. On that card, you don't just write their phone number — you also write:

- **Home address** (where they live)
- **Office address** (where they work)
- **Email address** (how to send them messages)
- **Alternate name** ("Hey, you can also reach me as Johnny instead of Jonathan")
- **Verification note** ("Yes, this is really my card — here's my signature")

DNS records work the same way. A domain name (like `google.com`) doesn't just have ONE piece of information — it has **multiple types of records**, each serving a different purpose. Some tell you the IP address, some tell you where to send email, some prove ownership, and some redirect you to other names.

---

## What Are DNS Records?

DNS records are **entries in a DNS zone file** that tell the world information about a domain. When someone queries `example.com`, the DNS server doesn't just return an IP — it looks up whichever **record type** was asked for.

Think of a DNS zone file as a spreadsheet:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ZONE FILE: example.com                            │
├──────────┬──────┬─────────┬──────────────────────────────────────────┤
│ Name     │ TTL  │ Type    │ Value                                    │
├──────────┼──────┼─────────┼──────────────────────────────────────────┤
│ @        │ 300  │ A       │ 93.184.216.34                            │
│ @        │ 300  │ AAAA    │ 2606:2800:220:1:248:1893:25c8:1946       │
│ www      │ 300  │ CNAME   │ example.com                              │
│ @        │ 3600 │ MX      │ 10 mail.example.com                      │
│ @        │ 3600 │ TXT     │ "v=spf1 include:_spf.google.com ~all"    │
│ @        │ 86400│ NS      │ ns1.exampledns.com                       │
│ @        │ 86400│ NS      │ ns2.exampledns.com                       │
│ @        │ 3600 │ SOA     │ ns1.exampledns.com admin.example.com ... │
└──────────┴──────┴─────────┴──────────────────────────────────────────┘
```

---

## Core DNS Record Types — Explained One by One

### 1. A Record (Address Record)

**Purpose**: Maps a domain name to an **IPv4 address** (the 32-bit address like `192.168.1.1`).

This is THE most common DNS record. When you type `google.com` in your browser, an A record lookup happens.

```
┌──────────────────────────────────────────┐
│          A Record Lookup                  │
│                                          │
│  Query:  "What is the IP of google.com?" │
│  Answer:  google.com → 142.250.190.78    │
└──────────────────────────────────────────┘

Browser ──▶ "Give me A record for google.com"
        ◀── "142.250.190.78"
Browser ──▶ Connects to 142.250.190.78:443
```

**Key facts about A records:**
- Returns a 32-bit IPv4 address
- A domain can have MULTIPLE A records (for load balancing)
- TTL controls how long the answer is cached

**Example zone entry:**
```
example.com.    300    IN    A    93.184.216.34
example.com.    300    IN    A    93.184.216.35   ← Multiple IPs!
```

---

### 2. AAAA Record (IPv6 Address Record)

**Purpose**: Maps a domain name to an **IPv6 address** (the 128-bit address like `2001:0db8::1`).

It's called "AAAA" (quad-A) because IPv6 addresses are 4× longer than IPv4 (128 bits vs 32 bits).

```
┌──────────────────────────────────────────────────────────────┐
│          AAAA Record Lookup                                    │
│                                                              │
│  Query:  "What is the IPv6 address of google.com?"            │
│  Answer:  google.com → 2607:f8b0:4004:800::200e              │
└──────────────────────────────────────────────────────────────┘
```

**Why does AAAA exist?**
- IPv4 has only ~4.3 billion addresses (we've run out!)
- IPv6 has 340 undecillion addresses (essentially infinite)
- Modern systems query BOTH A and AAAA records simultaneously

**Example zone entry:**
```
example.com.    300    IN    AAAA    2606:2800:220:1:248:1893:25c8:1946
```

---

### 3. CNAME Record (Canonical Name Record)

**Purpose**: Creates an **alias** — points one domain name to another domain name (NOT to an IP).

Think of it as a redirect: "If you're looking for `www.example.com`, go ask `example.com` instead."

```
┌─────────────────────────────────────────────────────────────┐
│              CNAME Chain                                       │
│                                                             │
│  www.example.com ──CNAME──▶ example.com ──A──▶ 93.184.216.34│
│                                                             │
│  blog.example.com ──CNAME──▶ myblog.wordpress.com           │
│                     ──A──▶ 192.0.78.13                      │
└─────────────────────────────────────────────────────────────┘
```

**How CNAME resolution works step-by-step:**

```
Step 1: Browser asks "What's the IP of www.example.com?"
Step 2: DNS says "www.example.com is a CNAME for example.com"
Step 3: Browser asks "What's the IP of example.com?"
Step 4: DNS says "example.com A record is 93.184.216.34"
Step 5: Browser connects to 93.184.216.34
```

**Critical CNAME rules:**
- ❌ CNAME **cannot** exist at the zone apex (root domain like `example.com`)
- ❌ CNAME **cannot** coexist with other records for the same name
- ✅ CNAME is great for subdomains (`www`, `blog`, `api`)
- ⚠️ CNAME chains add latency (each hop = another DNS lookup)

**Example zone entry:**
```
www.example.com.     300    IN    CNAME    example.com.
blog.example.com.    300    IN    CNAME    mysite.wordpress.com.
```

> **Why can't CNAME be at the root?**  
> Because the root domain MUST have an SOA and NS record. The DNS spec says if a CNAME exists for a name, NO other records can exist for that name. This would conflict with required SOA/NS records. Some providers (like Cloudflare) offer "CNAME flattening" as a workaround.

---

### 4. MX Record (Mail Exchange Record)

**Purpose**: Tells email systems **where to deliver email** for a domain.

When someone sends an email to `user@example.com`, their mail server asks: "Where should I deliver mail for `example.com`?" The MX record answers.

```
┌────────────────────────────────────────────────────────────────┐
│              Email Delivery via MX Records                       │
│                                                                │
│  Sender's Mail Server                                          │
│       │                                                        │
│       ▼                                                        │
│  DNS Query: "MX record for example.com?"                       │
│       │                                                        │
│       ▼                                                        │
│  Response: 10 mail1.example.com                                │
│            20 mail2.example.com  (backup)                      │
│       │                                                        │
│       ▼                                                        │
│  Connects to mail1.example.com (priority 10 = primary)         │
│  If down → tries mail2.example.com (priority 20 = backup)      │
└────────────────────────────────────────────────────────────────┘
```

**MX record structure:**
```
domain.    TTL    IN    MX    priority    mail-server
```

**Priority explained:**
- Lower number = **higher priority** (tried first)
- If priority 10 server is down, try priority 20
- Same priority = load balance between them

**Example zone entries (using Google Workspace):**
```
example.com.    3600    IN    MX    1     aspmx.l.google.com.
example.com.    3600    IN    MX    5     alt1.aspmx.l.google.com.
example.com.    3600    IN    MX    5     alt2.aspmx.l.google.com.
example.com.    3600    IN    MX    10    alt3.aspmx.l.google.com.
example.com.    3600    IN    MX    10    alt4.aspmx.l.google.com.
```

---

### 5. TXT Record (Text Record)

**Purpose**: Stores **arbitrary text** associated with a domain. Used primarily for verification and security.

TXT records are the Swiss Army knife of DNS. They're used for:

```
┌─────────────────────────────────────────────────────────────────┐
│                  Common TXT Record Uses                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. SPF (Sender Policy Framework)                               │
│     → "These IPs are allowed to send email for my domain"       │
│                                                                 │
│  2. DKIM (DomainKeys Identified Mail)                           │
│     → "Here's my public key to verify email signatures"         │
│                                                                 │
│  3. DMARC (Domain-based Message Authentication)                 │
│     → "Here's my policy for handling failed email checks"       │
│                                                                 │
│  4. Domain Verification                                         │
│     → "Google/AWS/Stripe, I own this domain — here's proof"     │
│                                                                 │
│  5. Custom Data                                                 │
│     → Any text up to 255 characters per string                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example zone entries:**
```
; SPF record — only Google's servers can send email for us
example.com.  3600  IN  TXT  "v=spf1 include:_spf.google.com ~all"

; Google domain verification
example.com.  3600  IN  TXT  "google-site-verification=abc123xyz"

; DMARC policy
_dmarc.example.com. 3600 IN TXT "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"

; DKIM selector
selector1._domainkey.example.com. 3600 IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCS..."
```

---

### 6. NS Record (Name Server Record)

**Purpose**: Specifies which **DNS servers are authoritative** for a domain — "these servers have the official answers."

```
┌──────────────────────────────────────────────────────────────────┐
│              NS Record Delegation                                  │
│                                                                  │
│  Root DNS: "Who handles .com?"                                   │
│       │                                                          │
│       ▼                                                          │
│  .com TLD: "Who handles example.com?"                            │
│       │                                                          │
│       ▼                                                          │
│  NS records say: ns1.cloudflare.com, ns2.cloudflare.com          │
│       │                                                          │
│       ▼                                                          │
│  Ask ns1.cloudflare.com for example.com's A record               │
│       │                                                          │
│       ▼                                                          │
│  Answer: 93.184.216.34                                           │
└──────────────────────────────────────────────────────────────────┘
```

**Key facts about NS records:**
- Every domain MUST have at least 2 NS records (for redundancy)
- NS records are set at the **registrar** level to delegate authority
- Changing NS records = "I'm moving my DNS to a new provider"

**Example zone entries:**
```
example.com.    86400    IN    NS    ns1.cloudflare.com.
example.com.    86400    IN    NS    ns2.cloudflare.com.
```

---

### 7. SOA Record (Start of Authority)

**Purpose**: Contains **administrative information** about the zone — who manages it, serial number, refresh timers.

Every DNS zone has exactly ONE SOA record. It's like the metadata header.

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOA Record Structure                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Primary NS:    ns1.example.com    (master name server)         │
│  Admin Email:   admin.example.com  (@ replaced with .)         │
│  Serial:        2024010101         (version number)             │
│  Refresh:       3600               (seconds between updates)    │
│  Retry:         600                (retry if refresh fails)     │
│  Expire:        604800             (give up after this)         │
│  Minimum TTL:   300                (negative caching TTL)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example:**
```
example.com. 86400 IN SOA ns1.example.com. admin.example.com. (
    2024010101   ; Serial (increment on every change)
    3600         ; Refresh (1 hour)
    600          ; Retry (10 minutes)
    604800       ; Expire (1 week)
    300          ; Minimum TTL (5 minutes)
)
```

---

### 8. Other Important Record Types

#### SRV Record (Service Record)

**Purpose**: Specifies the host and port for specific services (like SIP, XMPP, LDAP).

```
_service._protocol.domain.  TTL  IN  SRV  priority weight port target

Example:
_sip._tcp.example.com. 3600 IN SRV 10 60 5060 sipserver.example.com.
```

#### PTR Record (Pointer Record)

**Purpose**: Reverse DNS — maps an IP address **back** to a domain name.

```
34.216.184.93.in-addr.arpa.  3600  IN  PTR  example.com.
```

Used for:
- Email server verification (anti-spam)
- Network troubleshooting (`nslookup` on an IP)
- Security logging

#### CAA Record (Certification Authority Authorization)

**Purpose**: Specifies which Certificate Authorities (CAs) are allowed to issue SSL certificates for your domain.

```
example.com.  3600  IN  CAA  0 issue "letsencrypt.org"
example.com.  3600  IN  CAA  0 issuewild "letsencrypt.org"
```

#### ALIAS / ANAME Record (Provider-Specific)

**Purpose**: Like CNAME but allowed at the zone apex. Not a standard DNS record — implemented by providers like Cloudflare, Route 53, and DNSimple.

```
example.com.  300  IN  ALIAS  myapp.herokuapp.com.
```

---

## Complete Record Type Comparison Table

| Record | Purpose | Points To | At Root? | Example Use Case |
|--------|---------|-----------|----------|-----------------|
| **A** | Domain → IPv4 | IP Address | ✅ Yes | Website hosting |
| **AAAA** | Domain → IPv6 | IPv6 Address | ✅ Yes | Modern hosting |
| **CNAME** | Alias → Another domain | Domain name | ❌ No | `www` → apex, CDN |
| **MX** | Email routing | Mail server | ✅ Yes | Google Workspace |
| **TXT** | Text/verification | Free text | ✅ Yes | SPF, DKIM, ownership |
| **NS** | DNS delegation | Name server | ✅ Yes | DNS provider |
| **SOA** | Zone metadata | Admin info | ✅ Yes | Zone management |
| **SRV** | Service location | Host + Port | ✅ Yes | VoIP, gaming |
| **PTR** | Reverse DNS | Domain name | N/A | Email verification |
| **CAA** | CA authorization | CA domain | ✅ Yes | SSL security |

---

## How It Works Internally

### DNS Zone File Structure

A zone file is a plain text file stored on authoritative DNS servers. Here's the internal structure:

```
; Zone file for example.com
; This file is loaded by BIND/NSD/PowerDNS

$ORIGIN example.com.        ; Base domain (all relative names are appended)
$TTL 300                     ; Default TTL for all records

; --- SOA Record (required, exactly one) ---
@   IN  SOA  ns1.example.com. admin.example.com. (
        2024061501  ; Serial
        3600        ; Refresh
        900         ; Retry
        604800      ; Expire
        86400       ; Negative caching TTL
)

; --- Name Servers (required, at least 2) ---
@       IN  NS      ns1.example.com.
@       IN  NS      ns2.example.com.

; --- A Records ---
@       IN  A       93.184.216.34
ns1     IN  A       198.51.100.1
ns2     IN  A       198.51.100.2
www     IN  A       93.184.216.34

; --- AAAA Records ---
@       IN  AAAA    2606:2800:220:1:248:1893:25c8:1946

; --- CNAME Records ---
blog    IN  CNAME   example.com.
api     IN  CNAME   api-lb.us-east.elb.amazonaws.com.

; --- MX Records ---
@       IN  MX  10  mail.example.com.
@       IN  MX  20  mail-backup.example.com.
mail    IN  A       198.51.100.10

; --- TXT Records ---
@       IN  TXT     "v=spf1 include:_spf.google.com ~all"
```

### DNS Wire Format (How Records Travel Over the Network)

When DNS records are transmitted, they use a **binary wire format** (not plain text):

```
┌────────────────────────────────────────────────────────────┐
│              DNS Resource Record Wire Format                 │
├──────────────┬────────────────────────────────────────────────┤
│ NAME         │ Variable length (compressed domain name)      │
│ TYPE         │ 2 bytes (A=1, AAAA=28, CNAME=5, MX=15...)   │
│ CLASS        │ 2 bytes (IN=1 for Internet)                  │
│ TTL          │ 4 bytes (seconds to cache)                   │
│ RDLENGTH     │ 2 bytes (length of RDATA)                    │
│ RDATA        │ Variable (the actual answer data)            │
└──────────────┴────────────────────────────────────────────────┘
```

**Record type numbers (from IANA registry):**
```
A     = 1       NS    = 2       CNAME = 5
SOA   = 6       PTR   = 12      MX    = 15
TXT   = 16      AAAA  = 28      SRV   = 33
CAA   = 257
```

### Name Compression

DNS uses **pointer compression** to avoid repeating domain names. If `example.com` appears multiple times in a response, subsequent occurrences use a 2-byte pointer to the first occurrence:

```
Byte sequence: [0xC0][0x0C]
               ↑              ↑
          Pointer flag    Offset to name (byte 12)
```

This reduces packet sizes significantly — important because DNS typically uses UDP with a 512-byte limit (or 4096 with EDNS).

---

## Code Examples

### Python — Query All DNS Record Types

```python
import dns.resolver  # pip install dnspython

def query_all_records(domain):
    """Query and display all common DNS record types for a domain."""
    record_types = ['A', 'AAAA', 'CNAME', 'MX', 'TXT', 'NS', 'SOA', 'CAA']
    
    for rtype in record_types:
        try:
            answers = dns.resolver.resolve(domain, rtype)
            print(f"\n{'='*50}")
            print(f"  {rtype} Records for {domain}")
            print(f"{'='*50}")
            for rdata in answers:
                if rtype == 'MX':
                    print(f"  Priority: {rdata.preference}, Server: {rdata.exchange}")
                elif rtype == 'SOA':
                    print(f"  Primary NS: {rdata.mname}")
                    print(f"  Admin: {rdata.rname}")
                    print(f"  Serial: {rdata.serial}")
                else:
                    print(f"  {rdata}")
            print(f"  TTL: {answers.rrset.ttl} seconds")
        except dns.resolver.NoAnswer:
            print(f"  {rtype}: No records found")
        except dns.resolver.NXDOMAIN:
            print(f"  Domain {domain} does not exist!")
            break
        except Exception as e:
            print(f"  {rtype}: Error - {e}")

# Usage
query_all_records("google.com")
```

### Python — Create DNS Records Programmatically (with Cloudflare API)

```python
import requests

# Cloudflare API - Add DNS records programmatically
CLOUDFLARE_API = "https://api.cloudflare.com/client/v4"
ZONE_ID = "your_zone_id"
API_TOKEN = "your_api_token"  # Store securely, never hardcode!

headers = {
    "Authorization": f"Bearer {API_TOKEN}",
    "Content-Type": "application/json"
}

def add_dns_record(record_type, name, content, ttl=300, priority=None):
    """Add a DNS record via Cloudflare API."""
    payload = {
        "type": record_type,
        "name": name,
        "content": content,
        "ttl": ttl
    }
    if priority is not None:
        payload["priority"] = priority
    
    response = requests.post(
        f"{CLOUDFLARE_API}/zones/{ZONE_ID}/dns_records",
        headers=headers,
        json=payload
    )
    result = response.json()
    if result["success"]:
        print(f"✓ Created {record_type} record: {name} → {content}")
    else:
        print(f"✗ Error: {result['errors']}")
    return result

# Examples
add_dns_record("A", "api.example.com", "203.0.113.50")
add_dns_record("CNAME", "www.example.com", "example.com")
add_dns_record("MX", "example.com", "mail.example.com", priority=10)
add_dns_record("TXT", "example.com", "v=spf1 include:_spf.google.com ~all")
```

### Java — DNS Record Lookup

```java
import javax.naming.directory.*;
import javax.naming.NamingEnumeration;
import javax.naming.NamingException;
import java.util.Hashtable;

public class DnsRecordLookup {
    
    public static void main(String[] args) {
        String domain = "google.com";
        String[] recordTypes = {"A", "AAAA", "CNAME", "MX", "TXT", "NS", "SOA"};
        
        for (String type : recordTypes) {
            queryRecord(domain, type);
        }
    }
    
    static void queryRecord(String domain, String recordType) {
        try {
            // Configure JNDI to use DNS
            Hashtable<String, String> env = new Hashtable<>();
            env.put("java.naming.factory.initial", 
                    "com.sun.jndi.dns.DnsContextFactory");
            env.put("java.naming.provider.url", "dns://8.8.8.8");
            
            DirContext ctx = new InitialDirContext(env);
            Attributes attrs = ctx.getAttributes(domain, new String[]{recordType});
            
            Attribute attr = attrs.get(recordType);
            if (attr != null) {
                System.out.println("\n" + recordType + " records for " + domain + ":");
                NamingEnumeration<?> values = attr.getAll();
                while (values.hasMore()) {
                    System.out.println("  → " + values.next());
                }
            }
            ctx.close();
        } catch (NamingException e) {
            System.out.println(recordType + ": " + e.getMessage());
        }
    }
}
```

### Java — DNS Record Management with AWS SDK (Route 53)

```java
import software.amazon.awssdk.services.route53.Route53Client;
import software.amazon.awssdk.services.route53.model.*;

public class Route53DnsManager {

    private final Route53Client route53;
    private final String hostedZoneId;

    public Route53DnsManager(String hostedZoneId) {
        this.route53 = Route53Client.create();
        this.hostedZoneId = hostedZoneId;
    }

    public void createARecord(String name, String ip, long ttl) {
        // Build the change request for Route 53
        ResourceRecordSet recordSet = ResourceRecordSet.builder()
            .name(name)
            .type(RRType.A)
            .ttl(ttl)
            .resourceRecords(ResourceRecord.builder().value(ip).build())
            .build();

        ChangeBatch changeBatch = ChangeBatch.builder()
            .changes(Change.builder()
                .action(ChangeAction.UPSERT)  // Create or update
                .resourceRecordSet(recordSet)
                .build())
            .build();

        ChangeResourceRecordSetsRequest request = 
            ChangeResourceRecordSetsRequest.builder()
                .hostedZoneId(hostedZoneId)
                .changeBatch(changeBatch)
                .build();

        route53.changeResourceRecordSets(request);
        System.out.println("Created A record: " + name + " → " + ip);
    }
}
```

---

## Infrastructure Examples

### BIND Zone File (Most Popular DNS Server Software)

```bash
# /etc/bind/zones/db.example.com
# This is a production-ready zone file for BIND9

$ORIGIN example.com.
$TTL 300

@   IN  SOA  ns1.example.com. admin.example.com. (
        2024061501  ; Serial (YYYYMMDDNN format)
        3600        ; Refresh - 1 hour
        900         ; Retry - 15 minutes
        604800      ; Expire - 1 week
        86400       ; Min TTL - 1 day (negative cache)
)

; Name Servers
@           IN  NS      ns1.example.com.
@           IN  NS      ns2.example.com.
ns1         IN  A       198.51.100.1
ns2         IN  A       198.51.100.2

; Web Servers
@           IN  A       203.0.113.10
@           IN  A       203.0.113.11       ; Round-robin
@           IN  AAAA    2001:db8::10
www         IN  CNAME   example.com.

; Application subdomains
api         IN  A       203.0.113.20
staging     IN  A       203.0.113.30
cdn         IN  CNAME   d1234.cloudfront.net.

; Email
@           IN  MX  10  mail.example.com.
@           IN  MX  20  mail-backup.example.com.
mail        IN  A       203.0.113.40

; Security & Verification
@           IN  TXT     "v=spf1 ip4:203.0.113.40 include:_spf.google.com -all"
@           IN  CAA     0 issue "letsencrypt.org"
_dmarc      IN  TXT     "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"
```

### Terraform — Managing DNS Records as Code

```hcl
# AWS Route 53 DNS records managed with Terraform

resource "aws_route53_zone" "main" {
  name = "example.com"
}

resource "aws_route53_record" "root_a" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "A"
  
  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "www_cname" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "CNAME"
  ttl     = 300
  records = ["example.com"]
}

resource "aws_route53_record" "mx" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "MX"
  ttl     = 3600
  records = [
    "1 aspmx.l.google.com",
    "5 alt1.aspmx.l.google.com",
    "5 alt2.aspmx.l.google.com",
  ]
}
```

---

## Real-World Example

### How Cloudflare Manages DNS at Scale

Cloudflare handles **DNS queries for over 25 million domains** and answers queries in under 11 milliseconds on average globally.

```
┌─────────────────────────────────────────────────────────────────┐
│            Cloudflare DNS Architecture                            │
│                                                                 │
│  User Query                                                     │
│      │                                                          │
│      ▼                                                          │
│  Anycast Network (300+ data centers worldwide)                  │
│      │                                                          │
│      ▼                                                          │
│  Edge DNS Server (closest to user via BGP Anycast)              │
│      │                                                          │
│      ├── Cache Hit? → Return immediately (< 1ms)                │
│      │                                                          │
│      ├── Cache Miss? → Query authoritative + cache result       │
│      │                                                          │
│      ▼                                                          │
│  Zone stored in distributed KV store (replicated globally)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How they do it:**
- **Anycast routing**: Same IP address announced from 300+ locations. Your packet goes to the nearest one automatically.
- **Flat file format**: Zone data stored in a custom flat format optimized for fast lookups (not BIND format).
- **No BIND**: They use custom-built DNS software written in Go for performance.
- **DNSSEC signing**: Automatic signing of all records with ECDSA P-256 keys.

### How AWS Route 53 Uses Record Types

Amazon's Route 53 uses DNS records creatively:
- **Alias records** (proprietary): Point root domain to ALB/CloudFront without CNAME limitations
- **Health-checked A records**: Automatically remove unhealthy IPs
- **Weighted routing**: Split traffic via multiple A records with weights
- **Geolocation routing**: Return different A records based on querier's location

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| CNAME at zone apex | DNS resolution fails | Use ALIAS/ANAME or A record |
| Missing trailing dot in zone files | Name treated as relative | Always use `example.com.` (with dot) |
| TTL too low (< 60s) | Excessive DNS queries, higher costs | Use 300s minimum for production |
| TTL too high (> 86400s) | Changes take forever to propagate | Use 300-3600s for records that may change |
| No SPF/DKIM/DMARC | Email goes to spam | Add TXT records for email auth |
| Single NS record | Single point of failure | Always have 2+ NS records |
| Forgetting PTR record | Email rejected by recipients | Set up reverse DNS with your ISP |
| CNAME chain too deep | Slow resolution, potential loops | Max 1 CNAME hop |

---

## When to Use / When NOT to Use

### Use A/AAAA Records When:
- ✅ Pointing a domain directly to a server you control
- ✅ You need the fastest possible resolution (no extra hops)
- ✅ At the zone apex (root domain)

### Use CNAME Records When:
- ✅ Pointing to a cloud service (ALB, CloudFront, Heroku)
- ✅ You want changes in the target to automatically reflect
- ❌ NOT at the zone apex
- ❌ NOT when you need other records at the same name

### Use MX Records When:
- ✅ Setting up email for your domain
- ✅ You want email failover (multiple MX with priorities)

### Use TXT Records When:
- ✅ Email security (SPF, DKIM, DMARC)
- ✅ Domain ownership verification
- ✅ Any metadata that needs to be publicly queryable

---

## Key Takeaways

- **A records** map domains to IPv4 addresses — the most fundamental DNS record type.
- **AAAA records** are the IPv6 equivalent — increasingly important as IPv4 addresses are exhausted.
- **CNAME records** create aliases but CANNOT exist at the zone apex or alongside other records.
- **MX records** route email with priority-based failover (lower number = higher priority).
- **TXT records** store verification and security data (SPF, DKIM, DMARC are critical for email deliverability).
- **NS records** delegate authority — changing them means switching DNS providers.
- **Each record type has specific rules** — violating them causes silent failures that are hard to debug.

---

## What's Next?

Next, we'll trace exactly what happens when your computer resolves a domain name from start to finish — from your browser cache all the way to the root DNS servers and back. See [02-dns-resolution-flow.md](./02-dns-resolution-flow.md).
