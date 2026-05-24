# Chapter 8: Security Groups & NACLs (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Security Groups (SG)](#part-1-security-groups-sg)
- [Part 2: Network ACLs (NACLs)](#part-2-network-acls-nacls)
- [Part 3: Security Groups vs NACLs](#part-3-security-groups-vs-nacls)
- [Part 4: Common Security Group Patterns](#part-4-common-security-group-patterns)

---

## Overview

### What Are Security Groups and NACLs? Why Two Layers?

> **Real-World Analogy:** Security Groups are like a personal bodyguard for each server — they only let through people on the guest list (allow rules only, no block list). NACLs are like the security checkpoint at the building entrance — they check everyone both entering AND leaving, and can explicitly block banned visitors. You want both layers.

**Why does this matter?** "Port 443 isn't open" and "my app can't connect to the database" are the two most common issues beginners face. **90% of the time, it's a Security Group misconfiguration.** Understanding how SGs and NACLs work will save you hours of debugging.

| Feature | Security Group | NACL |
|---------|---------------|------|
| Level | Instance (ENI) | Subnet |
| Stateful? | Yes (return traffic auto-allowed) | No (must allow both directions) |
| Rule type | Allow only | Allow AND Deny |
| Evaluation | All rules checked | In order (first match wins) |
| Default inbound | Deny all | Allow all |

AWS provides two layers of network security at the VPC level: Security Groups (instance-level, stateful) and Network ACLs (subnet-level, stateless). Understanding when and how to use each is critical for a proper defense-in-depth strategy.

```
What you'll learn:
├── Security Groups (deep dive)
│   ├── How they work (stateful)
│   ├── Inbound & outbound rules
│   ├── All fields explained
│   ├── Referencing other SGs
│   ├── Best practices
│   └── Common patterns
├── Network ACLs (NACLs)
│   ├── How they work (stateless)
│   ├── Rule numbering & evaluation
│   ├── Ephemeral ports
│   └── When to use
├── Security Groups vs NACLs
├── Default VPC security
└── Real-world architecture patterns
```

---

## Part 1: Security Groups (SG)

### What and How They Work

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURITY GROUPS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A virtual firewall that controls traffic at the                │
│       INSTANCE (ENI) level — attached to EC2, RDS, ELB, etc.       │
│                                                                       │
│ Key characteristics:                                                 │
│ ┌─────────────────────────────────────────────────────────────┐     │
│ │ 1. STATEFUL                                                   │     │
│ │    If you allow inbound traffic, the response is               │     │
│ │    automatically allowed outbound (and vice versa).           │     │
│ │    You do NOT need to create a matching outbound rule.        │     │
│ │                                                                │     │
│ │    Example: Allow inbound TCP 443                              │     │
│ │    → Response traffic on ephemeral port auto-allowed out      │     │
│ │                                                                │     │
│ │ 2. ALLOW ONLY (no deny rules)                                 │     │
│ │    You can only create ALLOW rules.                           │     │
│ │    Everything not explicitly allowed is DENIED.               │     │
│ │    → Implicit deny (default: deny all inbound)                │     │
│ │                                                                │     │
│ │ 3. ALL RULES EVALUATED                                        │     │
│ │    All rules are evaluated before deciding.                   │     │
│ │    No priority/ordering — if ANY rule allows, it's allowed.  │     │
│ │                                                                │     │
│ │ 4. ATTACHED TO ENI (not subnet or VPC)                        │     │
│ │    Applied at the network interface level.                    │     │
│ │    One ENI can have up to 5 SGs (default, increasable).      │     │
│ │    One SG can be attached to multiple instances.              │     │
│ │                                                                │     │
│ │ 5. VPC-SCOPED                                                 │     │
│ │    Security group belongs to a VPC.                           │     │
│ │    Can reference SGs in same VPC or peered VPC (same region).│     │
│ └─────────────────────────────────────────────────────────────┘     │
│                                                                       │
│ Traffic flow:                                                        │
│                                                                       │
│ [Internet] ──► [NACL (subnet)] ──► [Security Group (instance)]     │
│                                          │                            │
│                                          ▼                            │
│                                     [EC2 Instance]                   │
│                                          │                            │
│                                          ▼                            │
│ [Internet] ◄── [NACL (subnet)] ◄── [Security Group (instance)]     │
│                                                                       │
│ Inbound: NACL first → then SG                                       │
│ Outbound: SG first → then NACL                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Default Security Group

```
┌─────────────────────────────────────────────────────────────────────┐
│              DEFAULT SECURITY GROUP                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Every VPC comes with a DEFAULT Security Group.                       │
│                                                                       │
│ Default INBOUND rules:                                               │
│ ┌──────────┬──────────┬──────┬────────┬──────────────────────┐     │
│ │ Type     │ Protocol │ Port │ Source │ Description          │     │
│ ├──────────┼──────────┼──────┼────────┼──────────────────────┤     │
│ │ All      │ All      │ All  │ sg-xxx │ Allow from same SG   │     │
│ │ traffic  │          │      │ (self) │ (instances in this   │     │
│ │          │          │      │        │  SG can talk to each │     │
│ │          │          │      │        │  other)              │     │
│ └──────────┴──────────┴──────┴────────┴──────────────────────┘     │
│                                                                       │
│ Default OUTBOUND rules:                                              │
│ ┌──────────┬──────────┬──────┬─────────────┬─────────────────┐     │
│ │ Type     │ Protocol │ Port │ Destination │ Description     │     │
│ ├──────────┼──────────┼──────┼─────────────┼─────────────────┤     │
│ │ All      │ All      │ All  │ 0.0.0.0/0   │ Allow all       │     │
│ │ traffic  │          │      │             │ outbound        │     │
│ └──────────┴──────────┴──────┴─────────────┴─────────────────┘     │
│                                                                       │
│ ⚠️ Best practice: NEVER use the default SG!                          │
│    Create custom SGs with specific rules.                            │
│    Default SG allows all traffic between members — too permissive.  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Security Group

```
Console → VPC → Security groups → Create security group

┌─────────────────────────────────────────────────────────────────┐
│              CREATE SECURITY GROUP                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Basic details:                                                   │
│   Security group name: [web-server-sg]                          │
│   Description:         [Allow HTTP/HTTPS from internet,         │
│                          SSH from bastion only]                  │
│   VPC:                 [prod-vpc ▼]                             │
│                                                                   │
│ Inbound rules:                                                   │
│ ┌──────────┬──────────┬───────┬──────────────┬──────────────┐  │
│ │ Type     │ Protocol │ Port  │ Source       │ Description  │  │
│ ├──────────┼──────────┼───────┼──────────────┼──────────────┤  │
│ │ HTTP     │ TCP      │ 80    │ 0.0.0.0/0   │ Public HTTP  │  │
│ │ HTTPS    │ TCP      │ 443   │ 0.0.0.0/0   │ Public HTTPS │  │
│ │ SSH      │ TCP      │ 22    │ sg-bastion   │ SSH from     │  │
│ │          │          │       │              │ bastion only │  │
│ └──────────┴──────────┴───────┴──────────────┴──────────────┘  │
│ [Add rule]                                                       │
│                                                                   │
│ Outbound rules:                                                  │
│ ┌──────────┬──────────┬───────┬──────────────┬──────────────┐  │
│ │ Type     │ Protocol │ Port  │ Destination  │ Description  │  │
│ ├──────────┼──────────┼───────┼──────────────┼──────────────┤  │
│ │ All      │ All      │ All   │ 0.0.0.0/0   │ Allow all    │  │
│ │ traffic  │          │       │              │ outbound     │  │
│ └──────────┴──────────┴───────┴──────────────┴──────────────┘  │
│                                                                   │
│ Tags:                                                            │
│   Environment: production                                        │
│   Name: web-server-sg                                           │
│                                                                   │
│ [Create security group]                                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### All Rule Fields Explained

```
┌─────────────────────────────────────────────────────────────────────┐
│              SECURITY GROUP RULE FIELDS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ TYPE (preset shortcuts):                                             │
│ ├── SSH (TCP 22)                                                    │
│ ├── HTTP (TCP 80)                                                   │
│ ├── HTTPS (TCP 443)                                                 │
│ ├── RDP (TCP 3389)                                                  │
│ ├── MySQL/Aurora (TCP 3306)                                         │
│ ├── PostgreSQL (TCP 5432)                                           │
│ ├── MSSQL (TCP 1433)                                                │
│ ├── MongoDB (TCP 27017)                                             │
│ ├── Redis (TCP 6379)                                                │
│ ├── NFS (TCP 2049)                                                  │
│ ├── DNS (UDP 53)                                                    │
│ ├── All traffic (All protocols, all ports)                         │
│ ├── All TCP (TCP 0-65535)                                           │
│ ├── All UDP (UDP 0-65535)                                           │
│ ├── All ICMP - IPv4 (ping)                                         │
│ └── Custom TCP/UDP (you specify port)                              │
│                                                                       │
│ PROTOCOL: TCP, UDP, ICMP, All                                       │
│                                                                       │
│ PORT RANGE:                                                          │
│ ├── Single port: 443                                                │
│ ├── Port range: 8080-8090                                          │
│ └── All: 0-65535                                                    │
│                                                                       │
│ SOURCE / DESTINATION:                                                │
│ ├── CIDR block: 10.0.0.0/16, 0.0.0.0/0 (anywhere IPv4)           │
│ ├── IPv6 CIDR: ::/0 (anywhere IPv6)                                │
│ ├── Security group: sg-xxxxxxxx (reference another SG!)            │
│ │   ⚡ This is the MOST POWERFUL feature!                           │
│ │   Instead of IP, reference a SG — any instance in that SG       │
│ │   is allowed. IPs can change, SG reference stays valid.          │
│ ├── Prefix list: pl-xxxxxxxx (managed prefix list — AWS IPs)      │
│ └── Self: sg-self (allow traffic within same SG)                   │
│                                                                       │
│ DESCRIPTION: Optional but HIGHLY recommended.                       │
│   "Allow HTTPS from ALB sg-12345"                                   │
│   "Allow SSH from VPN CIDR 192.168.1.0/24"                         │
│   Makes auditing much easier!                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Security Group References (The Power Feature)

```
┌─────────────────────────────────────────────────────────────────────┐
│           SG REFERENCING — THE KILLER FEATURE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instead of using IP addresses, reference another Security Group!    │
│                                                                       │
│ Example: 3-tier architecture                                         │
│                                                                       │
│ [ALB]                                                                │
│   SG: alb-sg                                                        │
│   Inbound: TCP 443 from 0.0.0.0/0                                  │
│      │                                                               │
│      ▼                                                               │
│ [App Server EC2]                                                     │
│   SG: app-sg                                                        │
│   Inbound: TCP 8080 from sg-alb-sg ← References ALB's SG!        │
│      │                                                               │
│      ▼                                                               │
│ [RDS Database]                                                       │
│   SG: db-sg                                                         │
│   Inbound: TCP 3306 from sg-app-sg ← References App's SG!         │
│                                                                       │
│ Benefits:                                                            │
│ ├── No need to track individual IPs                                │
│ ├── Auto Scaling adds/removes instances — SG ref still works!     │
│ ├── IP changes don't break anything                                │
│ ├── Clear intent: "app servers can reach database"                │
│ └── More secure than CIDR-based rules                              │
│                                                                       │
│ Cross-VPC SG reference (same region, peered VPCs):                  │
│ ├── Use SG ID from peered VPC as source                            │
│ ├── Only works for peered VPCs in SAME region                     │
│ └── For cross-region: Must use CIDR blocks                         │
│                                                                       │
│ Self-referencing:                                                    │
│ ├── Source = sg-itself                                              │
│ ├── All instances in this SG can talk to each other                │
│ └── Useful for: clusters (Redis, Kafka, Elasticsearch)             │
│     Inbound: All TCP from sg-redis-cluster-sg (self)               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Managed Prefix Lists

```
┌─────────────────────────────────────────────────────────────────────┐
│              MANAGED PREFIX LISTS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A set of CIDR blocks that you can reference in SG rules       │
│       and route tables. Managed centrally, used everywhere.         │
│                                                                       │
│ Types:                                                               │
│ ├── AWS-managed: AWS provides prefix lists for their services       │
│ │   com.amazonaws.region.s3 → pl-xxxxxxx (S3 IP ranges)           │
│ │   com.amazonaws.region.dynamodb → pl-yyyyyyy                     │
│ │                                                                    │
│ └── Customer-managed: You create your own                           │
│     Example: "office-ips" prefix list                               │
│     ├── 203.0.113.0/24 (Mumbai office)                             │
│     ├── 198.51.100.0/24 (Bangalore office)                         │
│     └── 192.0.2.0/24 (Delhi office)                                │
│                                                                       │
│ Use case: Multiple SGs need same set of IPs                         │
│   Instead of adding 3 IPs to every SG rule:                        │
│   Just reference pl-office-ips in the rule!                         │
│   Update the prefix list → all SGs auto-updated                    │
│                                                                       │
│ Create:                                                              │
│ aws ec2 create-managed-prefix-list \                                 │
│   --prefix-list-name office-ips \                                    │
│   --max-entries 10 \                                                  │
│   --address-family IPv4 \                                             │
│   --entries Cidr=203.0.113.0/24,Description="Mumbai" \               │
│             Cidr=198.51.100.0/24,Description="Bangalore"            │
│                                                                       │
│ Use in SG:                                                           │
│ aws ec2 authorize-security-group-ingress \                           │
│   --group-id sg-xxxxx \                                               │
│   --ip-permissions IpProtocol=tcp,FromPort=22,ToPort=22,\           │
│     PrefixListIds=[{PrefixListId=pl-xxxxx}]                        │
│                                                                       │
│ Can share via RAM (Resource Access Manager) across accounts!        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Security Group Limits

```
┌──────────────────────────────────────────┬───────────────────────┐
│ Limit                                    │ Default (adjustable)  │
├──────────────────────────────────────────┼───────────────────────┤
│ SGs per VPC                              │ 2,500                 │
│ Inbound rules per SG                     │ 60                    │
│ Outbound rules per SG                    │ 60                    │
│ SGs per ENI (network interface)          │ 5                     │
│ ENIs per instance                        │ Varies by type        │
│ Rules per SG × SGs per ENI ≤ 1,000       │ 60 × 5 = 300 max     │
└──────────────────────────────────────────┴───────────────────────┘

⚠️ If you increase rules per SG, AWS may decrease SGs per ENI:
   e.g., 120 rules × 3 SGs = 360 (stays under 1,000 total)

💡 Each SG reference counts as 1 rule, even though it may
   represent hundreds of IPs — use SG refs to save rule count!
```

### CLI Examples

```bash
# Create Security Group
aws ec2 create-security-group \
  --group-name web-server-sg \
  --description "Web server - HTTP/HTTPS public, SSH from bastion" \
  --vpc-id vpc-prod \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=web-server-sg}]'

# Add inbound rule (HTTPS from anywhere)
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Add inbound rule (SSH from another SG)
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 22 \
  --source-group sg-bastion

# Add inbound rule (app port from ALB SG)
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --ip-permissions 'IpProtocol=tcp,FromPort=8080,ToPort=8080,
    UserIdGroupPairs=[{GroupId=sg-alb,Description="Allow from ALB"}]'

# Remove a rule
aws ec2 revoke-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# View rules
aws ec2 describe-security-groups \
  --group-ids sg-xxxxx \
  --query 'SecurityGroups[0].IpPermissions'

# Find unused Security Groups
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?length(IpPermissions)==`0` && length(IpPermissionsEgress)==`1`].{ID:GroupId,Name:GroupName}'
```

### Terraform Example

```hcl
# Web server SG
resource "aws_security_group" "web" {
  name        = "web-server-sg"
  description = "Web server SG"
  vpc_id      = aws_vpc.prod.id

  tags = {
    Name = "web-server-sg"
  }
}

