# Chapter 8: Network Security Groups (NSG) Deep Dive (Azure)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: NSG Fundamentals](#part-1-nsg-fundamentals)
- [Part 2: Application Security Groups (ASG)](#part-2-application-security-groups-asg)
- [Part 3: Augmented Security Rules](#part-3-augmented-security-rules)
- [Part 4: NSG vs Azure Firewall](#part-4-nsg-vs-azure-firewall)
- [Part 5: NSG Diagnostics & Effective Rules](#part-5-nsg-diagnostics--effective-rules)
- [Part 6: Common NSG Patterns](#part-6-common-nsg-patterns)
- [Part 7: CLI & Bicep Reference](#part-7-cli--bicep-reference)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure uses Network Security Groups (NSGs) as the primary network firewall at the subnet and NIC level, plus Application Security Groups (ASGs) for logical grouping. This chapter covers every field, priority system, ASGs, and integration with Azure Firewall.

```
What you'll learn:
├── NSG (deep dive)
│   ├── How they work
│   ├── Inbound & outbound rules
│   ├── All fields explained
│   ├── Default rules
│   ├── Priority system (100-4096)
│   ├── Service tags
│   └── Augmented security rules
├── Application Security Groups (ASG)
├── NSG vs Azure Firewall
├── NSG Flow Logs & Diagnostics
├── Azure Firewall (overview)
└── Real-world patterns
```

---

## Part 1: NSG Fundamentals

### What and How They Work

```
┌─────────────────────────────────────────────────────────────────────┐
│              NETWORK SECURITY GROUPS (NSG)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A firewall that filters traffic to and from Azure resources   │
│       at the subnet and/or NIC level.                               │
│                                                                       │
│ Key characteristics:                                                 │
│ ┌─────────────────────────────────────────────────────────────┐     │
│ │ 1. STATEFUL                                                   │     │
│ │    Return traffic auto-allowed. Same as AWS SGs.             │     │
│ │                                                                │     │
│ │ 2. ALLOW and DENY rules                                       │     │
│ │    Like GCP, Azure has both (unlike AWS SGs = allow only).   │     │
│ │                                                                │     │
│ │ 3. PRIORITY-BASED (100-4096)                                   │     │
│ │    Lower number = higher priority.                            │     │
│ │    First matching rule wins.                                  │     │
│ │    Built-in rules use 65000-65500 (cannot be changed).       │     │
│ │                                                                │     │
│ │ 4. APPLIED AT TWO LEVELS                                       │     │
│ │    ├── Subnet level: All VMs in the subnet                   │     │
│ │    └── NIC level: Specific VM's network interface            │     │
│ │    If both: BOTH are evaluated (most restrictive wins)       │     │
│ │                                                                │     │
│ │ 5. USES SERVICE TAGS                                           │     │
│ │    Built-in groups of IPs: Internet, VirtualNetwork,         │     │
│ │    AzureLoadBalancer, Storage, Sql, etc.                      │     │
│ │    ⚡ No need to track Azure IP ranges manually!              │     │
│ │                                                                │     │
│ │ 6. REGIONAL RESOURCE                                           │     │
│ │    NSG must be in same region as the resources it protects.  │     │
│ └─────────────────────────────────────────────────────────────┘     │
│                                                                       │
│ Traffic flow (when NSG on both subnet and NIC):                     │
│                                                                       │
│ INBOUND:                                                             │
│ [Internet] → [Subnet NSG] → [NIC NSG] → [VM]                      │
│ Both must ALLOW for traffic to reach the VM.                        │
│                                                                       │
│ OUTBOUND:                                                            │
│ [VM] → [NIC NSG] → [Subnet NSG] → [Internet]                      │
│ Both must ALLOW for traffic to leave.                               │
│                                                                       │
│ ⚡ Best practice: Apply NSG at subnet level (simpler to manage)    │
│    Use NIC-level NSG only for exceptions within a subnet.          │
│                                                                       │
│ Cross-cloud comparison:                                              │
│ ┌──────────────────┬──────────┬──────────┬──────────────┐          │
│ │ Feature          │ AWS SG   │ GCP FW   │ Azure NSG    │          │
│ │ Stateful         │ Yes      │ Yes      │ Yes          │          │
│ │ Deny rules       │ No       │ Yes      │ Yes          │          │
│ │ Priority         │ No       │ 0-65535  │ 100-4096     │          │
│ │ Applied to       │ ENI      │ VM (tag) │ Subnet/NIC   │          │
│ │ Service tags     │ No*      │ No       │ Yes!         │          │
│ │ ASG (logical grp)│ No       │ Tags     │ Yes (ASG)    │          │
│ │ Hierarchical     │ No       │ Yes      │ No (use FW)  │          │
│ └──────────────────┴──────────┴──────────┴──────────────┘          │
│ * AWS has prefix lists, similar concept                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Default NSG Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│              DEFAULT NSG RULES (Cannot be deleted!)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Default INBOUND rules:                                               │
│ ┌───────┬──────────────────┬──────────┬───────────────┬──────────┐ │
│ │ Pri   │ Name             │ Source   │ Dest          │ Action   │ │
│ ├───────┼──────────────────┼──────────┼───────────────┼──────────┤ │
│ │ 65000 │ AllowVnetInBound │ Virtual  │ Virtual       │ Allow    │ │
│ │       │                  │ Network  │ Network       │          │ │
│ │ 65001 │ AllowAzureLB     │ Azure    │ Any           │ Allow    │ │
│ │       │ InBound          │ LB       │               │          │ │
│ │ 65500 │ DenyAllInBound   │ Any      │ Any           │ Deny     │ │
│ └───────┴──────────────────┴──────────┴───────────────┴──────────┘ │
│                                                                       │
│ Default OUTBOUND rules:                                              │
│ ┌───────┬──────────────────┬──────────┬───────────────┬──────────┐ │
│ │ Pri   │ Name             │ Source   │ Dest          │ Action   │ │
│ ├───────┼──────────────────┼──────────┼───────────────┼──────────┤ │
│ │ 65000 │ AllowVnetOutBound│ Virtual  │ Virtual       │ Allow    │ │
│ │       │                  │ Network  │ Network       │          │ │
│ │ 65001 │ AllowInternet    │ Any      │ Internet      │ Allow    │ │
│ │       │ OutBound         │          │               │          │ │
│ │ 65500 │ DenyAllOutBound  │ Any      │ Any           │ Deny     │ │
│ └───────┴──────────────────┴──────────┴───────────────┴──────────┘ │
│                                                                       │
│ What this means:                                                     │
│ ├── VNet-to-VNet traffic: ALLOWED (within same VNet)               │
│ ├── Azure Load Balancer health probes: ALLOWED                     │
│ ├── All other inbound: DENIED                                      │
│ ├── VNet outbound: ALLOWED                                         │
│ ├── Internet outbound: ALLOWED                                     │
│ └── All other outbound: DENIED                                     │
│                                                                       │
│ ⚠️ "VirtualNetwork" service tag includes:                           │
│    VNet address space + peered VNet space +                        │
│    on-premises ranges (if connected via VPN/ER)                    │
│    → This is broader than just your VNet!                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating an NSG

```
Console → Create a resource → Network security group

┌─────────────────────────────────────────────────────────────────┐
│           CREATE NETWORK SECURITY GROUP                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Subscription:    [Visual Studio Enterprise ▼]                  │
│ Resource group:  [rg-prod ▼]                                    │
│ Name:            [nsg-app-subnet]                               │
│ Region:          [Central India ▼]                              │
│                                                                   │
│ [Review + create]                                                │
│                                                                   │
│ Then add rules:                                                  │
│ NSG → Inbound security rules → Add                             │
│                                                                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Source:           ○ Any  ○ IP Addresses  ○ Service Tag   │   │
│ │                   ○ Application Security Group           │   │
│ │ Source value:     [Internet ▼]                           │   │
│ │ Source ports:     [*] (usually * for source)             │   │
│ │                                                           │   │
│ │ Destination:      ○ Any  ○ IP Addresses  ○ Service Tag  │   │
│ │                   ○ Application Security Group           │   │
│ │ Destination value:[VirtualNetwork ▼]                     │   │
│ │ Service:          [Custom ▼]  or [HTTP ▼] [HTTPS ▼]    │   │
│ │ Destination port: [443]                                  │   │
│ │ Protocol:         ○ Any  ● TCP  ○ UDP  ○ ICMP          │   │
│ │ Action:           ● Allow  ○ Deny                       │   │
│ │ Priority:         [300]                                  │   │
│ │ Name:             [AllowHTTPS_Inbound]                   │   │
│ │ Description:      [Allow HTTPS from internet]           │   │
│ │                                                           │   │
│ │ [Add]                                                     │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Associate with subnet:                                           │
│ NSG → Subnets → Associate                                      │
│   Virtual network: [prod-vnet ▼]                                │
│   Subnet: [app-subnet ▼]                                       │
│   [OK]                                                           │
│                                                                   │
│ Or associate with NIC:                                           │
│ NSG → Network interfaces → Associate                           │
│   Network interface: [app-vm-1-nic ▼]                          │
│   [OK]                                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### All Rule Fields Explained

```
┌─────────────────────────────────────────────────────────────────────┐
│              NSG RULE FIELDS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ SOURCE:                                                              │
│ ├── Any: 0.0.0.0/0 + ::/0 (any source)                            │
│ ├── IP Addresses: Specific CIDR (10.0.1.0/24, 203.0.113.5/32)    │
│ │   Can specify multiple CIDRs (comma-separated)                  │
│ ├── Service Tag: Named group of IPs managed by Azure              │
│ │   ├── Internet: All public IPs                                  │
│ │   ├── VirtualNetwork: VNet + peered + on-prem                   │
│ │   ├── AzureLoadBalancer: LB health probe IPs                    │
│ │   ├── AzureCloud: All Azure datacenter IPs                     │
│ │   ├── AzureCloud.CentralIndia: Region-specific                 │
│ │   ├── Storage: Azure Storage IPs                                │
│ │   ├── Storage.CentralIndia: Regional                            │
│ │   ├── Sql: Azure SQL IPs                                        │
│ │   ├── AzureActiveDirectory: Entra ID IPs                       │
│ │   ├── AzureKeyVault: Key Vault IPs                             │
│ │   ├── AzureMonitor: Monitor IPs                                │
│ │   ├── AzureContainerRegistry: ACR IPs                          │
│ │   ├── EventHub: Event Hubs IPs                                 │
│ │   ├── ServiceBus: Service Bus IPs                              │
│ │   ├── AzureDevOps: Azure DevOps IPs                            │
│ │   ├── GatewayManager: VPN/ER GW management                    │
│ │   └── ... 100+ service tags available!                          │
│ │   ⚡ Service tags auto-update when Azure adds IPs!              │
│ │                                                                    │
│ └── Application Security Group (ASG): Logical VM group            │
│     (See ASG section below)                                        │
│                                                                       │
│ SOURCE PORT RANGES:                                                  │
│   Usually "*" (any) — clients use random ephemeral ports           │
│   Rarely need to restrict source ports                             │
│                                                                       │
│ DESTINATION:                                                         │
│   Same options as Source (Any, IP, Service Tag, ASG)               │
│                                                                       │
│ DESTINATION PORT RANGES:                                             │
│   ├── Single port: 443                                             │
│   ├── Port range: 8080-8090                                        │
│   ├── Multiple: 80,443,8080                                       │
│   └── All: *                                                       │
│                                                                       │
│ SERVICE (shortcut):                                                  │
│   ├── HTTP (TCP 80)                                                │
│   ├── HTTPS (TCP 443)                                              │
│   ├── SSH (TCP 22)                                                 │
│   ├── RDP (TCP 3389)                                               │
│   ├── MS SQL (TCP 1433)                                            │
│   ├── MySQL (TCP 3306)                                             │
│   ├── PostgreSQL (TCP 5432)                                        │
│   ├── DNS (TCP/UDP 53)                                             │
│   ├── Custom (you specify)                                         │
│   └── When you select a service, port and protocol auto-fill      │
│                                                                       │
│ PROTOCOL: Any, TCP, UDP, ICMP                                       │
│                                                                       │
│ ACTION: Allow, Deny                                                  │
│                                                                       │
│ PRIORITY (100-4096):                                                 │
│   ├── Lower number = higher priority (checked first)              │
│   ├── First match wins                                             │
│   ├── 100-999: High-priority rules (deny specific IPs)           │
│   ├── 1000-2999: Standard application rules                      │
│   ├── 3000-4096: Lower priority / catch-alls                     │
│   ├── 65000-65500: Default rules (cannot change)                 │
│   └── Use increments of 10-100 for easy insertion later           │
│                                                                       │
│ NAME: Must be unique within the NSG. Descriptive!                   │
│   Convention: Allow|Deny_Protocol_Source_Destination                │
│   Example: AllowHTTPS_Internet_AppSubnet                           │
│                                                                       │
│ DESCRIPTION: Optional but recommended for audit/documentation      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Service Tags Deep Dive

```
┌─────────────────────────────────────────────────────────────────────┐
│              SERVICE TAGS — Azure's Killer Feature                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Named groups of IP prefixes managed by Azure.                 │
│       Azure updates them automatically — you never manage IPs!     │
│                                                                       │
│ ⚡ This is HUGE! AWS has no direct equivalent (closest: prefix     │
│    lists for S3/DynamoDB only). Azure has 100+ service tags.       │
│                                                                       │
│ Most commonly used:                                                  │
│ ┌──────────────────────────┬────────────────────────────────────┐  │
│ │ Service Tag              │ Use Case                           │  │
│ ├──────────────────────────┼────────────────────────────────────┤  │
│ │ Internet                 │ All public IPs (not VNet/on-prem) │  │
│ │ VirtualNetwork           │ VNet + peered + on-prem ranges    │  │
│ │ AzureLoadBalancer        │ Health probe source IPs           │  │
│ │ Storage                  │ Azure Storage IPs                 │  │
│ │ Storage.CentralIndia     │ Storage IPs in your region        │  │
│ │ Sql                      │ Azure SQL IPs                     │  │
│ │ Sql.CentralIndia         │ SQL IPs in your region            │  │
│ │ AzureActiveDirectory     │ Entra ID (for auth traffic)      │  │
│ │ AzureKeyVault            │ Key Vault IPs                    │  │
│ │ AzureMonitor             │ Monitor/Log Analytics IPs        │  │
│ │ AzureContainerRegistry   │ ACR IPs                          │  │
│ │ GatewayManager           │ VPN/ER gateway management        │  │
│ │ AzureBastionSubnet       │ Bastion subnet IPs               │  │
│ │ AzureDevOps              │ Azure DevOps IPs                 │  │
│ │ AzureCloud               │ All Azure datacenter IPs         │  │
│ │ AzureCloud.CentralIndia  │ All Azure IPs in your region    │  │
│ └──────────────────────────┴────────────────────────────────────┘  │
│                                                                       │
│ Example use cases:                                                   │
│ ├── Allow outbound to Azure Storage only (not all internet):      │
│ │   Dest = Storage, Port = 443, Action = Allow                    │
│ │                                                                    │
│ ├── Allow Azure DevOps agents to deploy:                          │
│ │   Source = AzureDevOps, Port = 22, Action = Allow               │
│ │                                                                    │
│ ├── Allow Bastion to reach VMs:                                   │
│ │   Source = AzureBastionSubnet, Port = 22,3389                   │
│ │                                                                    │
│ └── Block internet but allow Azure services:                      │
│     Deny Internet outbound + Allow AzureCloud outbound            │
│                                                                       │
│ List all available service tags:                                    │
│ az network list-service-tags --location centralindia \              │
│   --query 'values[].name' -o tsv                                    │
│                                                                       │
│ Get IPs for a specific tag:                                         │
│ az network list-service-tags --location centralindia \              │
│   --query "values[?name=='Storage.CentralIndia'].properties.       │
│     addressPrefixes" -o tsv                                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Application Security Groups (ASG)

```
┌─────────────────────────────────────────────────────────────────────┐
│          APPLICATION SECURITY GROUPS (ASG)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Logical grouping of VMs that you can reference in NSG rules  │
│       instead of using IP addresses. Like AWS SG references but    │
│       more flexible!                                                │
│                                                                       │
│ Problem: VMs get different IPs, Auto Scale adds/removes VMs.        │
│ Solution: Group VMs into ASGs, use ASG as source/destination.       │
│                                                                       │
│ Without ASG:                                                         │
│ NSG rule: Allow TCP 5432 from 10.0.2.4, 10.0.2.5, 10.0.2.6      │
│ ⚠️ Must update every time a VM is added/removed/IP changes         │
│                                                                       │
│ With ASG:                                                            │
│ NSG rule: Allow TCP 5432 from ASG: app-servers                     │
│ ✅ Any VM in "app-servers" ASG is automatically included           │
│                                                                       │
│ How it works:                                                        │
│                                                                       │
│ ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │
│ │ app-vm-1     │   │ app-vm-2     │   │ app-vm-3     │            │
│ │ NIC → ASG:   │   │ NIC → ASG:   │   │ NIC → ASG:   │            │
│ │ app-servers  │   │ app-servers  │   │ app-servers  │            │
│ └──────┬───────┘   └──────┬───────┘   └──────┬───────┘            │
│        └──────────────────┼──────────────────┘                      │
│                           ▼                                          │
│              NSG Rule: Allow TCP 5432                                │
│              Source ASG: app-servers                                 │
│              Dest ASG: db-servers                                    │
│                           ▼                                          │
│        ┌──────────────────┼──────────────────┐                      │
│ ┌──────┴───────┐   ┌──────┴───────┐                                │
│ │ db-vm-1      │   │ db-vm-2      │                                │
│ │ NIC → ASG:   │   │ NIC → ASG:   │                                │
│ │ db-servers   │   │ db-servers   │                                │
│ └──────────────┘   └──────────────┘                                │
│                                                                       │
│ Key facts:                                                           │
│ ├── ASG is a regional resource (must match VM region)              │
│ ├── A NIC can belong to multiple ASGs                              │
│ ├── ASGs can be used as Source AND Destination in NSG rules        │
│ ├── Both source and dest ASGs must be in same VNet               │
│ │   (or all NICs in rule must be in same VNet)                    │
│ ├── FREE (no charge for ASGs)                                      │
│ ├── Reduces rule count (one rule covers many VMs)                 │
│ └── ⚡ Works perfectly with VMSS (auto-associate on scale-out)     │
│                                                                       │
│ Comparison:                                                          │
│ ├── AWS: SG reference (similar but less flexible — SG = firewall  │
│ │   AND group combined, can't separate)                            │
│ ├── GCP: Network tags (similar concept, but tags are strings      │
│ │   on VMs, not separate resources)                                │
│ └── Azure ASG: Separate resource, more explicit, can attach       │
│     multiple ASGs to one NIC                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating and Using ASGs

```
Console → Create a resource → Application security group

Step 1: Create ASGs
┌─────────────────────────────────────────────────────────────────┐
│         CREATE APPLICATION SECURITY GROUP                        │
├─────────────────────────────────────────────────────────────────┤
│ Subscription:    [Visual Studio Enterprise ▼]                  │
│ Resource group:  [rg-prod ▼]                                    │
│ Name:            [asg-app-servers]                               │
│ Region:          [Central India ▼]                              │
│ [Review + create]                                                │
│                                                                   │
│ Repeat for: asg-web-servers, asg-db-servers, asg-cache-servers │
└─────────────────────────────────────────────────────────────────┘

Step 2: Associate VMs with ASGs
  VM → Networking → Application security groups
  [Add] → [asg-app-servers ▼] → [Save]

  Or during NIC configuration:
  NIC → IP configurations → Edit → Application security groups
  [asg-app-servers ✅]

Step 3: Use in NSG rules
  NSG → Inbound security rules → Add
  Source: Application Security Group → [asg-app-servers ▼]
  Destination: Application Security Group → [asg-db-servers ▼]
  Service: PostgreSQL
  Port: 5432
  Action: Allow
  Priority: 300
  Name: AllowPostgres_App_DB

CLI:
  # Create ASG
  az network asg create \
    --name asg-app-servers \
    --resource-group rg-prod \
    --location centralindia

  # Associate NIC with ASG
  az network nic ip-config update \
    --name ipconfig1 \
    --nic-name app-vm-1-nic \
    --resource-group rg-prod \
    --application-security-groups asg-app-servers

  # Create NSG rule with ASGs
  az network nsg rule create \
    --name AllowPostgres_App_DB \
    --nsg-name nsg-data-subnet \
    --resource-group rg-prod \
    --priority 300 \
    --direction Inbound \
    --access Allow \
    --protocol Tcp \
    --destination-port-ranges 5432 \
    --source-asgs asg-app-servers \
    --destination-asgs asg-db-servers

Bicep:
  resource asgApp 'Microsoft.Network/applicationSecurityGroups@2023-05-01' = {
    name: 'asg-app-servers'
    location: location
  }

  resource asgDb 'Microsoft.Network/applicationSecurityGroups@2023-05-01' = {
    name: 'asg-db-servers'
    location: location
  }

  resource nsgRule 'Microsoft.Network/networkSecurityGroups/securityRules@2023-05-01' = {
    parent: nsgDataSubnet
    name: 'AllowPostgres_App_DB'
    properties: {
      priority: 300
      direction: 'Inbound'
      access: 'Allow'
      protocol: 'Tcp'
      sourcePortRange: '*'
      destinationPortRange: '5432'
      sourceApplicationSecurityGroups: [
        { id: asgApp.id }
      ]
      destinationApplicationSecurityGroups: [
        { id: asgDb.id }
      ]
    }
  }
```

---

## Part 3: Augmented Security Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│          AUGMENTED SECURITY RULES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A single NSG rule that combines multiple sources, destinations│
│       ports, and service tags. Reduces rule count dramatically!     │
│                                                                       │
│ Without augmented rules (3 separate rules):                          │
│ ├── Rule 300: Allow TCP 80 from 10.0.1.0/24                       │
│ ├── Rule 310: Allow TCP 443 from 10.0.1.0/24                      │
│ └── Rule 320: Allow TCP 8080 from 10.0.1.0/24                     │
│                                                                       │
│ With augmented rules (1 rule):                                       │
│ └── Rule 300: Allow TCP 80,443,8080 from 10.0.1.0/24              │
│                                                                       │
│ You can also combine:                                                │
│ ├── Multiple source IPs: 10.0.1.0/24, 10.0.2.0/24                │
│ ├── Multiple destination IPs: 10.0.3.0/24, 10.0.4.0/24           │
│ ├── Multiple ports: 80, 443, 8080-8090                            │
│ └── Service tags + IPs in same rule                                │
│                                                                       │
│ Limits per NSG:                                                      │
│ ├── Rules per NSG: 1,000 (default, up to 5,000 with request)     │
│ ├── Source/dest per rule: Up to 4,000 combined                    │
│ ├── Ports per rule: Up to 250                                     │
│ └── With augmented rules: 1,000 rules can cover far more combos   │
│                                                                       │
│ az network nsg rule create \                                         │
│   --name AllowWebTraffic \                                           │
│   --nsg-name nsg-web-subnet \                                        │
│   --resource-group rg-prod \                                         │
│   --priority 300 \                                                    │
│   --direction Inbound \                                               │
│   --access Allow \                                                    │
│   --protocol Tcp \                                                    │
│   --source-address-prefixes 10.0.1.0/24 10.0.2.0/24 \              │
│   --destination-port-ranges 80 443 8080                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: NSG vs Azure Firewall

```
┌─────────────────────────────────────────────────────────────────────┐
│            NSG vs AZURE FIREWALL                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────┬──────────────────┬──────────────────────┐ │
│ │ Feature              │ NSG              │ Azure Firewall       │ │
│ ├──────────────────────┼──────────────────┼──────────────────────┤ │
│ │ Level                │ Subnet/NIC       │ VNet (centralized)   │ │
│ │ Layer                │ L3/L4            │ L3/L4/L7             │ │
│ │ FQDN filtering      │ No               │ Yes (*.google.com)   │ │
│ │ TLS inspection       │ No               │ Yes (Premium)        │ │
│ │ Threat intelligence  │ No               │ Yes                  │ │
│ │ IDPS                 │ No               │ Yes (Premium)        │ │
│ │ URL filtering        │ No               │ Yes                  │ │
│ │ NAT rules            │ No               │ Yes (DNAT/SNAT)      │ │
│ │ Central management   │ Per NSG          │ Firewall Manager     │ │
│ │ Cost                 │ FREE             │ ~$900+/month         │ │
│ │ Log analysis         │ Flow logs        │ Rich diagnostics     │ │
│ │ Cross-VNet           │ No               │ Yes (hub FW)         │ │
│ └──────────────────────┴──────────────────┴──────────────────────┘ │
│                                                                       │
│ When to use NSG:                                                     │
│ ├── Always! It's free and fundamental                              │
│ ├── L3/L4 port-based filtering                                     │
│ ├── Intra-VNet and subnet-level security                          │
│ └── 90% of your firewall needs                                    │
│                                                                       │
│ When to add Azure Firewall:                                          │
│ ├── Need FQDN filtering (allow *.ubuntu.com for updates)          │
│ ├── Centralized egress control (hub-and-spoke)                    │
│ ├── Threat intelligence and IDPS                                  │
│ ├── TLS inspection (compliance)                                   │
│ ├── Spoke-to-spoke traffic inspection                             │
│ └── Enterprise compliance requirements                            │
│                                                                       │
│ Typical pattern:                                                     │
│ ├── NSG on every subnet (always)                                  │
│ ├── Azure Firewall in hub VNet (if budget allows)                 │
│ └── UDRs force spoke traffic through Azure Firewall               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: NSG Diagnostics & Effective Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│          NSG DIAGNOSTICS                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Effective Security Rules:                                            │
│ When NSG is on both subnet AND NIC, the effective rules are         │
│ the combination of both. Azure shows you the merged view!           │
│                                                                       │
│ View effective rules:                                                │
│ VM → Networking → "Effective security rules" tab                    │
│ Shows: All rules from subnet NSG + NIC NSG, sorted by priority     │
│                                                                       │
│ Network Watcher → NSG Diagnostics:                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Source: 10.0.2.5                                              │   │
│ │ Destination: 10.0.3.5                                         │   │
│ │ Port: 5432                                                    │   │
│ │ Protocol: TCP                                                 │   │
│ │ Direction: Inbound                                            │   │
│ │                                                                │   │
│ │ Result: ✅ ALLOWED                                             │   │
│ │ Matched rule: AllowPostgres_App_DB (Priority 300)             │   │
│ │ NSG: nsg-data-subnet                                          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ IP Flow Verify (Network Watcher):                                   │
│ Tests if a specific packet would be allowed or denied.              │
│ Tells you WHICH RULE matched.                                       │
│ ⚡ Use this when debugging "why can't VM A reach VM B?"             │
│                                                                       │
│ CLI:                                                                 │
│ # Check if traffic is allowed                                       │
│ az network watcher test-ip-flow \                                    │
│   --vm app-vm-1 \                                                    │
│   --resource-group rg-prod \                                         │
│   --direction Inbound \                                               │
│   --protocol TCP \                                                    │
│   --local 10.0.2.5:5432 \                                           │
│   --remote 10.0.1.10:50000                                          │
│                                                                       │
│ # View effective security rules for a NIC                           │
│ az network nic list-effective-nsg \                                  │
│   --name app-vm-1-nic \                                              │
│   --resource-group rg-prod                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Common NSG Patterns

### 3-Tier Web Application

```
┌─────────────────────────────────────────────────────────────────────┐
│         3-TIER APPLICATION NSG PATTERN                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ NSG: nsg-web-subnet (on web-subnet)                                 │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Inbound:                                                      │    │
│ │ 300 Allow TCP 443 from Internet                               │    │
│ │ 310 Allow TCP 80 from Internet (redirect to HTTPS)           │    │
│ │ 320 Allow TCP 22 from AzureBastionSubnet (SSH via Bastion)  │    │
│ │ 65000 Allow VirtualNetwork (default)                          │    │
│ │ 65001 Allow AzureLoadBalancer (default)                      │    │
│ │ 65500 Deny All (default)                                      │    │
│ │                                                                │    │
│ │ Outbound:                                                      │    │
│ │ 300 Allow TCP 8080 to asg-app-servers                         │    │
│ │ 65000 Allow VirtualNetwork (default)                          │    │
│ │ 65001 Allow Internet (default)                                │    │
│ │ 65500 Deny All (default)                                      │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                        │                                              │
│                        ▼                                              │
│ NSG: nsg-app-subnet (on app-subnet)                                 │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Inbound:                                                      │    │
│ │ 300 Allow TCP 8080 from asg-web-servers                      │    │
│ │ 310 Allow TCP 22 from AzureBastionSubnet                     │    │
│ │ 65000 Allow VirtualNetwork (default)                          │    │
│ │ 65500 Deny All (default)                                      │    │
│ │                                                                │    │
│ │ Outbound:                                                      │    │
│ │ 300 Allow TCP 5432 to asg-db-servers                          │    │
│ │ 310 Allow TCP 6379 to asg-cache-servers                      │    │
│ │ 320 Allow TCP 443 to AzureKeyVault (service tag)             │    │
│ │ 330 Allow TCP 443 to Storage (service tag)                   │    │
│ │ 65000 Allow VirtualNetwork (default)                          │    │
│ │ 65001 Allow Internet (default)                                │    │
│ │ 65500 Deny All (default)                                      │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                        │                                              │
│                        ▼                                              │
│ NSG: nsg-data-subnet (on data-subnet)                               │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Inbound:                                                      │    │
│ │ 300 Allow TCP 5432 from asg-app-servers                      │    │
│ │ 310 Allow TCP 22 from AzureBastionSubnet (DBA access)       │    │
│ │ 65000 Allow VirtualNetwork (default)                          │    │
│ │ 65500 Deny All (default)                                      │    │
│ │                                                                │    │
│ │ Outbound:                                                      │    │
│ │ 300 Allow TCP 443 to Storage (for backups)                   │    │
│ │ 310 Allow TCP 443 to AzureMonitor (for diagnostics)         │    │
│ │ 400 Deny Any to Internet (block direct internet!)            │    │
│ │ 65000 Allow VirtualNetwork (default)                          │    │
│ │ 65500 Deny All (default)                                      │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Restricting Outbound (Hardened)

```
To block internet for sensitive subnets while allowing Azure services:

NSG rule on data-subnet:
  Priority 300: Allow TCP 443 to Storage (backups)
  Priority 310: Allow TCP 443 to Sql (if needed)
  Priority 320: Allow TCP 443 to AzureMonitor
  Priority 330: Allow TCP 443 to AzureKeyVault
  Priority 400: DENY Any to Internet  ← blocks everything else!

⚠️ Default rule 65001 AllowInternetOutBound still exists but
   your rule 400 has higher priority → Internet blocked!

The service tag rules (300-330) still work because they match
at higher priority before the deny rule.
```

---

## Part 7: CLI & Bicep Reference

```bash
# Create NSG
az network nsg create \
  --name nsg-app-subnet \
  --resource-group rg-prod \
  --location centralindia

# Add inbound rule
az network nsg rule create \
  --name AllowHTTPS \
  --nsg-name nsg-web-subnet \
  --resource-group rg-prod \
  --priority 300 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --destination-port-ranges 443

# Add rule with ASGs
az network nsg rule create \
  --name AllowApp_Web_App \
  --nsg-name nsg-app-subnet \
  --resource-group rg-prod \
  --priority 300 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-asgs asg-web-servers \
  --destination-asgs asg-app-servers \
  --destination-port-ranges 8080

# Add deny rule
az network nsg rule create \
  --name DenyInternet_Outbound \
  --nsg-name nsg-data-subnet \
  --resource-group rg-prod \
  --priority 400 \
  --direction Outbound \
  --access Deny \
  --protocol '*' \
  --destination-address-prefixes Internet \
  --destination-port-ranges '*'

# Associate with subnet
az network vnet subnet update \
  --name app-subnet \
  --resource-group rg-prod \
  --vnet-name prod-vnet \
  --network-security-group nsg-app-subnet

# List rules
az network nsg rule list \
  --nsg-name nsg-app-subnet \
  --resource-group rg-prod \
  --output table

# Delete a rule
az network nsg rule delete \
  --name OldRule \
  --nsg-name nsg-app-subnet \
  --resource-group rg-prod
```

```bicep
// Bicep - Complete NSG with rules
resource nsgAppSubnet 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'nsg-app-subnet'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowHTTPS_LB'
        properties: {
          priority: 300
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'AzureLoadBalancer'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '8080'
        }
      }
      {
        name: 'AllowSSH_Bastion'
        properties: {
          priority: 310
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'AzureBastionSubnet'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '22'
        }
      }
      {
        name: 'AllowPostgres_App_DB'
        properties: {
          priority: 300
          direction: 'Outbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceApplicationSecurityGroups: [
            { id: asgApp.id }
          ]
          sourcePortRange: '*'
          destinationApplicationSecurityGroups: [
            { id: asgDb.id }
          ]
          destinationPortRange: '5432'
        }
      }
    ]
  }
}
```

---

## Part 8: Real-World Patterns

### Startup (5-10 devs)

```
NSGs: 3-4 (one per subnet tier)
├── nsg-web-subnet: Allow 80/443 from Internet, SSH from Bastion
├── nsg-app-subnet: Allow 8080 from web, SSH from Bastion
├── nsg-data-subnet: Allow 5432 from app, deny Internet outbound
└── No ASGs needed (few VMs, can use IP ranges)

No Azure Firewall (save $900+/month).
NSG Flow Logs: Optional (enable on data-subnet for audit).
Use Bastion Developer (free preview) for SSH.
```

### Mid-Size (50-100 devs)

```
NSGs: 8-12 (per subnet per environment)
├── prod-nsg-web, prod-nsg-app, prod-nsg-data
├── staging-nsg-web, staging-nsg-app, staging-nsg-data
├── dev-nsg-web, dev-nsg-app, dev-nsg-data
└── shared-nsg-bastion, shared-nsg-management

ASGs: 10-15 (per role)
├── asg-web-servers, asg-app-servers, asg-db-servers
├── asg-cache-servers, asg-worker-servers
├── asg-monitoring, asg-bastion
└── Per environment ASGs if needed

Service tags: Heavy use (Storage, Sql, AzureMonitor, etc.)
NSG Flow Logs: All production subnets → Traffic Analytics
Azure Firewall: Consider for hub (centralized egress)
```

### Enterprise (500+ devs)

```
NSGs: 50-100+ (managed via Bicep modules)
├── Centralized NSG Bicep module with standard patterns
├── NSG changes via PR review (Git-based deployment)
├── Per-subscription, per-environment, per-tier NSGs
├── ASGs for every application role
├── Service tags extensively used
├── Augmented rules to reduce rule count
└── Outbound restricted on all sensitive subnets

Azure Firewall: In every hub VNet
├── Firewall Manager for central policy
├── FQDN rules for approved external services
├── TLS inspection (Premium) for compliance
├── Threat intelligence enabled
└── IDPS in alert+deny mode

NSG Flow Logs: All subnets
├── Traffic Analytics (10-min interval for prod)
├── Export to Log Analytics for KQL queries
├── Microsoft Sentinel integration (SIEM)
└── Automated alerts on anomalous traffic

Azure Policy: Enforce NSGs on all subnets
├── Deny subnet creation without NSG
├── Deny rules with 0.0.0.0/0 source for SSH/RDP
└── Audit overly permissive rules
```

---

## Quick Reference

| Feature | Details |
|---------|---------|
| Stateful | Yes |
| Allow + Deny | Yes |
| Priority | 100-4096 (user) + 65000-65500 (default) |
| Applied to | Subnet and/or NIC |
| Service Tags | 100+ (Internet, VirtualNetwork, Storage, etc.) |
| ASG support | Yes (logical VM grouping) |
| Default inbound | Deny all (except VNet + LB) |
| Default outbound | Allow VNet + Internet |
| Cost | FREE |
| Rules per NSG | 1,000 (up to 5,000) |
| Augmented rules | Multiple IPs/ports per rule |
| Flow Logs | Via Network Watcher |
| Diagnostics | IP Flow Verify, Effective Rules, NSG Diagnostics |

---

## What's Next?

In the next chapter, we'll cover Azure DNS — public and private DNS zones, record sets, alias records, and Private DNS zones for Private Endpoints.

→ Next: [Chapter 9: Azure DNS](09-azure-dns.md)

---

*Last Updated: May 2026*
