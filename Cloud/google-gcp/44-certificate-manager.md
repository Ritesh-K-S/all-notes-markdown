# Chapter 44 — Certificate Manager

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Certificate Manager Fundamentals](#part-1--certificate-manager-fundamentals)
- [Part 2: Google-Managed Certificates](#part-2--google-managed-certificates)
- [Part 3: Self-Managed Certificates](#part-3--self-managed-certificates)
- [Part 4: Certificate Maps & Map Entries](#part-4--certificate-maps--map-entries)
- [Part 5: DNS Authorization](#part-5--dns-authorization)
- [Part 6: Load Balancer Authorization](#part-6--load-balancer-authorization)
- [Part 7: Certificate Issuance Config & CA Service](#part-7--certificate-issuance-config--ca-service)
- [Part 8: Wildcard Certificates](#part-8--wildcard-certificates)
- [Part 9: Multi-Domain & SAN Certificates](#part-9--multi-domain--san-certificates)
- [Part 10: Certificate Lifecycle & Renewal](#part-10--certificate-lifecycle--renewal)
- [Part 11: Integration with Load Balancers](#part-11--integration-with-load-balancers)
- [Part 12: IAM & Security](#part-12--iam--security)
- [Part 13: Monitoring & Troubleshooting](#part-13--monitoring--troubleshooting)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Certificate Manager is Google Cloud's service for provisioning, managing, and deploying TLS/SSL certificates for use with Cloud Load Balancing. It supports Google-managed certificates (automatically provisioned and renewed) and self-managed certificates (you upload your own). Certificate Manager uses certificate maps to associate certificates with hostnames, enabling flexible multi-domain HTTPS configurations.

---

## Part 1 — Certificate Manager Fundamentals

### What Is Certificate Manager?

```
┌─────────────────────────────────────────────────────────────────┐
│              CERTIFICATE MANAGER OVERVIEW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Certificate Manager provides TLS certificates for              │
│  Google Cloud load balancers:                                    │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                                                           │   │
│  │  Internet                                                 │   │
│  │    │                                                      │   │
│  │    ▼ HTTPS (TLS)                                         │   │
│  │  ┌─────────────────────────────────────┐                │   │
│  │  │ Google Cloud Load Balancer           │                │   │
│  │  │ ┌─────────────────────────────────┐ │                │   │
│  │  │ │ Certificate Manager             │ │                │   │
│  │  │ │ ┌───────────────────────────┐   │ │                │   │
│  │  │ │ │ Certificate Map           │   │ │                │   │
│  │  │ │ │ ├── api.example.com → cert1│  │ │                │   │
│  │  │ │ │ ├── app.example.com → cert2│  │ │                │   │
│  │  │ │ │ └── *.staging.ex  → cert3 │   │ │                │   │
│  │  │ │ └───────────────────────────┘   │ │                │   │
│  │  │ └─────────────────────────────────┘ │                │   │
│  │  └─────────────────────────────────────┘                │   │
│  │    │                                                      │   │
│  │    ▼ HTTP (internal)                                     │   │
│  │  ┌─────────────┐                                         │   │
│  │  │ Backend     │                                         │   │
│  │  │ Services    │                                         │   │
│  │  └─────────────┘                                         │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Certificate types:                                               │
│  ┌──────────────────────┬──────────────────────────────────┐   │
│  │ Google-Managed       │ Self-Managed                      │   │
│  ├──────────────────────┼──────────────────────────────────┤   │
│  │ Google provisions &  │ You provide cert + private key   │   │
│  │ auto-renews          │ You handle renewal               │   │
│  │ Free (Let's Encrypt) │ Use your own CA (DigiCert, etc) │   │
│  │ DNS or LB auth       │ Upload to Certificate Manager    │   │
│  │ Recommended          │ For custom CA requirements       │   │
│  └──────────────────────┴──────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Certificate Manager | AWS Certificate Manager (ACM) | Azure Key Vault / App Gateway |
|---------|------------------------|-------------------------------|-------------------------------|
| Service | Certificate Manager | ACM | Key Vault Certificates |
| Managed certs | Yes (Google-managed) | Yes (ACM-issued) | Yes (via App Service) |
| Free certs | Yes | Yes | Depends on CA |
| Wildcard support | Yes | Yes | Yes |
| Auto-renewal | Yes | Yes | Yes (for managed) |
| DNS validation | Yes (DNS authorization) | Yes (CNAME record) | Yes |
| Multi-domain (SAN) | Yes (up to 100 domains) | Yes (up to 10 names) | Yes |
| Certificate maps | Yes | N/A (direct attachment) | N/A |
| Private CA | Yes (CA Service) | Yes (Private CA) | Yes (DigiCert, etc) |
| Used with | Cloud Load Balancing | ALB, NLB, CloudFront | App Gateway, Front Door |

### Certificate Manager vs Classic SSL Certificates

```
┌──────────────────────────────────────────────────────────────┐
│         CERTIFICATE MANAGER vs CLASSIC SSL CERTS              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Classic (legacy):                                             │
│  • google_compute_managed_ssl_certificate                    │
│  • google_compute_ssl_certificate                            │
│  • Attached directly to target HTTPS proxy                   │
│  • Max 15 certs per proxy                                    │
│  • No certificate maps                                       │
│  • Still works, but not recommended for new setups           │
│                                                                │
│  Certificate Manager (recommended):                            │
│  • google_certificate_manager_certificate                    │
│  • Uses certificate maps for flexible routing                │
│  • Max 10,000 certs per map                                  │
│  • DNS authorization (pre-provision before LB exists)       │
│  • Wildcard support with DNS auth                            │
│  • Better multi-domain support                               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Pricing

| Component | Cost |
|-----------|------|
| Google-managed certificates | Free |
| Self-managed certificates | Free |
| Certificate maps | Free |
| DNS authorizations | Free |
| Private CA (CA Service) | Starting at $20/month per CA |

---

## Part 2 — Google-Managed Certificates

### How Google-Managed Certificates Work

```
┌──────────────────────────────────────────────────────────────┐
│         GOOGLE-MANAGED CERTIFICATE LIFECYCLE                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. You create a certificate resource with domain names      │
│  2. Google initiates domain validation:                       │
│     • DNS authorization: you add a CNAME record              │
│     • LB authorization: Google validates via HTTP-01          │
│  3. Google issues certificate (from public CA)               │
│  4. Certificate attached to load balancer via cert map       │
│  5. Auto-renewal ~30 days before expiry (no action needed)   │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  States:                                              │    │
│  │                                                      │    │
│  │  PROVISIONING → ACTIVE → (auto-renew) → ACTIVE      │    │
│  │       │                                               │    │
│  │       └──→ FAILED (domain validation failed)         │    │
│  │            • DNS record not found                    │    │
│  │            • Domain doesn't resolve to LB            │    │
│  │            • CAA record blocks issuance              │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Certificate validity: 90 days (auto-renewed)                 │
│  Renewal: Begins ~30 days before expiry                      │
│  Issuer: Google Trust Services or Let's Encrypt              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating a Google-Managed Certificate

```bash
# With DNS authorization (recommended)
gcloud certificate-manager certificates create api-cert \
    --domains="api.example.com" \
    --dns-authorizations=api-dns-auth

# With load balancer authorization
gcloud certificate-manager certificates create app-cert \
    --domains="app.example.com"
    # No --dns-authorizations = defaults to LB authorization

# Multi-domain certificate
gcloud certificate-manager certificates create multi-cert \
    --domains="api.example.com,app.example.com,admin.example.com" \
    --dns-authorizations=api-dns-auth,app-dns-auth,admin-dns-auth

# Check certificate status
gcloud certificate-manager certificates describe api-cert
```

---

## Part 3 — Self-Managed Certificates

### Uploading Self-Managed Certificates

```bash
# Upload a self-managed certificate (your own cert + key)
gcloud certificate-manager certificates create custom-cert \
    --certificate-file=./server.crt \
    --private-key-file=./server.key \
    --description="Custom CA certificate for api.example.com"

# Update (replace) a self-managed certificate
gcloud certificate-manager certificates update custom-cert \
    --certificate-file=./new-server.crt \
    --private-key-file=./new-server.key
```

### When to Use Self-Managed

```
Use self-managed certificates when:
• Your organization requires a specific CA (e.g., DigiCert, Entrust)
• You need Extended Validation (EV) certificates
• Compliance requires certificates from an approved CA list
• You have existing certificates from another provider
• You need certificates signed by a private/internal CA

Remember:
• YOU are responsible for renewal before expiry
• Private key must be RSA-2048+ or ECDSA P-256/P-384
• Certificate chain must be complete (include intermediates)
• PEM format required
```

---

## Part 4 — Certificate Maps & Map Entries

### Certificate Map Architecture

```
┌──────────────────────────────────────────────────────────────┐
│         CERTIFICATE MAPS                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  A certificate map ROUTES incoming TLS requests to the       │
│  correct certificate based on the hostname (SNI):            │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Certificate Map: "prod-cert-map"                    │    │
│  │                                                      │    │
│  │  Map Entries:                                        │    │
│  │  ┌────────────────────────────────────────────────┐ │    │
│  │  │ Hostname: "api.example.com"                     │ │    │
│  │  │ Certificate: api-cert (Google-managed)          │ │    │
│  │  │ Matcher: EXACT                                  │ │    │
│  │  ├────────────────────────────────────────────────┤ │    │
│  │  │ Hostname: "app.example.com"                     │ │    │
│  │  │ Certificate: app-cert (Google-managed)          │ │    │
│  │  │ Matcher: EXACT                                  │ │    │
│  │  ├────────────────────────────────────────────────┤ │    │
│  │  │ Hostname: "*.staging.example.com"               │ │    │
│  │  │ Certificate: staging-wildcard (Google-managed)  │ │    │
│  │  │ Matcher: WILDCARD                               │ │    │
│  │  ├────────────────────────────────────────────────┤ │    │
│  │  │ PRIMARY (default — no hostname match)           │ │    │
│  │  │ Certificate: default-cert                       │ │    │
│  │  │ Matcher: DEFAULT                                │ │    │
│  │  └────────────────────────────────────────────────┘ │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  The map is attached to a target HTTPS proxy.                 │
│  When a TLS handshake arrives, the LB checks the SNI         │
│  hostname against the map entries and serves the              │
│  matching certificate.                                        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating Certificate Maps

```bash
# Create a certificate map
gcloud certificate-manager maps create prod-cert-map

# Create map entries (hostname → certificate)
gcloud certificate-manager maps entries create api-entry \
    --map=prod-cert-map \
    --certificates=api-cert \
    --hostname="api.example.com"

gcloud certificate-manager maps entries create app-entry \
    --map=prod-cert-map \
    --certificates=app-cert \
    --hostname="app.example.com"

# Wildcard entry
gcloud certificate-manager maps entries create staging-wildcard-entry \
    --map=prod-cert-map \
    --certificates=staging-wildcard-cert \
    --hostname="*.staging.example.com"

# Primary (default) entry — serves when no hostname matches
gcloud certificate-manager maps entries create default-entry \
    --map=prod-cert-map \
    --certificates=default-cert

# Attach map to target HTTPS proxy
gcloud compute target-https-proxies update my-https-proxy \
    --certificate-map=prod-cert-map

# List map entries
gcloud certificate-manager maps entries list --map=prod-cert-map
```

---

## Part 5 — DNS Authorization

### How DNS Authorization Works

```
┌──────────────────────────────────────────────────────────────┐
│         DNS AUTHORIZATION                                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  DNS authorization proves domain ownership via a DNS record  │
│  BEFORE the certificate is needed:                            │
│                                                                │
│  Step 1: Create DNS authorization                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  gcloud certificate-manager dns-authorizations       │    │
│  │    create api-dns-auth --domain="api.example.com"   │    │
│  │                                                      │    │
│  │  Output:                                             │    │
│  │  dnsResourceRecord:                                  │    │
│  │    name: _acme-challenge.api.example.com.           │    │
│  │    type: CNAME                                      │    │
│  │    data: XXXX.YY.acme.goog.                         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Step 2: Add the CNAME record to your DNS                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  _acme-challenge.api.example.com. CNAME              │    │
│  │    XXXX.YY.acme.goog.                                │    │
│  │                                                      │    │
│  │  (In Cloud DNS, Route 53, Cloudflare, etc.)          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Step 3: Create certificate referencing the DNS auth         │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  gcloud certificate-manager certificates create       │    │
│  │    api-cert --domains="api.example.com"              │    │
│  │    --dns-authorizations=api-dns-auth                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Benefits of DNS authorization:                                │
│  • Pre-provision certs BEFORE LB exists                      │
│  • Required for WILDCARD certificates                        │
│  • Faster issuance (validation already done)                 │
│  • Works for domains not yet pointing to GCP                 │
│  • Reusable across multiple certificates                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create DNS authorization
gcloud certificate-manager dns-authorizations create api-dns-auth \
    --domain="api.example.com"

# Describe (to get the CNAME record to add)
gcloud certificate-manager dns-authorizations describe api-dns-auth

# For wildcard (authorize the parent domain)
gcloud certificate-manager dns-authorizations create wildcard-auth \
    --domain="example.com"

# Create wildcard cert using DNS auth
gcloud certificate-manager certificates create wildcard-cert \
    --domains="*.example.com" \
    --dns-authorizations=wildcard-auth
```

---

## Part 6 — Load Balancer Authorization

### How LB Authorization Works

```
┌──────────────────────────────────────────────────────────────┐
│         LOAD BALANCER AUTHORIZATION                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  LB authorization validates domain ownership by checking     │
│  that the domain resolves to the load balancer's IP:         │
│                                                                │
│  Requirements:                                                 │
│  1. Domain must already point to the LB's IP address         │
│  2. LB must be created and running                           │
│  3. Certificate map entry attached to the HTTPS proxy        │
│                                                                │
│  Process:                                                      │
│  1. Create certificate (no --dns-authorizations flag)        │
│  2. Attach to LB via certificate map                         │
│  3. Google validates via HTTP-01 challenge                   │
│  4. Certificate becomes ACTIVE                               │
│                                                                │
│  Limitations:                                                  │
│  • Domain MUST resolve to the LB IP first                   │
│  • Does NOT support wildcard certificates                    │
│  • Slower provisioning (needs LB + DNS both ready)           │
│  • Can't pre-provision                                       │
│                                                                │
│  When to use:                                                  │
│  • Simple setups with domain already pointing to GCP LB     │
│  • When you don't have DNS management access                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 7 — Certificate Issuance Config & CA Service

### Private Certificates with CA Service

```
┌──────────────────────────────────────────────────────────────┐
│         PRIVATE CERTIFICATES (CA SERVICE)                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  For internal/private certificates, use Certificate Manager  │
│  with CA Service (Google's private CA):                       │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  CA Service                                          │    │
│  │  ├── CA Pool: "internal-pool"                        │    │
│  │  │   ├── Root CA: "internal-root-ca"                │    │
│  │  │   └── Subordinate CA: "internal-sub-ca"          │    │
│  │  │                                                   │    │
│  │  │           │                                       │    │
│  │  │           ▼                                       │    │
│  │  │  Certificate Issuance Config                     │    │
│  │  │  → Tells Certificate Manager which CA Pool       │    │
│  │  │    to use for issuing certificates                │    │
│  │  │           │                                       │    │
│  │  │           ▼                                       │    │
│  │  │  Certificate Manager                              │    │
│  │  │  → Creates cert using private CA                  │    │
│  │  │  → Auto-renewal via CA Service                   │    │
│  │  │           │                                       │    │
│  │  │           ▼                                       │    │
│  │  │  Internal Load Balancer                           │    │
│  │  │  → Serves private cert for internal traffic      │    │
│  │  └──────────────────────────────────────────────────┘    │
│  │                                                           │
│  │  Use cases:                                               │
│  │  • mTLS between internal services                        │
│  │  • Internal load balancers                               │
│  │  • Service mesh certificates                             │
│  │  • Private domains (*.corp.internal)                     │
│  │                                                           │
│  └──────────────────────────────────────────────────────────┘│
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create a certificate issuance config
gcloud certificate-manager issuance-configs create internal-config \
    --ca-pool="projects/my-project/locations/us-central1/caPools/internal-pool"

# Create a Google-managed cert using private CA
gcloud certificate-manager certificates create internal-cert \
    --domains="api.corp.internal" \
    --issuance-config=internal-config \
    --dns-authorizations=internal-dns-auth
```

---

## Part 8 — Wildcard Certificates

### Creating Wildcard Certificates

```
┌──────────────────────────────────────────────────────────────┐
│         WILDCARD CERTIFICATES                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Wildcard certs cover all subdomains at one level:            │
│  *.example.com matches:                                       │
│  ├── api.example.com          ✓                              │
│  ├── app.example.com          ✓                              │
│  ├── admin.example.com        ✓                              │
│  └── sub.api.example.com      ✗ (second level)              │
│                                                                │
│  Requirements:                                                 │
│  • MUST use DNS authorization (not LB authorization)         │
│  • DNS auth must be for the parent domain                    │
│  • One DNS CNAME record required                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Step 1: Create DNS authorization for parent domain
gcloud certificate-manager dns-authorizations create example-auth \
    --domain="example.com"

# Step 2: Add the CNAME record (from describe output)
gcloud certificate-manager dns-authorizations describe example-auth

# Step 3: Create wildcard certificate
gcloud certificate-manager certificates create wildcard-cert \
    --domains="*.example.com" \
    --dns-authorizations=example-auth

# Combine wildcard + apex domain
gcloud certificate-manager certificates create full-cert \
    --domains="example.com,*.example.com" \
    --dns-authorizations=example-auth
```

---

## Part 9 — Multi-Domain & SAN Certificates

### Subject Alternative Names (SAN)

```bash
# Single cert for multiple domains (up to 100)
gcloud certificate-manager certificates create multi-domain-cert \
    --domains="api.example.com,app.example.com,admin.example.com,docs.example.com" \
    --dns-authorizations=api-auth,app-auth,admin-auth,docs-auth

# Mix of exact and wildcard domains
gcloud certificate-manager certificates create mixed-cert \
    --domains="example.com,*.example.com,*.staging.example.com" \
    --dns-authorizations=example-auth,staging-auth
```

---

## Part 10 — Certificate Lifecycle & Renewal

### Lifecycle

```
┌──────────────────────────────────────────────────────────────┐
│         CERTIFICATE LIFECYCLE                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Google-Managed:                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  CREATE → PROVISIONING → ACTIVE ──────────────┐     │    │
│  │                                                │     │    │
│  │                          (auto every ~60 days) │     │    │
│  │                                                ▼     │    │
│  │                           RENEWAL_PENDING → ACTIVE   │    │
│  │                                                      │    │
│  │  No action needed — fully automatic                  │    │
│  │  Renewal starts ~30 days before expiry               │    │
│  │  No downtime during renewal                          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Self-Managed:                                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  UPLOAD → ACTIVE ─────────────────────────────┐     │    │
│  │                                                │     │    │
│  │                          (manual before expiry)│     │    │
│  │                                                ▼     │    │
│  │                           UPDATE (new cert) → ACTIVE │    │
│  │                                                      │    │
│  │  YOU must renew and re-upload before expiry          │    │
│  │  Set up monitoring alerts for expiry                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 11 — Integration with Load Balancers

### Supported Load Balancer Types

| Load Balancer Type | Certificate Manager Support |
|-------------------|---------------------------|
| Global external Application LB | Yes (recommended) |
| Regional external Application LB | Yes |
| Cross-region internal Application LB | Yes |
| Regional internal Application LB | Yes |
| Classic Application LB | No (use classic SSL certs) |
| Network LB (TCP/SSL proxy) | Yes |

### Attaching to Load Balancer

```bash
# Create the full chain: cert → map entry → map → HTTPS proxy

# 1. Create certificate (see Part 2)
# 2. Create certificate map
gcloud certificate-manager maps create prod-map

# 3. Create map entry
gcloud certificate-manager maps entries create api-entry \
    --map=prod-map \
    --certificates=api-cert \
    --hostname="api.example.com"

# 4. Create primary (default) entry
gcloud certificate-manager maps entries create default-entry \
    --map=prod-map \
    --certificates=default-cert

# 5. Attach map to target HTTPS proxy
gcloud compute target-https-proxies create my-https-proxy \
    --url-map=my-url-map \
    --certificate-map=prod-map

# OR update existing proxy
gcloud compute target-https-proxies update my-https-proxy \
    --certificate-map=prod-map
```

---

## Part 12 — IAM & Security

### IAM Roles

| Role | Description |
|------|-------------|
| `roles/certificatemanager.owner` | Full management |
| `roles/certificatemanager.editor` | Create/update/delete certs, maps |
| `roles/certificatemanager.viewer` | View certs, maps |

### Security Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│         SECURITY BEST PRACTICES                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Prefer Google-managed over self-managed                  │
│     → Auto-renewal eliminates expired cert risk              │
│                                                                │
│  2. Use DNS authorization over LB authorization              │
│     → Pre-provision certs before DNS migration               │
│     → Required for wildcard certs                            │
│                                                                │
│  3. Set up CAA DNS records                                   │
│     → Restrict which CAs can issue for your domain          │
│     → example.com. CAA 0 issue "pki.goog"                   │
│                                                                │
│  4. Monitor cert expiry for self-managed certs               │
│     → Cloud Monitoring alert 30 days before expiry          │
│                                                                │
│  5. Use certificate maps (not direct proxy attachment)       │
│     → Easier rotation and multi-domain management           │
│                                                                │
│  6. Keep DNS authorizations active                           │
│     → Don't delete CNAME records after cert is issued       │
│     → Needed for auto-renewal                               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 13 — Monitoring & Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Cert stuck in PROVISIONING | DNS record missing | Add required CNAME record |
| Cert stuck in PROVISIONING | DNS propagation delay | Wait up to 24 hours |
| Cert stuck in PROVISIONING | CAA record blocks issuance | Add `CAA 0 issue "pki.goog"` |
| Cert FAILED | Domain doesn't resolve to LB IP | Update DNS A record |
| Cert FAILED | DNS auth CNAME incorrect | Verify CNAME record value |
| Renewal failed | DNS auth CNAME deleted | Re-add the CNAME record |
| Wrong cert served | Map entry hostname mismatch | Check certificate map entries |
| Mixed content warnings | Backend serves HTTP resources | Update app to use HTTPS URLs |

### Monitoring

```bash
# Check certificate status
gcloud certificate-manager certificates describe CERT_NAME

# Check certificate map status
gcloud certificate-manager maps describe MAP_NAME

# List all certificates with state
gcloud certificate-manager certificates list \
    --format="table(name,domains,state,expireTime)"

# Log filter for certificate events
gcloud logging read 'resource.type="certificate_manager_certificate"' \
    --project=my-project --limit=20
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Complete Setup

```hcl
# ─── DNS Authorization ───────────────────────────────────────
resource "google_certificate_manager_dns_authorization" "api" {
  name   = "api-dns-auth"
  domain = "api.example.com"
}

resource "google_certificate_manager_dns_authorization" "wildcard" {
  name   = "wildcard-dns-auth"
  domain = "example.com"
}

# ─── Add CNAME records to Cloud DNS ──────────────────────────
resource "google_dns_record_set" "api_cert_validation" {
  managed_zone = google_dns_managed_zone.example.name
  name         = google_certificate_manager_dns_authorization.api.dns_resource_record[0].name
  type         = google_certificate_manager_dns_authorization.api.dns_resource_record[0].type
  ttl          = 300
  rrdatas      = [google_certificate_manager_dns_authorization.api.dns_resource_record[0].data]
}

resource "google_dns_record_set" "wildcard_cert_validation" {
  managed_zone = google_dns_managed_zone.example.name
  name         = google_certificate_manager_dns_authorization.wildcard.dns_resource_record[0].name
  type         = google_certificate_manager_dns_authorization.wildcard.dns_resource_record[0].type
  ttl          = 300
  rrdatas      = [google_certificate_manager_dns_authorization.wildcard.dns_resource_record[0].data]
}

# ─── Google-Managed Certificates ─────────────────────────────
resource "google_certificate_manager_certificate" "api" {
  name = "api-cert"

  managed {
    domains            = ["api.example.com"]
    dns_authorizations = [google_certificate_manager_dns_authorization.api.id]
  }
}

resource "google_certificate_manager_certificate" "wildcard" {
  name = "wildcard-cert"

  managed {
    domains            = ["*.example.com", "example.com"]
    dns_authorizations = [google_certificate_manager_dns_authorization.wildcard.id]
  }
}

# ─── Self-Managed Certificate ────────────────────────────────
resource "google_certificate_manager_certificate" "custom" {
  name = "custom-cert"

  self_managed {
    pem_certificate = file("${path.module}/certs/server.crt")
    pem_private_key = file("${path.module}/certs/server.key")
  }
}

# ─── Certificate Map ─────────────────────────────────────────
resource "google_certificate_manager_certificate_map" "prod" {
  name = "prod-cert-map"
}

# ─── Map Entries ──────────────────────────────────────────────
resource "google_certificate_manager_certificate_map_entry" "api" {
  name         = "api-entry"
  map          = google_certificate_manager_certificate_map.prod.name
  hostname     = "api.example.com"
  certificates = [google_certificate_manager_certificate.api.id]
}

resource "google_certificate_manager_certificate_map_entry" "wildcard" {
  name         = "wildcard-entry"
  map          = google_certificate_manager_certificate_map.prod.name
  hostname     = "*.example.com"
  certificates = [google_certificate_manager_certificate.wildcard.id]
}

# Primary (default) entry
resource "google_certificate_manager_certificate_map_entry" "default" {
  name         = "default-entry"
  map          = google_certificate_manager_certificate_map.prod.name
  certificates = [google_certificate_manager_certificate.wildcard.id]
  matcher      = "PRIMARY"
}

# ─── Attach to Load Balancer ─────────────────────────────────
resource "google_compute_target_https_proxy" "default" {
  name            = "my-https-proxy"
  url_map         = google_compute_url_map.default.id
  certificate_map = "//certificatemanager.googleapis.com/${google_certificate_manager_certificate_map.prod.id}"
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# DNS AUTHORIZATIONS
# ═══════════════════════════════════════════════════════════════

# Create
gcloud certificate-manager dns-authorizations create NAME --domain="DOMAIN"

# Describe (get CNAME record)
gcloud certificate-manager dns-authorizations describe NAME

# List
gcloud certificate-manager dns-authorizations list

# Delete
gcloud certificate-manager dns-authorizations delete NAME

# ═══════════════════════════════════════════════════════════════
# CERTIFICATES
# ═══════════════════════════════════════════════════════════════

# Google-managed with DNS auth
gcloud certificate-manager certificates create NAME \
    --domains="DOMAIN1,DOMAIN2" \
    --dns-authorizations=AUTH1,AUTH2

# Google-managed with LB auth
gcloud certificate-manager certificates create NAME \
    --domains="DOMAIN"

# Self-managed
gcloud certificate-manager certificates create NAME \
    --certificate-file=CERT.pem --private-key-file=KEY.pem

# List
gcloud certificate-manager certificates list

# Describe
gcloud certificate-manager certificates describe NAME

# Delete
gcloud certificate-manager certificates delete NAME

# ═══════════════════════════════════════════════════════════════
# CERTIFICATE MAPS
# ═══════════════════════════════════════════════════════════════

# Create map
gcloud certificate-manager maps create MAP_NAME

# Create map entry
gcloud certificate-manager maps entries create ENTRY_NAME \
    --map=MAP_NAME \
    --certificates=CERT_NAME \
    --hostname="HOSTNAME"

# Create primary entry
gcloud certificate-manager maps entries create ENTRY_NAME \
    --map=MAP_NAME --certificates=CERT_NAME

# List entries
gcloud certificate-manager maps entries list --map=MAP_NAME

# Attach to HTTPS proxy
gcloud compute target-https-proxies update PROXY_NAME \
    --certificate-map=MAP_NAME
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Multi-Environment Certificate Setup

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: MULTI-ENVIRONMENT CERTIFICATES                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Certificate Map: "prod-cert-map"                         │        │
│  │  ├── api.example.com → prod-api-cert (Google-managed)    │        │
│  │  ├── app.example.com → prod-app-cert (Google-managed)    │        │
│  │  └── *.example.com → prod-wildcard-cert (default)        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Certificate Map: "staging-cert-map"                      │        │
│  │  ├── api.staging.example.com → staging-cert               │        │
│  │  └── *.staging.example.com → staging-wildcard             │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  DNS authorizations created upfront for all domains.                  │
│  All certs Google-managed = zero renewal management.                  │
│  Terraform manages the full chain per environment.                    │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Zero-Downtime Certificate Migration

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: MIGRATING FROM CLASSIC TO CERTIFICATE MANAGER          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Step 1: Create DNS authorizations + Google-managed certs            │
│          (they provision in parallel, no impact on live traffic)     │
│                                                                        │
│  Step 2: Create certificate map + entries                             │
│          (map not attached yet — no impact)                           │
│                                                                        │
│  Step 3: Wait for all certs to reach ACTIVE state                    │
│                                                                        │
│  Step 4: Attach certificate map to HTTPS proxy                       │
│          (atomic switch — zero downtime)                              │
│          gcloud compute target-https-proxies update PROXY \          │
│              --certificate-map=prod-cert-map                          │
│                                                                        │
│  Step 5: Verify (test all hostnames with curl)                       │
│          curl -v https://api.example.com 2>&1 | grep issuer          │
│                                                                        │
│  Step 6: Clean up old classic SSL certificate resources               │
│                                                                        │
│  Rollback: Remove --certificate-map from proxy → reverts to classic  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: SaaS Multi-Tenant Custom Domains

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: CUSTOM DOMAINS PER CUSTOMER (SAAS)                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  SaaS platform where each customer brings their own domain:          │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Certificate Map: "saas-cert-map"                         │        │
│  │                                                           │        │
│  │  ├── shop.customer-a.com → cert-customer-a               │        │
│  │  ├── app.customer-b.io  → cert-customer-b                │        │
│  │  ├── store.customer-c.co → cert-customer-c               │        │
│  │  ├── ... (up to 10,000 entries per map)                  │        │
│  │  └── PRIMARY → *.saas-platform.com (default)             │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Onboarding a new customer:                                           │
│  1. Customer sets up DNS: CNAME shop.their-domain.com                │
│     → saas-platform.com (or A record to LB IP)                      │
│  2. Create DNS authorization for their domain                        │
│  3. Customer adds CNAME for _acme-challenge                          │
│  4. Create Google-managed cert                                        │
│  5. Add certificate map entry                                         │
│  6. Auto-renewal handled by Google — no maintenance                  │
│                                                                        │
│  Automation: API/Terraform module triggered by customer signup       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create DNS auth | `gcloud certificate-manager dns-authorizations create NAME --domain="D"` |
| Get DNS record | `gcloud certificate-manager dns-authorizations describe NAME` |
| Create managed cert | `gcloud certificate-manager certificates create NAME --domains="D" --dns-authorizations=A` |
| Create self-managed cert | `... create NAME --certificate-file=C --private-key-file=K` |
| Create wildcard cert | `--domains="*.example.com" --dns-authorizations=parent-auth` |
| Create cert map | `gcloud certificate-manager maps create MAP` |
| Create map entry | `... maps entries create E --map=M --certificates=C --hostname="H"` |
| Attach to proxy | `gcloud compute target-https-proxies update P --certificate-map=M` |
| Check cert status | `gcloud certificate-manager certificates describe NAME` |
| Max certs per map | 10,000 |
| Max domains per cert | 100 (SAN) |
| Auto-renewal | Yes (Google-managed, ~30 days before expiry) |
| Pricing | Free |

---

## What is TLS/SSL? (Beginner Explanation)

### Simple Analogy

```
┌──────────────────────────────────────────────────────────────┐
│         TLS/SSL IN PLAIN ENGLISH                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  TLS is like a SEALED ENVELOPE for your internet traffic.    │
│                                                                │
│  Without TLS (HTTP):                                           │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Your browser ──── postcard ────→ Server             │    │
│  │                                                      │    │
│  │  "Dear server, my credit card is 4111-1111-1111"     │    │
│  │                                                      │    │
│  │  Anyone in between (coffee shop wifi, ISP, etc.)     │    │
│  │  can READ the postcard. No privacy.                  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  With TLS (HTTPS):                                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Your browser ──── sealed envelope ────→ Server      │    │
│  │                                                      │    │
│  │  The envelope is:                                    │    │
│  │  • SEALED — nobody can read it in transit            │    │
│  │  • STAMPED — you know it's really from the server   │    │
│  │  • TAMPER-PROOF — nobody can modify it              │    │
│  │                                                      │    │
│  │  The "stamp" is the TLS CERTIFICATE.                 │    │
│  │  It proves: "I am really api.example.com"            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  What is a TLS certificate?                                    │
│  • A digital file that proves a website's identity            │
│  • Issued by a trusted Certificate Authority (CA)            │
│  • Contains the domain name, expiry date, and public key     │
│  • Your browser checks the cert before showing the 🔒 icon  │
│                                                                │
│  If the cert is missing, expired, or wrong:                   │
│  → Browser shows "Your connection is not private" ⚠️         │
│  → Users lose trust and leave your site                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Why Certificates Matter

```
1. HTTPS (required by modern browsers):
   • Chrome marks HTTP sites as "Not Secure"
   • HTTP/2 and HTTP/3 REQUIRE TLS
   • SEO ranking boost for HTTPS sites
   • No cert = no HTTPS = broken experience

2. Trust:
   • Certificates prove your server IS who it claims to be
   • Without a cert, attackers can impersonate your site
   • Users see the 🔒 padlock = "this is safe"

3. Compliance:
   • PCI-DSS requires TLS for payment processing
   • HIPAA requires encryption in transit
   • SOC 2 expects TLS for all web traffic

4. API Security:
   • Service-to-service communication needs TLS
   • mTLS (mutual TLS) for zero-trust architectures
   • API gateways terminate TLS at the edge
```

### Google-Managed vs Self-Managed — When to Use Which

```
┌──────────────────────────────────────────────────────────────┐
│  GOOGLE-MANAGED (recommended for most cases)                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ✓ Use when:                                                  │
│  • You want zero maintenance (auto-provisioned + renewed)    │
│  • Standard domain validation (DV) certs are sufficient      │
│  • You use Google Cloud Load Balancing                        │
│  • You want it free                                           │
│                                                                │
│  How it works:                                                 │
│  1. You tell Google: "I own api.example.com"                 │
│  2. You prove it (DNS record or LB validation)               │
│  3. Google issues a cert from a public CA                    │
│  4. Google auto-renews it every ~60 days                     │
│  5. You never think about it again                           │
│                                                                │
├──────────────────────────────────────────────────────────────┤
│  SELF-MANAGED (for specific requirements)                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ✓ Use when:                                                  │
│  • Your company REQUIRES a specific CA (DigiCert, Entrust)   │
│  • You need Extended Validation (EV) certs (green bar)       │
│  • Compliance mandates certs from an approved CA list        │
│  • You need Organization Validation (OV) certs               │
│  • You're using private/internal CA for internal services    │
│                                                                │
│  How it works:                                                 │
│  1. You buy/generate a cert from your chosen CA              │
│  2. You upload the cert + private key to Certificate Manager │
│  3. YOU must renew and re-upload before it expires           │
│  4. Set up monitoring alerts for expiry dates                │
│                                                                │
└──────────────────────────────────────────────────────────────┘

Rule of thumb: If you don't have a specific reason for
self-managed, use Google-managed. It's free and hands-off.
```

---

## Console Walkthrough: Creating & Managing Certificates

### Creating a Google-Managed Certificate from Console

```
Step 1: Navigate to Certificate Manager
────────────────────────────────────────
Console → Security → Certificate Manager
(Or search "Certificate Manager" in the Console search bar)

Note: Make sure the Certificate Manager API is enabled:
  gcloud services enable certificatemanager.googleapis.com

Step 2: Click "+ Create Certificate"
─────────────────────────────────────
• On the Certificates tab, click "Create Certificate"
• Fill in the form:

  ┌──────────────────────────────────────────────────────┐
  │  Name:         api-cert                               │
  │  Description:  Certificate for api.example.com        │
  │                                                      │
  │  Scope:        Default (global)                       │
  │                                                      │
  │  Certificate type:                                    │
  │    ● Create Google-managed certificate                │
  │    ○ Upload self-managed certificate                  │
  │                                                      │
  │  Domain names:                                        │
  │    api.example.com                                    │
  │    (click "+ Add Domain" for multiple domains)       │
  │                                                      │
  │  Authorization type:                                  │
  │    ● DNS authorization (recommended)                  │
  │    ○ Load balancer authorization                      │
  │                                                      │
  │  DNS authorization:                                   │
  │    ○ Select existing authorization                    │
  │    ● Create new DNS authorization                     │
  └──────────────────────────────────────────────────────┘

Step 3: Click "Create"
──────────────────────
• Certificate will be created in PROVISIONING state
• Google will provide a CNAME record you need to add to DNS
• Certificate becomes ACTIVE once domain is validated
```

### Creating DNS Authorization

```
Step 1: Go to DNS Authorizations Tab
─────────────────────────────────────
Console → Security → Certificate Manager → DNS Authorizations

Step 2: Click "+ Create DNS Authorization"
──────────────────────────────────────────
• Fill in:
  ┌──────────────────────────────────────────────────────┐
  │  Name:    api-dns-auth                                │
  │  Domain:  api.example.com                             │
  └──────────────────────────────────────────────────────┘

Step 3: Note the CNAME Record
─────────────────────────────
• After creation, you'll see a CNAME record to add:
  ┌──────────────────────────────────────────────────────┐
  │  DNS Record Name:  _acme-challenge.api.example.com   │
  │  Type:             CNAME                              │
  │  Data:             XXXX.YY.acme.goog.                 │
  └──────────────────────────────────────────────────────┘

Step 4: Add This Record to Your DNS Provider
────────────────────────────────────────────
• Go to your DNS provider (Cloud DNS, Route 53, Cloudflare, etc.)
• Add the CNAME record exactly as shown
• DNS propagation can take up to 24 hours (usually minutes)

Step 5: Verify Authorization Status
────────────────────────────────────
• Back in Certificate Manager → DNS Authorizations
• Status should change from PENDING to AUTHORIZED
• Now you can create certificates using this authorization

Note: Keep this CNAME record permanently — it's needed for
auto-renewal. Deleting it will cause renewal failures.
```

### Creating a Certificate Map and Attaching to Load Balancer

```
Step 1: Create a Certificate Map
─────────────────────────────────
Console → Security → Certificate Manager → Certificate Maps tab
• Click "+ Create Certificate Map"
• Name: prod-cert-map
• Click "Create"

Step 2: Add Map Entries
───────────────────────
• Click on the certificate map you just created
• Click "+ Add Certificate Map Entry"
• Fill in:

  ┌──────────────────────────────────────────────────────┐
  │  Entry name:  api-entry                               │
  │  Hostname:    api.example.com                         │
  │  Certificates: api-cert (select from dropdown)        │
  └──────────────────────────────────────────────────────┘

• Repeat for each hostname you need
• Add a PRIMARY entry (no hostname) as the default fallback

Step 3: Attach Map to Load Balancer
───────────────────────────────────
Console → Network services → Load balancing
• Click on your HTTPS load balancer → "Edit"
• In the Frontend configuration:
  ┌──────────────────────────────────────────────────────┐
  │  Protocol: HTTPS                                      │
  │  Certificate:                                         │
  │    ● Certificate map: prod-cert-map                   │
  │    ○ Classic certificate (not recommended)            │
  └──────────────────────────────────────────────────────┘
• Click "Update"

Alternatively via gcloud:
  gcloud compute target-https-proxies update my-https-proxy \
      --certificate-map=prod-cert-map

Step 4: Verify
──────────────
• Wait a few minutes for propagation
• Test with curl:
  curl -v https://api.example.com 2>&1 | grep "issuer"
• You should see the Google-issued certificate
```

### Deleting Certificates from Console

```
⚠️  Before deleting, make sure the certificate is not in use.

Step 1: Remove Certificate from Map Entries
───────────────────────────────────────────
Console → Security → Certificate Manager → Certificate Maps
• Click on the map → find entries using this certificate
• Delete or update those entries first
• You CANNOT delete a certificate that's still referenced
  by a map entry

Step 2: Delete the Certificate
──────────────────────────────
Console → Security → Certificate Manager → Certificates tab
• Find the certificate
• Click the three-dot menu (⋮) → "Delete"
• Confirm deletion

Step 3: Clean Up DNS Authorization (optional)
──────────────────────────────────────────────
• If you no longer need the domain authorization:
  Console → DNS Authorizations tab → Delete
• Also remove the CNAME record from your DNS provider

Deletion order (important!):
  1. Remove map entries referencing the cert
  2. Delete the certificate
  3. (Optional) Delete the DNS authorization
  4. (Optional) Delete the certificate map if empty
  5. (Optional) Remove CNAME records from DNS
```

### Troubleshooting: "PROVISIONING Stuck" Issue

```
┌──────────────────────────────────────────────────────────────┐
│         🔧 CERTIFICATE STUCK IN "PROVISIONING"               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  This is the #1 most common issue with Certificate Manager.  │
│  Your certificate was created but never becomes ACTIVE.      │
│                                                                │
│  Check these causes in order:                                  │
│                                                                │
│  1. DNS CNAME record missing or incorrect                     │
│  ────────────────────────────────────────                      │
│  • Describe your DNS authorization:                           │
│    gcloud certificate-manager dns-authorizations describe \  │
│        YOUR_AUTH_NAME                                         │
│  • Verify the CNAME record exists in your DNS:               │
│    nslookup -type=CNAME _acme-challenge.api.example.com      │
│  • Common mistakes:                                           │
│    ✗ Forgot to add the record entirely                       │
│    ✗ Added an A record instead of CNAME                      │
│    ✗ Typo in the CNAME target value                          │
│    ✗ Added it to the wrong DNS zone                          │
│                                                                │
│  2. DNS propagation delay                                      │
│  ────────────────────────────                                  │
│  • DNS changes can take up to 24-48 hours to propagate       │
│  • Check propagation: https://dns.google/                     │
│  • If using Cloudflare: ensure the CNAME is DNS-only         │
│    (grey cloud), NOT proxied (orange cloud)                  │
│                                                                │
│  3. CAA record blocking issuance                               │
│  ───────────────────────────────                               │
│  • If you have a CAA DNS record, it must allow Google's CA   │
│  • Add: example.com. CAA 0 issue "pki.goog"                  │
│  • Check: nslookup -type=CAA example.com                     │
│                                                                │
│  4. Domain doesn't resolve to LB (LB authorization only)      │
│  ────────────────────────────────────────────────────          │
│  • If using LB authorization (not DNS auth):                 │
│    The domain must point to your load balancer's IP           │
│  • Check: nslookup api.example.com                           │
│  • The A record must match your LB's IP address              │
│                                                                │
│  5. Certificate not attached to a load balancer               │
│  ─────────────────────────────────────────────                 │
│  • For LB authorization, the cert MUST be attached to a      │
│    map entry, which is attached to an HTTPS proxy            │
│  • Unattached certs with LB auth will never provision        │
│                                                                │
│  Quick fix checklist:                                          │
│  □ DNS authorization CNAME record exists and is correct      │
│  □ CNAME is not proxied (Cloudflare users)                   │
│  □ No CAA record blocking pki.goog                           │
│  □ (LB auth) Domain A record points to LB IP                │
│  □ (LB auth) Cert is attached via map entry to HTTPS proxy  │
│  □ Waited at least 24 hours for DNS propagation              │
│                                                                │
│  If still stuck after 48 hours:                                │
│  • Delete the certificate and DNS authorization              │
│  • Recreate both from scratch                                 │
│  • Double-check DNS records before creating the cert         │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 45: Identity-Aware Proxy (IAP)** → `45-identity-aware-proxy.md`
