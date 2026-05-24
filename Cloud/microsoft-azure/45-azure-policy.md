# Chapter 45: Azure Policy

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Policy Fundamentals](#part-1-azure-policy-fundamentals)
- [Part 2: Policy Definitions](#part-2-policy-definitions)
- [Part 3: Assigning Policies (Portal Walkthrough)](#part-3-assigning-policies-portal-walkthrough)
- [Part 4: Policy Initiatives (Groups of Policies)](#part-4-policy-initiatives-groups-of-policies)
- [Part 5: Compliance Dashboard](#part-5-compliance-dashboard)
- [Part 6: Remediation](#part-6-remediation)
- [Part 7: Custom Policy Definitions](#part-7-custom-policy-definitions)
- [Part 8: Terraform & az CLI Reference](#part-8-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Policy enforces rules and compliance across your Azure resources. While RBAC controls WHO can do things, Policy controls WHAT can be done. For example: "All resources must have tags", "Only certain VM sizes are allowed", or "Storage accounts must use encryption."

```
What you'll learn:
├── Azure Policy Fundamentals
│   ├── What is Azure Policy (governance & compliance)
│   ├── Policy vs RBAC (complementary, not competing)
│   └── Policy effects (deny, audit, modify, deploy)
├── Policy Definitions (built-in & custom)
├── Assigning Policies (Portal)
├── Policy Initiatives (groups of policies)
├── Compliance Dashboard (see what's compliant)
├── Remediation (auto-fix non-compliant resources)
├── Custom Policy Definitions
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Azure Policy Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE POLICY OVERVIEW                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ RBAC vs Policy:                                                      │
│ ├── RBAC: WHO can access? (people/apps)                          │
│ │   "Developers can create VMs in rg-dev"                       │
│ └── Policy: WHAT is allowed? (rules/compliance)                 │
│     "VMs must be Standard_B2s or smaller"                       │
│     "All resources must have a CostCenter tag"                  │
│                                                                       │
│ Policy effects (what happens when rule is violated):               │
│ ├── Deny → Block resource creation/modification               │
│ │   ⚡ "You cannot create a VM without tags"                  │
│ ├── Audit → Allow but flag as non-compliant                    │
│ │   ⚡ "This VM doesn't have tags (logged, not blocked)"      │
│ ├── AuditIfNotExists → Check if related resource exists       │
│ │   ⚡ "VM exists but has no diagnostic settings"             │
│ ├── DeployIfNotExists → Auto-create missing resources         │
│ │   ⚡ "Auto-enable diagnostics on all VMs"                   │
│ ├── Modify → Auto-add/change properties                       │
│ │   ⚡ "Auto-add tag Environment=Dev to new resources"        │
│ ├── Append → Add fields to resources                           │
│ └── Disabled → Turn off the policy temporarily                │
│                                                                       │
│ Evaluation:                                                          │
│ ├── New/updated resources: Evaluated immediately               │
│ ├── Existing resources: Evaluated every 24 hours               │
│ └── On-demand: Trigger evaluation manually                     │
│                                                                       │
│ ⚡ Policies are FREE! No additional cost.                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Policy Definitions

```
Common built-in policy definitions:

Tags:
├── Require a tag and its value on resources
├── Inherit a tag from the resource group
└── Add or replace a tag on resources

Compute:
├── Allowed virtual machine size SKUs
├── Audit VMs that do not use managed disks
└── Deploy default Microsoft IaC Anti-malware extension

Storage:
├── Storage accounts should restrict network access
├── Secure transfer (HTTPS) should be enabled
└── Storage accounts should use customer-managed key for encryption

Networking:
├── Network interfaces should not have public IPs
├── Subnets should have a Network Security Group
└── Public network access should be disabled for PaaS resources

Security:
├── Azure Defender should be enabled
├── MFA should be enabled on accounts with owner permissions
└── Vulnerabilities in security configuration should be remediated

Monitoring:
├── Azure Monitor should collect activity logs
├── Diagnostic settings should be enabled
└── Log Analytics agent should be installed on VMs

⚡ There are 600+ built-in policies!
```

---

## Part 3: Assigning Policies (Portal Walkthrough)

```
Console → Policy → Assignments → [+ Assign policy]

┌─────────────────────────────────────────────────────────────────────┐
│           ASSIGN POLICY                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Basics ──                                                        │
│ Scope: [subscription: prod-subscription ▼]                        │
│ ├── Can be: Management Group, Subscription, or Resource Group   │
│ └── Exclusions: [rg-sandbox ▼] (skip this RG)                  │
│                                                                       │
│ Policy definition: [Require a tag and its value on resources ▼] │
│ Assignment name: [Require CostCenter tag]                         │
│                                                                       │
│ ── Parameters ──                                                    │
│ Tag name: [CostCenter]                                             │
│ Tag value: (leave empty to require any value)                    │
│                                                                       │
│ ── Remediation ──                                                   │
│ Create remediation task: ☑                                        │
│ ⚡ For Modify/DeployIfNotExists: auto-fix existing resources    │
│ Managed identity: Auto-created (needed for remediation)          │
│                                                                       │
│ ── Non-compliance messages ──                                      │
│ Message: "All resources must have a CostCenter tag.              │
│           Contact platform team for help."                         │
│ ⚡ Users see this message when their deployment is denied!      │
│                                                                       │
│ ── Enforcement ──                                                   │
│ Policy enforcement: ● Enabled  ○ Disabled                       │
│ ⚡ Disabled = audit only (report but don't block)               │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Policy Initiatives (Groups of Policies)

```
┌─────────────────────────────────────────────────────────────────────┐
│           INITIATIVES (Policy Sets)                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Initiative = A group of related policies assigned together        │
│                                                                       │
│ Built-in initiatives:                                               │
│ ├── CIS Microsoft Azure Foundations Benchmark                    │
│ │   (100+ security policies based on CIS standards)            │
│ ├── Azure Security Benchmark                                     │
│ │   (Microsoft's recommended security baseline)                │
│ ├── ISO 27001:2013                                               │
│ ├── NIST SP 800-53                                               │
│ ├── PCI DSS 3.2.1                                               │
│ └── HIPAA/HITRUST                                                │
│                                                                       │
│ Create custom initiative:                                           │
│ Policy → Definitions → [+ Initiative definition]               │
│                                                                       │
│ Name: [Company Security Baseline]                                  │
│ Category: [Custom]                                                  │
│ Add policies:                                                        │
│ ├── Require CostCenter tag                                       │
│ ├── Allowed VM sizes                                              │
│ ├── Require HTTPS on Storage Accounts                           │
│ ├── Require NSG on subnets                                      │
│ └── Enable diagnostic settings                                   │
│                                                                       │
│ [Save] → Then assign the initiative (same as assigning a policy)│
│                                                                       │
│ ⚡ Initiatives simplify management — one assignment instead of 5!│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Compliance Dashboard

```
Console → Policy → Compliance

┌─────────────────────────────────────────────────────────────────────┐
│           COMPLIANCE DASHBOARD                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Overall compliance: 78%                                              │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────┐    │
│ │ Policy/Initiative          │ Compliant │ Non-Compliant    │    │
│ ├────────────────────────────┼───────────┼──────────────────┤    │
│ │ Require CostCenter tag     │ 45/58     │ 13 ⚠️            │    │
│ │ Allowed VM sizes           │ 12/12     │ 0 ✅             │    │
│ │ HTTPS on Storage           │ 8/10      │ 2 ⚠️             │    │
│ │ Azure Security Benchmark   │ 85%       │ 23 resources ⚠️  │    │
│ └────────────────────────────┴───────────┴──────────────────┘    │
│                                                                       │
│ Click any policy → See non-compliant resources                   │
│ Click resource → See why it's non-compliant                     │
│                                                                       │
│ Export compliance data:                                              │
│ ├── Azure Resource Graph queries                                 │
│ ├── Export to CSV                                                 │
│ └── Stream to Event Hub (for SIEM)                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Remediation

```
Policy → Remediation → [+ Remediation task]

For Modify/DeployIfNotExists policies:
├── Remediation fixes EXISTING non-compliant resources
├── New resources are automatically evaluated
├── Example: Policy "Auto-add Environment tag" with Modify effect
│   Remediation → Goes through all resources without the tag → Adds it

Remediation task:
├── Policy: [Inherit CostCenter tag from resource group]
├── Scope: [rg-myapp-prod]
├── Resources to remediate: [All non-compliant]
├── [Remediate]
│
└── Progress: 13 resources → 13 remediated ✅

⚡ Only Modify, Append, and DeployIfNotExists support remediation.
⚡ Deny and Audit effects cannot remediate (they only block/report).
```

---

## Part 7: Custom Policy Definitions

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Storage/storageAccounts"
        },
        {
          "field": "Microsoft.Storage/storageAccounts/minimumTlsVersion",
          "notEquals": "TLS1_2"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

```
This custom policy: "Deny storage accounts that don't use TLS 1.2"

Policy rule structure:
├── if: condition (when does this apply?)
│   ├── field: Resource property to check
│   ├── equals / notEquals / contains / in / like
│   └── allOf / anyOf / not (combine conditions)
└── then: effect (what to do?)
    └── effect: deny / audit / modify / deployIfNotExists
```

---

## Part 8: Terraform & az CLI Reference

### Terraform

```hcl
# Assign built-in policy
resource "azurerm_resource_group_policy_assignment" "require_tags" {
  name                 = "require-costcenter-tag"
  resource_group_id    = azurerm_resource_group.main.id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/1e30110a-5ceb-460c-a204-c1c3969c6d62"

  parameters = jsonencode({
    tagName = { value = "CostCenter" }
  })

  non_compliance_message {
    content = "All resources must have a CostCenter tag."
  }
}
```

### Bicep

```bicep
// Assign built-in policy (Require tag)
resource policyAssignment 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'require-costcenter-tag'
  properties: {
    displayName: 'Require CostCenter tag on all resources'
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/1e30110a-5ceb-460c-a204-c1c3969c6d62'
    parameters: {
      tagName: { value: 'CostCenter' }
    }
    nonComplianceMessages: [
      { message: 'All resources must have a CostCenter tag.' }
    ]
  }
}

// Custom policy definition
resource customPolicy 'Microsoft.Authorization/policyDefinitions@2021-06-01' = {
  name: 'deny-public-ip'
  properties: {
    displayName: 'Deny Public IP creation'
    policyType: 'Custom'
    mode: 'All'
    policyRule: {
      if: {
        field: 'type'
        equals: 'Microsoft.Network/publicIPAddresses'
      }
      then: {
        effect: 'Deny'
      }
    }
  }
}
```
  --name "deny-storage-no-tls12" \
  --display-name "Deny storage without TLS 1.2" \
  --rules @policy-rule.json \
  --mode Indexed

# Remove policy assignment
az policy assignment delete --name "require-tags" \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod
```

---

## Real-World Patterns

### Pattern 1: Governance at Scale with Initiatives

```
┌─────────────────────────────────────────────────┐
│       Enterprise Governance Architecture        │
├─────────────────────────────────────────────────┤
│                                                 │
│  Management Group (Root)                        │
│  └── Initiative: "Security Baseline"            │
│       ├── Require encryption on storage         │
│       ├── Require HTTPS on web apps             │
│       ├── Deny public IP on VMs                 │
│       └── Require tags (Environment, Owner)     │
│                                                 │
│  ┌─────────────┐  ┌─────────────┐              │
│  │ Production  │  │ Development │              │
│  │ Subscription│  │ Subscription│              │
│  ├─────────────┤  ├─────────────┤              │
│  │ + Deny      │  │ + Audit     │              │
│  │   delete    │  │   only mode │              │
│  │   locks     │  │             │              │
│  └─────────────┘  └─────────────┘              │
│                                                 │
│  Compliance dashboard shows % across all subs   │
└─────────────────────────────────────────────────┘
```

### Pattern 2: Cost Control with Allowed SKU Policy

```
┌─────────────────────────────────────────────────┐
│       Cost Control via Policy                   │
├─────────────────────────────────────────────────┤
│                                                 │
│  Policy: "Allowed VM SKUs"                      │
│  Effect: Deny                                   │
│  Scope: Dev/Test Subscription                   │
│                                                 │
│  Allowed:           Blocked:                    │
│  ✓ Standard_B2s     ✗ Standard_E64s_v5          │
│  ✓ Standard_B4ms    ✗ Standard_M128s            │
│  ✓ Standard_D2s_v5  ✗ Standard_NC24s_v3         │
│                                                 │
│  Result: Developers can't accidentally create   │
│  expensive VMs in non-production environments   │
│                                                 │
│  Remediation Task: Auto-tag non-compliant       │
│  resources for review                           │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Azure Policy = Enforce WHAT is allowed (governance & compliance)
RBAC = Control WHO has access (identity & permissions)

Effects: Deny (block), Audit (report), Modify (auto-fix),
         DeployIfNotExists (auto-create), Disabled (off)

Initiative = Group of related policies (assign together)
Built-in: CIS Benchmark, Azure Security Benchmark, PCI DSS, etc.

Compliance Dashboard: See % compliant, drill into violations
Remediation: Auto-fix non-compliant resources (Modify/Deploy effects)

Scope: Management Group > Subscription > Resource Group
Exclusions: Skip specific RGs from policy scope
Evaluation: Immediate (new resources), every 24h (existing)

⚡ Policies are FREE!
⚡ Start with Audit effect, then switch to Deny after review
```

---

## What's Next?

Next chapter: [Chapter 46: Microsoft Defender for Cloud](46-defender-for-cloud.md) — Cloud security posture management, threat protection, and security recommendations.
