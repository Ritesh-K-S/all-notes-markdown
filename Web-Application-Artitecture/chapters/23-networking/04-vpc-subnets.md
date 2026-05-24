# VPC, Subnets & Network Security Groups

> **What you'll learn**: How cloud providers create isolated virtual networks (VPCs), how subnets divide those networks into public and private zones, and how security groups and NACLs control traffic flow — the foundation of secure cloud architecture.

---

## Real-Life Analogy

Imagine you're building a **secure corporate campus**:

**VPC = The Entire Campus**
- You own the land (IP address range)
- It's fenced off from the outside world (isolated from other customers)
- You decide what buildings go where

**Subnets = Buildings within the Campus**
- **Public Subnet** = The lobby/reception building — visitors (internet traffic) can enter
- **Private Subnet** = The R&D lab — only employees with badges can enter, no public access
- Each building has its own address range (sub-range of IPs)

**Security Groups = Badge Readers on Each Door**
- "Only people with engineering badges can enter the server room"
- "Only the reception building can send visitors to the lab"
- Rules attached to individual rooms (instances)

**Network ACLs = Security Checkpoints at Building Entrances**
- "No one from this banned list can enter Building B"
- "Everyone in Building A can leave freely"
- Rules applied at the building (subnet) level

```
┌─────────────────────────── VPC (10.0.0.0/16) ───────────────────────────┐
│                                                                          │
│  ┌─── Public Subnet (10.0.1.0/24) ───┐  ┌─── Private Subnet (10.0.2.0/24) ──┐│
│  │                                     │  │                                    ││
│  │  ┌─────────┐    ┌─────────┐        │  │  ┌─────────┐    ┌─────────┐       ││
│  │  │ Web     │    │ NAT     │        │  │  │ App     │    │ DB      │       ││
│  │  │ Server  │    │ Gateway │        │  │  │ Server  │    │ Server  │       ││
│  │  └─────────┘    └─────────┘        │  │  └─────────┘    └─────────┘       ││
│  │       ▲                             │  │       ▲              ▲             ││
│  │       │ Internet                    │  │       │ (no direct   │             ││
│  │       │ Gateway                     │  │       │  internet)   │             ││
│  └───────┼─────────────────────────────┘  └───────┼──────────────┼─────────────┘│
│           │                                        │              │              │
│    ┌──────┴──────┐                          ┌─────┴─────┐       │              │
│    │   Internet  │                          │  Only via  │       │              │
│    │   Gateway   │                          │  NAT GW or │       │              │
│    └─────────────┘                          │  internal  │       │              │
│                                             └────────────┘       │              │
└──────────────────────────────────────────────────────────────────┘              │
                                                                                   │
```

---

## Core Concept Explained Step-by-Step

### Step 1: What is a VPC?

A **Virtual Private Cloud (VPC)** is your own private, isolated section of a cloud provider's network.

Think of it as getting your own private "chunk" of the internet within AWS, GCP, or Azure.

```
Cloud Provider's Network (e.g., AWS):
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ┌── Customer A's VPC ──┐  ┌── Customer B's VPC ──┐    │
│  │  10.0.0.0/16         │  │  10.0.0.0/16         │    │
│  │  (completely          │  │  (completely          │    │
│  │   isolated)           │  │   isolated)           │    │
│  └──────────────────────┘  └──────────────────────┘    │
│                                                          │
│  ┌── Customer C's VPC ──┐  ┌── Your VPC ───────────┐   │
│  │  172.16.0.0/16       │  │  10.0.0.0/16          │   │
│  │                       │  │  ← You control this!  │   │
│  └──────────────────────┘  └────────────────────────┘   │
│                                                          │
│  Each VPC is isolated — like separate private networks   │
│  They CANNOT see each other (unless you explicitly       │
│  connect them via peering/transit gateway)               │
└──────────────────────────────────────────────────────────┘
```

