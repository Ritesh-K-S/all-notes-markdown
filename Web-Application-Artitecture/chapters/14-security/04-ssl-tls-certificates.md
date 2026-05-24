# SSL/TLS & Certificate Management

> **What you'll learn**: How SSL/TLS encrypts data in transit between clients and servers, how certificates prove a server's identity, and how to properly manage certificates in production.

---

## Real-Life Analogy

Imagine you're sending a **letter** to your bank.

**Without TLS (like HTTP):**
- You write your account number and PIN on a **postcard** — anyone who handles it can read it.
- A mail carrier could copy your details without you knowing.
- Someone could even swap the postcard with a fake reply pretending to be your bank.

**With TLS (like HTTPS):**
1. You and the bank agree on a **secret code** that only you two know (the key exchange).
2. You put the letter in a **locked box** — only the bank has the key to open it.
3. The box has the bank's **official seal** — so you know it really came from them (certificate verification).
4. Even if someone intercepts the box, they can't open it or forge the seal.

```
HTTP (Postcard):                    HTTPS/TLS (Locked Box):
┌─────────┐  plain text  ┌─────┐   ┌─────────┐  encrypted  ┌─────┐
│ Browser │──────────────▶│ ISP │   │ Browser │──🔒────────▶│ ISP │
└─────────┘  "password123"└─────┘   └─────────┘  "x$k#9@..."└─────┘
                              │                                  │
              Anyone can read ▼                  Can't read it!  ▼
              "password123" 😱                   "x$k#9@..." 🤷
```

---

## Core Concept Explained Step-by-Step

### Step 1: What is TLS?

**TLS (Transport Layer Security)** encrypts the communication between two parties so that nobody in between can read or modify the data.

- **SSL** = Secure Sockets Layer (the old name, versions 1.0-3.0, all deprecated)
- **TLS** = Transport Layer Security (the current standard, versions 1.0-1.3)
- When people say "SSL," they usually mean TLS. We use TLS today.

```
┌────────────────────────────────────────────────────────────┐
│                   PROTOCOL HISTORY                           │
├────────────────────────────────────────────────────────────┤
│  SSL 1.0 (1994) — Never released (security flaws)          │
│  SSL 2.0 (1995) — ❌ Deprecated (insecure)                 │
│  SSL 3.0 (1996) — ❌ Deprecated (POODLE attack)            │
│  TLS 1.0 (1999) — ❌ Deprecated (BEAST attack)             │
│  TLS 1.1 (2006) — ❌ Deprecated                            │
│  TLS 1.2 (2008) — ✓ Still widely used                      │
│  TLS 1.3 (2018) — ⭐ Current standard (fastest, safest)    │
└────────────────────────────────────────────────────────────┘
```

### Step 2: What TLS Provides

TLS gives you three guarantees:

```
┌─────────────────────────────────────────────────────────────┐
│                  THREE GUARANTEES OF TLS                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. CONFIDENTIALITY (Encryption)                             │
│     └── Nobody can READ the data in transit                  │
│     └── Uses AES-256, ChaCha20-Poly1305                     │
│                                                              │
│  2. INTEGRITY (Tamper Detection)                             │
│     └── Nobody can MODIFY the data in transit                │
│     └── Uses HMAC, SHA-256                                   │
│                                                              │
│  3. AUTHENTICATION (Identity Verification)                   │
│     └── You're talking to the REAL server                    │
│     └── Uses X.509 certificates                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Step 3: The TLS Handshake — How a Secure Connection is Established

```
┌────────┐                                          ┌────────┐
│ Client │                                          │ Server │
│(Browser)│                                          │        │
└───┬────┘                                          └───┬────┘
    │                                                   │
    │ 1. CLIENT HELLO                                   │
    │    "Hi! I support TLS 1.3, these ciphers..."      │
    │──────────────────────────────────────────────────▶│
    │                                                   │
    │ 2. SERVER HELLO                                   │
    │    "Let's use TLS 1.3 with AES-256-GCM"          │
    │    + Server Certificate (public key)              │
    │    + Key Share (for key exchange)                 │
    │◀──────────────────────────────────────────────────│
    │                                                   │
    │ 3. Client verifies certificate:                   │
    │    - Is it signed by a trusted CA?                │
    │    - Is it for this domain?                       │
    │    - Has it expired?                              │
    │    - Has it been revoked?                         │
    │                                                   │
    │ 4. KEY EXCHANGE                                   │
    │    Both sides compute shared secret               │
    │    (using Diffie-Hellman or ECDHE)                │
    │──────────────────────────────────────────────────▶│
    │                                                   │
    │ ═══════════════════════════════════════════════════│
    │ From here, ALL data is encrypted with             │
    │ the shared secret key!                            │
    │ ═══════════════════════════════════════════════════│
    │                                                   │
    │ 5. Encrypted HTTP Request                         │
    │    GET /api/data                                  │
    │──────────────────────────────────────────────────▶│
    │                                                   │
    │ 6. Encrypted HTTP Response                        │
    │    {"data": "secret stuff"}                       │
    │◀──────────────────────────────────────────────────│
