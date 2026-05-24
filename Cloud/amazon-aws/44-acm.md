# Chapter 44: ACM (AWS Certificate Manager)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: ACM Fundamentals](#part-1-acm-fundamentals)
- [Part 2: Requesting a Public Certificate (Full Portal Walkthrough)](#part-2-requesting-a-public-certificate-full-portal-walkthrough)
- [Part 3: Requesting a Private Certificate](#part-3-requesting-a-private-certificate)
- [Part 4: Certificate Association & Renewal](#part-4-certificate-association--renewal)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is SSL/TLS? Why Do We Need Certificates?

When you visit a website with `https://` (the padlock icon 🔒 in your browser), the connection is **encrypted** — no one between you and the server can read the data (not your ISP, not hackers on public Wi-Fi).

This encryption is powered by an **SSL/TLS certificate** — a digital document that proves the website is who it claims to be. Without a certificate, browsers show a scary "Not Secure" warning.

**Getting certificates used to be painful:**
1. Generate a certificate request
2. Pay a Certificate Authority (CA) like DigiCert $100-300/year
3. Prove you own the domain
4. Install the certificate on your server
5. Remember to renew it before it expires (or your site goes down!)

**ACM makes this free and automatic:**
- 🆓 Public certificates are **FREE** (yes, $0)
- 🔄 ACM **auto-renews** certificates before they expire
- 🔗 One-click integration with ALB, CloudFront, API Gateway
- ✅ Domain validation via DNS record (automated with Route 53)

AWS Certificate Manager (ACM) provisions, manages, and deploys SSL/TLS certificates for use with AWS services. Public certificates are free. ACM handles automatic renewal.

```
What you'll learn:
├── ACM fundamentals (public vs private certificates)
├── Requesting a public certificate (portal walkthrough)
├── Validation methods (DNS vs email)
├── Requesting a private certificate (Private CA)
├── Associating certificates with AWS resources
├── Automatic renewal
└── Terraform & CLI examples
```

---

## Part 1: ACM Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW ACM WORKS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ACM Certificate → Associate with AWS resource → SSL/TLS enabled   │
│                                                                       │
│ Supported services:                                                  │
│ ├── Elastic Load Balancing (ALB, NLB, CLB)                      │
│ ├── Amazon CloudFront                                             │
│ ├── Amazon API Gateway                                            │
│ ├── AWS App Runner                                                │
│ ├── AWS Elastic Beanstalk                                        │
│ ├── AWS CloudFormation (via supported resources)                │
│ └── ⚠️ NOT directly on EC2 (use self-managed certs)             │
│                                                                       │
│ Certificate types:                                                   │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ Public        │ Private              │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Cost                 │ FREE          │ $400/month per CA    │   │
│ │ Trusted by browsers  │ ✅            │ ❌ (internal only)   │   │
│ │ Domain validation    │ DNS or email  │ Not required         │   │
│ │ Auto-renewal         │ ✅            │ ✅                   │   │
│ │ Use case             │ Public sites  │ Internal services    │   │
│ │ Exportable           │ ❌            │ ✅                   │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│                                                                       │
│ Key facts:                                                           │
│ ├── Public certificates are FREE (unlimited quantity)            │
│ ├── Private key never leaves ACM (non-exportable for public)   │
│ ├── Auto-renewal 60 days before expiry                          │
│ ├── RSA 2048 or ECDSA P-256/P-384                               │
│ ├── Wildcard support: *.example.com                              │
│ ├── SAN support: Multiple domains per certificate               │
│ ├── ⚠️ CloudFront certs must be in us-east-1                    │
│ └── Regional certs for ALB/API Gateway (match resource region) │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Requesting a Public Certificate (Full Portal Walkthrough)

```
Console → Certificate Manager → Request a certificate

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: CERTIFICATE TYPE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ● Request a public certificate                                 │
│   → Free SSL/TLS certificate for public domains              │
│   → Trusted by all major browsers                            │
│                                                                   │
│ ○ Request a private certificate                                │
│   → Requires AWS Private Certificate Authority               │
│   → For internal services only                               │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: DOMAIN NAMES                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Fully qualified domain name:                                   │
│ [example.com]                                                  │
│ → Primary domain for the certificate                         │
│                                                                   │
│ Add another name to this certificate:                         │
│ [+ Add another name]                                          │
│ ├── [*.example.com]                                           │
│ │   → Wildcard: Covers all subdomains one level deep         │
│ │   → e.g., api.example.com, www.example.com                │
│ │   → ⚠️ Does NOT cover sub.sub.example.com                 │
│ ├── [api.example.com]                                         │
│ └── [staging.example.com]                                     │
│                                                                   │
│ → ⚡ Common pattern: example.com + *.example.com in one cert │
│ → Max 10 domain names per certificate (request increase)     │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: VALIDATION METHOD                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ● DNS validation (⚡ recommended)                              │
│   → Add CNAME record to your DNS                             │
│   → ACM auto-validates (no manual steps if Route 53)        │
│   → Auto-renewal works automatically                        │
│   → Works with any DNS provider                             │
│                                                                   │
│ ○ Email validation                                             │
│   → Sends email to domain contacts                           │
│   → admin@example.com, webmaster@example.com, etc.          │
│   → Manual step required for each renewal                   │
│   → ⚠️ Renewal requires manual email approval               │
│                                                                   │
│ Key algorithm:                                                 │
│ ● RSA 2048 (⚡ widest compatibility)                         │
│ ○ ECDSA P 256                                                 │
│ ○ ECDSA P 384                                                 │
│ → ECDSA: Better performance, smaller key, newer standard    │
│                                                                   │
│ Tags:                                                          │
│ [Environment] = [Production]                                  │
│                                                                   │
│                    [Request]                                    │
└─────────────────────────────────────────────────────────────────┘

After requesting (DNS validation):

┌─────────────────────────────────────────────────────────────────┐
│           DNS VALIDATION RECORDS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Status: Pending validation                                     │
│                                                                   │
│ Domain: example.com                                            │
│ CNAME name:  _abc123.example.com                              │
│ CNAME value: _xyz789.acm-validations.aws.                    │
│                                                                   │
│ Domain: *.example.com                                          │
│ CNAME name:  _abc123.example.com (same as above)             │
│ CNAME value: _xyz789.acm-validations.aws. (same as above)   │
│ → ⚡ Wildcard uses same CNAME as base domain                 │
│                                                                   │
│ If using Route 53:                                             │
│ [Create records in Route 53] ← ⚡ One click, auto-creates   │
│                                                                   │
│ If using external DNS:                                         │
│ → Copy CNAME records and add to your DNS provider            │
│ → Validation usually completes in 5-30 minutes              │
│ → Can take up to 72 hours in some cases                     │
│                                                                   │
│ Once validated: Status → Issued ✅                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Requesting a Private Certificate

```
┌─────────────────────────────────────────────────────────────────────┐
│           AWS PRIVATE CERTIFICATE AUTHORITY (PCA)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Private CA → Create CA                                   │
│                                                                       │
│ CA type:                                                             │
│ ├── Root CA: Top-level CA (self-signed)                         │
│ └── Subordinate CA: Signed by Root CA (recommended for issuing) │
│                                                                       │
│ Subject:                                                              │
│ Organization: [MyCompany]                                           │
│ Organizational unit: [Engineering]                                  │
│ Country: [US]                                                        │
│ State: [California]                                                  │
│ Locality: [San Francisco]                                           │
│ Common name: [internal.mycompany.com]                              │
│                                                                       │
│ Key algorithm:                                                       │
│ ● RSA 2048                                                          │
│ ○ RSA 4096                                                          │
│ ○ ECDSA P256                                                        │
│ ○ ECDSA P384                                                        │
│                                                                       │
│ Pricing: $400/month per CA + $0.75/cert                             │
│                                                                       │
│ Use cases:                                                           │
│ ├── Service-to-service mTLS (mutual TLS)                        │
│ ├── Internal microservices communication                        │
│ ├── IoT device certificates                                     │
│ ├── Code signing                                                 │
│ └── VPN authentication                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Certificate Association & Renewal

```
┌─────────────────────────────────────────────────────────────────────┐
│           ASSOCIATING CERTIFICATES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ALB:                                                                 │
│ EC2 → Load Balancers → select ALB → Listeners → HTTPS:443        │
│ → Default SSL certificate: [Select ACM cert ▼]                   │
│ → ⚡ Add SNI certs for multiple domains on same ALB              │
│                                                                       │
│ CloudFront:                                                         │
│ CloudFront → Distribution → Settings → Edit                      │
│ → Custom SSL certificate: [Select ACM cert ▼]                    │
│ → ⚠️ Certificate MUST be in us-east-1                            │
│                                                                       │
│ API Gateway:                                                         │
│ API Gateway → Custom domain names → Create                       │
│ → ACM certificate: [Select cert ▼]                                │
│ → Endpoint type: Regional or Edge                                 │
│                                                                       │
│ Automatic renewal:                                                   │
│ ├── DNS-validated: Fully automatic (no action needed)           │
│ ├── Email-validated: Requires manual email approval              │
│ ├── Renewal starts 60 days before expiry                        │
│ ├── Certificate must be in use (associated with resource)       │
│ └── ⚡ DNS validation + Route 53 = zero maintenance             │
│                                                                       │
│ ⚠️ Common renewal failures:                                         │
│ ├── DNS CNAME record was removed                                 │
│ ├── Certificate not associated with any resource                │
│ ├── Email validation: Approval email missed/expired             │
│ └── Fix: Re-validate or request new certificate                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
# Request public certificate
resource "aws_acm_certificate" "main" {
  domain_name               = "example.com"
  subject_alternative_names = ["*.example.com"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

# Auto-create Route 53 validation records
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id = aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}

# Wait for validation
resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# Associate with ALB
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.web.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}
```

```bash
# Request a certificate
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names "*.example.com" \
  --validation-method DNS

# List certificates
aws acm list-certificates --certificate-statuses ISSUED

# Describe certificate
aws acm describe-certificate --certificate-arn arn:aws:acm:us-east-1:123456:certificate/abc-123

# Export certificate (private CA only)
aws acm export-certificate \
  --certificate-arn arn:aws:acm:... \
  --passphrase $(echo -n 'password' | base64)

# Delete certificate
aws acm delete-certificate --certificate-arn arn:aws:acm:...
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Standard Web App (ALB + CloudFront)                     │
│ ├── ACM cert in us-east-1 for CloudFront                        │
│ ├── ACM cert in app region for ALB                               │
│ ├── Domains: example.com + *.example.com                        │
│ ├── DNS validation with Route 53                                │
│ └── Zero maintenance (auto-renewal)                             │
│                                                                       │
│ Pattern 2: Multi-domain on Single ALB (SNI)                        │
│ ├── Default cert: example.com                                    │
│ ├── Additional certs: app1.com, app2.com                        │
│ ├── ALB uses SNI to serve correct cert                          │
│ └── Up to 25 certs per ALB                                       │
│                                                                       │
│ Pattern 3: Internal Microservices (mTLS)                            │
│ ├── Private CA for internal services                             │
│ ├── Each service gets its own certificate                       │
│ ├── Mutual TLS for service-to-service auth                     │
│ └── Short-lived certs (auto-rotated)                            │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Always use DNS validation                                      │
│ 2. Use wildcard + base domain in single cert                     │
│ 3. CloudFront certs → us-east-1 always                           │
│ 4. TLS 1.2+ minimum (security policy)                            │
│ 5. Monitor expiration with CloudWatch alarm                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting: Common ACM Issues

### "Certificate stuck in Pending Validation"

```
1. ☐ Did you add the CNAME validation records?
   Console → ACM → Certificate → Create records in Route 53
   (ACM can auto-create them if using Route 53)

2. ☐ Are the DNS records propagated?
   dig _xxxxx.example.com CNAME
   (May take 5-30 minutes for DNS propagation)

3. ☐ Did you use email validation? Check the domain owner's email
   (DNS validation is recommended — auto-renews)
```

### "Can't use certificate with CloudFront"

```
CloudFront ONLY accepts certificates from us-east-1!

Fix: Create the certificate in us-east-1 specifically.
Console: Switch region to us-east-1 → ACM → Request certificate
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Cert not in us-east-1 for CloudFront | Can't attach to distribution | Always create in us-east-1 for CF |
| Using email validation | Must manually approve renewal | Use DNS validation (auto-renews) |
| Removing CNAME records | Cert fails to auto-renew | Keep validation CNAMEs forever |
| Requesting for wrong domain | Cert doesn't match | Include both `example.com` and `*.example.com` |

---

## Quick Reference

```
ACM Quick Reference:
├── Public certs: FREE, auto-renewal, browser trusted
├── Private certs: $400/month CA + $0.75/cert, internal use
├── Validation: DNS (⚡ recommended) or email
├── Key types: RSA 2048 (default), ECDSA P-256/P-384
├── Wildcard: *.example.com (one level only)
├── SAN: Up to 10 domains per cert (request increase)
├── CloudFront: Certificate MUST be in us-east-1
├── ALB/API GW: Certificate in same region as resource
├── Renewal: Automatic 60 days before expiry (DNS validation)
├── Not for EC2: Use Let's Encrypt or self-managed
├── ⚡ DNS validation + Route 53 = zero maintenance
└── ⚡ Always use create_before_destroy in Terraform
```

---

## What's Next?

In **Chapter 45: SQS (Simple Queue Service)**, we'll cover message queuing, standard vs FIFO queues, dead-letter queues, and asynchronous decoupling patterns.
