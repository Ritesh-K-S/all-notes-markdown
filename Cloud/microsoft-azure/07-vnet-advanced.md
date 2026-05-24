# Chapter 7: VNet Advanced Networking (Azure)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: VNet Peering](#part-1-vnet-peering)
- [Part 2: VPN Gateway](#part-2-vpn-gateway)
- [Part 3: ExpressRoute](#part-3-expressroute)
- [Part 4: Azure Virtual WAN](#part-4-azure-virtual-wan)
- [Part 5: Azure Private Link & Private Endpoints](#part-5-azure-private-link--private-endpoints)
- [Part 6: Azure Bastion](#part-6-azure-bastion)
- [Part 7: User Defined Routes (UDR)](#part-7-user-defined-routes-udr)
- [Part 8: Network Watcher & NSG Flow Logs](#part-8-network-watcher--nsg-flow-logs)
- [Part 9: Real-World Architecture Patterns](#part-9-real-world-architecture-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Building on VNet fundamentals (Chapter 6), this chapter covers advanced networking: VNet Peering, VPN Gateway, ExpressRoute, Virtual WAN, Private Link, Azure Bastion, UDRs, and Network Watcher.

```
What you'll learn:
├── VNet Peering (regional & global)
├── VPN Gateway (site-to-site, point-to-site)
├── ExpressRoute
├── Azure Virtual WAN
├── Azure Private Link & Private Endpoints
├── Azure Bastion
├── User Defined Routes (UDR)
├── NSG Flow Logs & Network Watcher
├── Azure Firewall (overview)
└── Real-world architecture patterns
```

---

## Part 1: VNet Peering

```
┌─────────────────────────────────────────────────────────────────────┐
│                     VNET PEERING                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Connects two VNets so they can communicate using private IPs  │
│                                                                       │
│ ┌──────────────────┐      Peering     ┌──────────────────┐          │
│ │ prod-vnet         │ ◄═══════════►  │ shared-vnet       │          │
│ │ Sub: rg-prod      │   Connection   │ Sub: rg-shared    │          │
│ │ 10.0.0.0/16       │               │ 10.3.0.0/16       │          │
│ └──────────────────┘               └──────────────────┘          │
│                                                                       │
│ Two types:                                                           │
│ ├── VNet Peering: Same region                                      │
│ └── Global VNet Peering: Cross-region                               │
│     (Works! But costs more for data transfer)                       │
│                                                                       │
│ Key rules:                                                           │
│ ├── 1. NO transitive peering (same as AWS & GCP)                   │
│ │      A↔B and B↔C does NOT mean A↔C                               │
│ │      ⚡ Unless: Gateway Transit is enabled (see below)            │
│ │                                                                    │
│ ├── 2. CIDRs cannot overlap                                       │
│ │                                                                    │
│ ├── 3. Works cross-subscription and cross-tenant                   │
│ │                                                                    │
│ ├── 4. Both sides must create peering                              │
│ │      Azure creates link on BOTH sides (auto in portal)           │
│ │                                                                    │
│ ├── 5. NSG rules still apply                                       │
│ │                                                                    │
│ ├── 6. Gateway Transit: Share VPN/ExpressRoute gateway             │
│ │      Hub VNet has VPN Gateway → Spoke VNets use it              │
│ │      Hub: "Allow gateway transit" = ON                           │
│ │      Spoke: "Use remote gateways" = ON                           │
│ │      ⚡ Gives "transitive" behavior through gateway!              │
│ │                                                                    │
│ └── 7. Cost:                                                        │
│        Same region: $0.01/GB each direction                        │
│        Global (cross-region): $0.035-0.075/GB                      │
│                                                                       │
│ Azure vs AWS vs GCP Peering:                                         │
│ ┌─────────────────────┬────────┬────────┬───────────────────┐     │
│ │ Feature             │ AWS    │ GCP    │ Azure             │     │
│ │ Cross-region        │ Yes    │ Global │ Yes (Global)      │     │
│ │ Gateway transit     │ No     │ No     │ Yes!              │     │
│ │ Transitive          │ No     │ No     │ Via gateway only  │     │
│ │ Setup               │ Req+Acc│ Both   │ Both (auto)       │     │
│ │ Cross-account       │ Yes    │ Yes    │ Yes (sub/tenant)  │     │
│ └─────────────────────┴────────┴────────┴───────────────────┘     │
│                                                                       │
│ ⚡ Gateway Transit is Azure's unique advantage — enables            │
│    hub-and-spoke WITHOUT a dedicated appliance!                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating VNet Peering

```
Console → Virtual networks → prod-vnet → Peerings → Add

┌─────────────────────────────────────────────────────────────────┐
│                  ADD PEERING                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ This virtual network                                             │
│ ─────────────────────────────────────────────────────────       │
│ Peering link name:        [prod-to-shared]                      │
│                                                                   │
│ Traffic to remote VNet:   ● Allow                               │
│ Traffic forwarded from                                           │
│   remote VNet:            ○ Block  ● Allow                      │
│   └── Allows traffic from VPN/ER on remote VNet to pass through│
│                                                                   │
│ Virtual network gateway                                          │
│   or Route Server:        ○ None  ○ Use this VNet's gateway    │
│                            ● Use the remote VNet's gateway      │
│   └── Gateway Transit setting!                                  │
│                                                                   │
│ Remote virtual network                                           │
│ ─────────────────────────────────────────────────────────       │
│ Peering link name:        [shared-to-prod]                      │
│ Subscription:             [Visual Studio Enterprise ▼]          │
│ Virtual network:          [shared-vnet ▼]                       │
│                                                                   │
│ Traffic to remote VNet:   ● Allow                               │
│ Traffic forwarded from                                           │
│   remote VNet:            ● Allow                               │
│ Virtual network gateway                                          │
│   or Route Server:        ● Use this VNet's gateway             │
│                            ○ Use the remote VNet's gateway      │
│                                                                   │
│ [Add]                                                            │
│                                                                   │
│ Status: Connected (both sides created automatically)            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create peering from prod to shared
  az network vnet peering create \
    --name prod-to-shared \
    --resource-group rg-prod \
    --vnet-name prod-vnet \
    --remote-vnet /subscriptions/xxx/resourceGroups/rg-shared/providers/Microsoft.Network/virtualNetworks/shared-vnet \
    --allow-vnet-access \
    --allow-forwarded-traffic \
    --use-remote-gateways  # If shared has VPN GW

  # Create reverse peering
  az network vnet peering create \
    --name shared-to-prod \
    --resource-group rg-shared \
    --vnet-name shared-vnet \
    --remote-vnet /subscriptions/xxx/resourceGroups/rg-prod/providers/Microsoft.Network/virtualNetworks/prod-vnet \
    --allow-vnet-access \
    --allow-forwarded-traffic \
    --allow-gateway-transit  # If shared has VPN GW

Bicep:
  resource peering 'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2023-05-01' = {
    parent: prodVnet
    name: 'prod-to-shared'
    properties: {
      remoteVirtualNetwork: {
        id: sharedVnet.id
      }
      allowVirtualNetworkAccess: true
      allowForwardedTraffic: true
      useRemoteGateways: false
    }
  }
```

### Gateway Transit Pattern

```
┌─────────────────────────────────────────────────────────────────────┐
│              GATEWAY TRANSIT (Hub-and-Spoke via Peering)              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Without Gateway Transit:                                             │
│ Each spoke needs its own VPN Gateway ($150+/month each!)           │
│                                                                       │
│ With Gateway Transit:                                                │
│ Hub has VPN Gateway, spokes share it via peering!                   │
│                                                                       │
│ ┌──────────┐                                                        │
│ │ On-Prem  │──VPN──┐                                               │
│ └──────────┘       │                                                │
│                     ▼                                                │
│              ┌──────────────┐                                       │
│              │ Hub VNet      │                                       │
│              │ ┌──────────┐ │                                       │
│              │ │ VPN GW   │ │ ← "Allow gateway transit"           │
│              │ └──────────┘ │                                       │
│              └──────┬───────┘                                       │
│                 Peering                                               │
│           ┌─────────┼─────────┐                                     │
│           ▼         ▼         ▼                                     │
│    ┌──────────┐ ┌──────────┐ ┌──────────┐                          │
│    │ Spoke 1  │ │ Spoke 2  │ │ Spoke 3  │                          │
│    │ prod-vnet│ │ dev-vnet │ │ mgmt-vnet│                          │
│    │ Uses     │ │ Uses     │ │ Uses     │                          │
│    │ remote GW│ │ remote GW│ │ remote GW│                          │
│    └──────────┘ └──────────┘ └──────────┘                          │
│                                                                       │
│ Result: All spokes can reach on-premises through hub's VPN GW!     │
│         On-prem can reach all spoke VNets                           │
│         Hub pays for one VPN GW instead of one per spoke            │
│                                                                       │
│ ⚠️ Spokes still can't talk to each other via peering!               │
│    Spoke-to-spoke requires Azure Firewall/NVA in hub + UDRs       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: VPN Gateway

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VPN GATEWAY                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Encrypted tunnel between your VNet and on-premises or         │
│       another VNet (cross-region VNet-to-VNet over internet)        │
│                                                                       │
│ Types:                                                               │
│ ├── Site-to-Site (S2S): VNet ↔ On-premises (always on)            │
│ ├── Point-to-Site (P2S): VNet ↔ Individual client (remote work)  │
│ └── VNet-to-VNet: VNet ↔ VNet (via internet, encrypted)          │
│                                                                       │
│ Key facts:                                                           │
│ ├── Requires GatewaySubnet (/27 or larger recommended)            │
│ ├── Gets a public IP (or 2 for active-active)                     │
│ ├── Takes 30-45 minutes to deploy!                                │
│ ├── IPSec/IKE encryption                                          │
│ ├── BGP support for dynamic routing                               │
│ ├── Active-Active for HA                                          │
│ └── Zone-redundant SKUs available                                 │
│                                                                       │
│ SKUs (pricing varies by region):                                     │
│ ┌────────────────┬──────────┬──────────────┬──────────────┐       │
│ │ SKU            │ Tunnels  │ Bandwidth    │ ~Price/month │       │
│ ├────────────────┼──────────┼──────────────┼──────────────┤       │
│ │ VpnGw1         │ 30 S2S   │ 650 Mbps     │ ~$140        │       │
│ │ VpnGw2         │ 30 S2S   │ 1 Gbps       │ ~$260        │       │
│ │ VpnGw3         │ 30 S2S   │ 1.25 Gbps    │ ~$500        │       │
│ │ VpnGw4         │ 100 S2S  │ 5 Gbps       │ ~$940        │       │
│ │ VpnGw5         │ 100 S2S  │ 10 Gbps      │ ~$1,540      │       │
│ │ VpnGw1AZ       │ 30 S2S   │ 650 Mbps     │ ~$185        │       │
│ │ (zone-redund)  │          │              │              │       │
│ └────────────────┴──────────┴──────────────┴──────────────┘       │
│                                                                       │
│ ⚠️ VPN Gateway is a significant monthly cost! Plan carefully.       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Site-to-Site VPN

```
Console → Create a resource → VPN Gateway

Step 1: Create Gateway Subnet first
  Virtual networks → prod-vnet → Subnets → + Gateway subnet
  ┌─────────────────────────────────────────────────────────────┐
  │ Subnet name:    GatewaySubnet (auto-set, cannot change)    │
  │ Address range:  10.0.255.0/27 (must be named GatewaySubnet)│
  │ [Save]                                                      │
  └─────────────────────────────────────────────────────────────┘

Step 2: Create VPN Gateway
  ┌─────────────────────────────────────────────────────────────┐
  │            CREATE VPN GATEWAY                                │
  ├─────────────────────────────────────────────────────────────┤
  │ Subscription:     [Visual Studio Enterprise ▼]             │
  │ Name:             [prod-vpn-gw]                            │
  │ Region:           [Central India ▼]                        │
  │ Gateway type:     ● VPN  ○ ExpressRoute                   │
  │ SKU:              [VpnGw1 ▼]                               │
  │ Generation:       [Generation2 ▼]                          │
  │ Virtual network:  [prod-vnet ▼]                            │
  │ Gateway subnet:   10.0.255.0/27 (auto-detected)           │
  │ Public IP:        [Create new: prod-vpn-pip]               │
  │                                                             │
  │ Active-active:    ○ Disabled ● Enabled                    │
  │   Second public IP: [prod-vpn-pip-2]                       │
  │                                                             │
  │ BGP:              ● Enabled                                │
  │   ASN:            [65515] (default)                        │
  │                                                             │
  │ [Review + create]                                          │
  │                                                             │
  │ ⚠️ Deployment takes 30-45 minutes!                         │
  └─────────────────────────────────────────────────────────────┘

Step 3: Create Local Network Gateway (represents on-premises)
  ┌─────────────────────────────────────────────────────────────┐
  │         LOCAL NETWORK GATEWAY                                │
  ├─────────────────────────────────────────────────────────────┤
  │ Name:          [office-local-gw]                            │
  │ IP address:    [203.0.113.1] (your on-prem router IP)      │
  │ Address space: [192.168.0.0/16] (your on-prem network)    │
  │                                                             │
  │ BGP:           ● Yes                                       │
  │   Peer ASN:    [65001]                                     │
  │   BGP peer IP: [192.168.1.1]                               │
  │                                                             │
  │ [Create]                                                    │
  └─────────────────────────────────────────────────────────────┘

Step 4: Create Connection
  ┌─────────────────────────────────────────────────────────────┐
  │              CREATE CONNECTION                               │
  ├─────────────────────────────────────────────────────────────┤
  │ Name:              [prod-to-office]                        │
  │ Connection type:   [Site-to-site (IPsec) ▼]              │
  │ VPN Gateway:       [prod-vpn-gw ▼]                        │
  │ Local network GW:  [office-local-gw ▼]                    │
  │ Shared key (PSK):  [your-strong-key-here]                 │
  │ IKE Protocol:      [IKEv2 ▼]                              │
  │ BGP:               ● Enable                                │
  │ Connection mode:   [Default ▼]                             │
  │                                                             │
  │ [Create]                                                    │
  └─────────────────────────────────────────────────────────────┘

CLI:
  # Create GatewaySubnet
  az network vnet subnet create \
    --name GatewaySubnet \
    --resource-group rg-prod \
    --vnet-name prod-vnet \
    --address-prefixes 10.0.255.0/27

  # Create VPN Gateway (takes 30-45 min)
  az network vnet-gateway create \
    --name prod-vpn-gw \
    --resource-group rg-prod \
    --vnet prod-vnet \
    --gateway-type Vpn \
    --sku VpnGw1 \
    --generation Generation2 \
    --vpn-type RouteBased \
    --public-ip-addresses prod-vpn-pip \
    --asn 65515 \
    --no-wait

  # Create Local Network Gateway
  az network local-gateway create \
    --name office-local-gw \
    --resource-group rg-prod \
    --gateway-ip-address 203.0.113.1 \
    --local-address-prefixes 192.168.0.0/16

  # Create Connection
  az network vpn-connection create \
    --name prod-to-office \
    --resource-group rg-prod \
    --vnet-gateway1 prod-vpn-gw \
    --local-gateway2 office-local-gw \
    --shared-key "your-strong-key-here" \
    --enable-bgp
```

### Point-to-Site VPN (Remote Access)

```
┌─────────────────────────────────────────────────────────────────────┐
│               POINT-TO-SITE VPN                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Individual client (laptop) connects to Azure VNet             │
│       Great for: Remote developers, work-from-home access           │
│                                                                       │
│ ┌─────────────┐       P2S VPN        ┌──────────────────┐          │
│ │ Developer   │ ◄═══════════════►  │ VPN Gateway       │          │
│ │ Laptop      │     (SSL/IKEv2)     │ prod-vnet         │          │
│ │ Gets 172.x  │                      │                    │          │
│ └─────────────┘                      └──────────────────┘          │
│                                                                       │
│ Authentication options:                                              │
│ ├── Azure certificate (self-signed root + client certs)            │
│ ├── RADIUS authentication                                          │
│ ├── Microsoft Entra ID (AAD) — best for organizations!            │
│ │   └── Users sign in with their corporate credentials             │
│ └── OpenVPN + custom auth                                           │
│                                                                       │
│ Tunnel types:                                                        │
│ ├── OpenVPN (TCP 443): Works everywhere, all platforms             │
│ ├── IKEv2: Native on Windows/macOS, best performance              │
│ └── SSTP: Windows-only, TCP 443, works behind strict firewalls    │
│                                                                       │
│ Configure:                                                           │
│ VPN Gateway → Point-to-site configuration                           │
│   Address pool: [172.16.0.0/24] (client IP range)                  │
│   Tunnel type: [OpenVPN + IKEv2 ▼]                                │
│   Authentication: [Azure Active Directory ▼]                       │
│   Tenant: https://login.microsoftonline.com/TENANT_ID/            │
│   Audience: Azure VPN App ID                                        │
│   Issuer: https://sts.windows.net/TENANT_ID/                      │
│                                                                       │
│ Client setup:                                                        │
│ Download VPN client from portal → Install → Connect                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: ExpressRoute

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EXPRESSROUTE                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Dedicated private connection from on-premises to Azure        │
│       (NOT over the internet — via connectivity provider)            │
│                                                                       │
│ ┌──────────────┐                              ┌──────────────────┐ │
│ │ On-Premises  │── Private link ─► Provider ─►│ Azure            │ │
│ │ Data Center  │  (Equinix, etc.)   edge site │ ExpressRoute     │ │
│ └──────────────┘                              │ Gateway          │ │
│                                                └──────────────────┘ │
│                                                                       │
│ Key facts:                                                           │
│ ├── Dedicated private connection (not over internet)               │
│ ├── Bandwidth: 50 Mbps to 100 Gbps                                │
│ ├── Consistent low latency                                         │
│ ├── 99.95% SLA                                                     │
│ ├── Connectivity models:                                           │
│ │   ├── CloudExchange colocation (at provider data center)        │
│ │   ├── Point-to-point Ethernet                                    │
│ │   ├── Any-to-any (IPVPN) network                               │
│ │   └── ExpressRoute Direct (100 Gbps dedicated ports)            │
│ ├── Two circuits always (redundancy required for SLA)             │
│ └── Takes days/weeks to provision                                  │
│                                                                       │
│ Peering types:                                                       │
│ ├── Azure Private Peering: Access VNets (private IPs)             │
│ │   └── Most common, like AWS Direct Connect Private VIF          │
│ ├── Microsoft Peering: Access Microsoft 365, Azure PaaS           │
│ │   └── Includes Azure Storage, SQL, etc. via public IPs          │
│ └── Azure Public Peering: Deprecated → use Microsoft Peering      │
│                                                                       │
│ ExpressRoute + VPN (recommended HA):                                 │
│ ├── ExpressRoute as primary (high bandwidth, low latency)         │
│ ├── S2S VPN as failover (over internet)                            │
│ └── Coexist on same VNet via ExpressRoute + VPN Gateways          │
│                                                                       │
│ ExpressRoute Global Reach:                                           │
│ ├── Connect on-premises sites THROUGH Azure backbone              │
│ ├── Office-A → ExpressRoute → Azure → ExpressRoute → Office-B   │
│ └── Useful for multinational companies                             │
│                                                                       │
│ Pricing (example - Central India):                                   │
│ ├── Circuit: Depends on bandwidth & provider                      │
│ │   50 Mbps Standard: ~$55/month                                  │
│ │   1 Gbps Standard: ~$435/month                                  │
│ │   10 Gbps Standard: ~$2,650/month                               │
│ ├── ExpressRoute Gateway: $140-2,300/month (varies by SKU)       │
│ └── Data plans: Metered (per GB) or Unlimited                     │
│                                                                       │
│ SKU tiers:                                                           │
│ ├── Standard: Connect to resources in same geopolitical region     │
│ └── Premium: Connect across all Azure regions globally             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating ExpressRoute

```
Console → Create a resource → ExpressRoute circuit

┌─────────────────────────────────────────────────────────────────┐
│           CREATE EXPRESSROUTE CIRCUIT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Circuit name:    [prod-expressroute]                             │
│ Resource group:  [rg-networking ▼]                               │
│ Region:          [Central India ▼]                               │
│                                                                   │
│ Provider:        [Equinix ▼]                                    │
│ Peering location:[Mumbai ▼]                                     │
│ Bandwidth:       [1 Gbps ▼]                                    │
│                                                                   │
│ SKU:             ● Standard  ○ Premium                          │
│ Billing model:   ● Metered   ○ Unlimited                       │
│ Allow classic:   ○ Yes  ● No                                   │
│                                                                   │
│ [Review + create]                                                │
│                                                                   │
│ After creation:                                                  │
│ 1. Get Service Key from portal                                  │
│ 2. Send Service Key to provider (Equinix/Megaport/etc.)        │
│ 3. Provider provisions the circuit                              │
│ 4. Configure peering (Private and/or Microsoft)                │
│ 5. Create ExpressRoute Gateway in VNet                          │
│ 6. Create Connection (Gateway ↔ Circuit)                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create ExpressRoute circuit
  az network express-route create \
    --name prod-expressroute \
    --resource-group rg-networking \
    --location centralindia \
    --provider Equinix \
    --peering-location Mumbai \
    --bandwidth 1000 \
    --sku-tier Standard \
    --sku-family MeteredData

  # Create ExpressRoute Gateway
  az network vnet-gateway create \
    --name prod-er-gw \
    --resource-group rg-networking \
    --vnet prod-vnet \
    --gateway-type ExpressRoute \
    --sku Standard

  # Create Connection
  az network vpn-connection create \
    --name prod-er-connection \
    --resource-group rg-networking \
    --vnet-gateway1 prod-er-gw \
    --express-route-circuit2 prod-expressroute
```

---

## Part 4: Azure Virtual WAN

```
┌─────────────────────────────────────────────────────────────────────┐
│                  AZURE VIRTUAL WAN                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A managed hub-and-spoke networking service that provides       │
│       transit connectivity between VNets, VPN, ExpressRoute,        │
│       and branches — all through a central hub.                     │
│                                                                       │
│ Think of it as: Managed Transit Gateway + VPN Gateway +              │
│                 ExpressRoute Gateway + Firewall in one hub          │
│                                                                       │
│ Without Virtual WAN (DIY hub-and-spoke):                             │
│ ├── You create hub VNet manually                                   │
│ ├── You manage peering to each spoke                               │
│ ├── You deploy VPN GW, ER GW, Firewall manually                  │
│ ├── You configure UDRs for spoke-to-spoke                         │
│ └── You manage routing tables                                      │
│                                                                       │
│ With Virtual WAN:                                                    │
│ ├── Azure creates and manages the hub                              │
│ ├── Auto-configures spoke peering                                  │
│ ├── Built-in VPN Gateway, ExpressRoute Gateway                    │
│ ├── Integrated Azure Firewall option                               │
│ ├── Auto-routing between all connections                           │
│ └── Multi-hub for multi-region                                     │
│                                                                       │
│ Architecture:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                    Virtual WAN                                 │   │
│ │                                                                │   │
│ │  ┌────────────────┐         ┌────────────────┐               │   │
│ │  │ Hub: India      │ ◄═══► │ Hub: US East    │  Multi-hub   │   │
│ │  │ (managed VNet)  │         │ (managed VNet)  │               │   │
│ │  │ ┌──────┐       │         │ ┌──────┐       │               │   │
│ │  │ │VPN GW│       │         │ │ER GW │       │               │   │
│ │  │ └──────┘       │         │ └──────┘       │               │   │
│ │  │ ┌──────┐       │         │ ┌──────┐       │               │   │
│ │  │ │FW    │       │         │ │FW    │       │               │   │
│ │  │ └──────┘       │         │ └──────┘       │               │   │
│ │  └──┬──────┬──────┘         └──┬──────┬──────┘               │   │
│ │     │      │                    │      │                       │   │
│ │  ┌──┴──┐┌──┴──┐             ┌──┴──┐┌──┴──┐                  │   │
│ │  │prod ││dev  │             │prod ││dev  │                  │   │
│ │  │VNet ││VNet │             │VNet ││VNet │                  │   │
│ │  │India││India│             │US   ││US   │                  │   │
│ │  └─────┘└─────┘             └─────┘└─────┘                  │   │
│ │                                                                │   │
│ │  + VPN branches (offices)                                      │   │
│ │  + ExpressRoute circuits                                       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ SKUs:                                                                │
│ ├── Basic: VPN only (S2S), limited features                       │
│ └── Standard: Full features (VPN, ER, VNet, any-to-any)          │
│                                                                       │
│ Pricing: Hub deployment + VPN/ER GW + data processed               │
│ Hub: ~$0.25/hr ≈ $180/month (just the hub)                         │
│                                                                       │
│ When to use Virtual WAN vs DIY hub-and-spoke:                       │
│ ├── Virtual WAN: 10+ spokes, multi-region, branch offices        │
│ │   Simpler management, auto-routing, higher cost                 │
│ └── DIY hub: Few spokes, single region, need full control         │
│     More config, cheaper for small deployments                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Azure Private Link & Private Endpoints

```
┌─────────────────────────────────────────────────────────────────────┐
│            AZURE PRIVATE LINK & PRIVATE ENDPOINTS                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Access Azure PaaS services (Storage, SQL, Key Vault, etc.)    │
│       over a private endpoint in your VNet.                         │
│                                                                       │
│ Without Private Endpoint:                                            │
│ [VM] → Internet → mydb.database.windows.net (public!)              │
│ ⚠️ Traffic goes over Microsoft backbone but via public endpoint    │
│                                                                       │
│ With Private Endpoint:                                               │
│ [VM] → Private Endpoint (10.0.3.5) → SQL Database (private!)      │
│ ✅ Traffic stays in your VNet, no public exposure                   │
│                                                                       │
│ How it works:                                                        │
│ ├── Creates a NIC in your subnet with a private IP                 │
│ ├── Maps to a specific Azure PaaS resource                         │
│ ├── DNS: mydb.database.windows.net → 10.0.3.5 (private IP)       │
│ │   (via privatelink DNS zone)                                      │
│ ├── NSG rules apply to the private endpoint                        │
│ ├── Works across VNet peering and VPN/ExpressRoute                │
│ └── Can disable public access on the PaaS resource!                │
│                                                                       │
│ Supported services (100+):                                           │
│ ├── Azure SQL Database          ├── Azure Key Vault              │
│ ├── Azure Storage (blob, file)  ├── Azure Container Registry     │
│ ├── Azure Cosmos DB              ├── Azure App Service            │
│ ├── Azure Event Hubs             ├── Azure Service Bus            │
│ ├── Azure Cache for Redis        ├── Azure Monitor                │
│ └── ... many more                                                    │
│                                                                       │
│ Pricing:                                                             │
│ ├── Private Endpoint: $0.01/hr ≈ $7.30/month per endpoint         │
│ ├── Data processed: $0.01/GB inbound, $0.01/GB outbound           │
│ └── Each PaaS service needs its own private endpoint               │
│                                                                       │
│ Comparison across clouds:                                            │
│ ┌──────────────────────┬────────────────┬─────────┬─────────┐     │
│ │ Feature              │ AWS            │ GCP     │ Azure   │     │
│ │ Private PaaS access  │ Interface      │ PSC     │ Private │     │
│ │                      │ Endpoint       │         │ Endpoint│     │
│ │ Free option          │ S3/DDB Gateway │ PGA     │ Service │     │
│ │                      │ Endpoint       │ (free)  │ Endpoint│     │
│ │ Publish own service  │ PrivateLink    │ PSC     │ Private │     │
│ │                      │                │         │ Link Svc│     │
│ └──────────────────────┴────────────────┴─────────┴─────────┘     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Private Endpoint

```
Console → Create a resource → Private Endpoint

┌─────────────────────────────────────────────────────────────────┐
│           CREATE PRIVATE ENDPOINT                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Basics:                                                          │
│   Subscription:  [Visual Studio Enterprise ▼]                  │
│   Resource group:[rg-prod ▼]                                    │
│   Name:          [pe-prod-sql]                                  │
│   NIC name:      [pe-prod-sql-nic] (auto-generated)            │
│   Region:        [Central India ▼]                              │
│                                                                   │
│ Resource:                                                        │
│   Connection method: ○ Connect to resource in my directory     │
│                      ○ Connect by resource ID                  │
│   Resource type:    [Microsoft.Sql/servers ▼]                  │
│   Resource:         [prod-sql-server ▼]                        │
│   Target sub-resource: [sqlServer ▼]                           │
│                                                                   │
│ Virtual Network:                                                 │
│   Virtual network: [prod-vnet ▼]                                │
│   Subnet:          [data-subnet ▼]                              │
│   Private IP:      ○ Dynamically allocate ● Static             │
│   IP address:      [10.0.3.5]                                   │
│                                                                   │
│ DNS:                                                             │
│   Integrate with private DNS zone: ● Yes  ○ No                │
│   Private DNS zone:                                              │
│   [privatelink.database.windows.net ▼]                          │
│   └── Auto-creates DNS record:                                  │
│       prod-sql-server.database.windows.net                     │
│       → prod-sql-server.privatelink.database.windows.net       │
│       → 10.0.3.5                                                │
│                                                                   │
│ [Review + create]                                                │
│                                                                   │
│ After creation:                                                  │
│ ⚡ IMPORTANT: Disable public access on SQL Server!               │
│ SQL Server → Networking → Public access: Deny                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create Private Endpoint
  az network private-endpoint create \
    --name pe-prod-sql \
    --resource-group rg-prod \
    --vnet-name prod-vnet \
    --subnet data-subnet \
    --private-connection-resource-id /subscriptions/xxx/.../prod-sql-server \
    --group-id sqlServer \
    --connection-name pe-sql-connection

  # Create Private DNS Zone
  az network private-dns zone create \
    --name privatelink.database.windows.net \
    --resource-group rg-prod

  # Link DNS Zone to VNet
  az network private-dns link vnet create \
    --name prod-vnet-link \
    --resource-group rg-prod \
    --zone-name privatelink.database.windows.net \
    --virtual-network prod-vnet \
    --registration-enabled false

  # Create DNS record
  az network private-endpoint dns-zone-group create \
    --name sql-zone-group \
    --resource-group rg-prod \
    --endpoint-name pe-prod-sql \
    --private-dns-zone privatelink.database.windows.net \
    --zone-name sqlServer

Bicep:
  resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
    name: 'pe-prod-sql'
    location: location
    properties: {
      subnet: {
        id: dataSubnet.id
      }
      privateLinkServiceConnections: [
        {
          name: 'pe-sql-connection'
          properties: {
            privateLinkServiceId: sqlServer.id
            groupIds: ['sqlServer']
          }
        }
      ]
    }
  }

  resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
    name: 'privatelink.database.windows.net'
    location: 'global'
  }

  resource vnetLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
    parent: privateDnsZone
    name: 'prod-vnet-link'
    location: 'global'
    properties: {
      virtualNetwork: {
        id: prodVnet.id
      }
      registrationEnabled: false
    }
  }
```

### Private Link Service (Exposing Your Service)

```
┌─────────────────────────────────────────────────────────────────────┐
│          PRIVATE LINK SERVICE (Publish Your Own Service)              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Expose YOUR service to consumers in other VNets/subscriptions │
│       via Private Endpoint (like AWS PrivateLink)                    │
│                                                                       │
│ Provider VNet                          Consumer VNet                 │
│ ┌────────────────────────┐            ┌──────────────────────┐     │
│ │ [Your Service]         │            │ [Consumer App]       │     │
│ │      ↕                  │            │      ↕                │     │
│ │ [Standard LB]          │◄────────── │ [Private Endpoint]  │     │
│ │      ↕                  │  Private   │   10.1.0.10          │     │
│ │ [Private Link Service] │  Link      │                      │     │
│ └────────────────────────┘            └──────────────────────┘     │
│                                                                       │
│ Steps:                                                               │
│ 1. Deploy your service behind Standard Load Balancer               │
│ 2. Create Private Link Service → link to LB                        │
│ 3. Consumer creates Private Endpoint pointing to your PLS          │
│ 4. You approve the connection (or auto-approve)                    │
│                                                                       │
│ ⚡ CIDRs CAN overlap (no peering needed!)                            │
│ ⚡ Works cross-subscription and cross-tenant                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Azure Bastion

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AZURE BASTION                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed jump box service — SSH/RDP to VMs directly from       │
│       Azure Portal over TLS, no public IP on VMs needed.            │
│                                                                       │
│ Without Bastion:                                                     │
│ [You] → Internet → VM Public IP:3389 (RDP open to internet! ⚠️)    │
│                                                                       │
│ With Bastion:                                                        │
│ [You] → Azure Portal (HTTPS/443) → Bastion → VM (private IP)      │
│ ✅ No public IP on VMs, no NSG rules for RDP/SSH from internet     │
│                                                                       │
│ Requires: AzureBastionSubnet (/26 or larger)                       │
│                                                                       │
│ SKUs:                                                                │
│ ├── Developer: Free preview, limited features, single VM           │
│ ├── Basic: SSH/RDP from portal, 2 instances                       │
│ ├── Standard: + native client, file transfer, shareable links     │
│ └── Premium: + session recording, private-only deployment          │
│                                                                       │
│ Pricing:                                                             │
│ ├── Basic: ~$0.19/hr ≈ ~$138/month                               │
│ ├── Standard: ~$0.29/hr ≈ ~$211/month                             │
│ └── + Data transfer charges                                        │
│                                                                       │
│ ⚠️ Bastion is expensive for small teams. Alternatives:              │
│    ├── SSH via VPN (Point-to-Site VPN)                              │
│    ├── Azure Serial Console (free, limited)                        │
│    └── Just-in-time VM access (Microsoft Defender)                 │
│                                                                       │
│ Create:                                                              │
│ 1. Create AzureBastionSubnet in your VNet (/26 minimum)           │
│ 2. Create Bastion resource → link to VNet                          │
│ 3. Connect: VM → Connect → Bastion → Enter username/password      │
│                                                                       │
│ CLI:                                                                 │
│ az network vnet subnet create \                                      │
│   --name AzureBastionSubnet \                                        │
│   --resource-group rg-prod \                                         │
│   --vnet-name prod-vnet \                                            │
│   --address-prefixes 10.0.254.0/26                                   │
│                                                                       │
│ az network bastion create \                                          │
│   --name prod-bastion \                                              │
│   --resource-group rg-prod \                                         │
│   --vnet-name prod-vnet \                                            │
│   --public-ip-address prod-bastion-pip \                             │
│   --sku Standard                                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: User Defined Routes (UDR)

```
┌─────────────────────────────────────────────────────────────────────┐
│               USER DEFINED ROUTES (UDR)                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Custom routes that override Azure's default system routes.     │
│                                                                       │
│ Azure default routing:                                               │
│ ├── VNet address space → direct (within VNet)                      │
│ ├── 0.0.0.0/0 → Internet                                          │
│ ├── 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 → None (drop)    │
│ └── Peered VNet ranges → VNet peering                              │
│                                                                       │
│ Why override?                                                        │
│ ├── Force traffic through a firewall (NVA or Azure Firewall)      │
│ ├── Route internet traffic through a central egress point         │
│ ├── Block certain routes (next hop = None)                        │
│ └── Route to on-premises via VPN/ExpressRoute                     │
│                                                                       │
│ Common use case: Force all traffic through Azure Firewall          │
│                                                                       │
│ [Spoke VM] → UDR (0.0.0.0/0 → Azure FW IP) → Azure Firewall    │
│              → Internet (after inspection)                           │
│                                                                       │
│ Next hop types:                                                      │
│ ├── Virtual network gateway (VPN/ER)                               │
│ ├── Virtual network (within VNet)                                  │
│ ├── Internet                                                       │
│ ├── Virtual appliance (NVA IP address)                             │
│ ├── None (drop/blackhole)                                          │
│ └── Virtual network service endpoint                               │
│                                                                       │
│ Create route table:                                                  │
│ az network route-table create \                                      │
│   --name spoke-to-firewall-rt \                                      │
│   --resource-group rg-prod \                                         │
│   --disable-bgp-route-propagation true                               │
│                                                                       │
│ Add route:                                                           │
│ az network route-table route create \                                │
│   --name force-firewall \                                            │
│   --resource-group rg-prod \                                         │
│   --route-table-name spoke-to-firewall-rt \                          │
│   --address-prefix 0.0.0.0/0 \                                      │
│   --next-hop-type VirtualAppliance \                                 │
│   --next-hop-ip-address 10.3.1.4  # Azure Firewall private IP      │
│                                                                       │
│ Associate with subnet:                                               │
│ az network vnet subnet update \                                      │
│   --name app-subnet \                                                │
│   --resource-group rg-prod \                                         │
│   --vnet-name prod-vnet \                                            │
│   --route-table spoke-to-firewall-rt                                 │
│                                                                       │
│ Bicep:                                                               │
│ resource routeTable 'Microsoft.Network/routeTables@2023-05-01' = { │
│   name: 'spoke-to-firewall-rt'                                      │
│   location: location                                                  │
│   properties: {                                                       │
│     disableBgpRoutePropagation: true                                 │
│     routes: [                                                         │
│       {                                                               │
│         name: 'force-firewall'                                       │
│         properties: {                                                 │
│           addressPrefix: '0.0.0.0/0'                                 │
│           nextHopType: 'VirtualAppliance'                            │
│           nextHopIpAddress: '10.3.1.4'                               │
│         }                                                             │
│       }                                                               │
│     ]                                                                 │
│   }                                                                   │
│ }                                                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Network Watcher & NSG Flow Logs

```
┌─────────────────────────────────────────────────────────────────────┐
│              NETWORK WATCHER & NSG FLOW LOGS                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Network Watcher: Suite of network diagnostics tools (per region)    │
│                                                                       │
│ Tools:                                                               │
│ ├── IP flow verify: Can VM A reach VM B on port 443?              │
│ │   → Tests NSGs and says which rule allows/denies                │
│ │                                                                    │
│ ├── Next hop: Where does traffic go from VM A to 10.1.0.5?       │
│ │   → Shows the next hop (internet, VPN GW, NVA, etc.)           │
│ │                                                                    │
│ ├── Connection troubleshoot: Full path analysis A → B             │
│ │   → Shows latency, hops, and where it fails                    │
│ │                                                                    │
│ ├── Packet capture: Capture packets on a VM's NIC                 │
│ │   → Save to file for Wireshark analysis                        │
│ │                                                                    │
│ ├── NSG diagnostics: What NSG rules apply to a VM?               │
│ │   → Shows effective security rules                              │
│ │                                                                    │
│ ├── Topology: Visual map of your network resources                │
│ │                                                                    │
│ └── NSG Flow Logs: Network traffic logging                        │
│                                                                       │
│ NSG Flow Logs:                                                       │
│ ├── Captures traffic allowed/denied by NSGs                       │
│ ├── Stored in Storage Account (JSON format)                       │
│ ├── Analyze with: Traffic Analytics (built-in!)                   │
│ ├── Two versions: V1 (basic) and V2 (+ byte/packet counts)      │
│ └── Enable per NSG                                                 │
│                                                                       │
│ Enable NSG Flow Logs:                                                │
│ Console → Network Watcher → NSG flow logs → Create                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ NSG:                [app-subnet-nsg ▼]                       │   │
│ │ Storage account:    [stflowlogsprod ▼]                       │   │
│ │ Retention (days):   [30]                                      │   │
│ │ Flow logs version:  [Version 2 ▼]                             │   │
│ │                                                                │   │
│ │ Traffic Analytics:  ● Enabled                                 │   │
│ │ Processing interval:[Every 10 min ▼] (or every 1 hr)        │   │
│ │ Log Analytics workspace: [prod-la-workspace ▼]               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Traffic Analytics: Built-in dashboard over flow logs                 │
│ ├── Top talkers (which VMs generate most traffic)                  │
│ ├── Blocked flows (NSG denies)                                     │
│ ├── Traffic patterns (by protocol, port, destination)             │
│ ├── Geo-visualization (traffic by country)                        │
│ └── ⚡ One of Azure's best networking features!                     │
│                                                                       │
│ CLI:                                                                 │
│ az network watcher flow-log create \                                 │
│   --name prod-nsg-flowlog \                                          │
│   --nsg app-subnet-nsg \                                             │
│   --resource-group rg-prod \                                         │
│   --storage-account stflowlogsprod \                                 │
│   --enabled true \                                                    │
│   --format JSON \                                                     │
│   --log-version 2 \                                                   │
│   --retention 30 \                                                    │
│   --traffic-analytics true \                                         │
│   --workspace prod-la-workspace                                      │
│                                                                       │
│ Pricing:                                                             │
│ ├── NSG Flow Logs: $0.50/GB collected                              │
│ ├── Storage: Standard blob rates                                   │
│ ├── Traffic Analytics: $2.00/GB analyzed (10 min interval)        │
│ │                       $0.50/GB analyzed (60 min interval)        │
│ └── Can add up! Use 60-min interval for cost savings               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

# Query flow logs in Log Analytics (KQL)
AzureNetworkAnalytics_CL
| where FlowType_s == "MaliciousFlow"
| project TimeGenerated, SrcIP_s, DestIP_s, DestPort_d, FlowStatus_s
| order by TimeGenerated desc
| take 50
```

---

## Part 9: Real-World Architecture Patterns

### Startup: Simple VNet

```
┌─────────────────────────────────────────────────────────────────┐
│                    STARTUP PATTERN                                │
│                                                                   │
│  VNet: prod-vnet (10.0.0.0/16)                                 │
│  ├── web-subnet: App Service / VM with NSG                    │
│  ├── app-subnet: API VMs with NSG                              │
│  ├── data-subnet: SQL + Private Endpoints                     │
│  ├── Private Endpoints for SQL, Storage, Key Vault             │
│  ├── Service Endpoints for quick wins (if not PE)             │
│  ├── NSG Flow Logs + Traffic Analytics                        │
│  └── No VPN needed (use Azure Bastion Developer for SSH)      │
│                                                                   │
│  Monthly cost: ~$50 (Private Endpoints + flow logs)             │
│  No peering, no VPN Gateway, no ExpressRoute                   │
└─────────────────────────────────────────────────────────────────┘
```

### Mid-Size: Hub-and-Spoke with VPN

```
┌─────────────────────────────────────────────────────────────────┐
│                  MID-SIZE PATTERN                                 │
│                                                                   │
│  Hub VNet (10.3.0.0/16):                                       │
│  ├── VPN Gateway (VpnGw1) → Office                           │
│  ├── Azure Bastion (Standard)                                 │
│  ├── Azure Firewall (Standard) — optional                     │
│  ├── GatewaySubnet, AzureBastionSubnet, AzureFirewallSubnet  │
│  └── DNS Private Resolver                                      │
│                                                                   │
│  Spoke VNets (peered with Gateway Transit):                    │
│  ├── prod-vnet (10.0.0.0/16): Uses remote GW                 │
│  ├── dev-vnet (10.1.0.0/16): Uses remote GW                  │
│  └── Each has: Private Endpoints, NSGs, UDRs                  │
│                                                                   │
│  Private DNS Zones (linked to all VNets):                      │
│  ├── privatelink.database.windows.net                         │
│  ├── privatelink.blob.core.windows.net                        │
│  └── privatelink.vaultcore.azure.net                          │
│                                                                   │
│  Monthly cost: ~$400-800 (VPN GW + Bastion + PEs)             │
└─────────────────────────────────────────────────────────────────┘
```

### Enterprise: Virtual WAN

```
┌─────────────────────────────────────────────────────────────────┐
│                 ENTERPRISE PATTERN                                │
│                                                                   │
│  Azure Virtual WAN (Standard)                                   │
│  ├── Hub: Central India                                        │
│  │   ├── VPN Gateway → 5 branch offices                      │
│  │   ├── ExpressRoute → Data Center                          │
│  │   ├── Secured Hub (Azure Firewall Manager)                │
│  │   ├── Spoke: prod-vnet (10.0.0.0/16)                     │
│  │   ├── Spoke: staging-vnet (10.1.0.0/16)                  │
│  │   └── Spoke: shared-vnet (10.3.0.0/16)                   │
│  │                                                              │
│  ├── Hub: East US                                               │
│  │   ├── VPN Gateway → US branch offices                     │
│  │   ├── Secured Hub                                           │
│  │   ├── Spoke: us-prod-vnet (10.10.0.0/16)                 │
│  │   └── Spoke: us-dev-vnet (10.11.0.0/16)                  │
│  │                                                              │
│  ├── Hub-to-hub: Auto-connected (global transit)              │
│  │                                                              │
│  ├── Azure Firewall Manager: Central firewall policies        │
│  ├── Azure DDoS Protection Plan                               │
│  ├── Private DNS Zones (centralized)                          │
│  ├── Network Watcher + Traffic Analytics (all regions)        │
│  └── Microsoft Sentinel (SIEM integration with flow logs)     │
│                                                                   │
│  Monthly cost: $3,000-15,000+ networking                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Component | Purpose | Cost |
|-----------|---------|------|
| VNet Peering | Connect 2 VNets | $0.01/GB (same region) |
| Global VNet Peering | Cross-region VNets | $0.035-0.075/GB |
| VPN Gateway (VpnGw1) | Encrypted tunnel | ~$140/month |
| ExpressRoute | Dedicated private link | ~$55-2,650+/month |
| Virtual WAN Hub | Managed hub-and-spoke | ~$180/month + GW |
| Private Endpoint | Private PaaS access | ~$7.30/month + data |
| Private Link Service | Expose your service | ~$7.30/month |
| Azure Bastion | Managed SSH/RDP access | ~$138-211/month |
| UDR (Route Table) | Custom routing | FREE |
| NSG Flow Logs | Traffic logging | $0.50/GB |
| Traffic Analytics | Flow log analysis | $0.50-2.00/GB |
| Network Watcher | Diagnostics tools | Most tools FREE |

---

## What's Next?

In the next chapter, we'll cover NSG (Network Security Groups) deep dive — inbound/outbound rules, priorities, ASG (Application Security Groups), and NSG best practices.

→ Next: [Chapter 8: NSG Deep Dive & Application Security Groups](08-nsg.md)

---

*Last Updated: May 2026*