```

### Step 4: Certificates — How Identity is Proven

A **certificate** is a digital document that says: "This public key belongs to example.com, and we (the Certificate Authority) vouch for it."

```
┌─────────────────────────────────────────────────────────────┐
│              X.509 CERTIFICATE STRUCTURE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Subject:    CN=www.example.com                              │
│  Issuer:     CN=Let's Encrypt Authority X3                   │
│  Valid From:  2024-01-01                                     │
│  Valid To:    2024-04-01 (90 days for Let's Encrypt)         │
│  Public Key:  RSA 2048-bit or ECDSA P-256                    │
│  SANs:        example.com, www.example.com, api.example.com  │
│  Signature:   (signed by issuer's private key)               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Step 5: Certificate Chain of Trust

```
┌───────────────────┐
│   ROOT CA         │  Built into your OS/browser
│  (DigiCert,       │  Self-signed, ultra-trusted
│   Let's Encrypt)  │  Never changes (10-30 year validity)
└────────┬──────────┘
         │ signs
         ▼
┌───────────────────┐
│ INTERMEDIATE CA   │  Issued by Root CA
│ (Let's Encrypt    │  Signs end-entity certificates
│  Authority X3)    │  3-10 year validity
└────────┬──────────┘
         │ signs
         ▼
┌───────────────────┐
│ YOUR CERTIFICATE  │  For your domain (example.com)
│ (End Entity/Leaf) │  Short validity (90 days - 1 year)
│                   │  This is what your server presents
└───────────────────┘

Browser verifies: Leaf → Intermediate → Root (trusted?)
If the chain reaches a trusted Root CA → ✓ VALID
```

---

## How It Works Internally

### TLS 1.3 Handshake (Faster than 1.2)

TLS 1.3 requires only **1 round trip** (1-RTT) to establish a connection, vs 2 for TLS 1.2:

```
TLS 1.2 (2-RTT):                    TLS 1.3 (1-RTT):
┌────────┐      ┌────────┐          ┌────────┐      ┌────────┐
│ Client │      │ Server │          │ Client │      │ Server │
└───┬────┘      └───┬────┘          └───┬────┘      └───┬────┘
    │ ClientHello   │                    │ ClientHello   │
    │──────────────▶│ RTT 1              │ + KeyShare    │
    │ ServerHello   │                    │──────────────▶│ RTT 1
    │◀──────────────│                    │ ServerHello   │
    │ Key Exchange  │                    │ + Certificate │
    │──────────────▶│ RTT 2              │ + Finished    │
    │ Finished      │                    │◀──────────────│
    │◀──────────────│                    │ Finished      │
    │               │                    │──────────────▶│
    │ Data ─────────│                    │ Data ─────────│
    │  Total: ~200ms│                    │  Total: ~100ms│
```

### 0-RTT Resumption (TLS 1.3)

If you've connected before, TLS 1.3 can send data on the **first packet**:

```
Returning visitor (0-RTT):
┌────────┐                              ┌────────┐
│ Client │                              │ Server │
└───┬────┘                              └───┬────┘
    │ ClientHello + Pre-shared Key          │
    │ + Early Data (encrypted!)             │
    │──────────────────────────────────────▶│
    │                                       │ Can process request
    │ ServerHello + Finished                │ immediately!
    │◀──────────────────────────────────────│
    │                                       │
    │ (Connection fully established)        │
    
⚠️ 0-RTT is vulnerable to replay attacks — only safe for 
   idempotent requests (GET, not POST with payments!)
```

### Certificate Validation — What the Browser Checks

```
┌────────────────────────────────────────────────────────────┐
│            CERTIFICATE VALIDATION STEPS                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  1. ✓ Chain of trust                                        │
│     └── Does it chain to a trusted Root CA?                 │
│                                                             │
│  2. ✓ Domain match                                          │
│     └── Does cert's CN or SAN match the requested domain?   │
│     └── *.example.com matches sub.example.com               │
│     └── But NOT sub.sub.example.com                         │
│                                                             │
│  3. ✓ Validity period                                       │
│     └── Is current time between Not Before and Not After?   │
│                                                             │
│  4. ✓ Revocation status                                     │
│     └── CRL (Certificate Revocation List)                   │
│     └── OCSP (Online Certificate Status Protocol)           │
│     └── OCSP Stapling (faster, server provides proof)       │
│                                                             │
│  5. ✓ Key usage                                             │
│     └── Is the certificate authorized for TLS?              │
│                                                             │
│  If ANY check fails → Browser shows "Not Secure" warning    │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Creating a Self-Signed Certificate

```python
# generate_cert.py — Create self-signed cert for development
from cryptography import x509
from cryptography.x509.oid import NameOID
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa
import datetime

# Generate private key
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048,
)

# Build certificate
subject = issuer = x509.Name([
    x509.NameAttribute(NameOID.COUNTRY_NAME, "US"),
    x509.NameAttribute(NameOID.ORGANIZATION_NAME, "My Dev Server"),
    x509.NameAttribute(NameOID.COMMON_NAME, "localhost"),
])

cert = (
    x509.CertificateBuilder()
    .subject_name(subject)
    .issuer_name(issuer)
    .public_key(private_key.public_key())
    .serial_number(x509.random_serial_number())
    .not_valid_before(datetime.datetime.utcnow())
    .not_valid_after(datetime.datetime.utcnow() + datetime.timedelta(days=365))
    .add_extension(
        x509.SubjectAlternativeName([
            x509.DNSName("localhost"),
            x509.DNSName("*.local.dev"),
            x509.IPAddress(ipaddress.IPv4Address("127.0.0.1")),
        ]),
        critical=False,
    )
    .sign(private_key, hashes.SHA256())
)

# Save to files
with open("server.key", "wb") as f:
    f.write(private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.TraditionalOpenSSL,
        encryption_algorithm=serialization.NoEncryption(),
    ))