resource "aws_security_group_rule" "web_https_in" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web.id
  description       = "HTTPS from internet"
}

resource "aws_security_group_rule" "web_ssh_from_bastion" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion.id
  security_group_id        = aws_security_group.web.id
  description              = "SSH from bastion SG"
}

resource "aws_security_group_rule" "web_all_out" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web.id
  description       = "Allow all outbound"
}

# Database SG (only from app SG)
resource "aws_security_group" "db" {
  name        = "database-sg"
  description = "Database SG - only from app tier"
  vpc_id      = aws_vpc.prod.id
}

resource "aws_security_group_rule" "db_mysql_from_app" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.app.id
  security_group_id        = aws_security_group.db.id
  description              = "MySQL from app servers only"
}
```

---

## Part 2: Network ACLs (NACLs)

### What and How They Work

```
┌─────────────────────────────────────────────────────────────────────┐
│                   NETWORK ACLs (NACLs)                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A firewall at the SUBNET level that controls traffic          │
│       in and out of subnets.                                        │
│                                                                       │
│ Key characteristics:                                                 │
│ ┌─────────────────────────────────────────────────────────────┐     │
│ │ 1. STATELESS                                                  │     │
│ │    You MUST create rules for BOTH directions!                 │     │
│ │    If you allow inbound TCP 443, you must also allow         │     │
│ │    outbound on ephemeral ports for the response.             │     │
│ │                                                                │     │
│ │ 2. ALLOW and DENY rules                                       │     │
│ │    Unlike SGs, NACLs can explicitly DENY traffic.            │     │
│ │    Useful for blocking specific IPs.                          │     │
│ │                                                                │     │
│ │ 3. RULES EVALUATED IN ORDER (by rule number)                  │     │
│ │    Lowest number evaluated first.                             │     │
│ │    First match wins — stops evaluating.                       │     │
│ │    Rule 100 (ALLOW) checked before Rule 200 (DENY).          │     │
│ │    Final rule * (asterisk) = DENY ALL (default).             │     │
│ │                                                                │     │
│ │ 4. APPLIED TO SUBNET (not instance)                           │     │
│ │    All traffic entering/leaving the subnet passes through.   │     │
│ │    One NACL per subnet (1:1 mapping).                         │     │
│ │    One NACL can be associated with multiple subnets.         │     │
│ │                                                                │     │
│ │ 5. PROCESSED BEFORE Security Groups                           │     │
│ │    Inbound: NACL → SG                                        │     │
│ │    Outbound: SG → NACL                                       │     │
│ └─────────────────────────────────────────────────────────────┘     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Default NACL

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DEFAULT NACL                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Every VPC comes with a DEFAULT NACL.                                 │
│ New subnets are auto-associated with the default NACL.              │
│                                                                       │
│ Default NACL INBOUND rules:                                          │
│ ┌──────┬──────┬──────────┬──────┬──────────────┬────────┐          │
│ │ Rule │ Type │ Protocol │ Port │ Source       │ Action │          │
│ ├──────┼──────┼──────────┼──────┼──────────────┼────────┤          │
│ │ 100  │ All  │ All      │ All  │ 0.0.0.0/0   │ ALLOW  │          │
│ │ *    │ All  │ All      │ All  │ 0.0.0.0/0   │ DENY   │          │
│ └──────┴──────┴──────────┴──────┴──────────────┴────────┘          │
│                                                                       │
│ Default NACL OUTBOUND rules:                                         │
│ ┌──────┬──────┬──────────┬──────┬──────────────┬────────┐          │
│ │ Rule │ Type │ Protocol │ Port │ Destination  │ Action │          │
│ ├──────┼──────┼──────────┼──────┼──────────────┼────────┤          │
│ │ 100  │ All  │ All      │ All  │ 0.0.0.0/0   │ ALLOW  │          │
│ │ *    │ All  │ All      │ All  │ 0.0.0.0/0   │ DENY   │          │
│ └──────┴──────┴──────────┴──────┴──────────────┴────────┘          │
│                                                                       │
│ Default NACL = ALLOW ALL (both directions)                           │
│ ⚠️ This is fine for most cases — SGs are your primary defense      │
│                                                                       │
│ Custom NACLs start with DENY ALL (only * rule)                      │
│ You must explicitly add ALLOW rules!                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Custom NACL