**Key VPC properties:**
- **CIDR block** — Your IP address range (e.g., 10.0.0.0/16 = 65,536 IPs)
- **Region-bound** — A VPC exists within one cloud region
- **Isolated** — No traffic flows between VPCs by default
- **You control everything** — Subnets, routing, security rules

### Step 2: CIDR Notation Explained

**CIDR** (Classless Inter-Domain Routing) defines IP address ranges:

```
10.0.0.0/16 means:
  • Network: 10.0.x.x
  • The first 16 bits are fixed (10.0)
  • The remaining 16 bits can vary
  • Total IPs: 2^16 = 65,536 addresses

10.0.1.0/24 means:
  • Network: 10.0.1.x
  • The first 24 bits are fixed (10.0.1)
  • The remaining 8 bits can vary
  • Total IPs: 2^8 = 256 addresses

Common CIDR blocks:
  /16 = 65,536 IPs (typical VPC)
  /20 = 4,096 IPs (large subnet)
  /24 = 256 IPs (standard subnet)
  /28 = 16 IPs (tiny subnet)
  /32 = 1 IP (single host)
```

### Step 3: What are Subnets?

A **subnet** is a smaller network within your VPC. You divide your VPC's IP range into multiple subnets.

```
VPC: 10.0.0.0/16 (65,536 IPs)
├── Public Subnet A: 10.0.1.0/24 (256 IPs) — Availability Zone 1
├── Public Subnet B: 10.0.2.0/24 (256 IPs) — Availability Zone 2
├── Private Subnet A: 10.0.10.0/24 (256 IPs) — Availability Zone 1
├── Private Subnet B: 10.0.11.0/24 (256 IPs) — Availability Zone 2
├── Database Subnet A: 10.0.20.0/24 (256 IPs) — Availability Zone 1
└── Database Subnet B: 10.0.21.0/24 (256 IPs) — Availability Zone 2
```

**Public Subnet:**
- Has a route to the **Internet Gateway**
- Instances can have public IPs
- Accessible from the internet (if security groups allow)
- Examples: Load balancers, bastion hosts, NAT gateways

**Private Subnet:**
- NO route to the Internet Gateway
- Instances have ONLY private IPs
- Cannot be reached from the internet directly
- Can reach internet via NAT Gateway (for updates, etc.)
- Examples: Application servers, databases, internal services

### Step 4: Internet Gateway vs NAT Gateway

```
┌─────────────────── VPC ────────────────────────────┐
│                                                     │
│  Public Subnet                Private Subnet        │
│  ┌──────────────┐            ┌──────────────┐      │
│  │ Web Server   │            │ App Server   │      │
│  │ 10.0.1.5     │            │ 10.0.10.5    │      │
│  │ (public IP:  │            │ (NO public   │      │
│  │  54.23.1.100)│            │  IP!)        │      │
│  └──────┬───────┘            └──────┬───────┘      │
│         │                           │               │
│         │ Route: 0.0.0.0/0          │ Route: 0.0.0.0/0
│         │ → Internet GW             │ → NAT Gateway │
│         │                           │               │
│  ┌──────▼───────┐            ┌──────▼───────┐      │
│  │Internet      │            │NAT Gateway   │      │
│  │Gateway (IGW) │            │(in public    │      │
│  │              │            │ subnet)      │      │
│  └──────┬───────┘            └──────┬───────┘      │
│         │                           │               │
└─────────┼───────────────────────────┼───────────────┘
          │                           │
          ▼                           ▼
    ┌──────────┐               ┌──────────┐
    │ Internet │               │ Internet │
    │(inbound  │               │(outbound │
    │+ outbound)│              │ ONLY)    │
    └──────────┘               └──────────┘

Internet Gateway: TWO-WAY traffic (internet can reach your server)
NAT Gateway: ONE-WAY traffic (your server can reach internet, but internet CANNOT initiate connections to your private server)
```