with open("server.crt", "wb") as f:
    f.write(cert.public_bytes(serialization.Encoding.PEM))

print("✓ Certificate and key generated!")
```

### Python — HTTPS Server

```python
# https_server.py — Flask with TLS
from flask import Flask
import ssl

app = Flask(__name__)

@app.route("/")
def hello():
    return {"message": "This is encrypted!"}

if __name__ == "__main__":
    # Create SSL context
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    context.load_cert_chain("server.crt", "server.key")
    
    # Enforce TLS 1.2+
    context.minimum_version = ssl.TLSVersion.TLSv1_2
    
    app.run(host="0.0.0.0", port=443, ssl_context=context)
```

### Java — HTTPS with Spring Boot

```java
// application.yml — Enable HTTPS in Spring Boot
// server:
//   port: 8443
//   ssl:
//     key-store: classpath:keystore.p12
//     key-store-password: ${SSL_KEYSTORE_PASSWORD}
//     key-store-type: PKCS12
//     protocol: TLS
//     enabled-protocols: TLSv1.3,TLSv1.2

// HttpsConfig.java — Redirect HTTP to HTTPS
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class HttpsConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Force all requests to use HTTPS
            .requiresChannel(channel -> channel
                .anyRequest().requiresSecure()
            )
            // HSTS header — browser remembers to always use HTTPS
            .headers(headers -> headers
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)  // 1 year
                    .includeSubDomains(true)
                )
            );
        return http.build();
    }
}
```

### Python — Verifying a Certificate Programmatically

```python
# verify_cert.py — Check a website's certificate
import ssl
import socket
from datetime import datetime

def check_certificate(hostname: str, port: int = 443):
    """Check a server's TLS certificate details"""
    context = ssl.create_default_context()
    
    with socket.create_connection((hostname, port)) as sock:
        with context.wrap_socket(sock, server_hostname=hostname) as ssock:
            cert = ssock.getpeercert()
            
            # Certificate details
            print(f"Subject: {dict(x[0] for x in cert['subject'])}")
            print(f"Issuer: {dict(x[0] for x in cert['issuer'])}")
            print(f"Valid from: {cert['notBefore']}")
            print(f"Valid until: {cert['notAfter']}")
            print(f"SANs: {[x[1] for x in cert.get('subjectAltName', [])]}")
            print(f"TLS version: {ssock.version()}")
            print(f"Cipher: {ssock.cipher()}")
            
            # Check expiry
            expiry = datetime.strptime(cert['notAfter'], '%b %d %H:%M:%S %Y %Z')
            days_left = (expiry - datetime.utcnow()).days
            print(f"Days until expiry: {days_left}")
            if days_left < 30:
                print("⚠️  WARNING: Certificate expiring soon!")

