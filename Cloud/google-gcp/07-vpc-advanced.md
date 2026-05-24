# Chapter 7: VPC Advanced Networking (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: VPC Peering](#part-1-vpc-peering)
- [Part 2: Shared VPC](#part-2-shared-vpc)
- [Part 3: Cloud NAT](#part-3-cloud-nat)
- [Part 4: Cloud Router](#part-4-cloud-router)
- [Part 5: Cloud VPN](#part-5-cloud-vpn)
- [Part 6: Cloud Interconnect](#part-6-cloud-interconnect)
- [Part 7: Private Service Connect](#part-7-private-service-connect)
- [Part 8: VPC Flow Logs](#part-8-vpc-flow-logs)
- [Part 9: Network Intelligence Center](#part-9-network-intelligence-center)
- [Part 10: Real-World Architecture Patterns](#part-10-real-world-architecture-patterns)
- [Terraform Examples](#terraform-examples)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Building on VPC fundamentals (Chapter 6), this chapter covers advanced networking components: VPC Peering, Shared VPC, Cloud NAT, Cloud Router, Cloud VPN, Cloud Interconnect, and Private Service Connect.

```
What you'll learn:
├── VPC Peering
├── Shared VPC (GCP unique — powerful!)
├── Cloud NAT (deep dive)
├── Cloud Router (BGP)
├── Cloud VPN (HA VPN & Classic)
├── Cloud Interconnect (Dedicated & Partner)
├── Private Service Connect
├── Private Google Access (deep dive)
├── VPC Flow Logs
├── Network Intelligence Center
└── Real-world architecture patterns
```

---

## Part 1: VPC Peering

```
┌─────────────────────────────────────────────────────────────────────┐
│                     VPC PEERING (GCP)                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Connects two VPC networks so they can communicate using        │
│       internal IPs, even across projects and organizations.          │
│                                                                       │
│ ┌──────────────────┐      Peering     ┌──────────────────┐          │
│ │ prod-vpc          │ ◄═══════════►  │ dev-vpc            │          │
│ │ Project: tc-prod  │   Connection   │ Project: tc-dev    │          │
│ │ 10.0.0.0/16       │               │ 10.1.0.0/16        │          │
│ └──────────────────┘               └──────────────────┘          │
│                                                                       │
│ Key rules:                                                           │
│ ├── 1. NO transitive peering (same as AWS)                         │
│ │      A↔B and B↔C does NOT mean A↔C                               │
│ │                                                                    │
│ ├── 2. CIDRs cannot overlap (including subnet ranges)              │
│ │                                                                    │
│ ├── 3. Works cross-project and cross-organization                  │
│ │                                                                    │
│ ├── 4. Both sides must create peering config                       │
│ │      (no accept step — both sides configure, auto-connects)      │
│ │                                                                    │
│ ├── 5. Routes auto-exchanged (subnet routes imported by default)   │
│ │      Custom routes: Must enable import/export per peering        │
│ │                                                                    │
│ ├── 6. Firewall rules still apply                                  │
│ │      Must allow traffic from peer CIDR in firewall rules         │
│ │                                                                    │
│ ├── 7. Internal DNS does NOT resolve across peers                  │
│ │      (Use Private DNS zones for cross-VPC DNS)                   │
│ │                                                                    │
│ └── 8. Limit: 25 peering connections per VPC (can increase)        │
│                                                                       │
│ GCP VPC Peering vs AWS VPC Peering:                                  │
│ ┌──────────────────────┬──────────────┬─────────────────────┐      │
│ │ Feature              │ AWS          │ GCP                  │      │
│ │ Cross-region         │ Yes (paid)   │ N/A (VPC is global!) │      │
│ │ Route exchange       │ Manual       │ Automatic (subnets)  │      │
│ │ Accept step          │ Yes          │ No (both configure)  │      │
│ │ SG reference peer    │ Yes (same rgn)│ No (use CIDR)      │      │
│ │ Transitive           │ No           │ No                   │      │
│ └──────────────────────┴──────────────┴─────────────────────┘      │
│                                                                       │
│ ⚡ Since GCP VPCs are global, peering gives you global connectivity │
│    across all regions in both VPCs automatically!                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating VPC Peering

```
Console → VPC network → VPC network peering → Create connection

┌─────────────────────────────────────────────────────────────────┐
│           CREATE VPC NETWORK PEERING                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:                [prod-to-dev-peering]                       │
│ Your VPC network:    [prod-vpc ▼]                               │
│                                                                   │
│ Peered VPC network:                                              │
│ ○ In project [tc-dev]                                           │
│ ○ In another project                                            │
│   Project ID: [tc-dev]                                          │
│   VPC network name: [dev-vpc]                                   │
│                                                                   │
│ Exchange routes:                                                 │
│ ☑ Import custom routes                                          │
│ ☑ Export custom routes                                          │
│ ☑ Import subnet routes with public IP                          │
│ ☑ Export subnet routes with public IP                          │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
│ ⚠️ Must create peering on BOTH sides!                            │
│    Repeat in tc-dev project: dev-to-prod-peering                │
│    Status: "Inactive" until both sides created                  │
│    Status: "Active" when both sides configured                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # From prod project
  gcloud compute networks peerings create prod-to-dev \
    --network=prod-vpc \
    --peer-project=tc-dev \
    --peer-network=dev-vpc \
    --import-custom-routes \
    --export-custom-routes

  # From dev project
  gcloud compute networks peerings create dev-to-prod \
    --network=dev-vpc \
    --peer-project=tc-prod \
    --peer-network=prod-vpc \
    --import-custom-routes \
    --export-custom-routes
```

---

## Part 2: Shared VPC

```
┌─────────────────────────────────────────────────────────────────────┐
│                 SHARED VPC (GCP Unique!)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Share a VPC network across multiple GCP projects while         │
│       keeping resources in separate projects.                        │
│                                                                       │
│ This is one of GCP's MOST POWERFUL networking features!              │
│ Neither AWS nor Azure have a direct equivalent.                     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │                  ORGANIZATION: techcorp.com                   │    │
│ │                                                                │    │
│ │  HOST PROJECT (tc-shared-networking)                          │    │
│ │  ┌──────────────────────────────────────────────────────┐    │    │
│ │  │  Shared VPC: prod-vpc                                 │    │    │
│ │  │  ├── web-subnet (10.0.1.0/24)                        │    │    │
│ │  │  ├── app-subnet (10.0.2.0/24)                        │    │    │
│ │  │  ├── db-subnet (10.0.3.0/24)                         │    │    │
│ │  │  ├── Firewall rules (centralized!)                   │    │    │
│ │  │  ├── Cloud NAT (centralized!)                         │    │    │
│ │  │  └── Cloud Router                                     │    │    │
│ │  └──────────────────────────────────────────────────────┘    │    │
│ │      ↕              ↕              ↕                          │    │
│ │  SERVICE PROJECTS (use shared subnets)                        │    │
│ │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │    │
│ │  │ tc-prod  │  │ tc-staging│  │ tc-dev   │                   │    │
│ │  │ Backend  │  │ Backend   │  │ Backend  │                   │    │
│ │  │ VMs/GKE  │  │ VMs/GKE   │  │ VMs/GKE  │                   │    │
│ │  │ in       │  │ in        │  │ in       │                   │    │
│ │  │ app-subnet│  │ app-subnet│  │ app-subnet│                  │    │
│ │  └──────────┘  └──────────┘  └──────────┘                   │    │
│ │                                                                │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Benefits:                                                            │
│ ├── Centralized network management (one team controls network)     │
│ ├── Decentralized resource management (each team own project)      │
│ ├── Single set of firewall rules                                   │
│ ├── No VPC peering needed!                                         │
│ ├── No CIDR overlap issues (shared address space)                 │
│ ├── Centralized Cloud NAT, Cloud Router, VPN                      │
│ ├── Separation of duties: network admin ≠ project admin            │
│ └── IAM controls who can use which subnets                         │
│                                                                       │
│ When to use:                                                         │
│ ├── Multi-team organization (3+ teams)                             │
│ ├── Centralized network management requirement                     │
│ ├── GKE clusters that need shared networking                       │
│ └── When VPC Peering becomes too complex                            │
│                                                                       │
│ Shared VPC vs VPC Peering:                                           │
│ ┌──────────────────────┬──────────────┬──────────────────┐         │
│ │ Feature              │ Shared VPC   │ VPC Peering      │         │
│ │ Network admin        │ Centralized  │ Per VPC          │         │
│ │ IP space             │ Shared       │ Must not overlap │         │
│ │ Firewall rules       │ Centralized  │ Per VPC          │         │
│ │ Transitive           │ N/A (1 VPC)  │ No               │         │
│ │ Cross-org            │ No           │ Yes              │         │
│ │ Complexity           │ Lower        │ Higher at scale  │         │
│ └──────────────────────┴──────────────┴──────────────────┘         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Setting Up Shared VPC

```
Steps:

1. Enable Shared VPC on host project (Org Admin)
   Console → IAM & Admin → Shared VPC → Set up Shared VPC
   Select host project: tc-shared-networking
   [Enable Shared VPC host]

   CLI:
   gcloud compute shared-vpc enable tc-shared-networking

2. Associate service projects
   gcloud compute shared-vpc associated-projects add tc-prod \
     --host-project=tc-shared-networking

   gcloud compute shared-vpc associated-projects add tc-staging \
     --host-project=tc-shared-networking

3. Grant subnet-level IAM (which subnets each project can use)
   gcloud projects add-iam-policy-binding tc-shared-networking \
     --member="serviceAccount:tc-prod@tc-prod.iam.gserviceaccount.com" \
     --role="roles/compute.networkUser" \
     --condition='expression=resource.name.endsWith("app-subnet"),
       title=app-subnet-only'

   Or grant on specific subnet:
   gcloud compute networks subnets add-iam-policy-binding app-subnet \
     --region=asia-south1 \
     --project=tc-shared-networking \
     --member="serviceAccount:12345@cloudservices.gserviceaccount.com" \
     --role="roles/compute.networkUser"

4. Create resources in service project using shared subnet
   gcloud compute instances create app-vm-1 \
     --project=tc-prod \
     --zone=asia-south1-a \
     --subnet=projects/tc-shared-networking/regions/asia-south1/subnetworks/app-subnet \
     --no-address  # No external IP

IAM Roles needed:
├── Shared VPC Admin: Enable/manage Shared VPC (org-level)
├── Compute Network Admin: Manage subnets, FW, routes in host
├── Compute Network User: Use subnets (grant to service projects)
└── Compute Security Admin: Manage firewall rules only
```

---

## Part 3: Cloud NAT

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CLOUD NAT                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed NAT service for VMs without external IPs to           │
│       access the internet.                                           │
│                                                                       │
│ Key differences from AWS NAT Gateway:                                │
│ ┌──────────────────────┬──────────────────┬───────────────────┐    │
│ │ Feature              │ AWS NAT GW       │ GCP Cloud NAT     │    │
│ ├──────────────────────┼──────────────────┼───────────────────┤    │
│ │ Resource type        │ Gateway in subnet│ Software-defined   │    │
│ │ Deployed in          │ Public subnet    │ Cloud Router       │    │
│ │ AZ resilience        │ Single AZ        │ Regional (all zones)│   │
│ │ EIP required         │ Yes (1+ EIPs)    │ Auto or manual IPs │    │
│ │ Bandwidth            │ 5-100 Gbps       │ Dynamic scaling    │    │
│ │ HA                   │ Need 1 per AZ    │ Built-in regional  │    │
│ │ Pricing              │ $0.045/hr+data   │ ~$0.044/hr+data    │    │
│ └──────────────────────┴──────────────────┴───────────────────┘    │
│                                                                       │
│ ⚡ GCP Cloud NAT is regional and HA by default!                      │
│    No need to create one per zone (unlike AWS)                      │
│                                                                       │
│ How it works:                                                        │
│ [VM without external IP] → Cloud NAT → Internet                    │
│ Cloud NAT translates private IP → NAT IP (external)                 │
│ Responses come back through Cloud NAT → VM                         │
│                                                                       │
│ Cloud NAT does NOT require a route change!                          │
│ It's applied at the Cloud Router level (software-defined).          │
│ The default route 0.0.0.0/0 → internet gateway still works.       │
│ Cloud NAT intercepts traffic from VMs without external IPs.         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Cloud NAT

```
Console → Network services → Cloud NAT → Create Cloud NAT gateway

┌─────────────────────────────────────────────────────────────────┐
│              CREATE CLOUD NAT GATEWAY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Gateway name:     [prod-nat]                                    │
│ NAT type:         [Public ▼]                                    │
│ VPC network:      [prod-vpc ▼]                                  │
│ Region:           [asia-south1 ▼]                               │
│                                                                   │
│ Cloud Router:     [Create new router ▼]                         │
│   Router name:    [prod-router]                                 │
│   Network:        [prod-vpc]                                    │
│   Region:         [asia-south1]                                 │
│   BGP ASN:        [64512] (default, for dynamic routing)       │
│                                                                   │
│ Source (which subnets to NAT):                                  │
│ ● All subnets in the region                                    │
│ ○ Custom (select specific subnets)                              │
│   ☑ app-subnet                                                  │
│   ☑ db-subnet                                                   │
│   ☐ web-subnet (maybe has external IPs)                        │
│                                                                   │
│ NAT IP addresses:                                                │
│ ● Automatic (GCP allocates IPs, scales as needed)              │
│ ○ Manual (you specify static IPs — for whitelisting)           │
│   [Select or create static IPs]                                 │
│                                                                   │
│ Advanced configurations:                                         │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Minimum ports per VM:     [64] (default)                   │   │
│ │   └── Increase for VMs with many connections              │   │
│ │                                                            │   │
│ │ Maximum ports per VM:     [Not enabled ▼]                 │   │
│ │   └── Enable for dynamic port allocation                  │   │
│ │                                                            │   │
│ │ Enable Dynamic Port Allocation: ☑                         │   │
│ │   └── Auto-scales ports per VM based on usage             │   │
│ │                                                            │   │
│ │ NAT timeout configurations:                                │   │
│ │   UDP idle timeout:     30 sec (default)                  │   │
│ │   TCP established:      1200 sec (default)                │   │
│ │   TCP transitory:       30 sec (default)                  │   │
│ │   TCP time wait:        120 sec (default)                 │   │
│ │   ICMP idle:            30 sec (default)                  │   │
│ │                                                            │   │
│ │ Enable endpoint-independent mapping: ☐                    │   │
│ │   └── For WebRTC, gaming, P2P (consistent port mapping)  │   │
│ │                                                            │   │
│ │ Logging:                                                   │   │
│ │   ○ No logging                                            │   │
│ │   ○ Translation and errors                                │   │
│ │   ● Errors only                                           │   │
│ │   ○ Translation only                                      │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create Cloud Router first
  gcloud compute routers create prod-router \
    --network=prod-vpc \
    --region=asia-south1

  # Create Cloud NAT
  gcloud compute routers nats create prod-nat \
    --router=prod-router \
    --region=asia-south1 \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges \
    --enable-logging \
    --log-filter=ERRORS_ONLY \
    --enable-dynamic-port-allocation \
    --min-ports-per-vm=64

  # With manual IPs (for whitelisting)
  gcloud compute addresses create nat-ip-1 --region=asia-south1
  gcloud compute routers nats create prod-nat \
    --router=prod-router \
    --region=asia-south1 \
    --nat-external-ip-pool=nat-ip-1 \
    --nat-custom-subnet-ip-ranges=app-subnet,db-subnet

Pricing:
├── Per VM using NAT: ~$0.0044/hr per VM ≈ $3.17/month per VM
├── Per GB processed: $0.045/GB
├── NAT IP cost: Free while attached (charged if idle)
└── Minimum: ~$32/month + data charges (similar to AWS)
```

---

## Part 4: Cloud Router

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CLOUD ROUTER                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Fully managed router that dynamically exchanges routes         │
│       using BGP (Border Gateway Protocol).                           │
│                                                                       │
│ Used by:                                                             │
│ ├── Cloud NAT (required)                                            │
│ ├── Cloud VPN (HA VPN requires it)                                  │
│ ├── Cloud Interconnect                                              │
│ └── Router Appliance (3rd party NVAs)                               │
│                                                                       │
│ Key facts:                                                           │
│ ├── Regional resource                                               │
│ ├── Runs BGP to exchange routes with on-premises routers           │
│ ├── Supports dynamic route learning (no manual route updates!)     │
│ ├── ASN (Autonomous System Number) configurable                    │
│ ├── Custom route advertisements                                     │
│ ├── BFD (Bidirectional Forwarding Detection) for fast failover     │
│ └── FREE! (no charge for Cloud Router itself)                      │
│                                                                       │
│ Routing mode:                                                        │
│ ├── Regional: Routes only apply to subnets in router's region     │
│ └── Global: Routes apply to ALL subnets in VPC                     │
│     └── ⚡ Use Global for multi-region VPCs with VPN/Interconnect  │
│                                                                       │
│ Create:                                                              │
│ gcloud compute routers create prod-router \                          │
│   --network=prod-vpc \                                                │
│   --region=asia-south1 \                                              │
│   --asn=64512 \                                                       │
│   --advertisement-mode=CUSTOM \                                       │
│   --set-advertisement-ranges=10.0.0.0/16                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Cloud VPN

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CLOUD VPN                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Encrypted tunnel between your VPC and on-premises network     │
│                                                                       │
│ Two types:                                                           │
│                                                                       │
│ 1. HA VPN (recommended)                                              │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ ├── 99.99% SLA                                                │    │
│ │ ├── Two interfaces (2 external IPs)                          │    │
│ │ ├── Requires Cloud Router (BGP)                              │    │
│ │ ├── Supports 2 or 4 tunnels (HA configuration)              │    │
│ │ ├── Active/active or active/passive                          │    │
│ │ ├── Each tunnel: Up to 3 Gbps                               │    │
│ │ ├── ECMP for load balancing across tunnels                   │    │
│ │ └── ⚡ Use this for new deployments                           │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 2. Classic VPN (legacy, being deprecated)                            │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ ├── 99.9% SLA (lower than HA)                                │    │
│ │ ├── Single interface (1 external IP)                         │    │
│ │ ├── Supports static routing or BGP                           │    │
│ │ ├── Single tunnel per gateway                                │    │
│ │ └── ⚠️ Don't use for new deployments                         │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ HA VPN topology:                                                     │
│                                                                       │
│ GCP Side                              On-Premises                   │
│ ┌────────────────────┐               ┌────────────────────┐        │
│ │ HA VPN Gateway     │               │ Peer VPN Gateway   │        │
│ │ ┌────┐  ┌────┐     │               │                    │        │
│ │ │IF-0│  │IF-1│     │               │ ┌──────────────┐  │        │
│ │ └──┬─┘  └──┬─┘     │               │ │ Your router  │  │        │
│ │    │       │        │               │ │ (Cisco/etc.) │  │        │
│ │    │  ╔════╪════╗   │               │ └──────┬───────┘  │        │
│ │    ╠══╣ Tunnels ╠═══╪═══════════════╪════════╝          │        │
│ │    │  ╚════╪════╝   │  Encrypted    │                    │        │
│ │    │       │        │  IPSec        │                    │        │
│ │ Cloud Router        │               │                    │        │
│ │ (BGP sessions)      │               │                    │        │
│ └────────────────────┘               └────────────────────┘        │
│                                                                       │
│ Pricing:                                                             │
│ ├── HA VPN Gateway: $0.05/hr ≈ $36/month per gateway              │
│ ├── Tunnel: $0.075/hr per tunnel ≈ $54/month                       │
│ ├── Total HA (2 tunnels): ~$144/month                              │
│ └── Data: Standard egress rates                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating HA VPN

```
Console → Hybrid Connectivity → VPN → Create VPN connection

┌─────────────────────────────────────────────────────────────────┐
│             CREATE HA VPN                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ VPN options: ● High-availability (HA) VPN                       │
│              ○ Classic VPN                                       │
│                                                                   │
│ Step 1: HA VPN Gateway                                          │
│   Name:       [prod-vpn-gateway]                                │
│   Network:    [prod-vpc ▼]                                      │
│   Region:     [asia-south1 ▼]                                   │
│   IP stack:   [IPv4 ▼]                                          │
│   [Create and continue]                                          │
│   → Gets 2 external IPs (interface 0 and interface 1)          │
│                                                                   │
│ Step 2: Peer VPN Gateway                                        │
│   On-premises or Non-Google Cloud:                               │
│   Name:       [office-vpn-peer]                                 │
│   Interfaces: ○ One ● Two ○ Four                               │
│   Interface 0 IP: [203.0.113.1] (your router's public IP)     │
│   Interface 1 IP: [203.0.113.2] (your 2nd router, if HA)      │
│                                                                   │
│ Step 3: Cloud Router                                            │
│   [prod-router ▼] (or create new)                               │
│                                                                   │
│ Step 4: VPN tunnels                                             │
│   Tunnel 0:                                                      │
│     Name: [prod-tunnel-0]                                       │
│     Associated interface: [Interface 0 ▼]                       │
│     IKE version: [IKEv2 ▼]                                     │
│     IKE pre-shared key: [auto-generate or enter]               │
│     Cloud Router BGP session:                                   │
│       BGP session name: [bgp-session-0]                         │
│       Cloud Router ASN: [64512]                                 │
│       Peer ASN: [65001] (your on-prem AS number)               │
│       Cloud Router BGP IP: [169.254.0.1]                       │
│       Peer BGP IP: [169.254.0.2]                               │
│                                                                   │
│   Tunnel 1:                                                      │
│     (Same but different IPs and interface 1)                    │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
│ After creation:                                                  │
│ Configure your on-premises router with:                         │
│ ├── GCP gateway IPs (2 interfaces)                              │
│ ├── Pre-shared key                                               │
│ ├── BGP peer IPs                                                 │
│ └── IPSec settings                                               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create HA VPN Gateway
  gcloud compute vpn-gateways create prod-vpn-gw \
    --network=prod-vpc \
    --region=asia-south1

  # Create Peer VPN Gateway
  gcloud compute external-vpn-gateways create office-peer \
    --interfaces 0=203.0.113.1,1=203.0.113.2

  # Create VPN Tunnels
  gcloud compute vpn-tunnels create prod-tunnel-0 \
    --vpn-gateway=prod-vpn-gw \
    --vpn-gateway-region=asia-south1 \
    --peer-external-gateway=office-peer \
    --peer-external-gateway-interface=0 \
    --ike-version=2 \
    --shared-secret=SHARED_KEY_HERE \
    --router=prod-router \
    --region=asia-south1 \
    --interface=0

  # Create BGP Session
  gcloud compute routers add-interface prod-router \
    --interface-name=bgp-if-0 \
    --vpn-tunnel=prod-tunnel-0 \
    --ip-address=169.254.0.1 \
    --mask-length=30 \
    --region=asia-south1

  gcloud compute routers add-bgp-peer prod-router \
    --peer-name=bgp-peer-0 \
    --interface=bgp-if-0 \
    --peer-ip-address=169.254.0.2 \
    --peer-asn=65001 \
    --region=asia-south1
```

---

## Part 6: Cloud Interconnect

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CLOUD INTERCONNECT                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: High-bandwidth, low-latency connection between on-premises     │
│       and GCP — NOT over the internet.                               │
│                                                                       │
│ Two types:                                                           │
│                                                                       │
│ 1. DEDICATED INTERCONNECT                                            │
│ ├── Direct physical connection at Google colocation facility       │
│ ├── Bandwidth: 10 Gbps or 100 Gbps per link                      │
│ ├── Can bundle up to 8 links (800 Gbps max!)                     │
│ ├── 99.9% or 99.99% SLA (depends on topology)                    │
│ ├── Requires colocation with Google at edge PoP                   │
│ ├── Takes weeks to set up                                         │
│ └── For: Large enterprises with 10+ Gbps needs                   │
│                                                                       │
│ 2. PARTNER INTERCONNECT                                              │
│ ├── Connect via a service provider (Equinix, Megaport, etc.)     │
│ ├── Bandwidth: 50 Mbps to 50 Gbps                                │
│ ├── Don't need colocation with Google                             │
│ ├── 99.9% or 99.99% SLA                                          │
│ ├── VLAN attachment via provider                                  │
│ └── For: Smaller bandwidth needs or no Google PoP nearby         │
│                                                                       │
│ Cloud VPN vs Cloud Interconnect:                                     │
│ ┌──────────────────────┬──────────────────┬───────────────────┐    │
│ │ Feature              │ Cloud VPN        │ Cloud Interconnect│    │
│ ├──────────────────────┼──────────────────┼───────────────────┤    │
│ │ Connection           │ Over internet    │ Private/dedicated │    │
│ │ Bandwidth            │ 3 Gbps/tunnel    │ 10-100 Gbps/link │    │
│ │ Latency              │ Variable         │ Low, consistent   │    │
│ │ Encryption           │ Built-in (IPSec) │ Not encrypted*    │    │
│ │ Setup time           │ Minutes          │ Days/weeks        │    │
│ │ Cost                 │ Lower            │ Higher            │    │
│ │ SLA                  │ 99.99% (HA)      │ 99.9/99.99%      │    │
│ └──────────────────────┴──────────────────┴───────────────────┘    │
│ * Add HA VPN over Interconnect for encryption if needed            │
│                                                                       │
│ Best practice: Interconnect + VPN backup                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Private Service Connect

```
┌─────────────────────────────────────────────────────────────────────┐
│               PRIVATE SERVICE CONNECT (PSC)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Access Google APIs and services using a private endpoint       │
│       with an internal IP in your VPC (like AWS PrivateLink).        │
│                                                                       │
│ Two use cases:                                                       │
│                                                                       │
│ 1. Access Google APIs privately                                      │
│ ├── Create a PSC endpoint for googleapis.com                       │
│ ├── Gets a private IP in your VPC (e.g., 10.0.5.10)              │
│ ├── DNS resolves googleapis.com to your private IP                 │
│ ├── Replaces: Private Google Access (more control)                │
│ └── Works from on-premises via VPN/Interconnect!                  │
│     (Unlike Private Google Access)                                  │
│                                                                       │
│ 2. Publish/consume your own services                                 │
│ ├── Expose your service to other VPCs/projects privately           │
│ ├── No VPC peering or Shared VPC needed                            │
│ ├── CIDRs can overlap (no conflict!)                               │
│ ├── Service producer creates a Service Attachment                  │
│ ├── Consumer creates a PSC endpoint                                │
│ └── Similar to AWS PrivateLink concept                             │
│                                                                       │
│ PSC for Google APIs:                                                 │
│ gcloud compute addresses create psc-google-apis \                    │
│   --global \                                                          │
│   --purpose=PRIVATE_SERVICE_CONNECT \                                 │
│   --addresses=10.0.5.10 \                                             │
│   --network=prod-vpc                                                  │
│                                                                       │
│ gcloud compute forwarding-rules create psc-google-apis-rule \        │
│   --global \                                                          │
│   --network=prod-vpc \                                                │
│   --address=psc-google-apis \                                         │
│   --target-google-apis-bundle=all-apis                                │
│                                                                       │
│ Private Google Access vs PSC:                                        │
│ ┌──────────────────────┬─────────────────┬───────────────────┐     │
│ │ Feature              │ PGA             │ PSC               │     │
│ │ Access from on-prem  │ Not easily      │ Yes!              │     │
│ │ Custom IP            │ Google IPs      │ Your VPC IP       │     │
│ │ Per-VPC granularity  │ Per subnet      │ Per endpoint      │     │
│ │ Cost                 │ Free            │ Small charge      │     │
│ │ Publish own services │ No              │ Yes               │     │
│ └──────────────────────┴─────────────────┴───────────────────┘     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: VPC Flow Logs

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VPC FLOW LOGS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Captures samples of network flows at the subnet level          │
│                                                                       │
│ Enabled per subnet (not per VPC or per VM):                         │
│                                                                       │
│ Configure during subnet creation or edit:                            │
│   Flow Logs: On                                                      │
│   Aggregation interval: [5 sec ▼] (5s, 30s, 1min, 5min, 10min)   │
│   Include metadata: ☑                                               │
│   Sample rate: 50% (1.0 = all, 0.5 = half, reduces cost)          │
│   Filter: All (or specific expression)                              │
│                                                                       │
│ Stored in: Cloud Logging                                             │
│                                                                       │
│ Log fields:                                                          │
│ ├── connection: src_ip, src_port, dest_ip, dest_port, protocol    │
│ ├── bytes_sent, packets_sent                                       │
│ ├── start_time, end_time                                           │
│ ├── reporter: SRC or DEST                                          │
│ ├── VM instance details                                             │
│ └── Geographic info                                                 │
│                                                                       │
│ Use cases:                                                           │
│ ├── Network monitoring and troubleshooting                         │
│ ├── Security analysis (detect unusual patterns)                    │
│ ├── Cost optimization (find top traffic sources)                   │
│ └── Compliance (audit network access)                               │
│                                                                       │
│ Enable:                                                              │
│ gcloud compute networks subnets update app-subnet \                  │
│   --region=asia-south1 \                                              │
│   --enable-flow-logs \                                                │
│   --logging-aggregation-interval=interval-5-sec \                     │
│   --logging-flow-sampling=0.5 \                                       │
│   --logging-metadata=include-all                                      │
│                                                                       │
│ Pricing: Standard Cloud Logging rates ($0.50/GB)                    │
│ 💡 Use sampling (0.5) and longer intervals to reduce cost           │
│                                                                       │
│ Query in Cloud Logging:                                              │
│ resource.type="gce_subnetwork"                                       │
│ logName="projects/tc-prod/logs/compute.googleapis.com%2Fvpc_flows"  │
│ jsonPayload.connection.dest_port=3306                               │
│                                                                       │
│ Export to BigQuery for advanced analysis:                            │
│ ├── Create a Logs Router sink → BigQuery dataset                   │
│ ├── Run SQL queries on flow data                                   │
│ └── Build dashboards in Looker Studio                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Network Intelligence Center

```
┌─────────────────────────────────────────────────────────────────────┐
│              NETWORK INTELLIGENCE CENTER                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A suite of tools for network monitoring, troubleshooting,      │
│       and optimization.                                              │
│                                                                       │
│ Tools:                                                               │
│ ├── Network Topology: Visual map of your VPC resources             │
│ │   └── Shows VMs, subnets, regions, connections, traffic flow    │
│ │                                                                    │
│ ├── Connectivity Tests: Test if A can reach B                      │
│ │   └── Source: VM / IP  →  Destination: VM / IP / LB / URL       │
│ │   └── Shows: Pass/fail and which hop blocks traffic              │
│ │   └── Tests firewall rules, routes, without sending packets     │
│ │                                                                    │
│ ├── Performance Dashboard: Real-time latency and packet loss       │
│ │   └── Between zones, regions, and to Google services             │
│ │                                                                    │
│ ├── Firewall Insights: Analyze firewall rule usage                 │
│ │   └── Shadowed rules (rule that never matches)                  │
│ │   └── Overly permissive rules                                   │
│ │   └── Hit count per rule                                         │
│ │                                                                    │
│ └── Network Analyzer: Automated network configuration checks      │
│     └── Misconfigurations, unused resources, best practices       │
│                                                                       │
│ Most useful: Connectivity Tests                                      │
│ gcloud network-management connectivity-tests create test-app-to-db \│
│   --source-instance=projects/tc-prod/zones/asia-south1-a/           │
│     instances/app-vm-1 \                                             │
│   --destination-instance=projects/tc-prod/zones/asia-south1-a/      │
│     instances/db-vm-1 \                                              │
│   --protocol=TCP \                                                    │
│   --destination-port=5432                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Real-World Architecture Patterns

### Startup: Single VPC with Cloud NAT

```
┌─────────────────────────────────────────────────────────────────┐
│                    STARTUP PATTERN                                │
│                                                                   │
│  VPC: prod-vpc (global, custom mode)                            │
│  ├── Subnets in asia-south1: web, app, db                     │
│  ├── Cloud NAT (regional, HA built-in)                         │
│  ├── Private Google Access on all subnets                      │
│  ├── Firewall rules: allow-internal, allow-ssh-iap, allow-http│
│  ├── VPC Flow Logs on app and db subnets                       │
│  └── No VPN/Interconnect needed yet                            │
│                                                                   │
│  Monthly cost: ~$32 (Cloud NAT) + flow logs                    │
│                                                                   │
│  No peering, no Shared VPC needed                               │
└─────────────────────────────────────────────────────────────────┘
```

### Mid-Size: Shared VPC

```
┌─────────────────────────────────────────────────────────────────┐
│                  MID-SIZE PATTERN                                 │
│                                                                   │
│  Shared VPC (host project: tc-networking)                       │
│  ├── Subnets per environment per tier                          │
│  ├── Centralized Cloud NAT (one per region)                    │
│  ├── Centralized Cloud Router                                  │
│  ├── Centralized firewall rules                                │
│  ├── HA VPN to office                                          │
│  │                                                              │
│  ├── Service project: tc-prod → uses prod subnets              │
│  ├── Service project: tc-staging → uses staging subnets        │
│  └── Service project: tc-dev → uses dev subnets                │
│                                                                   │
│  Private Service Connect for Google APIs                        │
│  VPC Flow Logs on production subnets                            │
│  Network Intelligence Center for monitoring                     │
│                                                                   │
│  Monthly cost: ~$150-400 networking                             │
└─────────────────────────────────────────────────────────────────┘
```

### Enterprise: Shared VPC + Interconnect

```
┌─────────────────────────────────────────────────────────────────┐
│                 ENTERPRISE PATTERN                                │
│                                                                   │
│  Shared VPC (host project: tc-networking)                       │
│  ├── Multi-region subnets                                      │
│  ├── Dedicated Interconnect (10 Gbps to data center)          │
│  ├── HA VPN as backup to Interconnect                         │
│  ├── Cloud Router with BGP (dynamic routes)                   │
│  ├── Hierarchical firewall policies (org → folder → VPC)     │
│  ├── Private Service Connect for all Google APIs              │
│  ├── VPC Flow Logs → BigQuery for analysis                    │
│  ├── Network Intelligence Center (all tools)                  │
│  ├── Cloud IDS for intrusion detection                        │
│  └── Packet Mirroring for advanced inspection                 │
│                                                                   │
│  Service projects: 20+ per BU/team                              │
│  NCC (Network Connectivity Center) for multi-cloud             │
│                                                                   │
│  Monthly cost: $1,000-10,000+ networking                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Component | Purpose | Cost |
|-----------|---------|------|
| VPC Peering | Connect 2 VPCs | Free + data transfer |
| Shared VPC | Share network across projects | FREE |
| Cloud NAT | Private VMs → internet | ~$32/mo + $0.045/GB |
| Cloud Router | BGP routing | FREE |
| HA VPN | Encrypted tunnel to on-prem | ~$144/mo (2 tunnels) |
| Classic VPN | Legacy VPN | ~$36/mo |
| Dedicated Interconnect | Physical 10/100 Gbps link | Port + partner fees |
| Partner Interconnect | Via service provider | Partner fees |
| Private Service Connect | Private endpoint for services | Small charge |
| Private Google Access | Private VMs → Google APIs | FREE |
| VPC Flow Logs | Network traffic logging | $0.50/GB |
| Connectivity Tests | Test A→B reachability | FREE |

---

## Terraform Examples

### VPC Peering

```hcl
# VPC Peering between prod and dev projects
# ⚠️ Must create peering on BOTH sides!
resource "google_compute_network_peering" "prod_to_dev" {
  name         = "prod-to-dev"
  network      = google_compute_network.prod.self_link
  peer_network = google_compute_network.dev.self_link

  export_custom_routes                = true
  import_custom_routes                = true
  export_subnet_routes_with_public_ip = true
  import_subnet_routes_with_public_ip = true
}

resource "google_compute_network_peering" "dev_to_prod" {
  name         = "dev-to-prod"
  network      = google_compute_network.dev.self_link
  peer_network = google_compute_network.prod.self_link

  export_custom_routes                = true
  import_custom_routes                = true
  export_subnet_routes_with_public_ip = true
  import_subnet_routes_with_public_ip = true
}

# Cross-project peering
resource "google_compute_network_peering" "prod_to_shared" {
  name         = "prod-to-shared"
  network      = "projects/tc-prod/global/networks/prod-vpc"
  peer_network = "projects/tc-shared/global/networks/shared-vpc"

  export_custom_routes = true
  import_custom_routes = true
}
```

### Cloud NAT

```hcl
# Cloud Router (required by Cloud NAT)
resource "google_compute_router" "prod" {
  name    = "prod-router"
  network = google_compute_network.prod.id
  region  = "asia-south1"

  bgp {
    asn = 64512
  }
}

# Cloud NAT with auto IPs (simplest setup)
resource "google_compute_router_nat" "prod" {
  name                               = "prod-nat"
  router                             = google_compute_router.prod.name
  region                             = google_compute_router.prod.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  enable_dynamic_port_allocation = true
  min_ports_per_vm               = 64

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}

# Cloud NAT with manual IPs and specific subnets
# (use when you need static IPs for whitelisting)
resource "google_compute_address" "nat_ip" {
  name   = "nat-ip-1"
  region = "asia-south1"
}

resource "google_compute_router_nat" "prod_manual" {
  name                               = "prod-nat-manual"
  router                             = google_compute_router.prod.name
  region                             = google_compute_router.prod.region
  nat_ip_allocate_option             = "MANUAL_ONLY"
  nat_ips                            = [google_compute_address.nat_ip.self_link]
  source_subnetwork_ip_ranges_to_nat = "LIST_OF_SUBNETWORKS"

  subnetwork {
    name                    = google_compute_subnetwork.app.id
    source_ip_ranges_to_nat = ["ALL_IP_RANGES"]
  }

  subnetwork {
    name                    = google_compute_subnetwork.db.id
    source_ip_ranges_to_nat = ["ALL_IP_RANGES"]
  }

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}
```

### Cloud Router

```hcl
# Cloud Router with custom BGP advertisements
resource "google_compute_router" "prod_custom" {
  name    = "prod-router"
  network = google_compute_network.prod.id
  region  = "asia-south1"

  bgp {
    asn               = 64512
    advertise_mode    = "CUSTOM"
    advertised_groups = ["ALL_SUBNETS"]

    advertised_ip_ranges {
      range       = "10.0.0.0/16"
      description = "Production VPC range"
    }

    advertised_ip_ranges {
      range       = "10.1.0.0/16"
      description = "Shared services range"
    }
  }
}

# Cloud Router interface + BGP peer (for HA VPN)
resource "google_compute_router_interface" "vpn_if_0" {
  name       = "bgp-if-0"
  router     = google_compute_router.prod_custom.name
  region     = "asia-south1"
  vpn_tunnel = google_compute_vpn_tunnel.tunnel_0.name
  ip_range   = "169.254.0.1/30"
}

resource "google_compute_router_peer" "bgp_peer_0" {
  name                      = "bgp-peer-0"
  router                    = google_compute_router.prod_custom.name
  region                    = "asia-south1"
  interface                 = google_compute_router_interface.vpn_if_0.name
  peer_ip_address           = "169.254.0.2"
  peer_asn                  = 65001
  advertised_route_priority = 100
}
```

### Shared VPC (Host Project + Service Project)

```hcl
# ──────────────────────────────────────────────
# Step 1: Enable Shared VPC on host project
# ──────────────────────────────────────────────
resource "google_compute_shared_vpc_host_project" "host" {
  project = "tc-shared-networking"
}

# ──────────────────────────────────────────────
# Step 2: Attach service projects
# ──────────────────────────────────────────────
resource "google_compute_shared_vpc_service_project" "prod" {
  host_project    = google_compute_shared_vpc_host_project.host.project
  service_project = "tc-prod"
}

resource "google_compute_shared_vpc_service_project" "staging" {
  host_project    = google_compute_shared_vpc_host_project.host.project
  service_project = "tc-staging"
}

# ──────────────────────────────────────────────
# Step 3: Create VPC and subnets in host project
# ──────────────────────────────────────────────
resource "google_compute_network" "shared" {
  name                    = "shared-vpc"
  project                 = "tc-shared-networking"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "web" {
  name          = "web-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = "asia-south1"
  network       = google_compute_network.shared.id
  project       = "tc-shared-networking"

  private_ip_google_access = true
}

resource "google_compute_subnetwork" "app" {
  name          = "app-subnet"
  ip_cidr_range = "10.0.2.0/24"
  region        = "asia-south1"
  network       = google_compute_network.shared.id
  project       = "tc-shared-networking"

  private_ip_google_access = true

  log_config {
    aggregation_interval = "INTERVAL_5_SEC"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

resource "google_compute_subnetwork" "db" {
  name          = "db-subnet"
  ip_cidr_range = "10.0.3.0/24"
  region        = "asia-south1"
  network       = google_compute_network.shared.id
  project       = "tc-shared-networking"

  private_ip_google_access = true
}

# ──────────────────────────────────────────────
# Step 4: Grant networkUser on specific subnets
# ──────────────────────────────────────────────
resource "google_compute_subnetwork_iam_member" "prod_app" {
  project    = "tc-shared-networking"
  region     = "asia-south1"
  subnetwork = google_compute_subnetwork.app.name
  role       = "roles/compute.networkUser"
  member     = "serviceAccount:tc-prod@tc-prod.iam.gserviceaccount.com"
}

resource "google_compute_subnetwork_iam_member" "prod_db" {
  project    = "tc-shared-networking"
  region     = "asia-south1"
  subnetwork = google_compute_subnetwork.db.name
  role       = "roles/compute.networkUser"
  member     = "serviceAccount:tc-prod@tc-prod.iam.gserviceaccount.com"
}

# ──────────────────────────────────────────────
# Step 5: Create VM in service project using shared subnet
# ──────────────────────────────────────────────
resource "google_compute_instance" "app_vm" {
  project      = "tc-prod"
  name         = "app-vm-1"
  machine_type = "e2-medium"
  zone         = "asia-south1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    subnetwork         = google_compute_subnetwork.app.self_link
    subnetwork_project = "tc-shared-networking"
    # No access_config block = no external IP
  }
}

# Centralized Cloud NAT in host project
resource "google_compute_router" "shared" {
  name    = "shared-router"
  network = google_compute_network.shared.id
  region  = "asia-south1"
  project = "tc-shared-networking"

  bgp {
    asn = 64512
  }
}

resource "google_compute_router_nat" "shared" {
  name                               = "shared-nat"
  router                             = google_compute_router.shared.name
  region                             = google_compute_router.shared.region
  project                            = "tc-shared-networking"
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}
```

---

## What's Next?

In the next chapter, we'll cover Firewall Rules & Policies — ingress/egress rules, priority, network tags, service accounts, and hierarchical firewall policies.

→ Next: [Chapter 8: Firewall Rules & Policies](08-firewall-rules.md)

---

*Last Updated: May 2026*
