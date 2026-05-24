# Chapter 9: Cloud DNS (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud DNS Fundamentals](#part-1-cloud-dns-fundamentals)
- [Part 2: Private DNS Zones](#part-2-private-dns-zones)
- [Part 3: DNS Policies](#part-3-dns-policies)
- [Part 4: DNS Peering](#part-4-dns-peering)
- [Part 5: Response Policies](#part-5-response-policies)
- [Part 6: DNSSEC](#part-6-dnssec)
- [Part 7: Cloud Domains (Domain Registration)](#part-7-cloud-domains-domain-registration)
- [Part 8: Terraform Example](#part-8-terraform-example)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud DNS is a high-performance, resilient, global DNS service that publishes your domain names using Google's worldwide Anycast network. It supports public and private zones, DNSSEC, DNS policies, and peering.

```
What you'll learn:
├── Cloud DNS Components
│   ├── Managed Zones (public & private)
│   ├── Record Sets (A, AAAA, CNAME, MX, TXT, etc.)
│   └── All fields explained
├── Private DNS Zones
│   ├── VPC binding
│   ├── Cross-project sharing
│   └── Internal service discovery
├── DNSSEC
├── DNS Policies
│   ├── Inbound forwarding
│   ├── Outbound forwarding
│   └── Alternative name servers
├── DNS Peering
├── Response Policies
├── Domain Registration (Cloud Domains)
└── Real-world patterns
```

---

## Part 1: Cloud DNS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CLOUD DNS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed, authoritative DNS service with 100% uptime SLA.     │
│                                                                       │
│ Key facts:                                                           │
│ ├── Uses Google's global Anycast network                           │
│ ├── 100% uptime SLA (same as AWS Route 53)                        │
│ ├── Supports public and private zones                              │
│ ├── DNSSEC support                                                  │
│ ├── DNS forwarding (hybrid with on-prem)                           │
│ ├── Response policies (custom responses)                           │
│ └── Programmable with API/CLI/Terraform                            │
│                                                                       │
│ Cloud DNS vs Route 53:                                               │
│ ┌──────────────────────┬──────────────────┬────────────────────┐   │
│ │ Feature              │ Route 53         │ Cloud DNS          │   │
│ │ SLA                  │ 100%             │ 100%               │   │
│ │ Zone types           │ Public, Private  │ Public, Private    │   │
│ │ ALIAS record         │ Yes (AWS unique) │ No (use A/CNAME)  │   │
│ │ Routing policies     │ 7 types          │ Geo, Weighted, Failover│ │
│ │ Health checks        │ Built-in         │ Yes (with routing) │   │
│ │ Domain registration  │ Built-in         │ Cloud Domains      │   │
│ │ DNSSEC               │ Yes              │ Yes                │   │
│ │ DNS forwarding       │ Resolver         │ DNS policies       │   │
│ │ Response policies    │ No               │ Yes                │   │
│ │ Split-horizon        │ Yes              │ Yes                │   │
│ │ Pricing              │ $0.50/zone       │ $0.20/zone         │   │
│ └──────────────────────┴──────────────────┴────────────────────┘   │
│                                                                       │
│ ✅ GCP Cloud DNS NOW supports DNS routing policies!               │
│    Available routing types:                                        │
│    ├── Weighted Round Robin (split traffic by weight)             │
│    ├── Geolocation (route by user's location)                     │
│    ├── Failover (primary/backup with health checks)               │
│    Health checks can be attached to routing policy records!        │
│    Use Global LB for more advanced traffic management.            │
│                                                                       │
│ Pricing:                                                             │
│ ├── Managed zone: $0.20/month per zone (cheaper than AWS!)        │
│ ├── Queries: $0.40/million (first billion, then cheaper)          │
│ ├── DNSSEC: No extra charge                                        │
│ └── DNS policies: $0.20/month + forwarding queries $0.40/million  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Managed Zones

```
┌─────────────────────────────────────────────────────────────────────┐
│                 MANAGED ZONES                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. PUBLIC MANAGED ZONE                                               │
│    ├── Accessible from the internet                                │
│    ├── Authoritative DNS for your domain                           │
│    ├── Gets Google's Anycast name servers                          │
│    └── Used for: websites, APIs, public services                   │
│                                                                       │
│ 2. PRIVATE MANAGED ZONE                                              │
│    ├── Only accessible from authorized VPC networks                │
│    ├── Used for: internal services, databases                      │
│    ├── Must bind to one or more VPCs                               │
│    └── Example: db.internal.techcorp.com → 10.0.3.5               │
│                                                                       │
│ 3. FORWARDING ZONE                                                   │
│    ├── Forwards queries to another DNS server                      │
│    ├── Used for: hybrid DNS with on-premises                      │
│    └── Queries for corp.local → forward to 192.168.1.53           │
│                                                                       │
│ 4. PEERING ZONE                                                      │
│    ├── Forwards queries to another VPC's DNS                       │
│    ├── Used for: cross-VPC/cross-project DNS resolution           │
│    └── Queries in VPC-A resolved by VPC-B's private zones         │
│                                                                       │
│ Split-horizon DNS:                                                   │
│ ├── Public zone: api.techcorp.com → External LB IP               │
│ └── Private zone: api.techcorp.com → Internal LB IP              │
│     VPC queries resolve to private, internet queries to public    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Managed Zone

```
Console → Network services → Cloud DNS → Create zone

┌─────────────────────────────────────────────────────────────────┐
│             CREATE DNS ZONE                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Zone type:     ● Public  ○ Private  ○ Forwarding  ○ Peering   │
│                                                                   │
│ Zone name:     [techcorp-com]  (identifier, not domain)        │
│ DNS name:      [techcorp.com]                                   │
│ Description:   [Production DNS for techcorp.com]               │
│                                                                   │
│ DNSSEC:        ○ Off  ● On  ○ Transfer                        │
│                                                                   │
│ Cloud Logging: ☐ Enable (logs all DNS queries, costs $$)       │
│                                                                   │
│ (If Private):                                                    │
│   Networks:    [prod-vpc ▼] [Add network]                      │
│                [shared-vpc ▼]                                   │
│                                                                   │
│ (If Forwarding):                                                 │
│   Forwarding targets:                                           │
│   [192.168.1.53] (on-prem DNS)                                 │
│   [192.168.1.54] (on-prem DNS backup)                          │
│   Forwarding path: ○ Default  ● Private (via VPN/Interconnect)│
│                                                                   │
│ [Create]                                                         │
│                                                                   │
│ After creation (public):                                        │
│ You get Google's name servers:                                  │
│   ns-cloud-a1.googledomains.com                                │
│   ns-cloud-a2.googledomains.com                                │
│   ns-cloud-a3.googledomains.com                                │
│   ns-cloud-a4.googledomains.com                                │
│                                                                   │
│ ⚠️ Update NS at your registrar to these Google NS!              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Public zone
  gcloud dns managed-zones create techcorp-com \
    --dns-name=techcorp.com. \
    --description="Production DNS" \
    --dnssec-state=on \
    --visibility=public

  # Private zone
  gcloud dns managed-zones create internal-techcorp \
    --dns-name=internal.techcorp.com. \
    --description="Internal services DNS" \
    --visibility=private \
    --networks=prod-vpc,shared-vpc

  # Forwarding zone
  gcloud dns managed-zones create corp-local \
    --dns-name=corp.local. \
    --description="Forward to on-prem DNS" \
    --visibility=private \
    --networks=prod-vpc \
    --forwarding-targets=192.168.1.53,192.168.1.54 \
    --private-forwarding-targets=192.168.1.53,192.168.1.54
```

### Record Sets

```
┌─────────────────────────────────────────────────────────────────────┐
│                  RECORD SETS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Supported record types:                                              │
│ ├── A: Domain → IPv4                                               │
│ ├── AAAA: Domain → IPv6                                            │
│ ├── CNAME: Domain → another domain                                 │
│ │   ⚠️ Cannot be at zone apex (same as AWS/Azure)                 │
│ │   ⚠️ GCP has NO ALIAS record type (unlike AWS)                  │
│ │   For apex: Use A record pointing to LB static IP               │
│ ├── MX: Mail exchange                                              │
│ ├── TXT: Text (SPF, DKIM, verification)                           │
│ ├── NS: Name servers                                                │
│ ├── SOA: Start of Authority                                         │
│ ├── SRV: Service location                                           │
│ ├── CAA: Certificate Authority Authorization                       │
│ ├── PTR: Reverse DNS                                                │
│ ├── SPF: (deprecated, use TXT instead)                             │
│ └── NAPTR: Naming Authority Pointer                                 │
│                                                                       │
│ ⚠️ No ALIAS/ANAME record in GCP!                                    │
│    Workarounds for zone apex:                                       │
│    ├── Use A record with Global LB's static IP                     │
│    │   (GCP Global LBs have static IPs, unlike AWS ALB)           │
│    ├── Google provides a single Anycast IP for global LBs          │
│    └── Much simpler than AWS ALIAS!                                │
│                                                                       │
│ GCP vs AWS for zone apex:                                            │
│ ├── AWS: ALB has no static IP → need ALIAS record                 │
│ └── GCP: Global LB has static IP → just use A record!             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Records

```
Console → Cloud DNS → Zone → Add standard

┌─────────────────────────────────────────────────────────────────┐
│              CREATE RECORD SET                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ DNS name:       [api].techcorp.com.                             │
│ Resource record type: [A ▼]                                    │
│ TTL:            [300]                                           │
│ TTL unit:       [seconds ▼]                                    │
│ Routing policy: [None ▼]                                       │
│                                                                   │
│ IPv4 address:                                                    │
│   [34.120.100.50]                                               │
│   [Add item] (for multiple IPs = round-robin)                  │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
│ More examples:                                                   │
│                                                                   │
│ CNAME: blog.techcorp.com → techcorp.wordpress.com              │
│ MX: techcorp.com → 10 mail.techcorp.com                        │
│ TXT: techcorp.com → "v=spf1 include:_spf.google.com ~all"     │
│ CAA: techcorp.com → 0 issue "letsencrypt.org"                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Start a transaction (batch record changes)
  gcloud dns record-sets transaction start \
    --zone=techcorp-com

  # Add A record
  gcloud dns record-sets transaction add 34.120.100.50 \
    --name=api.techcorp.com. \
    --type=A \
    --ttl=300 \
    --zone=techcorp-com

  # Add CNAME
  gcloud dns record-sets transaction add techcorp.wordpress.com. \
    --name=blog.techcorp.com. \
    --type=CNAME \
    --ttl=300 \
    --zone=techcorp-com

  # Add MX
  gcloud dns record-sets transaction add \
    "10 mail.techcorp.com." \
    "20 mail2.techcorp.com." \
    --name=techcorp.com. \
    --type=MX \
    --ttl=3600 \
    --zone=techcorp-com

  # Add TXT (SPF)
  gcloud dns record-sets transaction add \
    '"v=spf1 include:_spf.google.com ~all"' \
    --name=techcorp.com. \
    --type=TXT \
    --ttl=3600 \
    --zone=techcorp-com

  # Execute all changes
  gcloud dns record-sets transaction execute \
    --zone=techcorp-com

  # Or add single record without transaction
  gcloud dns record-sets create api.techcorp.com. \
    --zone=techcorp-com \
    --type=A \
    --ttl=300 \
    --rrdatas=34.120.100.50
```

---

## Part 2: Private DNS Zones

```
┌─────────────────────────────────────────────────────────────────────┐
│              PRIVATE DNS ZONES                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: DNS zones visible only within bound VPC networks.             │
│                                                                       │
│ Use cases:                                                           │
│ ├── Internal service names: db.internal.techcorp.com               │
│ ├── Microservice discovery: user-svc.internal.techcorp.com         │
│ ├── Environment separation: db.staging.internal.techcorp.com       │
│ └── Private Google API endpoints                                   │
│                                                                       │
│ How resolution works:                                                │
│                                                                       │
│ VM queries "db.internal.techcorp.com"                               │
│    │                                                                  │
│    ▼                                                                  │
│ [GCP Internal DNS (169.254.169.254)]                                │
│    │ Is there a private zone for internal.techcorp.com?             │
│    │ Yes, and this VPC is authorized!                               │
│    ▼                                                                  │
│ [Private Zone: internal.techcorp.com]                               │
│    │ db.internal.techcorp.com → 10.0.3.5                           │
│    ▼                                                                  │
│ Response: 10.0.3.5                                                   │
│                                                                       │
│ Cross-project private zones:                                         │
│ ├── Create private zone in host project (Shared VPC)               │
│ ├── Bind to Shared VPC network                                     │
│ ├── All service projects using Shared VPC can resolve              │
│ └── Centralized DNS management!                                    │
│                                                                       │
│ Cross-VPC (without Shared VPC):                                      │
│ ├── Use DNS Peering zone                                           │
│ ├── VPC-A queries forwarded to VPC-B for resolution               │
│ └── Useful for hub-and-spoke DNS                                   │
│                                                                       │
│ Private zone for Google APIs:                                        │
│ Create private zone: googleapis.com                                 │
│ ├── *.googleapis.com CNAME restricted.googleapis.com               │
│ ├── restricted.googleapis.com A 199.36.153.4/30                   │
│ └── Forces all Google API traffic through restricted VIPs          │
│     (prevents data exfiltration via public Google APIs)            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: DNS Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DNS POLICIES                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Control DNS behavior for VPC networks —                       │
│       inbound forwarding, outbound forwarding, and alternative NS.  │
│                                                                       │
│ 1. INBOUND SERVER POLICY                                             │
│    ├── Allows on-premises DNS to query Cloud DNS                   │
│    ├── Creates inbound forwarder IPs in your VPC                   │
│    ├── On-prem forwards queries to these IPs                       │
│    ├── Similar to: AWS Route 53 Resolver Inbound Endpoint          │
│    └── On-prem can resolve GCP private zones!                      │
│                                                                       │
│    Create:                                                           │
│    gcloud dns policies create allow-inbound \                        │
│      --networks=prod-vpc \                                           │
│      --enable-inbound-forwarding \                                    │
│      --description="Allow on-prem to query Cloud DNS"               │
│                                                                       │
│    After creation: Gets IPs (one per subnet in the region).        │
│    Configure on-prem DNS to forward GCP domains to these IPs.      │
│                                                                       │
│ 2. OUTBOUND FORWARDING (via Forwarding Zone)                        │
│    ├── Forward specific domains to on-prem DNS                     │
│    ├── Configured as a forwarding zone (not a policy)              │
│    ├── Similar to: AWS Route 53 Resolver Outbound Endpoint         │
│    └── VPC can resolve on-prem domains!                            │
│                                                                       │
│ 3. ALTERNATIVE NAME SERVERS                                          │
│    ├── Override the default DNS for a VPC                          │
│    ├── All queries go to your custom DNS first                     │
│    ├── Use case: Custom DNS appliance, Pi-hole, etc.               │
│    └── gcloud dns policies create custom-dns \                      │
│          --networks=dev-vpc \                                        │
│          --alternative-name-servers=10.0.1.53,10.0.1.54            │
│                                                                       │
│ Pricing:                                                             │
│ ├── Inbound policy: $0.20/month + forwarded queries               │
│ ├── Forwarding zone: $0.20/month per zone                          │
│ └── Queries: $0.40/million                                          │
│                                                                       │
│ ⚡ Much cheaper than AWS Route 53 Resolver!                          │
│    AWS: ~$180/month per endpoint                                    │
│    GCP: ~$0.40/month (zone + policy)                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: DNS Peering

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DNS PEERING                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Forward DNS queries from one VPC to another VPC's DNS.        │
│       Allows cross-VPC DNS resolution without exposing zones.       │
│                                                                       │
│ ┌──────────────────┐    Peering     ┌──────────────────┐            │
│ │ Consumer VPC     │    Zone        │ Producer VPC     │            │
│ │ (dev-vpc)         │ ──────────►  │ (shared-vpc)      │            │
│ │                    │               │                    │            │
│ │ "Where is         │               │ Has private zone:  │            │
│ │  db.internal.     │               │ internal.techcorp  │            │
│ │  techcorp.com?"   │               │ .com               │            │
│ │                    │               │ db → 10.0.3.5      │            │
│ └──────────────────┘               └──────────────────┘            │
│                                                                       │
│ Use cases:                                                           │
│ ├── Hub-and-spoke DNS: Shared VPC has private zones,              │
│ │   other VPCs peer to resolve internal names                      │
│ ├── Cross-project DNS: Team A's VPC resolves Team B's records     │
│ └── Centralized DNS management with decentralized VPCs            │
│                                                                       │
│ ⚠️ DNS peering is NOT transitive!                                   │
│    VPC-A peers to VPC-B, VPC-B peers to VPC-C                     │
│    VPC-A CANNOT resolve VPC-C's zones through VPC-B               │
│                                                                       │
│ Create:                                                              │
│ gcloud dns managed-zones create peer-to-shared \                     │
│   --dns-name=internal.techcorp.com. \                                 │
│   --description="Peer to shared VPC DNS" \                            │
│   --visibility=private \                                              │
│   --networks=dev-vpc \                                                │
│   --target-network=shared-vpc \                                       │
│   --target-project=tc-shared-networking                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Response Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│              RESPONSE POLICIES                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Override DNS responses for specific queries.                  │
│       A GCP-unique feature (no direct AWS equivalent).             │
│                                                                       │
│ Use cases:                                                           │
│ ├── Block malicious domains (return NXDOMAIN)                      │
│ ├── DNS sinkholing (redirect malware domains to safe IP)          │
│ ├── Override external domains with internal IPs                    │
│ ├── A/B testing at DNS level                                       │
│ └── DNS-based content filtering                                    │
│                                                                       │
│ Actions:                                                             │
│ ├── Local data: Return custom response                             │
│ │   malware.example.com → 0.0.0.0 (block)                        │
│ │   api.partner.com → 10.0.5.10 (redirect to internal proxy)     │
│ └── NXDOMAIN: Return "domain not found"                            │
│                                                                       │
│ Create:                                                              │
│ gcloud dns response-policies create block-malware \                  │
│   --networks=prod-vpc \                                               │
│   --description="Block known malware domains"                        │
│                                                                       │
│ gcloud dns response-policies rules create block-evil \               │
│   --response-policy=block-malware \                                   │
│   --dns-name=evil-malware.example.com. \                              │
│   --behavior=behaviorUnspecified \                                    │
│   --local-data="evil-malware.example.com.,A,300,0.0.0.0"           │
│                                                                       │
│ # Or return NXDOMAIN (not found)                                    │
│ gcloud dns response-policies rules create block-phishing \           │
│   --response-policy=block-malware \                                   │
│   --dns-name=phishing.example.com. \                                  │
│   --behavior=behaviorUnspecified                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: DNSSEC

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DNSSEC                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Cryptographically sign DNS records to prevent spoofing.       │
│                                                                       │
│ Enable during zone creation:                                         │
│   DNSSEC: On                                                        │
│                                                                       │
│ Or enable on existing zone:                                          │
│ gcloud dns managed-zones update techcorp-com \                       │
│   --dnssec-state=on                                                   │
│                                                                       │
│ After enabling:                                                      │
│ 1. GCP generates DNSKEY records (KSK + ZSK)                       │
│ 2. Get the DS record:                                               │
│    gcloud dns dns-keys list --zone=techcorp-com                     │
│ 3. Add DS record at your domain registrar                          │
│ 4. Chain of trust established: Root → .com → techcorp.com         │
│                                                                       │
│ ⚡ GCP makes DNSSEC simpler than AWS!                                │
│    AWS requires KMS key for KSK. GCP manages keys automatically.  │
│                                                                       │
│ Verify: dig techcorp.com +dnssec                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Cloud Domains (Domain Registration)

```
┌─────────────────────────────────────────────────────────────────────┐
│              CLOUD DOMAINS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Register and manage domains directly in GCP.                  │
│                                                                       │
│ Console → Network services → Cloud Domains → Register domain       │
│   1. Search for domain                                              │
│   2. Select TLD (.com, .dev, .app, etc.)                           │
│   3. Fill contact info                                              │
│   4. Privacy protection: ☑ (free!)                                 │
│   5. Auto-renew: ☑                                                 │
│   6. DNS: Use Cloud DNS (auto-creates zone)                       │
│   7. DNSSEC: ☑ Enable                                              │
│                                                                       │
│ Pricing:                                                             │
│ ├── .com: $12/year                                                 │
│ ├── .dev: $12/year                                                 │
│ ├── .app: $14/year                                                 │
│ ├── .io: $20/year                                                  │
│ └── .in: $12/year                                                  │
│                                                                       │
│ CLI:                                                                 │
│ gcloud domains registrations register techcorp.com \                 │
│   --contact-data-from-file=contacts.yaml \                           │
│   --contact-privacy=private-contact-data \                           │
│   --cloud-dns-zone=techcorp-com \                                    │
│   --yearly-price="12.00 USD"                                         │
│                                                                       │
│ ⚡ Like AWS, registering with GCP auto-configures DNS.              │
│    Can also use external registrar and point NS to Cloud DNS.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform Example

```hcl
# Public zone
resource "google_dns_managed_zone" "public" {
  name        = "techcorp-com"
  dns_name    = "techcorp.com."
  description = "Production DNS"

  dnssec_config {
    state = "on"
  }
}

# Private zone
resource "google_dns_managed_zone" "private" {
  name        = "internal-techcorp"
  dns_name    = "internal.techcorp.com."
  description = "Internal services DNS"
  visibility  = "private"

  private_visibility_config {
    networks {
      network_url = google_compute_network.prod_vpc.id
    }
    networks {
      network_url = google_compute_network.shared_vpc.id
    }
  }
}

# A record (zone apex → Global LB static IP)
resource "google_dns_record_set" "apex" {
  name         = "techcorp.com."
  type         = "A"
  ttl          = 300
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = [google_compute_global_address.lb.address]
}

# A record (subdomain)
resource "google_dns_record_set" "api" {
  name         = "api.techcorp.com."
  type         = "A"
  ttl          = 300
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = [google_compute_global_address.api_lb.address]
}

# CNAME record
resource "google_dns_record_set" "blog" {
  name         = "blog.techcorp.com."
  type         = "CNAME"
  ttl          = 300
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = ["techcorp.wordpress.com."]
}

# MX records
resource "google_dns_record_set" "mail" {
  name         = "techcorp.com."
  type         = "MX"
  ttl          = 3600
  managed_zone = google_dns_managed_zone.public.name
  rrdatas = [
    "10 mail.techcorp.com.",
    "20 mail2.techcorp.com.",
  ]
}

# TXT record (SPF)
resource "google_dns_record_set" "spf" {
  name         = "techcorp.com."
  type         = "TXT"
  ttl          = 3600
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = ["\"v=spf1 include:_spf.google.com ~all\""]
}

# Private zone record
resource "google_dns_record_set" "db_internal" {
  name         = "db.internal.techcorp.com."
  type         = "A"
  ttl          = 60
  managed_zone = google_dns_managed_zone.private.name
  rrdatas      = ["10.0.3.5"]
}

# Forwarding zone (on-prem)
resource "google_dns_managed_zone" "corp_local" {
  name        = "corp-local"
  dns_name    = "corp.local."
  description = "Forward to on-prem DNS"
  visibility  = "private"

  private_visibility_config {
    networks {
      network_url = google_compute_network.prod_vpc.id
    }
  }

  forwarding_config {
    target_name_servers {
      ipv4_address    = "192.168.1.53"
      forwarding_path = "private"
    }
    target_name_servers {
      ipv4_address    = "192.168.1.54"
      forwarding_path = "private"
    }
  }
}

# DNS policy (inbound forwarding)
resource "google_dns_policy" "allow_inbound" {
  name                      = "allow-inbound"
  enable_inbound_forwarding = true

  networks {
    network_url = google_compute_network.prod_vpc.id
  }
}
```

---

## Part 9a: DNS Routing Policies

> **In Simple Terms:** DNS routing policies let you control WHERE your users go based on rules — like their location, or whether your server is healthy. Think of it like a traffic police officer directing cars to different roads based on conditions.

### Available Routing Policy Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                   DNS ROUTING POLICIES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. WEIGHTED ROUND ROBIN (WRR)                                       │
│     Split traffic across multiple targets by percentage              │
│     Example: 70% → primary server, 30% → canary server             │
│                                                                       │
│  2. GEOLOCATION                                                      │
│     Route users to the nearest server based on their location        │
│     Example: India users → asia-south1, US users → us-central1      │
│                                                                       │
│  3. FAILOVER                                                         │
│     Route to backup when primary is unhealthy                        │
│     Example: Primary → prod-server, Backup → dr-server              │
│     Requires health check attached to primary                       │
│                                                                       │
│  All policies support HEALTH CHECKS:                                 │
│  ├── Attach health checks to routing policy records                 │
│  ├── Unhealthy targets are automatically excluded                   │
│  └── Works with Compute Engine health checks                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Routing Policies

```
Console → Cloud DNS → Zone → Add Record Set

For Weighted Round Robin:
┌─────────────────────────────────────────────────────────┐
│ DNS name:     app.example.com                            │
│ Resource record type: A                                  │
│ Routing policy: Weighted Round Robin                     │
│                                                         │
│ Target 1:                                               │
│   Weight: 70                                            │
│   IPv4 address: 34.93.xx.xx (primary server)           │
│   Health check: prod-health-check                      │
│                                                         │
│ Target 2:                                               │
│   Weight: 30                                            │
│   IPv4 address: 35.200.xx.xx (canary server)           │
│   Health check: canary-health-check                    │
│                                                         │
│ [CREATE]                                                │
└─────────────────────────────────────────────────────────┘

For Geolocation:
┌─────────────────────────────────────────────────────────┐
│ DNS name:     api.example.com                            │
│ Resource record type: A                                  │
│ Routing policy: Geolocation                              │
│                                                         │
│ Policy item 1:                                          │
│   Source region: asia-south1                            │
│   IPv4 address: 10.0.1.5 (India server)               │
│                                                         │
│ Policy item 2:                                          │
│   Source region: us-central1                            │
│   IPv4 address: 10.0.2.5 (US server)                  │
│                                                         │
│ [CREATE]                                                │
└─────────────────────────────────────────────────────────┘

CLI — Weighted Round Robin:
  gcloud dns record-sets create app.example.com \
    --zone=my-zone \
    --type=A \
    --routing-policy-type=WRR \
    --routing-policy-data="0.7=34.93.xx.xx;0.3=35.200.xx.xx"

CLI — Geolocation:
  gcloud dns record-sets create api.example.com \
    --zone=my-zone \
    --type=A \
    --routing-policy-type=GEO \
    --routing-policy-data="asia-south1=10.0.1.5;us-central1=10.0.2.5"

CLI — Failover:
  gcloud dns record-sets create app.example.com \
    --zone=my-zone \
    --type=A \
    --routing-policy-type=FAILOVER \
    --routing-policy-primary-data="34.93.xx.xx" \
    --routing-policy-backup-data-type=GEO \
    --routing-policy-backup-data="asia-south1=35.200.xx.xx" \
    --backup-data-trickle-ratio=0.1
```

---

## Part 9b: Console Walkthrough — Managing & Deleting DNS Resources

### Updating DNS Records

```
Console → Network services → Cloud DNS → Click zone name

To Edit a Record:
┌─────────────────────────────────────────────────────────┐
│ 1. Click on the record you want to edit                 │
│ 2. Click "Edit" (pencil icon)                          │
│ 3. Modify the IP address, TTL, or routing policy       │
│ 4. Click "Save"                                        │
│                                                         │
│ ⚠️ Changes propagate based on TTL                       │
│    Old TTL=300 → takes up to 5 min for full update     │
└─────────────────────────────────────────────────────────┘

CLI:
  # Start a transaction (atomic update)
  gcloud dns record-sets transaction start --zone=my-zone
  
  # Remove old record
  gcloud dns record-sets transaction remove 1.2.3.4 \
    --name=app.example.com. --type=A --ttl=300 --zone=my-zone
  
  # Add new record
  gcloud dns record-sets transaction add 5.6.7.8 \
    --name=app.example.com. --type=A --ttl=300 --zone=my-zone
  
  # Execute the change
  gcloud dns record-sets transaction execute --zone=my-zone
```

### Deleting DNS Records

```
Console → Cloud DNS → Click zone → Select record → Delete

CLI:
  gcloud dns record-sets delete app.example.com \
    --zone=my-zone --type=A

  # Or via transaction:
  gcloud dns record-sets transaction start --zone=my-zone
  gcloud dns record-sets transaction remove 1.2.3.4 \
    --name=app.example.com. --type=A --ttl=300 --zone=my-zone
  gcloud dns record-sets transaction execute --zone=my-zone
```

### Deleting a DNS Zone

```
⚠️ You must delete ALL records (except NS and SOA) before deleting a zone!

Step 1: Delete all custom records
  Console → Cloud DNS → Click zone → Select records → Delete
  
  CLI: 
  gcloud dns record-sets list --zone=my-zone
  # Delete each record (NS and SOA are auto-managed)

Step 2: Delete the zone
  Console → Cloud DNS → Select zone checkbox → Delete
  
  CLI:
  gcloud dns managed-zones delete my-zone

  ⚠️ Cannot undo! All records in the zone are permanently deleted.
```

---

## Part 9c: Troubleshooting DNS Issues

```
Common Issues & How to Fix:
━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. "DNS record not resolving"
   ─────────────────────────
   # Check from Cloud Shell:
   dig app.example.com
   nslookup app.example.com
   
   # Check Cloud DNS name servers:
   gcloud dns managed-zones describe my-zone \
     --format="value(nameServers)"
   
   # Verify domain registrar points to Cloud DNS name servers
   # Common fix: Update NS records at your registrar

2. "Private DNS not working for VMs"
   ─────────────────────────────────
   # Check if zone is bound to the correct VPC:
   gcloud dns managed-zones describe my-private-zone
   
   # Verify VM is in the correct VPC
   # Check if Private Google Access is enabled on subnet

3. "DNS changes not propagating"
   ─────────────────────────────
   # Check TTL of the record — old cached values take TTL seconds
   dig app.example.com +short
   
   # Force query to Cloud DNS directly:
   dig @ns-cloud-a1.googledomains.com app.example.com
   
   # Reduce TTL before making changes (set to 60 seconds)
   # Wait for old TTL to expire, then make the change

4. "DNSSEC validation failures"
   ────────────────────────────
   # Verify DS record at registrar matches Cloud DNS:
   gcloud dns managed-zones describe my-zone \
     --format="value(dnsSecConfig)"
   
   # Check with dnsviz.net or dig +dnssec

5. "Forwarding zone not resolving"
   ───────────────────────────────
   # Check if on-prem DNS server is reachable:
   # Verify VPN/Interconnect is UP
   # Check firewall rules allow DNS (port 53 TCP/UDP)
   # Verify forwarding targets are correct
```

---

## Part 9d: DNS Logging & Analysis

```
Enable DNS Logging:
  Console → Cloud DNS → Click zone → Edit → Toggle "Cloud Logging" ON
  
  CLI:
  gcloud dns managed-zones update my-zone --log-dns-queries

View DNS Logs:
  Console → Logging → Log Explorer
  
  Filter:
  resource.type="dns_query"
  resource.labels.target_name="my-zone"
  
  # See which domains are being queried
  # Identify suspicious DNS queries
  # Debug resolution failures
```

---

## Part 9: Real-World Patterns

### Startup

```
Zones: 1 public (techcorp.com)
Records:
├── techcorp.com A → Global LB static IP
├── api.techcorp.com A → Global LB
├── www.techcorp.com CNAME → techcorp.com
├── MX → Google Workspace
├── TXT → SPF, DKIM
└── blog.techcorp.com CNAME → blog platform

No private zones (small team, single VPC).
DNSSEC: On (free, easy to enable).
Cost: ~$1/month
```

### Mid-Size

```
Zones:
├── techcorp.com (public)
├── internal.techcorp.com (private, bound to Shared VPC)
└── staging.techcorp.com (public or private)

Private records:
├── db.internal → Cloud SQL private IP
├── cache.internal → Memorystore private IP
├── api.internal → Internal LB IP
└── *.staging.internal → staging backends

DNS peering: Dev VPC → Shared VPC (resolve internal names)
DNS policy: Not needed (no on-prem)
Cost: ~$3-5/month
```

### Enterprise

```
Zones:
├── techcorp.com (public, DNSSEC on)
├── internal.techcorp.com (private, Shared VPC)
├── staging.internal.techcorp.com (private)
├── dev.internal.techcorp.com (private)
├── corp.local (forwarding zone → on-prem DNS)
├── googleapis.com (private, restricted VIPs)
└── Per-BU zones

DNS policies:
├── Inbound forwarding (on-prem → Cloud DNS)
├── Forwarding zones (Cloud DNS → on-prem for corp.local)
└── Alternative NS for isolated dev VPCs

Response policies:
├── Block known malware domains
├── Redirect partner APIs to internal proxy
└── DNS-based content filtering

DNS peering: Spoke VPCs → Hub VPC (centralized DNS)
Cloud Logging: On for all public zones (audit)
Cost: ~$20-50/month
```

---

## Quick Reference

| Feature | Details |
|---------|---------|
| Zone cost | $0.20/month (cheaper than AWS $0.50) |
| Query cost | $0.40/million |
| ALIAS record | Not available (use A record + static LB IP) |
| Routing policies | Geo, Weighted Round Robin, Failover (with health checks) |
| Health checks | Yes (attached to routing policy records) |
| DNSSEC | Yes (simpler than AWS, auto-managed keys) |
| Private zones | Yes (bind to VPC networks) |
| Forwarding zones | Yes (hybrid DNS) |
| DNS peering | Yes (cross-VPC resolution) |
| Response policies | Yes (GCP-unique, block/redirect) |
| SLA | 100% |

---

## What's Next?

In the next chapter, we'll cover Cloud CDN — GCP's content delivery network for caching at Google's edge locations.

→ Next: [Chapter 10: Cloud CDN](10-cloud-cdn.md)

---

*Last Updated: May 2026*
