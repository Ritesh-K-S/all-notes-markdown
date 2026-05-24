# Chapter 6: VPC Networks (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: What is a GCP VPC?](#part-1-what-is-a-gcp-vpc)
- [Part 2: Auto Mode vs Custom Mode VPC](#part-2-auto-mode-vs-custom-mode-vpc)
- [Part 3: Subnets](#part-3-subnets)
- [Part 4: IP Addressing](#part-4-ip-addressing)
- [Part 5: Routes](#part-5-routes)
- [Part 6: Firewall Rules (Basics)](#part-6-firewall-rules-basics)
- [Part 7: VPC Creation — Console Walkthrough](#part-7-vpc-creation--console-walkthrough)
- [Part 8: VPC Creation — gcloud CLI](#part-8-vpc-creation--gcloud-cli)
- [Part 9: VPC Creation — Terraform](#part-9-vpc-creation--terraform)
- [Part 10: Private Google Access](#part-10-private-google-access)
- [Part 11: Multi-Tier Architecture](#part-11-multi-tier-architecture)
- [Part 12: VPC Limits and Quotas](#part-12-vpc-limits-and-quotas)
- [Part 13: Real-World Patterns](#part-13-real-world-patterns)

---

## Overview

Google Cloud VPC (Virtual Private Cloud) is a global, software-defined network that spans all GCP regions. Unlike AWS/Azure where VPCs/VNets are regional, GCP VPCs are **global resources** with subnets being regional. This is a fundamental architectural difference.

```
What you'll learn:
├── What is a VPC in GCP
├── Global VPC vs Regional Subnets (GCP unique!)
├── Auto mode vs Custom mode VPC
├── Default VPC and default network
├── Subnets and IP ranges
├── IP addressing (internal, external, alias)
├── Routes and routing
├── Firewall rules (basics — deep dive in Ch8)
├── VPC creation walkthrough (Console + gcloud + Terraform)
├── Multi-tier architecture design
├── Private Google Access
├── VPC limits and quotas
└── Real-world patterns (startup, mid-size, enterprise)
```

---

## Part 1: What is a GCP VPC?

### The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                       GOOGLE CLOUD                                    │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │            YOUR VPC (GLOBAL resource!)                         │  │
│  │                                                                 │  │
│  │  Region: us-central1              Region: asia-south1          │  │
│  │  ┌──────────────────────┐         ┌──────────────────────┐    │  │
│  │  │  Subnet: 10.0.1.0/24│         │  Subnet: 10.0.2.0/24│    │  │
│  │  │  (regional resource) │         │  (regional resource) │    │  │
│  │  │                      │         │                      │    │  │
│  │  │  ┌───────┐ ┌──────┐ │         │  ┌───────┐ ┌──────┐ │    │  │
│  │  │  │ VM-1  │ │ VM-2 │ │         │  │ VM-3  │ │ VM-4 │ │    │  │
│  │  │  └───────┘ └──────┘ │         │  └───────┘ └──────┘ │    │  │
│  │  └──────────────────────┘         └──────────────────────┘    │  │
│  │                                                                 │  │
│  │  ⚡ VMs in different regions can communicate via               │  │
│  │     internal IPs (same VPC) — NO peering needed!              │  │
│  │     This is UNIQUE to GCP!                                     │  │
│  │                                                                 │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

KEY DIFFERENCES FROM AWS:
┌──────────────────────┬────────────────────┬────────────────────────┐
│ Feature              │ AWS VPC            │ GCP VPC                │
├──────────────────────┼────────────────────┼────────────────────────┤
│ VPC scope            │ Regional           │ GLOBAL                 │
│ Subnet scope         │ AZ-level           │ Regional               │
│ Cross-region in VPC  │ Need peering       │ Built-in (same VPC!)   │
│ Internet access      │ IGW + route table  │ External IP + FW rule  │
│ Firewall             │ SG + NACL          │ VPC firewall rules     │
│ Route management     │ Route tables       │ Routes (system + custom)│
│ Default network      │ Default VPC        │ "default" network      │
│ NAT                  │ NAT Gateway (paid) │ Cloud NAT (paid)       │
└──────────────────────┴────────────────────┴────────────────────────┘
```

### Why GCP's Global VPC Matters

```
┌─────────────────────────────────────────────────────────────────────┐
│                GLOBAL VPC BENEFITS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: App in asia-south1, disaster recovery in us-central1       │
│                                                                       │
│ AWS approach:                                                        │
│ ├── VPC in ap-south-1 + VPC in us-east-1                           │
│ ├── Set up VPC Peering between them                                 │
│ ├── Manage route tables in both VPCs                                │
│ ├── Data transfer charges apply                                     │
│ └── More configuration, more complexity                             │
│                                                                       │
│ GCP approach:                                                        │
│ ├── ONE VPC with subnets in both regions                            │
│ ├── VMs communicate via internal IPs automatically                  │
│ ├── No peering needed!                                               │
│ ├── Single set of firewall rules applies everywhere                 │
│ └── Simpler architecture, fewer moving parts                        │
│                                                                       │
│ ⚠️ Cross-region traffic still has data transfer costs               │
│    ($0.01/GB between zones, varies between regions)                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Auto Mode vs Custom Mode VPC

```
┌─────────────────────────────────────────────────────────────────────┐
│              AUTO MODE vs CUSTOM MODE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ AUTO MODE VPC                                                        │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ ├── Auto-creates one subnet per region (all regions!)        │    │
│ │ ├── Each subnet gets a /20 CIDR from 10.128.0.0/9 range    │    │
│ │ │   (10.128.0.0/20, 10.132.0.0/20, etc.)                   │    │
│ │ ├── New regions auto-get new subnets                        │    │
│ │ ├── Default firewall rules: deny all ingress, allow all egress│   │
│ │ │                                                             │    │
│ │ ├── ✅ Easy to start, quick setup                            │    │
│ │ ├── ⚠️ Uses predefined CIDR ranges (can't change)           │    │
│ │ ├── ⚠️ Subnets in ALL regions (may not want this)           │    │
│ │ ├── ⚠️ CIDRs from 10.128.0.0/9 may conflict with on-prem   │    │
│ │ ├── ⚠️ Not recommended for production                       │    │
│ │ └── Can convert to Custom mode (one-way, irreversible!)     │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ CUSTOM MODE VPC ← RECOMMENDED                                       │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ ├── YOU create subnets manually in regions you need          │    │
│ │ ├── YOU choose CIDR ranges (full control)                    │    │
│ │ ├── No auto-created subnets                                  │    │
│ │ ├── No subnets in regions you don't use                     │    │
│ │ │                                                             │    │
│ │ ├── ✅ Full control over IP ranges                           │    │
│ │ ├── ✅ No CIDR conflicts with on-premises                    │    │
│ │ ├── ✅ Only subnets where you need them                      │    │
│ │ ├── ✅ Required for production workloads                     │    │
│ │ └── ✅ Can expand subnets later (increase range!)            │    │
│ │                                                                │    │
│ │ ⚡ GCP unique: You can EXPAND subnet CIDR!                    │    │
│ │    (AWS/Azure: cannot change subnet CIDR after creation)      │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ The "default" network:                                               │
│ ├── Created automatically with every new GCP project               │
│ ├── Is an auto-mode VPC                                             │
│ ├── Has default firewall rules (allows SSH, RDP, ICMP, internal)  │
│ ├── ⚠️ Delete it for production projects!                           │
│ └── Production: Create custom mode VPC instead                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Subnets

### How Subnets Work in GCP

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GCP SUBNETS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ KEY: Subnets are REGIONAL (span all zones in a region)              │
│                                                                       │
│ VPC: prod-vpc (global)                                               │
│ │                                                                    │
│ ├── Subnet: web-subnet (asia-south1, 10.0.1.0/24)                 │
│ │   ├── VM in asia-south1-a ← uses this subnet                    │
│ │   ├── VM in asia-south1-b ← uses this SAME subnet               │
│ │   └── VM in asia-south1-c ← uses this SAME subnet               │
│ │   (All zones in asia-south1 share this subnet!)                  │
│ │                                                                    │
│ ├── Subnet: app-subnet (asia-south1, 10.0.2.0/24)                 │
│ │   └── App VMs in any zone of asia-south1                         │
│ │                                                                    │
│ └── Subnet: dr-subnet (us-central1, 10.0.3.0/24)                  │
│     └── DR VMs in any zone of us-central1                          │
│                                                                       │
│ AWS vs GCP subnet comparison:                                        │
│ ┌──────────────────────┬───────────────┬───────────────────────┐   │
│ │                      │ AWS           │ GCP                    │   │
│ │ Subnet scope         │ AZ (us-east-1a)│ Region (us-central1) │   │
│ │ HA across AZs        │ Multiple subnets│ Single subnet works  │   │
│ │ Expand after creation│ NO            │ YES!                   │   │
│ │ Public/Private       │ Route table   │ External IP + FW rule  │   │
│ └──────────────────────┴───────────────┴───────────────────────┘   │
│                                                                       │
│ Reserved IPs per subnet:                                             │
│ ├── First IP: Network address (e.g., 10.0.1.0)                    │
│ ├── Second IP: Gateway (e.g., 10.0.1.1)                           │
│ ├── Second-to-last: Reserved by GCP                                │
│ └── Last: Broadcast (e.g., 10.0.1.255)                            │
│ Usable for /24: 252 IPs (vs AWS: 251)                              │
│                                                                       │
│ Subnet CIDR rules:                                                   │
│ ├── Minimum: /29 (8 IPs, 4 usable)                                │
│ ├── Maximum: /8                                                     │
│ ├── Must be RFC 1918 for primary ranges                            │
│ ├── Cannot overlap with other subnets in same VPC                  │
│ ├── ⚡ CAN be expanded! (make CIDR larger, never smaller)          │
│ └── Secondary ranges: For alias IP (Pods in GKE)                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Public vs Private Access (GCP Way)

```
┌─────────────────────────────────────────────────────────────────────┐
│            GCP: NO "PUBLIC SUBNET" CONCEPT                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ In GCP, there's no "public subnet" vs "private subnet" concept!     │
│ Instead, public/private is determined per VM:                        │
│                                                                       │
│ VM has External IP + Firewall allows traffic = "Public"             │
│ VM has NO External IP = "Private"                                    │
│                                                                       │
│ Same subnet can have both:                                           │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Subnet: web-subnet (10.0.1.0/24)                             │    │
│ │                                                                │    │
│ │ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │    │
│ │ │ VM-1         │  │ VM-2         │  │ VM-3         │        │    │
│ │ │ Internal:    │  │ Internal:    │  │ Internal:    │        │    │
│ │ │ 10.0.1.2     │  │ 10.0.1.3     │  │ 10.0.1.4     │        │    │
│ │ │ External:    │  │ External:    │  │ External:    │        │    │
│ │ │ 34.93.xx.xx  │  │ NONE         │  │ NONE         │        │    │
│ │ │ = "public"   │  │ = "private"  │  │ = "private"  │        │    │
│ │ └──────────────┘  └──────────────┘  └──────────────┘        │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ For private VMs to reach internet:                                   │
│ └── Use Cloud NAT (covered in Ch7)                                  │
│                                                                       │
│ For private VMs to reach Google APIs (GCS, BigQuery, etc.):          │
│ └── Enable Private Google Access on the subnet                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: IP Addressing

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IP ADDRESSING IN GCP                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. INTERNAL (PRIVATE) IPs                                            │
│ ├── Assigned from subnet CIDR range                                 │
│ ├── Every VM gets one (required)                                    │
│ ├── Preserved through stop/start (within same zone)                │
│ ├── DNS: vm-name.zone.c.project-id.internal                       │
│ └── Used for: All internal communication                            │
│                                                                       │
│ 2. EXTERNAL (PUBLIC) IPs                                             │
│ ├── Ephemeral (changes on stop/start — DEFAULT)                    │
│ │   └── Free while VM is running; released when stopped            │
│ ├── Static (persists — you reserve it)                              │
│ │   ├── Standard tier: Regional ($0.004/hr idle)                   │
│ │   ├── Premium tier: Global ($0.004/hr idle)                      │
│ │   └── ⚠️ Charged when NOT attached!                               │
│ ├── Optional! VMs work fine without external IP                    │
│ └── With external IP: 1:1 NAT (internal ↔ external)               │
│                                                                       │
│ 3. ALIAS IP RANGES                                                   │
│ ├── Assign additional IP ranges to a VM's NIC                      │
│ ├── Used by GKE for Pod IPs                                        │
│ ├── From primary or secondary subnet range                         │
│ └── Example: VM gets 10.0.1.5 (primary) + 10.100.0.0/24 (alias)   │
│                                                                       │
│ 4. PRIVATE SERVICE CONNECT                                           │
│ ├── Access Google services via internal IPs                        │
│ ├── Similar to AWS PrivateLink                                      │
│ └── More in Ch7                                                     │
│                                                                       │
│ Static IP commands:                                                  │
│ # Reserve a regional static IP                                      │
│ gcloud compute addresses create web-ip \                             │
│   --region=asia-south1                                               │
│                                                                       │
│ # Reserve a global static IP (for LB)                                │
│ gcloud compute addresses create lb-ip \                              │
│   --global                                                           │
│                                                                       │
│ # List addresses                                                     │
│ gcloud compute addresses list                                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Routes

```
┌─────────────────────────────────────────────────────────────────────┐
│                      GCP ROUTES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Unlike AWS (explicit route tables per subnet), GCP uses a           │
│ single route collection that applies to the entire VPC.             │
│                                                                       │
│ Route types:                                                         │
│                                                                       │
│ 1. SYSTEM-GENERATED ROUTES (auto-created)                            │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ a) Default route: 0.0.0.0/0 → default internet gateway      │    │
│ │    (Can be deleted for fully private VPC)                    │    │
│ │                                                                │    │
│ │ b) Subnet routes: One per subnet, auto-managed               │    │
│ │    10.0.1.0/24 → subnet-web (auto-created with subnet)      │    │
│ │    10.0.2.0/24 → subnet-app                                 │    │
│ │    (Cannot be deleted manually)                               │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 2. CUSTOM ROUTES (you create)                                        │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ a) Static routes: Manually defined                            │    │
│ │    Next hop: VM instance, VPN tunnel, ILB, etc.              │    │
│ │                                                                │    │
│ │ b) Dynamic routes: Learned via Cloud Router (BGP)             │    │
│ │    Used with Cloud VPN, Cloud Interconnect                   │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 3. POLICY-BASED ROUTES (advanced)                                    │
│ ├── Route based on source IP, protocol, etc.                       │
│ └── For advanced use cases (firewalling, IDS)                       │
│                                                                       │
│ Route priority:                                                      │
│ ├── Most specific destination wins (longest prefix match)           │
│ ├── If same prefix: lowest priority number wins (0-65535)          │
│ ├── Default priority: 1000                                          │
│ └── Subnet routes always win over other routes for same prefix     │
│                                                                       │
│ Example routes in a production VPC:                                  │
│ ┌──────────────────┬──────────────────┬──────────┬────────────┐    │
│ │ Destination      │ Next Hop          │ Priority │ Type       │    │
│ ├──────────────────┼──────────────────┼──────────┼────────────┤    │
│ │ 10.0.1.0/24      │ subnet-web        │ 0        │ Subnet     │    │
│ │ 10.0.2.0/24      │ subnet-app        │ 0        │ Subnet     │    │
│ │ 0.0.0.0/0        │ default-internet-gw│ 1000    │ System     │    │
│ │ 192.168.0.0/16   │ vpn-tunnel-1      │ 1000     │ Dynamic    │    │
│ │ 10.1.0.0/16      │ peering-dev-vpc   │ 0        │ Peering    │    │
│ └──────────────────┴──────────────────┴──────────┴────────────┘    │
│                                                                       │
│ Network tags → Routes:                                               │
│ ├── Routes can be applied to VMs with specific network tags         │
│ ├── Example: Route 0.0.0.0/0 → Cloud NAT only for tag "private"   │
│ └── This enables per-VM routing (like having separate route tables) │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Firewall Rules (Basics)

```
┌─────────────────────────────────────────────────────────────────────┐
│                  FIREWALL RULES OVERVIEW                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GCP firewall rules are applied at the VPC level (not subnet level)  │
│ and target VMs using network tags or service accounts.              │
│                                                                       │
│ Default behavior:                                                    │
│ ├── Implied deny all INGRESS (default, lowest priority 65535)      │
│ ├── Implied allow all EGRESS (default, lowest priority 65535)      │
│ └── These cannot be deleted but can be overridden                   │
│                                                                       │
│ Firewall rule structure:                                             │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Direction:    Ingress (inbound) or Egress (outbound)         │    │
│ │ Priority:     0-65535 (lower number = higher priority)       │    │
│ │ Action:       Allow or Deny                                  │    │
│ │ Target:       All instances / Network tag / Service account  │    │
│ │ Source (ingr):IP ranges / Network tags / Service accounts    │    │
│ │ Dest (egress):IP ranges                                      │    │
│ │ Protocol:     TCP, UDP, ICMP, all / specific ports           │    │
│ │ Enforcement:  Enabled / Disabled                              │    │
│ │ Logging:      On / Off                                        │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Key differences from AWS Security Groups:                            │
│ ┌────────────────────┬─────────────────┬──────────────────────┐    │
│ │ Feature            │ AWS SG          │ GCP Firewall         │    │
│ ├────────────────────┼─────────────────┼──────────────────────┤    │
│ │ Level              │ Instance (ENI)  │ VPC-wide             │    │
│ │ Stateful?          │ Yes             │ Yes                  │    │
│ │ Deny rules         │ No (allow only) │ Yes (allow + deny!)  │    │
│ │ Priority           │ No              │ Yes (0-65535)        │    │
│ │ Targeting          │ SG attachment   │ Network tags / SA    │    │
│ │ Default ingress    │ Deny all        │ Deny all             │    │
│ │ Default egress     │ Allow all       │ Allow all            │    │
│ │ NACL equivalent    │ Separate (NACL) │ Same system (deny!)  │    │
│ └────────────────────┴─────────────────┴──────────────────────┘    │
│                                                                       │
│ ⚡ GCP advantage: No NACLs needed! Firewall rules do everything.    │
│    You can ALLOW and DENY, with priority, in one system.            │
│                                                                       │
│ (Deep dive in Chapter 8: Firewall Rules & Policies)                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Common Firewall Rules

```
# Allow SSH from IAP (recommended over 0.0.0.0/0)
gcloud compute firewall-rules create allow-ssh-iap \
  --network=prod-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=allow-ssh \
  --priority=1000 \
  --description="Allow SSH via IAP only"

# Allow HTTP/HTTPS from internet
gcloud compute firewall-rules create allow-http-https \
  --network=prod-vpc \
  --allow=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server \
  --priority=1000

# Allow internal communication (all VMs in VPC)
gcloud compute firewall-rules create allow-internal \
  --network=prod-vpc \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8 \
  --priority=1000

# Allow health checks from Google Cloud Load Balancer
gcloud compute firewall-rules create allow-health-check \
  --network=prod-vpc \
  --allow=tcp \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=web-server \
  --priority=1000

# Deny all ingress (explicit — already implied, but good to have)
gcloud compute firewall-rules create deny-all-ingress \
  --network=prod-vpc \
  --action=DENY \
  --rules=all \
  --source-ranges=0.0.0.0/0 \
  --priority=65534
```

---

## Part 7: VPC Creation — Console Walkthrough

```
Console → VPC network → VPC networks → Create VPC Network

┌─────────────────────────────────────────────────────────────────┐
│                CREATE VPC NETWORK                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:         [prod-vpc]                                        │
│ Description:  [Production VPC for TechCorp]                     │
│                                                                   │
│ Subnet creation mode:                                            │
│ ○ Automatic (one subnet per region, predefined CIDRs)          │
│ ● Custom (you define subnets) ← RECOMMENDED                    │
│                                                                   │
│ ═══════════════════════════════════════════════════════════════  │
│                                                                   │
│ Subnets:                                                         │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Subnet 1:                                                  │   │
│ │ Name:                  [web-subnet]                        │   │
│ │ Region:                [asia-south1 ▼]                    │   │
│ │ IP stack type:         ○ IPv4 ● IPv4 (single-stack)       │   │
│ │ IPv4 range:            [10.0.1.0/24]                      │   │
│ │                                                            │   │
│ │ Private Google Access: [On ▼] ← IMPORTANT! Enable!       │   │
│ │   └── Allows VMs without external IP to reach Google APIs │   │
│ │                                                            │   │
│ │ Flow Logs:             [Off ▼] (On for production/debug)  │   │
│ │   └── Aggregation: 5 sec, Sample rate: 50%                │   │
│ │                                                            │   │
│ │ [+ ADD SECONDARY IP RANGE]                                │   │
│ │   └── For GKE Pod IPs, alias IP ranges                    │   │
│ │   Name: [pods]  Range: [10.100.0.0/16]                   │   │
│ │   Name: [services]  Range: [10.200.0.0/20]               │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [+ ADD SUBNET]                                                   │
│                                                                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Subnet 2:                                                  │   │
│ │ Name:                  [app-subnet]                        │   │
│ │ Region:                [asia-south1 ▼]                    │   │
│ │ IPv4 range:            [10.0.2.0/24]                      │   │
│ │ Private Google Access: [On ▼]                              │   │
│ │ Flow Logs:             [Off ▼]                             │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Subnet 3:                                                  │   │
│ │ Name:                  [db-subnet]                         │   │
│ │ Region:                [asia-south1 ▼]                    │   │
│ │ IPv4 range:            [10.0.3.0/24]                      │   │
│ │ Private Google Access: [On ▼]                              │   │
│ │ Flow Logs:             [Off ▼]                             │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ ═══════════════════════════════════════════════════════════════  │
│                                                                   │
│ Firewall rules:                                                  │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Default firewall rules for the "default" network:          │   │
│ │ (These are only for auto-mode "default" network)           │   │
│ │                                                            │   │
│ │ For custom VPC: No pre-created rules!                     │   │
│ │ You MUST create firewall rules, or all ingress is denied! │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Dynamic routing mode:                                            │
│ ○ Regional (routes apply to subnet's region only)              │
│ ● Global (routes apply to all subnets) ← RECOMMENDED          │
│   └── Needed for Cloud VPN/Interconnect across regions          │
│                                                                   │
│ Maximum transmission unit (MTU):                                 │
│ ○ 1460 (default, works everywhere)                              │
│ ○ 1500 (standard)                                               │
│ ○ 8896 (jumbo frames — same zone only)                          │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

What you need to create after VPC:
├── Firewall rules (SSH, HTTP/HTTPS, internal, health checks)
├── Cloud NAT (if VMs need internet without external IP)
├── Cloud Router (for NAT and VPN)
└── Additional subnets in other regions (if needed)
```

---

## Part 8: VPC Creation — gcloud CLI

```bash
# 1. Create custom mode VPC
gcloud compute networks create prod-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=global \
  --description="Production VPC for TechCorp"

# 2. Create subnets
gcloud compute networks subnets create web-subnet \
  --network=prod-vpc \
  --region=asia-south1 \
  --range=10.0.1.0/24 \
  --enable-private-ip-google-access \
  --description="Web tier subnet"

gcloud compute networks subnets create app-subnet \
  --network=prod-vpc \
  --region=asia-south1 \
  --range=10.0.2.0/24 \
  --enable-private-ip-google-access \
  --description="Application tier subnet"

gcloud compute networks subnets create db-subnet \
  --network=prod-vpc \
  --region=asia-south1 \
  --range=10.0.3.0/24 \
  --enable-private-ip-google-access \
  --description="Database tier subnet"

# With secondary ranges for GKE
gcloud compute networks subnets create gke-subnet \
  --network=prod-vpc \
  --region=asia-south1 \
  --range=10.0.10.0/24 \
  --secondary-range=pods=10.100.0.0/16,services=10.200.0.0/20 \
  --enable-private-ip-google-access

# 3. Create firewall rules
gcloud compute firewall-rules create prod-allow-internal \
  --network=prod-vpc \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8

gcloud compute firewall-rules create prod-allow-ssh-iap \
  --network=prod-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=allow-ssh

gcloud compute firewall-rules create prod-allow-http \
  --network=prod-vpc \
  --allow=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server

gcloud compute firewall-rules create prod-allow-health-check \
  --network=prod-vpc \
  --allow=tcp \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=web-server

# 4. Delete default network (for security)
# First delete all firewall rules in default network
gcloud compute firewall-rules list --filter="network=default" \
  --format="value(name)" | while read rule; do
  gcloud compute firewall-rules delete "$rule" --quiet
done
gcloud compute networks delete default --quiet

# 5. Expand a subnet (GCP unique feature!)
gcloud compute networks subnets expand-ip-range web-subnet \
  --region=asia-south1 \
  --prefix-length=20
# Changed from /24 (256 IPs) to /20 (4096 IPs)!
```

---

## Part 9: VPC Creation — Terraform

```hcl
# terraform/vpc.tf

# ──────────────────────────────────────
# VPC Network
# ──────────────────────────────────────
resource "google_compute_network" "prod" {
  name                    = "prod-vpc"
  auto_create_subnetworks = false  # Custom mode
  routing_mode            = "GLOBAL"
  description             = "Production VPC for TechCorp"
}

# ──────────────────────────────────────
# Subnets
# ──────────────────────────────────────
resource "google_compute_subnetwork" "web" {
  name                     = "web-subnet"
  network                  = google_compute_network.prod.id
  region                   = "asia-south1"
  ip_cidr_range            = "10.0.1.0/24"
  private_ip_google_access = true

  log_config {
    aggregation_interval = "INTERVAL_5_SEC"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

resource "google_compute_subnetwork" "app" {
  name                     = "app-subnet"
  network                  = google_compute_network.prod.id
  region                   = "asia-south1"
  ip_cidr_range            = "10.0.2.0/24"
  private_ip_google_access = true
}

resource "google_compute_subnetwork" "db" {
  name                     = "db-subnet"
  network                  = google_compute_network.prod.id
  region                   = "asia-south1"
  ip_cidr_range            = "10.0.3.0/24"
  private_ip_google_access = true
}

# Subnet with secondary ranges (for GKE)
resource "google_compute_subnetwork" "gke" {
  name                     = "gke-subnet"
  network                  = google_compute_network.prod.id
  region                   = "asia-south1"
  ip_cidr_range            = "10.0.10.0/24"
  private_ip_google_access = true

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.100.0.0/16"
  }

  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.200.0.0/20"
  }
}

# ──────────────────────────────────────
# Firewall Rules
# ──────────────────────────────────────
resource "google_compute_firewall" "allow_internal" {
  name    = "prod-allow-internal"
  network = google_compute_network.prod.name

  allow {
    protocol = "tcp"
  }
  allow {
    protocol = "udp"
  }
  allow {
    protocol = "icmp"
  }

  source_ranges = ["10.0.0.0/8"]
  priority      = 1000
}

resource "google_compute_firewall" "allow_ssh_iap" {
  name    = "prod-allow-ssh-iap"
  network = google_compute_network.prod.name

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["35.235.240.0/20"]  # IAP range
  target_tags   = ["allow-ssh"]
  priority      = 1000
}

resource "google_compute_firewall" "allow_http" {
  name    = "prod-allow-http"
  network = google_compute_network.prod.name

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["web-server"]
  priority      = 1000
}

resource "google_compute_firewall" "allow_health_check" {
  name    = "prod-allow-health-check"
  network = google_compute_network.prod.name

  allow {
    protocol = "tcp"
  }

  source_ranges = ["130.211.0.0/22", "35.191.0.0/16"]
  target_tags   = ["web-server"]
  priority      = 1000
}

# ──────────────────────────────────────
# Cloud NAT (for private VMs internet)
# ──────────────────────────────────────
resource "google_compute_router" "prod" {
  name    = "prod-router"
  network = google_compute_network.prod.id
  region  = "asia-south1"
}

resource "google_compute_router_nat" "prod" {
  name                               = "prod-nat"
  router                             = google_compute_router.prod.name
  region                             = "asia-south1"
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}
```

---

## Part 10: Private Google Access

```
┌─────────────────────────────────────────────────────────────────────┐
│                  PRIVATE GOOGLE ACCESS                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Allows VMs WITHOUT external IPs to reach Google APIs           │
│       (Cloud Storage, BigQuery, Artifact Registry, etc.)             │
│                                                                       │
│ Without Private Google Access:                                       │
│ ├── VM without external IP → CANNOT reach Google APIs               │
│ ├── Must go through Cloud NAT → expensive!                          │
│ └── Or must have an external IP → security risk                     │
│                                                                       │
│ With Private Google Access:                                          │
│ ├── VM without external IP → CAN reach Google APIs                  │
│ ├── Traffic stays within Google's network                            │
│ ├── No Cloud NAT needed for Google API calls                        │
│ ├── FREE!                                                            │
│ └── Enabled per subnet                                               │
│                                                                       │
│ Enable:                                                              │
│ Console → VPC → Subnets → subnet → Edit → Private Google Access: On│
│                                                                       │
│ CLI:                                                                 │
│ gcloud compute networks subnets update web-subnet \                  │
│   --region=asia-south1 \                                              │
│   --enable-private-ip-google-access                                   │
│                                                                       │
│ ⚡ ALWAYS enable Private Google Access on subnets with               │
│    VMs that don't have external IPs!                                 │
│                                                                       │
│ Traffic flow:                                                        │
│ VM (10.0.2.5, no external IP) → storage.googleapis.com              │
│ → Routed to Google's internal network (not internet)                │
│ → Cloud Storage responds via Google internal network                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Multi-Tier Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│            STANDARD 3-TIER GCP ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                         INTERNET                                     │
│                            │                                          │
│                    ┌───────┴────────┐                                │
│                    │ Cloud Load     │                                 │
│                    │ Balancer       │  (Global, not in VPC)           │
│                    └───────┬────────┘                                │
│                            │                                          │
│  ┌─────────────────────────┼─────────────────────────────────────┐  │
│  │         VPC: prod-vpc (global)                                 │  │
│  │                         │                                       │  │
│  │  Region: asia-south1   │                                       │  │
│  │  ┌─────────────────────┼──────────────────────────────────┐  │  │
│  │  │    web-subnet: 10.0.1.0/24                              │  │  │
│  │  │                      │                                   │  │  │
│  │  │    zone-a            │       zone-b                     │  │  │
│  │  │    ┌──────────┐     │       ┌──────────┐              │  │  │
│  │  │    │ Web VM-1  │     │       │ Web VM-2  │              │  │  │
│  │  │    │ (no ext IP)│     │       │ (no ext IP)│             │  │  │
│  │  │    │ tag: web   │     │       │ tag: web   │             │  │  │
│  │  │    └──────────┘     │       └──────────┘              │  │  │
│  │  └─────────────────────┼──────────────────────────────────┘  │  │
│  │                         │                                       │  │
│  │  ┌─────────────────────┼──────────────────────────────────┐  │  │
│  │  │    app-subnet: 10.0.2.0/24                              │  │  │
│  │  │                      │                                   │  │  │
│  │  │    ┌──────────┐     │       ┌──────────┐              │  │  │
│  │  │    │ App VM-1  │     │       │ App VM-2  │              │  │  │
│  │  │    │ (no ext IP)│     │       │ (no ext IP)│             │  │  │
│  │  │    │ tag: app   │     │       │ tag: app   │             │  │  │
│  │  │    └──────────┘     │       └──────────┘              │  │  │
│  │  └─────────────────────┼──────────────────────────────────┘  │  │
│  │                         │                                       │  │
│  │  ┌─────────────────────┼──────────────────────────────────┐  │  │
│  │  │    db-subnet: 10.0.3.0/24                               │  │  │
│  │  │                      │                                   │  │  │
│  │  │    ┌──────────┐     │       ┌──────────┐              │  │  │
│  │  │    │ Cloud SQL │     │       │ Cloud SQL │              │  │  │
│  │  │    │ Primary   │     │       │ Replica   │              │  │  │
│  │  │    └──────────┘     │       └──────────┘              │  │  │
│  │  └─────────────────────┼──────────────────────────────────┘  │  │
│  │                         │                                       │  │
│  │                  ┌──────┴───────┐                               │  │
│  │                  │  Cloud NAT   │ (for outbound internet)      │  │
│  │                  │  + Router    │                               │  │
│  │                  └──────────────┘                               │  │
│  │                                                                 │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Key differences from AWS:                                            │
│ ├── LB is outside the VPC (global resource, no subnet needed)       │
│ ├── No IGW needed — LB handles external traffic routing             │
│ ├── VMs have NO external IPs (all private)                          │
│ ├── Cloud NAT for outbound internet (package updates, APIs)         │
│ ├── Private Google Access for Google Cloud API calls                │
│ ├── Subnet spans ALL zones in region (no per-zone subnets)          │
│ └── Firewall rules use network tags (web, app, db)                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: VPC Limits and Quotas

```
┌────────────────────────────────┬────────────────┬──────────────────┐
│ Resource                       │ Default Limit  │ Adjustable?      │
├────────────────────────────────┼────────────────┼──────────────────┤
│ Networks (VPCs) per project    │ 15             │ Yes              │
│ Subnets per network            │ 300            │ Yes              │
│ Secondary ranges per subnet    │ 30             │ Yes              │
│ Firewall rules per project     │ 500            │ Yes              │
│ Routes per network             │ 250            │ Yes              │
│ Static external IPs per region │ 8              │ Yes              │
│ Internal IP per VM (NICs)      │ 8 NICs per VM  │ No               │
│ Instances per zone             │ Varies by quota │ Yes             │
│ VPC Peering per network        │ 25             │ Yes              │
│ Cloud NAT gateways per router  │ 1              │ No               │
│ IP aliases per VM              │ Varies by type │ -                │
│ Network tags per VM            │ 64             │ No               │
└────────────────────────────────┴────────────────┴──────────────────┘

Request increase: Console → IAM & Admin → Quotas → Filter → Edit
```

---

## Part 13: Real-World Patterns

### Startup (5-10 developers)

```
Architecture:
├── 1 custom VPC (global — covers all needed regions!)
├── 3 subnets in asia-south1: web, app, db
├── Cloud NAT for outbound internet
├── Private Google Access on all subnets
├── No external IPs on VMs (use IAP for SSH)
├── Cloud Load Balancer for frontend
├── Delete "default" network!
└── Total VPC cost: ~$32/month (Cloud NAT)

Firewall rules:
├── allow-internal (10.0.0.0/8, all traffic)
├── allow-ssh-iap (35.235.240.0/20, tcp:22)
├── allow-http-lb (health check ranges + 0.0.0.0/0:80,443)
└── (implied deny-all-ingress handles the rest)

Do:
├── Use Custom mode VPC
├── Use IAP for SSH (not public IPs!)
├── Enable Private Google Access
├── Delete default network
└── Tag VMs properly for firewall targeting

Don't:
├── Use Auto mode VPC for production
├── Assign external IPs to VMs (use LB + Cloud NAT)
├── Open SSH to 0.0.0.0/0
└── Keep the default network
```

### Mid-Size (50-100 developers)

```
Architecture:
├── Separate VPCs or Shared VPC:
│   Option A: Shared VPC (host project + service projects)
│   Option B: Separate VPCs with VPC Peering
│   → Shared VPC is preferred for centralized networking
├── Custom subnets per tier per region
├── Cloud NAT per region
├── VPC Flow Logs on production subnets
├── GKE with separate subnets + secondary ranges
├── Cloud DNS private zones
├── Multiple regions for DR
└── Total: ~$100-300/month networking

Shared VPC approach:
├── Host project: owns the VPC and subnets
├── Service project (prod): uses shared subnets
├── Service project (dev): uses different shared subnets
├── Centralized firewall management
└── Network admin ≠ project admin (separation of duties)

Do:
├── Use Shared VPC (centralized network management)
├── Use hierarchical firewall policies (org-level)
├── Enable VPC Flow Logs (production)
├── Use service account-based firewall rules (more secure than tags)
├── Plan CIDR to avoid future conflicts
└── Use Cloud Armor with HTTP(S) LB
```

### Enterprise (500+ developers)

```
Architecture:
├── Shared VPC with host project
├── Multiple service projects per team/environment
├── Hierarchical firewall policies (org → folder → VPC)
├── Cloud Interconnect or HA VPN to on-premises
├── Multiple regions with global load balancing
├── Private Service Connect for Google APIs
├── DNS peering for hybrid name resolution
├── Network Intelligence Center for monitoring
├── Packet Mirroring for IDS/IPS
└── Total: $500-3,000+/month networking

Advanced patterns:
├── Hub-and-spoke with VPC Peering (or NCC)
├── Network Connectivity Center (NCC) as central hub
├── Cloud IDS for intrusion detection
├── Dedicated Interconnect (10 Gbps+ links)
├── BYOIP (Bring Your Own IP) for migration
├── Private pools for Cloud Build
├── Multi-region Cloud NAT with static IPs
└── Policy-based routes for traffic engineering
```

---

## Quick Reference

| Component | Purpose | Cost |
|-----------|---------|------|
| VPC Network | Global virtual network | FREE |
| Subnet | Regional IP range | FREE |
| Firewall rules | Traffic control | FREE |
| Routes | Traffic routing | FREE |
| External IP (ephemeral) | Public IP (changes) | FREE (while VM runs) |
| External IP (static) | Fixed public IP | ~$3/mo (idle) |
| Cloud NAT | Private VM → internet | ~$32/mo + $0.045/GB |
| Cloud Router | Dynamic routing (BGP) | FREE |
| VPC Peering | Connect two VPCs | FREE (data transfer) |
| Shared VPC | Share network across projects | FREE |
| VPC Flow Logs | Network traffic logging | $0.50/GB |
| Private Google Access | Private VM → Google APIs | FREE |

---

## What's Next?

In the next chapter, we'll cover VPC Advanced Networking — VPC Peering, Shared VPC, Private Google Access deep dive, Cloud NAT, Cloud Router, and Cloud VPN.

→ Next: [Chapter 7: VPC Advanced Networking](07-vpc-advanced.md)

---

*Last Updated: May 2026*