### Step 5: Security Groups (Instance-Level Firewall)

A **Security Group** is a virtual firewall attached to individual instances (EC2, RDS, etc.).

```
Security Group Rules — STATEFUL (if inbound is allowed, outbound response is automatic)

┌─────────────────────────────────────────────────────────┐
│ Security Group: "web-server-sg"                         │
├─────────────────────────────────────────────────────────┤
│ INBOUND RULES:                                          │
│ ┌─────────┬──────────┬─────────────────┬──────────┐    │
│ │ Type    │ Protocol │ Port Range      │ Source   │    │
│ ├─────────┼──────────┼─────────────────┼──────────┤    │
│ │ HTTP    │ TCP      │ 80              │ 0.0.0.0/0│    │
│ │ HTTPS   │ TCP      │ 443             │ 0.0.0.0/0│    │
│ │ SSH     │ TCP      │ 22              │ 10.0.0.0/8│   │
│ └─────────┴──────────┴─────────────────┴──────────┘    │
│                                                         │
│ OUTBOUND RULES:                                         │
│ ┌─────────┬──────────┬─────────────────┬──────────┐    │
│ │ Type    │ Protocol │ Port Range      │ Dest     │    │
│ ├─────────┼──────────┼─────────────────┼──────────┤    │
│ │ All     │ All      │ All             │ 0.0.0.0/0│    │
│ └─────────┴──────────┴─────────────────┴──────────┘    │
└─────────────────────────────────────────────────────────┘

Key properties:
• STATEFUL — if inbound traffic is allowed, response traffic is automatically allowed
• ALLOW rules only — no DENY rules (implicit deny for everything not allowed)
• Applied to ENI (Elastic Network Interface) level
• Can reference other security groups as source/destination
```

### Step 6: Network ACLs (Subnet-Level Firewall)

A **Network Access Control List (NACL)** is a firewall at the subnet boundary.

```
NACL Rules — STATELESS (must explicitly allow both inbound AND outbound)

┌─────────────────────────────────────────────────────────┐
│ NACL: "public-subnet-nacl"                              │
├─────────────────────────────────────────────────────────┤
│ INBOUND RULES (evaluated in order by rule number):      │
│ ┌──────┬──────────┬──────┬────────────┬────────┐       │
│ │ Rule │ Protocol │ Port │ Source     │ Action │       │
│ ├──────┼──────────┼──────┼────────────┼────────┤       │
│ │ 100  │ TCP      │ 80   │ 0.0.0.0/0 │ ALLOW  │       │
│ │ 110  │ TCP      │ 443  │ 0.0.0.0/0 │ ALLOW  │       │
│ │ 120  │ TCP      │ 22   │ 10.0.0.0/8│ ALLOW  │       │
│ │ *    │ All      │ All  │ 0.0.0.0/0 │ DENY   │       │
│ └──────┴──────────┴──────┴────────────┴────────┘       │
│                                                         │
│ OUTBOUND RULES:                                         │
│ ┌──────┬──────────┬────────────┬────────────┬────────┐ │
│ │ Rule │ Protocol │ Port       │ Dest       │ Action │ │
│ ├──────┼──────────┼────────────┼────────────┼────────┤ │
│ │ 100  │ TCP      │ 1024-65535 │ 0.0.0.0/0 │ ALLOW  │ │
│ │ 110  │ TCP      │ 443        │ 0.0.0.0/0 │ ALLOW  │ │
│ │ *    │ All      │ All        │ 0.0.0.0/0 │ DENY   │ │
│ └──────┴──────────┴────────────┴────────────┴────────┘ │
└─────────────────────────────────────────────────────────┘

Key properties:
• STATELESS — must allow traffic in BOTH directions explicitly
• ALLOW and DENY rules
• Rules processed in ORDER (lowest number first, first match wins)
• Applied at subnet boundary
• One NACL per subnet (but one NACL can be shared across subnets)
```