check_certificate("google.com")
```

---

## Infrastructure Examples

### Nginx — TLS Configuration (Production-Grade)

```nginx
# /etc/nginx/conf.d/tls.conf
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # Certificate files
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # TLS protocol versions (only 1.2 and 1.3)
    ssl_protocols TLSv1.2 TLSv1.3;

    # Strong cipher suites
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;  # Let client choose (TLS 1.3)

    # OCSP Stapling (faster cert validation)
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;

    # Session caching (avoid re-handshakes)
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;  # Better security

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;

    location / {
        proxy_pass http://backend:8080;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

### Let's Encrypt — Free Automated Certificates

```bash
# Install certbot and get a certificate
sudo apt install certbot python3-certbot-nginx

# Obtain certificate (auto-configures Nginx)
sudo certbot --nginx -d example.com -d www.example.com

# Auto-renewal (add to crontab)
# Certificates expire every 90 days — certbot renews at 60 days
echo "0 3 * * * certbot renew --quiet --post-hook 'systemctl reload nginx'" | sudo crontab -
```

### Kubernetes — TLS with cert-manager

```yaml
# cert-manager automatically provisions and renews certificates
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-cert  # cert-manager creates this automatically
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

---

## Real-World Example

### How Cloudflare Handles TLS at Scale

```
┌────────┐          ┌─────────────┐          ┌────────────┐
│ Client │──TLS────▶│ Cloudflare  │──TLS────▶│ Your Server│
│(Browser)│  1.3     │   Edge      │  1.2/1.3  │ (Origin)   │
└────────┘          └─────────────┘          └────────────┘
                          │
                    Handles:
                    - TLS termination
                    - Certificate management
                    - OCSP stapling
                    - HTTP/2, HTTP/3
                    - DDoS protection

Modes:
  "Flexible"  = Client→Cloudflare: HTTPS, Cloudflare→Origin: HTTP (not recommended)
  "Full"      = Client→Cloudflare: HTTPS, Cloudflare→Origin: HTTPS (any cert)
  "Full Strict" = Both HTTPS with valid certs (recommended)
```

### How Let's Encrypt Issues 3+ Million Certs/Day

- Fully automated via **ACME protocol**
- Certificate validity: **90 days** (encourages automation)
- Supports both HTTP-01 (prove domain ownership via HTTP) and DNS-01 (via DNS record)
- Used by over 300 million websites

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| Self-signed certs in production | Browser warnings scare users away | Use Let's Encrypt (free!) |
| Forgetting to redirect HTTP→HTTPS | Users access site unencrypted | Add 301 redirect + HSTS |
| Expired certificates | Site becomes unreachable | Automate renewal with certbot |
| Using TLS 1.0/1.1 | Known vulnerabilities | Only allow TLS 1.2+ |
| Weak cipher suites | Vulnerable to downgrade attacks | Use recommended cipher list |
| Storing private keys in code repos | Key compromise | Use secrets management |
| Mixed content (HTTP resources on HTTPS page) | Browser blocks resources | Audit all resource URLs |
| Not implementing HSTS | First visit can be intercepted | Add HSTS header |

---

## When to Use / When NOT to Use

### Always Use TLS For:
- **Everything.** There's no valid reason to use unencrypted HTTP in production anymore.
- APIs (REST, GraphQL, gRPC)
- Websites (even blogs — Google ranks HTTPS higher)
- Internal service-to-service communication (defense in depth)
- Database connections (PostgreSQL, MongoDB support TLS)

### mTLS (Mutual TLS) — When Both Sides Need Certificates:
- Service-to-service communication in microservices
- Zero-trust network architecture (Chapter 14.10)
- IoT device authentication
- Financial/healthcare compliance

### Certificate Type Decision:

| Need | Certificate Type |
|------|-----------------|
| Simple website/API | Domain Validated (DV) — Let's Encrypt |
| Business with validation | Organization Validated (OV) |
| Banks, e-commerce (green bar) | Extended Validation (EV) |
| Multiple subdomains | Wildcard (*.example.com) |
| Service-to-service | Self-signed or internal CA |

---

## Key Takeaways

- **TLS** encrypts data in transit — providing confidentiality, integrity, and authentication
- **TLS 1.3** is the current standard — faster (1-RTT) and more secure than 1.2
- **Certificates** prove server identity — verified via a chain of trust to a Root CA
- **Let's Encrypt** provides free, automated certificates — there's no excuse for HTTP
- **Always redirect** HTTP to HTTPS and use **HSTS** headers
- **Automate certificate renewal** — expired certs cause outages
- The private key must **never** leave the server or be committed to version control

---

## What's Next?

Next, we'll explore the **OWASP Top 10** — the ten most common and dangerous security vulnerabilities in web applications, from SQL injection to broken access control. That's Chapter 14.5.