```
Console → VPC → Network ACLs → Create network ACL

┌─────────────────────────────────────────────────────────────────┐
│              CREATE NETWORK ACL                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [prod-public-nacl]                                        │
│ VPC:  [prod-vpc ▼]                                              │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
│ Then edit inbound rules:                                        │
│ ┌──────┬──────────┬──────────┬──────┬──────────────┬────────┐  │
│ │ Rule │ Type     │ Protocol │ Port │ Source       │ Action │  │
│ ├──────┼──────────┼──────────┼──────┼──────────────┼────────┤  │
│ │ 100  │ HTTP     │ TCP      │ 80   │ 0.0.0.0/0   │ ALLOW  │  │
│ │ 110  │ HTTPS    │ TCP      │ 443  │ 0.0.0.0/0   │ ALLOW  │  │
│ │ 120  │ SSH      │ TCP      │ 22   │ 203.0.113/24│ ALLOW  │  │
│ │ 130  │ Custom   │ TCP      │ 1024-│ 0.0.0.0/0   │ ALLOW  │  │
│ │      │ TCP      │          │ 65535│             │        │  │
│ │      │          │          │      │ (ephemeral  │        │  │
│ │      │          │          │      │  ports for  │        │  │
│ │      │          │          │      │  responses) │        │  │
│ │ 900  │ All      │ All      │ All  │ 5.6.7.8/32  │ DENY   │  │
│ │      │ traffic  │          │      │ (block bad  │        │  │
│ │      │          │          │      │  actor)     │        │  │
│ │ *    │ All      │ All      │ All  │ 0.0.0.0/0   │ DENY   │  │
│ └──────┴──────────┴──────────┴──────┴──────────────┴────────┘  │
│                                                                   │
│ Then edit outbound rules:                                       │
│ ┌──────┬──────────┬──────────┬──────┬──────────────┬────────┐  │
│ │ Rule │ Type     │ Protocol │ Port │ Destination  │ Action │  │
│ ├──────┼──────────┼──────────┼──────┼──────────────┼────────┤  │
│ │ 100  │ HTTP     │ TCP      │ 80   │ 0.0.0.0/0   │ ALLOW  │  │
│ │ 110  │ HTTPS    │ TCP      │ 443  │ 0.0.0.0/0   │ ALLOW  │  │
│ │ 120  │ Custom   │ TCP      │ 1024-│ 0.0.0.0/0   │ ALLOW  │  │
│ │      │ TCP      │          │ 65535│             │        │  │
│ │ *    │ All      │ All      │ All  │ 0.0.0.0/0   │ DENY   │  │
│ └──────┴──────────┴──────────┴──────┴──────────────┴────────┘  │
│                                                                   │
│ Associate with subnet:                                           │
│ Network ACL → Subnet associations → Edit                        │
│ ☑ prod-public-a                                                 │
│ ☑ prod-public-b                                                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### NACL Rule Numbering Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│          NACL RULE NUMBERING                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Rules are evaluated from LOWEST number to HIGHEST.                   │
│ First match wins — processing stops.                                 │
│                                                                       │
│ ⚡ Number in increments of 10 or 100                                 │
│    This leaves room to insert rules later!                          │
│                                                                       │
│ Good numbering:                                                      │
│ 100: Allow HTTPS inbound ← main traffic                            │
│ 110: Allow HTTP inbound                                              │
│ 120: Allow SSH from office                                           │
│ 130: Allow ephemeral ports (for responses)                          │
│ 900: DENY specific bad IP ← block rules near end                   │
│ *:   DENY ALL (default, cannot be changed)                          │
│                                                                       │
│ Example of ordering that matters:                                    │
│                                                                       │
│ Rule 50: DENY 10.0.5.0/24 (block suspicious subnet)               │
│ Rule 100: ALLOW 10.0.0.0/16 (allow VPC traffic)                   │
│ → 10.0.5.x is denied because rule 50 matches first!               │
│                                                                       │
│ If reversed:                                                         │
│ Rule 50: ALLOW 10.0.0.0/16                                          │
│ Rule 100: DENY 10.0.5.0/24                                          │
│ → 10.0.5.x is ALLOWED because rule 50 matches first! ⚠️            │
│                                                                       │
│ ⚡ Put DENY rules BEFORE ALLOW rules if you need to block           │
│    specific IPs within an allowed range.                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Ephemeral Ports (Critical for NACLs!)

```
┌─────────────────────────────────────────────────────────────────────┐
│              EPHEMERAL PORTS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ When a client initiates a connection, the RESPONSE comes back       │
│ on a random high port (ephemeral/temporary port).                   │
│                                                                       │
│ Client (port 49152) ──TCP 443──► Server                             │
│ Client (port 49152) ◄──response──  Server                           │
│                                                                       │
│ Ephemeral port ranges by OS:                                         │
│ ├── Linux: 32768 - 60999                                           │
│ ├── Windows: 49152 - 65535                                          │
│ ├── NAT Gateway: 1024 - 65535                                      │
│ └── ELB: 1024 - 65535                                               │
│                                                                       │
│ ⚡ For NACLs: Allow 1024-65535 to cover ALL ephemeral ports         │
│                                                                       │
│ With Security Groups you don't worry about this (STATEFUL).         │
│ With NACLs you MUST allow ephemeral ports (STATELESS)!              │
│                                                                       │
│ Example: Web server in public subnet                                 │
│ Inbound NACL: Allow TCP 443 from 0.0.0.0/0                         │
│ Outbound NACL: Allow TCP 1024-65535 to 0.0.0.0/0 ← REQUIRED!     │
│                (for the response to reach the client)               │
│                                                                       │
│ Forgetting ephemeral ports is the #1 NACL mistake!                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Security Groups vs NACLs

