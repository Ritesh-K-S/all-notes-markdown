# 🚀 DevOps Mastery — From Zero to Beyond

> **Goal:** Become the world's best DevOps Engineer — mastering every layer from fundamentals to cutting-edge practices.

---

## How to Use This Index

- Each **main topic** is a major pillar of DevOps.
- Each **subtopic** will have its own dedicated notes file (we'll create them as we go).
- Topics are ordered in a **learning path** — start from the top and work your way down.
- `[Beginner]` `[Intermediate]` `[Advanced]` `[Expert]` tags indicate depth level.

---

## Table of Contents

1. [DevOps Fundamentals](#1-devops-fundamentals)
2. [Linux & Operating Systems](#2-linux--operating-systems)
3. [Networking Fundamentals](#3-networking-fundamentals)
4. [Version Control — Git & GitHub/GitLab](#4-version-control--git--githubgitlab)
5. [Programming & Scripting](#5-programming--scripting)
6. [Build Tools & Package Managers](#6-build-tools--package-managers)
7. [Databases & Data Management](#7-databases--data-management)
8. [Web Servers & Reverse Proxies](#8-web-servers--reverse-proxies)
9. [Containers — Docker](#9-containers--docker)
10. [Container Orchestration — Kubernetes](#10-container-orchestration--kubernetes)
11. [CI/CD — Continuous Integration & Continuous Delivery](#11-cicd--continuous-integration--continuous-delivery)
12. [Infrastructure as Code (IaC)](#12-infrastructure-as-code-iac)
13. [Configuration Management](#13-configuration-management)
14. [Cloud Platforms](#14-cloud-platforms)
15. [Monitoring, Logging & Observability](#15-monitoring-logging--observability)
16. [Security & DevSecOps](#16-security--devsecops)
17. [GitOps](#17-gitops)
18. [Service Mesh & Networking (Advanced)](#18-service-mesh--networking-advanced)
19. [Serverless & FaaS](#19-serverless--faas)
20. [Artifact Management & Registries](#20-artifact-management--registries)
21. [Secrets Management](#21-secrets-management)
22. [Chaos Engineering & Reliability](#22-chaos-engineering--reliability)
23. [Platform Engineering & Internal Developer Platforms](#23-platform-engineering--internal-developer-platforms)
24. [FinOps — Cloud Cost Optimization](#24-finops--cloud-cost-optimization)
25. [AI/ML Ops](#25-aiml-ops)
26. [Soft Skills & DevOps Culture](#26-soft-skills--devops-culture)
27. [Interview Preparation & Real-World Scenarios](#27-interview-preparation--real-world-scenarios)
28. [Projects & Hands-On Labs](#28-projects--hands-on-labs)

---

## 1. DevOps Fundamentals

> *Understand the WHY before the HOW.*

| # | Subtopic | Level |
|---|----------|-------|
| 1.1 | What is DevOps? — History, Evolution & Philosophy | Beginner |
| 1.2 | DevOps vs Traditional IT (Waterfall, Agile, SRE) | Beginner |
| 1.3 | DevOps Lifecycle — Plan, Code, Build, Test, Release, Deploy, Operate, Monitor | Beginner |
| 1.4 | Key Principles — CI/CD, IaC, Monitoring, Collaboration, Automation | Beginner |
| 1.5 | DevOps Culture — Breaking Silos, Blameless Postmortems, Shared Ownership | Beginner |
| 1.6 | CALMS Framework (Culture, Automation, Lean, Measurement, Sharing) | Beginner |
| 1.7 | Three Ways of DevOps (Flow, Feedback, Continuous Learning) | Beginner |
| 1.8 | SRE vs DevOps vs Platform Engineering — Differences & Overlap | Intermediate |
| 1.9 | DevOps Maturity Model — Where Are You? | Intermediate |
| 1.10 | Recommended Books & Resources | Beginner |

---

## 2. Linux & Operating Systems

> *Linux is the backbone of DevOps. Master it.*

| # | Subtopic | Level |
|---|----------|-------|
| 2.1 | Introduction to Linux — Distributions (Ubuntu, CentOS, RHEL, Alpine) | Beginner |
| 2.2 | Linux Installation & Virtual Machines (VirtualBox, VMware, WSL2) | Beginner |
| 2.3 | File System Hierarchy — /, /etc, /var, /home, /opt, /tmp | Beginner |
| 2.4 | Essential Commands — ls, cd, cp, mv, rm, find, grep, awk, sed, cut, sort, uniq | Beginner |
| 2.5 | File Permissions & Ownership — chmod, chown, umask, ACLs | Beginner |
| 2.6 | Users & Groups Management | Beginner |
| 2.7 | Process Management — ps, top, htop, kill, systemctl, journalctl | Beginner |
| 2.8 | Package Management — apt, yum, dnf, snap, rpm, dpkg | Beginner |
| 2.9 | Disk Management — df, du, lsblk, mount, fdisk, LVM | Intermediate |
| 2.10 | Networking Commands — ip, ifconfig, netstat, ss, nslookup, dig, curl, wget, traceroute | Intermediate |
| 2.11 | SSH — Keys, Config, Tunneling, SCP, SFTP | Intermediate |
| 2.12 | Shell Scripting — Bash Fundamentals (variables, loops, conditions, functions) | Intermediate |
| 2.13 | Advanced Bash — Arrays, Traps, Signal Handling, Debugging Scripts | Advanced |
| 2.14 | Cron Jobs & Task Scheduling (crontab, at, systemd timers) | Intermediate |
| 2.15 | System Performance Tuning — sysctl, ulimit, nice, iostat, vmstat | Advanced |
| 2.16 | Kernel Basics — Modules, Parameters, Boot Process (GRUB, initramfs) | Advanced |
| 2.17 | SELinux & AppArmor | Advanced |
| 2.18 | Linux Namespaces & Cgroups (Foundation of Containers) | Advanced |
| 2.19 | Systemd Deep Dive — Units, Targets, Sockets, Timers | Advanced |
| 2.20 | Troubleshooting & Debugging Linux Systems | Intermediate |

---

## 3. Networking Fundamentals

> *You can't deploy what you can't connect.*

| # | Subtopic | Level |
|---|----------|-------|
| 3.1 | OSI Model & TCP/IP Model — Every Layer Explained | Beginner |
| 3.2 | IP Addressing — IPv4, IPv6, Subnetting, CIDR | Beginner |
| 3.3 | DNS — How It Works, Records (A, AAAA, CNAME, MX, TXT, NS, SOA) | Beginner |
| 3.4 | HTTP/HTTPS — Methods, Status Codes, Headers, TLS/SSL Handshake | Beginner |
| 3.5 | TCP vs UDP — Deep Dive | Beginner |
| 3.6 | Ports, Sockets & Firewalls (iptables, ufw, firewalld, nftables) | Intermediate |
| 3.7 | Load Balancing — L4 vs L7, Algorithms (Round Robin, Least Conn, IP Hash) | Intermediate |
| 3.8 | Proxy vs Reverse Proxy vs Forward Proxy | Intermediate |
| 3.9 | VPN, VPC, Subnets, NAT, Internet Gateways | Intermediate |
| 3.10 | CDN — Content Delivery Networks (CloudFront, Cloudflare, Akamai) | Intermediate |
| 3.11 | Network Troubleshooting — ping, traceroute, mtr, tcpdump, Wireshark | Advanced |
| 3.12 | Software Defined Networking (SDN) | Advanced |
| 3.13 | Network Policies in Kubernetes | Advanced |
| 3.14 | BGP, OSPF & Routing Fundamentals | Expert |

---

## 4. Version Control — Git & GitHub/GitLab

> *If it's not in Git, it doesn't exist.*

| # | Subtopic | Level |
|---|----------|-------|
| 4.1 | What is Version Control? — Why Git? | Beginner |
| 4.2 | Git Installation & Configuration | Beginner |
| 4.3 | Git Basics — init, add, commit, status, log, diff | Beginner |
| 4.4 | Branching & Merging — branch, checkout, switch, merge | Beginner |
| 4.5 | Remote Repositories — clone, push, pull, fetch, remote | Beginner |
| 4.6 | Git Workflows — Feature Branch, Git Flow, Trunk-Based Development | Intermediate |
| 4.7 | Pull Requests / Merge Requests — Code Reviews Best Practices | Intermediate |
| 4.8 | Rebasing vs Merging — When to Use What | Intermediate |
| 4.9 | Git Stash, Cherry-Pick, Bisect, Reflog | Intermediate |
| 4.10 | Tags & Releases — Semantic Versioning | Intermediate |
| 4.11 | .gitignore, .gitattributes, Git Hooks (pre-commit, pre-push) | Intermediate |
| 4.12 | Git Submodules & Monorepos | Advanced |
| 4.13 | Signed Commits & GPG Keys | Advanced |
| 4.14 | GitHub Actions / GitLab CI Basics (Intro — detailed in CI/CD section) | Intermediate |
| 4.15 | Advanced Git Internals — Objects, Refs, Pack Files | Expert |

---

## 5. Programming & Scripting

> *Automate everything. Script first, click never.*

| # | Subtopic | Level |
|---|----------|-------|
| 5.1 | Bash Scripting — Complete Guide (linked from Linux section) | Beginner |
| 5.2 | Python for DevOps — Basics (variables, data types, loops, functions) | Beginner |
| 5.3 | Python for DevOps — File Handling, OS Module, Subprocess | Intermediate |
| 5.4 | Python for DevOps — Working with APIs (requests, boto3, kubernetes client) | Intermediate |
| 5.5 | Python for DevOps — Automation Scripts & Real-World Examples | Advanced |
| 5.6 | Go (Golang) for DevOps — Why Go? Basics & CLI Tools | Intermediate |
| 5.7 | Go for DevOps — Building Custom Operators & Tools | Advanced |
| 5.8 | YAML & JSON — Mastering Configuration Formats | Beginner |
| 5.9 | Jinja2 Templating | Intermediate |
| 5.10 | Regular Expressions (Regex) | Intermediate |
| 5.11 | PowerShell Fundamentals (for Windows environments) | Intermediate |

---

## 6. Build Tools & Package Managers

> *Understand how applications are built before you deploy them.*

| # | Subtopic | Level |
|---|----------|-------|
| 6.1 | What is a Build? — Compilation, Linking, Packaging | Beginner |
| 6.2 | Maven & Gradle (Java) | Intermediate |
| 6.3 | npm, yarn, pnpm (Node.js) | Intermediate |
| 6.4 | pip, poetry, pipenv (Python) | Intermediate |
| 6.5 | Make & Makefiles | Intermediate |
| 6.6 | Semantic Versioning & Dependency Management | Intermediate |
| 6.7 | Build Optimization & Caching Strategies | Advanced |
| 6.8 | Multi-Stage Builds (linked to Docker section) | Advanced |

---

## 7. Databases & Data Management

> *Every app has data. Know where it lives and how to manage it.*

| # | Subtopic | Level |
|---|----------|-------|
| 7.1 | Relational Databases — MySQL, PostgreSQL (Basics, Queries, Indexing) | Beginner |
| 7.2 | NoSQL Databases — MongoDB, Redis, DynamoDB, Cassandra | Intermediate |
| 7.3 | Database Backup & Restore Strategies | Intermediate |
| 7.4 | Database Replication — Master-Slave, Master-Master | Intermediate |
| 7.5 | Database Migration Tools — Flyway, Liquibase | Intermediate |
| 7.6 | Database in Containers & Kubernetes (StatefulSets, Operators) | Advanced |
| 7.7 | Connection Pooling & Performance Tuning | Advanced |
| 7.8 | Message Queues — RabbitMQ, Apache Kafka, SQS | Advanced |

---

## 8. Web Servers & Reverse Proxies

> *The gatekeepers of your applications.*

| # | Subtopic | Level |
|---|----------|-------|
| 8.1 | How Web Servers Work — Request/Response Lifecycle | Beginner |
| 8.2 | Nginx — Installation, Configuration, Virtual Hosts | Beginner |
| 8.3 | Nginx — Reverse Proxy, Load Balancing, SSL Termination | Intermediate |
| 8.4 | Nginx — Rate Limiting, Caching, Security Headers | Advanced |
| 8.5 | Apache HTTP Server — Basics & Configuration | Intermediate |
| 8.6 | HAProxy — Advanced Load Balancing | Advanced |
| 8.7 | Traefik — Cloud-Native Reverse Proxy & Ingress | Advanced |
| 8.8 | Caddy — Auto-HTTPS Web Server | Intermediate |

---

## 9. Containers — Docker

> *Containers changed everything. Docker is where it starts.*

| # | Subtopic | Level |
|---|----------|-------|
| 9.1 | What are Containers? — VMs vs Containers, OCI Standards | Beginner |
| 9.2 | Docker Architecture — Daemon, Client, Registry, Images, Containers | Beginner |
| 9.3 | Docker Installation (Linux, Mac, Windows, WSL2) | Beginner |
| 9.4 | Docker CLI — run, exec, ps, logs, inspect, stop, rm, images | Beginner |
| 9.5 | Docker Images — Pulling, Building, Tagging, Pushing | Beginner |
| 9.6 | Dockerfile — FROM, RUN, COPY, ADD, CMD, ENTRYPOINT, WORKDIR, ENV, ARG, EXPOSE | Beginner |
| 9.7 | Dockerfile Best Practices — Layer Caching, Multi-Stage Builds, .dockerignore | Intermediate |
| 9.8 | Docker Volumes — Named, Bind Mounts, tmpfs | Intermediate |
| 9.9 | Docker Networking — Bridge, Host, None, Overlay, Macvlan | Intermediate |
| 9.10 | Docker Compose — Multi-Container Applications (v2) | Intermediate |
| 9.11 | Docker Compose — Advanced (depends_on, healthchecks, profiles, extends) | Intermediate |
| 9.12 | Docker Registry — Docker Hub, Private Registry, Harbor, ECR, ACR, GCR | Intermediate |
| 9.13 | Docker Security — Rootless Containers, Image Scanning, Distroless Images | Advanced |
| 9.14 | Docker Build — BuildKit, Buildx, Multi-Platform Builds (ARM, AMD64) | Advanced |
| 9.15 | Docker Logging & Debugging — Log Drivers, Troubleshooting | Intermediate |
| 9.16 | Docker in CI/CD — Docker-in-Docker (DinD), Docker Socket Mounting | Advanced |
| 9.17 | Docker Swarm — Basics (Comparison with Kubernetes) | Intermediate |
| 9.18 | Container Runtimes — containerd, CRI-O, runc, gVisor, Kata Containers | Advanced |
| 9.19 | Podman, Buildah, Skopeo — Docker Alternatives | Advanced |
| 9.20 | Container Image Optimization — Minimizing Size, Vulnerability Reduction | Advanced |

---

## 10. Container Orchestration — Kubernetes

> *The operating system of the cloud. Master this, master DevOps.*

| # | Subtopic | Level |
|---|----------|-------|
| **Core Concepts** | | |
| 10.1 | What is Kubernetes? — Why Orchestration? Architecture Overview | Beginner |
| 10.2 | Kubernetes Architecture — Control Plane (API Server, etcd, Scheduler, Controller Manager) | Beginner |
| 10.3 | Kubernetes Architecture — Worker Nodes (kubelet, kube-proxy, Container Runtime) | Beginner |
| 10.4 | Setting Up — Minikube, Kind, k3s, kubeadm, Managed (EKS, AKS, GKE) | Beginner |
| 10.5 | kubectl — The Swiss Army Knife (get, describe, apply, delete, logs, exec) | Beginner |
| **Workloads** | | |
| 10.6 | Pods — The Smallest Unit (Single & Multi-Container Pods, Init Containers) | Beginner |
| 10.7 | ReplicaSets — Desired State & Self-Healing | Beginner |
| 10.8 | Deployments — Rolling Updates, Rollbacks, Scaling | Beginner |
| 10.9 | StatefulSets — Stateful Applications (Databases, Kafka) | Intermediate |
| 10.10 | DaemonSets — Running Pods on Every Node | Intermediate |
| 10.11 | Jobs & CronJobs — Batch Processing | Intermediate |
| **Networking** | | |
| 10.12 | Services — ClusterIP, NodePort, LoadBalancer, ExternalName | Beginner |
| 10.13 | Ingress — Nginx Ingress Controller, Path-Based & Host-Based Routing | Intermediate |
| 10.14 | Ingress Controllers — Nginx, Traefik, HAProxy, AWS ALB | Intermediate |
| 10.15 | Network Policies — Pod-to-Pod Communication Control | Advanced |
| 10.16 | DNS in Kubernetes — CoreDNS, Service Discovery | Intermediate |
| 10.17 | Gateway API — Next-Gen Ingress | Advanced |
| **Storage** | | |
| 10.18 | Volumes — emptyDir, hostPath, PersistentVolume (PV) | Intermediate |
| 10.19 | PersistentVolumeClaims (PVC) & StorageClasses | Intermediate |
| 10.20 | CSI Drivers — EBS, EFS, Azure Disk, GCE PD | Advanced |
| **Configuration** | | |
| 10.21 | ConfigMaps — Externalizing Configuration | Beginner |
| 10.22 | Secrets — Managing Sensitive Data | Intermediate |
| 10.23 | Environment Variables, Volume Mounts, and Projections | Intermediate |
| **Scheduling & Resources** | | |
| 10.24 | Resource Requests & Limits — CPU, Memory, QoS Classes | Intermediate |
| 10.25 | Node Selectors, Affinity & Anti-Affinity | Advanced |
| 10.26 | Taints & Tolerations | Advanced |
| 10.27 | Pod Priority & Preemption | Advanced |
| 10.28 | Horizontal Pod Autoscaler (HPA) | Intermediate |
| 10.29 | Vertical Pod Autoscaler (VPA) | Advanced |
| 10.30 | Cluster Autoscaler — Node Auto-Scaling | Advanced |
| 10.31 | KEDA — Event-Driven Autoscaling | Advanced |
| **Security** | | |
| 10.32 | RBAC — Roles, ClusterRoles, RoleBindings | Intermediate |
| 10.33 | Service Accounts | Intermediate |
| 10.34 | Pod Security Standards (PSS) & Pod Security Admission (PSA) | Advanced |
| 10.35 | OPA/Gatekeeper — Policy as Code | Advanced |
| 10.36 | Kyverno — Kubernetes-Native Policy Management | Advanced |
| 10.37 | Image Security — Admission Controllers, Image Signing (Cosign, Notary) | Expert |
| **Advanced** | | |
| 10.38 | Helm — Package Manager for Kubernetes (Charts, Values, Templating, Hooks) | Intermediate |
| 10.39 | Kustomize — Template-Free Configuration Management | Intermediate |
| 10.40 | Operators — Custom Controllers & CRDs | Advanced |
| 10.41 | Operator Framework & Operator SDK | Expert |
| 10.42 | Custom Resource Definitions (CRDs) Deep Dive | Advanced |
| 10.43 | etcd — Deep Dive, Backup & Restore | Advanced |
| 10.44 | Kubernetes API — Client Libraries, Extending the API | Expert |
| 10.45 | Multi-Cluster Management — Federation, Rancher, Cluster API | Expert |
| 10.46 | Kubernetes Troubleshooting — Debug Pods, Events, CrashLoopBackOff | Intermediate |
| 10.47 | Kubernetes Upgrades — kubeadm upgrade, Zero Downtime | Advanced |
| 10.48 | Windows Containers on Kubernetes | Expert |
| 10.49 | eBPF in Kubernetes (Cilium, Tetragon) | Expert |

---

## 11. CI/CD — Continuous Integration & Continuous Delivery

> *The assembly line of modern software. Automate the pipeline.*

| # | Subtopic | Level |
|---|----------|-------|
| 11.1 | What is CI/CD? — Continuous Integration vs Delivery vs Deployment | Beginner |
| 11.2 | CI/CD Pipeline Design — Stages, Gates, Artifacts | Beginner |
| **Jenkins** | | |
| 11.3 | Jenkins — Installation, Configuration, UI Overview | Beginner |
| 11.4 | Jenkins — Freestyle Jobs vs Pipeline Jobs | Beginner |
| 11.5 | Jenkins — Declarative Pipeline (Jenkinsfile) | Intermediate |
| 11.6 | Jenkins — Scripted Pipeline (Groovy) | Advanced |
| 11.7 | Jenkins — Shared Libraries | Advanced |
| 11.8 | Jenkins — Agents, Distributed Builds, Docker Agents | Intermediate |
| 11.9 | Jenkins — Plugins (Blue Ocean, Credentials, Git, Docker, Kubernetes) | Intermediate |
| 11.10 | Jenkins — Security, RBAC, Credentials Management | Intermediate |
| 11.11 | Jenkins — Scaling with Kubernetes (Jenkins on K8s) | Advanced |
| **GitHub Actions** | | |
| 11.12 | GitHub Actions — Workflows, Jobs, Steps, Triggers | Beginner |
| 11.13 | GitHub Actions — Actions Marketplace, Custom Actions | Intermediate |
| 11.14 | GitHub Actions — Secrets, Environments, Matrix Builds | Intermediate |
| 11.15 | GitHub Actions — Self-Hosted Runners | Advanced |
| 11.16 | GitHub Actions — Reusable Workflows & Composite Actions | Advanced |
| **GitLab CI/CD** | | |
| 11.17 | GitLab CI/CD — .gitlab-ci.yml, Stages, Jobs, Runners | Intermediate |
| 11.18 | GitLab CI/CD — Variables, Artifacts, Caching | Intermediate |
| 11.19 | GitLab CI/CD — Auto DevOps, Review Apps | Advanced |
| **Other CI/CD Tools** | | |
| 11.20 | Azure DevOps Pipelines | Intermediate |
| 11.21 | CircleCI, Travis CI — Overview | Intermediate |
| 11.22 | Tekton — Cloud-Native CI/CD on Kubernetes | Advanced |
| 11.23 | Drone CI | Advanced |
| **Advanced CI/CD** | | |
| 11.24 | Pipeline as Code — Best Practices | Intermediate |
| 11.25 | Deployment Strategies — Blue/Green, Canary, Rolling, A/B Testing | Advanced |
| 11.26 | Feature Flags — LaunchDarkly, Unleash, Flagsmith | Advanced |
| 11.27 | Release Management & Changelog Automation | Intermediate |
| 11.28 | CI/CD Security — SAST, DAST, SCA, Pipeline Hardening | Advanced |

---

## 12. Infrastructure as Code (IaC)

> *Treat your infrastructure like software. Version it. Test it. Automate it.*

| # | Subtopic | Level |
|---|----------|-------|
| 12.1 | What is IaC? — Mutable vs Immutable Infrastructure | Beginner |
| 12.2 | Declarative vs Imperative Approaches | Beginner |
| **Terraform** | | |
| 12.3 | Terraform — Introduction, Architecture, Providers | Beginner |
| 12.4 | Terraform — HCL Syntax, Resources, Data Sources | Beginner |
| 12.5 | Terraform — Variables, Outputs, Locals | Beginner |
| 12.6 | Terraform — State Management, Remote State (S3, Consul, Terraform Cloud) | Intermediate |
| 12.7 | Terraform — Modules — Writing, Publishing, Using | Intermediate |
| 12.8 | Terraform — Workspaces, Environments | Intermediate |
| 12.9 | Terraform — Provisioners, null_resource, local-exec, remote-exec | Intermediate |
| 12.10 | Terraform — Import Existing Infrastructure | Intermediate |
| 12.11 | Terraform — Best Practices — Directory Structure, Naming, Tagging | Intermediate |
| 12.12 | Terraform — Testing — Terratest, Checkov, tflint | Advanced |
| 12.13 | Terraform — CI/CD Integration — Atlantis, Terraform Cloud/Enterprise | Advanced |
| 12.14 | Terraform — Advanced — Dynamic Blocks, for_each, count, Conditionals | Advanced |
| 12.15 | Terraform — State Manipulation — mv, rm, taint, untaint | Advanced |
| 12.16 | OpenTofu — Open-Source Terraform Fork | Intermediate |
| **Pulumi** | | |
| 12.17 | Pulumi — IaC with Real Programming Languages (TypeScript, Python, Go) | Advanced |
| **CloudFormation** | | |
| 12.18 | AWS CloudFormation — Templates, Stacks, StackSets, Drift Detection | Intermediate |
| 12.19 | AWS CDK — Cloud Development Kit | Advanced |
| **Other IaC** | | |
| 12.20 | Crossplane — Kubernetes-Native IaC | Expert |
| 12.21 | Bicep — Azure IaC | Intermediate |

---

## 13. Configuration Management

> *Ensure every server is configured exactly how you want — every time.*

| # | Subtopic | Level |
|---|----------|-------|
| 13.1 | What is Configuration Management? — Push vs Pull Model | Beginner |
| **Ansible** | | |
| 13.2 | Ansible — Introduction, Architecture (Agentless, SSH-Based) | Beginner |
| 13.3 | Ansible — Installation & Inventory | Beginner |
| 13.4 | Ansible — Ad-Hoc Commands | Beginner |
| 13.5 | Ansible — Playbooks (Tasks, Handlers, Variables, Templates) | Intermediate |
| 13.6 | Ansible — Roles — Structure, Galaxy, Best Practices | Intermediate |
| 13.7 | Ansible — Modules — file, copy, template, service, apt, yum, docker, k8s | Intermediate |
| 13.8 | Ansible — Variables — Facts, Vault, Precedence | Intermediate |
| 13.9 | Ansible — Conditionals, Loops, Error Handling | Intermediate |
| 13.10 | Ansible — Ansible Vault — Encrypting Secrets | Intermediate |
| 13.11 | Ansible — Dynamic Inventory (AWS, Azure, GCP) | Advanced |
| 13.12 | Ansible — AWX / Ansible Tower / AAP | Advanced |
| 13.13 | Ansible — Testing (Molecule, Testinfra) | Advanced |
| 13.14 | Ansible — Advanced Patterns — Delegation, Serial, Strategies | Advanced |
| **Other Tools** | | |
| 13.15 | Chef — Overview (Recipes, Cookbooks, Knife) | Intermediate |
| 13.16 | Puppet — Overview (Manifests, Modules, Puppet Forge) | Intermediate |
| 13.17 | SaltStack — Overview | Intermediate |

---

## 14. Cloud Platforms

> *The cloud is the data center. Know it inside out.*

| # | Subtopic | Level |
|---|----------|-------|
| 14.1 | Cloud Computing Fundamentals — IaaS, PaaS, SaaS, Public/Private/Hybrid | Beginner |
| **AWS (Amazon Web Services)** | | |
| 14.2 | AWS — Account Setup, IAM (Users, Groups, Roles, Policies) | Beginner |
| 14.3 | AWS — EC2 (Instances, AMIs, Security Groups, Key Pairs, Auto Scaling) | Beginner |
| 14.4 | AWS — VPC (Subnets, Route Tables, Internet Gateway, NAT, Security Groups, NACLs) | Intermediate |
| 14.5 | AWS — S3 (Buckets, Versioning, Lifecycle, Replication, Encryption) | Beginner |
| 14.6 | AWS — Route 53 (DNS, Hosted Zones, Routing Policies) | Intermediate |
| 14.7 | AWS — ELB (ALB, NLB, CLB) & Target Groups | Intermediate |
| 14.8 | AWS — RDS, Aurora, DynamoDB | Intermediate |
| 14.9 | AWS — ECS & Fargate (Container Services) | Intermediate |
| 14.10 | AWS — EKS (Elastic Kubernetes Service) | Advanced |
| 14.11 | AWS — Lambda (Serverless) & API Gateway | Intermediate |
| 14.12 | AWS — CloudWatch, CloudTrail, AWS Config | Intermediate |
| 14.13 | AWS — SQS, SNS, EventBridge | Intermediate |
| 14.14 | AWS — CodePipeline, CodeBuild, CodeDeploy | Intermediate |
| 14.15 | AWS — ECR (Elastic Container Registry) | Intermediate |
| 14.16 | AWS — Secrets Manager & Systems Manager (SSM) | Intermediate |
| 14.17 | AWS — CloudFront & WAF | Advanced |
| 14.18 | AWS — Organizations, Control Tower, Landing Zone | Advanced |
| 14.19 | AWS — Well-Architected Framework | Advanced |
| **Azure** | | |
| 14.20 | Azure — Resource Groups, Subscriptions, RBAC | Beginner |
| 14.21 | Azure — Virtual Machines, VMSS, Availability Sets | Intermediate |
| 14.22 | Azure — Virtual Network, NSG, Azure Load Balancer, Application Gateway | Intermediate |
| 14.23 | Azure — AKS (Azure Kubernetes Service) | Advanced |
| 14.24 | Azure — Azure DevOps, Repos, Pipelines, Boards | Intermediate |
| 14.25 | Azure — App Services, Functions, Container Instances | Intermediate |
| 14.26 | Azure — Storage Accounts, Blob, Key Vault | Intermediate |
| 14.27 | Azure — Monitor, Log Analytics, Application Insights | Intermediate |
| **GCP (Google Cloud Platform)** | | |
| 14.28 | GCP — Projects, IAM, Service Accounts | Beginner |
| 14.29 | GCP — Compute Engine, Cloud Run, GKE | Intermediate |
| 14.30 | GCP — VPC, Cloud Load Balancing, Cloud DNS | Intermediate |
| 14.31 | GCP — Cloud Storage, Cloud SQL, Firestore | Intermediate |
| 14.32 | GCP — Cloud Build, Cloud Deploy | Intermediate |
| **Multi-Cloud** | | |
| 14.33 | Multi-Cloud Strategy — Pros, Cons, Patterns | Advanced |
| 14.34 | Cloud Comparison — AWS vs Azure vs GCP | Intermediate |

---

## 15. Monitoring, Logging & Observability

> *You can't fix what you can't see.*

| # | Subtopic | Level |
|---|----------|-------|
| 15.1 | Observability Fundamentals — Metrics, Logs, Traces (The Three Pillars) | Beginner |
| 15.2 | Monitoring vs Observability — What's the Difference? | Beginner |
| **Metrics & Monitoring** | | |
| 15.3 | Prometheus — Architecture, PromQL, Targets, Exporters, AlertManager | Intermediate |
| 15.4 | Grafana — Dashboards, Panels, Data Sources, Variables, Alerting | Intermediate |
| 15.5 | Prometheus + Grafana Stack — Complete Setup | Intermediate |
| 15.6 | Node Exporter, cAdvisor, Blackbox Exporter, Custom Exporters | Advanced |
| 15.7 | Thanos / Cortex / Mimir — Scalable Prometheus | Expert |
| 15.8 | Datadog — Overview & Integration | Intermediate |
| 15.9 | New Relic, Dynatrace — APM Tools | Intermediate |
| **Logging** | | |
| 15.10 | ELK Stack — Elasticsearch, Logstash, Kibana | Intermediate |
| 15.11 | EFK Stack — Elasticsearch, Fluentd/Fluent Bit, Kibana | Intermediate |
| 15.12 | Loki + Grafana — Log Aggregation (Lightweight Alternative to ELK) | Intermediate |
| 15.13 | Structured Logging — Best Practices | Intermediate |
| 15.14 | Log Shipping & Centralized Logging Architecture | Advanced |
| **Tracing** | | |
| 15.15 | Distributed Tracing — Concepts (Spans, Traces, Context Propagation) | Intermediate |
| 15.16 | Jaeger — Distributed Tracing | Advanced |
| 15.17 | OpenTelemetry — Unified Observability (Metrics + Logs + Traces) | Advanced |
| 15.18 | Tempo — Distributed Tracing Backend | Advanced |
| **Alerting** | | |
| 15.19 | Alerting Best Practices — SLI, SLO, SLA, Error Budgets | Intermediate |
| 15.20 | PagerDuty, Opsgenie, VictorOps — Incident Management | Intermediate |
| 15.21 | On-Call Best Practices & Runbooks | Advanced |

---

## 16. Security & DevSecOps

> *Security is not a phase. It's every phase.*

| # | Subtopic | Level |
|---|----------|-------|
| 16.1 | DevSecOps — Shifting Security Left | Beginner |
| 16.2 | OWASP Top 10 — Web Application Vulnerabilities | Intermediate |
| 16.3 | SAST — Static Application Security Testing (SonarQube, Semgrep, CodeQL) | Intermediate |
| 16.4 | DAST — Dynamic Application Security Testing (OWASP ZAP, Burp Suite) | Intermediate |
| 16.5 | SCA — Software Composition Analysis (Snyk, Dependabot, Trivy) | Intermediate |
| 16.6 | Container Image Scanning — Trivy, Grype, Anchore | Intermediate |
| 16.7 | Infrastructure Security Scanning — Checkov, tfsec, KICS | Advanced |
| 16.8 | Supply Chain Security — SBOM, Sigstore, SLSA Framework | Advanced |
| 16.9 | SSL/TLS Certificates — Let's Encrypt, cert-manager | Intermediate |
| 16.10 | Zero Trust Architecture | Advanced |
| 16.11 | Network Security — Firewalls, WAF, DDoS Protection | Intermediate |
| 16.12 | Identity & Access Management — OAuth, OIDC, SAML, SSO | Advanced |
| 16.13 | Compliance — SOC2, HIPAA, PCI-DSS, GDPR | Expert |
| 16.14 | Penetration Testing Basics for DevOps | Advanced |
| 16.15 | Runtime Security — Falco, Sysdig, Aqua | Expert |

---

## 17. GitOps

> *Git as the single source of truth for everything.*

| # | Subtopic | Level |
|---|----------|-------|
| 17.1 | What is GitOps? — Principles & Benefits | Beginner |
| 17.2 | GitOps vs Traditional CI/CD — Pull vs Push Model | Intermediate |
| 17.3 | ArgoCD — Installation, Application CRD, Sync Policies | Intermediate |
| 17.4 | ArgoCD — App of Apps Pattern, ApplicationSets | Advanced |
| 17.5 | ArgoCD — Multi-Cluster, SSO, RBAC, Notifications | Advanced |
| 17.6 | Flux CD — Installation, GitRepository, Kustomization, HelmRelease | Intermediate |
| 17.7 | Flux CD — Image Automation, Notifications | Advanced |
| 17.8 | ArgoCD vs Flux — Comparison & When to Use | Intermediate |
| 17.9 | GitOps for Infrastructure (Terraform + GitOps) | Advanced |
| 17.10 | GitOps Best Practices — Repo Structure, Environment Promotion | Advanced |

---

## 18. Service Mesh & Networking (Advanced)

> *Manage microservice communication like a pro.*

| # | Subtopic | Level |
|---|----------|-------|
| 18.1 | What is a Service Mesh? — Sidecar Proxy Pattern | Intermediate |
| 18.2 | Istio — Architecture (Envoy, Istiod), Installation | Advanced |
| 18.3 | Istio — Traffic Management (VirtualService, DestinationRule, Gateway) | Advanced |
| 18.4 | Istio — Observability (Kiali, Jaeger, Prometheus Integration) | Advanced |
| 18.5 | Istio — Security (mTLS, AuthorizationPolicy) | Advanced |
| 18.6 | Linkerd — Lightweight Service Mesh | Advanced |
| 18.7 | Consul Connect — Service Mesh by HashiCorp | Advanced |
| 18.8 | Cilium Service Mesh — eBPF-Based | Expert |

---

## 19. Serverless & FaaS

> *Run code without managing servers.*

| # | Subtopic | Level |
|---|----------|-------|
| 19.1 | Serverless Concepts — FaaS, BaaS, Cold Starts, Event-Driven | Beginner |
| 19.2 | AWS Lambda — Deep Dive (Triggers, Layers, Versions, Aliases) | Intermediate |
| 19.3 | Azure Functions | Intermediate |
| 19.4 | Google Cloud Functions & Cloud Run | Intermediate |
| 19.5 | Serverless Framework — Multi-Cloud Deployment | Intermediate |
| 19.6 | Knative — Serverless on Kubernetes | Advanced |
| 19.7 | OpenFaaS — Self-Hosted Serverless | Advanced |

---

## 20. Artifact Management & Registries

> *Store, version, and distribute your build artifacts.*

| # | Subtopic | Level |
|---|----------|-------|
| 20.1 | What are Artifacts? — Types (Docker Images, Helm Charts, JARs, npm packages) | Beginner |
| 20.2 | JFrog Artifactory — Universal Artifact Manager | Intermediate |
| 20.3 | Sonatype Nexus — Repository Manager | Intermediate |
| 20.4 | Container Registries — Docker Hub, Harbor, ECR, ACR, GCR, GHCR | Intermediate |
| 20.5 | Helm Chart Repositories — ChartMuseum, OCI Registries | Intermediate |
| 20.6 | Artifact Promotion Strategies | Advanced |

---

## 21. Secrets Management

> *Never hardcode secrets. Never.*

| # | Subtopic | Level |
|---|----------|-------|
| 21.1 | Why Secrets Management Matters — Common Anti-Patterns | Beginner |
| 21.2 | HashiCorp Vault — Architecture, Secrets Engines, Auth Methods | Intermediate |
| 21.3 | HashiCorp Vault — Dynamic Secrets, PKI, Transit Encryption | Advanced |
| 21.4 | HashiCorp Vault — Kubernetes Integration (Vault Agent, CSI Driver) | Advanced |
| 21.5 | AWS Secrets Manager & SSM Parameter Store | Intermediate |
| 21.6 | Azure Key Vault | Intermediate |
| 21.7 | SOPS — Encrypted Files in Git | Intermediate |
| 21.8 | Sealed Secrets — Kubernetes-Native Secret Encryption | Intermediate |
| 21.9 | External Secrets Operator | Advanced |

---

## 22. Chaos Engineering & Reliability

> *Break things on purpose to build things that don't break.*

| # | Subtopic | Level |
|---|----------|-------|
| 22.1 | What is Chaos Engineering? — Principles & Practices | Intermediate |
| 22.2 | Chaos Monkey — Netflix's Approach | Intermediate |
| 22.3 | Litmus Chaos — Kubernetes Chaos Engineering | Advanced |
| 22.4 | Chaos Mesh — Cloud-Native Chaos Engineering | Advanced |
| 22.5 | Gremlin — Enterprise Chaos Engineering | Advanced |
| 22.6 | Game Days & Disaster Recovery Drills | Advanced |
| 22.7 | Site Reliability Engineering (SRE) — Toil, Error Budgets, SLIs/SLOs/SLAs | Intermediate |
| 22.8 | Incident Management — Blameless Postmortems, RCA (Root Cause Analysis) | Intermediate |
| 22.9 | High Availability (HA) & Disaster Recovery (DR) Patterns | Advanced |
| 22.10 | Backup Strategies — RTO & RPO | Intermediate |

---

## 23. Platform Engineering & Internal Developer Platforms

> *Build the platform that empowers developers.*

| # | Subtopic | Level |
|---|----------|-------|
| 23.1 | What is Platform Engineering? — Evolution from DevOps | Intermediate |
| 23.2 | Internal Developer Platforms (IDPs) — Concepts & Benefits | Intermediate |
| 23.3 | Backstage — Developer Portal by Spotify | Advanced |
| 23.4 | Self-Service Infrastructure — Templates, Guardrails, Golden Paths | Advanced |
| 23.5 | Developer Experience (DevEx) — Metrics & Improvement | Advanced |
| 23.6 | Crossplane — Control Plane for Infrastructure | Expert |
| 23.7 | Port, Humanitec — Platform Orchestration | Expert |

---

## 24. FinOps — Cloud Cost Optimization

> *Cloud is easy to spend on, hard to optimize.*

| # | Subtopic | Level |
|---|----------|-------|
| 24.1 | FinOps Fundamentals — Inform, Optimize, Operate | Beginner |
| 24.2 | AWS Cost Explorer, Budgets, Cost Allocation Tags | Intermediate |
| 24.3 | Reserved Instances, Savings Plans, Spot Instances | Intermediate |
| 24.4 | Right-Sizing — Identifying Over-Provisioned Resources | Intermediate |
| 24.5 | Kubernetes Cost Optimization — Kubecost, OpenCost | Advanced |
| 24.6 | Infracost — IaC Cost Estimation | Intermediate |
| 24.7 | Cloud Waste Detection & Automation | Advanced |

---

## 25. AI/ML Ops

> *The future of DevOps meets the future of AI.*

| # | Subtopic | Level |
|---|----------|-------|
| 25.1 | What is MLOps? — ML Lifecycle & Challenges | Intermediate |
| 25.2 | ML Pipelines — Data Ingestion, Training, Serving | Intermediate |
| 25.3 | Kubeflow — ML on Kubernetes | Advanced |
| 25.4 | MLflow — Experiment Tracking, Model Registry | Advanced |
| 25.5 | Feature Stores — Feast | Advanced |
| 25.6 | Model Serving — TensorFlow Serving, Seldon, KServe | Expert |
| 25.7 | LLMOps — Deploying & Managing Large Language Models | Expert |
| 25.8 | GPU Workloads on Kubernetes — NVIDIA Device Plugin | Expert |
| 25.9 | AI for DevOps — AIOps, Predictive Monitoring, Auto-Remediation | Expert |

---

## 26. Soft Skills & DevOps Culture

> *Tools are 30%. Culture is 70%.*

| # | Subtopic | Level |
|---|----------|-------|
| 26.1 | Communication & Collaboration in DevOps Teams | Beginner |
| 26.2 | Agile & Scrum for DevOps Engineers | Beginner |
| 26.3 | Documentation — Writing Runbooks, ADRs, READMEs | Beginner |
| 26.4 | Blameless Culture & Psychological Safety | Intermediate |
| 26.5 | Technical Writing & Knowledge Sharing | Intermediate |
| 26.6 | Time Management & Prioritization (On-Call, Toil Reduction) | Intermediate |
| 26.7 | Building a DevOps Community in Your Organization | Advanced |
| 26.8 | Public Speaking, Blogging & Conference Talks | Advanced |

---

## 27. Interview Preparation & Real-World Scenarios

> *Get the job. Ace the interview. Solve real problems.*

| # | Subtopic | Level |
|---|----------|-------|
| 27.1 | DevOps Interview — Linux Questions (Top 50) | All |
| 27.2 | DevOps Interview — Networking Questions (Top 30) | All |
| 27.3 | DevOps Interview — Docker Questions (Top 50) | All |
| 27.4 | DevOps Interview — Kubernetes Questions (Top 80) | All |
| 27.5 | DevOps Interview — CI/CD Questions (Top 30) | All |
| 27.6 | DevOps Interview — Terraform Questions (Top 40) | All |
| 27.7 | DevOps Interview — AWS Questions (Top 50) | All |
| 27.8 | DevOps Interview — Ansible Questions (Top 30) | All |
| 27.9 | DevOps Interview — Monitoring & Observability Questions | All |
| 27.10 | DevOps Interview — Scenario-Based Questions | All |
| 27.11 | DevOps Interview — System Design (Design a CI/CD Pipeline, Design a Monitoring Stack) | All |
| 27.12 | Resume Building for DevOps Engineers | All |

---

## 28. Projects & Hands-On Labs

> *Theory without practice is useless. Build these.*

| # | Project | Level |
|---|---------|-------|
| 28.1 | Deploy a Static Website on Nginx (Linux VM) | Beginner |
| 28.2 | Containerize a Node.js/Python App with Docker | Beginner |
| 28.3 | Multi-Container App with Docker Compose (App + DB + Redis) | Intermediate |
| 28.4 | CI/CD Pipeline with Jenkins (Build → Test → Docker → Deploy) | Intermediate |
| 28.5 | CI/CD Pipeline with GitHub Actions (Build → Test → Push to ECR → Deploy to EKS) | Intermediate |
| 28.6 | Deploy Kubernetes Cluster with kubeadm (3-node) | Intermediate |
| 28.7 | Deploy Microservices on Kubernetes (Deployments, Services, Ingress, HPA) | Intermediate |
| 28.8 | Helm Chart — Create & Deploy Your Own Chart | Intermediate |
| 28.9 | Terraform — Provision AWS Infrastructure (VPC, EC2, RDS, S3) | Intermediate |
| 28.10 | Ansible — Configure 10 Servers Automatically | Intermediate |
| 28.11 | Monitoring Stack — Prometheus + Grafana + AlertManager on Kubernetes | Advanced |
| 28.12 | EFK/ELK Logging Stack on Kubernetes | Advanced |
| 28.13 | GitOps with ArgoCD — Full Application Lifecycle | Advanced |
| 28.14 | HashiCorp Vault — Dynamic Secrets for Kubernetes Apps | Advanced |
| 28.15 | Full Production Pipeline — Code → Build → Test → Scan → Deploy → Monitor | Advanced |
| 28.16 | Multi-Cloud Deployment with Terraform | Advanced |
| 28.17 | Chaos Engineering — Inject Failures in Kubernetes with Litmus | Advanced |
| 28.18 | Build Your Own Kubernetes Operator | Expert |
| 28.19 | Design & Build an Internal Developer Platform | Expert |
| 28.20 | End-to-End DevSecOps Pipeline with Security Gates | Expert |

---

## 🗺️ Recommended Learning Path

```
Phase 1 — Foundation (Month 1-2)
├── DevOps Fundamentals (Section 1)
├── Linux & OS (Section 2)
├── Networking (Section 3)
├── Git (Section 4)
└── Scripting — Bash & Python (Section 5)

Phase 2 — Core DevOps (Month 3-5)
├── Docker (Section 9)
├── CI/CD — Jenkins & GitHub Actions (Section 11)
├── Cloud — AWS Core Services (Section 14)
├── Web Servers — Nginx (Section 8)
└── Databases Basics (Section 7)

Phase 3 — Orchestration & IaC (Month 5-8)
├── Kubernetes — Full Deep Dive (Section 10)
├── Terraform (Section 12)
├── Ansible (Section 13)
├── Helm & Kustomize (Section 10.38-10.39)
└── Monitoring — Prometheus + Grafana (Section 15)

Phase 4 — Advanced & Specialization (Month 8-12)
├── GitOps — ArgoCD / Flux (Section 17)
├── Security & DevSecOps (Section 16)
├── Secrets Management — Vault (Section 21)
├── Service Mesh — Istio (Section 18)
├── Logging — ELK / Loki (Section 15)
└── Advanced Kubernetes (Operators, CRDs, RBAC)

Phase 5 — Expert & Beyond (Month 12+)
├── Platform Engineering (Section 23)
├── Chaos Engineering (Section 22)
├── FinOps (Section 24)
├── MLOps / AIOps (Section 25)
├── Multi-Cloud & Multi-Cluster (Section 14.33, 10.45)
└── Build Real-World Projects (Section 28)
```

---

## Legend

| Tag | Meaning |
|-----|---------|
| `Beginner` | Start here. No prior knowledge needed. |
| `Intermediate` | Requires foundational understanding. |
| `Advanced` | Requires solid hands-on experience. |
| `Expert` | Deep specialization. Production-grade knowledge. |

---

> **"The best DevOps engineer is not the one who knows every tool — it's the one who knows WHY and WHEN to use each tool."**

---

*Last Updated: May 2026*
