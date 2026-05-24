# Chapter 8: Firewall Rules & Policies (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: VPC Firewall Rules](#part-1-vpc-firewall-rules)
- [Part 2: IAP (Identity-Aware Proxy) for SSH](#part-2-iap-identity-aware-proxy-for-ssh)
- [Part 3: Hierarchical Firewall Policies](#part-3-hierarchical-firewall-policies)
- [Part 4: Firewall Rules Logging & Insights](#part-4-firewall-rules-logging--insights)
- [Part 5: Common Firewall Patterns](#part-5-common-firewall-patterns)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)

---

## Overview

GCP uses VPC firewall rules and hierarchical firewall policies to control traffic. Unlike AWS (which has Security Groups + NACLs), GCP has a single, powerful firewall system with priorities, network tags, and service account targeting.

```
What you'll learn:
├── VPC Firewall Rules (deep dive)
│   ├── How they work
│   ├── Ingress & egress rules
│   ├── All fields explained
│   ├── Network tags vs service accounts
│   ├── Priority system
│   └── Implied & pre-populated rules
├── Hierarchical Firewall Policies
│   ├── Organization-level policies
│   ├── Folder-level policies
│   ├── Evaluation order
│   └── "goto_next" action
├── Cloud Armor (overview)
├── Cloud IDS
└── Real-world patterns
```

---

## Part 1: VPC Firewall Rules

### What and How They Work

```
┌─────────────────────────────────────────────────────────────────────┐
│                GCP VPC FIREWALL RULES                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Distributed firewall rules applied at the VM instance level   │
│       but defined at the VPC network level.                         │
│                                                                       │
│ Key characteristics:                                                 │
│ ┌─────────────────────────────────────────────────────────────┐     │
│ │ 1. STATEFUL                                                   │     │
│ │    If a connection is allowed, return traffic auto-allowed.   │     │
│ │    Same as AWS Security Groups.                               │     │
│ │                                                                │     │
│ │ 2. ALLOW and DENY rules                                       │     │
│ │    Unlike AWS SGs (allow only), GCP has BOTH!                │     │
│ │    Combines SG + NACL functionality in one system.           │     │
│ │                                                                │     │
│ │ 3. PRIORITY-BASED evaluation (0-65535)                        │     │
│ │    Lower number = higher priority.                            │     │
│ │    First matching rule wins (by priority).                    │     │
│ │    If same priority: DENY wins over ALLOW.                    │     │
│ │                                                                │     │
│ │ 4. APPLIED TO VMs via TARGETS                                 │     │
│ │    ├── All instances in the network                          │     │
│ │    ├── Instances with specific network tags                  │     │
│ │    └── Instances with specific service accounts              │     │
│ │                                                                │     │
│ │ 5. DISTRIBUTED (not a central chokepoint)                     │     │
│ │    Rules enforced on each VM's virtual NIC.                  │     │
│ │    No bottleneck, scales automatically.                      │     │
│ │                                                                │     │
│ │ 6. VPC-LEVEL resource                                         │     │
│ │    Defined per VPC network.                                  │     │
│ │    Applies globally across all regions/subnets in that VPC.  │     │
│ └─────────────────────────────────────────────────────────────┘     │
│                                                                       │
│ GCP Firewall vs AWS:                                                 │
│ ┌─────────────────────┬────────────────┬──────────────────┐         │
│ │ Feature             │ AWS            │ GCP              │         │
│ │ Stateful            │ SG: Yes        │ Yes              │         │
│ │                     │ NACL: No       │                  │         │
│ │ Deny rules          │ SG: No         │ Yes              │         │
│ │                     │ NACL: Yes      │                  │         │
│ │ Priority            │ SG: No         │ Yes (0-65535)    │         │
│ │                     │ NACL: Yes      │                  │         │
│ │ Target by tag       │ No (SG attach) │ Yes (network tag)│         │
│ │ Target by SA        │ No             │ Yes (unique!)    │         │
│ │ Level               │ SG: ENI        │ VM NIC           │         │
│ │                     │ NACL: Subnet   │                  │         │
│ │ Scope               │ Per VPC        │ Per VPC          │         │
│ │ Hierarchical        │ No             │ Yes (org/folder) │         │
│ └─────────────────────┴────────────────┴──────────────────┘         │
│                                                                       │
│ ⚡ GCP gives you ONE system that does what AWS needs TWO for        │
│    (Security Groups + NACLs combined!)                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Implied and Pre-Populated Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│          IMPLIED & PRE-POPULATED RULES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ IMPLIED RULES (cannot be deleted, hidden, lowest priority):          │
│ ┌──────┬───────────┬────────────┬───────────────────────────┐      │
│ │ Pri  │ Direction │ Action     │ Description               │      │
│ ├──────┼───────────┼────────────┼───────────────────────────┤      │
│ │ 65535│ Egress    │ ALLOW      │ Allow all egress          │      │
│ │ 65535│ Ingress   │ DENY       │ Deny all ingress          │      │
│ └──────┴───────────┴────────────┴───────────────────────────┘      │
│                                                                       │
│ → Default: All outbound allowed, all inbound denied.                │
│ → Same concept as AWS SG defaults!                                  │
│                                                                       │
│ PRE-POPULATED RULES (auto-created, CAN be deleted):                 │
│ ┌──────┬───────────┬────────────┬───────────────────────────┐      │
│ │ Pri  │ Direction │ Action     │ Description               │      │
│ ├──────┼───────────┼────────────┼───────────────────────────┤      │
│ │ 65534│ Ingress   │ ALLOW      │ default-allow-internal    │      │
│ │      │           │            │ Allow all from 10.128.0.0/9│     │
│ │      │           │            │ (all auto-mode subnets)   │      │
│ │ 65534│ Ingress   │ ALLOW      │ default-allow-ssh         │      │
│ │      │           │            │ TCP 22 from 0.0.0.0/0     │      │
│ │ 65534│ Ingress   │ ALLOW      │ default-allow-rdp         │      │
│ │      │           │            │ TCP 3389 from 0.0.0.0/0   │      │
│ │ 65534│ Ingress   │ ALLOW      │ default-allow-icmp        │      │
│ │      │           │            │ ICMP from 0.0.0.0/0       │      │
│ └──────┴───────────┴────────────┴───────────────────────────┘      │
│                                                                       │
│ ⚠️ These are only in the "default" VPC network!                     │
│    Custom VPCs only have the implied rules (deny all ingress).     │
│                                                                       │
│ ⚠️ default-allow-ssh allows SSH from ANYWHERE!                      │
│    DELETE this in production! Use IAP for SSH instead.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Firewall Rules

```
Console → VPC network → Firewall → Create firewall rule

┌─────────────────────────────────────────────────────────────────┐
│           CREATE FIREWALL RULE                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:        [allow-https-web]                                  │
│ Description: [Allow HTTPS to web-tagged instances]              │
│ Network:     [prod-vpc ▼]                                      │
│ Priority:    [1000] (lower = higher priority)                   │
│                                                                   │
│ Direction:   ● Ingress  ○ Egress                               │
│ Action:      ● Allow    ○ Deny                                 │
│                                                                   │
│ Targets (who this rule applies to):                             │
│ ○ All instances in the network                                  │
│ ● Specified target tags                                         │
│   Tags: [web-server]                                            │
│ ○ Specified service account                                     │
│   SA: [web-sa@project.iam.gserviceaccount.com]                 │
│                                                                   │
│ Source filter (for ingress):                                    │
│ ● IPv4 ranges:  [0.0.0.0/0]                                   │
│ ○ IPv4 ranges + source tags                                    │
│ ○ IPv4 ranges + source service accounts                        │
│                                                                   │
│ Source tags (ingress only):                                      │
│ [app-server]  ← traffic from VMs tagged "app-server"           │
│                                                                   │
│ Protocols and ports:                                             │
│ ○ Allow all                                                     │
│ ● Specified protocols and ports                                 │
│   ☑ TCP: [443]                                                 │
│   ☐ UDP:                                                        │
│   ☐ Other:                                                      │
│                                                                   │
│ Enforcement: ● Enabled  ○ Disabled                             │
│   └── Disable to test without blocking (logs only)             │
│                                                                   │
│ Logs: ○ Off  ● On                                              │
│   └── Firewall Rules Logging for audit                         │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### All Rule Fields Explained

```
┌─────────────────────────────────────────────────────────────────────┐
│            FIREWALL RULE FIELDS DEEP DIVE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ NAME:                                                                │
│   Must be unique in the project.                                    │
│   Convention: allow|deny-protocol-source-target                     │
│   Examples: allow-https-web, deny-ssh-all, allow-internal-app      │
│                                                                       │
│ PRIORITY (0 - 65535):                                                │
│   ├── 0 = highest priority (checked first)                         │
│   ├── 65535 = lowest (implied rules)                               │
│   ├── 65534 = pre-populated rules                                  │
│   ├── Default if not set: 1000                                     │
│   └── Recommended numbering:                                       │
│       100-999: Critical rules (deny bad IPs, org policies)        │
│       1000-1999: Standard allow rules                              │
│       2000-2999: Less common rules                                 │
│       3000+: Catch-all / defaults                                  │
│                                                                       │
│ DIRECTION:                                                           │
│   ├── Ingress: Controls incoming traffic TO target VMs             │
│   └── Egress: Controls outgoing traffic FROM target VMs            │
│                                                                       │
│ ACTION:                                                              │
│   ├── Allow: Permit matching traffic                               │
│   └── Deny: Block matching traffic                                 │
│   ⚡ If two rules match at same priority: DENY wins!               │
│                                                                       │
│ TARGET:                                                              │
│   Who this rule applies to (which VMs):                             │
│   ├── All instances in the network (applies to every VM)          │
│   ├── Specified target tags: Only VMs with these network tags      │
│   └── Specified service accounts: Only VMs running as this SA      │
│                                                                       │
│ SOURCE (for ingress rules):                                          │
│   Where traffic comes from:                                         │
│   ├── IPv4 ranges: CIDR blocks (0.0.0.0/0, 10.0.0.0/16)         │
│   ├── Source tags: VMs with these network tags                     │
│   ├── Source service accounts: VMs running as this SA              │
│   └── Combination: Ranges + tags (OR logic)                       │
│                                                                       │
│ DESTINATION (for egress rules):                                      │
│   Where traffic goes to:                                            │
│   ├── IPv4 ranges: CIDR blocks                                     │
│   └── Destination only supports IP ranges (not tags)               │
│                                                                       │
│ PROTOCOLS AND PORTS:                                                 │
│   ├── All: Any protocol, any port                                  │
│   ├── TCP: [port or range] e.g., 443, 8080-8090                  │
│   ├── UDP: [port or range] e.g., 53, 500                         │
│   ├── ICMP: No port needed                                         │
│   └── Other: ESP (50), AH (51), SCTP (132)                       │
│                                                                       │
│ ENFORCEMENT:                                                         │
│   ├── Enabled: Rule is active                                      │
│   └── Disabled: Rule exists but not enforced (test mode!)         │
│       ⚡ Great for testing new rules without risk                   │
│                                                                       │
│ LOGGING:                                                             │
│   ├── On: Log every connection that matches this rule              │
│   └── Off: No logging (default)                                    │
│   ⚠️ Logging costs money (Cloud Logging charges)                   │
│   Enable for: important rules, audit requirements, debugging      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Network Tags vs Service Accounts

```
┌─────────────────────────────────────────────────────────────────────┐
│      NETWORK TAGS vs SERVICE ACCOUNTS as targets                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ METHOD 1: Network Tags (simple, most common)                        │
│                                                                       │
│ VM has tag: "web-server"                                             │
│ Firewall rule: Target tag = "web-server", allow TCP 443             │
│                                                                       │
│ Pros:                                                                │
│ ├── Easy to understand and manage                                  │
│ ├── Can add/remove tags on running VMs                             │
│ ├── Any project member with compute.instances.setTags can change  │
│ └── Good for: simple setups, dev environments                      │
│                                                                       │
│ Cons:                                                                │
│ ├── Anyone with instance edit permission can change tags!          │
│ ├── Not tied to IAM (less secure)                                  │
│ └── Tags are mutable by VM admins                                  │
│                                                                       │
│ METHOD 2: Service Accounts (more secure)                            │
│                                                                       │
│ VM runs as: web-sa@project.iam.gserviceaccount.com                  │
│ Firewall rule: Target SA = web-sa@..., allow TCP 443                │
│                                                                       │
│ Pros:                                                                │
│ ├── Tied to IAM — only SA admins can change assignment             │
│ ├── More secure (developers can't accidentally change)             │
│ ├── Better for production, compliance environments                 │
│ └── Also controls what APIs the VM can call (dual purpose!)       │
│                                                                       │
│ Cons:                                                                │
│ ├── VM SA can't change after creation (set at create time)        │
│ ├── More complex to manage                                         │
│ └── One SA per VM (but one SA can be on many VMs)                 │
│                                                                       │
│ Recommendation:                                                      │
│ ├── Dev/Staging: Network tags (simpler)                            │
│ ├── Production: Service accounts (more secure)                     │
│ └── Mix: Use tags for broad rules, SA for security-critical ones  │
│                                                                       │
│ Example with tags:                                                   │
│ gcloud compute instances add-tags web-vm-1 \                         │
│   --tags=web-server,http-server \                                     │
│   --zone=asia-south1-a                                                │
│                                                                       │
│ Example with service accounts:                                       │
│ gcloud compute instances create web-vm-1 \                           │
│   --service-account=web-sa@project.iam.gserviceaccount.com \         │
│   --zone=asia-south1-a                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Priority & Rule Evaluation

```
┌─────────────────────────────────────────────────────────────────────┐
│         PRIORITY & RULE EVALUATION ORDER                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Rules evaluated from LOWEST priority number (highest priority)      │
│ to HIGHEST priority number (lowest priority).                       │
│                                                                       │
│ First matching rule wins — stops evaluation.                        │
│                                                                       │
│ Example:                                                             │
│ ┌──────┬──────────────────────────────────────┬────────┐           │
│ │ Pri  │ Rule                                 │ Action │           │
│ ├──────┼──────────────────────────────────────┼────────┤           │
│ │ 100  │ DENY TCP 22 from 0.0.0.0/0          │ DENY   │ ← Wins! │
│ │ 1000 │ ALLOW TCP 22 from 0.0.0.0/0         │ ALLOW  │           │
│ │ 65535│ DENY all ingress (implied)           │ DENY   │           │
│ └──────┴──────────────────────────────────────┴────────┘           │
│                                                                       │
│ SSH from anywhere → matches rule 100 first → DENIED!               │
│                                                                       │
│ Better pattern:                                                      │
│ ┌──────┬──────────────────────────────────────┬────────┐           │
│ │ Pri  │ Rule                                 │ Action │           │
│ │ 100  │ DENY TCP 22 from 0.0.0.0/0          │ DENY   │           │
│ │ 200  │ ALLOW TCP 22 from 35.235.240.0/20   │ ALLOW  │ IAP range│
│ │      │ (IAP tunnel IP range)                │        │           │
│ │ 1000 │ ALLOW TCP 443 from 0.0.0.0/0        │ ALLOW  │           │
│ │ 65535│ DENY all (implied)                   │ DENY   │           │
│ └──────┴──────────────────────────────────────┴────────┘           │
│                                                                       │
│ Wait — rule 100 denies SSH from everywhere, including IAP!         │
│ Fix: IAP rule needs LOWER priority number than the deny:           │
│                                                                       │
│ ┌──────┬──────────────────────────────────────┬────────┐           │
│ │ 50   │ ALLOW TCP 22 from 35.235.240.0/20   │ ALLOW  │ IAP first│
│ │ 100  │ DENY TCP 22 from 0.0.0.0/0          │ DENY   │ then deny│
│ │ 1000 │ ALLOW TCP 443 from 0.0.0.0/0        │ ALLOW  │           │
│ │ 65535│ DENY all (implied)                   │ DENY   │           │
│ └──────┴──────────────────────────────────────┴────────┘           │
│                                                                       │
│ Now: IAP SSH allowed (rule 50), all other SSH denied (rule 100)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### CLI Examples

```bash
# Create allow HTTPS rule
gcloud compute firewall-rules create allow-https-web \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server \
  --priority=1000 \
  --description="Allow HTTPS to web servers" \
  --enable-logging

# Create allow internal rule
gcloud compute firewall-rules create allow-internal \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:0-65535,udp:0-65535,icmp \
  --source-ranges=10.0.0.0/16 \
  --priority=1000 \
  --description="Allow all internal VPC traffic"

# Create allow SSH from IAP only
gcloud compute firewall-rules create allow-ssh-iap \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=allow-ssh \
  --priority=500 \
  --description="Allow SSH via IAP tunnel only"

# Create deny SSH from internet
gcloud compute firewall-rules create deny-ssh-internet \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=DENY \
  --rules=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --priority=900 \
  --description="Deny SSH from internet (use IAP)"

# Create rule using source tags (app → db)
gcloud compute firewall-rules create allow-db-from-app \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:5432 \
  --source-tags=app-server \
  --target-tags=db-server \
  --priority=1000 \
  --description="Allow PostgreSQL from app servers"

# Create rule using service account
gcloud compute firewall-rules create allow-https-web-sa \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-service-accounts=web-sa@tc-prod.iam.gserviceaccount.com \
  --priority=1000

# Create egress deny rule (block external for db)
gcloud compute firewall-rules create deny-internet-db \
  --network=prod-vpc \
  --direction=EGRESS \
  --action=DENY \
  --rules=all \
  --destination-ranges=0.0.0.0/0 \
  --target-tags=db-server \
  --priority=900 \
  --description="Block internet for DB servers"

# But allow Google APIs for db (higher priority)
gcloud compute firewall-rules create allow-google-apis-db \
  --network=prod-vpc \
  --direction=EGRESS \
  --action=ALLOW \
  --rules=tcp:443 \
  --destination-ranges=199.36.153.8/30 \
  --target-tags=db-server \
  --priority=800 \
  --description="Allow Google APIs via restricted VIP"

# List all rules
gcloud compute firewall-rules list \
  --filter="network=prod-vpc" \
  --sort-by=PRIORITY \
  --format="table(name,direction,priority,allowed[].map().firewall_rule().list():label=ALLOWED,denied[].map().firewall_rule().list():label=DENIED,sourceRanges.list():label=SRC_RANGES,targetTags.list():label=TARGET_TAGS)"

# Delete a rule
gcloud compute firewall-rules delete allow-ssh-internet

# Disable a rule (for testing)
gcloud compute firewall-rules update allow-https-web --disabled

# Enable it back
gcloud compute firewall-rules update allow-https-web --no-disabled
```

### Terraform Example

```hcl
# Allow HTTPS to web servers
resource "google_compute_firewall" "allow_https_web" {
  name    = "allow-https-web"
  network = google_compute_network.prod_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["443"]
  }

  direction     = "INGRESS"
  priority      = 1000
  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["web-server"]
  description   = "Allow HTTPS to web-tagged instances"

  log_config {
    metadata = "INCLUDE_ALL_METADATA"
  }
}

# Allow SSH from IAP only
resource "google_compute_firewall" "allow_ssh_iap" {
  name    = "allow-ssh-iap"
  network = google_compute_network.prod_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  direction     = "INGRESS"
  priority      = 500
  source_ranges = ["35.235.240.0/20"]  # IAP range
  target_tags   = ["allow-ssh"]
}

# Allow app → db (using tags)
resource "google_compute_firewall" "allow_db_from_app" {
  name    = "allow-db-from-app"
  network = google_compute_network.prod_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["5432"]
  }

  direction   = "INGRESS"
  priority    = 1000
  source_tags = ["app-server"]
  target_tags = ["db-server"]
}

# Deny all egress for DB (then allow specific)
resource "google_compute_firewall" "deny_internet_db" {
  name    = "deny-internet-db"
  network = google_compute_network.prod_vpc.name

  deny {
    protocol = "all"
  }

  direction          = "EGRESS"
  priority           = 900
  destination_ranges = ["0.0.0.0/0"]
  target_tags        = ["db-server"]
}
```

---

## Part 2: IAP (Identity-Aware Proxy) for SSH

```
┌─────────────────────────────────────────────────────────────────────┐
│          IAP FOR SSH/RDP (GCP's Best Practice)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: SSH into VMs through Google's IAP tunnel — NO public IP       │
│       needed, NO VPN needed, authenticated via Google identity.     │
│                                                                       │
│ Without IAP:                                                         │
│ [You] → Internet → VM Public IP:22 (SSH open to internet! ⚠️)      │
│                                                                       │
│ With IAP:                                                            │
│ [You] → Google IAP → Tunnel → VM (private IP, port 22)            │
│ ✅ VM has no public IP                                               │
│ ✅ SSH not open to internet                                         │
│ ✅ Authenticated via Google identity                                │
│ ✅ Audited in Cloud Audit Logs                                      │
│                                                                       │
│ Setup:                                                               │
│ 1. Firewall rule: Allow TCP 22 from 35.235.240.0/20 (IAP range)   │
│ 2. IAM: Grant roles/iap.tunnelResourceAccessor to users            │
│ 3. SSH: gcloud compute ssh VM_NAME --zone=ZONE --tunnel-through-iap│
│                                                                       │
│ ⚡ This is GCP's equivalent of AWS SSM Session Manager + Bastion   │
│    Combined into one elegant solution!                              │
│                                                                       │
│ Firewall rule:                                                       │
│ gcloud compute firewall-rules create allow-ssh-iap \                 │
│   --network=prod-vpc \                                                │
│   --direction=INGRESS \                                               │
│   --action=ALLOW \                                                    │
│   --rules=tcp:22 \                                                    │
│   --source-ranges=35.235.240.0/20 \                                  │
│   --priority=500                                                      │
│                                                                       │
│ Connect:                                                             │
│ gcloud compute ssh app-vm-1 --zone=asia-south1-a \                   │
│   --tunnel-through-iap                                                │
│                                                                       │
│ Or for RDP (Windows):                                                │
│ gcloud compute start-iap-tunnel win-vm-1 3389 \                      │
│   --local-host-port=localhost:3390 \                                  │
│   --zone=asia-south1-a                                                │
│ Then connect RDP to localhost:3390                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Hierarchical Firewall Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│         HIERARCHICAL FIREWALL POLICIES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Firewall policies that can be set at the Organization or      │
│       Folder level, applied BEFORE VPC-level firewall rules.        │
│                                                                       │
│ ⚡ This is a GCP-unique feature! Neither AWS nor Azure has this.    │
│                                                                       │
│ Evaluation order (top to bottom):                                    │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ 1. Organization-level firewall policy                        │    │
│ │    ↓                                                          │    │
│ │ 2. Folder-level firewall policy (if exists)                  │    │
│ │    ↓                                                          │    │
│ │ 3. VPC-level firewall rules                                  │    │
│ │    ↓                                                          │    │
│ │ 4. Implied rules (allow egress, deny ingress)                │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Three actions:                                                       │
│ ├── ALLOW: Allow and stop evaluating                               │
│ ├── DENY: Deny and stop evaluating                                 │
│ └── GOTO_NEXT: Pass to the next level (folder → VPC rules)        │
│     ⚡ This is the KEY! Org can say "I don't care, let VPC decide" │
│                                                                       │
│ Example:                                                             │
│                                                                       │
│ Organization policy:                                                 │
│ ├── DENY TCP 22 from 0.0.0.0/0, priority 100                      │
│ │   (Nobody in the org can open SSH to internet!)                  │
│ ├── ALLOW TCP 22 from 35.235.240.0/20, priority 50                │
│ │   (But IAP SSH is always allowed)                                │
│ └── GOTO_NEXT for everything else                                   │
│     (Let VPC rules handle the rest)                                 │
│                                                                       │
│ Folder: Production                                                   │
│ ├── DENY all egress to known malware IPs, priority 100             │
│ ├── ALLOW TCP 443 from health-check ranges, priority 200          │
│ └── GOTO_NEXT for everything else                                   │
│                                                                       │
│ VPC level: Normal firewall rules apply                              │
│                                                                       │
│ Result:                                                              │
│ ├── SSH from internet: DENIED at org level (no VPC can override!)  │
│ ├── SSH from IAP: ALLOWED at org level                             │
│ ├── HTTPS: Handled by VPC rules (org says GOTO_NEXT)               │
│ └── Org-wide security guaranteed, VPC admins have flexibility      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Hierarchical Firewall Policies

```
Console → VPC network → Firewall policies → Create policy

┌─────────────────────────────────────────────────────────────────┐
│         CREATE FIREWALL POLICY                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:     [org-security-policy]                                 │
│ Scope:    ○ Organization  ● Folder [Production ▼]              │
│                                                                   │
│ Rules:                                                           │
│ ┌──────┬───────┬──────────────────────┬───────────┬──────────┐ │
│ │ Pri  │Action │ Match                │ Target    │ Logging  │ │
│ ├──────┼───────┼──────────────────────┼───────────┼──────────┤ │
│ │ 50   │ALLOW  │ TCP 22 from          │ All       │ On       │ │
│ │      │       │ 35.235.240.0/20      │ instances │          │ │
│ │ 100  │DENY   │ TCP 22 from          │ All       │ On       │ │
│ │      │       │ 0.0.0.0/0            │ instances │          │ │
│ │ 200  │ALLOW  │ TCP 443 from         │ All       │ Off      │ │
│ │      │       │ 130.211.0.0/22,      │ instances │          │ │
│ │      │       │ 35.191.0.0/16        │           │          │ │
│ │      │       │ (health check IPs)   │           │          │ │
│ │ 9999 │GOTO   │ All traffic          │ All       │ Off      │ │
│ │      │NEXT   │                      │ instances │          │ │
│ └──────┴───────┴──────────────────────┴───────────┴──────────┘ │
│                                                                   │
│ Associations:                                                    │
│ Apply to: [folders/12345678] (Production folder)                │
│           [folders/87654321] (Staging folder)                   │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create org-level policy
  gcloud compute firewall-policies create \
    --organization=123456789 \
    --short-name=org-security-policy \
    --description="Organization-wide security policy"

  # Add rule: allow SSH from IAP
  gcloud compute firewall-policies rules create 50 \
    --firewall-policy=org-security-policy \
    --organization=123456789 \
    --action=allow \
    --direction=INGRESS \
    --src-ip-ranges=35.235.240.0/20 \
    --layer4-configs=tcp:22 \
    --description="Allow SSH from IAP"

  # Add rule: deny SSH from internet
  gcloud compute firewall-policies rules create 100 \
    --firewall-policy=org-security-policy \
    --organization=123456789 \
    --action=deny \
    --direction=INGRESS \
    --src-ip-ranges=0.0.0.0/0 \
    --layer4-configs=tcp:22 \
    --description="Deny SSH from internet"

  # Add rule: goto_next for everything else
  gcloud compute firewall-policies rules create 9999 \
    --firewall-policy=org-security-policy \
    --organization=123456789 \
    --action=goto_next \
    --direction=INGRESS \
    --src-ip-ranges=0.0.0.0/0 \
    --description="Pass to VPC rules"

  # Associate with org
  gcloud compute firewall-policies associations create \
    --firewall-policy=org-security-policy \
    --organization=123456789
```

---

## Part 4: Firewall Rules Logging & Insights

```
┌─────────────────────────────────────────────────────────────────────┐
│         FIREWALL RULES LOGGING & INSIGHTS                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Firewall Rules Logging:                                              │
│ ├── Logs connections that match a rule (allowed or denied)         │
│ ├── Enable per rule (not globally)                                 │
│ ├── Stored in Cloud Logging                                        │
│ ├── Fields: src_ip, dest_ip, port, protocol, rule name, action    │
│ └── Cost: Cloud Logging rates ($0.50/GB)                           │
│                                                                       │
│ Enable logging on a rule:                                            │
│ gcloud compute firewall-rules update allow-https-web \               │
│   --enable-logging \                                                  │
│   --logging-metadata=INCLUDE_ALL_METADATA                            │
│                                                                       │
│ Query in Cloud Logging:                                              │
│ resource.type="gce_subnetwork"                                       │
│ logName="projects/tc-prod/logs/                                      │
│   compute.googleapis.com%2Ffirewall"                                 │
│ jsonPayload.rule_details.action="DENY"                              │
│                                                                       │
│ Firewall Insights (part of Network Intelligence Center):            │
│ ├── Shadowed rules: Rules that NEVER match (a higher-priority     │
│ │   rule always matches first — wasted/confusing rules)            │
│ ├── Overly permissive rules: Rules allowing more than needed      │
│ ├── Hit count: How many times each rule matched                   │
│ ├── Deny count: How many connections were denied                  │
│ └── ⚡ Use to clean up unused/redundant rules!                     │
│                                                                       │
│ Console → Network Intelligence Center → Firewall Insights          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Common Firewall Patterns

### 3-Tier Web Application

```
┌─────────────────────────────────────────────────────────────────────┐
│         3-TIER APPLICATION FIREWALL PATTERN                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Rule: allow-https-lb (health checks + client traffic)               │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Direction: Ingress    Priority: 1000    Action: ALLOW        │    │
│ │ Source: 130.211.0.0/22, 35.191.0.0/16 (LB health checks)   │    │
│ │         0.0.0.0/0 (if using external LB)                     │    │
│ │ Target tags: web-server                                      │    │
│ │ Ports: TCP 443, TCP 8080                                     │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                        │                                              │
│                        ▼                                              │
│ Rule: allow-app-from-web                                             │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Direction: Ingress    Priority: 1000    Action: ALLOW        │    │
│ │ Source tags: web-server                                      │    │
│ │ Target tags: app-server                                      │    │
│ │ Ports: TCP 8080                                              │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                        │                                              │
│                        ▼                                              │
│ Rule: allow-db-from-app                                              │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Direction: Ingress    Priority: 1000    Action: ALLOW        │    │
│ │ Source tags: app-server                                      │    │
│ │ Target tags: db-server                                       │    │
│ │ Ports: TCP 5432                                              │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Rule: allow-ssh-iap                                                  │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Direction: Ingress    Priority: 500     Action: ALLOW        │    │
│ │ Source: 35.235.240.0/20 (IAP range)                          │    │
│ │ Target tags: allow-ssh                                       │    │
│ │ Ports: TCP 22                                                │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Rule: deny-ssh-internet                                              │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Direction: Ingress    Priority: 900     Action: DENY         │    │
│ │ Source: 0.0.0.0/0                                            │    │
│ │ Target: All instances                                        │    │
│ │ Ports: TCP 22                                                │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Rule: allow-internal                                                 │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Direction: Ingress    Priority: 1000    Action: ALLOW        │    │
│ │ Source: 10.0.0.0/16 (VPC CIDR)                               │    │
│ │ Target: All instances                                        │    │
│ │ Ports: TCP all, UDP all, ICMP                                │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Egress Restrictions (Hardened DB)

```
# Block all internet for DB servers
gcloud compute firewall-rules create deny-egress-db \
  --network=prod-vpc \
  --direction=EGRESS \
  --action=DENY \
  --rules=all \
  --destination-ranges=0.0.0.0/0 \
  --target-tags=db-server \
  --priority=1000

# Allow Google APIs (for backups, etc.)
gcloud compute firewall-rules create allow-google-apis-db \
  --network=prod-vpc \
  --direction=EGRESS \
  --action=ALLOW \
  --rules=tcp:443 \
  --destination-ranges=199.36.153.8/30 \
  --target-tags=db-server \
  --priority=900

# Allow internal VPC traffic
gcloud compute firewall-rules create allow-egress-internal-db \
  --network=prod-vpc \
  --direction=EGRESS \
  --action=ALLOW \
  --rules=all \
  --destination-ranges=10.0.0.0/16 \
  --target-tags=db-server \
  --priority=900
```

---

## Part 6: Real-World Patterns

### Startup (5-10 devs)

```
Rules: ~8-10 firewall rules
├── allow-https-web (TCP 443, 0.0.0.0/0, tag: web-server)
├── allow-http-web (TCP 80, 0.0.0.0/0, tag: web-server)
├── allow-ssh-iap (TCP 22, 35.235.240.0/20)
├── deny-ssh-internet (TCP 22, 0.0.0.0/0, DENY)
├── allow-internal (all, VPC CIDR, all instances)
├── allow-health-checks (TCP, health check ranges)
├── allow-db-from-app (TCP 5432, tag: app→db)
└── allow-redis-from-app (TCP 6379, tag: app→cache)

No hierarchical policies needed.
Use network tags for targeting.
Enable logging on deny rules only (save cost).
```

### Mid-Size (50-100 devs)

```
Rules: ~20-30 firewall rules per VPC
├── Per-service rules with specific tags
├── Service account-based targeting for production
├── Egress restrictions on data-tier VMs
├── Logging on critical rules (SSH, deny, DB access)
├── Firewall Insights enabled for cleanup
├── Health check rules with specific ranges
└── GKE-specific rules for node pools

Hierarchical: Folder-level policy for production
├── Deny SSH from internet (enforced everywhere)
├── Allow IAP SSH
├── Allow GCP health check ranges
└── GOTO_NEXT for application-specific rules
```

### Enterprise (500+ devs)

```
Rules: 50-100+ per VPC (managed via Terraform)
├── Organization-level firewall policy
│   ├── Deny SSH/RDP from internet (global)
│   ├── Allow IAP access
│   ├── Block known malicious IPs (threat intel)
│   ├── Allow required GCP infrastructure ranges
│   └── GOTO_NEXT for everything else
│
├── Folder-level policies per environment
│   ├── Production: Strict egress, audit logging
│   ├── Dev: More permissive, no logging
│   └── Shared Services: Allow monitoring, CI/CD
│
├── VPC-level rules per application/service
│   ├── Service account-based targeting
│   ├── Full egress restrictions on sensitive workloads
│   └── Firewall Rules Logging on all production rules

Terraform modules for standard rule sets.
Firewall Insights + Security Command Center integration.
Regular rule audit (quarterly).
```

---

## Quick Reference

| Feature | Details |
|---------|---------|
| Stateful | Yes (return traffic auto-allowed) |
| Allow + Deny | Yes (unlike AWS SGs which are allow-only) |
| Priority | 0-65535 (lower = higher priority) |
| Target by | All instances, network tags, service accounts |
| Source by | IP ranges, tags, service accounts |
| Default ingress | Deny all (implied rule 65535) |
| Default egress | Allow all (implied rule 65535) |
| Hierarchical | Yes (Org → Folder → VPC) |
| GOTO_NEXT | Delegate to lower level (unique to GCP) |
| IAP SSH range | 35.235.240.0/20 |
| Health check ranges | 130.211.0.0/22, 35.191.0.0/16 |
| Logging | Per-rule (costs Cloud Logging rates) |
| Cost | Rules are FREE |

---

## Console Walkthrough: Deleting & Managing Firewall Rules

Over time, VPC firewall rules accumulate — unused rules from old services, test rules, duplicate rules. Regular cleanup keeps your network security tight and easy to audit.

### Deleting a Firewall Rule from Console

```
Console → VPC network → Firewall → select rule → DELETE
```

**Step by step:**

1. Go to **VPC network → Firewall** in the Console
2. Find the rule you want to delete (use the search/filter bar)
3. Click the rule name to open its details
4. Click **DELETE** at the top → confirm

> **⚠️ Warning:** Deleting a firewall rule takes effect immediately. If the rule was allowing traffic (e.g., SSH access), that traffic will be blocked as soon as the rule is removed.

### Bulk Deleting Firewall Rules

You can select multiple rules at once:

1. Go to **VPC network → Firewall**
2. Check the boxes next to each rule you want to delete
3. Click **DELETE** at the top of the list
4. Confirm the bulk deletion

**Via CLI (faster for many rules):**

```bash
# Delete a single rule
gcloud compute firewall-rules delete allow-ssh-test --quiet

# Delete multiple rules at once
gcloud compute firewall-rules delete rule-1 rule-2 rule-3 --quiet

# Delete all rules matching a pattern (careful!)
gcloud compute firewall-rules list \
    --filter="name~'^temp-'" \
    --format="value(name)" | xargs -I {} gcloud compute firewall-rules delete {} --quiet
```

### Auditing Firewall Rules

Before deleting, audit which rules are actually being used:

```
Console → Network Intelligence Center → Firewall Insights
```

**Firewall Insights shows you:**
- **Shadowed rules** — rules that never match because a higher-priority rule catches all the same traffic
- **Overly permissive rules** — rules allowing traffic to ports/IPs that never receive connections
- **Unused rules** — rules with zero hits over a time period (requires Firewall Rules Logging enabled)
- **Deny rules with hits** — shows what's being blocked (useful for troubleshooting)

**Enable Firewall Rules Logging** on rules you suspect are unused:

```bash
# Enable logging on a specific rule
gcloud compute firewall-rules update allow-old-service \
    --enable-logging

# Check hit counts after a few days
gcloud compute firewall-rules describe allow-old-service \
    --format="value(logConfig)"
```

> **💡 Tip:** Enable logging on suspicious rules, wait 1-2 weeks, then check Firewall Insights. If a rule has zero hits, it's safe to delete. Always verify with the team that owns the service before removing.

### Best Practices for Rule Management

| Practice | Why |
|----------|-----|
| Name rules descriptively | `allow-web-from-lb` not `rule-47` |
| Use consistent naming conventions | `allow-<service>-<source>-<port>` |
| Tag rules with purpose | Add descriptions explaining *why* the rule exists |
| Review quarterly | Use Firewall Insights to find unused/shadowed rules |
| Delete in dev first | Test deletion in dev/staging before production |
| Document before deleting | Screenshot or export the rule config before removal |

---

## What's Next?

In the next chapter, we'll cover Cloud DNS — managed DNS zones, record types, DNSSEC, DNS policies, and private DNS zones.

→ Next: [Chapter 9: Cloud DNS](09-cloud-dns.md)

---

*Last Updated: May 2026*