```
┌───────────────────────────────┬──────────────────┬──────────────────┐
│ Feature                       │ Security Group   │ Network ACL      │
├───────────────────────────────┼──────────────────┼──────────────────┤
│ Level                         │ Instance (ENI)   │ Subnet           │
│ State                         │ Stateful         │ Stateless        │
│ Rules                         │ Allow only       │ Allow and Deny   │
│ Rule evaluation               │ All rules checked│ In order (first  │
│                               │ (any match=allow)│ match wins)      │
│ Default inbound               │ Deny all         │ Allow all (dflt) │
│ Default outbound              │ Allow all        │ Allow all (dflt) │
│ Ephemeral ports               │ Not needed       │ Must configure   │
│ Applied to                    │ Specific instance│ All instances in  │
│                               │ (opt-in)         │ subnet (auto)    │
│ SG reference as source        │ Yes              │ No (CIDR only)   │
│ Use case                      │ Primary defense  │ Extra layer /    │
│                               │                  │ IP blocking      │
│ Changes take effect           │ Immediately      │ Immediately      │
│ Number per VPC                │ 2,500            │ 200              │
│ Rules per                     │ 60 in + 60 out   │ 20 in + 20 out   │
│                               │                  │ (adjustable)     │
└───────────────────────────────┴──────────────────┴──────────────────┘

When to use what:

Security Groups (ALWAYS use):
├── Primary line of defense
├── Control access per service/application
├── Reference other SGs for dynamic IPs
└── 99% of your firewall rules go here

NACLs (use when needed):
├── Block specific IPs/CIDRs (e.g., known attackers)
├── Compliance requirements (extra layer)
├── Subnet-wide deny rules
└── Defense-in-depth requirement

💡 Most teams: Use SGs extensively + keep default NACLs (allow all).
   Add custom NACLs only for specific deny requirements.
```

