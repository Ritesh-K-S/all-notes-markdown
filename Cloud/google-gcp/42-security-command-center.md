# Chapter 42 — Security Command Center

---

## Table of Contents

- [Overview](#overview)
- [Part 1: SCC Fundamentals](#part-1--scc-fundamentals)
- [Part 2: SCC Tiers — Standard vs Premium vs Enterprise](#part-2--scc-tiers--standard-vs-premium-vs-enterprise)
- [Part 3: Assets & Asset Inventory](#part-3--assets--asset-inventory)
- [Part 4: Findings](#part-4--findings)
- [Part 5: Built-in Services — Security Health Analytics](#part-5--built-in-services--security-health-analytics)
- [Part 6: Web Security Scanner](#part-6--web-security-scanner)
- [Part 7: Event Threat Detection](#part-7--event-threat-detection)
- [Part 8: Container Threat Detection](#part-8--container-threat-detection)
- [Part 9: Virtual Machine Threat Detection](#part-9--virtual-machine-threat-detection)
- [Part 10: Vulnerability Assessment & Rapid Vulnerability Detection](#part-10--vulnerability-assessment--rapid-vulnerability-detection)
- [Part 11: Security Marks & Muting](#part-11--security-marks--muting)
- [Part 12: Notifications & Exports](#part-12--notifications--exports)
- [Part 13: IAM & Organization-Level Setup](#part-13--iam--organization-level-setup)
- [Part 14: Console Walkthrough — SCC Setup & Findings](#part-14-console-walkthrough--scc-setup--findings)
- [Part 15: Terraform & gcloud CLI Reference](#part-15--terraform--gcloud-cli-reference)
- [Part 16: Real-World Patterns](#part-16--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Security Command Center (SCC) is Google Cloud's centralized security and risk management platform. It provides asset inventory, threat detection, vulnerability scanning, and compliance monitoring across your entire GCP organization. SCC aggregates findings from multiple security services into a single dashboard, enabling security teams to identify, prioritise, and remediate risks.

---

## Part 1 — SCC Fundamentals

### What Is Security Command Center?

```
┌─────────────────────────────────────────────────────────────────┐
│              SECURITY COMMAND CENTER OVERVIEW                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  SCC is the "security dashboard" for your GCP organization:     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                                                           │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐    │   │
│  │  │ Security   │  │ Event      │  │ Container      │    │   │
│  │  │ Health     │  │ Threat     │  │ Threat         │    │   │
│  │  │ Analytics  │  │ Detection  │  │ Detection      │    │   │
│  │  └──────┬─────┘  └──────┬─────┘  └──────┬─────────┘    │   │
│  │         │               │               │               │   │
│  │  ┌──────┴───────┐ ┌────┴────┐  ┌──────┴─────────┐    │   │
│  │  │ Web Security │ │ VM      │  │ Third-Party    │    │   │
│  │  │ Scanner     │ │ Threat  │  │ Integrations   │    │   │
│  │  │             │ │ Detect. │  │ (Chronicle,    │    │   │
│  │  └──────┬──────┘ └────┬────┘  │  Palo Alto...) │    │   │
│  │         │              │       └──────┬─────────┘    │   │
│  │         └──────────────┼──────────────┘               │   │
│  │                        ▼                               │   │
│  │  ┌─────────────────────────────────────────────────┐  │   │
│  │  │        SECURITY COMMAND CENTER                   │  │   │
│  │  │                                                   │  │   │
│  │  │  ┌──────────┐ ┌──────────┐ ┌───────────────┐  │  │   │
│  │  │  │ Assets   │ │ Findings │ │ Compliance    │  │  │   │
│  │  │  │ Inventory│ │ Dashboard│ │ Reports       │  │  │   │
│  │  │  └──────────┘ └──────────┘ └───────────────┘  │  │   │
│  │  │                                                   │  │   │
│  │  │  ┌──────────┐ ┌──────────┐ ┌───────────────┐  │  │   │
│  │  │  │ Attack   │ │ Security │ │ Notifications │  │  │   │
│  │  │  │ Exposure │ │ Posture  │ │ & Exports     │  │  │   │
│  │  │  └──────────┘ └──────────┘ └───────────────┘  │  │   │
│  │  └─────────────────────────────────────────────────┘  │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP SCC | AWS Security Hub | Azure Defender for Cloud |
|---------|---------|-----------------|------------------------|
| Service | Security Command Center | Security Hub | Microsoft Defender for Cloud |
| Scope | Organization-wide | Account / Organization | Subscription / Management Group |
| Asset inventory | Built-in | AWS Config | Resource Graph |
| Vulnerability scanning | SHA, Web Scanner | Inspector | Qualys / Defender |
| Threat detection | ETD, CTD, VMTD | GuardDuty | Defender plans |
| Compliance | CIS, PCI, ISO, NIST | CIS, PCI, NIST | CIS, PCI, ISO, NIST |
| SIEM integration | Chronicle, Pub/Sub | EventBridge | Sentinel |
| Attack path | Attack exposure (Premium) | Not built-in | Attack path analysis |
| Free tier | Standard tier | 30-day free trial | Basic free, plans paid |
| Premium pricing | Per-resource pricing | $0.0010/check | Per-plan pricing |

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Asset** | Any GCP resource (VM, bucket, network, SA, etc.) |
| **Finding** | A security issue detected by a source (vulnerability, misconfiguration, threat) |
| **Source** | A service that produces findings (SHA, ETD, CTD, Web Scanner, third-party) |
| **Security Mark** | Custom key-value metadata you attach to assets or findings |
| **Mute Rule** | Suppress findings matching a filter (for known acceptable risks) |
| **Notification Config** | Pub/Sub topic that receives new findings |
| **Attack Exposure** | Simulated attack path showing how an attacker could reach critical resources |

---

## Part 2 — SCC Tiers — Standard vs Premium vs Enterprise

### Tier Comparison

```
┌──────────────────────────────────────────────────────────────┐
│         SCC TIERS                                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  STANDARD (Free)                                     │    │
│  │  ─────────────────                                   │    │
│  │  ✓ Security Health Analytics (limited detectors)    │    │
│  │  ✓ Web Security Scanner (manual scans only)         │    │
│  │  ✓ Asset inventory                                   │    │
│  │  ✗ Event Threat Detection                            │    │
│  │  ✗ Container Threat Detection                        │    │
│  │  ✗ VM Threat Detection                               │    │
│  │  ✗ Compliance reports                                │    │
│  │  ✗ Attack exposure simulation                        │    │
│  │  ✗ Continuous exports                                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  PREMIUM                                              │    │
│  │  ──────────                                           │    │
│  │  ✓ Everything in Standard                            │    │
│  │  ✓ Security Health Analytics (ALL detectors, 140+)  │    │
│  │  ✓ Web Security Scanner (managed scans)             │    │
│  │  ✓ Event Threat Detection                            │    │
│  │  ✓ Container Threat Detection                        │    │
│  │  ✓ VM Threat Detection                               │    │
│  │  ✓ Rapid Vulnerability Detection                     │    │
│  │  ✓ Compliance reports (CIS, PCI, NIST, ISO)        │    │
│  │  ✓ Attack exposure simulation                        │    │
│  │  ✓ Continuous exports to BigQuery/Pub/Sub            │    │
│  │  ✓ Security posture management                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ENTERPRISE                                           │    │
│  │  ──────────────                                       │    │
│  │  ✓ Everything in Premium                             │    │
│  │  ✓ Chronicle SIEM + SOAR integration                │    │
│  │  ✓ Multi-cloud (AWS, Azure scanning)                │    │
│  │  ✓ Mandiant threat intelligence                      │    │
│  │  ✓ AI-powered investigation                          │    │
│  │  ✓ Case management & playbooks                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Activation

```
SCC is activated at the ORGANIZATION level:
• Standard: auto-enabled for all orgs
• Premium: requires activation + billing
• Enterprise: requires contract

Can also be activated at PROJECT level:
• For organizations that don't want org-wide coverage
• Limited to single project's resources
```

---

## Part 3 — Assets & Asset Inventory

### Asset Inventory

```
┌──────────────────────────────────────────────────────────────┐
│         ASSET INVENTORY                                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  SCC maintains an inventory of ALL GCP resources in your     │
│  organization:                                                │
│                                                                │
│  Asset types tracked:                                          │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Compute:     VMs, disks, instance groups            │    │
│  │  Networking:  VPCs, subnets, firewalls, IPs          │    │
│  │  Storage:     Buckets, objects                         │    │
│  │  Databases:   Cloud SQL, Spanner, Bigtable           │    │
│  │  IAM:         Service accounts, roles, policies       │    │
│  │  Kubernetes:  GKE clusters, nodes, pods              │    │
│  │  Serverless:  Cloud Functions, Cloud Run              │    │
│  │  Security:    KMS keys, secrets                       │    │
│  │  Logging:     Sinks, metrics                          │    │
│  │  And more... (most GCP resource types)               │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  For each asset:                                               │
│  • Resource name, type, project, location                    │
│  • Creation time, update time                                │
│  • Security marks (custom labels)                            │
│  • IAM policy                                                 │
│  • Associated findings                                        │
│  • Change history                                             │
│                                                                │
│  Console → Security → Security Command Center → Assets       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Querying Assets

```bash
# List assets via gcloud
gcloud scc assets list ORGANIZATION_ID \
    --filter="securityCenterProperties.resourceType=\"google.compute.Instance\""

# List assets in a project
gcloud scc assets list ORGANIZATION_ID \
    --filter="securityCenterProperties.resourceProject=\"projects/my-project\""

# List assets with specific security marks
gcloud scc assets list ORGANIZATION_ID \
    --filter="securityMarks.marks.environment=\"production\""
```

---

## Part 4 — Findings

### What Are Findings?

```
┌──────────────────────────────────────────────────────────────┐
│         FINDINGS                                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  A finding is a security issue detected by an SCC source:    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Finding:                                             │    │
│  │  ├── Category: "PUBLIC_BUCKET_ACL"                   │    │
│  │  ├── Source: Security Health Analytics                │    │
│  │  ├── Severity: HIGH                                   │    │
│  │  ├── State: ACTIVE                                    │    │
│  │  ├── Resource: gs://my-public-bucket                 │    │
│  │  ├── Project: my-project                             │    │
│  │  ├── Description: "Bucket is publicly accessible"    │    │
│  │  ├── Recommendation: "Remove public access"          │    │
│  │  ├── Compliance: CIS 5.1, PCI-DSS 7.1               │    │
│  │  └── External URI: link to documentation             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Finding types:                                                │
│  • Vulnerability: misconfiguration or software vulnerability │
│  • Threat: active attack or suspicious activity              │
│  • Observation: informational (not necessarily a risk)       │
│  • Error: SCC couldn't scan a resource                       │
│                                                                │
│  Severity levels:                                              │
│  • CRITICAL: Immediate remediation required                  │
│  • HIGH: Remediate soon                                       │
│  • MEDIUM: Remediate at normal pace                          │
│  • LOW: Best practice improvement                            │
│                                                                │
│  States:                                                       │
│  • ACTIVE: Issue exists                                       │
│  • INACTIVE: Issue resolved (auto-detected or manual)        │
│  • MUTED: Suppressed by mute rule                            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Viewing & Filtering Findings

```bash
# List all active findings
gcloud scc findings list ORGANIZATION_ID \
    --source="-" \
    --filter="state=\"ACTIVE\""

# Filter by severity
gcloud scc findings list ORGANIZATION_ID \
    --source="-" \
    --filter="state=\"ACTIVE\" AND severity=\"HIGH\""

# Filter by category
gcloud scc findings list ORGANIZATION_ID \
    --source="-" \
    --filter="state=\"ACTIVE\" AND category=\"PUBLIC_BUCKET_ACL\""

# Filter by resource type
gcloud scc findings list ORGANIZATION_ID \
    --source="-" \
    --filter="state=\"ACTIVE\" AND resourceName:\"compute.googleapis.com\""

# Update finding state (manual resolution)
gcloud scc findings update FINDING_NAME \
    --source=SOURCE_ID \
    --organization=ORGANIZATION_ID \
    --state=INACTIVE
```

---

## Part 5 — Built-in Services — Security Health Analytics

### What Is Security Health Analytics (SHA)?

```
┌──────────────────────────────────────────────────────────────┐
│         SECURITY HEALTH ANALYTICS (SHA)                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  SHA continuously scans your GCP resources for                │
│  misconfigurations and policy violations:                     │
│                                                                │
│  140+ detectors covering:                                      │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                                                      │    │
│  │  IAM & Authentication:                                │    │
│  │  • Over-privileged service accounts                  │    │
│  │  • SA key not rotated (>90 days)                     │    │
│  │  • MFA not enforced                                  │    │
│  │  • Default SA used on VM                              │    │
│  │                                                      │    │
│  │  Networking:                                          │    │
│  │  • Open firewall rules (0.0.0.0/0)                   │    │
│  │  • SSH open to internet                               │    │
│  │  • RDP open to internet                               │    │
│  │  • Default network exists                             │    │
│  │                                                      │    │
│  │  Storage & Data:                                      │    │
│  │  • Public bucket                                      │    │
│  │  • Unencrypted disk (no CMEK)                        │    │
│  │  • Cloud SQL public IP                                │    │
│  │  • Cloud SQL no SSL                                   │    │
│  │                                                      │    │
│  │  Compute:                                             │    │
│  │  • OS Login not enabled                               │    │
│  │  • Serial port enabled                                │    │
│  │  • Shielded VM not enabled                            │    │
│  │  • IP forwarding enabled                              │    │
│  │                                                      │    │
│  │  Logging & Monitoring:                                │    │
│  │  • Audit logging disabled                             │    │
│  │  • No log sinks configured                           │    │
│  │  • VPC flow logs disabled                             │    │
│  │                                                      │    │
│  │  Kubernetes:                                          │    │
│  │  • GKE dashboard enabled                              │    │
│  │  • Legacy authorization enabled                       │    │
│  │  • Node auto-upgrade disabled                        │    │
│  │  • Network policy disabled                            │    │
│  │                                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Standard tier: ~40 detectors                                 │
│  Premium tier: 140+ detectors                                 │
│                                                                │
│  Scan frequency: near real-time (within minutes of change)   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Common SHA Findings

| Finding Category | Severity | Description |
|-----------------|----------|-------------|
| `PUBLIC_BUCKET_ACL` | HIGH | Cloud Storage bucket is publicly accessible |
| `OPEN_FIREWALL` | HIGH | Firewall allows 0.0.0.0/0 ingress |
| `OPEN_SSH_PORT` | HIGH | SSH port (22) open to internet |
| `OPEN_RDP_PORT` | HIGH | RDP port (3389) open to internet |
| `SQL_PUBLIC_IP` | HIGH | Cloud SQL instance has public IP |
| `SQL_NO_ROOT_PASSWORD` | CRITICAL | Cloud SQL has no root password |
| `DEFAULT_SERVICE_ACCOUNT_USED` | MEDIUM | VM uses default compute SA |
| `OVER_PRIVILEGED_SERVICE_ACCOUNT` | MEDIUM | SA has owner/editor role |
| `SA_KEY_NOT_ROTATED` | MEDIUM | SA key > 90 days old |
| `MFA_NOT_ENFORCED` | HIGH | MFA not enforced for org members |
| `AUDIT_LOGGING_DISABLED` | MEDIUM | Data access audit logging off |
| `FLOW_LOGS_DISABLED` | LOW | VPC flow logs not enabled |
| `LEGACY_AUTHORIZATION_ENABLED` | HIGH | GKE legacy ABAC enabled |
| `MASTER_AUTHORIZED_NETWORKS_DISABLED` | HIGH | GKE API open to internet |

---

## Part 6 — Web Security Scanner

### What Is Web Security Scanner?

```
┌──────────────────────────────────────────────────────────────┐
│         WEB SECURITY SCANNER                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Scans web applications for common vulnerabilities:           │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Supported targets:                                   │    │
│  │  • App Engine (Standard & Flexible)                  │    │
│  │  • Compute Engine (public web apps)                  │    │
│  │  • GKE (Ingress-exposed services)                    │    │
│  │  • Cloud Run (public services)                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Vulnerability detection (OWASP Top 10):                      │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  • Cross-Site Scripting (XSS)                        │    │
│  │  • SQL Injection                                      │    │
│  │  • Server-Side Request Forgery (SSRF)                │    │
│  │  • Outdated/vulnerable libraries                     │    │
│  │  • Mixed content (HTTP in HTTPS page)                │    │
│  │  • Clear text password fields                        │    │
│  │  • Insecure headers (missing CSP, HSTS)             │    │
│  │  • XML External Entity (XXE)                         │    │
│  │  • Prototype pollution                                │    │
│  │  • Invalid/misconfigured SSL                         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Standard: manual scans only                                  │
│  Premium: managed (automatic) scans on schedule              │
│                                                                │
│  ⚠ Scanner sends real HTTP requests to your application      │
│  ⚠ Use only on apps you own / have permission to scan        │
│  ⚠ May create test data (forms, sign-ups)                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating a Scan

```bash
# Create a Web Security Scanner scan config
gcloud beta web-security-scanner scan-configs create \
    --display-name="My App Scan" \
    --starting-urls="https://my-app.run.app/" \
    --target-platforms=CLOUD_RUN

# Start a scan
gcloud beta web-security-scanner scan-runs start \
    --scan-config=SCAN_CONFIG_ID

# List scan results
gcloud beta web-security-scanner scan-runs list \
    --scan-config=SCAN_CONFIG_ID
```

---

## Part 7 — Event Threat Detection

### What Is Event Threat Detection (ETD)?

```
┌──────────────────────────────────────────────────────────────┐
│         EVENT THREAT DETECTION (Premium)                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ETD analyses Cloud Logging and Cloud Audit Logs in           │
│  real-time to detect active threats:                          │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Cloud Audit Logs                                     │    │
│  │  VPC Flow Logs     ─────►  Event Threat Detection    │    │
│  │  DNS Logs                  (ML + rules engine)        │    │
│  │                            │                          │    │
│  │                            ▼                          │    │
│  │                     Findings in SCC                    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Threats detected:                                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Account threats:                                     │    │
│  │  • Compromised credentials                           │    │
│  │  • Brute force SSH/RDP                                │    │
│  │  • Government-backed attack                           │    │
│  │  • Anomalous IAM grants                               │    │
│  │                                                      │    │
│  │  Crypto mining:                                       │    │
│  │  • Unauthorized cryptocurrency mining                 │    │
│  │  • Mining pool communication                          │    │
│  │                                                      │    │
│  │  Data exfiltration:                                   │    │
│  │  • BigQuery data exfiltration                        │    │
│  │  • Cloud SQL data exfiltration                       │    │
│  │  • Cloud Storage data exfiltration                   │    │
│  │                                                      │    │
│  │  Malware & C2:                                        │    │
│  │  • Malware detected (via VirusTotal integration)     │    │
│  │  • Command & control communication                   │    │
│  │  • DNS tunneling                                      │    │
│  │                                                      │    │
│  │  Evasion:                                             │    │
│  │  • VPC firewall rule modification                    │    │
│  │  • Logging disabled / modified                        │    │
│  │  • Resource access from Tor exit nodes               │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  No agent needed — analyses existing logs                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 8 — Container Threat Detection

### What Is Container Threat Detection (CTD)?

```
┌──────────────────────────────────────────────────────────────┐
│         CONTAINER THREAT DETECTION (Premium)                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  CTD monitors GKE clusters for container-specific threats:    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  GKE Cluster                                         │    │
│  │  ├── Node 1 ──► CTD DaemonSet                       │    │
│  │  ├── Node 2 ──► CTD DaemonSet                       │    │
│  │  └── Node 3 ──► CTD DaemonSet                       │    │
│  │                    │                                  │    │
│  │                    ▼                                  │    │
│  │              Container Threat Detection               │    │
│  │                    │                                  │    │
│  │                    ▼                                  │    │
│  │              Findings in SCC                          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Threats detected:                                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  • Added binary executed                              │    │
│  │    (unexpected binary dropped into container)         │    │
│  │                                                      │    │
│  │  • Added library loaded                               │    │
│  │    (unexpected shared library loaded)                 │    │
│  │                                                      │    │
│  │  • Reverse shell                                      │    │
│  │    (stdin redirected from network socket)             │    │
│  │                                                      │    │
│  │  • Malicious script executed                          │    │
│  │    (known malicious script patterns)                  │    │
│  │                                                      │    │
│  │  • Unexpected child shell                             │    │
│  │    (shell spawned by process that shouldn't)         │    │
│  │                                                      │    │
│  │  • Malicious URL observed                             │    │
│  │    (container accessed known-malicious domain)       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Requirements:                                                 │
│  • GKE Standard or Autopilot                                 │
│  • Container-Optimized OS or Ubuntu node images              │
│  • SCC Premium tier                                           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 9 — Virtual Machine Threat Detection

### What Is VM Threat Detection (VMTD)?

```
┌──────────────────────────────────────────────────────────────┐
│         VIRTUAL MACHINE THREAT DETECTION (Premium)            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  VMTD scans VM memory from OUTSIDE the VM (hypervisor-level) │
│  — no agent needed, can't be evaded by malware:              │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Compute Engine VM                                    │    │
│  │  ┌──────────────┐                                    │    │
│  │  │ Guest OS     │                                    │    │
│  │  │ (no agent)   │                                    │    │
│  │  └──────────────┘                                    │    │
│  │  ┌──────────────────────────────────────┐            │    │
│  │  │ Hypervisor (host)                     │            │    │
│  │  │ → VMTD inspects guest memory         │            │    │
│  │  │ → Detects crypto miners, rootkits    │            │    │
│  │  └──────────────────────────────────────┘            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Threats detected:                                             │
│  • Cryptocurrency mining (kernel-level detection)            │
│  • Rootkits                                                    │
│  • Process injection                                           │
│  • Known malware signatures in memory                        │
│                                                                │
│  Key advantage: AGENTLESS                                      │
│  • Malware can't detect or disable VMTD                      │
│  • Works even if guest OS is compromised                     │
│  • No performance impact from agent                          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 10 — Vulnerability Assessment & Rapid Vulnerability Detection

### Rapid Vulnerability Detection

```
┌──────────────────────────────────────────────────────────────┐
│         RAPID VULNERABILITY DETECTION (Premium)               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Network-based vulnerability scanner for external-facing     │
│  resources:                                                    │
│                                                                │
│  • Scans for known CVEs in network-accessible software       │
│  • Checks for weak credentials                               │
│  • Detects exposed admin interfaces                          │
│  • Tests for misconfigured services                          │
│                                                                │
│  Targets:                                                      │
│  • Compute Engine VMs with public IPs                        │
│  • GKE nodes with public IPs                                 │
│  • Public-facing load balancers                              │
│                                                                │
│  Scan frequency: weekly (automatic)                           │
│                                                                │
│  Common findings:                                              │
│  • Log4Shell (CVE-2021-44228)                                │
│  • Outdated Apache/Nginx versions                            │
│  • Exposed databases (MongoDB, Redis without auth)           │
│  • Default credentials                                        │
│  • Open management ports (e.g., phpMyAdmin)                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 11 — Security Marks & Muting

### Security Marks

```bash
# Add security marks to an asset
gcloud scc assets update-marks ASSET_ID \
    --organization=ORG_ID \
    --security-marks="environment=production,data_classification=pii,team=backend"

# Add security marks to a finding
gcloud scc findings update-marks FINDING_NAME \
    --source=SOURCE_ID \
    --organization=ORG_ID \
    --security-marks="false_positive=true,reviewed_by=security-team"

# Use marks to filter findings
gcloud scc findings list ORG_ID \
    --source="-" \
    --filter="securityMarks.marks.environment=\"production\" AND severity=\"HIGH\""
```

### Muting Findings

```
┌──────────────────────────────────────────────────────────────┐
│         MUTING FINDINGS                                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Mute rules suppress notifications for known acceptable      │
│  risks without removing the finding:                          │
│                                                                │
│  Use cases:                                                    │
│  • Public buckets that are intentionally public              │
│  • Dev/test resources with relaxed security                  │
│  • Known false positives                                      │
│  • Findings already tracked in external issue tracker        │
│                                                                │
│  Mute types:                                                   │
│  • Static mute: manually mute individual findings            │
│  • Dynamic mute: auto-mute findings matching a filter        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create a mute rule (dynamic — auto-mutes matching findings)
gcloud scc muteconfigs create dev-public-buckets \
    --organization=ORG_ID \
    --description="Mute public bucket findings in dev projects" \
    --filter="category=\"PUBLIC_BUCKET_ACL\" AND resource.projectDisplayName=\"dev-project\""

# Mute a specific finding (static)
gcloud scc findings set-mute FINDING_NAME \
    --source=SOURCE_ID \
    --organization=ORG_ID \
    --mute=MUTED

# Unmute
gcloud scc findings set-mute FINDING_NAME \
    --source=SOURCE_ID \
    --organization=ORG_ID \
    --mute=UNMUTED

# List mute configs
gcloud scc muteconfigs list --organization=ORG_ID
```

---

## Part 12 — Notifications & Exports

### Pub/Sub Notifications

```
┌──────────────────────────────────────────────────────────────┐
│         FINDING NOTIFICATIONS                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────┐                                    │
│  │ SCC Findings         │                                    │
│  │ (new / state change) │                                    │
│  └──────────┬───────────┘                                    │
│             │                                                │
│             ▼                                                │
│  ┌──────────────────────┐                                    │
│  │ Notification Config  │                                    │
│  │ (filter: HIGH+CRIT)  │                                    │
│  └──────────┬───────────┘                                    │
│             │                                                │
│             ▼                                                │
│  ┌──────────────────────┐                                    │
│  │ Pub/Sub Topic        │                                    │
│  │ "scc-critical-alerts"│                                    │
│  └──────────┬───────────┘                                    │
│             │                                                │
│       ┌─────┼──────┐                                        │
│       ▼     ▼      ▼                                        │
│  ┌──────┐ ┌────┐ ┌──────┐                                  │
│  │Slack │ │Jira│ │SIEM  │                                  │
│  │      │ │    │ │      │                                  │
│  └──────┘ └────┘ └──────┘                                  │
│  (via Cloud Functions)                                      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create notification config
gcloud scc notifications create critical-findings \
    --organization=ORG_ID \
    --pubsub-topic="projects/my-project/topics/scc-critical-alerts" \
    --filter="state=\"ACTIVE\" AND (severity=\"HIGH\" OR severity=\"CRITICAL\")"

# List notification configs
gcloud scc notifications list --organization=ORG_ID

# Delete notification config
gcloud scc notifications delete critical-findings --organization=ORG_ID
```

### BigQuery Export (Continuous)

```bash
# Create continuous export to BigQuery (Premium)
gcloud scc bqexports create scc-findings-export \
    --organization=ORG_ID \
    --dataset="projects/my-project/datasets/scc_findings" \
    --filter="state=\"ACTIVE\""
```

---

## Part 13 — IAM & Organization-Level Setup

### IAM Roles

| Role | Description |
|------|-------------|
| `roles/securitycenter.admin` | Full SCC management |
| `roles/securitycenter.editor` | Edit findings, marks, configs |
| `roles/securitycenter.findingsEditor` | Update finding state/marks |
| `roles/securitycenter.findingsViewer` | View findings |
| `roles/securitycenter.assetsViewer` | View assets |
| `roles/securitycenter.sourcesEditor` | Manage sources |
| `roles/securitycenter.notificationConfigEditor` | Manage notifications |
| `roles/securitycenter.muteConfigsEditor` | Manage mute rules |

### Setup

```bash
# Enable SCC API
gcloud services enable securitycenter.googleapis.com

# Grant SCC admin to security team
gcloud organizations add-iam-policy-binding ORG_ID \
    --member="group:security-team@example.com" \
    --role="roles/securitycenter.admin"

# Grant findings viewer to DevOps
gcloud organizations add-iam-policy-binding ORG_ID \
    --member="group:devops@example.com" \
    --role="roles/securitycenter.findingsViewer"

# Grant findings editor to incident responders
gcloud organizations add-iam-policy-binding ORG_ID \
    --member="group:incident-response@example.com" \
    --role="roles/securitycenter.findingsEditor"
```

---

## Part 14: Console Walkthrough — SCC Setup & Findings

### Enabling Security Command Center

```
Console → Security → Security Command Center → ACTIVATE

┌─────────────────────────────────────────────────────────────────┐
│           SECURITY COMMAND CENTER SETUP                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Select tier ──                                                │
│ ○ Standard (free)                                              │
│   ├── Security Health Analytics                               │
│   ├── Web Security Scanner (limited)                           │
│   └── Asset inventory                                          │
│                                                                   │
│ ● Premium (paid)                                               │
│   ├── All Standard features                                   │
│   ├── Event Threat Detection                                  │
│   ├── Container Threat Detection                               │
│   ├── VM Threat Detection                                      │
│   ├── Vulnerability Assessment                                │
│   └── Compliance monitoring                                    │
│                                                                   │
│ ○ Enterprise (highest tier)                                     │
│   ├── All Premium features                                    │
│   ├── Chronicle SIEM integration                               │
│   └── Mandiant threat intelligence                             │
│                                                                   │
│ ── Scope ──                                                      │
│ ● Organization-level (scan all projects) ← recommended        │
│ ○ Project-level (this project only)                            │
│                                                                   │
│                            [ACTIVATE]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Viewing & Managing Findings

```
Console → Security → Security Command Center → Findings

┌─────────────────────────────────────────────────────────────────┐
│           FINDINGS                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Filters: [Severity ▼] [Source ▼] [Category ▼] [State ▼]       │
│          [Time range ▼] [Project ▼]                             │
│                                                                   │
│ ┌──────────────────────────────────────────────────────────┐  │
│ │ Category         │ Severity │ Source    │ State  │ Count │  │
│ ├──────────────────────────────────────────────────────────┤  │
│ │ PUBLIC_BUCKET    │ HIGH     │ SHA      │ Active │  3    │  │
│ │ OPEN_FIREWALL    │ HIGH     │ SHA      │ Active │  5    │  │
│ │ MFA_NOT_ENFORCED │ MEDIUM   │ SHA      │ Active │  1    │  │
│ │ SSL_NOT_ENFORCED │ MEDIUM   │ SHA      │ Muted  │  2    │  │
│ └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│ Click a finding to see:                                         │
│ ├── Affected resource (link to resource page)                   │
│ ├── Recommendation (how to fix)                                │
│ ├── Source properties (detailed info)                          │
│ └── Actions: Mark as Muted / Set security marks               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Mute Rule

```
Console → Security → SCC → Findings → MUTE OPTIONS → Create mute rule

┌─────────────────────────────────────────────────────────────────┐
│           CREATE MUTE RULE                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Mute rule ID: [mute-dev-ssl-findings]                           │
│ Description: [Mute SSL findings in dev environment]            │
│                                                                   │
│ ── Findings query ──                                            │
│ [category = "SSL_NOT_ENFORCED" AND                             │
│  resource.project_display_name = "dev-project"]                │
│                                                                   │
│ ⚡ All future findings matching this query will be auto-muted. │
│   Existing matching findings are also muted.                   │
│                                                                   │
│ Mute type:                                                      │
│ ● Static (permanent until rule deleted)                        │
│ ○ Dynamic (auto-unmutes if finding changes)                    │
│                                                                   │
│                              [CREATE]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Notification Config

```
Console → Security → SCC → Settings → Notifications → CREATE NOTIFICATION

┌─────────────────────────────────────────────────────────────────┐
│           CREATE NOTIFICATION CONFIG                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Config ID: [critical-findings-alert]                            │
│ Description: [Alert on critical and high findings]             │
│                                                                   │
│ ── Pub/Sub topic ──                                             │
│ Topic: [projects/my-project/topics/scc-alerts ▼]               │
│ ⚡ Must create the Pub/Sub topic first.                         │
│   SCC will publish finding notifications to this topic.        │
│                                                                   │
│ ── Findings query ──                                            │
│ [severity = "CRITICAL" OR severity = "HIGH"]                   │
│ ⚡ Only findings matching this filter trigger notifications.    │
│                                                                   │
│                              [SAVE]                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 15 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Enable SCC API ──────────────────────────────────────────
resource "google_project_service" "scc" {
  project = var.project_id
  service = "securitycenter.googleapis.com"
}

# ─── Notification Config ─────────────────────────────────────
resource "google_pubsub_topic" "scc_alerts" {
  name    = "scc-critical-alerts"
  project = var.project_id
}

resource "google_scc_notification_config" "critical" {
  config_id    = "critical-findings"
  organization = var.org_id
  pubsub_topic = google_pubsub_topic.scc_alerts.id

  streaming_config {
    filter = "state = \"ACTIVE\" AND (severity = \"HIGH\" OR severity = \"CRITICAL\")"
  }
}

# ─── Mute Config ─────────────────────────────────────────────
resource "google_scc_mute_config" "dev_public_buckets" {
  mute_config_id = "dev-public-buckets"
  parent         = "organizations/${var.org_id}"
  description    = "Mute public bucket findings in dev projects"
  filter         = "category = \"PUBLIC_BUCKET_ACL\" AND resource.projectDisplayName = \"dev-project\""
}

# ─── BigQuery Export ──────────────────────────────────────────
resource "google_bigquery_dataset" "scc_findings" {
  dataset_id = "scc_findings"
  project    = var.project_id
  location   = "US"
}

resource "google_scc_project_custom_module" "example" {
  # Custom SHA module (Premium)
  display_name     = "custom-ssh-check"
  enablement_state = "ENABLED"
  parent           = "organizations/${var.org_id}"

  custom_config {
    predicate {
      expression = "resource.rotationPeriod > duration('7776000s')"
    }
    resource_selector {
      resource_types = ["cloudkms.googleapis.com/CryptoKey"]
    }
    severity    = "MEDIUM"
    description = "KMS key rotation period exceeds 90 days"
    recommendation = "Set rotation period to 90 days or less"
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# FINDINGS
# ═══════════════════════════════════════════════════════════════

# List all active findings
gcloud scc findings list ORG_ID --source="-" \
    --filter="state=\"ACTIVE\""

# List critical findings
gcloud scc findings list ORG_ID --source="-" \
    --filter="state=\"ACTIVE\" AND severity=\"CRITICAL\""

# Group findings by category
gcloud scc findings group ORG_ID --source="-" \
    --group-by="category"

# Update finding state
gcloud scc findings update FINDING_NAME \
    --source=SOURCE_ID --organization=ORG_ID \
    --state=INACTIVE

# ═══════════════════════════════════════════════════════════════
# ASSETS
# ═══════════════════════════════════════════════════════════════

# List assets
gcloud scc assets list ORG_ID

# List by type
gcloud scc assets list ORG_ID \
    --filter="securityCenterProperties.resourceType=\"google.compute.Instance\""

# Update security marks
gcloud scc assets update-marks ASSET_ID --organization=ORG_ID \
    --security-marks="env=prod"

# ═══════════════════════════════════════════════════════════════
# NOTIFICATIONS
# ═══════════════════════════════════════════════════════════════

# Create notification
gcloud scc notifications create NAME --organization=ORG_ID \
    --pubsub-topic=TOPIC --filter="FILTER"

# List notifications
gcloud scc notifications list --organization=ORG_ID

# Delete notification
gcloud scc notifications delete NAME --organization=ORG_ID

# ═══════════════════════════════════════════════════════════════
# MUTING
# ═══════════════════════════════════════════════════════════════

# Create mute rule
gcloud scc muteconfigs create NAME --organization=ORG_ID \
    --description="DESC" --filter="FILTER"

# Mute a finding
gcloud scc findings set-mute FINDING --source=SRC \
    --organization=ORG_ID --mute=MUTED

# List mute configs
gcloud scc muteconfigs list --organization=ORG_ID

# ═══════════════════════════════════════════════════════════════
# SOURCES
# ═══════════════════════════════════════════════════════════════

# List sources (built-in + third-party)
gcloud scc sources list --organization=ORG_ID
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Automated Security Finding Remediation

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: AUTO-REMEDIATION PIPELINE                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────┐  ┌──────────┐  ┌─────────────────────────┐       │
│  │ SCC Finding  │─►│ Pub/Sub  │─►│ Cloud Function          │       │
│  │ (new active) │  │          │  │ (auto-remediate)        │       │
│  └──────────────┘  └──────────┘  └────────────┬────────────┘       │
│                                                │                     │
│                                     ┌──────────┼──────────┐         │
│                                     ▼          ▼          ▼         │
│                               ┌──────────┐┌────────┐┌──────────┐   │
│                               │ Fix      ││ Fix    ││ Create   │   │
│                               │ Firewall ││ Public ││ Jira     │   │
│                               │ Rule     ││ Bucket ││ Ticket   │   │
│                               └──────────┘└────────┘└──────────┘   │
│                                                                        │
│  Auto-remediable findings:                                            │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  OPEN_FIREWALL → Delete/restrict the rule               │        │
│  │  PUBLIC_BUCKET_ACL → Remove allUsers/allAuthenticatedUsers│       │
│  │  FLOW_LOGS_DISABLED → Enable VPC flow logs              │        │
│  │  AUDIT_LOGGING_DISABLED → Enable audit logging          │        │
│  │  DEFAULT_NETWORK → Delete default VPC network           │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Non-auto-remediable (create ticket instead):                         │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  OVER_PRIVILEGED_SA → Needs human review                 │        │
│  │  SA_KEY_NOT_ROTATED → Needs app team coordination       │        │
│  │  SQL_PUBLIC_IP → Needs architecture change               │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Compliance Dashboard

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: COMPLIANCE MONITORING & REPORTING                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  SCC Premium provides compliance mappings:                            │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Supported frameworks:                                    │        │
│  │  • CIS Google Cloud Foundation Benchmark v1.3 / v2.0    │        │
│  │  • PCI-DSS v3.2.1 / v4.0                                │        │
│  │  • NIST 800-53 Rev 5                                     │        │
│  │  • ISO 27001                                              │        │
│  │  • SOC 2                                                  │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Dashboard view:                                                       │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  CIS Benchmark v2.0 Compliance                            │        │
│  │  ──────────────────────────────────                      │        │
│  │  Overall: 87% compliant (156/180 controls)               │        │
│  │                                                           │        │
│  │  Section 1 - IAM:       ████████████░░  85%             │        │
│  │  Section 2 - Logging:   ██████████████  100%            │        │
│  │  Section 3 - Networking: ████████░░░░░  65%  ⚠         │        │
│  │  Section 4 - VMs:       ████████████░░  90%             │        │
│  │  Section 5 - Storage:   ██████████████  100%            │        │
│  │  Section 6 - Databases: ████████████░░  88%             │        │
│  │  Section 7 - BigQuery:  ██████████████  95%             │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Export to BigQuery for historical tracking & board reporting.        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Multi-Project Security Monitoring

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: ORG-WIDE SECURITY OPERATIONS                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Organization                                                          │
│  ├── Folder: Production                                               │
│  │   ├── project-api-prod        ┐                                    │
│  │   ├── project-web-prod        │ HIGH severity                     │
│  │   └── project-data-prod       │ → PagerDuty (immediate)          │
│  │                                ┘                                    │
│  ├── Folder: Staging                                                  │
│  │   ├── project-api-staging     ┐                                    │
│  │   └── project-web-staging     │ HIGH severity                     │
│  │                                │ → Slack #staging-alerts          │
│  │                                ┘                                    │
│  └── Folder: Development                                              │
│      ├── project-dev-team-a      ┐                                    │
│      └── project-dev-team-b      │ CRITICAL only                     │
│                                   │ → Slack #dev-security            │
│                                   ┘                                    │
│                                                                        │
│  Notification configs:                                                 │
│  1. "prod-critical" → filter: severity=CRITICAL & folder=prod       │
│     → Pub/Sub → PagerDuty (page on-call)                             │
│                                                                        │
│  2. "prod-high" → filter: severity=HIGH & folder=prod                │
│     → Pub/Sub → Slack + Jira (P2 ticket)                             │
│                                                                        │
│  3. "staging-high" → filter: severity=HIGH & folder=staging          │
│     → Pub/Sub → Slack (informational)                                 │
│                                                                        │
│  4. "all-threats" → filter: findingClass=THREAT                      │
│     → Pub/Sub → Chronicle SIEM (for investigation)                   │
│                                                                        │
│  Mute rules:                                                           │
│  • Mute PUBLIC_BUCKET_ACL in dev projects                             │
│  • Mute FLOW_LOGS_DISABLED in dev projects                           │
│  • Never mute findings in production                                  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Enable SCC | `gcloud services enable securitycenter.googleapis.com` |
| List findings | `gcloud scc findings list ORG_ID --source="-"` |
| Filter active HIGH | `--filter="state=\"ACTIVE\" AND severity=\"HIGH\""` |
| List assets | `gcloud scc assets list ORG_ID` |
| Create notification | `gcloud scc notifications create NAME --organization=ORG_ID --pubsub-topic=TOPIC --filter=FILTER` |
| Create mute rule | `gcloud scc muteconfigs create NAME --organization=ORG_ID --filter=FILTER` |
| Mute finding | `gcloud scc findings set-mute FINDING --mute=MUTED` |
| Add security mark | `gcloud scc assets update-marks ASSET --security-marks="key=value"` |
| Group findings | `gcloud scc findings group ORG_ID --source="-" --group-by="category"` |
| Standard tier | Free (limited detectors) |
| Premium tier | All features (140+ SHA detectors, ETD, CTD, VMTD) |
| Enterprise tier | + Chronicle SIEM/SOAR, multi-cloud |

---

## What's Next?

Continue to **Chapter 43: VPC Service Controls** → `43-vpc-service-controls.md`
