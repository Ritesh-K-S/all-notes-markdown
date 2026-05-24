# Chapter 6: VPC - Virtual Private Cloud (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: What is a VPC?](#part-1-what-is-a-vpc)
- [Part 2: Default VPC vs Custom VPC](#part-2-default-vpc-vs-custom-vpc)
- [Part 3: CIDR Blocks and IP Addressing](#part-3-cidr-blocks-and-ip-addressing)
- [Part 4: Subnets](#part-4-subnets)
- [Part 5: Internet Gateway (IGW)](#part-5-internet-gateway-igw)
- [Part 6: Route Tables](#part-6-route-tables)
- [Part 7: Elastic IP Addresses](#part-7-elastic-ip-addresses)
- [Part 8: VPC Creation — Console Walkthrough](#part-8-vpc-creation--console-walkthrough)
- [Part 9: VPC Creation — CLI](#part-9-vpc-creation--cli)

---

## Overview

### What is a VPC? Why Do I Need to Understand Networking?

> **Real-World Analogy:** A VPC is like your own private office floor in a shared building. You control who can enter (Security Groups), which rooms connect to the hallway (public subnets) vs. which are internal only (private subnets), and where the front door to the street is (Internet Gateway). Other tenants can't see or access your floor.

**Why does this matter?** Every single AWS resource — EC2, RDS, Lambda, ECS — runs inside a VPC. If you don't understand VPC, you can't debug "why can't my EC2 reach the internet?" or "why can't my app connect to the database?" — the two most common beginner networking issues.

> 💡 **For your first VPC**, use `10.0.0.0/16`. It gives 65,536 IPs which is way more than you'll need, but leaves room for growth and subnets.

Amazon VPC lets you create a logically isolated virtual network in the AWS cloud. You have complete control over IP addressing, subnets, route tables, and network gateways. Every resource you launch in AWS (EC2, RDS, Lambda, etc.) runs inside a VPC.

```
What you'll learn:
├── What is a VPC and why you need it
├── Default VPC vs Custom VPC
├── CIDR blocks and IP addressing
├── Subnets (public vs private)
├── Internet Gateway (IGW)
├── Route Tables
├── Elastic IP Addresses
├── VPC creation walkthrough (Console + CLI + Terraform)
├── Multi-tier architecture design
├── DNS in VPC
├── VPC Limits and quotas
└── Real-world patterns (startup, mid-size, enterprise)
```

---

## Part 1: What is a VPC?

### The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AWS CLOUD                                    │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    YOUR VPC (10.0.0.0/16)                      │  │
│  │                    Your private network in AWS                  │  │
│  │                                                                 │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                    │  │
│  │  │  Public Subnet    │  │  Private Subnet   │                   │  │
│  │  │  10.0.1.0/24      │  │  10.0.2.0/24      │                   │  │
│  │  │                    │  │                    │                   │  │
│  │  │  ┌──────────────┐ │  │  ┌──────────────┐ │                   │  │
│  │  │  │  Web Server  │ │  │  │  Database     │ │                   │  │
│  │  │  │  (EC2)       │ │  │  │  (RDS)        │ │                   │  │
│  │  │  └──────────────┘ │  │  └──────────────┘ │                   │  │
│  │  └────────┬─────────┘  └──────────────────┘                    │  │
│  │           │                                                      │  │
│  │  ┌────────┴─────────┐                                           │  │
│  │  │  Internet Gateway │ ← Door to the internet                  │  │
│  │  └────────┬─────────┘                                           │  │
│  └───────────┼─────────────────────────────────────────────────────┘  │
│              │                                                        │
│         ─────┴───── INTERNET ─────                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

KEY CONCEPTS:
├── VPC = Your own private network (like your office LAN, but in the cloud)
├── Subnet = A segment of your VPC in a specific AZ
├── Public Subnet = Has route to Internet Gateway (internet-accessible)
├── Private Subnet = NO route to Internet Gateway (isolated)
├── Internet Gateway = The "door" between your VPC and the internet
├── Route Table = Rules that determine where traffic goes
├── CIDR Block = The IP address range for your VPC
└── Everything you launch in AWS goes into a VPC
```

### Why VPCs Matter

```
Without VPC:
├── All resources on the public internet
├── No network isolation
├── No control over IP ranges
└── Security nightmare!

With VPC:
├── Complete network isolation
├── You control which resources are public vs private
├── You define IP ranges
├── You control routing
├── You set firewall rules (Security Groups + NACLs)
├── You can connect to on-premises (VPN / Direct Connect)
└── Industry standard for any production workload
```

---

## Part 2: Default VPC vs Custom VPC

### Default VPC

```
┌─────────────────────────────────────────────────────────────────────┐
│                       DEFAULT VPC                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ AWS creates a DEFAULT VPC in every region for your account           │
│                                                                       │
│ What you get automatically:                                          │
│ ├── VPC with CIDR 172.31.0.0/16 (65,536 IPs)                      │
│ ├── One public subnet per AZ (e.g., 172.31.0.0/20, etc.)          │
│ ├── Internet Gateway attached                                       │
│ ├── Default route table (0.0.0.0/0 → IGW)                         │
│ ├── Default Security Group (allow all outbound, deny all inbound)  │
│ ├── Default NACL (allow all traffic)                                │
│ ├── Public IP auto-assign enabled on subnets                       │
│ └── DHCP options set                                                │
│                                                                       │
│ Characteristics:                                                     │
│ ├── ✅ Easy to start — just launch EC2, it works                    │
│ ├── ⚠️ All subnets are PUBLIC (auto-assign public IP)              │
│ ├── ⚠️ Not recommended for production                               │
│ ├── ⚠️ No private subnets by default                                │
│ ├── ⚠️ If deleted, can be recreated via Support or CLI              │
│ └── Best for: Quick testing, learning, prototypes                   │
│                                                                       │
│ Recreate default VPC:                                                │
│   aws ec2 create-default-vpc                                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Custom VPC (What You Should Use)

```
┌─────────────────────────────────────────────────────────────────────┐
│                       CUSTOM VPC                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ You create it yourself with:                                         │
│ ├── Your chosen CIDR block                                          │
│ ├── Your subnet layout (public + private)                           │
│ ├── Your route tables                                                │
│ ├── Your Internet Gateway (attached manually)                       │
│ ├── Your NAT Gateway (for private subnet internet access)           │
│ ├── Your Security Groups and NACLs                                  │
│                                                                       │
│ Characteristics:                                                     │
│ ├── ✅ Full control over network architecture                        │
│ ├── ✅ Proper public/private subnet separation                       │
│ ├── ✅ Required for production workloads                              │
│ ├── ✅ Can peer with other VPCs                                      │
│ ├── ✅ Can connect to on-premises                                    │
│ └── Best for: Everything beyond quick testing                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: CIDR Blocks and IP Addressing

### Understanding CIDR

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CIDR NOTATION EXPLAINED                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ CIDR = Classless Inter-Domain Routing                                │
│ Format: <IP address>/<prefix length>                                │
│                                                                       │
│ Examples:                                                            │
│ ┌────────────────┬──────────────┬─────────────────────────────┐    │
│ │ CIDR           │ Total IPs    │ Range                        │    │
│ ├────────────────┼──────────────┼─────────────────────────────┤    │
│ │ 10.0.0.0/8     │ 16,777,216   │ 10.0.0.0 - 10.255.255.255   │    │
│ │ 10.0.0.0/16    │ 65,536       │ 10.0.0.0 - 10.0.255.255     │    │
│ │ 10.0.0.0/20    │ 4,096        │ 10.0.0.0 - 10.0.15.255      │    │
│ │ 10.0.0.0/24    │ 256          │ 10.0.0.0 - 10.0.0.255       │    │
│ │ 10.0.0.0/28    │ 16           │ 10.0.0.0 - 10.0.0.15        │    │
│ └────────────────┴──────────────┴─────────────────────────────┘    │
│                                                                       │
│ Quick math: Total IPs = 2^(32 - prefix)                             │
│ /16 = 2^16 = 65,536 IPs                                             │
│ /24 = 2^8 = 256 IPs                                                 │
│ /28 = 2^4 = 16 IPs                                                  │
│                                                                       │
│ AWS VPC CIDR Rules:                                                  │
│ ├── Minimum: /28 (16 IPs)                                          │
│ ├── Maximum: /16 (65,536 IPs)                                      │
│ ├── Cannot be changed after creation (but can add secondary CIDRs) │
│ ├── Cannot overlap with other VPCs you want to peer                │
│ └── Use private IP ranges (RFC 1918):                               │
│     ├── 10.0.0.0/8     (10.0.0.0 → 10.255.255.255)                │
│     ├── 172.16.0.0/12  (172.16.0.0 → 172.31.255.255)              │
│     └── 192.168.0.0/16 (192.168.0.0 → 192.168.255.255)            │
│                                                                       │
│ ⚠️ PLAN YOUR CIDR CAREFULLY!                                        │
│ ├── Don't overlap with on-premises networks                        │
│ ├── Don't overlap with other VPCs you'll peer                      │
│ ├── Leave room for growth                                           │
│ └── Use /16 for production VPCs (plenty of room)                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### AWS Reserved IPs in Each Subnet

```
In every subnet, AWS reserves 5 IPs:

Example: Subnet 10.0.1.0/24 (256 IPs total, 251 usable)

┌──────────────┬──────────────────────────────────────────────────┐
│ IP Address   │ Purpose                                           │
├──────────────┼──────────────────────────────────────────────────┤
│ 10.0.1.0     │ Network address (cannot be used)                  │
│ 10.0.1.1     │ VPC Router (AWS uses this)                        │
│ 10.0.1.2     │ DNS server (Amazon-provided DNS)                  │
│ 10.0.1.3     │ Reserved for future AWS use                       │
│ 10.0.1.255   │ Broadcast address (not supported but reserved)    │
├──────────────┼──────────────────────────────────────────────────┤
│ Usable       │ 10.0.1.4 → 10.0.1.254 = 251 IPs                 │
└──────────────┴──────────────────────────────────────────────────┘

⚠️ This matters for small subnets:
/28 = 16 total - 5 reserved = 11 usable IPs
/24 = 256 total - 5 reserved = 251 usable IPs
```

### Recommended CIDR Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│              CIDR PLANNING STRATEGY                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Company-wide allocation:                                             │
│ ┌─────────────────────────────────────────────────────────────┐    │
│ │ Overall range:   10.0.0.0/8 (for all environments)          │    │
│ │                                                               │    │
│ │ Production VPC:  10.0.0.0/16   (65,536 IPs)                 │    │
│ │ Staging VPC:     10.1.0.0/16   (65,536 IPs)                 │    │
│ │ Development VPC: 10.2.0.0/16   (65,536 IPs)                 │    │
│ │ Shared/Mgmt VPC: 10.3.0.0/16   (65,536 IPs)                │    │
│ │ On-premises:     192.168.0.0/16 (no overlap!)               │    │
│ │                                                               │    │
│ │ 💡 This gives each VPC 65K IPs and none overlap              │    │
│ └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Within a VPC (10.0.0.0/16):                                         │
│ ┌─────────────────────────────────────────────────────────────┐    │
│ │ Public subnets (web tier):                                    │    │
│ │   AZ-a: 10.0.1.0/24   (251 usable IPs)                     │    │
│ │   AZ-b: 10.0.2.0/24   (251 usable IPs)                     │    │
│ │   AZ-c: 10.0.3.0/24   (251 usable IPs)                     │    │
│ │                                                               │    │
│ │ Private subnets (app tier):                                   │    │
│ │   AZ-a: 10.0.11.0/24  (251 usable IPs)                     │    │
│ │   AZ-b: 10.0.12.0/24  (251 usable IPs)                     │    │
│ │   AZ-c: 10.0.13.0/24  (251 usable IPs)                     │    │
│ │                                                               │    │
│ │ Data subnets (database tier):                                 │    │
│ │   AZ-a: 10.0.21.0/24  (251 usable IPs)                     │    │
│ │   AZ-b: 10.0.22.0/24  (251 usable IPs)                     │    │
│ │   AZ-c: 10.0.23.0/24  (251 usable IPs)                     │    │
│ └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Subnets

### Public vs Private Subnets

```
┌─────────────────────────────────────────────────────────────────────┐
│                 PUBLIC vs PRIVATE SUBNETS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ PUBLIC SUBNET                           PRIVATE SUBNET               │
│ ─────────────                           ──────────────               │
│ Has route to Internet Gateway           NO route to IGW              │
│ Resources CAN have public IPs           Resources have private IPs   │
│ Directly reachable from internet        NOT reachable from internet  │
│                                         (needs NAT for outbound)     │
│                                                                       │
│ What goes here:                         What goes here:              │
│ ├── Load Balancers                      ├── Application servers      │
│ ├── NAT Gateways                        ├── Databases (RDS)          │
│ ├── Bastion hosts                       ├── Cache (ElastiCache)      │
│ └── Public-facing services              ├── Backend workers          │
│                                         └── Internal microservices   │
│                                                                       │
│ Route table:                            Route table:                 │
│ ┌────────────────────────────┐         ┌────────────────────────┐   │
│ │ Dest        │ Target       │         │ Dest      │ Target     │   │
│ │ 10.0.0.0/16 │ local        │         │ 10.0.0.0/16│ local     │   │
│ │ 0.0.0.0/0   │ igw-xxxxx    │←KEY     │ 0.0.0.0/0 │ nat-xxxxx │   │
│ └────────────────────────────┘         └────────────────────────┘   │
│                                         (or no 0.0.0.0/0 route     │
│ ⚡ The ONLY difference between                for full isolation)   │
│ public and private subnet is the                                    │
│ route table entry pointing to IGW                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### How Subnets Work

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SUBNET RULES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Each subnet exists in exactly ONE Availability Zone               │
│    └── Subnet in AZ-a CANNOT span to AZ-b                          │
│                                                                       │
│ 2. Each subnet has exactly ONE route table                           │
│    └── But one route table can be shared by multiple subnets        │
│                                                                       │
│ 3. Subnet CIDR must be within VPC CIDR                               │
│    └── VPC: 10.0.0.0/16, Subnet: 10.0.1.0/24 ✅                   │
│    └── VPC: 10.0.0.0/16, Subnet: 10.1.0.0/24 ❌ (outside range)   │
│                                                                       │
│ 4. Subnet CIDRs within a VPC cannot overlap                         │
│    └── 10.0.1.0/24 and 10.0.1.0/25 ❌ (overlap)                    │
│    └── 10.0.1.0/24 and 10.0.2.0/24 ✅ (no overlap)                 │
│                                                                       │
│ 5. Auto-assign public IP can be enabled per subnet                   │
│    └── Enable for public subnets, disable for private               │
│                                                                       │
│ Best practice: Deploy at least 2 subnets in different AZs            │
│ for high availability (HA)                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Internet Gateway (IGW)

```
┌─────────────────────────────────────────────────────────────────────┐
│                  INTERNET GATEWAY (IGW)                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A horizontally scaled, redundant, high-available               │
│       component that allows communication between VPC and internet   │
│                                                                       │
│ Key facts:                                                           │
│ ├── ONE IGW per VPC (1:1 relationship)                              │
│ ├── Fully managed by AWS (no patching, no bandwidth limit)          │
│ ├── FREE! (no hourly cost, no data processing cost)                 │
│ ├── Horizontally scaled and redundant                                │
│ ├── Does NAT for instances with public IPv4 addresses               │
│ │   (translates private IP ↔ public IP)                             │
│ └── Must be attached to VPC to work                                 │
│                                                                       │
│ For a resource to reach the internet, ALL of these must be true:     │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ 1. ✅ VPC has an Internet Gateway attached                    │    │
│ │ 2. ✅ Subnet route table has 0.0.0.0/0 → IGW route          │    │
│ │ 3. ✅ Resource has a public IP or Elastic IP                  │    │
│ │ 4. ✅ Security Group allows the traffic                       │    │
│ │ 5. ✅ NACL allows the traffic                                 │    │
│ │                                                                │    │
│ │ Missing ANY one = no internet access!                         │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Traffic flow:                                                        │
│                                                                       │
│ [User] → Internet → IGW → Route Table → Subnet →                   │
│   Security Group → NACL → EC2 Instance                              │
│                                                                       │
│ [EC2] → NACL → Security Group → Subnet → Route Table →             │
│   IGW → Internet → [User]                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Route Tables

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ROUTE TABLES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A set of rules (routes) that determine where network           │
│       traffic from your subnet is directed.                          │
│                                                                       │
│ Key facts:                                                           │
│ ├── Every VPC has a "Main" route table (created automatically)      │
│ ├── Main route table applies to subnets with no explicit RT         │
│ ├── You can create custom route tables                               │
│ ├── Each subnet associates with exactly ONE route table             │
│ ├── Routes are evaluated from most specific to least specific       │
│ └── "local" route is always present (cannot delete it)              │
│                                                                       │
│ Route table entry format:                                            │
│ ┌──────────────────┬────────────────────────────────────────────┐   │
│ │ Destination      │ Target                                      │   │
│ ├──────────────────┼────────────────────────────────────────────┤   │
│ │ 10.0.0.0/16      │ local (within VPC — always present)        │   │
│ │ 0.0.0.0/0        │ igw-xxxxxxxx (Internet Gateway)            │   │
│ │ 0.0.0.0/0        │ nat-xxxxxxxx (NAT Gateway)                 │   │
│ │ 10.1.0.0/16      │ pcx-xxxxxxxx (VPC Peering)                 │   │
│ │ 10.3.0.0/16      │ tgw-xxxxxxxx (Transit Gateway)             │   │
│ │ 192.168.0.0/16   │ vgw-xxxxxxxx (Virtual Private Gateway/VPN) │   │
│ └──────────────────┴────────────────────────────────────────────┘   │
│                                                                       │
│ Example: Production VPC Route Tables                                 │
│                                                                       │
│ Public Route Table (rt-public):                                      │
│ ┌──────────────────┬──────────────────────┐                         │
│ │ Destination      │ Target               │                         │
│ │ 10.0.0.0/16      │ local                │ ← VPC internal         │
│ │ 0.0.0.0/0        │ igw-abc123           │ ← Internet             │
│ └──────────────────┴──────────────────────┘                         │
│ Associated with: public-subnet-a, public-subnet-b, public-subnet-c │
│                                                                       │
│ Private Route Table (rt-private):                                    │
│ ┌──────────────────┬──────────────────────┐                         │
│ │ Destination      │ Target               │                         │
│ │ 10.0.0.0/16      │ local                │ ← VPC internal         │
│ │ 0.0.0.0/0        │ nat-xyz789           │ ← Internet via NAT     │
│ └──────────────────┴──────────────────────┘                         │
│ Associated with: private-subnet-a, private-subnet-b, ...           │
│                                                                       │
│ ⚡ Route with most specific CIDR wins:                                │
│ If you have 10.0.0.0/16 → local AND 10.0.1.0/24 → peering         │
│ Traffic to 10.0.1.5 goes to PEERING (more specific /24 > /16)      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Elastic IP Addresses

```
┌─────────────────────────────────────────────────────────────────────┐
│                   ELASTIC IP (EIP)                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A static, public IPv4 address that you can allocate to your    │
│       account and associate with resources.                          │
│                                                                       │
│ Key facts:                                                           │
│ ├── Static — doesn't change (unlike auto-assigned public IP)        │
│ ├── Persists even if you stop/start the instance                    │
│ ├── Can be moved between instances (failover)                       │
│ ├── Associated with ONE resource at a time                          │
│ ├── Limit: 5 per region (can request increase)                      │
│ │                                                                    │
│ ├── PRICING:                                                        │
│ │   ├── In-use (associated with running instance): FREE             │
│ │   ├── NOT in use: $0.005/hour ≈ $3.65/month                      │
│ │   │   ⚠️ AWS charges for IDLE Elastic IPs!                       │
│ │   ├── Additional EIP on running instance: $0.005/hour            │
│ │   └── Public IPv4 (Feb 2024+): $0.005/hr for ALL public IPs!     │
│ │       ⚠️ NEW: Even auto-assigned public IPs now cost money       │
│ │                                                                    │
│ └── Use cases:                                                      │
│     ├── NAT Gateway requires EIP                                    │
│     ├── Instances that need a fixed public IP                       │
│     ├── Failover between instances                                  │
│     └── Whitelisting with third parties                             │
│                                                                       │
│ Allocate:                                                            │
│   aws ec2 allocate-address --domain vpc                              │
│                                                                       │
│ Associate with instance:                                             │
│   aws ec2 associate-address \                                        │
│     --instance-id i-1234567890abcdef0 \                              │
│     --allocation-id eipalloc-xxxxxxxx                                │
│                                                                       │
│ ⚠️ Always release unused EIPs! They cost money when idle.            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: VPC Creation — Console Walkthrough

### Method 1: VPC and More (Recommended)

```
Console → VPC → Create VPC

┌─────────────────────────────────────────────────────────────────┐
│                    CREATE VPC                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Resources to create:                                             │
│ ○ VPC only                                                      │
│ ● VPC and more ← RECOMMENDED (creates full network!)           │
│                                                                   │
│ ═══════════════════════════════════════════════════════════════  │
│                                                                   │
│ Auto-generate name tag: ☑ [prod]                                │
│ (Prefix for all resources: prod-vpc, prod-subnet-public1, etc.) │
│                                                                   │
│ IPv4 CIDR block: [10.0.0.0/16]                                  │
│   └── Size: 65,536 IP addresses                                │
│   └── ⚠️ Cannot change after creation                           │
│                                                                   │
│ IPv6 CIDR block:                                                 │
│   ○ No IPv6 CIDR block                                          │
│   ○ Amazon-provided IPv6 CIDR block                             │
│   ○ IPAM-allocated IPv6 CIDR block                              │
│                                                                   │
│ Tenancy: [Default ▼]                                            │
│   ├── Default: Shared hardware (cost-effective)                 │
│   └── Dedicated: Single-tenant hardware (expensive! compliance) │
│                                                                   │
│ ─────────────────────────────────────────────────────────────── │
│                                                                   │
│ Number of Availability Zones: [2 ▼] (1, 2, or 3)               │
│   └── Recommendation: 2 minimum for HA, 3 for production        │
│                                                                   │
│ Number of public subnets:  [2 ▼]                                │
│ Number of private subnets: [2 ▼]                                │
│                                                                   │
│ Subnet CIDR blocks:                                              │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Public subnet AZ-a:   [10.0.0.0/20]   (4,091 IPs)       │   │
│ │ Public subnet AZ-b:   [10.0.16.0/20]  (4,091 IPs)       │   │
│ │ Private subnet AZ-a:  [10.0.128.0/20] (4,091 IPs)       │   │
│ │ Private subnet AZ-b:  [10.0.144.0/20] (4,091 IPs)       │   │
│ │                                                            │   │
│ │ 💡 You can customize these to /24 for typical workloads   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ NAT gateways ($):                                                │
│ ○ None                                                          │
│ ● In 1 AZ ← Good for dev/staging ($32/month)                  │
│ ○ 1 per AZ ← Recommended for production ($32/month × AZs)     │
│                                                                   │
│ VPC endpoints:                                                   │
│ ☑ S3 Gateway ← FREE! Always enable                             │
│ ☐ DynamoDB Gateway                                               │
│                                                                   │
│ DNS options:                                                     │
│ ☑ Enable DNS hostnames                                          │
│ ☑ Enable DNS resolution                                         │
│                                                                   │
│ [Create VPC]                                                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

What gets created with "VPC and more":
├── 1 VPC
├── 2 public subnets (one per AZ)
├── 2 private subnets (one per AZ)
├── 1 Internet Gateway (attached to VPC)
├── 1 NAT Gateway (if selected) + 1 Elastic IP
├── 2 route tables (public + private)
│   ├── Public RT: 0.0.0.0/0 → IGW
│   └── Private RT: 0.0.0.0/0 → NAT Gateway
├── S3 Gateway Endpoint (if selected)
└── All properly connected!

💡 This is the EASIEST way to create a production-ready VPC
```

### Method 2: VPC Only (Manual Setup)

```
If you choose "VPC only", you must manually:
1. Create VPC
2. Create Internet Gateway → Attach to VPC
3. Create subnets (specify AZ + CIDR for each)
4. Create route tables
5. Add routes (0.0.0.0/0 → IGW for public RT)
6. Associate subnets with route tables
7. Enable auto-assign public IP on public subnets
8. Create NAT Gateway (if needed)
9. Add NAT route to private RT

More work but useful for understanding each component.
```

---

## Part 9: VPC Creation — CLI

```bash
# 1. Create VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc},{Key=Environment,Value=production}]'

# Output: VpcId = vpc-0abc123def456

# 2. Enable DNS hostname support
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-0abc123def456 \
  --enable-dns-hostnames '{"Value": true}'

# 3. Create Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-igw}]'

# Output: InternetGatewayId = igw-0abc123

# 4. Attach IGW to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0abc123 \
  --vpc-id vpc-0abc123def456

# 5. Create public subnets
aws ec2 create-subnet \
  --vpc-id vpc-0abc123def456 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-public-a}]'

aws ec2 create-subnet \
  --vpc-id vpc-0abc123def456 \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-public-b}]'

# 6. Create private subnets
aws ec2 create-subnet \
  --vpc-id vpc-0abc123def456 \
  --cidr-block 10.0.11.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-private-a}]'

aws ec2 create-subnet \
  --vpc-id vpc-0abc123def456 \
  --cidr-block 10.0.12.0/24 \
  --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-private-b}]'

# 7. Enable auto-assign public IP on public subnets
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-pub-a \
  --map-public-ip-on-launch

# 8. Create public route table
aws ec2 create-route-table \
  --vpc-id vpc-0abc123def456 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=prod-public-rt}]'

# 9. Add internet route to public RT
aws ec2 create-route \
  --route-table-id rtb-public \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc123

# 10. Associate public subnets with public RT
aws ec2 associate-route-table \
  --route-table-id rtb-public \
  --subnet-id subnet-pub-a

aws ec2 associate-route-table \
  --route-table-id rtb-public \
  --subnet-id subnet-pub-b

# 11. Create NAT Gateway (in public subnet!)
# First: Allocate Elastic IP
aws ec2 allocate-address --domain vpc
# Output: AllocationId = eipalloc-xxxxx

aws ec2 create-nat-gateway \
  --subnet-id subnet-pub-a \
  --allocation-id eipalloc-xxxxx \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-nat}]'

# 12. Create private route table
aws ec2 create-route-table \
  --vpc-id vpc-0abc123def456 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=prod-private-rt}]'

# 13. Add NAT route to private RT
aws ec2 create-route \
  --route-table-id rtb-private \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-xxxxx

# 14. Associate private subnets with private RT
aws ec2 associate-route-table \
  --route-table-id rtb-private \
  --subnet-id subnet-priv-a

aws ec2 associate-route-table \
  --route-table-id rtb-private \
  --subnet-id subnet-priv-b
```

---

## Part 10: VPC Creation — Terraform

```hcl
# terraform/vpc.tf

# ──────────────────────────────────────
# VPC
# ──────────────────────────────────────
resource "aws_vpc" "prod" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "prod-vpc"
    Environment = "production"
  }
}

# ──────────────────────────────────────
# Internet Gateway
# ──────────────────────────────────────
resource "aws_internet_gateway" "prod" {
  vpc_id = aws_vpc.prod.id

  tags = {
    Name = "prod-igw"
  }
}

# ──────────────────────────────────────
# Public Subnets
# ──────────────────────────────────────
resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.prod.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "prod-public-a"
    Tier = "public"
  }
}

resource "aws_subnet" "public_b" {
  vpc_id                  = aws_vpc.prod.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "ap-south-1b"
  map_public_ip_on_launch = true

  tags = {
    Name = "prod-public-b"
    Tier = "public"
  }
}

# ──────────────────────────────────────
# Private Subnets
# ──────────────────────────────────────
resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.prod.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "ap-south-1a"

  tags = {
    Name = "prod-private-a"
    Tier = "private"
  }
}

resource "aws_subnet" "private_b" {
  vpc_id            = aws_vpc.prod.id
  cidr_block        = "10.0.12.0/24"
  availability_zone = "ap-south-1b"

  tags = {
    Name = "prod-private-b"
    Tier = "private"
  }
}

# ──────────────────────────────────────
# Elastic IP for NAT Gateway
# ──────────────────────────────────────
resource "aws_eip" "nat" {
  domain = "vpc"

  tags = {
    Name = "prod-nat-eip"
  }
}

# ──────────────────────────────────────
# NAT Gateway (in public subnet!)
# ──────────────────────────────────────
resource "aws_nat_gateway" "prod" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_a.id

  tags = {
    Name = "prod-nat"
  }

  depends_on = [aws_internet_gateway.prod]
}

# ──────────────────────────────────────
# Public Route Table
# ──────────────────────────────────────
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.prod.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.prod.id
  }

  tags = {
    Name = "prod-public-rt"
  }
}

resource "aws_route_table_association" "public_a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}

# ──────────────────────────────────────
# Private Route Table
# ──────────────────────────────────────
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.prod.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.prod.id
  }

  tags = {
    Name = "prod-private-rt"
  }
}

resource "aws_route_table_association" "private_a" {
  subnet_id      = aws_subnet.private_a.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "private_b" {
  subnet_id      = aws_subnet.private_b.id
  route_table_id = aws_route_table.private.id
}
```

---

## Part 11: Multi-Tier Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│            STANDARD 3-TIER VPC ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                         INTERNET                                     │
│                            │                                          │
│                    ┌───────┴────────┐                                │
│                    │ Internet Gateway│                                │
│                    └───────┬────────┘                                │
│                            │                                          │
│  ┌─────────────────────────┼─────────────────────────────────────┐  │
│  │                  VPC: 10.0.0.0/16                              │  │
│  │                         │                                       │  │
│  │  ┌─────────────────────┼─────────────────────────────────┐   │  │
│  │  │         PUBLIC SUBNETS (Web Tier)                       │   │  │
│  │  │                      │                                  │   │  │
│  │  │  AZ-a: 10.0.1.0/24  │  AZ-b: 10.0.2.0/24             │   │  │
│  │  │  ┌────────────────┐ │  ┌────────────────┐             │   │  │
│  │  │  │ ALB            │ │  │ ALB            │              │   │  │
│  │  │  │ NAT Gateway    │ │  │ (NAT GW if HA)│              │   │  │
│  │  │  │ Bastion Host   │ │  │               │              │   │  │
│  │  │  └────────────────┘ │  └────────────────┘             │   │  │
│  │  └─────────────────────┼─────────────────────────────────┘   │  │
│  │                         │                                       │  │
│  │  ┌─────────────────────┼─────────────────────────────────┐   │  │
│  │  │         PRIVATE SUBNETS (App Tier)                      │   │  │
│  │  │                      │                                  │   │  │
│  │  │  AZ-a: 10.0.11.0/24 │  AZ-b: 10.0.12.0/24            │   │  │
│  │  │  ┌────────────────┐ │  ┌────────────────┐             │   │  │
│  │  │  │ App Server 1   │ │  │ App Server 2   │             │   │  │
│  │  │  │ (EC2 / ECS)    │ │  │ (EC2 / ECS)    │             │   │  │
│  │  │  └────────────────┘ │  └────────────────┘             │   │  │
│  │  └─────────────────────┼─────────────────────────────────┘   │  │
│  │                         │                                       │  │
│  │  ┌─────────────────────┼─────────────────────────────────┐   │  │
│  │  │         PRIVATE SUBNETS (Data Tier)                     │   │  │
│  │  │                      │                                  │   │  │
│  │  │  AZ-a: 10.0.21.0/24 │  AZ-b: 10.0.22.0/24            │   │  │
│  │  │  ┌────────────────┐ │  ┌────────────────┐             │   │  │
│  │  │  │ RDS Primary    │ │  │ RDS Standby    │             │   │  │
│  │  │  │ ElastiCache    │ │  │ ElastiCache    │             │   │  │
│  │  │  └────────────────┘ │  └────────────────┘             │   │  │
│  │  └─────────────────────┼─────────────────────────────────┘   │  │
│  │                                                                 │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Traffic flow:                                                        │
│ User → ALB (public) → App Server (private) → DB (private)          │
│ App → NAT GW (public) → Internet (for package updates, APIs)       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: DNS in VPC

```
┌─────────────────────────────────────────────────────────────────────┐
│                      DNS IN VPC                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ VPC DNS Settings (two key flags):                                    │
│                                                                       │
│ 1. enableDnsSupport (DNS Resolution)                                 │
│    ├── Default: true                                                │
│    ├── Queries Route 53 Resolver (at VPC CIDR + 2 address)         │
│    └── Must be true for DNS to work in VPC                          │
│                                                                       │
│ 2. enableDnsHostnames (DNS Hostnames)                                │
│    ├── Default: false (for custom VPC), true (for default VPC)      │
│    ├── Assigns public DNS hostname to instances with public IP      │
│    │   e.g., ec2-54-210-167-99.compute-1.amazonaws.com              │
│    ├── Also assigns private DNS hostname                            │
│    │   e.g., ip-10-0-1-42.ec2.internal                              │
│    └── Must enable for VPC Endpoints, private hosted zones          │
│                                                                       │
│ Best practice: Enable BOTH for production VPCs                       │
│                                                                       │
│ Amazon-provided DNS:                                                 │
│ ├── VPC CIDR base + 2 (e.g., 10.0.0.2 for 10.0.0.0/16)           │
│ ├── Also accessible at 169.254.169.253                              │
│ ├── Resolves: public DNS → private IP within VPC                   │
│ ├── Resolves: private DNS hostnames                                 │
│ └── Forwards external queries to public DNS                         │
│                                                                       │
│ DHCP Options Set:                                                    │
│ ├── Controls DNS servers, domain name, NTP servers                  │
│ ├── Each VPC has one DHCP options set                               │
│ ├── Default: Amazon-provided DNS                                    │
│ ├── Custom: Point to your own DNS servers                           │
│ └── Cannot modify in-place — create new and associate               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: VPC Limits and Quotas

```
┌────────────────────────────────┬────────────────┬──────────────────┐
│ Resource                       │ Default Limit  │ Adjustable?      │
├────────────────────────────────┼────────────────┼──────────────────┤
│ VPCs per region                │ 5              │ Yes (up to 100+) │
│ Subnets per VPC                │ 200            │ Yes              │
│ IPv4 CIDR blocks per VPC       │ 5 (secondary)  │ Yes (up to 50)   │
│ Internet Gateways per region   │ 5 (1 per VPC)  │ Yes              │
│ NAT Gateways per AZ            │ 5              │ Yes              │
│ Elastic IPs per region         │ 5              │ Yes              │
│ Route tables per VPC           │ 200            │ Yes              │
│ Routes per route table         │ 50             │ Yes (up to 1000) │
│ Security Groups per VPC        │ 2,500          │ Yes              │
│ Rules per Security Group       │ 60 (in+out)    │ Yes (up to 200)  │
│ Network ACLs per VPC           │ 200            │ Yes              │
│ VPC Peering per VPC            │ 50             │ Yes (up to 125)  │
│ VPC Endpoints per VPC          │ 20 (Gateway)   │ Yes              │
│                                │ 50 (Interface) │ Yes              │
└────────────────────────────────┴────────────────┴──────────────────┘

💡 Request limit increases via AWS Support or Service Quotas console.
```

---

## Part 14: Real-World Patterns

### Startup (5-10 developers)

```
Architecture:
├── 1 VPC per environment (prod + dev) = 2 VPCs
├── VPC CIDR: 10.0.0.0/16 (prod), 10.1.0.0/16 (dev)
├── 2 AZs (cost-effective HA)
├── 2 public subnets + 2 private subnets
├── 1 NAT Gateway (single AZ — save $32/month)
├── S3 Gateway Endpoint (free)
├── No VPC Peering yet (simple setup)
└── Total VPC cost: ~$32/month (NAT GW) + EIP ($3.65)

Network flow:
  Users → ALB (public subnet) → App (private subnet) → DB (private)
  App → NAT GW → Internet (API calls, packages)

Do:
├── Use "VPC and more" wizard for quick setup
├── Put DB in private subnets
├── Enable DNS hostnames
├── Use Security Groups (not NACLs) for firewall rules
└── Tag everything (Environment, Team)

Don't:
├── Put databases in public subnets
├── Open port 22 (SSH) to 0.0.0.0/0
├── Use default VPC for production
└── Over-engineer with 3 AZs if budget is tight
```

### Mid-Size (50-100 developers)

```
Architecture:
├── Separate VPC per environment: Prod, Staging, Dev, Shared-Services
├── CIDR plan: 10.0.0.0/16, 10.1.0.0/16, 10.2.0.0/16, 10.3.0.0/16
├── 3 AZs for production (full HA)
├── 3-tier subnets: Public, App-Private, Data-Private per AZ
├── NAT Gateway per AZ in production (HA)
├── 1 NAT Gateway in dev/staging
├── VPC Peering or Transit Gateway between VPCs
├── VPC Endpoints for S3, DynamoDB, ECR, CloudWatch
│   (reduces NAT costs + faster)
├── Shared Services VPC for tools (Jenkins, monitoring, bastion)
└── Total: ~$200-400/month networking

Do:
├── Use Transit Gateway if >3 VPCs (simplifies routing)
├── Use VPC Flow Logs for security/debugging
├── Implement proper CIDR planning document
├── VPC Endpoints for frequently used AWS services
├── Centralize bastion/jump host in Shared Services VPC
└── Consider AWS Systems Manager Session Manager instead of bastion

Don't:
├── Use overlapping CIDRs (blocks peering)
├── Use single NAT GW for production (single AZ failure)
└── Skip VPC Flow Logs in production
```

### Enterprise (500+ developers)

```
Architecture:
├── Landing Zone / Control Tower setup
├── Separate AWS account per workload/team + VPC per account
├── Hub-and-spoke: Transit Gateway as central hub
│   ├── Shared Services VPC (hub)
│   ├── Prod VPCs (spokes)
│   ├── Dev VPCs (spokes)
│   └── On-premises (spoke via VPN/Direct Connect)
├── Centralized egress: All internet traffic via shared NAT
├── Centralized inspection: AWS Network Firewall in transit VPC
├── IPAM (IP Address Manager) for CIDR allocation
├── RAM (Resource Access Manager) to share subnets
├── VPC Endpoints in shared VPC (PrivateLink)
├── Direct Connect for on-premises connectivity
└── Total: $1,000-5,000+/month networking

Advanced:
├── AWS Network Firewall for deep packet inspection
├── Gateway Load Balancer for 3rd party firewalls
├── AWS PrivateLink for exposing services across accounts
├── Route 53 Resolver for hybrid DNS (cloud ↔ on-prem)
├── Multiple Direct Connect links (redundancy)
└── TGW route tables for network segmentation
```

---

## Troubleshooting: Common VPC Issues

### "My EC2 instance can't reach the internet"

```
Checklist (check in this order):

1. ☐ Is an Internet Gateway attached to the VPC?
   Console → VPC → Internet Gateways → check "Attached"

2. ☐ Does the subnet's route table have 0.0.0.0/0 → IGW?
   Console → VPC → Subnets → select subnet → Route table tab

3. ☐ Does the instance have a public IP or Elastic IP?
   Console → EC2 → Instances → check "Public IPv4 address"
   (Private subnets use NAT Gateway instead — see step 5)

4. ☐ Does the Security Group allow outbound traffic?
   Console → EC2 → Security Groups → Outbound rules
   (Default SG allows all outbound — custom SGs may not)

5. ☐ For private subnets: Is there a NAT Gateway?
   Route table should have 0.0.0.0/0 → nat-xxxxx

6. ☐ Does the NACL allow the traffic?
   Console → VPC → Network ACLs → check both inbound AND outbound
   (Remember: NACLs are stateless — you need rules in both directions)
```

### "My EC2 can't connect to my RDS database"

```
1. ☐ Are they in the same VPC?
2. ☐ Does the RDS security group allow inbound on port 3306/5432
   from the EC2's security group?
   (Best practice: reference SG ID, not IP addresses)
3. ☐ Is the RDS in a subnet that the EC2 can route to?
4. ☐ Is the RDS publicly accessible? (Should be NO for production)
```

### Common Beginner Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Launching in wrong region | Resources invisible in console | Always check region selector (top-right) |
| Deleting default VPC | Some services need it | Recreate: `Actions → Create default VPC` |
| Using 0.0.0.0/0 in Security Groups | Open to the entire internet | Use specific IPs or SG references |
| Forgetting NAT Gateway costs | ~$32/month minimum | Delete NAT GW in dev environments at night |
| Not using multiple AZs | Single point of failure | Always deploy subnets in 2-3 AZs |

---

## Quick Reference

| Component | Purpose | Cost |
|-----------|---------|------|
| VPC | Virtual network | FREE |
| Subnet | Network segment in an AZ | FREE |
| Internet Gateway | Internet access | FREE |
| Route Table | Traffic routing rules | FREE |
| Elastic IP | Static public IP | $3.65/mo (idle) |
| Public IPv4 | Any public IP (since Feb 2024) | $3.65/mo |
| NAT Gateway | Private subnet → internet | $32/mo + $0.045/GB |
| VPC Peering | Connect two VPCs | FREE (data transfer only) |
| Transit Gateway | Hub for multiple VPCs | $0.05/hr + $0.02/GB |
| VPC Endpoint (Gateway) | S3/DynamoDB private access | FREE |
| VPC Endpoint (Interface) | Other services private access | $0.01/hr + $0.01/GB |

---

## What's Next?

In the next chapter, we'll cover VPC Advanced Networking — NAT Gateway deep dive, VPC Peering, Transit Gateway, VPC Endpoints, and PrivateLink.

→ Next: [Chapter 7: VPC Advanced Networking](07-vpc-advanced.md)

---

*Last Updated: May 2026*