### Security Groups vs NACLs — Comparison

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ Security Group       │ Network ACL          │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Level            │ Instance (ENI)       │ Subnet               │
│ Stateful?        │ YES                  │ NO (stateless)       │
│ Rules            │ ALLOW only           │ ALLOW and DENY       │
│ Evaluation       │ All rules checked    │ Rules in order       │
│ Default          │ Deny all inbound     │ Allow all            │
│                  │ Allow all outbound   │ (default NACL)       │
│ Applied to       │ Specific instances   │ All instances in     │
│                  │                      │ the subnet           │
│ Use case         │ Fine-grained per-    │ Broad subnet-level   │
│                  │ instance control     │ blocking             │
└──────────────────┴──────────────────────┴──────────────────────┘
```

---

## How It Works Internally

### VPC Packet Flow — Complete Path

```
A request from the internet to your private app server:

Internet (Client: 203.0.113.50)
    │
    ▼
┌──────────────────┐
│ Internet Gateway │  ← Converts public IP → private IP (1:1 NAT)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Route Table      │  ← "10.0.1.0/24 → local"
│ (public subnet)  │     "0.0.0.0/0 → igw-xxx"
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Network ACL      │  ← Subnet-level firewall (STATELESS)
│ (public subnet)  │     Check: Is port 443 allowed inbound? YES
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Security Group   │  ← Instance-level firewall (STATEFUL)
│ (ALB)            │     Check: Is port 443 from 0.0.0.0/0 allowed? YES
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ ALB (10.0.1.50)  │  ← Application Load Balancer in public subnet
│                  │     Forwards to private subnet
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Route Table      │  ← "10.0.10.0/24 → local" (within VPC)
│ (between subnets)│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Network ACL      │  ← Private subnet NACL
│ (private subnet) │     Check: Is port 8080 from 10.0.1.0/24 allowed? YES
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Security Group   │  ← App server security group
│ (app-server-sg)  │     Check: Is port 8080 from "alb-sg" allowed? YES
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ App Server       │  ← Your application processes the request
│ (10.0.10.5)      │
└──────────────────┘
```

### VPC Peering and Transit Gateway

```
Connecting Multiple VPCs:

