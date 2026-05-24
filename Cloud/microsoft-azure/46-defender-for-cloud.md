# Chapter 46: Microsoft Defender for Cloud

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Defender Fundamentals](#part-1-defender-fundamentals)
- [Part 2: Security Posture (CSPM)](#part-2-security-posture-cspm)
- [Part 3: Defender Plans (Portal Walkthrough)](#part-3-defender-plans-portal-walkthrough)
- [Part 4: Security Recommendations](#part-4-security-recommendations)
- [Part 5: Threat Protection & Alerts](#part-5-threat-protection--alerts)
- [Part 6: Regulatory Compliance](#part-6-regulatory-compliance)
- [Part 7: az CLI Reference](#part-7-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Microsoft Defender for Cloud is Azure's cloud security center. It continuously assesses your Azure resources for security weaknesses, gives you a security score with recommendations, and provides threat protection (detecting attacks in real-time). It has a free tier (basic posture) and paid plans (advanced protection per service).

```
What you'll learn:
├── Defender Fundamentals
│   ├── CSPM (Cloud Security Posture Management)
│   ├── CWP (Cloud Workload Protection)
│   └── Free vs Paid tiers
├── Security Posture & Secure Score
├── Defender Plans (per-service protection)
├── Security Recommendations (actionable fixes)
├── Threat Protection & Security Alerts
├── Regulatory Compliance (benchmarks)
├── az CLI reference
└── Quick reference
```

---

## Part 1: Defender Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEFENDER FOR CLOUD OVERVIEW                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Two main capabilities:                                               │
│                                                                       │
│ 1. CSPM (Cloud Security Posture Management) — FREE               │
│    ├── Continuous assessment of your security posture            │
│    ├── Secure Score (0-100%)                                     │
│    ├── Security recommendations                                  │
│    ├── Azure Security Benchmark                                  │
│    └── "Here's what's wrong, here's how to fix it"             │
│                                                                       │
│ 2. CWP (Cloud Workload Protection) — PAID (per plan)             │
│    ├── Real-time threat detection                                │
│    ├── Security alerts ("Someone is attacking your VM!")        │
│    ├── Vulnerability scanning                                    │
│    ├── Just-in-time VM access                                    │
│    ├── Adaptive application controls                             │
│    └── "Something suspicious is happening, take action!"       │
│                                                                       │
│ Available for:                                                       │
│ ├── Azure resources (all types)                                  │
│ ├── AWS accounts (multi-cloud!)                                  │
│ ├── GCP projects (multi-cloud!)                                  │
│ └── On-premises servers (hybrid)                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Security Posture (CSPM)

```
Console → Microsoft Defender for Cloud → Overview

┌─────────────────────────────────────────────────────────────────────┐
│           SECURE SCORE                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Secure Score: 72/100 (72%)                                          │
│                                                                       │
│ ████████████████████████████░░░░░░░░░░ 72%                        │
│                                                                       │
│ Score breakdown by category:                                        │
│ ├── Identity & Access: 85% ████████░░                             │
│ ├── Network Security: 70% ███████░░░                              │
│ ├── Data Security: 65% ██████░░░░                                 │
│ ├── Compute Security: 80% ████████░░                              │
│ └── Application Security: 60% ██████░░░░                          │
│                                                                       │
│ How to improve score:                                                │
│ Each recommendation has a score impact (points you'll gain)      │
│ Fix high-impact recommendations first!                             │
│                                                                       │
│ ⚡ Secure Score is FREE and always-on.                            │
│ ⚡ Goal: Get as close to 100% as possible.                       │
│ ⚡ Focus on high-severity recommendations first.                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Defender Plans (Portal Walkthrough)

```
Defender for Cloud → Environment settings → [subscription] → Defender plans

┌─────────────────────────────────────────────────────────────────────┐
│           DEFENDER PLANS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Plan                        │ Price          │ What it protects    │
│ ────────────────────────────┼────────────────┼─────────────────────│
│ Foundational CSPM           │ FREE ✅        │ Score, recommendations│
│ Defender CSPM               │ $5/server/mo   │ Attack paths, governance│
│ Defender for Servers P1     │ $5/server/mo   │ Endpoint detection   │
│ Defender for Servers P2     │ $15/server/mo  │ + Vuln scanning, JIT│
│ Defender for Containers     │ $7/vCore/mo    │ AKS, registry scanning│
│ Defender for App Service    │ $15/instance/mo│ Web app threats      │
│ Defender for Databases      │ Various        │ SQL, Cosmos, etc.    │
│ Defender for Storage        │ $10/account/mo │ Malware, anomalies  │
│ Defender for Key Vault      │ $0.02/10k ops  │ Unusual access       │
│ Defender for DNS            │ $0.7/million q │ Malicious DNS queries│
│                                                                       │
│ ⚡ Start with FREE tier. Enable paid plans for production.      │
│ ⚡ Enable per plan — you don't need all of them.               │
│ ⚡ Defender for Servers is most commonly enabled.               │
│                                                                       │
│ Enable: Toggle [On/Off] per plan → [Save]                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Security Recommendations

```
Defender for Cloud → Recommendations

┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY RECOMMENDATIONS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Severity │ Recommendation                       │ Resources │ Score│
│ ─────────┼──────────────────────────────────────┼───────────┼──────│
│ 🔴 High  │ MFA should be enabled for owner accts│ 3 of 5   │ +10 │
│ 🔴 High  │ Management ports should be closed    │ 4 VMs     │ +8  │
│ 🟡 Med   │ Storage should use private endpoints │ 2 accts   │ +5  │
│ 🟡 Med   │ SQL should have TDE enabled          │ 1 DB      │ +3  │
│ 🔵 Low   │ Diagnostic logs should be enabled    │ 12 res    │ +2  │
│                                                                       │
│ Click any recommendation →                                        │
│ ├── Description (what's the risk?)                               │
│ ├── Remediation steps (how to fix, step by step!)              │
│ ├── Affected resources (which ones are non-compliant)          │
│ ├── [Fix] button (auto-remediate if supported)                 │
│ └── [Exempt] button (exclude with justification)               │
│                                                                       │
│ Exemptions:                                                          │
│ ├── Waiver: Accept risk, reviewed and approved                  │
│ ├── Mitigated: Fixed by other means (3rd party tool)           │
│ └── Set expiration date for exemption                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Threat Protection & Alerts

```
Defender for Cloud → Security alerts

┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY ALERTS                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Alert examples (Defender for Servers):                               │
│ 🔴 "Suspicious process execution on VM"                            │
│ 🔴 "SSH brute force attack detected (1000+ failed logins)"       │
│ 🟡 "Unusual outbound network traffic from VM"                     │
│ 🟡 "Crypto mining activity detected"                               │
│                                                                       │
│ Alert examples (Defender for Storage):                              │
│ 🔴 "Access from a Tor exit node"                                  │
│ 🟡 "Unusual blob download pattern"                                │
│                                                                       │
│ Alert examples (Defender for SQL):                                  │
│ 🔴 "SQL injection attempt detected"                               │
│ 🟡 "Login from unusual location"                                  │
│                                                                       │
│ Alert details:                                                       │
│ ├── Severity (High/Medium/Low)                                   │
│ ├── Description (what happened)                                  │
│ ├── Affected resource                                             │
│ ├── Attack timeline                                               │
│ ├── MITRE ATT&CK mapping                                        │
│ ├── Remediation steps                                             │
│ └── [Trigger Logic App] for automated response                  │
│                                                                       │
│ Just-in-Time (JIT) VM Access (Defender for Servers P2):            │
│ ├── Management ports (SSH/RDP) CLOSED by default                │
│ ├── Request access → Approved → Port opens for limited time   │
│ ├── Auto-closes after time expires                               │
│ └── Reduces attack surface for brute force attacks              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Regulatory Compliance

```
Defender for Cloud → Regulatory compliance

┌─────────────────────────────────────────────────────────────────────┐
│           COMPLIANCE DASHBOARD                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Standard                    │ Compliance │ Controls              │
│ ────────────────────────────┼────────────┼───────────────────────│
│ Azure Security Benchmark    │ 72%        │ 38/52 passed         │
│ CIS Azure Foundations 1.4   │ 65%        │ 42/65 passed         │
│ PCI DSS 3.2.1              │ 58%        │ 70/120 passed        │
│ ISO 27001:2013             │ 80%        │ 45/56 passed         │
│ NIST SP 800-53 Rev 5       │ 62%        │ 95/153 passed        │
│                                                                       │
│ Add standard:                                                        │
│ [+ Add more standards] → Select from list → Assign            │
│                                                                       │
│ ⚡ Maps Azure Policy controls to compliance framework controls. │
│ ⚡ Export compliance report as PDF for auditors.                 │
│ ⚡ Azure Security Benchmark is added by default.                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform, Bicep & az CLI Reference

### Terraform

```hcl
# Enable Defender for Servers
resource "azurerm_security_center_subscription_pricing" "servers" {
  tier          = "Standard"
  resource_type = "VirtualMachines"
  subplan       = "P2"
}

# Enable Defender for Storage
resource "azurerm_security_center_subscription_pricing" "storage" {
  tier          = "Standard"
  resource_type = "StorageAccounts"
}

# Enable Defender for SQL
resource "azurerm_security_center_subscription_pricing" "sql" {
  tier          = "Standard"
  resource_type = "SqlServers"
}

# Enable Defender for App Service
resource "azurerm_security_center_subscription_pricing" "appservice" {
  tier          = "Standard"
  resource_type = "AppServices"
}

# Enable Defender for Key Vault
resource "azurerm_security_center_subscription_pricing" "keyvault" {
  tier          = "Standard"
  resource_type = "KeyVaults"
}

# Enable Defender for Containers
resource "azurerm_security_center_subscription_pricing" "containers" {
  tier          = "Standard"
  resource_type = "Containers"
}

# Auto-provision Log Analytics agent
resource "azurerm_security_center_auto_provisioning" "main" {
  auto_provision = "On"
}

# Set security contact
resource "azurerm_security_center_contact" "main" {
  name                = "default"
  email               = "security@company.com"
  phone               = "+91-9999999999"
  alert_notifications = true
  alerts_to_admins    = true
}
```

### Bicep

```bicep
// Enable Defender for Servers P2
resource defenderServers 'Microsoft.Security/pricings@2023-01-01' = {
  name: 'VirtualMachines'
  properties: {
    pricingTier: 'Standard'
    subPlan: 'P2'
  }
}

// Enable Defender for Storage
resource defenderStorage 'Microsoft.Security/pricings@2023-01-01' = {
  name: 'StorageAccounts'
  properties: {
    pricingTier: 'Standard'
  }
}

// Enable Defender for SQL
resource defenderSql 'Microsoft.Security/pricings@2023-01-01' = {
  name: 'SqlServers'
  properties: {
    pricingTier: 'Standard'
  }
}

// Enable Defender for Containers
resource defenderContainers 'Microsoft.Security/pricings@2023-01-01' = {
  name: 'Containers'
  properties: {
    pricingTier: 'Standard'
  }
}

// Security contact
resource securityContact 'Microsoft.Security/securityContacts@2020-01-01-preview' = {
  name: 'default'
  properties: {
    emails: 'security@company.com'
    phone: '+91-9999999999'
    alertNotifications: {
      state: 'On'
      minimalSeverity: 'Medium'
    }
    notificationsByRole: {
      state: 'On'
      roles: ['Owner', 'Contributor']
    }
  }
}
```

### az CLI

```bash
# List all Defender pricing tiers (which plans are enabled)
az security pricing list --output table

# Enable Defender for Servers (P2)
az security pricing create --name VirtualMachines --tier Standard

# Enable Defender for Storage
az security pricing create --name StorageAccounts --tier Standard

# Enable Defender for SQL
az security pricing create --name SqlServers --tier Standard

# Enable Defender for App Service
az security pricing create --name AppServices --tier Standard

# Enable Defender for Key Vault
az security pricing create --name KeyVaults --tier Standard

# Enable Defender for Containers
az security pricing create --name Containers --tier Standard

# Enable Defender for DNS
az security pricing create --name Dns --tier Standard

# Disable a plan (revert to free)
az security pricing create --name VirtualMachines --tier Free

# List security recommendations
az security assessment list --output table

# Get specific recommendation details
az security assessment show --name <assessment-id>

# List security alerts
az security alert list --output table

# Get alert details
az security alert show --name <alert-name> --location centralindia

# Dismiss an alert
az security alert update --name <alert-name> --status Dismissed --location centralindia

# Activate an alert (re-open dismissed)
az security alert update --name <alert-name> --status Active --location centralindia

# Get secure score
az security secure-score list --output table

# Get secure score controls (breakdown)
az security secure-score-controls list --output table

# List compliance standards
az security regulatory-compliance-standards list --output table

# List compliance controls for a standard
az security regulatory-compliance-controls list \
  --standard-name "Azure-CIS-1.1.0" --output table

# Set security contact
az security contact create --name default \
  --email "security@company.com" \
  --phone "+91-9999999999" \
  --alert-notifications on \
  --alerts-admins on

# Enable auto-provisioning of monitoring agent
az security auto-provisioning-setting update --name default --auto-provision on

# List JIT VM access policies
az security jit-policy list --output table

# Configure JIT access for a VM
az security jit-policy create \
  --name default \
  --resource-group rg-prod \
  --location centralindia \
  --virtual-machines '[{"id":"/subscriptions/.../vm1","ports":[{"number":22,"protocol":"TCP","allowedSourceAddressPrefix":"*","maxRequestAccessDuration":"PT3H"}]}]'
```

---

## Real-World Patterns

### Pattern 1: Full Security Posture Management

```
┌─────────────────────────────────────────────────┐
│       Security Posture Architecture             │
├─────────────────────────────────────────────────┤
│                                                 │
│  Defender for Cloud (CSPM)                      │
│       │                                         │
│       ├── Secure Score: 78/100                  │
│       │    ├── Enable MFA (High)                │
│       │    ├── Encrypt disks (Medium)           │
│       │    └── Close open ports (High)          │
│       │                                         │
│       ├── Regulatory Compliance                 │
│       │    ├── CIS Azure Benchmark              │
│       │    ├── PCI DSS 4.0                      │
│       │    └── ISO 27001                        │
│       │                                         │
│       └── Recommendations                      │
│            └── Auto-remediate with Logic Apps   │
│                                                 │
│  Defender Plans Enabled:                        │
│  ✓ Servers  ✓ Storage  ✓ SQL  ✓ Key Vault      │
│  ✓ App Service  ✓ Containers  ✓ DNS            │
└─────────────────────────────────────────────────┘
```

### Pattern 2: Threat Detection and Response

```
┌─────────────────────────────────────────────────┐
│       Automated Threat Response                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  Defender Alert ──→ Logic App Workflow           │
│  (Suspicious login)     │                       │
│                    ┌────┴────┐                  │
│                    │ Actions │                  │
│                    ├─────────┤                  │
│                    │ 1. Send Teams alert        │
│                    │ 2. Create ServiceNow ticket│
│                    │ 3. Block IP in NSG         │
│                    │ 4. Isolate VM              │
│                    └─────────┘                  │
│                                                 │
│  Integration: Sentinel SIEM for correlation     │
│  across all Azure + on-prem security signals    │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Defender for Cloud = Azure's security center

Free (CSPM):
  Secure Score (0-100%), Security Recommendations
  Azure Security Benchmark, Regulatory Compliance

Paid (CWP, per plan):
  Defender for Servers (endpoint detection, JIT, vuln scanning)
  Defender for Containers (AKS protection, image scanning)
  Defender for SQL (injection detection, anomaly detection)
  Defender for Storage (malware, unusual access)
  Defender for Key Vault, App Service, DNS, etc.

Security Recommendations: Actionable fixes with score impact
Security Alerts: Real-time threat detection notifications
Regulatory Compliance: Map to CIS, PCI DSS, ISO 27001, NIST

JIT VM Access: Open management ports only when needed
Multi-cloud: Also monitors AWS and GCP!

⚡ Enable free tier immediately. Add paid plans for production.
```

---

## What's Next?

Next chapter: [Chapter 47: Azure Network Security](47-network-security.md) — WAF, DDoS Protection, Azure Firewall, and Private Link.