### Defense-in-Depth Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│              DEFENSE IN DEPTH                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Internet                                                             │
│    │                                                                  │
│    ▼                                                                  │
│ [AWS Shield / WAF]          ← Layer 7: DDoS, SQL injection, XSS    │
│    │                                                                  │
│    ▼                                                                  │
│ [Route Table]               ← Layer 3: Can this subnet be reached?  │
│    │                                                                  │
│    ▼                                                                  │
│ [NACL - Subnet]             ← Layer 4: Block bad IPs / restrict     │
│    │                           ports at subnet boundary              │
│    ▼                                                                  │
│ [Security Group - Instance] ← Layer 4: Allow specific sources/ports │
│    │                           to specific instances                 │
│    ▼                                                                  │
│ [OS Firewall (iptables)]    ← OS Level: Additional host protection  │
│    │                                                                  │
│    ▼                                                                  │
│ [Application]               ← App Level: Auth, authorization        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Common Security Group Patterns

### 3-Tier Web Application

```
┌─────────────────────────────────────────────────────────────────────┐
│         3-TIER APPLICATION SG PATTERN                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ SG: alb-sg                                                           │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Inbound: TCP 443 from 0.0.0.0/0                             │    │
│ │ Inbound: TCP 80 from 0.0.0.0/0 (redirect to HTTPS)         │    │
│ │ Outbound: All to 0.0.0.0/0                                  │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                        │                                              │
│                        ▼                                              │
│ SG: app-sg                                                           │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Inbound: TCP 8080 from sg-alb-sg                             │    │
│ │ Inbound: TCP 22 from sg-bastion-sg                           │    │
│ │ Outbound: All to 0.0.0.0/0                                  │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                        │                                              │
│                        ▼                                              │
│ SG: db-sg                                                            │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Inbound: TCP 5432 from sg-app-sg                             │    │
│ │ Inbound: TCP 5432 from sg-bastion-sg (for DBA access)       │    │
│ │ Outbound: All to 0.0.0.0/0                                  │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ SG: bastion-sg                                                       │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Inbound: TCP 22 from pl-office-ips (prefix list)            │    │
│ │ Outbound: All to 0.0.0.0/0                                  │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ SG: cache-sg (Redis/ElastiCache)                                    │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Inbound: TCP 6379 from sg-app-sg                             │    │
│ │ Outbound: All to 0.0.0.0/0                                  │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### ECS/EKS Container Workloads

```
SG: ecs-task-sg
┌──────────────────────────────────────────────────────────────┐
│ Inbound: TCP 8080 from sg-alb-sg (ALB health checks + traffic)│
│ Outbound: TCP 443 to 0.0.0.0/0 (AWS APIs, external services) │
│ Outbound: TCP 5432 to sg-db-sg (database)                     │
│ Outbound: TCP 6379 to sg-cache-sg (Redis)                     │
└──────────────────────────────────────────────────────────────┘

