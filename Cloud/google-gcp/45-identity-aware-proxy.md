# Chapter 45 — Identity-Aware Proxy (IAP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: IAP Fundamentals](#part-1--iap-fundamentals)
- [Part 2: How IAP Works — Architecture](#part-2--how-iap-works--architecture)
- [Part 3: IAP for Web Applications](#part-3--iap-for-web-applications)
- [Part 4: IAP for Compute Engine & GKE](#part-4--iap-for-compute-engine--gke)
- [Part 5: IAP TCP Forwarding (SSH & RDP Tunnels)](#part-5--iap-tcp-forwarding-ssh--rdp-tunnels)
- [Part 6: IAP Access Levels & Context-Aware Access](#part-6--iap-access-levels--context-aware-access)
- [Part 7: OAuth Consent Screen & Credentials](#part-7--oauth-consent-screen--credentials)
- [Part 8: Programmatic Access & Service Accounts](#part-8--programmatic-access--service-accounts)
- [Part 9: Custom Headers & User Identity](#part-9--custom-headers--user-identity)
- [Part 10: IAP with Cloud Run & App Engine](#part-10--iap-with-cloud-run--app-engine)
- [Part 11: IAP with On-Premises Applications](#part-11--iap-with-on-premises-applications)
- [Part 12: IAM & Security](#part-12--iam--security)
- [Part 13: Monitoring & Audit Logging](#part-13--monitoring--audit-logging)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Identity-Aware Proxy (IAP) is a Google Cloud service that controls access to web applications and VMs based on user identity and request context, rather than relying on traditional network-level controls like VPNs. IAP verifies user identity and checks access policies before allowing access, enabling a zero-trust "BeyondCorp" security model where access decisions are based on who you are and the context of your request, not which network you're on.

---

## Part 1 — IAP Fundamentals

### What Is Identity-Aware Proxy?

```
┌─────────────────────────────────────────────────────────────────┐
│              IDENTITY-AWARE PROXY OVERVIEW                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Traditional approach (VPN):                                     │
│  ┌──────────┐  VPN  ┌────────────┐      ┌────────────┐        │
│  │ Employee │───────│ Corp       │─────│ Internal   │        │
│  │ (laptop) │       │ Network    │      │ App        │        │
│  └──────────┘       └────────────┘      └────────────┘        │
│  Problem: Anyone on the network can access internal apps.      │
│  VPN = single point of trust. If VPN is compromised → game over│
│                                                                   │
│  IAP approach (zero trust):                                      │
│  ┌──────────┐       ┌────────────┐      ┌────────────┐        │
│  │ Employee │──────►│ IAP        │─────│ Internal   │        │
│  │ (laptop, │ HTTPS │ (identity  │      │ App        │        │
│  │  anywhere)│      │  + context │      └────────────┘        │
│  └──────────┘       │  check)    │                             │
│                     └────────────┘                             │
│  No VPN needed. Access based on:                                │
│  ✓ Who you are (Google identity)                                │
│  ✓ What device you're using (managed? encrypted?)              │
│  ✓ Where you are (IP, region)                                  │
│  ✓ What you're accessing (specific app/resource)               │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  IAP protects:                                            │   │
│  │  • Web applications (behind HTTPS LB)                    │   │
│  │  • App Engine apps                                        │   │
│  │  • Cloud Run services                                     │   │
│  │  • VM SSH/RDP access (TCP forwarding)                    │   │
│  │  • GKE workloads (via Ingress)                           │   │
│  │  • On-premises apps (via IAP connector)                  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP IAP | AWS Verified Access | Azure App Proxy |
|---------|---------|---------------------|-----------------|
| Service | Identity-Aware Proxy | AWS Verified Access | Azure AD App Proxy |
| Zero-trust model | Yes (BeyondCorp) | Yes | Yes |
| Web app protection | Yes | Yes | Yes |
| TCP tunneling | Yes (SSH, RDP) | No (web only) | No (use Bastion) |
| Identity provider | Google Identity / Workforce Identity | AWS IAM Identity Center | Azure AD |
| Device trust | Yes (BeyondCorp, Endpoint Verification) | Yes (via device posture) | Yes (Conditional Access) |
| Context-aware access | Yes (access levels) | Yes (verified access policies) | Yes (Conditional Access) |
| On-prem apps | Yes (IAP connector) | No | Yes (App Proxy connector) |
| Pricing | Free (IAP itself) | $0.27/user/month | Included with Azure AD P1 |
| No VPN needed | Yes | Yes | Yes |

### Pricing

| Component | Cost |
|-----------|------|
| IAP (web app protection) | Free |
| IAP TCP forwarding | Free |
| Access Context Manager | Free |
| BeyondCorp Enterprise (advanced) | $6/user/month |
| OAuth consent screen | Free |

---

## Part 2 — How IAP Works — Architecture

### Request Flow

```
┌──────────────────────────────────────────────────────────────┐
│         IAP REQUEST FLOW                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  User makes request to protected application:                 │
│                                                                │
│  ┌──────────┐                                                │
│  │ User     │─── HTTPS ──► https://app.example.com           │
│  │ (browser)│                    │                             │
│  └──────────┘                    ▼                             │
│                          ┌──────────────┐                    │
│                          │ Cloud LB     │                    │
│                          │ (HTTPS proxy)│                    │
│                          └──────┬───────┘                    │
│                                 │                             │
│                                 ▼                             │
│                          ┌──────────────┐                    │
│  Step 1:                │ IAP          │                    │
│  Is user                │              │                    │
│  authenticated? ◄───────│ Check:       │                    │
│  NO → redirect to      │ 1. Identity  │                    │
│       Google login      │ 2. IAM role  │                    │
│                         │ 3. Access    │                    │
│  Step 2:                │    Level     │                    │
│  Does user have         │              │                    │
│  IAP role?              └──────┬───────┘                    │
│  NO → 403 Forbidden           │                             │
│                                │ ALL checks pass             │
│  Step 3:                       ▼                             │
│  Does context meet      ┌──────────────┐                    │
│  access level?          │ Backend      │                    │
│  NO → 403 Forbidden    │ Service      │                    │
│                         │              │                    │
│  ALL PASS:             │ Headers:     │                    │
│  → Forward request     │ X-Goog-      │                    │
│    with identity       │ Authenticated│                    │
│    headers             │ -User-Email  │                    │
│                         └──────────────┘                    │
│                                                                │
│  Key: IAP sits between the LB and your backend.             │
│  Your app NEVER receives unauthenticated requests.           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### IAP for TCP Forwarding (SSH/RDP)

```
┌──────────────────────────────────────────────────────────────┐
│         IAP TCP FORWARDING                                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────┐                    ┌──────────────┐           │
│  │ Admin    │── gcloud compute ──│ IAP Tunnel   │           │
│  │ (laptop) │   ssh VM_NAME     │ Proxy        │           │
│  │          │   --tunnel-through │              │           │
│  │          │   -iap             │ Authenticates│           │
│  └──────────┘                    │ + authorizes │           │
│       │                          └──────┬───────┘           │
│       │ IAP tunnel                      │                    │
│       │ (WebSocket over HTTPS)         │ TCP                │
│       │                                 ▼                    │
│       │                          ┌──────────────┐           │
│       │                          │ VM Instance  │           │
│       │                          │ (no public   │           │
│       └─────────────────────────►│  IP needed!) │           │
│                                  └──────────────┘           │
│                                                                │
│  Benefits:                                                     │
│  • VMs don't need public IPs                                 │
│  • No VPN or bastion host needed                             │
│  • Audit log of every SSH/RDP session                        │
│  • IAM controls who can connect to which VM                  │
│  • Context-aware (device trust, IP range)                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 3 — IAP for Web Applications

### Enabling IAP for a Web App

```bash
# Prerequisites:
# 1. App behind an HTTPS Load Balancer (external Application LB)
# 2. OAuth consent screen configured
# 3. Backend service exists

# Enable IAP on a backend service
gcloud compute backend-services update my-backend-service \
    --iap=enabled \
    --global

# Grant IAP access to users
gcloud projects add-iam-policy-binding my-project \
    --member="user:alice@example.com" \
    --role="roles/iap.httpsResourceAccessor"

# Grant IAP access to a group
gcloud projects add-iam-policy-binding my-project \
    --member="group:engineering@example.com" \
    --role="roles/iap.httpsResourceAccessor"

# Grant access to specific backend service only
gcloud iap web add-iam-policy-binding \
    --resource-type=backend-services \
    --service=my-backend-service \
    --member="user:alice@example.com" \
    --role="roles/iap.httpsResourceAccessor"
```

### User Experience

```
1. User navigates to https://app.example.com
2. IAP redirects to Google Sign-In (if not already signed in)
3. User authenticates with Google account
4. IAP checks:
   a. Does user have roles/iap.httpsResourceAccessor?
   b. Does request context match access levels?
5. If authorized → app loads normally
6. If not → 403 page ("You don't have access")
```

---

## Part 4 — IAP for Compute Engine & GKE

### IAP for Compute Engine (Web)

```bash
# For VMs behind a load balancer serving web traffic:
# Same as Part 3 — enable IAP on the backend service

# For VM-to-VM traffic or admin interfaces:
# Use IAP TCP Forwarding (Part 5)
```

### IAP for GKE

```
┌──────────────────────────────────────────────────────────────┐
│         IAP FOR GKE                                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  GKE apps behind an Ingress with IAP enabled:                │
│                                                                │
│  Internet                                                      │
│    │                                                           │
│    ▼                                                           │
│  ┌──────────────────────────────────┐                        │
│  │ GKE Ingress (HTTPS LB)          │                        │
│  │ ├── IAP enabled                  │                        │
│  │ └── Backend: GKE Service         │                        │
│  └──────────────────────────────────┘                        │
│    │                                                           │
│    ▼                                                           │
│  ┌──────────────────────────────────┐                        │
│  │ GKE Pods                         │                        │
│  │ (receive X-Goog-Authenticated-*  │                        │
│  │  headers with user identity)     │                        │
│  └──────────────────────────────────┘                        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```yaml
# GKE Ingress with IAP enabled
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: iap-config
spec:
  iap:
    enabled: true
    oauthclientCredentials:
      secretName: iap-oauth-secret
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    cloud.google.com/backend-config: '{"default": "iap-config"}'
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
---
# OAuth secret (create from Console credentials)
kubectl create secret generic iap-oauth-secret \
    --from-literal=client_id=CLIENT_ID \
    --from-literal=client_secret=CLIENT_SECRET
```

---

## Part 5 — IAP TCP Forwarding (SSH & RDP Tunnels)

### SSH via IAP Tunnel

```bash
# SSH to a VM via IAP (no public IP needed!)
gcloud compute ssh VM_NAME \
    --zone=us-central1-a \
    --tunnel-through-iap

# SSH to a specific internal IP
gcloud compute ssh VM_NAME \
    --zone=us-central1-a \
    --tunnel-through-iap \
    --internal-ip

# SCP (file transfer) via IAP
gcloud compute scp LOCAL_FILE VM_NAME:~/remote-path \
    --zone=us-central1-a \
    --tunnel-through-iap

# Custom port forwarding via IAP
gcloud compute start-iap-tunnel VM_NAME 8080 \
    --local-host-port=localhost:8080 \
    --zone=us-central1-a
# Then access http://localhost:8080 → tunneled to VM:8080
```

### RDP via IAP Tunnel

```bash
# Create IAP tunnel for RDP (Windows VM)
gcloud compute start-iap-tunnel WINDOWS_VM 3389 \
    --local-host-port=localhost:3389 \
    --zone=us-central1-a

# Then connect with Remote Desktop client to localhost:3389
```

### IAM for TCP Forwarding

```bash
# Grant SSH access via IAP
gcloud projects add-iam-policy-binding my-project \
    --member="user:alice@example.com" \
    --role="roles/iap.tunnelResourceAccessor"

# Grant access to specific VM only
gcloud compute instances add-iam-policy-binding VM_NAME \
    --zone=us-central1-a \
    --member="user:alice@example.com" \
    --role="roles/iap.tunnelResourceAccessor"

# Also need compute.instances.setMetadata for SSH key management
# (or use OS Login instead)
gcloud projects add-iam-policy-binding my-project \
    --member="user:alice@example.com" \
    --role="roles/compute.instanceAdmin.v1"
```

### Firewall Rules for IAP

```bash
# Allow IAP TCP forwarding through firewall
# IAP uses IP range 35.235.240.0/20
gcloud compute firewall-rules create allow-iap-ssh \
    --network=my-vpc \
    --allow=tcp:22 \
    --source-ranges=35.235.240.0/20 \
    --description="Allow SSH from IAP"

gcloud compute firewall-rules create allow-iap-rdp \
    --network=my-vpc \
    --allow=tcp:3389 \
    --source-ranges=35.235.240.0/20 \
    --description="Allow RDP from IAP"
```

---

## Part 6 — IAP Access Levels & Context-Aware Access

### Context-Aware Access

```
┌──────────────────────────────────────────────────────────────┐
│         CONTEXT-AWARE ACCESS                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Combine IAP with Access Context Manager access levels       │
│  (same access levels used by VPC Service Controls):          │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  IAP checks:                                         │    │
│  │                                                      │    │
│  │  1. Identity: Is user authenticated? ✓/✗            │    │
│  │  2. Authorization: Has IAP role? ✓/✗                │    │
│  │  3. Context (access level):                          │    │
│  │     ├── IP address in allowed range? ✓/✗            │    │
│  │     ├── Device managed by org? ✓/✗                  │    │
│  │     ├── Disk encrypted? ✓/✗                         │    │
│  │     ├── Screen lock enabled? ✓/✗                    │    │
│  │     └── OS up to date? ✓/✗                          │    │
│  │                                                      │    │
│  │  ALL must pass → access granted                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Example policies:                                             │
│  • Admin panel: only from managed devices on corp network   │
│  • Internal wiki: any device, any network (identity only)   │
│  • Production ops: managed device + US/EU region only       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Configuring Context-Aware Access

```bash
# Create access level (reuses Access Context Manager)
cat > managed-device.yaml << 'EOF'
- devicePolicy:
    requireScreenlock: true
    allowedEncryptionStatuses:
      - ENCRYPTED
    allowedDeviceManagementLevels:
      - COMPLETE
EOF

gcloud access-context-manager levels create managed-device \
    --policy=POLICY_ID \
    --title="Managed Device" \
    --basic-level-spec=managed-device.yaml

# Apply access level to IAP resource
gcloud iap web add-iam-policy-binding \
    --resource-type=backend-services \
    --service=admin-backend \
    --member="group:admins@example.com" \
    --role="roles/iap.httpsResourceAccessor" \
    --condition="expression=accessPolicies/POLICY_ID/accessLevels/managed-device,title=Managed Device Required"
```

---

## Part 7 — OAuth Consent Screen & Credentials

### Setting Up OAuth

```
┌──────────────────────────────────────────────────────────────┐
│         OAUTH SETUP FOR IAP                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  IAP uses OAuth 2.0 for user authentication.                 │
│  You must configure:                                          │
│                                                                │
│  1. OAuth Consent Screen (once per project):                 │
│     Console → APIs & Services → OAuth consent screen         │
│     ┌──────────────────────────────────────────────────┐    │
│     │  User type: Internal (org users only)             │    │
│     │  App name: "My Company Portal"                    │    │
│     │  Support email: admin@example.com                 │    │
│     │  Authorized domains: example.com                  │    │
│     └──────────────────────────────────────────────────┘    │
│                                                                │
│  2. OAuth Credentials (per IAP-protected resource):          │
│     Console → APIs & Services → Credentials                  │
│     → Create OAuth 2.0 Client ID                            │
│     ┌──────────────────────────────────────────────────┐    │
│     │  Application type: Web application                │    │
│     │  Name: "IAP Client"                               │    │
│     │  Authorized redirect URI:                         │    │
│     │    https://iap.googleapis.com/v1/oauth/           │    │
│     │    clientIds/CLIENT_ID:handleRedirect             │    │
│     └──────────────────────────────────────────────────┘    │
│                                                                │
│  3. Enable IAP on backend service with OAuth client ID       │
│                                                                │
│  For GKE: store client_id and client_secret in K8s secret   │
│  For backend services: configure via gcloud or Console       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 8 — Programmatic Access & Service Accounts

### Accessing IAP-Protected Resources Programmatically

```python
# Python — access IAP-protected web app with service account
import google.auth
import google.auth.transport.requests
from google.auth import impersonated_credentials
import requests

def make_iap_request(url, client_id):
    """Make a request to an IAP-protected resource."""
    
    # Get default credentials
    creds, project = google.auth.default()
    
    # Create credentials for IAP
    from google.oauth2 import id_token
    from google.auth.transport.requests import Request
    
    # Get an ID token targeting the IAP client ID
    open_id_connect_token = id_token.fetch_id_token(
        Request(), client_id
    )
    
    # Make the request with the ID token
    response = requests.get(
        url,
        headers={
            "Authorization": f"Bearer {open_id_connect_token}"
        }
    )
    
    return response.text

# Usage
result = make_iap_request(
    "https://app.example.com/api/data",
    "CLIENT_ID.apps.googleusercontent.com"
)
```

```bash
# Using curl with service account (for testing)
TOKEN=$(gcloud auth print-identity-token \
    --audiences="CLIENT_ID.apps.googleusercontent.com")

curl -H "Authorization: Bearer $TOKEN" \
    https://app.example.com/api/data
```

### Service Account Access

```bash
# Grant a service account IAP access
gcloud iap web add-iam-policy-binding \
    --resource-type=backend-services \
    --service=my-backend-service \
    --member="serviceAccount:cicd@my-project.iam.gserviceaccount.com" \
    --role="roles/iap.httpsResourceAccessor"
```

---

## Part 9 — Custom Headers & User Identity

### IAP Identity Headers

```
┌──────────────────────────────────────────────────────────────┐
│         IAP IDENTITY HEADERS                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  When IAP forwards a request to your backend, it adds        │
│  headers with the authenticated user's identity:              │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  X-Goog-Authenticated-User-Email                     │    │
│  │  → accounts.google.com:alice@example.com             │    │
│  │                                                      │    │
│  │  X-Goog-Authenticated-User-Id                       │    │
│  │  → accounts.google.com:1234567890                    │    │
│  │                                                      │    │
│  │  X-Goog-Iap-Jwt-Assertion                           │    │
│  │  → Signed JWT with full identity claims              │    │
│  │    (most secure — verify the signature!)             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ⚠ SECURITY WARNING:                                         │
│  The email/ID headers can be spoofed by direct backend      │
│  access (bypassing IAP). ALWAYS verify the JWT instead:     │
│                                                                │
│  1. Verify the JWT signature (using Google's public keys)   │
│  2. Check the audience matches your app                     │
│  3. Check the issuer is https://cloud.google.com/iap       │
│  4. THEN read the email claim                               │
│                                                                │
│  Or: ensure backend is ONLY reachable via the LB            │
│  (use firewall rules to block direct access)                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Verifying the IAP JWT

```python
# Python — verify IAP JWT in your backend
from google.auth.transport import requests
from google.oauth2 import id_token

def verify_iap_jwt(iap_jwt, expected_audience):
    """Verify an IAP JWT and return the user's email."""
    try:
        decoded_jwt = id_token.verify_token(
            iap_jwt,
            requests.Request(),
            audience=expected_audience,
            certs_url="https://www.gstatic.com/iap/verify/public_key",
        )
        return decoded_jwt["email"]
    except Exception as e:
        print(f"JWT verification failed: {e}")
        return None

# In your Flask app
@app.route("/api/data")
def get_data():
    jwt = request.headers.get("X-Goog-Iap-Jwt-Assertion")
    if not jwt:
        return "Unauthorized", 401
    
    email = verify_iap_jwt(jwt, "/projects/PROJECT_NUM/apps/PROJECT_ID")
    if not email:
        return "Invalid token", 403
    
    # email is the verified identity of the user
    return f"Hello, {email}"
```

---

## Part 10 — IAP with Cloud Run & App Engine

### IAP for App Engine

```bash
# Enable IAP for App Engine (entire app)
gcloud iap web enable \
    --resource-type=app-engine \
    --oauth2-client-id=CLIENT_ID \
    --oauth2-client-secret=CLIENT_SECRET

# Grant access
gcloud iap web add-iam-policy-binding \
    --resource-type=app-engine \
    --member="group:engineering@example.com" \
    --role="roles/iap.httpsResourceAccessor"
```

### IAP for Cloud Run

```
┌──────────────────────────────────────────────────────────────┐
│         IAP FOR CLOUD RUN                                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Cloud Run doesn't natively support IAP directly.            │
│  Use one of these approaches:                                 │
│                                                                │
│  Option 1: Cloud Run behind HTTPS LB + IAP                   │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Internet → HTTPS LB → IAP → Serverless NEG →       │    │
│  │  Cloud Run (--ingress=internal-and-cloud-load-       │    │
│  │            balancing)                                 │    │
│  └──────────────────────────────────────────────────────┘    │
│  → IAP enabled on the backend service of the LB             │
│  → Cloud Run only accepts traffic from the LB                │
│                                                                │
│  Option 2: Cloud Run with IAM authentication                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Cloud Run with --no-allow-unauthenticated            │    │
│  │  → Clients must present a valid ID token             │    │
│  │  → No Google Sign-In UI (programmatic only)          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Cloud Run behind LB with IAP
# 1. Create Cloud Run service (internal only)
gcloud run deploy my-app \
    --image=gcr.io/my-project/my-app:latest \
    --ingress=internal-and-cloud-load-balancing \
    --no-allow-unauthenticated

# 2. Create serverless NEG
gcloud compute network-endpoint-groups create my-neg \
    --region=us-central1 \
    --network-endpoint-type=serverless \
    --cloud-run-service=my-app

# 3. Create backend service + LB (see Ch11)
# 4. Enable IAP on backend service
gcloud compute backend-services update my-backend \
    --iap=enabled --global
```

---

## Part 11 — IAP with On-Premises Applications

### IAP Connector for On-Prem

```
┌──────────────────────────────────────────────────────────────┐
│         IAP FOR ON-PREMISES APPLICATIONS                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  IAP can protect on-premises web apps without moving them    │
│  to GCP using the IAP connector:                              │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                                                      │    │
│  │  Internet                                            │    │
│  │    │                                                 │    │
│  │    ▼                                                 │    │
│  │  ┌──────────────────────┐                           │    │
│  │  │ Google Cloud LB +    │                           │    │
│  │  │ IAP                  │                           │    │
│  │  └──────────┬───────────┘                           │    │
│  │             │                                        │    │
│  │             ▼                                        │    │
│  │  ┌──────────────────────┐                           │    │
│  │  │ IAP Connector        │  (runs on GCE VM or GKE  │    │
│  │  │ (Envoy-based)        │   inside your VPC)        │    │
│  │  └──────────┬───────────┘                           │    │
│  │             │ VPN / Interconnect                     │    │
│  │             ▼                                        │    │
│  │  ┌──────────────────────┐                           │    │
│  │  │ On-Prem Application  │                           │    │
│  │  │ (internal web app)   │                           │    │
│  │  └──────────────────────┘                           │    │
│  │                                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Use cases:                                                    │
│  • Legacy intranet apps without modern auth                  │
│  • ERP / CRM systems (SAP, internal tools)                  │
│  • Gradual migration to zero trust                           │
│  • Replace VPN access to internal tools                      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 12 — IAM & Security

### IAM Roles

| Role | Description | Use For |
|------|-------------|---------|
| `roles/iap.httpsResourceAccessor` | Access IAP-protected web apps | End users |
| `roles/iap.tunnelResourceAccessor` | Use IAP TCP tunnels (SSH/RDP) | Admins, developers |
| `roles/iap.admin` | Manage IAP settings | IAP administrators |
| `roles/iap.settingsAdmin` | Edit IAP settings | Configuration managers |

### Security Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│         IAP SECURITY BEST PRACTICES                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Always verify the IAP JWT in your backend                │
│     → Don't trust X-Goog-* headers without verification     │
│     → Use the signed JWT (X-Goog-Iap-Jwt-Assertion)        │
│                                                                │
│  2. Block direct access to backend                            │
│     → Firewall: only allow traffic from LB IP ranges        │
│     → Cloud Run: --ingress=internal-and-cloud-load-balancing │
│     → App Engine: firewall allow only LB IPs                 │
│                                                                │
│  3. Use context-aware access for sensitive apps              │
│     → Require managed devices for admin panels               │
│     → Restrict by IP for production access                   │
│                                                                │
│  4. Grant IAP roles per-resource (not project-wide)          │
│     → Different teams → different IAP-protected apps         │
│                                                                │
│  5. Use groups, not individual users                         │
│     → Easier to manage access at scale                       │
│                                                                │
│  6. Enable audit logging                                      │
│     → Track who accessed what and when                       │
│     → Required for compliance                                │
│                                                                │
│  7. Set OAuth consent screen to "Internal"                   │
│     → Only org members can sign in                           │
│     → Blocks external Google accounts                        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 13 — Monitoring & Audit Logging

### Audit Logs

```bash
# IAP generates audit logs for every access:

# Admin Activity (always on):
# • IAP configuration changes
# • IAM policy changes on IAP resources

# Data Access (must enable):
# • Every user access to IAP-protected resource
# • Includes user email, IP, resource, and result

# Log filter — IAP access attempts
gcloud logging read '
  resource.type="gce_backend_service" AND
  protoPayload.serviceName="iap.googleapis.com"
' --project=my-project --limit=20

# Failed access attempts (403s)
gcloud logging read '
  resource.type="gce_backend_service" AND
  protoPayload.serviceName="iap.googleapis.com" AND
  protoPayload.status.code=7
' --project=my-project --limit=20

# IAP TCP tunnel audit logs
gcloud logging read '
  resource.type="gce_instance" AND
  protoPayload.serviceName="iap.googleapis.com" AND
  protoPayload.methodName="AuthorizeUser"
' --project=my-project --limit=20
```

### Monitoring

```bash
# Alert on repeated IAP access failures (possible attack)
gcloud logging metrics create iap-access-denied \
    --description="IAP access denied events" \
    --log-filter='protoPayload.serviceName="iap.googleapis.com" AND protoPayload.status.code=7'
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Complete Setup

```hcl
# ─── OAuth Brand (consent screen) ───────────────────────────
resource "google_iap_brand" "default" {
  support_email     = "admin@example.com"
  application_title = "My Company Portal"
  project           = var.project_number
}

# ─── OAuth Client ────────────────────────────────────────────
resource "google_iap_client" "default" {
  display_name = "IAP Client"
  brand        = google_iap_brand.default.name
}

# ─── Enable IAP on Backend Service ───────────────────────────
resource "google_compute_backend_service" "app" {
  name        = "my-app-backend"
  project     = var.project_id
  protocol    = "HTTP"
  port_name   = "http"
  timeout_sec = 30

  backend {
    group = google_compute_instance_group_manager.app.instance_group
  }

  health_checks = [google_compute_health_check.app.id]

  iap {
    oauth2_client_id     = google_iap_client.default.client_id
    oauth2_client_secret = google_iap_client.default.secret
  }
}

# ─── IAM — Web Access ────────────────────────────────────────
resource "google_iap_web_backend_service_iam_member" "users" {
  project             = var.project_id
  web_backend_service = google_compute_backend_service.app.name
  role                = "roles/iap.httpsResourceAccessor"
  member              = "group:engineering@example.com"
}

# ─── IAM — Web Access with Context-Aware Condition ───────────
resource "google_iap_web_backend_service_iam_member" "admins" {
  project             = var.project_id
  web_backend_service = google_compute_backend_service.app.name
  role                = "roles/iap.httpsResourceAccessor"
  member              = "group:admins@example.com"

  condition {
    title       = "Managed Device Required"
    description = "Only allow access from managed devices"
    expression  = "\"accessPolicies/${var.access_policy_id}/accessLevels/managed-device\" in request.auth.access_levels"
  }
}

# ─── IAM — TCP Tunnel Access ─────────────────────────────────
resource "google_iap_tunnel_instance_iam_member" "ssh" {
  project  = var.project_id
  zone     = "us-central1-a"
  instance = google_compute_instance.bastion.name
  role     = "roles/iap.tunnelResourceAccessor"
  member   = "group:devops@example.com"
}

# ─── Firewall for IAP TCP ────────────────────────────────────
resource "google_compute_firewall" "allow_iap" {
  name    = "allow-iap-ssh"
  network = google_compute_network.vpc.name
  project = var.project_id

  allow {
    protocol = "tcp"
    ports    = ["22", "3389"]
  }

  source_ranges = ["35.235.240.0/20"]  # IAP IP range
  direction     = "INGRESS"
}

# ─── IAP for App Engine ──────────────────────────────────────
resource "google_iap_app_engine_version_iam_member" "users" {
  project    = var.project_id
  app_id     = google_app_engine_application.app.app_id
  service    = "default"
  version_id = "v1"
  role       = "roles/iap.httpsResourceAccessor"
  member     = "group:users@example.com"
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# WEB APP IAP
# ═══════════════════════════════════════════════════════════════

# Enable IAP on backend service
gcloud compute backend-services update BACKEND --iap=enabled --global

# Disable IAP
gcloud compute backend-services update BACKEND --iap=disabled --global

# Enable IAP for App Engine
gcloud iap web enable --resource-type=app-engine \
    --oauth2-client-id=ID --oauth2-client-secret=SECRET

# Grant web access
gcloud iap web add-iam-policy-binding \
    --resource-type=backend-services \
    --service=BACKEND \
    --member=MEMBER --role=roles/iap.httpsResourceAccessor

# View IAP policy
gcloud iap web get-iam-policy \
    --resource-type=backend-services \
    --service=BACKEND

# ═══════════════════════════════════════════════════════════════
# TCP TUNNELING
# ═══════════════════════════════════════════════════════════════

# SSH via IAP
gcloud compute ssh VM_NAME --zone=ZONE --tunnel-through-iap

# SCP via IAP
gcloud compute scp FILE VM:PATH --zone=ZONE --tunnel-through-iap

# Port forwarding via IAP
gcloud compute start-iap-tunnel VM_NAME PORT \
    --local-host-port=localhost:LOCAL_PORT --zone=ZONE

# Grant tunnel access
gcloud projects add-iam-policy-binding PROJECT \
    --member=MEMBER --role=roles/iap.tunnelResourceAccessor

# Grant tunnel access to specific VM
gcloud compute instances add-iam-policy-binding VM_NAME \
    --zone=ZONE --member=MEMBER \
    --role=roles/iap.tunnelResourceAccessor

# ═══════════════════════════════════════════════════════════════
# SETTINGS
# ═══════════════════════════════════════════════════════════════

# Get IAP settings
gcloud iap settings get --project=PROJECT

# Update IAP settings
gcloud iap settings set SETTINGS_FILE --project=PROJECT
```

---

## Part 15 — Real-World Patterns

### Pattern 1: VPN-Less Admin Access

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: REPLACE VPN WITH IAP FOR ADMIN ACCESS                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Before (VPN):                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Admin laptop → VPN → Corp network → Jump box → VM      │        │
│  │  Problems:                                                │        │
│  │  • VPN gives access to entire network                    │        │
│  │  • Jump box is a single point of compromise              │        │
│  │  • Hard to audit who accessed what                       │        │
│  │  • VPN client required on all devices                    │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  After (IAP):                                                         │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Admin laptop → IAP tunnel → specific VM                 │        │
│  │                                                           │        │
│  │  Setup:                                                   │        │
│  │  1. Remove public IPs from all VMs                       │        │
│  │  2. Create firewall: allow SSH only from 35.235.240.0/20│        │
│  │  3. Grant roles/iap.tunnelResourceAccessor per team:     │        │
│  │     • DevOps → all VMs                                   │        │
│  │     • Backend team → app servers only                    │        │
│  │     • DBA → database VMs only                            │        │
│  │  4. Add context-aware access level:                       │        │
│  │     • Production VMs → managed device required           │        │
│  │     • Dev VMs → any authenticated user                   │        │
│  │                                                           │        │
│  │  Benefits:                                                │        │
│  │  ✓ No VPN infrastructure to maintain                    │        │
│  │  ✓ Per-VM access control (not network-wide)             │        │
│  │  ✓ Full audit trail of every SSH session                │        │
│  │  ✓ Device trust enforcement                             │        │
│  │  ✓ Works from any network (home, cafe, mobile)          │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Internal Tools Portal

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: INTERNAL TOOLS BEHIND IAP                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Multiple internal tools protected by a single IAP setup:            │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  HTTPS Load Balancer                                      │        │
│  │  ├── /admin → Backend: admin-app (IAP: admins group)     │        │
│  │  ├── /grafana → Backend: grafana (IAP: devops group)     │        │
│  │  ├── /jenkins → Backend: jenkins (IAP: devs group)       │        │
│  │  └── /* → Backend: wiki (IAP: all-employees group)       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  IAM configuration (per backend service):                             │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  admin-app:                                               │        │
│  │    → roles/iap.httpsResourceAccessor for admins@          │        │
│  │    → access level: managed-device + corp-network          │        │
│  │                                                           │        │
│  │  grafana:                                                 │        │
│  │    → roles/iap.httpsResourceAccessor for devops@          │        │
│  │    → access level: managed-device                         │        │
│  │                                                           │        │
│  │  jenkins:                                                 │        │
│  │    → roles/iap.httpsResourceAccessor for devs@            │        │
│  │    → access level: none (identity only)                   │        │
│  │                                                           │        │
│  │  wiki:                                                    │        │
│  │    → roles/iap.httpsResourceAccessor for all-employees@  │        │
│  │    → access level: none (identity only)                   │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  All tools accessible via browser with Google SSO.                    │
│  No VPN. No passwords. Per-app access control.                        │
│  Full audit trail.                                                     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Zero-Trust Migration Path

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: GRADUAL ZERO-TRUST MIGRATION                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Phase 1: IAP for new internal tools (week 1-2)                      │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • New admin dashboards behind IAP                       │        │
│  │  • SSH via IAP for new VMs                                │        │
│  │  • Keep VPN running for existing tools                   │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Phase 2: Migrate high-value tools (month 1-2)                       │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • CI/CD (Jenkins/GitLab) behind IAP                     │        │
│  │  • Monitoring (Grafana) behind IAP                       │        │
│  │  • Add context-aware access for production               │        │
│  │  • Install Endpoint Verification on all laptops          │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Phase 3: Migrate all tools + SSH (month 3-4)                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • All internal web apps behind IAP                      │        │
│  │  • All VM SSH/RDP via IAP tunnel                         │        │
│  │  • Remove public IPs from all VMs                        │        │
│  │  • Require managed devices for production access         │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Phase 4: Decommission VPN (month 5-6)                               │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Verify all access works via IAP                       │        │
│  │  • Shut down VPN concentrators                           │        │
│  │  • Reduce network attack surface                         │        │
│  │  • Full BeyondCorp zero-trust model achieved             │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Enable IAP on backend | `gcloud compute backend-services update BE --iap=enabled --global` |
| Grant web access | `gcloud iap web add-iam-policy-binding --service=BE --member=M --role=roles/iap.httpsResourceAccessor` |
| Grant tunnel access | `--role=roles/iap.tunnelResourceAccessor` |
| SSH via IAP | `gcloud compute ssh VM --zone=Z --tunnel-through-iap` |
| SCP via IAP | `gcloud compute scp FILE VM:PATH --zone=Z --tunnel-through-iap` |
| Port forward via IAP | `gcloud compute start-iap-tunnel VM PORT --local-host-port=localhost:PORT` |
| IAP firewall rule | Allow `35.235.240.0/20` on TCP ports (22, 3389, etc.) |
| IAP JWT header | `X-Goog-Iap-Jwt-Assertion` (verify signature!) |
| User email header | `X-Goog-Authenticated-User-Email` (verify JWT first) |
| OAuth consent screen | Console → APIs & Services → OAuth consent screen |
| Pricing | Free (IAP itself) |
| BeyondCorp Enterprise | $6/user/month (advanced features) |

---

## What is Zero Trust? (Beginner Explanation)

### The Office Building Analogy

Think of **Zero Trust as a strict office building** — even if you're inside, you must prove your identity at every door. "Don't trust anyone by default — even if they're inside your network, they must prove their identity."

```
┌──────────────────────────────────────────────────────────────┐
│         ZERO TRUST = STRICT OFFICE BUILDING                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Old approach (VPN = perimeter security):                     │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  🏢 Office Building                                    │  │
│  │  • Show badge at front door ONCE                       │  │
│  │  • Once inside → access everything freely              │  │
│  │  • Lost badge? Anyone who finds it has full access     │  │
│  │  • Problem: If someone sneaks in, they own everything  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
│  Zero Trust approach (IAP):                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  🏢 Strict Office Building                             │  │
│  │  • Badge check at EVERY door, EVERY time               │  │
│  │  • Conference room? Prove who you are                  │  │
│  │  • Server room? Prove who you are + verify clearance   │  │
│  │  • Also checks: Is this YOUR badge? YOUR device?       │  │
│  │    Is it during working hours? Are you in the office?  │  │
│  │  • Even the CEO gets checked at every door             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
│  VPN vs Zero Trust:                                           │
│  ─────────────────                                            │
│  VPN:        "You're on our network? Come on in!"            │
│  Zero Trust: "Prove who you are, what device you're using,   │
│               and why you need access — EVERY time."          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Breaking It Down

| Concept | Office Analogy | What It Really Means |
|---------|---------------|---------------------|
| **VPN (old model)** | Show badge once at the front door, then roam freely | Connect to company network once, then access all internal apps |
| **Zero Trust** | Badge check at every door, every time | Verify identity and context for every request, every time |
| **IAP** | The smart badge reader at every door | Google's service that checks identity before allowing access |
| **Identity** | Your employee badge (who you are) | Your Google account or corporate identity |
| **Context** | The badge reader also checks: Is this a company-issued badge? During work hours? | IAP also checks: Managed device? Encrypted disk? Correct IP range? |
| **Access Level** | Different clearance levels for different rooms | Policies that define extra conditions (device trust, location) |
| **BeyondCorp** | Google's own "no VPN" approach for 100K+ employees | Google's framework for zero trust — IAP implements it |

### Why Does Zero Trust Matter?

1. **VPNs are a single point of failure** — If an attacker compromises the VPN, they get access to everything inside
2. **Remote work changed everything** — Employees work from home, coffee shops, airports. The "network perimeter" is meaningless
3. **IAP protects each app individually** — Even if one app is compromised, others remain protected
4. **No VPN needed** — Users access apps directly via HTTPS. Simpler, faster, more secure
5. **Audit trail** — Every access attempt is logged with who, what, when, where, and which device

> **One-liner**: Zero Trust means treating every request like it's coming from an untrusted network — because in today's world, it probably is.

---

## Console Walkthrough: Setting Up IAP

### Enabling IAP on App Engine

```
┌──────────────────────────────────────────────────────────────┐
│         ENABLE IAP ON APP ENGINE                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Configure OAuth Consent Screen                       │
│  Console → APIs & Services → OAuth consent screen             │
│  • User type: Internal (for org users only)                   │
│  • App name: "My App"                                        │
│  • Support email: your-email@company.com                      │
│  • Click SAVE                                                  │
│                                                                │
│  Step 2: Enable IAP API                                        │
│  Console → APIs & Services → Library                          │
│  Search for "Identity-Aware Proxy API" → ENABLE               │
│                                                                │
│  Step 3: Turn on IAP                                           │
│  Console → Security → Identity-Aware Proxy                    │
│  • Find your App Engine app in the list                       │
│  • Toggle the IAP switch to ON                                │
│  • Confirm in the dialog                                      │
│                                                                │
│  Step 4: Add authorized users                                  │
│  • Click the App Engine resource row                          │
│  • In the right panel, click ADD PRINCIPAL                    │
│  • Enter user email or group                                  │
│  • Role: IAP-secured Web App User                             │
│  • Click SAVE                                                  │
│                                                                │
│  ⚡ Only users with this role can access the app.              │
│    Everyone else gets a "You don't have access" page.         │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Enabling IAP on Cloud Run

```
┌──────────────────────────────────────────────────────────────┐
│         ENABLE IAP ON CLOUD RUN                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Prerequisites:                                                │
│  • Cloud Run service must be behind an HTTPS Load Balancer   │
│  • OAuth consent screen configured (see above)                │
│                                                                │
│  Step 1: Set up HTTPS Load Balancer for Cloud Run             │
│  Console → Network services → Load balancing → CREATE         │
│  • Type: Application Load Balancer (HTTPS)                    │
│  • Backend: Serverless NEG → Cloud Run service                │
│  • Frontend: HTTPS with SSL certificate                       │
│                                                                │
│  Step 2: Enable IAP on the backend service                    │
│  Console → Security → Identity-Aware Proxy                    │
│  • Find the backend service (under "HTTPS Resources")        │
│  • Toggle IAP to ON                                           │
│  • Select or create OAuth credentials                         │
│                                                                │
│  Step 3: Add authorized users                                  │
│  • Same as App Engine — add principals with                   │
│    IAP-secured Web App User role                               │
│                                                                │
│  ⚡ Note: Cloud Run's built-in URL (*.run.app) bypasses IAP. │
│    Set ingress to "Internal and Cloud Load Balancing" to     │
│    ensure ALL traffic goes through the LB (and thus IAP).    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Testing IAP Access

```
┌──────────────────────────────────────────────────────────────┐
│         TEST IAP ACCESS                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Test 1: Authorized user                                       │
│  • Open the app URL in a browser (logged into authorized     │
│    Google account)                                            │
│  • Expected: Google Sign-In → app loads normally              │
│                                                                │
│  Test 2: Unauthorized user                                     │
│  • Open the app URL in an incognito window                    │
│  • Log in with a different Google account (not authorized)    │
│  • Expected: "You don't have access" error page              │
│                                                                │
│  Test 3: No authentication                                     │
│  • Try with curl (no auth headers):                           │
│    curl https://app.example.com                                │
│  • Expected: HTML redirect to Google Sign-In page             │
│                                                                │
│  Test 4: Verify in audit logs                                  │
│  • Console → Logging → Logs Explorer                          │
│  • Filter: resource.type="gce_backend_service"               │
│  • See both allowed and denied access attempts                │
│                                                                │
│  ⚡ Tip: IAP changes can take 2-5 minutes to propagate.       │
│    If a newly added user still gets denied, wait and retry.   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 46: Pub/Sub** → `46-pubsub.md`