Option 1: VPC Peering (point-to-point)
┌──────────┐          ┌──────────┐
│  VPC A   │◀────────▶│  VPC B   │  Direct 1:1 connection
│10.0.0.0/16│          │10.1.0.0/16│  Not transitive!
└──────────┘          └──────────┘  (A→B doesn't mean A→C via B)

Option 2: Transit Gateway (hub-and-spoke)
                 ┌─────────────────┐
                 │ Transit Gateway  │
                 │ (central hub)    │
                 └────────┬────────┘
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  VPC A   │ │  VPC B   │ │  VPC C   │
        └──────────┘ └──────────┘ └──────────┘
        
  All VPCs can talk to each other through the transit gateway.
  Also connects to on-premise via VPN/Direct Connect.
```

---

## Code Examples

### Python — Create VPC with Boto3 (AWS SDK)

```python
# create_vpc.py — Set up a complete VPC with public and private subnets
import boto3

ec2 = boto3.resource('ec2')
ec2_client = boto3.client('ec2')

# Step 1: Create VPC
vpc = ec2.create_vpc(CidrBlock='10.0.0.0/16')
vpc.create_tags(Tags=[{'Key': 'Name', 'Value': 'production-vpc'}])
vpc.wait_until_available()
print(f"VPC created: {vpc.id}")

# Step 2: Create Internet Gateway (for public subnets)
igw = ec2.create_internet_gateway()
vpc.attach_internet_gateway(InternetGatewayId=igw.id)
print(f"Internet Gateway attached: {igw.id}")

# Step 3: Create Public Subnet (AZ-a)
public_subnet = ec2.create_subnet(
    VpcId=vpc.id,
    CidrBlock='10.0.1.0/24',
    AvailabilityZone='us-east-1a'
)
public_subnet.create_tags(Tags=[{'Key': 'Name', 'Value': 'public-subnet-1a'}])

# Enable auto-assign public IP for instances in public subnet
ec2_client.modify_subnet_attribute(
    SubnetId=public_subnet.id,
    MapPublicIpOnLaunch={'Value': True}
)

# Step 4: Create Private Subnet (AZ-a)
private_subnet = ec2.create_subnet(
    VpcId=vpc.id,
    CidrBlock='10.0.10.0/24',
    AvailabilityZone='us-east-1a'
)
private_subnet.create_tags(Tags=[{'Key': 'Name', 'Value': 'private-subnet-1a'}])

# Step 5: Create Route Table for public subnet → Internet Gateway
public_rt = vpc.create_route_table()
public_rt.create_route(DestinationCidrBlock='0.0.0.0/0', GatewayId=igw.id)
public_rt.associate_with_subnet(SubnetId=public_subnet.id)

# Step 6: Create NAT Gateway (for private subnet outbound)
eip = ec2_client.allocate_address(Domain='vpc')
nat_gw = ec2_client.create_nat_gateway(
    SubnetId=public_subnet.id,  # NAT GW goes in PUBLIC subnet
    AllocationId=eip['AllocationId']
)
print(f"NAT Gateway creating: {nat_gw['NatGateway']['NatGatewayId']}")

# Step 7: Create Route Table for private subnet → NAT Gateway
# (Wait for NAT GW to be available first in production)
private_rt = vpc.create_route_table()
private_rt.create_route(
    DestinationCidrBlock='0.0.0.0/0',
    NatGatewayId=nat_gw['NatGateway']['NatGatewayId']
)
private_rt.associate_with_subnet(SubnetId=private_subnet.id)

# Step 8: Create Security Groups
web_sg = ec2.create_security_group(
    GroupName='web-server-sg',
    Description='Allow HTTP/HTTPS from internet',
    VpcId=vpc.id
)
web_sg.authorize_ingress(
    IpPermissions=[
        {'IpProtocol': 'tcp', 'FromPort': 80, 'ToPort': 80,
         'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
        {'IpProtocol': 'tcp', 'FromPort': 443, 'ToPort': 443,
         'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
    ]
)

app_sg = ec2.create_security_group(
    GroupName='app-server-sg',
    Description='Allow traffic only from web servers',
    VpcId=vpc.id
)
app_sg.authorize_ingress(
    IpPermissions=[
        {'IpProtocol': 'tcp', 'FromPort': 8080, 'ToPort': 8080,
         'UserIdGroupPairs': [{'GroupId': web_sg.id}]},  # Only from web-sg!
    ]
)

print("VPC setup complete!")
print(f"  Public Subnet: {public_subnet.id}")
print(f"  Private Subnet: {private_subnet.id}")
print(f"  Web SG: {web_sg.id}")
print(f"  App SG: {app_sg.id}")
```

### Java — Security Group Management (AWS SDK v2)

```java
// VpcSecurityManager.java — Manage security groups programmatically
import software.amazon.awssdk.services.ec2.Ec2Client;
import software.amazon.awssdk.services.ec2.model.*;
import java.util.List;

public class VpcSecurityManager {

    private final Ec2Client ec2;

    public VpcSecurityManager() {
        this.ec2 = Ec2Client.builder().build();
    }

    // Create a security group with specific rules
    public String createAppSecurityGroup(String vpcId, String albSecurityGroupId) {
        // Create the security group
        CreateSecurityGroupResponse sgResponse = ec2.createSecurityGroup(
            CreateSecurityGroupRequest.builder()
                .vpcId(vpcId)
                .groupName("app-server-sg")
                .description("Allow traffic from ALB only")
                .build()
        );
        String sgId = sgResponse.groupId();

        // Add inbound rule: Allow port 8080 ONLY from ALB security group
        ec2.authorizeSecurityGroupIngress(
            AuthorizeSecurityGroupIngressRequest.builder()
                .groupId(sgId)
                .ipPermissions(List.of(
                    IpPermission.builder()
                        .ipProtocol("tcp")
                        .fromPort(8080)
                        .toPort(8080)
                        .userIdGroupPairs(List.of(
                            UserIdGroupPair.builder()
                                .groupId(albSecurityGroupId)  // Only from ALB!
                                .build()
                        ))
                        .build()
                ))
                .build()
        );

        System.out.println("Created SG: " + sgId);
        return sgId;
    }

    // Audit: Find security groups with 0.0.0.0/0 (too permissive)
    public void auditOpenSecurityGroups(String vpcId) {
        DescribeSecurityGroupsResponse response = ec2.describeSecurityGroups(
            DescribeSecurityGroupsRequest.builder()
                .filters(List.of(
                    Filter.builder().name("vpc-id").values(vpcId).build()
                ))
                .build()
        );

        for (SecurityGroup sg : response.securityGroups()) {
            for (IpPermission perm : sg.ipPermissions()) {
                for (IpRange range : perm.ipRanges()) {
                    if ("0.0.0.0/0".equals(range.cidrIp())) {
                        System.out.printf("WARNING: SG %s (%s) allows %s:%d-%d from 0.0.0.0/0%n",
                            sg.groupId(), sg.groupName(),
                            perm.ipProtocol(), perm.fromPort(), perm.toPort());
                    }
                }
            }
        }
    }
}
```

---

## Infrastructure Examples

### Terraform — Complete Production VPC

```hcl
# vpc.tf — Production-grade VPC with Terraform

# VPC
resource "aws_vpc" "production" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = { Name = "production-vpc" }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.production.id
  tags   = { Name = "production-igw" }
}

# Public Subnets (multi-AZ for high availability)
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.production.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = { Name = "public-subnet-${count.index + 1}" }
}

# Private Subnets (multi-AZ)
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.production.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = { Name = "private-subnet-${count.index + 1}" }
}

