# Chapter 13: Cloud Interconnect & VPN (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Connectivity Options](#part-1-connectivity-options)
- [Part 2: Cloud Router (BGP)](#part-2-cloud-router-bgp)
- [Part 3: HA VPN (Full Walkthrough)](#part-3-ha-vpn-full-walkthrough)
- [Part 4: Dedicated Interconnect](#part-4-dedicated-interconnect)
- [Part 5: Partner Interconnect](#part-5-partner-interconnect)
- [Part 6: Network Connectivity Center](#part-6-network-connectivity-center)
- [Part 7: Terraform Examples](#part-7-terraform-examples)
- [Part 8: gcloud CLI Reference](#part-8-gcloud-cli-reference)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

This chapter covers how to connect your on-premises network (office, data center) to GCP. There are three main options: Cloud VPN (encrypted tunnels over the internet), Dedicated Interconnect (direct physical connection), and Partner Interconnect (via a partner's network). Each has different performance, cost, and complexity trade-offs.

```
What you'll learn:
├── Connectivity Options Comparison
├── Cloud VPN
│   ├── HA VPN (recommended) — 99.99% SLA
│   ├── Classic VPN (legacy)
│   ├── VPN tunnels & IKE
│   ├── Cloud Router & BGP
│   ├── Full creation walkthrough
│   └── Routing options (dynamic BGP vs static)
├── Dedicated Interconnect
│   ├── Physical cross-connect at colocation facility
│   ├── VLAN attachments
│   ├── 10 Gbps / 100 Gbps links
│   └── Setup process
├── Partner Interconnect
│   ├── Via service provider
│   ├── When you can't reach a Google colocation
│   ├── 50 Mbps to 50 Gbps
│   └── Setup process
├── Cloud Router (BGP)
├── Network Connectivity Center
├── Comparison with AWS & Azure
├── Terraform examples
└── Real-world patterns
```

---

## Part 1: Connectivity Options

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONNECTIVITY OPTIONS COMPARISON                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┬───────────┬──────────┬──────────┬───────────┐ │
│ │ Feature          │ HA VPN    │ Dedicated│ Partner  │ Direct    │ │
│ │                  │           │ Intercon.│ Intercon.│ Peering   │ │
│ ├──────────────────┼───────────┼──────────┼──────────┼───────────┤ │
│ │ Connection       │ Internet  │ Physical │ Via      │ Physical  │ │
│ │                  │ (IPsec)   │ cross-   │ partner  │ (no SLA)  │ │
│ │                  │           │ connect  │          │           │ │
│ │ Encryption       │ Yes ✅    │ No ❌    │ No ❌   │ No ❌    │ │
│ │ (add MACsec)     │ (IPsec)   │ (add HA  │          │           │ │
│ │                  │           │ VPN over)│          │           │ │
│ │ SLA              │ 99.99%    │ 99.99%   │ 99.99%  │ No SLA    │ │
│ │ Bandwidth        │ 3 Gbps    │ 10/100   │ 50 Mbps │ 10 Gbps  │ │
│ │ per tunnel       │ per tunnel│ Gbps     │ to 50   │           │ │
│ │                  │ (multi-   │ per link │ Gbps    │           │ │
│ │                  │ tunnel)   │          │          │           │ │
│ │ Latency          │ Variable  │ Low      │ Low-Med │ Low       │ │
│ │                  │ (internet)│ (direct) │          │           │ │
│ │ Setup time       │ Minutes   │ Weeks    │ Days-   │ Days      │ │
│ │                  │           │          │ Weeks   │           │ │
│ │ Cost             │ Low       │ High     │ Medium  │ Free      │ │
│ │                  │ ~$36/mo   │ ~$1700/  │ varies  │ (peering) │ │
│ │                  │ per tunnel│ mo/10Gbps│ by      │           │ │
│ │                  │           │          │ partner │           │ │
│ │ Routing          │ BGP or    │ BGP      │ BGP     │ BGP       │ │
│ │                  │ static    │ (Cloud   │ (Cloud  │           │ │
│ │                  │           │ Router)  │ Router) │           │ │
│ │ Max connections  │ Unlimited │ Limited  │ Limited │ Limited   │ │
│ │                  │ tunnels   │ by colo  │ by      │           │ │
│ │                  │           │ facility │ partner │           │ │
│ └──────────────────┴───────────┴──────────┴──────────┴───────────┘ │
│                                                                       │
│ Decision guide:                                                      │
│ ├── Quick setup, low bandwidth → HA VPN                           │
│ ├── High bandwidth (>3 Gbps), low latency → Dedicated Interconnect│
│ ├── Can't reach Google colo facility → Partner Interconnect      │
│ ├── Need encryption → HA VPN (or Interconnect + VPN overlay)     │
│ └── Both HA VPN and Interconnect can coexist (backup)            │
│                                                                       │
│ AWS/Azure equivalents:                                               │
│ ┌──────────────────────┬──────────────────┬──────────────────────┐│
│ │ GCP                  │ AWS              │ Azure                ││
│ ├──────────────────────┼──────────────────┼──────────────────────┤│
│ │ HA VPN               │ Site-to-Site VPN │ VPN Gateway          ││
│ │ Cloud Router         │ VGW / TGW (BGP)  │ VPN Gateway (BGP)   ││
│ │ Dedicated Interconnect│ Direct Connect  │ ExpressRoute         ││
│ │ Partner Interconnect │ Direct Connect   │ ExpressRoute via     ││
│ │                      │ (via partner)    │ partner              ││
│ └──────────────────────┴──────────────────┴──────────────────────┘│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Cloud Router (BGP)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD ROUTER                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A fully managed BGP router that enables dynamic routing     │
│ between your VPC and on-premises networks.                         │
│                                                                       │
│ Used by:                                                             │
│ ├── HA VPN (announces VPC routes to on-prem)                     │
│ ├── Dedicated Interconnect (announces VPC routes)                │
│ ├── Partner Interconnect                                          │
│ └── Cloud NAT (uses Cloud Router for NAT config)                 │
│                                                                       │
│ Why BGP (dynamic routing):                                          │
│ ├── Automatic route exchange (no manual route tables)            │
│ ├── On-prem advertises its subnets → Cloud Router learns        │
│ ├── Cloud Router advertises VPC subnets → on-prem learns        │
│ ├── Add new subnet in VPC → automatically advertised            │
│ ├── Add new subnet on-prem → automatically learned              │
│ └── Failover: If link goes down, BGP detects + reroutes         │
│                                                                       │
│ Cloud Router modes:                                                  │
│ ├── Regional: Advertises subnets in same region only             │
│ └── Global: Advertises subnets from ALL regions in the VPC      │
│     ⚡ Use Global for multi-region VPCs                             │
│                                                                       │
│ Create:                                                              │
│ Console → Hybrid Connectivity → Cloud Routers → Create            │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Name: [cr-prod-asia]                                        │   │
│ │ Network: [vpc-prod ▼]                                      │   │
│ │ Region: [asia-south1 ▼]                                    │   │
│ │ Google ASN: [65001]                                         │   │
│ │   Range: 64512-65534 or 4200000000-4294967294 (private)    │   │
│ │   Must be different from on-prem ASN                        │   │
│ │                                                              │   │
│ │ BGP keepalive interval: [20] seconds (default)             │   │
│ │ Advertised routes:                                          │   │
│ │   ● All subnets visible to Cloud Router                    │   │
│ │   ○ Custom (select which subnets to advertise)             │   │
│ │                                                              │   │
│ │ Route advertisement mode:                                    │   │
│ │   ○ Regional (default)                                     │   │
│ │   ● Global (advertise subnets from all regions)            │   │
│ │                                                              │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ BGP session:                                                        │
│ ├── Created automatically when attaching VPN tunnel              │
│ ├── Or manually for Interconnect                                  │
│ ├── Each tunnel/VLAN has its own BGP session                     │
│ ├── BGP peer IP: Assigned from link-local range (169.254.x.x)  │
│ └── View status: Cloud Router → BGP sessions → Status: UP/DOWN │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: HA VPN (Full Walkthrough)

```
┌─────────────────────────────────────────────────────────────────────┐
│           HA VPN OVERVIEW                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ HA VPN provides 99.99% SLA with 2 tunnels (redundant).            │
│                                                                       │
│ Architecture:                                                        │
│                                                                       │
│ On-premises                              GCP                        │
│ ┌─────────────┐                    ┌──────────────────┐            │
│ │ VPN Gateway  │    IPsec tunnel 1 │ HA VPN Gateway   │            │
│ │ (your router)│ ◄═══════════════► │ Interface 0      │            │
│ │              │    IPsec tunnel 2 │ Interface 1      │            │
│ │              │ ◄═══════════════► │                  │            │
│ └─────────────┘                    └────────┬─────────┘            │
│                                             │                       │
│                                    ┌────────▼─────────┐            │
│                                    │ Cloud Router      │            │
│                                    │ (BGP sessions)    │            │
│                                    └────────┬─────────┘            │
│                                             │                       │
│                                    ┌────────▼─────────┐            │
│                                    │ VPC Network       │            │
│                                    │ (vpc-prod)        │            │
│                                    └──────────────────┘            │
│                                                                       │
│ HA VPN Gateway: 2 interfaces, each with its own public IP         │
│ ├── Interface 0: 35.x.x.x                                        │
│ ├── Interface 1: 35.y.y.y                                        │
│ └── 2 tunnels = 99.99% SLA (1 tunnel = 99.9%)                   │
│                                                                       │
│ Each tunnel: Up to 3 Gbps (total: 6 Gbps with 2 tunnels)        │
│ Add more HA VPN gateways for more bandwidth.                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

Console → Hybrid Connectivity → VPN → Create VPN connection

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: CREATE HA VPN GATEWAY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ VPN options:                                                    │
│   ● High-availability (HA) VPN  (recommended)                 │
│   ○ Classic VPN (legacy — no SLA guarantee)                   │
│                                                                   │
│ Name:    [vpn-gw-prod-asia]                                     │
│ Network: [vpc-prod ▼]                                          │
│ Region:  [asia-south1 ▼]                                       │
│ VPN gateway interfaces:                                        │
│   ● Two interfaces (0, 1)  — for 99.99% SLA                  │
│                                                                   │
│ [Create & continue]                                              │
│                                                                   │
│ Result:                                                          │
│ Interface 0 IP: 35.220.xx.xx                                   │
│ Interface 1 IP: 35.220.yy.yy                                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: ADD PEER VPN GATEWAY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Peer VPN gateway:                                               │
│   ● On-prem or Non-Google Cloud                                │
│   ○ Google Cloud (VPN between GCP VPCs)                       │
│                                                                   │
│ Name: [peer-gw-office]                                          │
│                                                                   │
│ Peer VPN gateway interfaces:                                    │
│   ● Two interfaces (your on-prem has 2 VPN endpoints)         │
│   ○ One interface                                              │
│   ○ Four interfaces                                            │
│                                                                   │
│   Interface 0 IP: [203.0.113.10] (your on-prem VPN device)   │
│   Interface 1 IP: [203.0.113.11] (second VPN device/IP)      │
│                                                                   │
│   If one on-prem device:                                       │
│   ● One interface                                              │
│   Interface 0 IP: [203.0.113.10]                              │
│   (Both GCP tunnels connect to same on-prem IP)              │
│                                                                   │
│ [Create & continue]                                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: ADD CLOUD ROUTER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Cloud Router:                                                    │
│   [Create new ▼]                                               │
│   Name: [cr-prod-asia]                                          │
│   Google ASN: [65001]                                           │
│   BGP keepalive: [20] seconds                                  │
│   Route advertisement: Global (all regions)                    │
│                                                                   │
│ [Create & continue]                                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 4: ADD VPN TUNNELS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Tunnel 1:                                                        │
│ Name: [tunnel-prod-asia-0]                                      │
│ Associated Cloud VPN gateway interface: [0 ▼]                 │
│ Associated peer VPN gateway interface: [0 ▼]                  │
│ IKE version: [IKEv2 ▼] (recommended)                         │
│ IKE pre-shared key:                                             │
│   ● Generate and copy                                          │
│   ○ Enter manually                                             │
│   Key: [xK9mP2qR7... ▼]                                      │
│   ⚠️ Copy this key! You'll need it to configure on-prem VPN.   │
│   ⚠️ Use a strong, random pre-shared key (32+ chars).           │
│                                                                   │
│ Tunnel 2:                                                        │
│ Name: [tunnel-prod-asia-1]                                      │
│ Associated Cloud VPN gateway interface: [1 ▼]                 │
│ Associated peer VPN gateway interface: [1 ▼]                  │
│ IKE version: [IKEv2 ▼]                                        │
│ IKE pre-shared key: [yL8nQ3rS... ▼]                          │
│ (Different key for each tunnel recommended)                    │
│                                                                   │
│ [Create & continue]                                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 5: CONFIGURE BGP SESSIONS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ BGP session for Tunnel 1:                                       │
│ Name: [bgp-session-0]                                           │
│ Peer ASN: [65002] (your on-prem ASN)                          │
│ Allocate BGP IPv4 address:                                     │
│   ● Automatically (recommended)                                │
│   ○ Manually                                                   │
│                                                                   │
│ Auto-assigned:                                                   │
│   Cloud Router BGP IP: 169.254.1.1                            │
│   Peer BGP IP: 169.254.1.2                                    │
│                                                                   │
│ BGP session for Tunnel 2:                                       │
│ Name: [bgp-session-1]                                           │
│ Peer ASN: [65002]                                               │
│ Cloud Router BGP IP: 169.254.2.1                               │
│ Peer BGP IP: 169.254.2.2                                       │
│                                                                   │
│ Advertised routes (per session):                                │
│   ● Use Cloud Router settings (all subnets)                   │
│   ○ Custom                                                     │
│                                                                   │
│ [Save BGP configuration]                                        │
│                                                                   │
│ Summary:                                                         │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Tunnel 0: vpn-gw:if0 (35.220.xx.xx) ←→ peer:if0          │ │
│ │   BGP: 169.254.1.1 (Cloud Router) ←→ 169.254.1.2 (peer) │ │
│ │   IKE: v2, PSK: xK9mP2qR7...                             │ │
│ │                                                             │ │
│ │ Tunnel 1: vpn-gw:if1 (35.220.yy.yy) ←→ peer:if1          │ │
│ │   BGP: 169.254.2.1 (Cloud Router) ←→ 169.254.2.2 (peer) │ │
│ │   IKE: v2, PSK: yL8nQ3rS...                               │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│ ⚠️ After creating on GCP side, you MUST configure the on-prem   │
│    VPN device with matching settings (IKE version, PSK, BGP    │
│    peer IPs, ASNs). Tunnel comes UP only when both sides match.│
│                                                                   │
│ Status monitoring:                                               │
│ VPN → Tunnels → Status: ESTABLISHED (green) or DOWN (red)     │
│ Cloud Router → BGP sessions → Status: UP or DOWN              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Dedicated Interconnect

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEDICATED INTERCONNECT                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A direct physical connection from your data center to        │
│ Google's network at a colocation facility.                         │
│                                                                       │
│ Physical setup:                                                      │
│                                                                       │
│ Your data center          Colocation facility       Google           │
│ ┌──────────────┐      ┌──────────────────────┐  ┌──────────────┐  │
│ │ Your router   │─────│ Your cage/rack       │  │ Google's     │  │
│ │              │      │    ↕ cross-connect   │──│ edge router  │  │
│ │              │      │ (physical fiber)     │  │              │  │
│ └──────────────┘      └──────────────────────┘  └──────┬───────┘  │
│                                                         │          │
│                                                  ┌──────▼───────┐  │
│                                                  │ Google Cloud  │  │
│                                                  │ (your VPC)   │  │
│                                                  └──────────────┘  │
│                                                                       │
│ Connection types:                                                    │
│ ├── 10 Gbps: Single 10G Ethernet link                             │
│ ├── 100 Gbps: Single 100G Ethernet link                           │
│ └── You can order multiple links (2x10G, 4x10G, etc.)           │
│                                                                       │
│ For 99.99% SLA:                                                     │
│ ├── 2 Interconnect connections in 2 different facilities         │
│ ├── Each with at least 2 VLAN attachments                        │
│ └── Total: 4 VLAN attachments minimum                             │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 99.99% SLA Topology:                                         │   │
│ │                                                              │   │
│ │ Colo Facility A                    Colo Facility B          │   │
│ │ ┌────────────────┐                ┌────────────────┐        │   │
│ │ │ Interconnect 1 │                │ Interconnect 2 │        │   │
│ │ │ VLAN attach 1a │──► VPC ◄──── │ VLAN attach 2a │        │   │
│ │ │ VLAN attach 1b │──► VPC ◄──── │ VLAN attach 2b │        │   │
│ │ └────────────────┘                └────────────────┘        │   │
│ │       ↑                                  ↑                   │   │
│ │       └──── Your router A ──── Your router B ────┘          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Setup process:                                                       │
│ 1. Choose colocation facility (near your data center)             │
│    Google's colocation facilities listed at:                       │
│    https://cloud.google.com/interconnect/docs/concepts/colocation │
│ 2. Order Interconnect in GCP console                               │
│ 3. Google provisions port → you get LOA-CFA (Letter of Auth)     │
│ 4. Take LOA to colocation → they install cross-connect            │
│ 5. Test link → light comes up                                     │
│ 6. Create VLAN attachment → attach to Cloud Router                │
│ 7. Configure BGP → routes exchanged → traffic flows              │
│                                                                       │
│ Timeline: 2-8 weeks (physical installation)                        │
│ Cost: ~$1,700/month per 10 Gbps link                              │
│       + data transfer ($0.02/GB egress)                            │
│                                                                       │
│ VLAN attachment (Interconnect attachment):                          │
│ ├── Maps to a specific VLAN on the physical link                 │
│ ├── Each attachment connects to one Cloud Router                 │
│ ├── Multiple attachments on one link → multiple VPCs            │
│ ├── Bandwidth: 50 Mbps to 50 Gbps per attachment               │
│ └── Each attachment gets its own BGP session                     │
│                                                                       │
│ ⚠️ Dedicated Interconnect is NOT encrypted by default!              │
│    Data travels over private link but in plain text.               │
│    For encryption: Run HA VPN over the Interconnect link.         │
│    (VPN tunnel through the dedicated link)                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Partner Interconnect

```
┌─────────────────────────────────────────────────────────────────────┐
│           PARTNER INTERCONNECT                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Connect to GCP via a service provider's network              │
│ instead of a direct physical cross-connect.                        │
│                                                                       │
│ Use when:                                                            │
│ ├── Your office/DC is NOT near a Google colocation facility       │
│ ├── You don't need 10/100 Gbps (need 50 Mbps - 50 Gbps)        │
│ ├── You already have a MPLS/WAN provider                         │
│ └── Faster setup than Dedicated (days vs weeks)                  │
│                                                                       │
│ Your office           Service Provider        Google               │
│ ┌──────────┐      ┌──────────────────┐    ┌──────────────┐       │
│ │ Your     │      │ Provider network │    │ Google Cloud  │       │
│ │ network  │──────│ (e.g., Tata,     │────│ (your VPC)   │       │
│ │          │      │  Airtel, AT&T)   │    │              │       │
│ └──────────┘      └──────────────────┘    └──────────────┘       │
│                                                                       │
│ Bandwidth: 50 Mbps, 100 Mbps, 200 Mbps, 500 Mbps,                │
│            1 Gbps, 2 Gbps, 5 Gbps, 10 Gbps, 20 Gbps, 50 Gbps   │
│                                                                       │
│ Setup:                                                               │
│ 1. Create VLAN attachment in GCP console (Partner type)           │
│ 2. Get pairing key from GCP                                       │
│ 3. Give pairing key to your service provider                      │
│ 4. Provider configures their side                                  │
│ 5. Approve the connection in GCP console                          │
│ 6. Configure BGP (via Cloud Router or pre-activated)              │
│                                                                       │
│ Two types:                                                           │
│ ├── Layer 2: Provider gives you a VLAN — you manage BGP         │
│ │   (More control, you configure Cloud Router BGP)              │
│ └── Layer 3: Provider manages BGP — you just get routes         │
│     (Simpler, provider handles routing)                          │
│                                                                       │
│ Cost: GCP charges ~$55/month per attachment                       │
│       + Provider charges (varies by provider and bandwidth)      │
│       + Data transfer                                              │
│                                                                       │
│ Partners (India examples):                                          │
│ ├── Tata Communications                                           │
│ ├── Bharti Airtel                                                  │
│ ├── Reliance Jio                                                   │
│ ├── Sify Technologies                                              │
│ └── Equinix, Megaport (global)                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Network Connectivity Center

```
┌─────────────────────────────────────────────────────────────────────┐
│           NETWORK CONNECTIVITY CENTER                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A hub-and-spoke model for connecting multiple sites          │
│ (on-prem, other clouds, GCP VPCs) through Google's backbone.     │
│                                                                       │
│ Think of it as GCP's version of AWS Transit Gateway.              │
│                                                                       │
│                  ┌────────────────────┐                             │
│                  │  Connectivity Hub   │                            │
│                  │  (Google backbone)  │                            │
│                  └────┬───┬───┬───┬───┘                            │
│                       │   │   │   │                                 │
│              ┌────────┘   │   │   └────────┐                       │
│              ▼            ▼   ▼            ▼                       │
│         ┌────────┐  ┌────────┐ ┌────────┐ ┌────────┐             │
│         │Office A│  │Office B│ │ AWS    │ │GCP VPC│             │
│         │(VPN)   │  │(Interc)│ │(VPN)   │ │(spoke) │             │
│         └────────┘  └────────┘ └────────┘ └────────┘             │
│                                                                       │
│ Use cases:                                                           │
│ ├── Connect 10+ offices through Google's backbone                │
│ ├── Multi-cloud: Connect AWS/Azure to GCP                        │
│ ├── Transit between on-prem sites via Google                     │
│ └── Replace expensive MPLS WAN with SD-WAN over GCP             │
│                                                                       │
│ Spokes:                                                              │
│ ├── VPN tunnels (HA VPN)                                          │
│ ├── Interconnect attachments (VLAN)                               │
│ ├── Router appliance (third-party SD-WAN VMs)                   │
│ └── VPC spokes (connect VPCs)                                    │
│                                                                       │
│ ⚡ Unique: Use Google's backbone as your WAN backbone!              │
│    Office A (India) ←→ Google backbone ←→ Office B (US)          │
│    Better than public internet, cheaper than MPLS.                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform Examples

```hcl
# HA VPN Gateway
resource "google_compute_ha_vpn_gateway" "prod" {
  name    = "vpn-gw-prod-asia"
  region  = "asia-south1"
  network = google_compute_network.prod.id
}

# Peer VPN Gateway (on-premises)
resource "google_compute_external_vpn_gateway" "office" {
  name            = "peer-gw-office"
  redundancy_type = "TWO_IPS_REDUNDANCY"

  interface {
    id         = 0
    ip_address = "203.0.113.10"
  }

  interface {
    id         = 1
    ip_address = "203.0.113.11"
  }
}

# Cloud Router
resource "google_compute_router" "prod" {
  name    = "cr-prod-asia"
  region  = "asia-south1"
  network = google_compute_network.prod.id

  bgp {
    asn               = 65001
    advertise_mode    = "DEFAULT"
    keepalive_interval = 20
  }
}

# VPN Tunnel 0
resource "google_compute_vpn_tunnel" "tunnel_0" {
  name                            = "tunnel-prod-asia-0"
  region                          = "asia-south1"
  vpn_gateway                     = google_compute_ha_vpn_gateway.prod.id
  vpn_gateway_interface           = 0
  peer_external_gateway           = google_compute_external_vpn_gateway.office.id
  peer_external_gateway_interface = 0
  shared_secret                   = var.vpn_shared_secret_0
  router                          = google_compute_router.prod.id
  ike_version                     = 2
}

# VPN Tunnel 1
resource "google_compute_vpn_tunnel" "tunnel_1" {
  name                            = "tunnel-prod-asia-1"
  region                          = "asia-south1"
  vpn_gateway                     = google_compute_ha_vpn_gateway.prod.id
  vpn_gateway_interface           = 1
  peer_external_gateway           = google_compute_external_vpn_gateway.office.id
  peer_external_gateway_interface = 1
  shared_secret                   = var.vpn_shared_secret_1
  router                          = google_compute_router.prod.id
  ike_version                     = 2
}

# BGP session for Tunnel 0
resource "google_compute_router_interface" "if_0" {
  name       = "bgp-if-0"
  router     = google_compute_router.prod.name
  region     = "asia-south1"
  ip_range   = "169.254.1.1/30"
  vpn_tunnel = google_compute_vpn_tunnel.tunnel_0.name
}

resource "google_compute_router_peer" "peer_0" {
  name                      = "bgp-peer-0"
  router                    = google_compute_router.prod.name
  region                    = "asia-south1"
  peer_ip_address           = "169.254.1.2"
  peer_asn                  = 65002
  advertised_route_priority = 100
  interface                 = google_compute_router_interface.if_0.name
}

# BGP session for Tunnel 1
resource "google_compute_router_interface" "if_1" {
  name       = "bgp-if-1"
  router     = google_compute_router.prod.name
  region     = "asia-south1"
  ip_range   = "169.254.2.1/30"
  vpn_tunnel = google_compute_vpn_tunnel.tunnel_1.name
}

resource "google_compute_router_peer" "peer_1" {
  name                      = "bgp-peer-1"
  router                    = google_compute_router.prod.name
  region                    = "asia-south1"
  peer_ip_address           = "169.254.2.2"
  peer_asn                  = 65002
  advertised_route_priority = 100
  interface                 = google_compute_router_interface.if_1.name
}

# Dedicated Interconnect VLAN attachment
resource "google_compute_interconnect_attachment" "prod" {
  name                     = "vlan-prod-asia"
  region                   = "asia-south1"
  interconnect             = "projects/my-project/global/interconnects/ic-prod-asia"
  router                   = google_compute_router.prod.id
  type                     = "DEDICATED"
  bandwidth                = "BPS_1G"
  vlan_tag8021q            = 100
  candidate_subnets        = ["169.254.10.0/29"]
  admin_enabled            = true
}
```

---

## Part 8: gcloud CLI Reference

```bash
# Create HA VPN gateway
gcloud compute vpn-gateways create vpn-gw-prod-asia \
  --network=vpc-prod \
  --region=asia-south1

# Create peer VPN gateway
gcloud compute external-vpn-gateways create peer-gw-office \
  --interfaces=0=203.0.113.10,1=203.0.113.11

# Create Cloud Router
gcloud compute routers create cr-prod-asia \
  --network=vpc-prod \
  --region=asia-south1 \
  --asn=65001

# Create VPN tunnels
gcloud compute vpn-tunnels create tunnel-prod-asia-0 \
  --region=asia-south1 \
  --vpn-gateway=vpn-gw-prod-asia \
  --vpn-gateway-region=asia-south1 \
  --peer-external-gateway=peer-gw-office \
  --peer-external-gateway-interface=0 \
  --vpn-gateway-interface=0 \
  --shared-secret=xK9mP2qR7... \
  --router=cr-prod-asia \
  --ike-version=2

# Create BGP session
gcloud compute routers add-interface cr-prod-asia \
  --region=asia-south1 \
  --interface-name=bgp-if-0 \
  --vpn-tunnel=tunnel-prod-asia-0 \
  --ip-address=169.254.1.1 \
  --mask-length=30

gcloud compute routers add-bgp-peer cr-prod-asia \
  --region=asia-south1 \
  --peer-name=bgp-peer-0 \
  --interface=bgp-if-0 \
  --peer-ip-address=169.254.1.2 \
  --peer-asn=65002

# Check tunnel status
gcloud compute vpn-tunnels describe tunnel-prod-asia-0 \
  --region=asia-south1 \
  --format="get(status,detailedStatus)"

# Check BGP session status
gcloud compute routers get-status cr-prod-asia \
  --region=asia-south1

# List Interconnect attachments
gcloud compute interconnects attachments list
```

---

## Part 9: Real-World Patterns

### Startup

```
Connectivity: HA VPN only

Setup:
├── 1 HA VPN gateway (asia-south1)
├── 2 tunnels to office VPN device (Ubiquiti/pfSense/FortiGate)
├── Cloud Router with BGP (dynamic routing)
├── On-prem routes: 192.168.0.0/16 (office network)
├── GCP routes: 10.0.0.0/16 (VPC subnets)
└── 2 tunnels = 99.99% SLA, up to 6 Gbps total

On-prem VPN device: FortiGate 60F or pfSense
(Most firewalls support IKEv2 + BGP)

Cost: ~$72/month (2 tunnels × $36)
```

### Mid-Size

```
Connectivity: HA VPN + Partner Interconnect

HA VPN (primary quick setup):
├── 2 HA VPN gateways (asia-south1, us-east1)
├── 4 tunnels per region (office + DR site)
└── Cloud Router with global routing

Partner Interconnect (higher bandwidth):
├── 1 Gbps via Tata Communications (Mumbai)
├── 2 VLAN attachments (redundancy)
├── BGP via Cloud Router
└── For database replication, bulk data transfer

Routing priority:
├── Interconnect: Lower MED (preferred path)
└── VPN: Higher MED (backup if Interconnect fails)

Cost: ~$200-500/month (VPN + Interconnect + provider)
```

### Enterprise

```
Connectivity: Dedicated Interconnect + HA VPN backup

Dedicated Interconnect:
├── 2 × 10 Gbps in Mumbai colocation (99.99% SLA)
├── 2 × 10 Gbps in Singapore colocation (99.99% SLA)
├── 4 VLAN attachments per Interconnect pair
├── VLANs: Prod VPC, Dev VPC, Shared Services VPC
└── Encrypted via HA VPN overlay (compliance)

HA VPN (backup):
├── 4 HA VPN gateways (4 regions)
├── Each with 2 tunnels (99.99%)
├── BGP priority: Higher MED than Interconnect
└── Auto-failover if Interconnect link fails

Network Connectivity Center:
├── Hub connecting 5 offices globally
├── Office traffic routed via Google backbone
├── Replacing legacy MPLS WAN
├── SD-WAN integration (Palo Alto Prisma)
└── 40% cost savings vs MPLS

Routing:
├── Global Cloud Router (all regions)
├── Custom route advertisements (security zones)
├── Shared VPC for centralized networking
└── BGP communities for route tagging

Cost: ~$5000-15000/month (Interconnect + VPN + data transfer)
```

---

## Quick Reference

| Feature | HA VPN | Dedicated Interconnect | Partner Interconnect |
|---------|--------|----------------------|---------------------|
| Type | IPsec over internet | Physical cross-connect | Via service provider |
| Bandwidth | 3 Gbps/tunnel | 10/100 Gbps/link | 50 Mbps-50 Gbps |
| Encryption | Yes (IPsec) | No (add VPN overlay) | No (add VPN overlay) |
| SLA | 99.99% (2 tunnels) | 99.99% (2 facilities) | 99.99% (redundant) |
| Latency | Variable (internet) | Low (direct) | Low-medium |
| Setup time | Minutes | Weeks | Days-weeks |
| Routing | BGP or static | BGP (Cloud Router) | BGP (Cloud Router) |
| Cost | ~$36/mo/tunnel | ~$1700/mo/10Gbps | ~$55/mo + provider |
| AWS equiv. | Site-to-Site VPN | Direct Connect | Direct Connect partner |
| Azure equiv. | VPN Gateway | ExpressRoute | ExpressRoute partner |

---

## Troubleshooting VPN Connectivity

VPN tunnels can fail for many reasons. Here's a systematic approach to diagnosing the most common issues.

### Tunnel Status: WAITING_FOR_FULL_CONFIG

This means your tunnel is created on the GCP side but isn't fully configured yet.

**Common causes:**
- The peer (on-premises) side hasn't been configured yet
- IKE pre-shared key mismatch between GCP and peer
- Peer gateway IP address is wrong
- The remote side's firewall is blocking UDP ports 500 and 4500

**How to fix:**

```bash
# Check tunnel status
gcloud compute vpn-tunnels describe my-tunnel \
    --region=us-central1 \
    --format="value(status,detailedStatus)"

# Verify tunnel configuration
gcloud compute vpn-tunnels describe my-tunnel \
    --region=us-central1
```

1. Verify the **peer IP address** matches the actual on-prem VPN device
2. Confirm the **pre-shared key** is identical on both sides (copy-paste, don't retype)
3. Ensure the on-prem firewall allows **UDP 500** (IKE) and **UDP 4500** (NAT-T)
4. Check that the on-prem VPN device is configured and running

### BGP Session DOWN

The tunnel is up (ESTABLISHED) but BGP isn't exchanging routes.

**Common causes:**
- BGP peer IP addresses are misconfigured
- ASN mismatch
- On-prem router isn't advertising routes
- Link-local IP range conflict

**How to fix:**

```bash
# Check BGP session status
gcloud compute routers get-status my-cloud-router \
    --region=us-central1

# Look at BGP peer details
gcloud compute routers describe my-cloud-router \
    --region=us-central1 \
    --format="yaml(bgpPeers)"
```

**Verify these match on both sides:**

| Setting | GCP Side | On-Prem Side |
|---------|----------|-------------|
| BGP ASN | Your GCP ASN (e.g., 65001) | Peer ASN (e.g., 65002) |
| Peer ASN | Peer ASN (e.g., 65002) | Your on-prem ASN (e.g., 65002) |
| BGP IP | e.g., 169.254.0.1/30 | e.g., 169.254.0.2/30 |
| Peer BGP IP | e.g., 169.254.0.2/30 | e.g., 169.254.0.1/30 |

> **💡 Tip:** The BGP link-local IPs (169.254.x.x/30) are assigned by GCP when you create the tunnel. Make sure the on-prem side uses the **exact same** /30 subnet with the correct peer IP.

### MTU Issues

Symptoms: Small packets work (ping), but large transfers fail or hang. SSH works, but SCP/SFTP stalls.

**Why it happens:**
- VPN tunnels add IPsec overhead, reducing the usable MTU
- HA VPN tunnels support a maximum MTU of **1460 bytes** (vs standard 1500)
- If PMTUD (Path MTU Discovery) is blocked, large packets get silently dropped

**How to fix:**

```bash
# Check the VPN tunnel MTU
gcloud compute vpn-tunnels describe my-tunnel \
    --region=us-central1 \
    --format="value(mtu)"

# Set VPC network MTU to match
gcloud compute networks update my-vpc --mtu=1460
```

1. Set the **VPC MTU to 1460** for networks using VPN tunnels
2. Ensure VMs in the VPC pick up the new MTU (may require reboot or `ip link set dev eth0 mtu 1460`)
3. Configure the on-prem side to use **MTU 1460** on the tunnel interface
4. Don't block ICMP "Fragmentation Needed" messages (needed for PMTUD)

### IKE Negotiation Failures

The tunnel stays in NEGOTIATION_FAILURE or keeps cycling between statuses.

**Common causes and fixes:**

| Issue | Solution |
|-------|----------|
| IKE version mismatch | GCP supports IKEv2 (preferred) and IKEv1 — match your on-prem device |
| Cipher suite mismatch | Use GCP's recommended ciphers: AES-256-CBC, SHA-256, DH Group 14+ |
| Pre-shared key mismatch | Copy the key from GCP Console, don't retype it |
| Lifetime mismatch | GCP uses 36000s (Phase 1) and 10800s (Phase 2) by default |
| NAT between devices | Ensure NAT-T is enabled (UDP 4500) on both sides |
| DPD (Dead Peer Detection) | GCP sends DPD — on-prem must respond to DPD keepalives |

```bash
# View detailed tunnel status for IKE errors
gcloud compute vpn-tunnels describe my-tunnel \
    --region=us-central1 \
    --format="yaml(status,detailedStatus,ikeVersion)"
```

> **💡 Tip:** GCP's VPN logs go to Cloud Logging. Filter with `resource.type="vpn_gateway"` to see IKE negotiation details, including which phase is failing.

### Quick Diagnostic Checklist

```
1. ✅ Tunnel status — is it ESTABLISHED?
2. ✅ BGP session — is it UP and exchanging routes?
3. ✅ Routes — do you see on-prem routes in the VPC route table?
4. ✅ Firewall rules — does GCP allow traffic to/from on-prem CIDRs?
5. ✅ On-prem firewall — does it allow traffic to/from GCP CIDRs?
6. ✅ MTU — is it set to 1460 on both sides?
7. ✅ Ping test — can you ping from a VM to an on-prem host?
```

---

## What's Next?

In the next chapter, we'll cover Compute Engine — GCP's virtual machine service.

→ Next: [Chapter 14: Compute Engine](14-compute-engine.md)

---

*Last Updated: May 2026*
