# Chapter 43: GuardDuty & Security Hub

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Amazon GuardDuty](#part-1-amazon-guardduty)
- [Part 2: Enabling GuardDuty (Full Portal Walkthrough)](#part-2-enabling-guardduty-full-portal-walkthrough)
- [Part 3: GuardDuty Findings & Response](#part-3-guardduty-findings--response)
- [Part 4: AWS Security Hub](#part-4-aws-security-hub)
- [Part 5: Enabling Security Hub (Full Portal Walkthrough)](#part-5-enabling-security-hub-full-portal-walkthrough)
- [Part 6: Other Security Services](#part-6-other-security-services)
- [Part 7: Terraform & CLI Examples](#part-7-terraform--cli-examples)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Threat Detection? Why Do We Need GuardDuty and Security Hub?

Imagine your house has multiple security systems: door sensors, window alarms, motion detectors, and security cameras. Each one generates alerts independently. **GuardDuty** is like a **smart alarm system** that uses AI to detect suspicious activity. **Security Hub** is like a **security control room** that shows all alerts from all systems on one screen.

**GuardDuty** automatically analyzes:
- 🌐 VPC Flow Logs ("Someone is port-scanning your servers")
- 📝 CloudTrail logs ("An API key is being used from a country you've never operated in")
- 📧 DNS logs ("An EC2 instance is communicating with a known malware server")

**Security Hub** aggregates findings from:
- GuardDuty (threat detection)
- Inspector (vulnerability scanning)
- Macie (sensitive data discovery)
- Firewall Manager, IAM Access Analyzer, and third-party tools

**Which security service for what?**
- **GuardDuty**: "Is anyone trying to break into my account?" (threat detection)
- **Inspector**: "Do my EC2 instances or containers have vulnerabilities?" (vulnerability scanning)
- **Macie**: "Is there sensitive data (PII, credit cards) in my S3 buckets?" (data discovery)
- **Detective**: "An incident happened — help me investigate" (root cause analysis)
- **Security Hub**: "Show me ALL security findings in one place" (aggregation)

Amazon GuardDuty uses ML and threat intelligence to detect threats. AWS Security Hub aggregates security findings from GuardDuty, Inspector, Macie, and third-party tools into a single dashboard with compliance checks.

```
What you'll learn:
├── GuardDuty (threat detection, ML-based)
├── Enabling GuardDuty (portal walkthrough)
├── GuardDuty findings & automated response
├── Security Hub (central security dashboard)
├── Enabling Security Hub (portal walkthrough)
├── Inspector, Macie, Detective (other security services)
└── Terraform & CLI examples
```

---

## Part 1: Amazon GuardDuty

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW GUARDDUTY WORKS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Data Sources → GuardDuty (ML + Threat Intel) → Findings           │
│                                                                       │
│ Input data sources (analyzed automatically):                       │
│ ├── CloudTrail Management Events                                │
│ ├── CloudTrail S3 Data Events                                   │
│ ├── VPC Flow Logs                                                │
│ ├── DNS Logs (Route 53 Resolver)                                │
│ ├── EKS Audit Logs + Runtime Monitoring                        │
│ ├── Lambda Network Activity                                     │
│ ├── RDS Login Activity                                           │
│ ├── S3 Malware Protection                                       │
│ └── EC2 Runtime Monitoring (agent-based)                       │
│                                                                       │
│ ⚠️ GuardDuty does NOT read your actual data/logs.                 │
│ It analyzes metadata/flow patterns independently.                 │
│ No impact on performance.                                          │
│                                                                       │
│ Detection categories:                                               │
│ ├── Reconnaissance: Port scanning, unusual API calls           │
│ ├── Instance compromise: Crypto mining, malware C&C            │
│ ├── Account compromise: Unusual logins, API from TOR           │
│ ├── Bucket compromise: Public access, unusual access patterns │
│ ├── Kubernetes: Suspicious pods, privilege escalation          │
│ └── Malware: Malicious files detected in EC2/S3               │
│                                                                       │
│ Finding severity:                                                   │
│ ├── High (7.0-8.9): Immediate action required                  │
│ ├── Medium (4.0-6.9): Investigate and remediate                │
│ └── Low (1.0-3.9): Suspicious but not immediately dangerous   │
│                                                                       │
│ Pricing (pay per data analyzed):                                    │
│ ├── CloudTrail: $4.00/million events (first 500M free)        │
│ ├── VPC Flow Logs: $1.00/GB (first 500 GB free)              │
│ ├── DNS Logs: $1.00/million queries (first 1M free)          │
│ └── 30-day free trial available                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Enabling GuardDuty (Full Portal Walkthrough)

```
Console → GuardDuty → Get Started → Enable GuardDuty

┌─────────────────────────────────────────────────────────────────┐
│           ENABLE GUARDDUTY                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Service role permissions:                                      │
│ GuardDuty creates a service-linked role to analyze:           │
│ ├── CloudTrail events                                         │
│ ├── VPC Flow Logs metadata                                    │
│ └── DNS query logs                                             │
│                                                                   │
│                    [Enable GuardDuty]                            │
└─────────────────────────────────────────────────────────────────┘

After enabling → Configure protection plans:

Console → GuardDuty → Settings → Protection plans

┌─────────────────────────────────────────────────────────────────┐
│           PROTECTION PLANS                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ☑ S3 Protection: ☑ Enabled                                   │
│   → Monitors S3 data events for suspicious access            │
│                                                                   │
│ ☑ EKS Protection:                                             │
│   ☑ EKS Audit Log Monitoring                                 │
│   ☑ EKS Runtime Monitoring                                   │
│   → Detects suspicious Kubernetes activity                   │
│                                                                   │
│ ☑ Malware Protection:                                         │
│   ☑ EBS Malware Protection                                   │
│   → Scans EBS volumes for malware when EC2 findings occur   │
│                                                                   │
│ ☑ RDS Protection: ☑ Enabled                                  │
│   → Monitors RDS login activity (brute force, etc.)         │
│                                                                   │
│ ☑ Lambda Protection: ☑ Enabled                               │
│   → Monitors Lambda network activity                         │
│                                                                   │
│ ☑ Runtime Monitoring (EC2): ☑ Enabled                        │
│   → Agent-based monitoring for EC2 instances                │
│   → Detects: Process injection, crypto mining, reverse shell│
│                                                                   │
│ ── Multi-account setup ──                                      │
│ Console → GuardDuty → Settings → Accounts                     │
│ ├── Delegated administrator: Management or Security account  │
│ ├── Auto-enable for new accounts: ☑                          │
│ └── Invite existing accounts                                  │
│ → ⚡ Always enable for all accounts in Organization          │
│                                                                   │
│ ── Trusted IP & Threat lists ──                                │
│ Console → GuardDuty → Settings → Lists                        │
│ Trusted IP list: IPs to never flag (your offices)            │
│ Threat IP list: Custom threat intelligence (your blocklist)  │
│                                                                   │
│ ── Findings export ──                                          │
│ Frequency: [Every 15 minutes ▼]                               │
│ S3 bucket: [guardduty-findings-bucket ▼]                      │
│ KMS key: [alias/guardduty-key ▼]                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: GuardDuty Findings & Response

```
┌─────────────────────────────────────────────────────────────────────┐
│           FINDINGS & AUTOMATED RESPONSE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Sample findings:                                                     │
│ ├── UnauthorizedAccess:EC2/MaliciousIPCaller.Custom             │
│ │   → EC2 instance communicating with known malicious IP       │
│ ├── CryptoCurrency:EC2/BitcoinTool.B!DNS                       │
│ │   → EC2 instance querying crypto mining DNS                  │
│ ├── Recon:EC2/PortProbeUnprotectedPort                          │
│ │   → Port scanning detected on unprotected port              │
│ ├── UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B           │
│ │   → Successful console login from unusual location          │
│ ├── Policy:S3/BucketPublicAccessGranted                        │
│ │   → S3 bucket made public                                    │
│ └── Backdoor:EC2/C&CActivity.B                                 │
│     → Instance communicating with command & control server    │
│                                                                       │
│ Automated response pattern:                                         │
│ GuardDuty Finding → EventBridge Rule → Lambda/Step Functions    │
│                                                                       │
│ Example: Auto-isolate compromised EC2                             │
│ 1. GuardDuty detects: CryptoCurrency:EC2/BitcoinTool            │
│ 2. EventBridge rule matches finding type                         │
│ 3. Lambda function:                                                │
│    a. Creates forensic snapshot of EBS volume                   │
│    b. Isolates instance (change SG to deny-all)               │
│    c. Sends alert to Slack/PagerDuty                           │
│    d. Creates Jira incident ticket                              │
│                                                                       │
│ Suppression rules:                                                   │
│ Console → GuardDuty → Settings → Suppression rules               │
│ → Auto-archive findings matching criteria (reduce noise)        │
│ → Example: Suppress known scanner IPs from your pen testing    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: AWS Security Hub

```
┌─────────────────────────────────────────────────────────────────────┐
│           AWS SECURITY HUB                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Central security dashboard that aggregates findings from    │
│ multiple AWS security services + third-party tools.               │
│                                                                       │
│ ┌──────────────┐                                                    │
│ │ Security Hub │◄── GuardDuty (threat detection)                 │
│ │              │◄── Inspector (vulnerability scanning)           │
│ │ ┌──────────┐ │◄── Macie (sensitive data detection)             │
│ │ │Findings  │ │◄── Firewall Manager                             │
│ │ │Dashboard │ │◄── IAM Access Analyzer                          │
│ │ │Compliance│ │◄── AWS Config (compliance)                      │
│ │ │Standards │ │◄── Third-party tools (Palo Alto, CrowdStrike)  │
│ │ └──────────┘ │                                                    │
│ └──────────────┘                                                    │
│                                                                       │
│ Key features:                                                        │
│ ├── Unified security findings (ASFF format)                     │
│ ├── Automated compliance checks (standards)                     │
│ ├── Cross-account aggregation                                    │
│ ├── Automated workflows (EventBridge integration)               │
│ ├── Custom insights (filtered finding views)                    │
│ └── Security scores per standard                                │
│                                                                       │
│ Compliance standards built-in:                                      │
│ ├── AWS Foundational Security Best Practices (FSBP)            │
│ ├── CIS AWS Foundations Benchmark (v1.4, v3.0)                 │
│ ├── PCI DSS v3.2.1                                               │
│ ├── NIST SP 800-53 Rev. 5                                       │
│ └── SOC 2                                                         │
│                                                                       │
│ Pricing: $0.0010/check/account/region/month                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Enabling Security Hub (Full Portal Walkthrough)

```
Console → Security Hub → Go to Security Hub → Enable Security Hub

┌─────────────────────────────────────────────────────────────────┐
│           ENABLE SECURITY HUB                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Security standards:                                             │
│ ☑ AWS Foundational Security Best Practices v1.0.0             │
│   → ⚡ Always enable (AWS-specific best practices)            │
│   → 200+ automated security checks                           │
│ ☑ CIS AWS Foundations Benchmark v3.0.0                        │
│   → Industry-standard security benchmark                     │
│ ☐ PCI DSS v3.2.1                                              │
│   → Enable if processing payment card data                   │
│ ☐ NIST Special Publication 800-53 Rev. 5                      │
│   → Enable for US government / FedRAMP compliance            │
│                                                                   │
│ ☑ Enable default standards automatically for new members     │
│                                                                   │
│ Prerequisite: AWS Config must be enabled                      │
│ → Security Hub uses Config rules for compliance checks       │
│                                                                   │
│                    [Enable Security Hub]                        │
└─────────────────────────────────────────────────────────────────┘

After enabling:

┌─────────────────────────────────────────────────────────────────┐
│           SECURITY HUB DASHBOARD                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Summary:                                                        │
│ ├── Security score: 78% (FSBP), 82% (CIS)                   │
│ ├── Active findings: 156 (42 critical, 67 high)              │
│ ├── Resources with most findings                              │
│ └── Accounts with most findings                               │
│                                                                   │
│ Top critical findings:                                         │
│ ├── [CRITICAL] S3.2: S3 buckets should not have public read │
│ ├── [CRITICAL] IAM.4: IAM root user should not have access  │
│ │   keys                                                      │
│ ├── [CRITICAL] EC2.19: Security groups should not allow      │
│ │   unrestricted access to high risk ports                   │
│ └── [HIGH] RDS.3: RDS instances should have encryption       │
│     enabled                                                    │
│                                                                   │
│ Actions per finding:                                           │
│ ├── Investigate (view details)                                │
│ ├── Suppress (false positive / accepted risk)                │
│ ├── Set workflow status: NEW → NOTIFIED → RESOLVED          │
│ └── Send to EventBridge (auto-remediation)                   │
│                                                                   │
│ Multi-account:                                                  │
│ Console → Security Hub → Settings → Accounts                  │
│ ├── Delegated administrator                                   │
│ ├── Auto-enable for new organization members                 │
│ └── Cross-region aggregation (findings from all regions)    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Other Security Services

```
┌─────────────────────────────────────────────────────────────────────┐
│           RELATED SECURITY SERVICES                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Amazon Inspector:                                                    │
│ ├── Automated vulnerability scanning                             │
│ ├── Scans: EC2 instances, ECR container images, Lambda         │
│ ├── CVE database (Common Vulnerabilities & Exposures)          │
│ ├── Network reachability analysis                                │
│ ├── Software bill of materials (SBOM)                           │
│ └── Findings → Security Hub                                     │
│                                                                       │
│ Amazon Macie:                                                        │
│ ├── Discovers sensitive data in S3 (PII, PHI, financial)       │
│ ├── ML-based classification                                      │
│ ├── Alerts on public/unencrypted sensitive data                │
│ └── Findings → Security Hub                                     │
│                                                                       │
│ Amazon Detective:                                                    │
│ ├── Security investigation tool                                  │
│ ├── Visualizes relationships between GuardDuty findings        │
│ ├── Auto-correlates: CloudTrail, VPC Flow, GuardDuty          │
│ └── "Who, what, when, where" for security incidents           │
│                                                                       │
│ IAM Access Analyzer:                                                │
│ ├── Finds resources shared externally                           │
│ ├── S3 buckets, IAM roles, KMS keys, Lambda, SQS, etc.       │
│ ├── Policy validation (checks IAM policies for issues)        │
│ ├── Policy generation (generate policies from CloudTrail)    │
│ └── Findings → Security Hub                                     │
│                                                                       │
│ ⚡ Enable all of these: GuardDuty + Inspector + Security Hub     │
│   = minimum enterprise security baseline                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & CLI Examples

```hcl
# Enable GuardDuty
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs      { enable = true }
    kubernetes   { audit_logs { enable = true } }
    malware_protection { scan_ec2_instance_with_findings {
      ebs_volumes { enable = true }
    }}
  }

  finding_publishing_frequency = "FIFTEEN_MINUTES"
}

# Enable Security Hub
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "fsbp" {
  standards_arn = "arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0"
  depends_on    = [aws_securityhub_account.main]
}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/3.0.0"
  depends_on    = [aws_securityhub_account.main]
}

# Auto-remediate GuardDuty findings via EventBridge
resource "aws_cloudwatch_event_rule" "guardduty_high" {
  name = "guardduty-high-severity"
  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail      = { severity = [{ numeric = [">=", 7] }] }
  })
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule = aws_cloudwatch_event_rule.guardduty_high.name
  arn  = aws_lambda_function.incident_response.arn
}
```

```bash
# Enable GuardDuty
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES

# List findings
aws guardduty list-findings --detector-id detector-id

# Get finding details
aws guardduty get-findings --detector-id detector-id --finding-ids finding-id

# Enable Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards

# Get security score
aws securityhub get-findings \
  --filters '{"ComplianceStatus":[{"Value":"FAILED","Comparison":"EQUALS"}]}' \
  --max-items 10
```

---

## Quick Reference

```
GuardDuty & Security Hub Quick Reference:
├── GuardDuty: ML-based threat detection (no agent for most)
│   ├── Sources: CloudTrail, VPC Flow, DNS, EKS, RDS, Lambda
│   ├── Detects: Recon, compromise, crypto mining, malware
│   ├── Severity: High (7+), Medium (4-6), Low (1-3)
│   └── Response: EventBridge → Lambda auto-remediation
├── Security Hub: Central security findings dashboard
│   ├── Aggregates: GuardDuty, Inspector, Macie, Config
│   ├── Standards: FSBP, CIS, PCI, NIST, SOC 2
│   ├── Security scores per standard
│   └── Cross-account + cross-region aggregation
├── Inspector: Vulnerability scanning (EC2, ECR, Lambda)
├── Macie: Sensitive data discovery in S3
├── Detective: Security investigation visualization
├── ⚡ Minimum: GuardDuty + Security Hub + Inspector
└── ⚡ Enable for ALL accounts in Organization
```

---

## What's Next?

In **Chapter 44: ACM (Certificate Manager)**, we'll cover SSL/TLS certificate provisioning, renewal, and management for your AWS resources.