# Database Subnets (multi-AZ, extra isolation)
resource "aws_subnet" "database" {
  count             = 2
  vpc_id            = aws_vpc.production.id
  cidr_block        = "10.0.${count.index + 20}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = { Name = "database-subnet-${count.index + 1}" }
}

# NAT Gateway (one per AZ for HA)
resource "aws_eip" "nat" {
  count  = 2
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = 2
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = { Name = "nat-gw-${count.index + 1}" }
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.production.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "public-rt" }
}

resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.production.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = { Name = "private-rt-${count.index + 1}" }
}

# Security Groups
resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = aws_vpc.production.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Internet → ALB (HTTPS only)
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "app" {
  name   = "app-server-sg"
  vpc_id = aws_vpc.production.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Only from ALB!
  }
}

resource "aws_security_group" "database" {
  name   = "database-sg"
  vpc_id = aws_vpc.production.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # Only from app servers!
  }
}
```

### Kubernetes — Network Policies (VPC-Like Controls in K8s)

```yaml
# network-policy.yaml — Kubernetes equivalent of security groups
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-server-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend-api
  policyTypes:
    - Ingress
    - Egress
  
  # Inbound: Only allow from pods with label "role: frontend"
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 8080
  
  # Outbound: Only allow to database pods and external DNS
  egress:
    - to:
        - podSelector:
            matchLabels:
              role: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}  # Any namespace
      ports:
        - protocol: UDP
          port: 53  # DNS