SG: vpc-endpoint-sg (for ECR, CloudWatch, etc.)
┌──────────────────────────────────────────────────────────────┐
│ Inbound: TCP 443 from 10.0.0.0/16 (VPC CIDR)               │
│ Outbound: All to 0.0.0.0/0                                   │
└──────────────────────────────────────────────────────────────┘
```

### Restricting Outbound (Hardened)

```
⚠️ Default: Allow ALL outbound — this is convenient but less secure.

For security-sensitive workloads, restrict outbound:

SG: hardened-app-sg
┌──────────────────────────────────────────────────────────────┐
│ Inbound: TCP 8080 from sg-alb-sg                             │
│                                                                │
│ Outbound: TCP 5432 to sg-db-sg (only database)               │
│ Outbound: TCP 6379 to sg-cache-sg (only Redis)               │
│ Outbound: TCP 443 to pl-s3 (S3 prefix list)                  │
│ Outbound: TCP 443 to sg-vpc-endpoint (ECR, Logs, etc.)      │
│ NO 0.0.0.0/0 outbound! ← blocks unintended internet access  │
└──────────────────────────────────────────────────────────────┘

When to restrict outbound:
├── PCI-DSS / HIPAA compliance
├── Handling sensitive data (PII, financial)
├── Preventing data exfiltration
└── Zero-trust architecture
```

---

## Part 5: Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│              SECURITY GROUP BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. One SG per role/function                                          │
│    ├── web-sg, app-sg, db-sg, bastion-sg, cache-sg                 │
│    └── NOT: "allow-everything-sg"                                   │
│                                                                       │
│ 2. Reference SGs, not IPs                                            │
│    ├── Source: sg-app-sg (not 10.0.2.0/24)                         │
│    └── Works with Auto Scaling, IP changes                         │
│                                                                       │
│ 3. Never use 0.0.0.0/0 for SSH/RDP                                  │
│    ├── Use bastion SG reference                                     │
│    ├── Or office IP prefix list                                     │
│    └── Or use SSM Session Manager (no SSH needed!)                 │
│                                                                       │
│ 4. Use descriptive names and descriptions                           │
│    ├── Rule description: "HTTPS from ALB sg-12345"                 │
│    └── SG name: "prod-app-server-sg"                               │
│                                                                       │
│ 5. Audit regularly                                                   │
│    ├── AWS Config: Check for overly permissive rules               │
│    ├── Security Hub: Automated compliance checks                   │
│    └── Remove unused SGs                                            │
│                                                                       │
│ 6. Use prefix lists for repeated IP sets                            │
│    ├── Office IPs, partner IPs, VPN CIDRs                         │
│    └── Update once → all SG rules updated                          │
│                                                                       │
│ 7. Restrict outbound for sensitive workloads                        │
│    └── Only allow specific destinations                            │
│                                                                       │
│ 8. Tag everything                                                    │
│    ├── Environment: production/staging/dev                         │
│    ├── Application: api/web/worker                                 │
│    └── Team: backend/frontend/devops                               │
│                                                                       │
│ 9. Keep default NACL unless you need specific deny rules            │
│    └── Don't over-complicate with custom NACLs                     │
│                                                                       │
│ 10. Use VPC Flow Logs to verify rules work as intended              │
│     └── Check for unexpected ACCEPT or REJECT entries              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Real-World Patterns

### Startup (5-10 devs)

```
SGs: 5-6 security groups
├── alb-sg: HTTPS from anywhere
├── app-sg: 8080 from alb-sg, SSH from bastion-sg
├── db-sg: 5432 from app-sg
├── bastion-sg: SSH from office IPs
└── cache-sg: 6379 from app-sg

