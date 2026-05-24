# Chapter 34 — Deployment Manager & Terraform on GCP

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Fundamentals](#part-1--fundamentals)
- [Part 2: Deployment Manager Architecture](#part-2--deployment-manager-architecture)
- [Part 3: Configuration Files](#part-3--configuration-files)
- [Part 4: Templates (Jinja2)](#part-4--templates-jinja2)
- [Part 5: Templates (Python)](#part-5--templates-python)
- [Part 6: Deployments (Create, Update, Delete)](#part-6--deployments-create-update-delete)
- [Part 7: Advanced Deployment Manager Features](#part-7--advanced-deployment-manager-features)
- [Part 7a: Console Walkthrough — Deployment Manager](#part-7a-console-walkthrough--deployment-manager)
- [Part 8: Terraform on GCP — Setup & Basics](#part-8--terraform-on-gcp--setup--basics)
- [Part 9: Terraform on GCP — Common Resources](#part-9--terraform-on-gcp--common-resources)
- [Part 10: Terraform on GCP — State Management](#part-10--terraform-on-gcp--state-management)
- [Part 11: Terraform on GCP — Modules & Best Practices](#part-11--terraform-on-gcp--modules--best-practices)
- [Part 12: Terraform on GCP — CI/CD Integration](#part-12--terraform-on-gcp--cicd-integration)
- [Part 13: Migrating from Deployment Manager to Terraform](#part-13--migrating-from-deployment-manager-to-terraform)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud Deployment Manager is GCP's native Infrastructure as Code (IaC) service that lets you define, deploy, and manage cloud resources using declarative YAML templates. While Deployment Manager remains available, Google now recommends Terraform with the Google provider as the primary IaC tool for GCP. This chapter covers both—Deployment Manager for legacy/existing workflows and Terraform on GCP for modern infrastructure management.

---

## Part 1 — Fundamentals

### What Is Deployment Manager?

Deployment Manager is GCP's built-in IaC service that creates and manages resources through declarative configuration files. You describe your desired infrastructure state, and Deployment Manager figures out the API calls needed to achieve it.

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPLOYMENT MANAGER OVERVIEW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────┐         ┌─────────────────────────────┐    │
│  │  Configuration  │         │       Deployment             │    │
│  │  (YAML)        │────────▶│                              │    │
│  │                │         │  ┌───────┐  ┌───────┐       │    │
│  │  • Resources   │         │  │ VPC   │  │ GCE   │       │    │
│  │  • Properties  │         │  │Network│  │ VM    │       │    │
│  │  • Templates   │         │  └───────┘  └───────┘       │    │
│  │  • Imports     │         │  ┌───────┐  ┌───────┐       │    │
│  └────────────────┘         │  │ Fire- │  │ GCS   │       │    │
│                             │  │ wall  │  │Bucket │       │    │
│  ┌────────────────┐         │  └───────┘  └───────┘       │    │
│  │   Templates    │         │                              │    │
│  │  (Jinja/Python)│────────▶│  Managed as a single unit   │    │
│  └────────────────┘         └─────────────────────────────┘    │
│                                                                   │
│  Key Concepts:                                                    │
│  • Configuration = top-level YAML defining resources             │
│  • Template = reusable Jinja2 or Python file                     │
│  • Deployment = instantiated set of resources                    │
│  • Manifest = read-only expanded config (what was deployed)      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Deployment Manager vs Terraform

| Feature | Deployment Manager | Terraform |
|---------|-------------------|-----------|
| Provider | GCP only | Multi-cloud + SaaS |
| Language | YAML + Jinja2/Python | HCL (HashiCorp Config Language) |
| State | Managed by GCP (server-side) | Local file or remote backend |
| Preview | `--preview` flag | `terraform plan` |
| Modules | Templates (Jinja/Python) | Modules (local/registry) |
| Ecosystem | Limited | Massive provider ecosystem |
| Community | Small | Very large |
| Future | Maintenance mode | Actively developed |
| Drift detection | Manual | `terraform plan` |
| Import existing | `gcloud deploy import` | `terraform import` |
| Google recommendation | Legacy workloads | New projects |

### Cross-Cloud IaC Comparison

| Feature | GCP Deployment Manager | AWS CloudFormation | Azure ARM/Bicep |
|---------|----------------------|-------------------|-----------------|
| Config format | YAML + Jinja2/Python | JSON/YAML | JSON (ARM) / Bicep |
| Stacks/Deployments | Deployments | Stacks | Deployments |
| Modules/Nested | Templates | Nested stacks | Linked templates / Modules |
| Preview | `--preview` | Change sets | What-if |
| Rollback | Delete + redeploy | Auto-rollback | Manual |
| Drift detection | No built-in | Drift detection | No built-in |
| State storage | GCP-managed | AWS-managed | Azure-managed |
| Cross-account | No | StackSets | Deployment scopes |
| Condition support | Jinja2/Python logic | Conditions section | Conditions (Bicep) |
| CDK equivalent | None (use Terraform CDK) | AWS CDK | Azure Developer CLI |

### When to Use What

| Scenario | Recommended Tool |
|----------|-----------------|
| New GCP project, no existing IaC | Terraform |
| Multi-cloud infrastructure | Terraform |
| Existing Deployment Manager configs | Continue DM or migrate |
| Simple, GCP-only, quick prototype | Deployment Manager |
| Enterprise with compliance needs | Terraform + Sentinel/OPA |
| Google Cloud Foundation Toolkit | Terraform modules |
| Infrastructure Manager service | Terraform (backend) |

---

## Part 2 — Deployment Manager Architecture

### Components

```
┌──────────────────────────────────────────────────────────────────┐
│            DEPLOYMENT MANAGER ARCHITECTURE                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  User                                                              │
│   │                                                                │
│   │  gcloud deployment-manager deployments create                  │
│   ▼                                                                │
│  ┌────────────────────────────────────────────────────────┐       │
│  │              Configuration File (.yaml)                 │       │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │       │
│  │  │ imports  │  │resources │  │   outputs        │    │       │
│  │  │(templates│  │(type +   │  │   (references)   │    │       │
│  │  │ .jinja/  │  │properties│  │                  │    │       │
│  │  │ .py)     │  │)         │  │                  │    │       │
│  │  └──────────┘  └──────────┘  └──────────────────┘    │       │
│  └────────────────────────┬───────────────────────────────┘       │
│                           │                                        │
│                           ▼                                        │
│  ┌────────────────────────────────────────────────────────┐       │
│  │           Deployment Manager Service                    │       │
│  │                                                        │       │
│  │  1. Expand templates (Jinja2/Python)                   │       │
│  │  2. Resolve references & dependencies                  │       │
│  │  3. Determine API calls (create/update/delete)         │       │
│  │  4. Execute in dependency order                        │       │
│  │  5. Store manifest (expanded state)                    │       │
│  └────────────────────────┬───────────────────────────────┘       │
│                           │                                        │
│                           ▼                                        │
│  ┌────────────────────────────────────────────────────────┐       │
│  │              GCP Resource APIs                          │       │
│  │  Compute │ Storage │ Networking │ IAM │ ...            │       │
│  └────────────────────────────────────────────────────────┘       │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Resource Types

Deployment Manager resources map directly to GCP API resources:

| Resource Type | Description |
|--------------|-------------|
| `compute.v1.instance` | Compute Engine VM |
| `compute.v1.network` | VPC network |
| `compute.v1.firewall` | Firewall rule |
| `storage.v1.bucket` | Cloud Storage bucket |
| `sqladmin.v1beta4.instance` | Cloud SQL instance |
| `container.v1.cluster` | GKE cluster |
| `iam.v1.serviceAccount` | Service account |
| `dns.v1.managedZone` | Cloud DNS zone |

```bash
# List all available resource types
gcloud deployment-manager types list

# List types for a specific API
gcloud deployment-manager types list --filter="name~compute"
```

### Deployment Lifecycle

| State | Description |
|-------|-------------|
| `PENDING` | Deployment submitted, not yet started |
| `RUNNING` | Resources being created/updated |
| `DONE` | Deployment completed successfully |
| `FAILED` | One or more resources failed |

---

## Part 3 — Configuration Files

### Basic Configuration

```yaml
# config.yaml — Simple VM deployment
resources:
  - name: my-vm
    type: compute.v1.instance
    properties:
      zone: us-central1-a
      machineType: zones/us-central1-a/machineTypes/e2-medium
      disks:
        - deviceName: boot
          type: PERSISTENT
          boot: true
          autoDelete: true
          initializeParams:
            sourceImage: projects/debian-cloud/global/images/family/debian-11
      networkInterfaces:
        - network: global/networks/default
          accessConfigs:
            - name: External NAT
              type: ONE_TO_ONE_NAT
      metadata:
        items:
          - key: startup-script
            value: |
              #!/bin/bash
              apt-get update
              apt-get install -y nginx
```

### Configuration with Multiple Resources

```yaml
# network-config.yaml — VPC + Subnet + Firewall
resources:
  # VPC Network
  - name: my-network
    type: compute.v1.network
    properties:
      autoCreateSubnetworks: false
      routingConfig:
        routingMode: REGIONAL

  # Subnet
  - name: my-subnet
    type: compute.v1.subnetwork
    properties:
      region: us-central1
      network: $(ref.my-network.selfLink)
      ipCidrRange: 10.0.1.0/24
      privateIpGoogleAccess: true

  # Firewall — allow SSH
  - name: allow-ssh
    type: compute.v1.firewall
    properties:
      network: $(ref.my-network.selfLink)
      allowed:
        - IPProtocol: tcp
          ports:
            - "22"
      sourceRanges:
        - "35.235.240.0/20"  # IAP range
      targetTags:
        - ssh-enabled

  # Firewall — allow HTTP
  - name: allow-http
    type: compute.v1.firewall
    properties:
      network: $(ref.my-network.selfLink)
      allowed:
        - IPProtocol: tcp
          ports:
            - "80"
            - "443"
      sourceRanges:
        - "0.0.0.0/0"
      targetTags:
        - web-server

outputs:
  - name: networkSelfLink
    value: $(ref.my-network.selfLink)
  - name: subnetSelfLink
    value: $(ref.my-subnet.selfLink)
```

### References Between Resources

```yaml
# Use $(ref.RESOURCE_NAME.PROPERTY) for dependencies
resources:
  - name: my-network
    type: compute.v1.network
    properties:
      autoCreateSubnetworks: false

  - name: my-vm
    type: compute.v1.instance
    properties:
      zone: us-central1-a
      machineType: zones/us-central1-a/machineTypes/e2-medium
      disks:
        - deviceName: boot
          type: PERSISTENT
          boot: true
          autoDelete: true
          initializeParams:
            sourceImage: projects/debian-cloud/global/images/family/debian-11
      networkInterfaces:
        # Reference the network created above
        - network: $(ref.my-network.selfLink)
```

### Environment Variables

Deployment Manager provides built-in environment variables:

| Variable | Description |
|----------|-------------|
| `project` | Current project ID |
| `deployment` | Deployment name |
| `name` | Resource name |
| `project_number` | Project number |
| `current_time` | UTC timestamp |
| `type.name` | Resource type |

```yaml
resources:
  - name: {{ env["deployment"] }}-bucket
    type: storage.v1.bucket
    properties:
      name: {{ env["project"] }}-{{ env["deployment"] }}-data
      location: US
```

---

## Part 4 — Templates (Jinja2)

### Jinja2 Template Basics

```
┌──────────────────────────────────────────────────────────────┐
│                  TEMPLATE STRUCTURE                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  config.yaml (top-level)                                      │
│       │                                                        │
│       │  imports:                                              │
│       │    - path: vm-template.jinja                          │
│       │                                                        │
│       │  resources:                                            │
│       │    - name: web-server                                 │
│       │      type: vm-template.jinja                          │
│       │      properties:                                       │
│       │        zone: us-central1-a                            │
│       │        machineType: e2-medium                         │
│       │                                                        │
│       ▼                                                        │
│  vm-template.jinja                                            │
│  ┌──────────────────────────────────────┐                    │
│  │  resources:                          │                    │
│  │    - name: {{ env["name"] }}         │                    │
│  │      type: compute.v1.instance       │                    │
│  │      properties:                     │                    │
│  │        zone: {{ properties["zone"] }}│                    │
│  │        machineType: ...              │                    │
│  └──────────────────────────────────────┘                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Jinja2 Template Example

```jinja2
{# vm-template.jinja — Reusable VM template #}

resources:
  - name: {{ env["name"] }}
    type: compute.v1.instance
    properties:
      zone: {{ properties["zone"] }}
      machineType: zones/{{ properties["zone"] }}/machineTypes/{{ properties["machineType"] }}
      disks:
        - deviceName: boot
          type: PERSISTENT
          boot: true
          autoDelete: true
          initializeParams:
            sourceImage: {{ properties["sourceImage"] }}
            diskSizeGb: {{ properties["diskSizeGb"] | default(20) }}
      networkInterfaces:
        - network: {{ properties["network"] }}
          {% if properties["externalIp"] %}
          accessConfigs:
            - name: External NAT
              type: ONE_TO_ONE_NAT
          {% endif %}
      tags:
        items:
          {% for tag in properties["tags"] %}
          - {{ tag }}
          {% endfor %}
      metadata:
        items:
          - key: startup-script
            value: |
              {{ properties["startupScript"] | indent(14) }}

outputs:
  - name: selfLink
    value: $(ref.{{ env["name"] }}.selfLink)
  - name: internalIp
    value: $(ref.{{ env["name"] }}.networkInterfaces[0].networkIP)
```

### Template Schema (Validation)

```yaml
# vm-template.jinja.schema — Validates properties passed to template
info:
  title: VM Instance Template
  author: Platform Team
  description: Creates a Compute Engine VM instance
  version: 1.0

imports:
  - path: vm-template.jinja

required:
  - zone
  - machineType
  - sourceImage
  - network

properties:
  zone:
    type: string
    description: The zone for the VM
  machineType:
    type: string
    description: Machine type (e.g., e2-medium)
    default: e2-small
  sourceImage:
    type: string
    description: Boot disk image
  network:
    type: string
    description: VPC network self link
  diskSizeGb:
    type: integer
    default: 20
    minimum: 10
    maximum: 2000
  externalIp:
    type: boolean
    default: false
  tags:
    type: array
    items:
      type: string
  startupScript:
    type: string
    default: ""
```

### Using Templates in Configuration

```yaml
# config.yaml — Using the Jinja template
imports:
  - path: vm-template.jinja

resources:
  - name: web-server-1
    type: vm-template.jinja
    properties:
      zone: us-central1-a
      machineType: e2-medium
      sourceImage: projects/debian-cloud/global/images/family/debian-11
      network: global/networks/default
      externalIp: true
      diskSizeGb: 50
      tags:
        - web-server
        - http-enabled
      startupScript: |
        #!/bin/bash
        apt-get update && apt-get install -y nginx

  - name: web-server-2
    type: vm-template.jinja
    properties:
      zone: us-central1-b
      machineType: e2-medium
      sourceImage: projects/debian-cloud/global/images/family/debian-11
      network: global/networks/default
      externalIp: true
      diskSizeGb: 50
      tags:
        - web-server
        - http-enabled
      startupScript: |
        #!/bin/bash
        apt-get update && apt-get install -y nginx
```

### Jinja2 Loops and Conditionals

```jinja2
{# multi-vm.jinja — Create multiple VMs with a loop #}

{% set zones = ["us-central1-a", "us-central1-b", "us-central1-c"] %}

resources:
{% for zone in zones %}
  - name: {{ env["name"] }}-{{ loop.index }}
    type: compute.v1.instance
    properties:
      zone: {{ zone }}
      machineType: zones/{{ zone }}/machineTypes/{{ properties["machineType"] }}
      disks:
        - deviceName: boot
          type: PERSISTENT
          boot: true
          autoDelete: true
          initializeParams:
            sourceImage: {{ properties["sourceImage"] }}
      networkInterfaces:
        - network: {{ properties["network"] }}
      labels:
        index: "{{ loop.index }}"
        deployment: {{ env["deployment"] }}
{% endfor %}
```

---

## Part 5 — Templates (Python)

### Python Template Basics

Python templates offer more flexibility than Jinja2 — full programming logic, API calls, and complex conditionals:

```python
# vm_template.py — Python template for VM creation

def generate_config(context):
    """Generate Deployment Manager configuration."""
    
    resources = []
    outputs = []
    
    # Extract properties
    zone = context.properties['zone']
    machine_type = context.properties['machineType']
    source_image = context.properties['sourceImage']
    network = context.properties.get('network', 'global/networks/default')
    disk_size = context.properties.get('diskSizeGb', 20)
    
    # VM resource
    vm = {
        'name': context.env['name'],
        'type': 'compute.v1.instance',
        'properties': {
            'zone': zone,
            'machineType': f'zones/{zone}/machineTypes/{machine_type}',
            'disks': [{
                'deviceName': 'boot',
                'type': 'PERSISTENT',
                'boot': True,
                'autoDelete': True,
                'initializeParams': {
                    'sourceImage': source_image,
                    'diskSizeGb': disk_size
                }
            }],
            'networkInterfaces': [{
                'network': network
            }],
            'labels': {
                'deployment': context.env['deployment'],
                'created-by': 'deployment-manager'
            }
        }
    }
    
    # Add external IP if requested
    if context.properties.get('externalIp', False):
        vm['properties']['networkInterfaces'][0]['accessConfigs'] = [{
            'name': 'External NAT',
            'type': 'ONE_TO_ONE_NAT'
        }]
    
    # Add tags
    tags = context.properties.get('tags', [])
    if tags:
        vm['properties']['tags'] = {'items': tags}
    
    resources.append(vm)
    
    # Outputs
    outputs.append({
        'name': 'selfLink',
        'value': f'$(ref.{context.env["name"]}.selfLink)'
    })
    outputs.append({
        'name': 'internalIp',
        'value': f'$(ref.{context.env["name"]}.networkInterfaces[0].networkIP)'
    })
    
    return {'resources': resources, 'outputs': outputs}
```

### Python Template with Multiple Resources

```python
# network_template.py — Creates VPC + subnets + firewall rules

def generate_config(context):
    """Create a complete network setup."""
    
    project = context.env['project']
    deployment = context.env['deployment']
    name_prefix = context.properties['namePrefix']
    region = context.properties['region']
    subnets = context.properties.get('subnets', [])
    
    resources = []
    
    # VPC Network
    network_name = f'{name_prefix}-network'
    resources.append({
        'name': network_name,
        'type': 'compute.v1.network',
        'properties': {
            'autoCreateSubnetworks': False,
            'routingConfig': {
                'routingMode': context.properties.get('routingMode', 'REGIONAL')
            }
        }
    })
    
    # Subnets
    for i, subnet in enumerate(subnets):
        subnet_name = f'{name_prefix}-subnet-{subnet["name"]}'
        resources.append({
            'name': subnet_name,
            'type': 'compute.v1.subnetwork',
            'properties': {
                'region': subnet.get('region', region),
                'network': f'$(ref.{network_name}.selfLink)',
                'ipCidrRange': subnet['cidr'],
                'privateIpGoogleAccess': subnet.get('privateGoogle', True)
            }
        })
    
    # Default firewall rules
    # Allow internal traffic
    resources.append({
        'name': f'{name_prefix}-allow-internal',
        'type': 'compute.v1.firewall',
        'properties': {
            'network': f'$(ref.{network_name}.selfLink)',
            'allowed': [
                {'IPProtocol': 'tcp', 'ports': ['0-65535']},
                {'IPProtocol': 'udp', 'ports': ['0-65535']},
                {'IPProtocol': 'icmp'}
            ],
            'sourceRanges': [s['cidr'] for s in subnets]
        }
    })
    
    # Allow IAP SSH
    resources.append({
        'name': f'{name_prefix}-allow-iap-ssh',
        'type': 'compute.v1.firewall',
        'properties': {
            'network': f'$(ref.{network_name}.selfLink)',
            'allowed': [{'IPProtocol': 'tcp', 'ports': ['22']}],
            'sourceRanges': ['35.235.240.0/20']
        }
    })
    
    outputs = [
        {'name': 'networkSelfLink', 'value': f'$(ref.{network_name}.selfLink)'},
        {'name': 'networkName', 'value': network_name}
    ]
    
    return {'resources': resources, 'outputs': outputs}
```

### Using Python Templates

```yaml
# config.yaml — Using Python template
imports:
  - path: network_template.py

resources:
  - name: production-network
    type: network_template.py
    properties:
      namePrefix: prod
      region: us-central1
      routingMode: GLOBAL
      subnets:
        - name: web
          cidr: 10.0.1.0/24
          region: us-central1
        - name: app
          cidr: 10.0.2.0/24
          region: us-central1
        - name: data
          cidr: 10.0.3.0/24
          region: us-central1
          privateGoogle: true
```

### Python Template with Helper Functions

```python
# helpers.py — Shared utility functions
def make_vm(name, zone, machine_type, image, network, tags=None):
    """Helper to create a VM resource dict."""
    vm = {
        'name': name,
        'type': 'compute.v1.instance',
        'properties': {
            'zone': zone,
            'machineType': f'zones/{zone}/machineTypes/{machine_type}',
            'disks': [{
                'deviceName': 'boot',
                'type': 'PERSISTENT',
                'boot': True,
                'autoDelete': True,
                'initializeParams': {'sourceImage': image}
            }],
            'networkInterfaces': [{'network': network}]
        }
    }
    if tags:
        vm['properties']['tags'] = {'items': tags}
    return vm


# cluster_template.py — Uses helpers
import helpers

def generate_config(context):
    resources = []
    count = context.properties.get('count', 3)
    zone = context.properties['zone']
    
    for i in range(count):
        vm = helpers.make_vm(
            name=f'{context.env["name"]}-node-{i}',
            zone=zone,
            machine_type=context.properties['machineType'],
            image=context.properties['sourceImage'],
            network=context.properties['network'],
            tags=['cluster-node']
        )
        resources.append(vm)
    
    return {'resources': resources}
```

---

## Part 6 — Deployments (Create, Update, Delete)

### Creating a Deployment

```bash
# Create deployment from configuration file
gcloud deployment-manager deployments create my-deployment \
    --config=config.yaml

# Create with description
gcloud deployment-manager deployments create my-deployment \
    --config=config.yaml \
    --description="Production web infrastructure"

# Create with labels
gcloud deployment-manager deployments create my-deployment \
    --config=config.yaml \
    --labels=env=prod,team=platform
```

### Preview Before Deploying

```bash
# Preview changes (dry run)
gcloud deployment-manager deployments create my-deployment \
    --config=config.yaml \
    --preview

# Review the preview
gcloud deployment-manager deployments describe my-deployment

# Apply the previewed deployment
gcloud deployment-manager deployments update my-deployment

# Cancel a preview
gcloud deployment-manager deployments cancel-preview my-deployment
```

### Updating a Deployment

```bash
# Update with new configuration
gcloud deployment-manager deployments update my-deployment \
    --config=updated-config.yaml

# Update with preview first
gcloud deployment-manager deployments update my-deployment \
    --config=updated-config.yaml \
    --preview

# Then apply
gcloud deployment-manager deployments update my-deployment

# Update with create policy (how to handle new resources)
gcloud deployment-manager deployments update my-deployment \
    --config=updated-config.yaml \
    --create-policy=CREATE_OR_ACQUIRE

# Update with delete policy (how to handle removed resources)
gcloud deployment-manager deployments update my-deployment \
    --config=updated-config.yaml \
    --delete-policy=ABANDON  # Don't delete removed resources
```

### Delete Policies

| Policy | Behavior |
|--------|----------|
| `DELETE` (default) | Delete resources removed from config |
| `ABANDON` | Remove from deployment but don't delete the resource |

### Deleting a Deployment

```bash
# Delete deployment and all its resources
gcloud deployment-manager deployments delete my-deployment

# Delete deployment but keep resources (abandon)
gcloud deployment-manager deployments delete my-deployment \
    --delete-policy=ABANDON
```

### Listing and Describing

```bash
# List all deployments
gcloud deployment-manager deployments list

# Describe a deployment
gcloud deployment-manager deployments describe my-deployment

# List resources in a deployment
gcloud deployment-manager resources list --deployment=my-deployment

# Describe a specific resource
gcloud deployment-manager resources describe my-vm \
    --deployment=my-deployment

# View manifests (expanded config)
gcloud deployment-manager manifests list --deployment=my-deployment

# Describe latest manifest
gcloud deployment-manager manifests describe --deployment=my-deployment
```

---

## Part 7 — Advanced Deployment Manager Features

### Type Providers (Custom Types)

Register external APIs as custom types:

```yaml
# Register a type provider
resources:
  - name: my-type-provider
    type: deploymentmanager.v2beta.typeProvider
    properties:
      descriptorUrl: https://my-api.example.com/openapi.json
      options:
        inputMappings:
          - fieldName: Authorization
            location: HEADER
            value: $.concat("Bearer ", $.googleOauth2AccessToken())
```

### Composite Types

Bundle multiple resources into a reusable composite type:

```bash
# Create a composite type from a template
gcloud deployment-manager types create my-webapp-type \
    --template=webapp-template.jinja \
    --status=EXPERIMENTAL

# Use the composite type
# config.yaml
resources:
  - name: my-app
    type: my-project/composite:my-webapp-type
    properties:
      zone: us-central1-a
      machineType: e2-medium
```

### Runtime Configurator (Deprecated)

```yaml
# Creating runtime config variables (being deprecated)
resources:
  - name: my-config
    type: runtimeconfig.v1beta1.config
    properties:
      config: my-config
      description: "Application configuration"
  - name: my-variable
    type: runtimeconfig.v1beta1.variable
    properties:
      parent: $(ref.my-config.name)
      variable: /config/db-host
      text: "10.0.1.5"
```

### IAM Bindings in Deployment Manager

```yaml
# Grant IAM roles as part of deployment
resources:
  - name: my-bucket
    type: storage.v1.bucket
    properties:
      name: my-project-data-bucket
      location: US
    accessControl:
      gcpIamPolicy:
        bindings:
          - role: roles/storage.objectViewer
            members:
              - serviceAccount:my-sa@my-project.iam.gserviceaccount.com
          - role: roles/storage.objectAdmin
            members:
              - group:data-team@example.com
```

### Metadata and Dependencies

```yaml
resources:
  - name: my-network
    type: compute.v1.network
    properties:
      autoCreateSubnetworks: false

  - name: my-vm
    type: compute.v1.instance
    metadata:
      dependsOn:
        - my-network   # Explicit dependency (beyond references)
    properties:
      zone: us-central1-a
      machineType: zones/us-central1-a/machineTypes/e2-medium
      # ...
```

---

## Part 7a: Console Walkthrough — Deployment Manager

### Viewing Deployments

```
Console → Deployment Manager → Deployments

┌─────────────────────────────────────────────────────────────────┐
│           DEPLOYMENTS                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ Name              │ Status     │ Last updated  │ Actions  │  │
│ ├────────────────────────────────────────────────────────────┤  │
│ │ web-app-prod      │ ✅ DONE    │ 2024-03-15    │ ⋮        │  │
│ │ vpc-network       │ ✅ DONE    │ 2024-03-10    │ ⋮        │  │
│ │ dev-environment   │ ❌ FAILED  │ 2024-03-14    │ ⋮        │  │
│ └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│ Actions menu (⋮):                                               │
│ ├── View deployment details                                    │
│ ├── Update deployment                                          │
│ ├── Delete deployment                                          │
│ └── View manifest/config                                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Viewing Deployment Details

```
Console → Deployment Manager → Click deployment name

┌─────────────────────────────────────────────────────────────────┐
│           DEPLOYMENT: web-app-prod                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Status: ✅ DONE                                                  │
│ Created: 2024-03-15 10:30:00 UTC                                │
│                                                                   │
│ ── Overview tab ──                                              │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ Resource            │ Type                    │ Status     │  │
│ ├────────────────────────────────────────────────────────────┤  │
│ │ web-vm              │ compute.v1.instance     │ COMPLETED  │  │
│ │ web-firewall        │ compute.v1.firewall     │ COMPLETED  │  │
│ │ web-network         │ compute.v1.network      │ COMPLETED  │  │
│ │ web-static-ip       │ compute.v1.address      │ COMPLETED  │  │
│ └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│ ── Layout tab ──                                                │
│ Shows the expanded YAML configuration with all properties       │
│ and resolved template values                                    │
│                                                                   │
│ ⚡ Click any resource → navigates to that resource's page       │
│   (e.g., click web-vm → opens Compute Engine instance page)    │
│                                                                   │
│ ⚠️ Note: Deployment Manager is primarily CLI-driven.            │
│   Deployments are CREATED via gcloud CLI, not the console.     │
│   Console is used for VIEWING, INSPECTING, and DELETING.       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Deleting a Deployment from Console

```
Console → Deployment Manager → Select deployment(s) → DELETE

┌─────────────────────────────────────────────────────────────────┐
│           DELETE DEPLOYMENT                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Delete deployment "web-app-prod"?                               │
│                                                                   │
│ Deletion policy:                                                │
│ ● Delete deployment and all resources                          │
│ ○ Abandon resources (delete deployment record only,            │
│   keep resources running)                                      │
│                                                                   │
│ ⚠️ Deleting will remove ALL resources created by               │
│   this deployment (VMs, networks, firewalls, etc.)             │
│                                                                   │
│                      [DELETE]   [CANCEL]                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 8 — Terraform on GCP — Setup & Basics

### Setting Up Terraform for GCP

```
┌──────────────────────────────────────────────────────────────┐
│              TERRAFORM ON GCP — SETUP                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────────────────────────┐                       │
│  │  1. Install Terraform CLI           │                       │
│  │  2. Configure Google Provider       │                       │
│  │  3. Authenticate (SA or user)       │                       │
│  │  4. Set up remote state backend     │                       │
│  └────────────────────────────────────┘                       │
│                                                                │
│  Authentication Methods:                                       │
│  ┌──────────────────┬──────────────────────────────────┐     │
│  │ Method           │ Use Case                          │     │
│  ├──────────────────┼──────────────────────────────────┤     │
│  │ gcloud auth      │ Local development                 │     │
│  │ Service Account  │ CI/CD pipelines                   │     │
│  │ key file         │                                   │     │
│  │ Workload Identity│ Cloud Build, GKE                  │     │
│  │ Federation       │ External (AWS, Azure, GitHub)     │     │
│  └──────────────────┴──────────────────────────────────┘     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Provider Configuration

```hcl
# versions.tf — Provider requirements
terraform {
  required_version = ">= 1.5"
  
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
  }
}

# provider.tf — Configure the provider
provider "google" {
  project = var.project_id
  region  = var.region
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}
```

### Remote State Backend (GCS)

```hcl
# backend.tf — Store state in Cloud Storage
terraform {
  backend "gcs" {
    bucket = "my-project-terraform-state"
    prefix = "environments/prod"
  }
}
```

### Creating the State Bucket

```bash
# Create GCS bucket for Terraform state
gsutil mb -l us-central1 gs://my-project-terraform-state

# Enable versioning (critical for state recovery)
gsutil versioning set on gs://my-project-terraform-state

# Enable uniform bucket-level access
gsutil uniformbucketlevelaccess set on gs://my-project-terraform-state
```

### Variables and Outputs

```hcl
# variables.tf
variable "project_id" {
  description = "GCP project ID"
  type        = string
}

variable "region" {
  description = "Default region"
  type        = string
  default     = "us-central1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# outputs.tf
output "network_self_link" {
  description = "VPC network self link"
  value       = google_compute_network.main.self_link
}

output "subnet_ids" {
  description = "Map of subnet names to IDs"
  value       = { for s in google_compute_subnetwork.subnets : s.name => s.id }
}
```

### Authentication for CI/CD

```hcl
# Using Workload Identity Federation (no keys)
provider "google" {
  project = var.project_id
  region  = var.region
  # Credentials automatically provided by:
  # - Cloud Build (built-in SA)
  # - GitHub Actions (Workload Identity Federation)
  # - Cloud Shell (user credentials)
}
```

```bash
# Local development — use application default credentials
gcloud auth application-default login

# CI/CD — use service account key (less preferred)
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"

# CI/CD — use Workload Identity Federation (preferred)
# Configured at provider level with access_token or impersonation
```

---

## Part 9 — Terraform on GCP — Common Resources

### VPC Network

```hcl
# VPC Network
resource "google_compute_network" "main" {
  name                    = "${var.environment}-network"
  auto_create_subnetworks = false
  routing_mode            = "GLOBAL"
  project                 = var.project_id
}

# Subnets
resource "google_compute_subnetwork" "subnets" {
  for_each = var.subnets

  name          = "${var.environment}-${each.key}"
  ip_cidr_range = each.value.cidr
  region        = each.value.region
  network       = google_compute_network.main.id
  project       = var.project_id

  private_ip_google_access = true

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = each.value.pods_cidr
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = each.value.services_cidr
  }
}

# Firewall Rules
resource "google_compute_firewall" "allow_internal" {
  name    = "${var.environment}-allow-internal"
  network = google_compute_network.main.name
  project = var.project_id

  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }
  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }
  allow {
    protocol = "icmp"
  }

  source_ranges = [for s in google_compute_subnetwork.subnets : s.ip_cidr_range]
}
```

### GKE Cluster

```hcl
resource "google_container_cluster" "primary" {
  name     = "${var.environment}-cluster"
  location = var.region
  project  = var.project_id

  # Use separately managed node pool
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.main.name
  subnetwork = google_compute_subnetwork.subnets["app"].name

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  release_channel {
    channel = "REGULAR"
  }
}

resource "google_container_node_pool" "primary" {
  name       = "${var.environment}-node-pool"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  project    = var.project_id
  node_count = var.node_count

  node_config {
    machine_type    = var.machine_type
    service_account = google_service_account.gke_nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]

    labels = {
      environment = var.environment
    }

    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }

  autoscaling {
    min_node_count = var.min_nodes
    max_node_count = var.max_nodes
  }
}
```

### Cloud SQL

```hcl
resource "google_sql_database_instance" "main" {
  name             = "${var.environment}-db-instance"
  database_version = "POSTGRES_15"
  region           = var.region
  project          = var.project_id

  settings {
    tier              = var.db_tier
    availability_type = var.environment == "prod" ? "REGIONAL" : "ZONAL"
    disk_size         = var.db_disk_size
    disk_autoresize   = true

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
    }

    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
      start_time                     = "03:00"
    }

    maintenance_window {
      day          = 7  # Sunday
      hour         = 4
      update_track = "stable"
    }
  }

  deletion_protection = var.environment == "prod"
}

resource "google_sql_database" "app" {
  name     = "app_db"
  instance = google_sql_database_instance.main.name
  project  = var.project_id
}

resource "google_sql_user" "app" {
  name     = "app_user"
  instance = google_sql_database_instance.main.name
  password = random_password.db_password.result
  project  = var.project_id
}
```

### Cloud Run Service

```hcl
resource "google_cloud_run_v2_service" "app" {
  name     = "${var.environment}-app"
  location = var.region
  project  = var.project_id

  template {
    scaling {
      min_instance_count = var.environment == "prod" ? 2 : 0
      max_instance_count = 100
    }

    containers {
      image = var.container_image

      ports {
        container_port = 8080
      }

      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
      }

      env {
        name  = "ENV"
        value = var.environment
      }

      env {
        name = "DB_PASSWORD"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.db_password.secret_id
            version = "latest"
          }
        }
      }
    }

    service_account = google_service_account.app.email
  }

  traffic {
    type    = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST"
    percent = 100
  }
}
```

---

## Part 10 — Terraform on GCP — State Management

### State Backend Options

```
┌──────────────────────────────────────────────────────────────┐
│              TERRAFORM STATE ON GCP                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Recommended: GCS Backend                             │    │
│  │                                                      │    │
│  │  ┌─────────┐      ┌──────────────────────────┐     │    │
│  │  │Terraform│─────▶│  GCS Bucket              │     │    │
│  │  │  CLI    │      │  • Versioned             │     │    │
│  │  └─────────┘      │  • Encrypted at rest     │     │    │
│  │                    │  • IAM access control    │     │    │
│  │                    │  • Lock via GCS locking  │     │    │
│  │                    └──────────────────────────┘     │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  State per environment:                                        │
│  gs://tf-state-bucket/                                        │
│  ├── environments/dev/default.tfstate                         │
│  ├── environments/staging/default.tfstate                     │
│  └── environments/prod/default.tfstate                        │
│                                                                │
│  Locking: GCS provides object-level locking automatically     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### State Backend with Encryption

```hcl
terraform {
  backend "gcs" {
    bucket               = "my-project-tf-state"
    prefix               = "prod"
    # CMEK encryption (optional — default is Google-managed)
    # encryption_key     = "projects/my-project/locations/global/keyRings/tf/cryptoKeys/state"
  }
}
```

### State Operations

```bash
# List resources in state
terraform state list

# Show a specific resource
terraform state show google_compute_instance.web

# Move a resource (rename in state)
terraform state mv google_compute_instance.web google_compute_instance.app

# Remove from state (without destroying)
terraform state rm google_compute_instance.legacy

# Import existing resource into state
terraform import google_compute_instance.existing \
    projects/my-project/zones/us-central1-a/instances/my-vm

# Pull remote state locally
terraform state pull > state.json

# Force unlock (use with caution)
terraform force-unlock LOCK_ID
```

### Workspaces (Alternative to Prefixes)

```bash
# Create workspace per environment
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch workspace
terraform workspace select prod

# Use workspace in configuration
locals {
  environment = terraform.workspace
}
```

### Data Sources for Cross-State References

```hcl
# Reference another Terraform state
data "terraform_remote_state" "network" {
  backend = "gcs"
  config = {
    bucket = "my-project-tf-state"
    prefix = "network"
  }
}

# Use outputs from remote state
resource "google_compute_instance" "app" {
  # ...
  network_interface {
    subnetwork = data.terraform_remote_state.network.outputs.subnet_id
  }
}
```

---

## Part 11 — Terraform on GCP — Modules & Best Practices

### Module Structure

```
┌──────────────────────────────────────────────────────────────┐
│              TERRAFORM PROJECT STRUCTURE                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  terraform/                                                    │
│  ├── modules/                                                  │
│  │   ├── network/                                             │
│  │   │   ├── main.tf                                          │
│  │   │   ├── variables.tf                                     │
│  │   │   ├── outputs.tf                                       │
│  │   │   └── versions.tf                                      │
│  │   ├── gke/                                                 │
│  │   │   ├── main.tf                                          │
│  │   │   ├── variables.tf                                     │
│  │   │   └── outputs.tf                                       │
│  │   └── cloud-sql/                                           │
│  │       ├── main.tf                                          │
│  │       ├── variables.tf                                     │
│  │       └── outputs.tf                                       │
│  ├── environments/                                             │
│  │   ├── dev/                                                 │
│  │   │   ├── main.tf          (module calls)                  │
│  │   │   ├── backend.tf       (GCS state)                     │
│  │   │   ├── variables.tf                                     │
│  │   │   ├── terraform.tfvars (env values)                    │
│  │   │   └── outputs.tf                                       │
│  │   ├── staging/                                             │
│  │   │   └── ...                                              │
│  │   └── prod/                                                │
│  │       └── ...                                              │
│  └── global/                  (project-level resources)       │
│      ├── main.tf                                              │
│      └── ...                                                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Google Cloud Foundation Toolkit Modules

Google provides official Terraform modules:

```hcl
# Use official Google network module
module "vpc" {
  source  = "terraform-google-modules/network/google"
  version = "~> 9.0"

  project_id   = var.project_id
  network_name = "${var.environment}-vpc"
  routing_mode = "GLOBAL"

  subnets = [
    {
      subnet_name           = "app"
      subnet_ip             = "10.0.1.0/24"
      subnet_region         = var.region
      subnet_private_access = true
    },
    {
      subnet_name           = "data"
      subnet_ip             = "10.0.2.0/24"
      subnet_region         = var.region
      subnet_private_access = true
    }
  ]

  secondary_ranges = {
    app = [
      { range_name = "pods", ip_cidr_range = "10.1.0.0/16" },
      { range_name = "services", ip_cidr_range = "10.2.0.0/20" }
    ]
  }
}

# Use official GKE module
module "gke" {
  source  = "terraform-google-modules/kubernetes-engine/google//modules/private-cluster"
  version = "~> 30.0"

  project_id = var.project_id
  name       = "${var.environment}-cluster"
  region     = var.region

  network           = module.vpc.network_name
  subnetwork        = module.vpc.subnets_names[0]
  ip_range_pods     = "pods"
  ip_range_services = "services"

  enable_private_nodes    = true
  enable_private_endpoint = false
  master_ipv4_cidr_block  = "172.16.0.0/28"

  node_pools = [
    {
      name         = "default"
      machine_type = "e2-standard-4"
      min_count    = 1
      max_count    = 10
      auto_repair  = true
      auto_upgrade = true
    }
  ]
}

# Use official Cloud SQL module
module "cloud_sql" {
  source  = "terraform-google-modules/sql-db/google//modules/postgresql"
  version = "~> 18.0"

  project_id       = var.project_id
  name             = "${var.environment}-postgres"
  database_version = "POSTGRES_15"
  region           = var.region
  zone             = "${var.region}-a"
  tier             = "db-custom-2-8192"

  ip_configuration = {
    ipv4_enabled    = false
    private_network = module.vpc.network_self_link
  }

  backup_configuration = {
    enabled                        = true
    point_in_time_recovery_enabled = true
    start_time                     = "03:00"
  }
}
```

### Terragrunt for DRY Configurations

```hcl
# terragrunt.hcl (root)
remote_state {
  backend = "gcs"
  config = {
    bucket   = "my-project-tf-state"
    prefix   = "${path_relative_to_include()}"
    project  = "my-project"
    location = "us-central1"
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "google" {
  project = var.project_id
  region  = var.region
}
EOF
}

# environments/prod/network/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/network"
}

inputs = {
  project_id  = "my-prod-project"
  region      = "us-central1"
  environment = "prod"
  subnets = {
    app  = { cidr = "10.0.1.0/24", region = "us-central1" }
    data = { cidr = "10.0.2.0/24", region = "us-central1" }
  }
}
```

---

## Part 12 — Terraform on GCP — CI/CD Integration

### Cloud Build + Terraform

```
┌──────────────────────────────────────────────────────────────┐
│           TERRAFORM CI/CD WITH CLOUD BUILD                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Developer                                                     │
│    │  Push to feature branch                                  │
│    ▼                                                           │
│  ┌──────────────┐                                             │
│  │ Cloud Build  │                                             │
│  │ (PR trigger) │                                             │
│  │              │                                             │
│  │ terraform    │──── Plan output posted as PR comment        │
│  │   init       │                                             │
│  │   plan       │                                             │
│  └──────────────┘                                             │
│                                                                │
│  Merge to main                                                 │
│    │                                                           │
│    ▼                                                           │
│  ┌──────────────┐                                             │
│  │ Cloud Build  │                                             │
│  │(main trigger)│                                             │
│  │              │                                             │
│  │ terraform    │──── Apply changes to infrastructure         │
│  │   init       │                                             │
│  │   apply      │                                             │
│  │   -auto-     │                                             │
│  │   approve    │                                             │
│  └──────────────┘                                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Cloud Build Configuration for Terraform

```yaml
# cloudbuild-plan.yaml — Run on PR
steps:
  - id: 'terraform-init'
    name: 'hashicorp/terraform:1.7'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        cd environments/${_ENV}
        terraform init -no-color
    
  - id: 'terraform-validate'
    name: 'hashicorp/terraform:1.7'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        cd environments/${_ENV}
        terraform validate -no-color

  - id: 'terraform-plan'
    name: 'hashicorp/terraform:1.7'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        cd environments/${_ENV}
        terraform plan -no-color -out=tfplan
        terraform show -no-color tfplan > plan_output.txt

substitutions:
  _ENV: 'prod'

options:
  logging: CLOUD_LOGGING_ONLY
```

```yaml
# cloudbuild-apply.yaml — Run on merge to main
steps:
  - id: 'terraform-init'
    name: 'hashicorp/terraform:1.7'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        cd environments/${_ENV}
        terraform init -no-color

  - id: 'terraform-apply'
    name: 'hashicorp/terraform:1.7'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        cd environments/${_ENV}
        terraform apply -auto-approve -no-color

substitutions:
  _ENV: 'prod'

options:
  logging: CLOUD_LOGGING_ONLY
```

### Service Account for Terraform in CI/CD

```bash
# Create SA for Terraform
gcloud iam service-accounts create terraform-ci \
    --display-name="Terraform CI/CD"

# Grant project-level roles
ROLES=(
  "roles/compute.admin"
  "roles/container.admin"
  "roles/iam.serviceAccountAdmin"
  "roles/resourcemanager.projectIamAdmin"
  "roles/storage.admin"
  "roles/cloudsql.admin"
  "roles/run.admin"
  "roles/secretmanager.admin"
)

for ROLE in "${ROLES[@]}"; do
  gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:terraform-ci@my-project.iam.gserviceaccount.com" \
    --role="$ROLE"
done

# Grant Cloud Build SA ability to impersonate Terraform SA
gcloud iam service-accounts add-iam-policy-binding \
    terraform-ci@my-project.iam.gserviceaccount.com \
    --member="serviceAccount:PROJECT_NUM@cloudbuild.gserviceaccount.com" \
    --role="roles/iam.serviceAccountTokenCreator"
```

### GitHub Actions + Terraform on GCP

```yaml
# .github/workflows/terraform.yml
name: Terraform
on:
  pull_request:
    paths: ['terraform/**']
  push:
    branches: [main]
    paths: ['terraform/**']

permissions:
  contents: read
  id-token: write      # Required for Workload Identity Federation
  pull-requests: write # For PR comments

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform/environments/prod

    steps:
      - uses: actions/checkout@v4

      - id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/PROJECT_NUM/locations/global/workloadIdentityPools/github/providers/github-actions'
          service_account: 'terraform-ci@my-project.iam.gserviceaccount.com'

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7"

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color -out=tfplan
        
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve -no-color
```

---

## Part 13 — Migrating from Deployment Manager to Terraform

### Migration Strategy

```
┌──────────────────────────────────────────────────────────────┐
│          MIGRATION: DEPLOYMENT MANAGER → TERRAFORM            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Inventory existing deployments                       │
│  ┌──────────────────────────────────────┐                    │
│  │ gcloud deployment-manager            │                    │
│  │   deployments list                   │                    │
│  │ gcloud deployment-manager            │                    │
│  │   resources list --deployment=NAME   │                    │
│  └──────────────────────────────────────┘                    │
│                                                                │
│  Step 2: Write equivalent Terraform configs                   │
│  ┌──────────────────────────────────────┐                    │
│  │ For each DM resource → HCL resource  │                    │
│  │ Map properties → arguments           │                    │
│  └──────────────────────────────────────┘                    │
│                                                                │
│  Step 3: Import existing resources into Terraform state       │
│  ┌──────────────────────────────────────┐                    │
│  │ terraform import RESOURCE ID         │                    │
│  │ terraform plan (should show no diff) │                    │
│  └──────────────────────────────────────┘                    │
│                                                                │
│  Step 4: Abandon resources from DM (don't delete them)       │
│  ┌──────────────────────────────────────┐                    │
│  │ gcloud deployment-manager            │                    │
│  │   deployments delete NAME            │                    │
│  │   --delete-policy=ABANDON            │                    │
│  └──────────────────────────────────────┘                    │
│                                                                │
│  Step 5: Terraform now manages the resources                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Import Examples

```bash
# Import a Compute Engine instance
terraform import google_compute_instance.web \
    projects/my-project/zones/us-central1-a/instances/web-server

# Import a VPC network
terraform import google_compute_network.main \
    projects/my-project/global/networks/my-network

# Import a GCS bucket
terraform import google_storage_bucket.data \
    my-project-data-bucket

# Import a Cloud SQL instance
terraform import google_sql_database_instance.main \
    projects/my-project/instances/my-db-instance

# Import a GKE cluster
terraform import google_container_cluster.primary \
    projects/my-project/locations/us-central1/clusters/my-cluster

# Import a service account
terraform import google_service_account.app \
    projects/my-project/serviceAccounts/app-sa@my-project.iam.gserviceaccount.com
```

### Using `gcloud beta resource-config bulk-export`

```bash
# Export all resources in a project as Terraform HCL
gcloud beta resource-config bulk-export \
    --project=my-project \
    --resource-format=terraform \
    --path=./exported-terraform

# Export specific resource types
gcloud beta resource-config bulk-export \
    --project=my-project \
    --resource-format=terraform \
    --resource-types=ComputeInstance,ComputeNetwork,ComputeFirewall \
    --path=./exported-compute

# Generate import statements
gcloud beta resource-config bulk-export \
    --project=my-project \
    --resource-format=terraform \
    --path=./export \
    --on-error=continue
```

### Mapping DM Types to Terraform Resources

| Deployment Manager Type | Terraform Resource |
|------------------------|-------------------|
| `compute.v1.instance` | `google_compute_instance` |
| `compute.v1.network` | `google_compute_network` |
| `compute.v1.subnetwork` | `google_compute_subnetwork` |
| `compute.v1.firewall` | `google_compute_firewall` |
| `storage.v1.bucket` | `google_storage_bucket` |
| `sqladmin.v1beta4.instance` | `google_sql_database_instance` |
| `container.v1.cluster` | `google_container_cluster` |
| `iam.v1.serviceAccount` | `google_service_account` |
| `dns.v1.managedZone` | `google_dns_managed_zone` |
| `cloudresourcemanager.v1.project` | `google_project` |

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform Full Example — Complete Environment

```hcl
# main.tf — Complete environment deployment
terraform {
  required_version = ">= 1.5"
  
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  
  backend "gcs" {
    bucket = "my-project-tf-state"
    prefix = "prod"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Enable required APIs
resource "google_project_service" "apis" {
  for_each = toset([
    "compute.googleapis.com",
    "container.googleapis.com",
    "sqladmin.googleapis.com",
    "run.googleapis.com",
    "secretmanager.googleapis.com",
    "clouddeploy.googleapis.com",
  ])

  project = var.project_id
  service = each.value

  disable_dependent_services = false
  disable_on_destroy         = false
}

# Network
resource "google_compute_network" "main" {
  name                    = "${var.environment}-vpc"
  auto_create_subnetworks = false
  project                 = var.project_id
  depends_on              = [google_project_service.apis]
}

resource "google_compute_subnetwork" "app" {
  name          = "${var.environment}-app-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = var.region
  network       = google_compute_network.main.id

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.1.0.0/16"
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.2.0.0/20"
  }
}

# NAT for private instances
resource "google_compute_router" "router" {
  name    = "${var.environment}-router"
  region  = var.region
  network = google_compute_network.main.id
}

resource "google_compute_router_nat" "nat" {
  name                               = "${var.environment}-nat"
  router                             = google_compute_router.router.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}

# Service Account
resource "google_service_account" "app" {
  account_id   = "${var.environment}-app-sa"
  display_name = "Application Service Account"
  project      = var.project_id
}

# Secret
resource "google_secret_manager_secret" "db_password" {
  secret_id = "${var.environment}-db-password"
  project   = var.project_id

  replication {
    auto {}
  }
}
```

### gcloud Deployment Manager CLI Reference

```bash
# ─── Deployment Operations ────────────────────────────────────
# Create deployment
gcloud deployment-manager deployments create NAME --config=FILE.yaml

# Create with preview
gcloud deployment-manager deployments create NAME --config=FILE.yaml --preview

# Update deployment
gcloud deployment-manager deployments update NAME --config=FILE.yaml

# Delete deployment
gcloud deployment-manager deployments delete NAME

# Delete but keep resources
gcloud deployment-manager deployments delete NAME --delete-policy=ABANDON

# Cancel preview
gcloud deployment-manager deployments cancel-preview NAME

# ─── Listing & Describing ────────────────────────────────────
# List deployments
gcloud deployment-manager deployments list

# Describe deployment
gcloud deployment-manager deployments describe NAME

# List resources in deployment
gcloud deployment-manager resources list --deployment=NAME

# Describe resource
gcloud deployment-manager resources describe RESOURCE --deployment=NAME

# List manifests
gcloud deployment-manager manifests list --deployment=NAME

# Describe manifest (see expanded config)
gcloud deployment-manager manifests describe MANIFEST --deployment=NAME

# ─── Types ────────────────────────────────────────────────────
# List available types
gcloud deployment-manager types list

# List types filtered
gcloud deployment-manager types list --filter="name~compute"

# ─── Operations ───────────────────────────────────────────────
# List operations
gcloud deployment-manager operations list

# Describe operation
gcloud deployment-manager operations describe OPERATION

# ─── Import/Export ────────────────────────────────────────────
# Export project resources as Terraform
gcloud beta resource-config bulk-export \
    --project=PROJECT_ID \
    --resource-format=terraform \
    --path=./output
```

### Terraform CLI Reference for GCP

```bash
# ─── Initialization ──────────────────────────────────────────
terraform init                    # Initialize, download providers
terraform init -upgrade           # Upgrade providers
terraform init -migrate-state     # Migrate state backend
terraform init -reconfigure       # Reconfigure backend

# ─── Planning ────────────────────────────────────────────────
terraform plan                    # Show execution plan
terraform plan -out=tfplan        # Save plan to file
terraform plan -target=RESOURCE   # Plan specific resource
terraform plan -var="key=value"   # Override variable
terraform plan -destroy           # Plan destruction

# ─── Applying ────────────────────────────────────────────────
terraform apply                   # Apply changes (interactive)
terraform apply -auto-approve     # Apply without confirmation
terraform apply tfplan            # Apply saved plan
terraform apply -target=RESOURCE  # Apply specific resource
terraform apply -replace=RESOURCE # Force recreate resource

# ─── Destroying ──────────────────────────────────────────────
terraform destroy                 # Destroy all resources
terraform destroy -target=RES     # Destroy specific resource

# ─── State ───────────────────────────────────────────────────
terraform state list              # List resources in state
terraform state show RESOURCE     # Show resource details
terraform state mv SRC DEST       # Rename/move in state
terraform state rm RESOURCE       # Remove from state
terraform state pull              # Download remote state
terraform state push              # Upload local state

# ─── Import ──────────────────────────────────────────────────
terraform import RESOURCE ID      # Import existing resource

# ─── Workspace ───────────────────────────────────────────────
terraform workspace list          # List workspaces
terraform workspace new NAME      # Create workspace
terraform workspace select NAME   # Switch workspace

# ─── Validation ──────────────────────────────────────────────
terraform validate                # Validate configuration
terraform fmt                     # Format files
terraform fmt -check              # Check formatting (CI)

# ─── Output ──────────────────────────────────────────────────
terraform output                  # Show all outputs
terraform output NAME             # Show specific output
terraform output -json            # JSON format
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Enterprise Landing Zone with Terraform

```
┌──────────────────────────────────────────────────────────────────────┐
│        PATTERN 1: GCP LANDING ZONE (TERRAFORM)                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  Organization                                                │     │
│  │                                                              │     │
│  │  ┌─────────────────────────────────────────────────────┐   │     │
│  │  │  Terraform manages:                                  │   │     │
│  │  │  • Folder structure                                  │   │     │
│  │  │  • Projects (via project factory module)             │   │     │
│  │  │  • Shared VPC networks                               │   │     │
│  │  │  • Organization policies                             │   │     │
│  │  │  • IAM at org/folder level                          │   │     │
│  │  │  • Logging & monitoring                             │   │     │
│  │  └─────────────────────────────────────────────────────┘   │     │
│  │                                                              │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │     │
│  │  │  Shared/     │  │  Production/ │  │  Dev/        │     │     │
│  │  │  ├── network │  │  ├── app-1   │  │  ├── app-1   │     │     │
│  │  │  ├── logging │  │  ├── app-2   │  │  ├── app-2   │     │     │
│  │  │  └── security│  │  └── data    │  │  └── sandbox │     │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘     │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                        │
│  Uses Google Cloud Foundation Toolkit:                                 │
│  • terraform-google-modules/project-factory                           │
│  • terraform-google-modules/network                                   │
│  • terraform-google-modules/org-policy                                │
│  • terraform-google-modules/log-export                                │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```hcl
# Organization-level Terraform
module "org_policies" {
  source  = "terraform-google-modules/org-policy/google"
  version = "~> 5.0"

  project_id  = var.project_id
  policy_for  = "organization"
  organization_id = var.org_id

  constraint  = "constraints/compute.disableSerialPortAccess"
  policy_type = "boolean"
  enforce     = true
}

# Project Factory — create projects consistently
module "project_prod_app1" {
  source  = "terraform-google-modules/project-factory/google"
  version = "~> 15.0"

  name              = "prod-app1"
  org_id            = var.org_id
  folder_id         = google_folder.production.id
  billing_account   = var.billing_account
  
  activate_apis = [
    "compute.googleapis.com",
    "container.googleapis.com",
    "run.googleapis.com",
  ]

  shared_vpc       = module.shared_vpc.project_id
  shared_vpc_subnets = [
    "projects/${var.host_project}/regions/us-central1/subnetworks/prod-app"
  ]

  labels = {
    environment = "production"
    team        = "app1"
  }
}

# Shared VPC
module "shared_vpc" {
  source  = "terraform-google-modules/network/google"
  version = "~> 9.0"

  project_id   = var.host_project_id
  network_name = "shared-vpc"
  routing_mode = "GLOBAL"

  subnets = [
    {
      subnet_name   = "prod-app"
      subnet_ip     = "10.0.0.0/20"
      subnet_region = "us-central1"
    },
    {
      subnet_name   = "prod-data"
      subnet_ip     = "10.0.16.0/20"
      subnet_region = "us-central1"
    },
    {
      subnet_name   = "dev-app"
      subnet_ip     = "10.1.0.0/20"
      subnet_region = "us-central1"
    }
  ]
}
```

### Pattern 2: GitOps Infrastructure Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│        PATTERN 2: GITOPS INFRASTRUCTURE PIPELINE                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────┐     ┌──────────────┐     ┌────────────────┐           │
│  │Developer │────▶│   GitHub     │────▶│  Cloud Build   │           │
│  │ PR       │     │   (review)   │     │  (plan/apply)  │           │
│  └──────────┘     └──────────────┘     └────────────────┘           │
│                                                                        │
│  Workflow:                                                             │
│  1. Developer creates branch, modifies .tf files                      │
│  2. Opens PR → triggers Cloud Build (plan only)                       │
│  3. Plan output posted as PR comment                                  │
│  4. Reviewer approves PR                                              │
│  5. Merge → triggers Cloud Build (apply)                              │
│  6. State updated in GCS                                              │
│  7. Drift detection runs on schedule                                  │
│                                                                        │
│  Branch strategy:                                                      │
│  ┌────────────────────────────────────────────┐                      │
│  │ main ──────────────────────── (prod apply) │                      │
│  │   └── feature/* ── (plan only on PR)       │                      │
│  │                                             │                      │
│  │ environments/dev ─────────── (dev apply)    │                      │
│  │ environments/staging ─────── (stg apply)    │                      │
│  └────────────────────────────────────────────┘                      │
│                                                                        │
│  Security:                                                             │
│  • Least-privilege SA per environment                                  │
│  • Workload Identity Federation (no key files)                        │
│  • Policy-as-Code (OPA/Sentinel) in CI                                │
│  • State bucket with versioning + CMEK                                │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Deployment Manager to Terraform Migration

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: PHASED MIGRATION (DM → TERRAFORM)                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Phase 1: Assessment & Planning                                        │
│  ┌──────────────────────────────────────────────────────┐            │
│  │ • Inventory all DM deployments                       │            │
│  │ • Map resources to Terraform equivalents             │            │
│  │ • Identify dependencies between deployments          │            │
│  │ • Prioritize: start with stateless, low-risk         │            │
│  └──────────────────────────────────────────────────────┘            │
│                                                                        │
│  Phase 2: Write Terraform + Import (per deployment)                   │
│  ┌──────────────────────────────────────────────────────┐            │
│  │ • Write HCL for each resource                        │            │
│  │ • terraform import (no changes to infra)             │            │
│  │ • terraform plan → should show no changes            │            │
│  │ • Fix any drift between DM and actual state          │            │
│  └──────────────────────────────────────────────────────┘            │
│                                                                        │
│  Phase 3: Cut Over (per deployment)                                   │
│  ┌──────────────────────────────────────────────────────┐            │
│  │ • Delete DM deployment with --delete-policy=ABANDON  │            │
│  │ • Verify terraform plan still shows no changes       │            │
│  │ • Terraform now owns the resources                   │            │
│  │ • Set up CI/CD pipeline for Terraform                │            │
│  └──────────────────────────────────────────────────────┘            │
│                                                                        │
│  Phase 4: Validation & Cleanup                                        │
│  ┌──────────────────────────────────────────────────────┐            │
│  │ • Run terraform plan on schedule (drift detection)   │            │
│  │ • Remove old DM template files from repos            │            │
│  │ • Update documentation and runbooks                  │            │
│  │ • Train team on Terraform workflows                  │            │
│  └──────────────────────────────────────────────────────┘            │
│                                                                        │
│  Timeline: Start with network/compute (well-understood),              │
│  then databases, then IAM (most complex dependencies)                 │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Migration Script:**

```bash
#!/bin/bash
# migrate-deployment.sh — Migrate one DM deployment to Terraform

DEPLOYMENT=$1
PROJECT=$2

echo "=== Phase 1: Export DM resources ==="
gcloud deployment-manager resources list \
    --deployment="$DEPLOYMENT" \
    --format="table(name,type,id)" \
    --project="$PROJECT"

echo "=== Phase 2: Bulk export as Terraform ==="
gcloud beta resource-config bulk-export \
    --project="$PROJECT" \
    --resource-format=terraform \
    --path="./migrated/${DEPLOYMENT}"

echo "=== Phase 3: Initialize Terraform ==="
cd "./migrated/${DEPLOYMENT}"
terraform init

echo "=== Phase 4: Import resources ==="
# Generate import commands from DM resource list
gcloud deployment-manager resources list \
    --deployment="$DEPLOYMENT" \
    --format="csv[no-heading](name,type,id)" \
    --project="$PROJECT" | while IFS=',' read -r NAME TYPE ID; do
    
    TF_TYPE=$(echo "$TYPE" | sed 's/compute.v1.instance/google_compute_instance/' \
        | sed 's/compute.v1.network/google_compute_network/' \
        | sed 's/storage.v1.bucket/google_storage_bucket/')
    
    echo "terraform import ${TF_TYPE}.${NAME} ${ID}"
done

echo "=== Phase 5: Validate (plan should show no changes) ==="
terraform plan

echo "=== Phase 6: Abandon DM deployment ==="
echo "Run: gcloud deployment-manager deployments delete $DEPLOYMENT --delete-policy=ABANDON"
```

---

## Quick Reference

| Tool | Action | Command |
|------|--------|---------|
| DM | Create deployment | `gcloud deployment-manager deployments create NAME --config=FILE` |
| DM | Preview | `gcloud deployment-manager deployments create NAME --config=FILE --preview` |
| DM | Update | `gcloud deployment-manager deployments update NAME --config=FILE` |
| DM | Delete (keep resources) | `gcloud deployment-manager deployments delete NAME --delete-policy=ABANDON` |
| DM | List resources | `gcloud deployment-manager resources list --deployment=NAME` |
| DM | List types | `gcloud deployment-manager types list` |
| TF | Initialize | `terraform init` |
| TF | Plan | `terraform plan -out=tfplan` |
| TF | Apply | `terraform apply tfplan` |
| TF | Import | `terraform import RESOURCE_TYPE.NAME RESOURCE_ID` |
| TF | State list | `terraform state list` |
| TF | Destroy | `terraform destroy` |
| TF | Format | `terraform fmt -recursive` |
| TF | Validate | `terraform validate` |
| GCP | Export as TF | `gcloud beta resource-config bulk-export --resource-format=terraform` |

---

## Console Walkthrough: Deleting Deployments

When you delete a Deployment Manager deployment, you have a critical choice: **delete the resources** or **keep them**. Understanding this difference saves you from accidentally destroying production infrastructure.

### Abandon vs Delete: What Happens to Resources?

| Delete Policy | What Happens | When to Use |
|---------------|-------------|-------------|
| **DELETE** (default) | Deployment AND all resources are destroyed | Tearing down test/dev environments |
| **ABANDON** | Deployment is removed, but resources keep running | Migrating to Terraform, or detaching from DM |

> **⚠️ Critical:** The default behavior is **DELETE** — it will destroy your VMs, networks, buckets, and everything else in the deployment. Always double-check which policy you're using.

### Deleting from the Console

```
Console → Deployment Manager → Deployments
├── Select the deployment checkbox
├── Click DELETE at the top
├── Choose delete policy:
│   ├── "Delete deployment and all resources" (default — DESTRUCTIVE)
│   └── "Abandon — delete deployment but keep resources"
└── Confirm
```

**Step by step:**

1. Go to **Deployment Manager → Deployments** in the Console
2. Find the deployment you want to remove
3. Check the box next to it
4. Click **DELETE** at the top of the list
5. **Read the confirmation dialog carefully** — it tells you which policy will be used
6. Confirm the deletion

### Deleting via CLI

```bash
# Delete deployment AND destroy all resources (default — DESTRUCTIVE!)
gcloud deployment-manager deployments delete my-deployment

# Delete deployment but KEEP all resources running
gcloud deployment-manager deployments delete my-deployment \
    --delete-policy=ABANDON

# Skip confirmation prompt
gcloud deployment-manager deployments delete my-deployment --quiet

# Preview what would be deleted (dry run)
gcloud deployment-manager deployments delete my-deployment \
    --preview
```

### Before You Delete: Check What's in the Deployment

Always list the resources first so you know what will be affected:

```bash
# List all resources in the deployment
gcloud deployment-manager resources list \
    --deployment=my-deployment

# Get detailed output
gcloud deployment-manager resources list \
    --deployment=my-deployment \
    --format="table(name,type,id,update.state)"
```

Example output:

```
NAME              TYPE                    ID                  STATE
my-vm             compute.v1.instance     1234567890123456    COMPLETED
my-network        compute.v1.network      9876543210987654    COMPLETED
my-firewall       compute.v1.firewall     1122334455667788    COMPLETED
my-bucket         storage.v1.bucket       my-bucket           COMPLETED
```

> All four of these resources will be **destroyed** if you use the default DELETE policy.

### Common Scenarios

**Scenario 1: Tear down a test environment**

```bash
# Everything can go — use default delete
gcloud deployment-manager deployments delete test-env --quiet
```

**Scenario 2: Migrating to Terraform**

```bash
# Keep resources running, just remove DM's control
gcloud deployment-manager deployments delete prod-infra \
    --delete-policy=ABANDON

# Now import resources into Terraform
terraform import google_compute_instance.my_vm my-project/us-central1-a/my-vm
```

**Scenario 3: Deployment is stuck in a bad state**

```bash
# If a deployment failed mid-update and can't be fixed
# Abandon it to preserve resources, then recreate if needed
gcloud deployment-manager deployments delete broken-deployment \
    --delete-policy=ABANDON
```

### What If Delete Fails?

Sometimes a deployment delete fails because a resource can't be deleted (e.g., a bucket with objects, or a network with VMs still using it).

```bash
# Check the deployment for errors
gcloud deployment-manager deployments describe my-deployment

# If stuck, abandon instead
gcloud deployment-manager deployments delete my-deployment \
    --delete-policy=ABANDON

# Then manually clean up the problem resources
gsutil rm -r gs://my-bucket/**
gcloud compute instances delete my-vm --zone=us-central1-a
```

---

## What's Next?

Continue to **Chapter 35: Infrastructure Manager** → `35-infrastructure-manager.md`
