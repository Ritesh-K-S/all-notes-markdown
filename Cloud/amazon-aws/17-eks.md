# Chapter 17: EKS - Elastic Kubernetes Service

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Kubernetes Fundamentals (for EKS)](#part-1-kubernetes-fundamentals-for-eks)
- [Part 2: EKS Architecture](#part-2-eks-architecture)
- [Part 3: Cluster Creation (Full Console Walkthrough)](#part-3-cluster-creation-full-console-walkthrough)
- [Part 4: Managed Node Groups](#part-4-managed-node-groups)
- [Part 5: Fargate Profiles](#part-5-fargate-profiles)
- [Part 6: IRSA (IAM Roles for Service Accounts)](#part-6-irsa-iam-roles-for-service-accounts)
- [Part 7: EKS Add-ons Deep Dive](#part-7-eks-add-ons-deep-dive)
- [Part 8: EKS Access Management](#part-8-eks-access-management)
- [Part 9: Terraform](#part-9-terraform)
- [Part 10: eksctl & kubectl CLI Reference](#part-10-eksctl--kubectl-cli-reference)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Kubernetes? Why EKS?

> **Real-World Analogy:** If ECS is a managed apartment building (AWS designed, AWS rules), then EKS is like a co-working space built on a universal blueprint (Kubernetes). You bring your Kubernetes knowledge from anywhere — Google Cloud, Azure, on-premises — and it works the same way. AWS just handles the building management (control plane) so you can focus on your work.

**Why does this matter?** Kubernetes is the industry standard for container orchestration. If your company uses Kubernetes (and most large companies do), EKS is how you run it on AWS without managing the control plane yourself.

**Do I need EKS?** If you're just starting with containers, ECS is simpler. Use EKS when your team already knows Kubernetes or you need portability across clouds.

Amazon Elastic Kubernetes Service (EKS) is a fully managed Kubernetes control plane. You don't install, operate, or maintain the Kubernetes master nodes — AWS manages them. You focus on deploying your workloads (pods) on worker nodes.

```
What you'll learn:
├── Kubernetes Fundamentals (for EKS context)
│   ├── What is Kubernetes, why use it
│   ├── Key concepts: Pods, Deployments, Services
│   └── Control plane vs data plane
├── EKS Architecture
│   ├── Managed control plane
│   ├── Worker nodes (EC2 vs Fargate)
│   └── EKS vs ECS comparison
├── Cluster Creation (Full Console Walkthrough)
│   ├── Cluster config, version, IAM role
│   ├── Networking (VPC, subnets, endpoint access)
│   ├── Logging, secrets encryption
│   └── Add-ons (CoreDNS, kube-proxy, VPC CNI, EBS CSI)
├── Node Groups (Managed)
│   ├── AMI type, instance types, scaling
│   ├── Launch template, labels, taints
│   └── Spot instances in node groups
├── Fargate Profiles
│   ├── Serverless pods
│   ├── Namespace + label selectors
│   └── Fargate vs managed nodes
├── IRSA (IAM Roles for Service Accounts)
├── EKS Add-ons
├── kubectl & eksctl CLI
├── Terraform
└── Real-world patterns
```

---

## Part 1: Kubernetes Fundamentals (for EKS)

```
┌─────────────────────────────────────────────────────────────────────┐
│           KUBERNETES BASICS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Kubernetes?                                                 │
│ ├── Container orchestration platform                              │
│ ├── Manages WHERE containers run, HOW MANY, and NETWORKING      │
│ ├── Self-healing: Restarts failed containers automatically      │
│ ├── Scaling: Add/remove containers based on demand              │
│ └── Rolling updates: Deploy new versions without downtime       │
│                                                                       │
│ Without Kubernetes:                                                 │
│ "I have 50 containers. Which server does each go on?             │
│  What if a server dies? What if traffic spikes?"                 │
│ → Kubernetes answers ALL of these automatically.                 │
│                                                                       │
│ Key concepts:                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Pod        │ Smallest unit. 1+ containers sharing network.  │   │
│ │            │ Usually 1 container per pod.                    │   │
│ ├────────────┼───────────────────────────────────────────────-─┤   │
│ │ Deployment │ Manages pod replicas. "Run 3 copies of my app" │   │
│ │            │ Handles rolling updates & rollbacks.            │   │
│ ├────────────┼────────────────────────────────────────────────-┤   │
│ │ Service    │ Stable network endpoint for pods.               │   │
│ │            │ Types: ClusterIP, NodePort, LoadBalancer.       │   │
│ ├────────────┼────────────────────────────────────────────────-┤   │
│ │ Namespace  │ Virtual cluster isolation. Group resources.     │   │
│ ├────────────┼────────────────────────────────────────────────-┤   │
│ │ Node       │ A machine (EC2) that runs pods.                 │   │
│ ├────────────┼────────────────────────────────────────────────-┤   │
│ │ Ingress    │ HTTP routing rules (path/host → service).       │   │
│ └────────────┴────────────────────────────────────────────────-┘   │
│                                                                       │
│ Architecture:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Control Plane (Masters — managed by AWS in EKS)             │  │
│ │ ├── API Server: kubectl talks to this                      │  │
│ │ ├── etcd: Cluster state database                           │  │
│ │ ├── Scheduler: Decides which node runs each pod            │  │
│ │ └── Controller Manager: Maintains desired state            │  │
│ ├──────────────────────────────────────────────────────────────┤  │
│ │ Data Plane (Workers — you manage or use Fargate)           │  │
│ │ ├── Node 1: [Pod A] [Pod B]                                │  │
│ │ ├── Node 2: [Pod A] [Pod C]                                │  │
│ │ └── Node 3: [Pod B] [Pod D]                                │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ In EKS: AWS manages the entire control plane.                     │
│ You manage: Worker nodes (or use Fargate to skip that too).      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: EKS Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│           EKS ARCHITECTURE                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ AWS Managed (you don't see these)                           │  │
│ │ ┌────────────────────────────────────────────────────────┐  │  │
│ │ │ EKS Control Plane                                     │  │  │
│ │ │ ├── 3 API servers (across 3 AZs) — HA!               │  │  │
│ │ │ ├── etcd (replicated across 3 AZs) — HA!             │  │  │
│ │ │ ├── Scheduler                                         │  │  │
│ │ │ └── Controller Manager                                │  │  │
│ │ │ Auto-scales, auto-patches, auto-upgrades (optional)  │  │  │
│ │ └────────────────────────────────────────────────────────┘  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                           │                                        │
│                    API (kubectl)                                   │
│                           │                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Your AWS Account (you manage these)                         │  │
│ │                                                              │  │
│ │ Option A: Managed Node Groups (EC2 instances)              │  │
│ │ ┌────────────────────────────────────────────────────────┐  │  │
│ │ │ Node Group: ng-general                                 │  │  │
│ │ │ ┌──────┐ ┌──────┐ ┌──────┐                           │  │  │
│ │ │ │ EC2  │ │ EC2  │ │ EC2  │  ← You choose size/count │  │  │
│ │ │ │Pod A │ │Pod B │ │Pod C │                           │  │  │
│ │ │ │Pod D │ │Pod E │ │Pod F │                           │  │  │
│ │ │ └──────┘ └──────┘ └──────┘                           │  │  │
│ │ └────────────────────────────────────────────────────────┘  │  │
│ │                                                              │  │
│ │ Option B: Fargate (serverless pods)                        │  │
│ │ ┌────────────────────────────────────────────────────────┐  │  │
│ │ │ Fargate Profile: fp-backend                            │  │  │
│ │ │ ┌─────┐ ┌─────┐ ┌─────┐                              │  │  │
│ │ │ │Pod G│ │Pod H│ │Pod I│  ← Each pod gets own VM     │  │  │
│ │ │ └─────┘ └─────┘ └─────┘    (micro-VM, AWS managed)  │  │  │
│ │ └────────────────────────────────────────────────────────┘  │  │
│ │                                                              │  │
│ │ Option C: Both! (mix node groups + Fargate)                │  │
│ │ Web pods → Node Group (consistent, GPU, etc.)             │  │
│ │ Batch pods → Fargate (scale to zero, no nodes to manage) │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pricing:                                                             │
│ ├── Control plane: $0.10/hr per cluster ($73/month)             │
│ ├── Worker nodes (EC2): Standard EC2 pricing                    │
│ ├── Fargate: Per vCPU/hr + per GB memory/hr (per pod)          │
│ └── Data transfer: Standard AWS rates                            │
│                                                                       │
│ ⚡ EKS vs ECS comparison:                                          │
│ ┌────────────────┬───────────────────┬───────────────────────┐   │
│ │ Feature         │ ECS                │ EKS                   │   │
│ ├────────────────┼───────────────────┼───────────────────────┤   │
│ │ Orchestrator    │ AWS proprietary    │ Kubernetes (standard) │   │
│ │ Learning curve  │ Easier             │ Steeper                │   │
│ │ Portability     │ AWS only           │ Multi-cloud ✅         │   │
│ │ Control plane   │ Free               │ $73/month             │   │
│ │ Ecosystem       │ AWS-native         │ Huge K8s ecosystem    │   │
│ │ Fargate support │ Yes                │ Yes                    │   │
│ │ Best for        │ AWS-only teams     │ K8s-experienced teams │   │
│ └────────────────┴───────────────────┴───────────────────────┘   │
│                                                                       │
│ ⚡ GCP equivalent: GKE (Google Kubernetes Engine)                  │
│ ⚡ Azure equivalent: AKS (Azure Kubernetes Service)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Cluster Creation (Full Console Walkthrough)

```
Console → EKS → Clusters → Create cluster

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 1: CONFIGURE CLUSTER                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [eks-prod]                                                    │
│ ⚡ Naming: eks-{env}  or  eks-{team}-{env}                         │
│                                                                       │
│ Kubernetes version: [1.30 ▼]                                       │
│ ⚡ Always use latest stable. AWS supports N-3 versions.            │
│   1.30, 1.29, 1.28, 1.27 (deprecated in ~12 months per version). │
│   ⚡ Extended support: AWS charges extra after version is          │
│   deprecated. Stay current!                                       │
│                                                                       │
│ Cluster service role: [eks-cluster-role ▼]                        │
│ ⚡ This IAM role allows EKS to manage AWS resources.              │
│                                                                       │
│   Trust policy:                                                     │
│   {                                                                  │
│     "Effect": "Allow",                                              │
│     "Principal": {"Service": "eks.amazonaws.com"},                 │
│     "Action": "sts:AssumeRole"                                     │
│   }                                                                  │
│   Managed policy: AmazonEKSClusterPolicy                          │
│                                                                       │
│ Cluster access:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Bootstrap cluster administrator access: ☑ Enabled          │  │
│ │ ⚡ Gives YOUR IAM user/role admin access to the cluster.    │  │
│ │   Without this: You create the cluster but can't access it!│  │
│ │                                                              │  │
│ │ Cluster authentication mode:                                │  │
│ │   ○ EKS API (recommended) ← Use AWS IAM for auth          │  │
│ │   ○ EKS API and ConfigMap (hybrid)                         │  │
│ │   ○ ConfigMap only (legacy — aws-auth configmap)           │  │
│ │                                                              │  │
│ │ ⚡ EKS API mode: Manage access via EKS Access Entries.      │  │
│ │   No more editing the fragile aws-auth ConfigMap!          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Tags: environment=prod, team=platform                              │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 2: SPECIFY NETWORKING                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ VPC: [vpc-prod ▼]                                                  │
│                                                                       │
│ Subnets:                                                             │
│ ☑ subnet-private-1a (10.0.1.0/24 - AZ a)                        │
│ ☑ subnet-private-1b (10.0.2.0/24 - AZ b)                        │
│ ☑ subnet-private-1c (10.0.3.0/24 - AZ c)                        │
│ ☑ subnet-public-1a  (10.0.101.0/24 - AZ a) ← for ALB           │
│ ☑ subnet-public-1b  (10.0.102.0/24 - AZ b) ← for ALB           │
│ ☑ subnet-public-1c  (10.0.103.0/24 - AZ c) ← for ALB           │
│                                                                       │
│ ⚡ SUBNET REQUIREMENTS:                                              │
│ ├── Worker nodes → Private subnets (with NAT gateway)           │
│ ├── Load Balancers → Public subnets                              │
│ ├── Must span ≥ 2 AZs (3 recommended)                          │
│ ├── Tag public subnets:                                          │
│ │   kubernetes.io/role/elb = 1                                  │
│ ├── Tag private subnets:                                         │
│ │   kubernetes.io/role/internal-elb = 1                         │
│ └── Tag all subnets:                                             │
│     kubernetes.io/cluster/eks-prod = shared                     │
│                                                                       │
│ Security groups:                                                     │
│ ├── EKS creates a cluster security group automatically          │
│ ├── Additional SGs: [sg-eks-additional ▼] (optional)           │
│ └── ⚡ The auto-created SG allows node↔control plane comms.      │
│                                                                       │
│ Cluster endpoint access:                                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ○ Public (kubectl from internet — NOT for production!)     │  │
│ │ ○ Public and Private (default — public for kubectl,        │  │
│ │   private for nodes)                                        │  │
│ │ ● Private (recommended — kubectl only from within VPC)    │  │
│ │                                                              │  │
│ │ ⚡ Private endpoint:                                         │  │
│ │   ├── Most secure. API only reachable from VPC.            │  │
│ │   ├── Need VPN/Direct Connect to run kubectl.              │  │
│ │   ├── Or use Cloud9/Bastion within VPC.                    │  │
│ │   └── Nodes always use private endpoint.                   │  │
│ │                                                              │  │
│ │ Public and Private with CIDR restriction:                   │  │
│ │   Advanced → Public access CIDRs: [203.0.113.0/24]       │  │
│ │   ⚡ "Public but only from my office IP"                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Kubernetes networking ──                                         │
│                                                                       │
│ CNI: Amazon VPC CNI (default — pods get real VPC IPs!)            │
│ ├── Every pod gets a VPC IP from the subnet CIDR                │
│ ├── Pods can talk to any VPC resource directly                  │
│ ├── ⚡ This means subnet CIDR must be large enough!              │
│ │   /24 = 256 IPs (might not be enough for many pods)          │
│ │   /20 = 4096 IPs (recommended for node subnets)              │
│ └── Prefix delegation: Assign /28 prefixes to nodes            │
│     (16 IPs per prefix — fits more pods per node)              │
│                                                                       │
│ Service IPv4 range: [10.100.0.0/16] (default)                    │
│ ⚡ CIDR for Kubernetes ClusterIP services. Can't overlap with    │
│   VPC CIDR. Default 10.100.0.0/16 is fine for most cases.      │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 3: CONFIGURE OBSERVABILITY                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Control plane logging:                                               │
│ ☑ API server        ← Logs all API calls                         │
│ ☑ Audit             ← Who did what? (security!)                  │
│ ☑ Authenticator     ← Authentication events                      │
│ ☐ Controller manager ← Usually noisy, enable if debugging       │
│ ☐ Scheduler          ← Usually noisy, enable if debugging       │
│                                                                       │
│ ⚡ Logs go to CloudWatch Logs group:                                │
│   /aws/eks/eks-prod/cluster                                       │
│   ⚠️ These logs cost money! API server + Audit most important.   │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 4: SELECT ADD-ONS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚡ Add-ons are pre-built Kubernetes components managed by AWS.      │
│   AWS keeps them updated. You don't manually install.             │
│                                                                       │
│ ☑ Amazon VPC CNI (networking — required!)                         │
│   Makes pods get VPC IP addresses.                                │
│                                                                       │
│ ☑ CoreDNS (DNS — required!)                                      │
│   Kubernetes internal DNS (service discovery).                    │
│   my-service.my-namespace.svc.cluster.local                      │
│                                                                       │
│ ☑ kube-proxy (networking — required!)                             │
│   Network rules for service routing on each node.                │
│                                                                       │
│ ☑ Amazon EBS CSI Driver (storage — recommended)                  │
│   Provisions EBS volumes for PersistentVolumeClaims.             │
│   Required for any stateful workload (databases, etc.)           │
│                                                                       │
│ ☐ Amazon EFS CSI Driver (shared storage)                         │
│   For ReadWriteMany volumes (shared across pods/nodes).          │
│                                                                       │
│ ☐ AWS Load Balancer Controller                                    │
│   Maps K8s Ingress/Service → ALB/NLB.                            │
│   ⚡ Install this for production! Either as add-on or Helm chart.│
│                                                                       │
│ ☐ Amazon CloudWatch Observability                                 │
│   Container Insights, Prometheus metrics, FluentBit logs.        │
│                                                                       │
│ ☐ AWS Mountpoint for S3 CSI Driver                               │
│   Mount S3 buckets as file systems in pods.                      │
│                                                                       │
│ ☐ Metrics Server                                                  │
│   For kubectl top, Horizontal Pod Autoscaler.                    │
│   ⚡ Required for HPA! Install this.                              │
│                                                                       │
│ [Next → Configure add-ons versions/roles]                         │
│                                                                       │
│ For each add-on:                                                    │
│ ├── Version: [Latest ▼]                                          │
│ ├── IAM role (for IRSA): [arn:aws:iam::role/vpc-cni-role ▼]    │
│ │   ⚡ Best practice: Give each add-on its own IRSA role.        │
│ └── Conflict resolution: [Overwrite ▼]                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 5: SECRETS ENCRYPTION (Optional)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Envelope encryption of Kubernetes secrets:                         │
│ ☑ Enable                                                           │
│ KMS key: [arn:aws:kms:...:key/abcd-1234 ▼]                      │
│                                                                       │
│ ⚡ By default, K8s secrets are base64-encoded (NOT encrypted!)    │
│   in etcd. Envelope encryption encrypts them with your KMS key. │
│   Recommended for production.                                    │
│                                                                       │
│ [Review + Create] → [Create]                                       │
│                                                                       │
│ ⚡ Cluster creation takes 10-15 minutes.                           │
│   Control plane spins up across 3 AZs.                           │
│   After creation: No worker nodes yet! Add node groups next.    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Managed Node Groups

```
Cluster → Compute → Node groups → Add node group

┌─────────────────────────────────────────────────────────────────────┐
│           NODE GROUP CONFIGURATION                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [ng-general]                                                  │
│ ⚡ Naming: ng-{purpose}  →  ng-general, ng-gpu, ng-spot           │
│                                                                       │
│ Node IAM role: [eks-node-role ▼]                                  │
│ ⚡ Role that EC2 instances assume. Needs these policies:           │
│   ├── AmazonEKSWorkerNodePolicy                                 │
│   ├── AmazonEKS_CNI_Policy (or use IRSA for VPC CNI)           │
│   ├── AmazonEC2ContainerRegistryReadOnly (pull from ECR)       │
│   └── AmazonSSMManagedInstanceCore (optional, for SSM access)  │
│                                                                       │
│ Launch template: [Use custom launch template ▼] (optional)       │
│ ⚡ Custom launch template lets you:                                 │
│   ├── Set custom AMI                                              │
│   ├── Add user data (bootstrap script)                           │
│   ├── Configure storage (EBS size, type, encryption)            │
│   ├── Add metadata options (IMDSv2 enforcement)                 │
│   └── Set security groups beyond EKS default                    │
│                                                                       │
│ Kubernetes labels:                                                   │
│ ├── role = general                                                │
│ ├── env = prod                                                    │
│ ⚡ Labels are used by pod nodeSelector/affinity to choose nodes.  │
│   Example: Deploy GPU pods only to ng-gpu nodes.                │
│                                                                       │
│ Kubernetes taints:                                                   │
│ ├── Key: gpu   Value: true   Effect: NoSchedule                 │
│ ⚡ Taints PREVENT pods from scheduling on these nodes             │
│   unless the pod has a matching toleration.                     │
│   Use for: GPU nodes, Spot nodes, dedicated workloads.         │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           NODE GROUP COMPUTE                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ AMI type: [Amazon Linux 2023 (AL2023_x86_64_STANDARD) ▼]        │
│   ○ AL2023_x86_64_STANDARD ← Default, recommended              │
│   ○ AL2_x86_64 ← Older Amazon Linux 2                           │
│   ○ BOTTLEROCKET_x86_64 ← Minimal container OS (secure!) ✅    │
│   ○ AL2023_ARM_64_STANDARD ← ARM/Graviton instances            │
│   ○ WINDOWS_CORE_2022_x86_64 ← Windows containers             │
│   ○ Custom AMI via launch template                               │
│                                                                       │
│ ⚡ Bottlerocket: AWS-built, minimal Linux for containers.          │
│   No package manager, no SSH by default = smaller attack surface.│
│   Recommended for security-conscious production clusters.       │
│                                                                       │
│ Capacity type:                                                       │
│   ● On-Demand                                                     │
│   ○ Spot ← Up to 90% savings, can be interrupted              │
│                                                                       │
│ Instance types: [t3.large ▼]                                      │
│ [Add instance type] → t3.xlarge, m5.large, m6i.large             │
│ ⚡ Add multiple types for Spot (more capacity pools = fewer      │
│   interruptions). Also good for on-demand for flexibility.      │
│                                                                       │
│ ⚡ How many pods per node?                                          │
│ ┌──────────────────┬──────────┬──────────────────────────────┐   │
│ │ Instance Type     │ Max Pods │ Why                           │   │
│ ├──────────────────┼──────────┼──────────────────────────────┤   │
│ │ t3.medium (2/4)   │ 17       │ Limited ENIs (3×6-1=17)    │   │
│ │ t3.large (2/8)    │ 35       │ 3 ENIs × 12 IPs           │   │
│ │ m5.large (2/8)    │ 29       │ 3 ENIs × 10 IPs           │   │
│ │ m5.xlarge (4/16)  │ 58       │ 4 ENIs × 15 IPs           │   │
│ │ m5.2xlarge (8/32) │ 58       │ 4 ENIs × 15 IPs           │   │
│ │ c5.2xlarge (8/16) │ 58       │ Same ENI config as m5.2xl │   │
│ └──────────────────┴──────────┴──────────────────────────────┘   │
│ ⚡ Max pods = (ENIs × IPs per ENI) - 1                            │
│   Enable prefix delegation for 110+ pods per node.              │
│                                                                       │
│ Disk size: [50] GiB (default 20, use 50-100 for production)     │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           NODE GROUP SCALING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Desired size: [3]                                                   │
│ Minimum size: [2]                                                   │
│ Maximum size: [10]                                                  │
│                                                                       │
│ ⚡ This is NODE count (not pod count!).                             │
│   Pod autoscaling is handled by HPA (Horizontal Pod Autoscaler).│
│   Node autoscaling is handled by:                                │
│   ├── Cluster Autoscaler (old) → modifies node group size      │
│   └── Karpenter (new, recommended) → provisions EC2 directly  │
│                                                                       │
│ Node group update config:                                           │
│ Max unavailable: [1] or [25%]                                     │
│ ⚡ During node group updates (AMI change, version upgrade),       │
│   how many nodes can be unavailable at once.                    │
│   1 = safe, one at a time.                                      │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           NODE GROUP NETWORKING                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subnets:                                                             │
│ ☑ subnet-private-1a (10.0.1.0/24)                                │
│ ☑ subnet-private-1b (10.0.2.0/24)                                │
│ ☑ subnet-private-1c (10.0.3.0/24)                                │
│                                                                       │
│ ⚡ ALWAYS use private subnets for nodes!                            │
│   Nodes don't need public IPs. They reach internet via NAT GW.  │
│   Public subnets = only for load balancers.                     │
│                                                                       │
│ SSH access to nodes:                                                │
│ Key pair: [eks-prod-key ▼]                                        │
│ Allow remote access from: [Security group: sg-bastion ▼]        │
│ ⚡ Production: Disable SSH. Use SSM Session Manager instead!     │
│                                                                       │
│ [Create]                                                             │
│ ⚡ Node group creation takes 3-5 minutes.                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Fargate Profiles

```
Cluster → Compute → Fargate profiles → Create Fargate profile

┌─────────────────────────────────────────────────────────────────────┐
│           FARGATE PROFILE                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [fp-backend]                                                  │
│                                                                       │
│ Pod execution role: [eks-fargate-role ▼]                          │
│ ⚡ IAM role that Fargate uses to pull images, send logs, etc.     │
│   Trust: eks-fargate-pods.amazonaws.com                           │
│   Policy: AmazonEKSFargatePodExecutionRolePolicy                │
│                                                                       │
│ Subnets:                                                             │
│ ☑ subnet-private-1a                                               │
│ ☑ subnet-private-1b                                               │
│ ☑ subnet-private-1c                                               │
│ ⚡ Fargate ONLY supports private subnets!                          │
│                                                                       │
│ Pod selectors (which pods run on Fargate):                         │
│                                                                       │
│ Selector 1:                                                         │
│   Namespace: [backend]                                             │
│   Labels:                                                           │
│     compute: fargate                                               │
│                                                                       │
│ Selector 2:                                                         │
│   Namespace: [batch-jobs]                                          │
│   (No labels — ALL pods in this namespace go to Fargate)          │
│                                                                       │
│ ⚡ Matching logic:                                                   │
│   Pod in namespace "backend" WITH label compute=fargate           │
│   → Runs on Fargate                                               │
│   Pod in namespace "backend" WITHOUT the label                    │
│   → Runs on node group (EC2)                                     │
│   Pod in namespace "batch-jobs" (any labels)                      │
│   → Runs on Fargate                                               │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           FARGATE vs NODE GROUP                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────────────┬─────────────────────┬─────────────────────┐   │
│ │ Feature          │ Managed Node Group   │ Fargate              │   │
│ ├─────────────────┼─────────────────────┼─────────────────────┤   │
│ │ You manage nodes │ Yes (EC2 fleet)      │ No (serverless) ✅  │   │
│ │ Scale to zero    │ No (min=1)           │ Yes ✅               │   │
│ │ Per-pod isolation│ No (shared node)     │ Yes (micro-VM) ✅   │   │
│ │ DaemonSets       │ Yes                  │ No ❌                │   │
│ │ GPU support      │ Yes ✅               │ No ❌                │   │
│ │ HostNetwork      │ Yes                  │ No ❌                │   │
│ │ EBS volumes      │ Yes                  │ Yes (20 GiB temp)  │   │
│ │ EFS volumes      │ Yes                  │ Yes ✅               │   │
│ │ Startup time     │ Node: 2-3 min        │ Pod: 30-60 sec     │   │
│ │ Cost model       │ EC2 hourly           │ Per vCPU/GB/second │   │
│ │ Best for         │ Always-on services   │ Burst/batch/cron   │   │
│ └─────────────────┴─────────────────────┴─────────────────────┘   │
│                                                                       │
│ ⚡ Fargate limitations:                                              │
│ ├── No DaemonSets (no node to run on!)                           │
│ │   → CoreDNS, FluentBit must be configured differently        │
│ ├── No GPU instances                                              │
│ ├── No privileged containers                                    │
│ ├── Max 4 vCPU, 30 GB memory per pod                            │
│ ├── No HostPort or HostNetwork                                  │
│ └── No persistent local storage (ephemeral only, 20 GiB)      │
│                                                                       │
│ ⚡ Common pattern: Use BOTH!                                        │
│   ├── Node group: Long-running web services, monitoring agents │
│   ├── Fargate: Batch jobs, cron jobs, low-traffic services     │
│   └── Selector routes pods to the right compute automatically  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: IRSA (IAM Roles for Service Accounts)

```
┌─────────────────────────────────────────────────────────────────────┐
│           IRSA — IAM ROLES FOR SERVICE ACCOUNTS                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: Pods need to access AWS services (S3, DynamoDB, SQS).   │
│                                                                       │
│ ❌ Bad: Attach permissions to the node IAM role.                   │
│   → ALL pods on that node get ALL permissions!                   │
│   Pod A (web) can access S3, DynamoDB, SQS, Secrets Manager... │
│   Even though it only needs DynamoDB.                            │
│                                                                       │
│ ✅ Good: IRSA — Each pod gets its OWN IAM role!                   │
│   → Pod A (web) → Role: DynamoDB read-only                      │
│   → Pod B (worker) → Role: SQS read, S3 write                  │
│   → Pod C (analytics) → Role: Athena query                     │
│   Least privilege per pod!                                       │
│                                                                       │
│ How IRSA works:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ 1. EKS cluster has an OIDC provider (identity provider)     │  │
│ │ 2. Create IAM role with trust policy for the OIDC provider  │  │
│ │ 3. Create K8s ServiceAccount, annotate with IAM role ARN    │  │
│ │ 4. Pod uses ServiceAccount → gets temporary AWS credentials│  │
│ │                                                              │  │
│ │ K8s ServiceAccount ←→ IAM Role (via OIDC federation)       │  │
│ │                                                              │  │
│ │  ┌──────────┐     OIDC      ┌──────────────┐              │  │
│ │  │ Pod with  │──────────────▶│ IAM Role     │              │  │
│ │  │ SA: app-sa│  federation  │ app-role     │              │  │
│ │  └──────────┘               │ → S3 read    │              │  │
│ │                              │ → DDB write  │              │  │
│ │                              └──────────────┘              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Setup steps:                                                         │
│                                                                       │
│ Step 1: Enable OIDC provider for cluster                           │
│ aws eks describe-cluster --name eks-prod \                        │
│   --query "cluster.identity.oidc.issuer"                         │
│ # Output: https://oidc.eks.ap-south-1.amazonaws.com/id/EXAMPL   │
│                                                                       │
│ eksctl utils associate-iam-oidc-provider \                        │
│   --cluster eks-prod --approve                                    │
│                                                                       │
│ Step 2: Create IAM role with OIDC trust                            │
│ Trust policy:                                                       │
│ {                                                                    │
│   "Effect": "Allow",                                                │
│   "Principal": {                                                    │
│     "Federated": "arn:aws:iam::oidc-provider/oidc.eks..."       │
│   },                                                                 │
│   "Action": "sts:AssumeRoleWithWebIdentity",                      │
│   "Condition": {                                                    │
│     "StringEquals": {                                               │
│       "oidc.eks...:sub":                                           │
│         "system:serviceaccount:backend:app-sa"                    │
│     }                                                                │
│   }                                                                  │
│ }                                                                    │
│ ⚡ Condition limits: ONLY pods using ServiceAccount "app-sa"       │
│   in namespace "backend" can assume this role.                   │
│                                                                       │
│ Step 3: Create Kubernetes ServiceAccount                            │
│ apiVersion: v1                                                      │
│ kind: ServiceAccount                                                │
│ metadata:                                                            │
│   name: app-sa                                                      │
│   namespace: backend                                                │
│   annotations:                                                      │
│     eks.amazonaws.com/role-arn:                                    │
│       arn:aws:iam::role/app-role                                  │
│                                                                       │
│ Step 4: Use in pod/deployment                                       │
│ spec:                                                                │
│   serviceAccountName: app-sa   ← This pod gets app-role creds   │
│                                                                       │
│ ⚡ Newer alternative: EKS Pod Identity (simpler setup!)            │
│   No OIDC trust policy needed. Just associate SA↔Role in EKS.  │
│   Console: Cluster → Access → Pod Identity associations          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: EKS Add-ons Deep Dive

```
┌─────────────────────────────────────────────────────────────────────┐
│           KEY ADD-ONS                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. VPC CNI (amazon-vpc-cni-k8s)                                   │
│    ├── Assigns VPC IPs to pods                                    │
│    ├── IRSA role: AmazonEKS_CNI_Policy                           │
│    ├── Config: ENABLE_PREFIX_DELEGATION=true (more pods/node)   │
│    ├── Config: WARM_IP_TARGET=5 (pre-allocate 5 IPs)           │
│    └── ⚡ Essential. Without this, pods can't get IPs.            │
│                                                                       │
│ 2. CoreDNS                                                          │
│    ├── Kubernetes DNS server                                      │
│    ├── Resolves: my-svc.my-ns.svc.cluster.local                │
│    ├── Default: 2 replicas (HA)                                  │
│    └── ⚡ Scale CoreDNS replicas for large clusters (100+ nodes)│
│                                                                       │
│ 3. kube-proxy                                                       │
│    ├── Manages iptables/IPVS rules for service routing          │
│    ├── Runs as DaemonSet on every node                          │
│    └── ⚡ Switch to IPVS mode for 1000+ services                 │
│                                                                       │
│ 4. EBS CSI Driver                                                   │
│    ├── Provisions EBS volumes for PersistentVolumeClaims        │
│    ├── IRSA role: AmazonEBSCSIDriverPolicy                     │
│    ├── StorageClass: gp3 (default), io2 (high IOPS)            │
│    └── ⚡ Required for any stateful workload                      │
│                                                                       │
│ 5. AWS Load Balancer Controller (install via Helm or add-on)     │
│    ├── Maps K8s Ingress → ALB                                    │
│    │   annotations:                                               │
│    │     kubernetes.io/ingress.class: alb                        │
│    │     alb.ingress.kubernetes.io/scheme: internet-facing      │
│    │     alb.ingress.kubernetes.io/target-type: ip              │
│    ├── Maps K8s Service type: LoadBalancer → NLB                │
│    │   annotations:                                               │
│    │     service.beta.kubernetes.io/aws-load-balancer-type:     │
│    │       external                                               │
│    │     service.beta.kubernetes.io/                              │
│    │       aws-load-balancer-nlb-target-type: ip                │
│    └── ⚡ CRITICAL for production. Without it, K8s creates       │
│        Classic LBs (deprecated and limited).                    │
│                                                                       │
│ 6. Cluster Autoscaler / Karpenter                                  │
│    ├── Cluster Autoscaler (legacy):                               │
│    │   Watches pending pods → scales node groups up/down        │
│    │   Config: node group min/max size                           │
│    │   Slow: ~2-5 min to add a node                             │
│    │                                                               │
│    ├── Karpenter (recommended for new clusters):                 │
│    │   Provisions RIGHT-SIZED EC2 directly (no node groups!)   │
│    │   Fast: ~30-60 sec to provision                            │
│    │   Auto-selects instance type based on pod requirements    │
│    │   Consolidation: Moves pods to fewer nodes (saves cost)  │
│    │   NodePool CRD defines constraints:                        │
│    │   apiVersion: karpenter.sh/v1                              │
│    │   kind: NodePool                                            │
│    │   spec:                                                     │
│    │     template:                                               │
│    │       spec:                                                 │
│    │         requirements:                                       │
│    │           - key: karpenter.sh/capacity-type                │
│    │             operator: In                                    │
│    │             values: ["on-demand", "spot"]                  │
│    │           - key: kubernetes.io/arch                        │
│    │             operator: In                                    │
│    │             values: ["amd64", "arm64"]                     │
│    │           - key: node.kubernetes.io/instance-type          │
│    │             operator: In                                    │
│    │             values: ["m5.large","m5.xlarge","m6i.large"]  │
│    │     limits:                                                 │
│    │       cpu: "100"    # Max 100 vCPUs total                 │
│    │       memory: 400Gi # Max 400 GiB total                   │
│    │     disruption:                                             │
│    │       consolidationPolicy: WhenEmptyOrUnderutilized       │
│    │       consolidateAfter: 30s                                │
│    └── ⚡ Karpenter is AWS-recommended for EKS (2024+).          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: EKS Access Management

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACCESS ENTRIES (new way)                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cluster → Access → IAM access entries                              │
│                                                                       │
│ [Create access entry]                                               │
│ IAM principal ARN: [arn:aws:iam::role/dev-team-role ▼]           │
│ Type: Standard                                                      │
│ Username: (auto-generated)                                         │
│                                                                       │
│ Access policies:                                                     │
│ ├── AmazonEKSClusterAdminPolicy → Full cluster admin            │
│ ├── AmazonEKSAdminPolicy → Admin (non-cluster-level)           │
│ ├── AmazonEKSEditPolicy → Edit resources (not RBAC)            │
│ └── AmazonEKSViewPolicy → Read-only                             │
│                                                                       │
│ Access scope:                                                        │
│ ● Cluster (all namespaces)                                        │
│ ○ Namespace: [backend, frontend] (specific namespaces)           │
│                                                                       │
│ ⚡ Access entries replace the old aws-auth ConfigMap:              │
│   Old way: kubectl edit cm aws-auth -n kube-system               │
│            (fragile, typo = locked out of cluster!)              │
│   New way: EKS API access entries (safe, IAM-managed) ✅         │
│                                                                       │
│ Example team setup:                                                  │
│ ├── Platform team: ClusterAdmin (full access)                   │
│ ├── Backend devs: Edit policy, scope: backend namespace         │
│ ├── Frontend devs: Edit policy, scope: frontend namespace      │
│ ├── QA team: View policy, scope: staging namespace              │
│ └── CI/CD role: Edit policy, scope: all namespaces              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform

```hcl
# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "eks-prod"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.30"

  vpc_config {
    subnet_ids = concat(
      aws_subnet.private[*].id,
      aws_subnet.public[*].id
    )
    endpoint_private_access = true
    endpoint_public_access  = false  # Private only
    security_group_ids      = [aws_security_group.eks_additional.id]
  }

  access_config {
    authentication_mode                         = "API"
    bootstrap_cluster_creator_admin_permissions = true
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator"]

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  tags = {
    Environment = "prod"
    Team        = "platform"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
  ]
}

# Cluster IAM Role
resource "aws_iam_role" "eks_cluster" {
  name = "eks-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster.name
}

# Node IAM Role
resource "aws_iam_role" "eks_nodes" {
  name = "eks-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "node_policies" {
  for_each = toset([
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
    "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore",
  ])

  policy_arn = each.value
  role       = aws_iam_role.eks_nodes.name
}

# Managed Node Group — General
resource "aws_eks_node_group" "general" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "ng-general"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id

  ami_type       = "AL2023_x86_64_STANDARD"
  capacity_type  = "ON_DEMAND"
  instance_types = ["m6i.large", "m5.large"]
  disk_size      = 50

  scaling_config {
    desired_size = 3
    min_size     = 2
    max_size     = 10
  }

  update_config {
    max_unavailable = 1
  }

  labels = {
    role = "general"
    env  = "prod"
  }

  tags = {
    Environment = "prod"
  }

  depends_on = [
    aws_iam_role_policy_attachment.node_policies,
  ]
}

# Managed Node Group — Spot (for batch/non-critical)
resource "aws_eks_node_group" "spot" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "ng-spot"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id

  ami_type       = "AL2023_x86_64_STANDARD"
  capacity_type  = "SPOT"
  instance_types = ["m5.large", "m5.xlarge", "m6i.large", "m6i.xlarge",
                    "c5.large", "c5.xlarge", "r5.large"]
  disk_size      = 50

  scaling_config {
    desired_size = 2
    min_size     = 0
    max_size     = 20
  }

  labels = {
    role = "spot"
  }

  taint {
    key    = "spot"
    value  = "true"
    effect = "NO_SCHEDULE"
  }

  tags = {
    Environment = "prod"
  }
}

# Fargate Profile
resource "aws_eks_fargate_profile" "batch" {
  cluster_name           = aws_eks_cluster.main.name
  fargate_profile_name   = "fp-batch"
  pod_execution_role_arn = aws_iam_role.fargate.arn
  subnet_ids             = aws_subnet.private[*].id

  selector {
    namespace = "batch-jobs"
  }

  selector {
    namespace = "backend"
    labels = {
      compute = "fargate"
    }
  }
}

resource "aws_iam_role" "fargate" {
  name = "eks-fargate-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks-fargate-pods.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "fargate" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy"
  role       = aws_iam_role.fargate.name
}

# EKS Add-ons
resource "aws_eks_addon" "vpc_cni" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "vpc-cni"
  addon_version = "v1.18.5-eksbuild.1"
  resolve_conflicts_on_update = "OVERWRITE"
}

resource "aws_eks_addon" "coredns" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "coredns"
  resolve_conflicts_on_update = "OVERWRITE"
}

resource "aws_eks_addon" "kube_proxy" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "kube-proxy"
  resolve_conflicts_on_update = "OVERWRITE"
}

resource "aws_eks_addon" "ebs_csi" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "aws-ebs-csi-driver"
  service_account_role_arn = aws_iam_role.ebs_csi.arn
  resolve_conflicts_on_update = "OVERWRITE"
}

# OIDC Provider for IRSA
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

# IRSA example — EBS CSI Driver Role
resource "aws_iam_role" "ebs_csi" {
  name = "eks-ebs-csi-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.eks.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" =
            "system:serviceaccount:kube-system:ebs-csi-controller-sa"
          "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" =
            "sts.amazonaws.com"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ebs_csi" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  role       = aws_iam_role.ebs_csi.name
}

# Access Entry for dev team
resource "aws_eks_access_entry" "dev_team" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = "arn:aws:iam::role/dev-team-role"
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "dev_team" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = "arn:aws:iam::role/dev-team-role"
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy"

  access_scope {
    type       = "namespace"
    namespaces = ["backend", "frontend"]
  }
}
```

---

## Part 10: eksctl & kubectl CLI Reference

```bash
# ═══ eksctl — EKS cluster management ═══

# Create cluster (quick start — creates VPC, subnets, node group)
eksctl create cluster \
  --name eks-prod \
  --region ap-south-1 \
  --version 1.30 \
  --nodegroup-name ng-general \
  --node-type m6i.large \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed \
  --with-oidc

# Create cluster from config file (recommended)
# cluster-config.yaml:
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: eks-prod
  region: ap-south-1
  version: "1.30"
vpc:
  clusterEndpoints:
    privateAccess: true
    publicAccess: false
managedNodeGroups:
  - name: ng-general
    instanceType: m6i.large
    desiredCapacity: 3
    minSize: 2
    maxSize: 10
    privateNetworking: true
    iam:
      withAddonPolicies:
        ebs: true
        efs: true
        albIngress: true
        cloudWatch: true
    labels:
      role: general
  - name: ng-spot
    instanceTypes: ["m5.large", "m5.xlarge", "c5.large"]
    spot: true
    desiredCapacity: 2
    minSize: 0
    maxSize: 20
    privateNetworking: true
    labels:
      role: spot
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
fargateProfiles:
  - name: fp-batch
    selectors:
      - namespace: batch-jobs
iam:
  withOIDC: true
addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: aws-ebs-csi-driver
cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator"]

eksctl create cluster -f cluster-config.yaml

# Add node group
eksctl create nodegroup \
  --cluster eks-prod \
  --name ng-gpu \
  --node-type p3.2xlarge \
  --nodes 1 \
  --nodes-min 0 \
  --nodes-max 4

# Scale node group
eksctl scale nodegroup \
  --cluster eks-prod \
  --name ng-general \
  --nodes 5 \
  --nodes-min 3

# Upgrade cluster version
eksctl upgrade cluster --name eks-prod --version 1.31 --approve

# Upgrade node group (new AMI)
eksctl upgrade nodegroup \
  --cluster eks-prod \
  --name ng-general

# Create IRSA role
eksctl create iamserviceaccount \
  --name app-sa \
  --namespace backend \
  --cluster eks-prod \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess \
  --approve

# Delete cluster
eksctl delete cluster --name eks-prod

# ═══ kubectl — Kubernetes resource management ═══

# Configure kubectl for EKS
aws eks update-kubeconfig --name eks-prod --region ap-south-1

# Verify connection
kubectl get nodes
kubectl get pods -A  # All namespaces

# Deploy an application
kubectl create namespace backend
kubectl apply -f deployment.yaml -n backend

# Example deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      serviceAccountName: app-sa  # IRSA
      containers:
        - name: web
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/web-app:v1.2
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: host

---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: backend
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 8080

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app
  namespace: backend
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...:certificate/abc
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-app
                port:
                  number: 80

# Common kubectl commands
kubectl get pods -n backend                     # List pods
kubectl describe pod web-app-xyz -n backend     # Pod details
kubectl logs web-app-xyz -n backend -f          # Stream logs
kubectl exec -it web-app-xyz -n backend -- bash # Shell into pod
kubectl top pods -n backend                     # CPU/memory usage
kubectl rollout status deployment/web-app -n backend
kubectl rollout undo deployment/web-app -n backend   # Rollback
kubectl scale deployment/web-app --replicas=5 -n backend

# HPA (Horizontal Pod Autoscaler)
kubectl autoscale deployment web-app \
  -n backend \
  --cpu-percent=70 \
  --min=3 \
  --max=20

# Check HPA
kubectl get hpa -n backend
```

---

## Part 11: Real-World Patterns

### Startup

```
EKS for a small team (5-10 devs):

Cluster: eks-prod
├── Version: 1.30 (latest stable)
├── Endpoint: Public + Private (restrict public to office IP)
├── Logging: API server + Audit
├── Auth: EKS API mode
└── Cost: $73/month (control plane)

Node group: ng-general
├── Instance: t3.large (2 vCPU, 8 GiB) — burstable
├── Nodes: min 2, max 6, desired 3
├── Capacity: On-Demand
├── Disk: 50 GiB
└── Cost: ~$180/month (3× t3.large)

Workloads:
├── Namespace: frontend (React SSR, 2 replicas)
├── Namespace: backend (API, 3 replicas)
├── Namespace: workers (queue consumers, 2 replicas)
└── HPA: CPU > 70% → scale pods

Add-ons: VPC CNI, CoreDNS, kube-proxy, EBS CSI
Ingress: ALB Ingress Controller → ALB
Autoscaler: Karpenter (simple NodePool)

Total: ~$300/month
```

### Mid-Size

```
EKS multi-tier (50-100 devs, 10+ microservices):

Cluster: eks-prod
├── Private endpoint only (VPN access for kubectl)
├── K8s 1.30, extended support plan
├── All logging enabled
├── Secrets encryption: KMS
├── Auth: EKS API + RBAC per team
│   ├── platform-team → ClusterAdmin
│   ├── backend-devs → Edit (backend namespace)
│   ├── frontend-devs → Edit (frontend namespace)
│   └── ci-cd-role → Edit (all namespaces)
└── Cost: $73/month

Compute:
├── ng-general (m6i.large, 3-15 On-Demand)
│   Labels: role=general
│   For: Web services, APIs
│
├── ng-spot (m5/m6i/c5 mix, 0-30 Spot)
│   Labels: role=spot, Taint: spot=true:NoSchedule
│   For: Batch processing, non-critical workers
│
├── fp-cron (Fargate profile, namespace: cron-jobs)
│   For: Scheduled jobs, scale-to-zero
│
└── Karpenter NodePool (fallback, any workload)
    Consolidation: WhenEmptyOrUnderutilized

Namespaces:
├── frontend (SSR, CDN origin, 3-10 pods)
├── backend (API gateway + microservices, 10-30 pods)
├── workers (queue consumers, 5-20 pods)
├── cron-jobs (Fargate, scheduled batch)
├── monitoring (Prometheus, Grafana)
└── ingress (ALB controller, external-dns)

Networking:
├── ALB Ingress: app.company.com → frontend
├── ALB Ingress: api.company.com → backend
├── Internal NLB: gRPC between services
├── NetworkPolicies: Namespace isolation
└── External DNS: Auto-manage Route 53 records

IRSA per service:
├── web-app-sa → S3 read (static assets)
├── api-sa → DynamoDB, SQS, Secrets Manager
├── worker-sa → SQS, S3, SES
└── cron-sa → S3, Athena

CI/CD:
├── GitHub Actions → Build image → Push ECR
├── ArgoCD: GitOps deployments
├── Image: 123456.dkr.ecr.region.amazonaws.com/service:sha
└── Rollout: Argo Rollouts (canary, 10%→50%→100%)

Monitoring:
├── Prometheus + Grafana (in-cluster)
├── Container Insights (CloudWatch)
├── Alerts: PagerDuty for pod crashes, node NotReady
└── Logs: FluentBit → CloudWatch Logs / OpenSearch

Cost: ~$2,000-4,000/month (with Spot savings)
```

### Enterprise

```
Multi-cluster, multi-region EKS platform:

Region 1 (ap-south-1):
├── eks-prod-web (customer-facing)
│   ├── ng-web (m6i.2xlarge, 10-50, On-Demand)
│   ├── ng-api (m6i.xlarge, 10-40, On-Demand)
│   └── Karpenter: Auto-provision remaining
│
├── eks-prod-data (data pipeline)
│   ├── ng-spark (r6i.2xlarge, 0-30, Spot)
│   └── Fargate: Airflow tasks
│
└── eks-prod-ml (ML inference)
    └── ng-gpu (p3.2xlarge, 2-8, On-Demand)

Region 2 (eu-west-1):
├── eks-prod-web-eu (same as Region 1)
└── Global Accelerator: Route traffic to nearest region

Platform standards:
├── Kubernetes: 1.30 (all clusters on same version)
├── OS: Bottlerocket (minimal, hardened)
├── Images: ECR with scanning (block critical CVEs)
├── Secrets: External Secrets Operator → AWS Secrets Manager
├── Mesh: Istio service mesh (mTLS, traffic management)
├── Policy: OPA Gatekeeper (enforce resource limits, labels,
│   no privileged containers, approved registries only)
├── GitOps: ArgoCD (app-of-apps pattern)
└── Backup: Velero (cluster state backup to S3)

Security:
├── Private endpoints only
├── Pod Security Standards: Restricted
├── Network Policies: Calico (default deny, allow per service)
├── IRSA everywhere (no node-level permissions)
├── Image signing: Cosign + admission controller
├── Runtime security: Falco (detect anomalous behavior)
├── Secrets: Never in ConfigMaps, always External Secrets
├── Audit: CloudTrail + K8s audit logs → SIEM
└── Compliance: SOC 2, ISO 27001, HIPAA

Upgrade strategy:
├── Test cluster: Upgrade first, run e2e tests
├── Staging: Upgrade, soak for 1 week
├── Prod: Blue-green cluster upgrade
│   New cluster (1.31) → migrate workloads → retire old (1.30)
├── Node groups: Rolling update, 1 node at a time
└── Add-ons: Upgrade after cluster, test compatibility

Cost: $20,000-80,000/month (with savings plans + Spot)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Managed Kubernetes control plane |
| Control plane | AWS-managed, 3 AZs, auto-patched |
| Control plane cost | $0.10/hr ($73/month) per cluster |
| Worker options | Managed Node Groups (EC2) or Fargate (serverless) |
| Networking | VPC CNI (pods get VPC IPs) |
| Auth | EKS API access entries (or legacy aws-auth ConfigMap) |
| Pod IAM | IRSA (OIDC) or EKS Pod Identity |
| Node scaling | Karpenter (recommended) or Cluster Autoscaler |
| Pod scaling | HPA (CPU/memory) or KEDA (event-driven) |
| Add-ons | VPC CNI, CoreDNS, kube-proxy, EBS CSI, LB Controller |
| Versions | Latest minus 3 supported (extended support = extra cost) |
| GCP equivalent | GKE (Google Kubernetes Engine) |
| Azure equivalent | AKS (Azure Kubernetes Service) |

---

## What's Next?

In the next chapter, we'll cover AWS Elastic Beanstalk — a PaaS that manages infrastructure for you.

→ Next: [Chapter 18: Elastic Beanstalk](18-elastic-beanstalk.md)

---

*Last Updated: May 2026*
