# Chapter 7: VPC Advanced Networking (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: NAT Gateway](#part-1-nat-gateway)
- [Part 2: VPC Peering](#part-2-vpc-peering)
- [Part 3: Transit Gateway](#part-3-transit-gateway)
- [Part 4: VPC Endpoints](#part-4-vpc-endpoints)
- [Part 5: AWS PrivateLink (Exposing Your Services)](#part-5-aws-privatelink-exposing-your-services)
- [Part 6: Site-to-Site VPN](#part-6-site-to-site-vpn)
- [Part 7: Direct Connect (Overview)](#part-7-direct-connect-overview)

---

## Overview

### When Do I Need These Advanced Features?

> **Real-World Analogy:** Chapter 6 built your private office floor. This chapter connects floors together (VPC Peering), adds a central lobby hub connecting all floors (Transit Gateway), installs a private elevator to AWS services that bypasses the public street (VPC Endpoints), and builds a private tunnel to your company HQ (VPN/Direct Connect).

**Why does this matter?** Real companies never use a single isolated VPC. Production, staging, and dev need to communicate. On-premises data centers need secure connections. And sending traffic through the public internet to reach S3 costs money (NAT Gateway charges) when a free VPC Endpoint exists. This chapter can **save your company thousands of dollars per month**.

**Which connectivity option should I use?**

| Scenario | Solution |
|----------|----------|
| 2 VPCs need to talk | VPC Peering (free) |
| 5+ VPCs need to talk | Transit Gateway |
| Private subnet needs internet | NAT Gateway |
| Private subnet needs S3/DynamoDB | Gateway Endpoint (free!) |
| On-premises ↔ AWS (quick setup) | Site-to-Site VPN |
| On-premises ↔ AWS (high bandwidth) | Direct Connect |

Building on VPC fundamentals (Chapter 6), this chapter covers advanced networking components that enable private internet access, cross-VPC communication, hybrid connectivity, and private access to AWS services.

```
What you'll learn:
├── NAT Gateway (deep dive)
├── NAT Instance (legacy, but good to know)
├── VPC Peering
├── Transit Gateway
├── VPC Endpoints (Gateway & Interface)
├── AWS PrivateLink
├── VPN Connections (Site-to-Site)
├── Direct Connect (overview)
├── VPC Flow Logs
├── Traffic Mirroring
└── Real-world architecture patterns
```

---

## Part 1: NAT Gateway

### What and Why

```
┌─────────────────────────────────────────────────────────────────────┐
│                     NAT GATEWAY                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: Private subnet VMs need internet access for:                │
│ ├── OS package updates (apt update, yum install)                    │
│ ├── Downloading dependencies (npm install, pip install)             │
│ ├── Calling external APIs (payment gateways, email services)        │
│ └── But should NOT be reachable FROM the internet                   │
│                                                                       │
│ Solution: NAT Gateway — translates private IPs to public IP         │
│                                                                       │
│ Traffic flow:                                                        │
│                                                                       │
│ [Private VM 10.0.11.5]                                               │
│      │                                                                │
│      ▼ (route: 0.0.0.0/0 → nat-xxxxx)                              │
│ [NAT Gateway in PUBLIC subnet]                                       │
│      │ (has Elastic IP: 54.210.xxx.xxx)                              │
│      ▼                                                                │
│ [Internet Gateway]                                                    │
│      │                                                                │
│      ▼                                                                │
│ [Internet] ← sees traffic from 54.210.xxx.xxx                       │
│                                                                       │
│ ⚠️ NAT Gateway lives in PUBLIC subnet (needs IGW route)             │
│    Private subnet routes 0.0.0.0/0 → NAT Gateway                   │
│    NAT Gateway routes to IGW via public subnet's route table        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### NAT Gateway Properties

```
┌─────────────────────────────────────────────────────────────────────┐
│                NAT GATEWAY DETAILS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Type:                                                                │
│ ├── Public NAT Gateway (most common — internet access)              │
│ └── Private NAT Gateway (for private-to-private translation)        │
│                                                                       │
│ Key facts:                                                           │
│ ├── Managed by AWS (no patching, no OS)                             │
│ ├── Highly available within an AZ (not across AZs!)                │
│ ├── Requires Elastic IP (public NAT)                                │
│ ├── Bandwidth: Starts at 5 Gbps, scales to 100 Gbps               │
│ ├── Supports 55,000 simultaneous connections per IP                │
│ ├── Can add up to 8 EIPs for more connections                     │
│ ├── Cannot be used with Security Groups (use NACLs on subnet)     │
│ ├── Processes: TCP, UDP, ICMP                                      │
│ └── IPv6: Use Egress-only Internet Gateway instead                 │
│                                                                       │
│ Pricing:                                                             │
│ ├── Per hour: $0.045/hr ≈ $32.40/month per NAT GW                 │
│ ├── Per GB processed: $0.045/GB                                     │
│ ├── Elastic IP: $0.005/hr ≈ $3.65/month                           │
│ └── Total for 100GB/month: ~$40.55/month                           │
│                                                                       │
│ ⚠️ NAT Gateway is one of the biggest "surprise cost" items!         │
│    $32/month just sitting there + data processing charges           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a NAT Gateway

```
Console → VPC → NAT Gateways → Create NAT gateway

┌─────────────────────────────────────────────────────────────────┐
│                  CREATE NAT GATEWAY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:              [prod-nat-a]                                  │
│ Subnet:            [prod-public-a ▼] ← MUST be public subnet   │
│ Connectivity type: ● Public  ○ Private                          │
│ Elastic IP:        [Allocate Elastic IP] or [eipalloc-xxx ▼]   │
│                                                                   │
│ Tags:                                                            │
│   Environment: production                                        │
│   Name: prod-nat-a                                               │
│                                                                   │
│ [Create NAT gateway]                                             │
│                                                                   │
│ After creation:                                                  │
│ Add route in PRIVATE route table:                                │
│   Destination: 0.0.0.0/0                                        │
│   Target: nat-xxxxxxxx                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Allocate EIP
  aws ec2 allocate-address --domain vpc

  # Create NAT Gateway
  aws ec2 create-nat-gateway \
    --subnet-id subnet-pub-a \
    --allocation-id eipalloc-xxxxx \
    --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-nat-a}]'

  # Add route to private route table
  aws ec2 create-route \
    --route-table-id rtb-private \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id nat-xxxxx
```

### HA Pattern for NAT Gateway

```
┌─────────────────────────────────────────────────────────────────────┐
│          NAT GATEWAY HIGH AVAILABILITY                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️ NAT Gateway is NOT cross-AZ! If AZ goes down, NAT goes down    │
│                                                                       │
│ DEV/STAGING: One NAT GW (save cost, accept AZ failure risk)        │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ Public-a               │ Private-a        │ Private-b      │    │
│  │ ┌──────────┐          │ ┌──────────┐     │ ┌──────────┐  │    │
│  │ │ NAT GW   │ ◄────── │ │ App VM   │     │ │ App VM   │  │    │
│  │ │ (single) │          │ └──────────┘     │ └──────────┘  │    │
│  │ └──────────┘          │       ↑           │      ↑        │    │
│  │                        │       └───────────┴──────┘        │    │
│  │                        │     Both use same NAT GW          │    │
│  └────────────────────────────────────────────────────────────┘    │
│  Cost: ~$36/month                                                   │
│                                                                       │
│ PRODUCTION: One NAT GW per AZ (full HA)                             │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ AZ-a                        │ AZ-b                         │    │
│  │ Public-a    Private-a       │ Public-b    Private-b        │    │
│  │ ┌────────┐ ┌──────────┐    │ ┌────────┐ ┌──────────┐    │    │
│  │ │ NAT-a  │◄│ App VM   │    │ │ NAT-b  │◄│ App VM   │    │    │
│  │ └────────┘ └──────────┘    │ └────────┘ └──────────┘    │    │
│  │     ↕                       │     ↕                       │    │
│  │   IGW                       │   IGW                       │    │
│  └────────────────────────────────────────────────────────────┘    │
│  Cost: ~$72/month (2× NAT GW)                                      │
│                                                                       │
│  Each AZ has its own NAT GW + own private route table               │
│  If AZ-a fails, AZ-b still has internet access                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

Terraform (HA pattern):
  resource "aws_nat_gateway" "az_a" {
    allocation_id = aws_eip.nat_a.id
    subnet_id     = aws_subnet.public_a.id
  }

  resource "aws_nat_gateway" "az_b" {
    allocation_id = aws_eip.nat_b.id
    subnet_id     = aws_subnet.public_b.id
  }

  # Separate route tables per AZ
  resource "aws_route_table" "private_a" {
    vpc_id = aws_vpc.prod.id
    route {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.az_a.id
    }
  }

  resource "aws_route_table" "private_b" {
    vpc_id = aws_vpc.prod.id
    route {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.az_b.id
    }
  }
```

### NAT Cost Optimization

```
┌─────────────────────────────────────────────────────────────────────┐
│              NAT GATEWAY COST OPTIMIZATION                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ NAT data processing ($0.045/GB) adds up fast!                        │
│                                                                       │
│ Strategies:                                                          │
│ ├── 1. VPC Endpoints for AWS services (S3, DynamoDB, ECR, etc.)    │
│ │      S3 Gateway Endpoint: FREE → saves NAT cost for S3 traffic   │
│ │      ECR Interface Endpoint: Saves docker pull NAT costs         │
│ │                                                                    │
│ ├── 2. Use S3 Gateway Endpoint (always, it's free!)                │
│ │      Saves: $0.045/GB for all S3 traffic                         │
│ │                                                                    │
│ ├── 3. ECR VPC Endpoints (for container workloads)                 │
│ │      Docker pulls can be HUGE (GB of images)                     │
│ │      com.amazonaws.region.ecr.api                                │
│ │      com.amazonaws.region.ecr.dkr                                │
│ │      com.amazonaws.region.s3 (for image layers)                  │
│ │                                                                    │
│ ├── 4. CloudWatch VPC Endpoint                                     │
│ │      Logs can generate significant traffic                       │
│ │                                                                    │
│ ├── 5. Use single NAT GW for non-prod (save $32+/month per AZ)   │
│ │                                                                    │
│ ├── 6. Check NAT Gateway CloudWatch metrics                       │
│ │      BytesOutToDestination, BytesOutToSource                    │
│ │      Find which destinations consume most data                  │
│ │                                                                    │
│ └── 7. Consider NAT Instance for very low traffic (<1GB/month)    │
│        t3.nano: ~$3.50/month vs NAT GW: $32/month                │
│        ⚠️ But you manage it (patching, HA, monitoring)             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: VPC Peering

```
┌─────────────────────────────────────────────────────────────────────┐
│                     VPC PEERING                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A networking connection between two VPCs that enables          │
│       routing traffic between them using private IPs.               │
│                                                                       │
│ ┌──────────────────┐     Peering     ┌──────────────────┐          │
│ │ Prod VPC          │ ◄═══════════► │ Shared VPC        │          │
│ │ 10.0.0.0/16       │   Connection   │ 10.3.0.0/16      │          │
│ │                    │               │                    │          │
│ │ App servers       │               │ Jenkins, Monitoring│          │
│ │ Databases         │               │ Bastion             │          │
│ └──────────────────┘               └──────────────────┘          │
│                                                                       │
│ Key rules:                                                           │
│ ├── 1. NO transitive peering!                                      │
│ │      A↔B and B↔C does NOT mean A↔C                               │
│ │      Each pair needs its own peering connection                  │
│ │                                                                    │
│ ├── 2. CIDRs cannot overlap                                       │
│ │      10.0.0.0/16 ↔ 10.0.0.0/16 ❌ (same CIDR)                  │
│ │      10.0.0.0/16 ↔ 10.1.0.0/16 ✅                               │
│ │                                                                    │
│ ├── 3. Works cross-region and cross-account                        │
│ │      VPC in us-east-1 ↔ VPC in ap-south-1 ✅                     │
│ │      VPC in Account A ↔ VPC in Account B ✅                      │
│ │                                                                    │
│ ├── 4. Requester-accepter model                                    │
│ │      One side sends request, other must accept                   │
│ │                                                                    │
│ ├── 5. Must update route tables on BOTH sides                      │
│ │                                                                    │
│ ├── 6. Security groups can reference peered VPC SG                 │
│ │      (same region only)                                          │
│ │                                                                    │
│ └── 7. Cost: FREE (data transfer charges apply for cross-region)   │
│        Same region: $0.01/GB each direction                        │
│        Cross region: Standard inter-region rates                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating VPC Peering

```
Console → VPC → Peering connections → Create peering connection

┌─────────────────────────────────────────────────────────────────┐
│             CREATE PEERING CONNECTION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [prod-to-shared-peering]                                  │
│                                                                   │
│ Requester:                                                       │
│   VPC ID (Requester): [vpc-prod ▼]                              │
│                                                                   │
│ Accepter:                                                        │
│   Account: ○ My account  ○ Another account                     │
│   Region:  ○ This Region ○ Another Region                      │
│   VPC ID (Accepter): [vpc-shared ▼]                             │
│                                                                   │
│ [Create peering connection]                                      │
│                                                                   │
│ After creation:                                                  │
│ 1. Accept the peering (if cross-account, login to other account)│
│    Status changes: pending-acceptance → active                  │
│                                                                   │
│ 2. Update BOTH route tables:                                    │
│    Prod VPC RT:   10.3.0.0/16 → pcx-xxxxx                     │
│    Shared VPC RT: 10.0.0.0/16 → pcx-xxxxx                     │
│                                                                   │
│ 3. Update Security Groups to allow traffic from peer CIDR       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create peering
  aws ec2 create-vpc-peering-connection \
    --vpc-id vpc-prod \
    --peer-vpc-id vpc-shared \
    --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=prod-to-shared}]'

  # Accept peering (other side)
  aws ec2 accept-vpc-peering-connection \
    --vpc-peering-connection-id pcx-xxxxx

  # Add routes (both sides)
  aws ec2 create-route --route-table-id rtb-prod --destination-cidr-block 10.3.0.0/16 --vpc-peering-connection-id pcx-xxxxx
  aws ec2 create-route --route-table-id rtb-shared --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id pcx-xxxxx
```

### Peering Limitations

```
The "No Transitive Peering" Problem:

  VPC-A ←→ VPC-B ←→ VPC-C

  A can talk to B ✅
  B can talk to C ✅
  A can talk to C ❌  (must create A↔C peering!)

  With 4 VPCs: Need 6 peering connections (n*(n-1)/2)
  With 10 VPCs: Need 45 peering connections!

  Solution: Transit Gateway (see next section)

  Full mesh peering:
  ┌─────┐     ┌─────┐
  │ VPC │─────│ VPC │
  │  A  │╲   ╱│  B  │
  └─────┘ ╲ ╱ └─────┘
           ╳
  ┌─────┐ ╱ ╲ ┌─────┐
  │ VPC │╱   ╲│ VPC │
  │  C  │─────│  D  │
  └─────┘     └─────┘

  6 peering connections needed! Gets messy fast.
```

---

## Part 3: Transit Gateway

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TRANSIT GATEWAY (TGW)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A regional network hub that connects VPCs and on-premises      │
│       networks through a central gateway.                            │
│                                                                       │
│ Without TGW (peering mesh):          With TGW (hub-and-spoke):      │
│                                                                       │
│  ┌───┐  ┌───┐  ┌───┐               ┌───┐                           │
│  │VPC│──│VPC│──│VPC│               │VPC│                            │
│  │ A │╲ │ B │╱ │ C │               │ A │──┐                         │
│  └───┘ ╲└───┘╱ └───┘               └───┘  │                         │
│         ╲   ╱                              │ ┌──────────────┐       │
│          ╲ ╱                        ┌───┐  ├─│ Transit      │       │
│  ┌───┐   ╳    ┌───┐               │VPC│──┤ │ Gateway      │       │
│  │VPC│──╱ ╲──│VPC│               │ B │  │ │ (Central Hub)│       │
│  │ D │╱     ╲│ E │               └───┘  │ └──────────────┘       │
│  └───┘       └───┘               ┌───┐  │        │               │
│                                   │VPC│──┤        │               │
│  10 peering connections!          │ C │  │    ┌───┴───┐          │
│                                   └───┘  │    │On-Prem│          │
│                                   ┌───┐  │    │(VPN)  │          │
│                                   │VPC│──┘    └───────┘          │
│                                   │ D │                            │
│                                   └───┘                            │
│                                   5 attachments (much simpler!)    │
│                                                                       │
│ Key features:                                                        │
│ ├── Hub-and-spoke model (any-to-any)                               │
│ ├── Supports: VPCs, VPN, Direct Connect                            │
│ ├── Route tables for traffic segmentation                          │
│ ├── Cross-region peering (TGW Peering)                             │
│ ├── Cross-account (via RAM sharing)                                │
│ ├── Bandwidth: Up to 50 Gbps per VPC attachment                   │
│ ├── Supports multicast                                              │
│ ├── Network Manager for global policy                              │
│ └── Transitive routing! (A → TGW → C works!)                      │
│                                                                       │
│ Pricing:                                                             │
│ ├── Per attachment: $0.05/hr ≈ $36/month per VPC                  │
│ ├── Per GB data processed: $0.02/GB                                │
│ └── More expensive than peering, but much simpler at scale         │
│                                                                       │
│ When to use:                                                         │
│ ├── 3+ VPCs that need to communicate                               │
│ ├── VPN/Direct Connect to multiple VPCs                            │
│ ├── Centralized egress (shared NAT Gateway)                        │
│ ├── Centralized inspection (firewall in hub VPC)                   │
│ └── Multi-account landing zone                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Transit Gateway

```
Console → VPC → Transit Gateways → Create transit gateway

┌─────────────────────────────────────────────────────────────────┐
│              CREATE TRANSIT GATEWAY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:                    [prod-tgw]                              │
│ Description:             [Central hub for all VPCs]             │
│ Amazon side ASN:         [64512] (for BGP — default is fine)    │
│                                                                   │
│ DNS support:             ☑ Enable                               │
│ VPN ECMP support:        ☑ Enable (load balance VPN tunnels)   │
│ Default route table                                              │
│   association:           ☑ Enable                               │
│ Default route table                                              │
│   propagation:           ☑ Enable                               │
│ Multicast support:       ☐                                      │
│ Auto accept shared                                               │
│   attachments:           ☐ (for cross-account via RAM)          │
│                                                                   │
│ [Create transit gateway]                                         │
│                                                                   │
│ Then attach VPCs:                                                │
│ Transit Gateway → Attachments → Create attachment               │
│   Name: [prod-vpc-attachment]                                   │
│   Transit gateway: [tgw-xxxxx ▼]                                │
│   Attachment type: [VPC ▼]                                      │
│   VPC ID: [vpc-prod ▼]                                          │
│   Subnet IDs: [subnet-private-a ▼] [subnet-private-b ▼]       │
│                                                                   │
│ Update VPC route tables:                                         │
│   Destination: 10.0.0.0/8 (or specific VPC CIDRs)             │
│   Target: tgw-xxxxx                                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### TGW Route Tables for Segmentation

```
┌─────────────────────────────────────────────────────────────────────┐
│          TGW ROUTE TABLE SEGMENTATION                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Prod and Dev should not talk to each other                 │
│           Both can access Shared Services                            │
│                                                                       │
│ TGW Route Table: "prod-rt"                                          │
│ ┌──────────────────┬──────────────────────┐                         │
│ │ Destination      │ Attachment           │                         │
│ │ 10.0.0.0/16      │ prod-vpc             │ (own VPC)              │
│ │ 10.3.0.0/16      │ shared-vpc           │ (allowed)             │
│ │                   │ NO dev route!        │ (blocked!)            │
│ └──────────────────┴──────────────────────┘                         │
│ Associated with: prod-vpc attachment                                │
│                                                                       │
│ TGW Route Table: "dev-rt"                                           │
│ ┌──────────────────┬──────────────────────┐                         │
│ │ Destination      │ Attachment           │                         │
│ │ 10.2.0.0/16      │ dev-vpc              │ (own VPC)              │
│ │ 10.3.0.0/16      │ shared-vpc           │ (allowed)             │
│ │                   │ NO prod route!       │ (blocked!)            │
│ └──────────────────┴──────────────────────┘                         │
│ Associated with: dev-vpc attachment                                 │
│                                                                       │
│ Result: Prod ↔ Shared ✅, Dev ↔ Shared ✅, Prod ↔ Dev ❌           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: VPC Endpoints

### Gateway Endpoints

```
┌─────────────────────────────────────────────────────────────────────┐
│                 GATEWAY ENDPOINTS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Private access to S3 and DynamoDB without using internet      │
│                                                                       │
│ Without Gateway Endpoint:                                            │
│ [Private VM] → NAT GW ($) → IGW → Internet → S3                    │
│                                                                       │
│ With Gateway Endpoint:                                               │
│ [Private VM] → Gateway Endpoint → S3 (private, within AWS)         │
│                                                                       │
│ Available for:                                                       │
│ ├── Amazon S3                                                       │
│ └── Amazon DynamoDB                                                 │
│ (Only these two! Everything else uses Interface Endpoints)          │
│                                                                       │
│ Key facts:                                                           │
│ ├── 🆓 FREE! (no hourly charge, no data processing charge)         │
│ ├── Adds a route to your route table (prefix list → vpce)         │
│ ├── Region-specific (com.amazonaws.ap-south-1.s3)                  │
│ ├── Cannot be used across VPC Peering/VPN/TGW                     │
│ ├── Can attach resource policies (restrict to specific buckets)    │
│ ├── HA and scalable (managed by AWS)                               │
│ └── ⚡ ALWAYS create S3 Gateway Endpoint — it's free!               │
│                                                                       │
│ Create:                                                              │
│ Console → VPC → Endpoints → Create endpoint                        │
│   Service: com.amazonaws.ap-south-1.s3 (Type: Gateway)             │
│   VPC: vpc-prod                                                     │
│   Route tables: [rt-private-a ✅] [rt-private-b ✅]                │
│   Policy: Full Access (or custom to restrict to specific buckets)  │
│                                                                       │
│ CLI:                                                                 │
│ aws ec2 create-vpc-endpoint \                                        │
│   --vpc-id vpc-prod \                                                │
│   --service-name com.amazonaws.ap-south-1.s3 \                       │
│   --route-table-ids rtb-private-a rtb-private-b                     │
│                                                                       │
│ Route table after creation:                                          │
│ ┌────────────────────┬──────────────────────┐                       │
│ │ Destination        │ Target               │                       │
│ │ 10.0.0.0/16        │ local                │                       │
│ │ 0.0.0.0/0          │ nat-xxxxx            │                       │
│ │ pl-xxxxxxx (S3)    │ vpce-xxxxx           │ ← Auto-added!       │
│ └────────────────────┴──────────────────────┘                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Interface Endpoints (AWS PrivateLink)

```
┌─────────────────────────────────────────────────────────────────────┐
│              INTERFACE ENDPOINTS (PrivateLink)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Private access to AWS services via an ENI in your subnet      │
│                                                                       │
│ How it works:                                                        │
│ ├── Creates an ENI (Elastic Network Interface) in your subnet      │
│ ├── ENI gets a private IP from your subnet CIDR                    │
│ ├── DNS resolves service endpoint to this private IP               │
│ ├── Traffic stays within AWS network                                │
│ └── Powered by AWS PrivateLink technology                          │
│                                                                       │
│ Without Interface Endpoint:                                          │
│ [VM] → NAT → IGW → Internet → ecr.api.amazonaws.com               │
│                                                                       │
│ With Interface Endpoint:                                             │
│ [VM] → ENI (10.0.11.25) → ECR (private, no internet!)             │
│                                                                       │
│ Available for (100+ services):                                       │
│ ├── ECR (ecr.api, ecr.dkr)                                         │
│ ├── CloudWatch Logs (logs)                                          │
│ ├── CloudWatch Monitoring (monitoring)                              │
│ ├── STS (sts)                                                       │
│ ├── SSM (ssm, ssmmessages, ec2messages)                            │
│ ├── Secrets Manager (secretsmanager)                                │
│ ├── KMS (kms)                                                       │
│ ├── SQS (sqs)                                                       │
│ ├── SNS (sns)                                                       │
│ ├── API Gateway (execute-api)                                       │
│ ├── ECS (ecs, ecs-agent, ecs-telemetry)                            │
│ └── ... many more                                                    │
│                                                                       │
│ Pricing:                                                             │
│ ├── Per hour per AZ: $0.01/hr ≈ $7.20/month per AZ                │
│ │   (2 AZs = $14.40/month per endpoint)                            │
│ ├── Per GB data processed: $0.01/GB                                │
│ └── Cheaper than NAT Gateway if high traffic to specific services  │
│                                                                       │
│ Key features:                                                        │
│ ├── Private DNS enabled by default (transparent to applications)   │
│ ├── Can attach Security Groups (control access)                    │
│ ├── Works across VPC Peering and TGW (unlike Gateway Endpoints!)  │
│ ├── HA: Deploy in 2+ AZs                                          │
│ └── Policy: Can restrict to specific resources                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Interface Endpoint

```
Console → VPC → Endpoints → Create endpoint

┌─────────────────────────────────────────────────────────────────┐
│             CREATE INTERFACE ENDPOINT                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:        [prod-ecr-api-endpoint]                            │
│ Service category: ○ AWS services ● AWS services (filtered)     │
│ Service:     [com.amazonaws.ap-south-1.ecr.api ▼]              │
│ Type:        Interface                                           │
│                                                                   │
│ VPC:         [vpc-prod ▼]                                       │
│                                                                   │
│ Subnets (select AZs):                                           │
│ ☑ ap-south-1a: [prod-private-a ▼]                              │
│ ☑ ap-south-1b: [prod-private-b ▼]                              │
│                                                                   │
│ Security groups:                                                 │
│ [vpce-sg ▼] (allow TCP 443 from VPC CIDR)                      │
│                                                                   │
│ Private DNS:  ☑ Enable (IMPORTANT!)                             │
│ └── Makes ecr.api.amazonaws.com resolve to private IP           │
│ └── No code changes needed in applications!                     │
│                                                                   │
│ Policy: Full Access (or restrict to specific registries)        │
│                                                                   │
│ [Create endpoint]                                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Essential endpoints for ECS/EKS workloads:
  com.amazonaws.REGION.ecr.api
  com.amazonaws.REGION.ecr.dkr
  com.amazonaws.REGION.s3        (Gateway — FREE)
  com.amazonaws.REGION.logs
  com.amazonaws.REGION.sts
  com.amazonaws.REGION.ssm       (if using SSM)
  com.amazonaws.REGION.secretsmanager
```

### Gateway vs Interface Endpoints

```
┌────────────────────────┬──────────────────┬──────────────────────┐
│ Feature                │ Gateway Endpoint │ Interface Endpoint   │
├────────────────────────┼──────────────────┼──────────────────────┤
│ Supported services     │ S3, DynamoDB only│ 100+ AWS services    │
│ How it works           │ Route table entry│ ENI in your subnet   │
│ Cost                   │ FREE!            │ $0.01/hr/AZ + data   │
│ Security Group         │ No               │ Yes                  │
│ Works over peering/TGW │ No               │ Yes                  │
│ Private DNS            │ No (uses prefix) │ Yes (transparent)    │
│ HA                     │ Built-in         │ Deploy in 2+ AZs     │
│ Policy                 │ Yes              │ Yes                  │
│ Implementation         │ Route table mod  │ ENI + DNS change     │
└────────────────────────┴──────────────────┴──────────────────────┘

Decision: For S3/DynamoDB → Gateway (free!)
          For everything else → Interface
```

---

## Part 5: AWS PrivateLink (Exposing Your Services)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS PRIVATELINK                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Expose YOUR service privately to other VPCs/accounts          │
│       (The technology behind Interface Endpoints)                    │
│                                                                       │
│ Use case: Your team runs an API that other teams/accounts need      │
│                                                                       │
│ Without PrivateLink:                                                 │
│ Other VPC → Internet → Your ALB (public) → Your service            │
│ ⚠️ Traffic goes over internet, need public exposure                 │
│                                                                       │
│ With PrivateLink:                                                    │
│ Other VPC → Interface Endpoint → NLB → Your service (all private!) │
│ ✅ No internet, no VPC peering needed, no CIDR overlap issues      │
│                                                                       │
│ Architecture:                                                        │
│                                                                       │
│ Provider VPC (10.0.0.0/16)        Consumer VPC (10.0.0.0/16)       │
│ ┌──────────────────────┐          ┌──────────────────────┐         │
│ │                      │          │                      │         │
│ │ [Your Service]       │          │ [Consumer App]       │         │
│ │      ↕                │          │      ↕                │         │
│ │ [NLB / GWLB]         │◄─────── │ [VPC Endpoint (ENI)] │         │
│ │      ↕                │ Private  │   10.0.5.42          │         │
│ │ [Endpoint Service]   │ Link     │                      │         │
│ │                      │          │                      │         │
│ └──────────────────────┘          └──────────────────────┘         │
│                                                                       │
│ ⚡ CIDRs CAN overlap! (no peering = no CIDR conflict)               │
│                                                                       │
│ Steps to set up:                                                     │
│ Provider side:                                                       │
│ 1. Deploy service behind NLB (Network Load Balancer)               │
│ 2. Create Endpoint Service pointing to NLB                         │
│ 3. Accept connection requests (or auto-accept)                     │
│                                                                       │
│ Consumer side:                                                       │
│ 1. Create Interface Endpoint pointing to the Endpoint Service      │
│ 2. Use the private DNS name or endpoint DNS to access              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Site-to-Site VPN

```
┌─────────────────────────────────────────────────────────────────────┐
│                   SITE-TO-SITE VPN                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Encrypted tunnel between your VPC and on-premises network     │
│                                                                       │
│ ┌──────────────────┐         Encrypted         ┌────────────────┐ │
│ │ AWS VPC           │ ◄══════ IPSec ══════════► │ Office/DC      │ │
│ │                    │       Tunnel(s)           │                │ │
│ │ Virtual Private   │                            │ Customer       │ │
│ │ Gateway (VGW)     │                            │ Gateway (CGW) │ │
│ └──────────────────┘                            └────────────────┘ │
│                                                                       │
│ Components:                                                          │
│ ├── Virtual Private Gateway (VGW): AWS side of VPN                 │
│ │   └── Attached to your VPC                                        │
│ ├── Customer Gateway (CGW): Represents your on-prem device         │
│ │   └── Your router/firewall IP and configuration                  │
│ └── VPN Connection: The actual tunnel(s)                            │
│     └── Always creates 2 tunnels (HA across AZs)                  │
│                                                                       │
│ Key facts:                                                           │
│ ├── IPSec encryption (AES-256)                                      │
│ ├── 2 tunnels per VPN connection (active/passive or ECMP)          │
│ ├── Bandwidth: Up to 1.25 Gbps per tunnel                         │
│ ├── Supports BGP (dynamic routing) or static routes               │
│ ├── Pricing: $0.05/hr ≈ $36/month per VPN connection              │
│ ├── Data transfer: Standard rates                                   │
│ └── Accelerated VPN: Uses Global Accelerator for better perf      │
│                                                                       │
│ Create:                                                              │
│ 1. Create Customer Gateway (your on-prem router IP)                │
│ 2. Create Virtual Private Gateway → Attach to VPC                  │
│ 3. Create VPN Connection (VGW + CGW)                                │
│ 4. Download configuration for your router                          │
│ 5. Update VPC route table (on-prem CIDR → VGW)                    │
│                                                                       │
│ When to use VPN vs Direct Connect:                                   │
│ ├── VPN: Quick setup (hours), lower cost, over internet            │
│ ├── Direct Connect: Dedicated line, higher bandwidth, consistent   │
│ │   latency, takes weeks to set up, $$$                            │
│ └── Both: VPN as backup for Direct Connect                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Direct Connect (Overview)

```
┌─────────────────────────────────────────────────────────────────────┐
│                  AWS DIRECT CONNECT                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Dedicated private network connection from your premises to AWS│
│                                                                       │
│ ┌──────────────┐                              ┌──────────────────┐ │
│ │ On-Premises  │── Physical fiber ──► DC ──►  │ AWS              │ │
│ │ Data Center  │    (not internet!)    Location│ Region           │ │
│ └──────────────┘                              └──────────────────┘ │
│                                                                       │
│ Key facts:                                                           │
│ ├── Physical fiber connection (NOT over internet)                  │
│ ├── Speeds: 50 Mbps to 100 Gbps                                   │
│ │   Dedicated: 1, 10, 100 Gbps (your own port)                   │
│ │   Hosted: 50 Mbps to 10 Gbps (shared via partner)              │
│ ├── Consistent latency (not affected by internet congestion)       │
│ ├── Lower data transfer costs vs internet                          │
│ ├── Takes 2-12 weeks to set up                                     │
│ ├── Requires colocation or partner                                 │
│ └── Monthly fee: Port + partner charges                            │
│                                                                       │
│ Virtual Interfaces (VIFs):                                           │
│ ├── Private VIF: Access VPC resources (private IPs)               │
│ ├── Public VIF: Access AWS public services (S3, etc.)             │
│ └── Transit VIF: Access Transit Gateway                            │
│                                                                       │
│ Best practice: Use Direct Connect + VPN backup                      │
│                                                                       │
│ When to use:                                                         │
│ ├── High bandwidth needs (>1 Gbps consistently)                   │
│ ├── Consistent low latency requirement                             │
│ ├── Large data transfers (TB/PB)                                   │
│ ├── Regulatory requirements (data not over internet)               │
│ └── Cost optimization (lower per-GB transfer costs)                │
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
│ What: Captures IP traffic flow information for network interfaces    │
│                                                                       │
│ Levels:                                                              │
│ ├── VPC level: All ENIs in the VPC                                 │
│ ├── Subnet level: All ENIs in the subnet                          │
│ └── ENI level: Specific network interface                          │
│                                                                       │
│ Destination:                                                         │
│ ├── CloudWatch Logs (easy to query, more expensive)                │
│ ├── S3 (cheaper storage, use Athena to query)                      │
│ └── Kinesis Data Firehose (real-time analysis)                     │
│                                                                       │
│ Sample flow log record:                                              │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ 2 123456789012 eni-abc123 10.0.1.5 54.210.100.50            │    │
│ │ 49152 443 6 20 4000 1620140661 1620140720 ACCEPT OK         │    │
│ │                                                                │    │
│ │ Fields:                                                        │    │
│ │ version, account, eni, srcaddr, dstaddr,                      │    │
│ │ srcport, dstport, protocol, packets, bytes,                   │    │
│ │ start, end, action, log-status                                 │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Use cases:                                                           │
│ ├── Troubleshoot: Why can't VM A connect to VM B?                  │
│ │   → Check for REJECT entries                                     │
│ ├── Security: Detect unusual traffic patterns                      │
│ ├── Compliance: Audit network access                               │
│ └── Cost: Find top traffic sources (NAT optimization)              │
│                                                                       │
│ Create:                                                              │
│ Console → VPC → Your VPC → Flow logs tab → Create flow log         │
│   Name: prod-vpc-flow-logs                                          │
│   Filter: All (Accept + Reject)                                     │
│   Maximum aggregation interval: 1 minute (or 10 minutes)          │
│   Destination: Send to CloudWatch Logs / S3                        │
│   Log format: AWS default or custom                                 │
│                                                                       │
│ ⚠️ Flow logs capture metadata only (NOT packet content!)            │
│    For packet inspection: Use Traffic Mirroring                     │
│                                                                       │
│ Pricing:                                                             │
│ ├── S3: ~$0.50/GB ingested                                         │
│ ├── CloudWatch: ~$0.50/GB ingested + storage                      │
│ └── Can generate significant volume — use filters wisely            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

# Query flow logs with Athena (when stored in S3)
SELECT srcaddr, dstaddr, dstport, action, COUNT(*) as flow_count
FROM vpc_flow_logs
WHERE action = 'REJECT'
  AND date = '2026/05/16'
GROUP BY srcaddr, dstaddr, dstport, action
ORDER BY flow_count DESC
LIMIT 20;
```

---

## Part 9: Real-World Architecture Patterns

### Startup: Simple VPC

```
┌─────────────────────────────────────────────────────────────────┐
│                    STARTUP PATTERN                                │
│                                                                   │
│  VPC: 10.0.0.0/16                                               │
│  ├── Public subnets (2 AZs): ALB, NAT GW                      │
│  ├── Private subnets (2 AZs): EC2/ECS, Lambda                 │
│  ├── Data subnets (2 AZs): RDS                                │
│  ├── 1 NAT Gateway (single AZ — save cost)                    │
│  ├── S3 Gateway Endpoint (free!)                               │
│  └── VPC Flow Logs to S3 (for debugging)                       │
│                                                                   │
│  Monthly cost: ~$36 (NAT GW + EIP)                              │
│                                                                   │
│  No peering, no TGW, no VPN needed yet                          │
└─────────────────────────────────────────────────────────────────┘
```

### Mid-Size: Multi-VPC with Peering

```
┌─────────────────────────────────────────────────────────────────┐
│                  MID-SIZE PATTERN                                 │
│                                                                   │
│  Prod VPC (10.0.0.0/16) ←peering→ Shared VPC (10.3.0.0/16)   │
│  Dev VPC (10.2.0.0/16) ←peering→ Shared VPC                   │
│                                                                   │
│  Each VPC:                                                      │
│  ├── 3-tier subnets (public, app, data)                        │
│  ├── NAT GW (HA for prod, single for dev)                     │
│  ├── S3 + DynamoDB Gateway Endpoints                           │
│  ├── Interface Endpoints (ECR, Logs, SSM)                      │
│  ├── VPC Flow Logs                                              │
│  └── NSGs + NACLs                                               │
│                                                                   │
│  Shared VPC: Bastion, CI/CD, monitoring                        │
│  Monthly cost: ~$200-400 networking                             │
└─────────────────────────────────────────────────────────────────┘
```

### Enterprise: Transit Gateway Hub

```
┌─────────────────────────────────────────────────────────────────┐
│                 ENTERPRISE PATTERN                                │
│                                                                   │
│  Transit Gateway (central hub)                                  │
│  ├── Prod VPC (10.0.0.0/16)                                   │
│  ├── Staging VPC (10.1.0.0/16)                                │
│  ├── Dev VPC (10.2.0.0/16)                                    │
│  ├── Shared Services VPC (10.3.0.0/16)                        │
│  │   ├── Centralized NAT (egress VPC)                         │
│  │   ├── Network Firewall (inspection VPC)                    │
│  │   ├── Bastion / SSM endpoints                               │
│  │   └── DNS resolvers                                         │
│  ├── VPN Connection → On-premises (192.168.0.0/16)            │
│  └── Direct Connect → Data Center                              │
│                                                                   │
│  TGW route tables for segmentation:                             │
│  ├── prod-rt: Prod ↔ Shared ✅, Prod ↔ Dev ❌                │
│  ├── dev-rt: Dev ↔ Shared ✅, Dev ↔ Prod ❌                  │
│  └── shared-rt: Shared ↔ All ✅, On-prem ↔ All ✅            │
│                                                                   │
│  Centralized VPC Endpoints in Shared VPC:                       │
│  ├── Interface Endpoints shared via TGW                        │
│  └── Saves cost vs endpoints in every VPC                      │
│                                                                   │
│  Monthly cost: $1,000-5,000+ networking                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Component | Purpose | Cost |
|-----------|---------|------|
| NAT Gateway | Private → internet | $32/mo + $0.045/GB |
| VPC Peering | Connect 2 VPCs | Free + data transfer |
| Transit Gateway | Hub for multiple VPCs | $36/mo/attachment + $0.02/GB |
| Gateway Endpoint | S3/DynamoDB private access | FREE |
| Interface Endpoint | AWS service private access | $7.20/mo/AZ + $0.01/GB |
| PrivateLink | Expose your service privately | $7.20/mo/AZ + $0.01/GB |
| Site-to-Site VPN | Encrypted tunnel to on-prem | $36/mo + data |
| Direct Connect | Dedicated line to on-prem | Port fee + partner fee |
| VPC Flow Logs | Network traffic logging | ~$0.50/GB |

---

## What's Next?

In the next chapter, we'll cover Security Groups and NACLs — the firewall rules that control traffic at the instance and subnet level.

→ Next: [Chapter 8: Security Groups & NACLs](08-security-groups-nacls.md)

---

*Last Updated: May 2026*