```

### AWS VPC Architecture Diagram — Three-Tier Application

```
┌────────────────────────────── AWS Region (us-east-1) ──────────────────────────────┐
│                                                                                     │
│ ┌──────────────────────────── VPC (10.0.0.0/16) ──────────────────────────────┐    │
│ │                                                                              │    │
│ │  ┌─── AZ: us-east-1a ────────────────┐  ┌─── AZ: us-east-1b ────────────┐  │    │
│ │  │                                    │  │                                │  │    │
│ │  │  Public Subnet (10.0.1.0/24)      │  │  Public Subnet (10.0.2.0/24)  │  │    │
│ │  │  ┌─────┐  ┌─────────┐            │  │  ┌─────┐  ┌─────────┐        │  │    │
│ │  │  │ ALB │  │ NAT GW  │            │  │  │ ALB │  │ NAT GW  │        │  │    │
│ │  │  └──┬──┘  └────┬────┘            │  │  └──┬──┘  └────┬────┘        │  │    │
│ │  │     │           │                  │  │     │           │              │  │    │
│ │  │─────┼───────────┼──────────────────│──│─────┼───────────┼──────────────│  │    │
│ │  │     │           │                  │  │     │           │              │  │    │
│ │  │  Private Subnet (10.0.10.0/24)    │  │  Private Subnet (10.0.11.0/24)│  │    │
│ │  │  ┌─────────┐  ┌─────────┐        │  │  ┌─────────┐  ┌─────────┐    │  │    │
│ │  │  │ App Srv │  │ App Srv │        │  │  │ App Srv │  │ App Srv │    │  │    │
│ │  │  │  (EC2)  │  │  (EC2)  │        │  │  │  (EC2)  │  │  (EC2)  │    │  │    │
│ │  │  └────┬────┘  └────┬────┘        │  │  └────┬────┘  └────┬────┘    │  │    │
│ │  │       │             │              │  │       │             │          │  │    │
│ │  │───────┼─────────────┼──────────────│──│───────┼─────────────┼──────────│  │    │
│ │  │       │             │              │  │       │             │          │  │    │
│ │  │  Database Subnet (10.0.20.0/24)   │  │  Database Subnet (10.0.21.0/24)│ │    │
│ │  │  ┌──────────────────────┐         │  │  ┌──────────────────────┐     │  │    │
│ │  │  │ RDS Primary          │         │  │  │ RDS Standby          │     │  │    │
│ │  │  │ (PostgreSQL)         │◀───────synchronous──▶│ (Read Replica)      │     │  │    │
│ │  │  └──────────────────────┘         │  │  └──────────────────────┘     │  │    │
│ │  │                                    │  │                                │  │    │
│ │  └────────────────────────────────────┘  └────────────────────────────────┘  │    │
│ │                                                                              │    │
│ └──────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Netflix VPC Architecture

```
Netflix deploys across multiple AWS regions with interconnected VPCs:

┌──── us-east-1 ─────────────────────┐
│  ┌── VPC: Streaming ──────────────┐│
│  │ • Public: CDN Edge, ALB        ││
│  │ • Private: Encoding servers    ││
│  │ • Private: Content catalog DB  ││
│  └────────────────────────────────┘│
│  ┌── VPC: Auth/Billing ──────────┐ │
│  │ • Private: Auth microservices  │ │
│  │ • Private: Payment processing  │ │
│  │ • DB Subnet: User database     │ │
│  └────────────────────────────────┘ │
│         │ VPC Peering                │
│         ▼                            │
└──────────────────────────────────────┘
          │ Transit Gateway
          ▼
┌──── eu-west-1 ─────────────────────┐
│  (Same structure, replicated)       │
│  Serves European users              │
└──────────────────────────────────────┘

Key practices:
• Each microservice team gets their own VPC (blast radius isolation)
• Transit Gateway connects all VPCs in a region
• Cross-region via VPC peering for data replication
• Security groups reference each other (e.g., "only auth-sg can reach billing-sg")
```

### Uber's Network Architecture