NACLs: Default (allow all) — SGs do the work
VPC Flow Logs: Enabled on VPC level
```

### Mid-Size (50-100 devs)

```
SGs: 15-20 security groups (per service/function)
├── Per environment: prod-app-sg, staging-app-sg, dev-app-sg
├── Per service: api-sg, worker-sg, scheduler-sg
├── Shared: bastion-sg, vpn-sg, monitoring-sg
├── Data: rds-sg, elasticache-sg, elasticsearch-sg
└── Infra: vpc-endpoint-sg, efs-sg

NACLs: Custom for public subnets (restrict SSH to office)
Prefix lists: Office IPs, VPN CIDRs
AWS Config: Rules to check for 0.0.0.0/0 SSH
```

### Enterprise (500+ devs)

```
SGs: 100+ (managed via Terraform modules)
├── Centralized SG module with standard patterns
├── Auto-tagging via Terraform
├── SG changes via PR review
├── Cross-account SG references (via peering)
└── Prefix lists shared via RAM

NACLs: Custom per subnet tier
├── Public NACL: Allow HTTP/HTTPS, deny known bad IPs
├── App NACL: Allow from ALB subnets, deny internet
├── Data NACL: Allow from app subnets only

AWS Config: Automated remediation
Security Hub: Continuous compliance
GuardDuty: Threat detection on flow logs
VPC Flow Logs: All VPCs → S3 → Athena for analysis
```

---

## Troubleshooting: Common Security Group & NACL Issues

### "My app can't connect to port 443/80"

```
Checklist:
1. ☐ Security Group allows inbound on port 443 from 0.0.0.0/0?
2. ☐ If using NACL: Inbound rule allows port 443?
3. ☐ If using NACL: Outbound rule allows ephemeral ports 1024-65535?
   (NACLs are stateless — response traffic needs explicit allow)