```
Uber uses a multi-tier VPC design:

Tier 1: Edge (Public Subnets)
├── Load Balancers
├── API Gateway
└── DDoS mitigation

Tier 2: Application (Private Subnets)
├── Rider service
├── Driver service
├── Matching service
├── Pricing service
└── ETA service

Tier 3: Data (Isolated Subnets)
├── MySQL (trip data)
├── Cassandra (driver location)
├── Redis (caching)
└── Kafka (event streaming)

Rules:
• Tier 1 can reach Tier 2 (on specific ports only)
• Tier 2 can reach Tier 3 (database ports only)
• Tier 3 CANNOT reach Tier 1 (no direct internet access)
• Each tier has its own NACL (defense in depth)
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using default VPC for production | Shared with all accounts in region, less secure | Create a custom VPC with proper CIDR planning |
| Making database subnet public | Database exposed to internet (security disaster!) | Always put databases in private subnets with no IGW route |
| Security group with 0.0.0.0/0 on port 22 | SSH open to the world (brute force attacks) | Restrict SSH to VPN IP or use SSM Session Manager |
| Single NAT Gateway (no HA) | If NAT GW fails, all private instances lose internet | One NAT GW per AZ for high availability |
| Overlapping CIDR blocks between VPCs | Cannot peer VPCs with overlapping IPs | Plan CIDR allocation upfront (use non-overlapping ranges) |
| Not using multiple AZs | Single AZ failure takes down entire application | Spread subnets across 2-3 AZs minimum |
| Too large subnets (/16 for everything) | Waste IPs, harder to apply different routing rules | Right-size subnets: /24 for most, /20 for large workloads |
| Not restricting outbound traffic | Compromised instance can exfiltrate data freely | Restrict egress in security groups (allow only needed ports) |
| Hardcoding IPs in security groups | Doesn't scale when instances are replaced | Use security group references instead of CIDR blocks |

---

## When to Use / When NOT to Use

### Use VPC (Custom) When:
- ✅ Running production workloads
- ✅ You need isolation between environments (dev/staging/prod)
- ✅ Compliance requirements (HIPAA, PCI-DSS, SOC2)
- ✅ Multi-tier architecture (web/app/database layers)
- ✅ Connecting to on-premise networks (VPN/Direct Connect)
- ✅ Multiple teams sharing the same AWS account

### Use Default VPC When:
- ✅ Quick prototyping / learning
- ✅ Single throwaway EC2 instance for testing
- ✅ Non-sensitive workloads with no compliance needs

### Design Guidelines:
- ✅ **Least privilege**: Only open ports that are actually needed
- ✅ **Defense in depth**: Use BOTH security groups AND NACLs
- ✅ **Multi-AZ**: Always for production (minimum 2 AZs)
- ✅ **Private by default**: Put everything in private subnets unless it MUST be public
- ✅ **Reference SGs, not IPs**: Security groups should reference other SGs, not hardcoded CIDRs

---

## Key Takeaways

1. A **VPC** is your isolated private network in the cloud — you control the IP range, subnets, routing, and security.
2. **Public subnets** have a route to the Internet Gateway; **private subnets** do not (use NAT Gateway for outbound-only).
3. **Security Groups** are stateful, instance-level firewalls with ALLOW-only rules. They're your primary defense.
4. **NACLs** are stateless, subnet-level firewalls with ALLOW and DENY rules. Use them as a secondary layer.
5. Always put **databases and application servers in private subnets** — only load balancers and bastion hosts belong in public subnets.
6. **Reference security groups** instead of hardcoding IPs — this scales automatically as instances are added/removed.
7. Design for **multi-AZ** from day one — it's much harder to add later than to build in from the start.

---

## What's Next?

Next, we'll clarify the often-confused relationship between **API Gateway vs Reverse Proxy vs Load Balancer** — three components that overlap in functionality but serve distinct purposes in production architecture.

→ [05-gateway-vs-proxy-vs-lb.md](./05-gateway-vs-proxy-vs-lb.md)