4. ☐ Is the security group attached to the correct ENI/instance?
```

### What Are Ephemeral Ports?

When a client connects to your server on port 443, the server responds on a **random high port** (1024-65535) called an ephemeral port. Because NACLs are **stateless**, you must explicitly allow outbound traffic on these ports — otherwise responses are blocked even though inbound is allowed.

```
Client (port 52431) ───── port 443 ───→ Server     ← NACL inbound: allow 443 ✔️
Client (port 52431) ─── port 52431 ─── Server     ← NACL outbound: allow 1024-65535 ✔️
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Opening SSH (22) to 0.0.0.0/0 | Anyone can attempt to log in | Restrict to your IP or use a bastion SG |
| Using the default SG | Has permissive rules you may not realize | Create custom SGs for each tier |
| Forgetting ephemeral ports in NACLs | Responses blocked, connections timeout | Allow outbound 1024-65535 |
| Too many inline rules | Hard to manage, hitting limits | Use prefix lists or SG references |
| Not referencing SGs from other SGs | Fragile IP-based rules | Use `sg-xxxxx` as source instead of IPs |

---

## Quick Reference

| Feature | Security Group | NACL |
|---------|---------------|------|
| Level | Instance (ENI) | Subnet |
| Stateful | Yes | No |
| Deny rules | No | Yes |
| Evaluation | All rules | In order |
| Primary use | Per-service firewall | IP blocking, compliance |
| Cost | Free | Free |
| Limit | 2,500 per VPC | 200 per VPC |

---

## What's Next?

In the next chapter, we'll cover Route 53 — AWS's DNS service for domain registration, hosted zones, record types, and routing policies.

→ Next: [Chapter 9: Route 53 - DNS Management](09-route53.md)

---

*Last Updated: May 2026*
