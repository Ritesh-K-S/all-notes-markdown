# Chapter 42: WAF & Shield

---

## Table of Contents

- [Overview](#overview)
- [Part 1: AWS WAF Fundamentals](#part-1-aws-waf-fundamentals)
- [Part 2: Creating a Web ACL (Full Portal Walkthrough)](#part-2-creating-a-web-acl-full-portal-walkthrough)
- [Part 3: WAF Rules & Rule Groups](#part-3-waf-rules--rule-groups)
- [Part 4: AWS Shield (Standard & Advanced)](#part-4-aws-shield-standard--advanced)
- [Part 5: AWS Firewall Manager](#part-5-aws-firewall-manager)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What Are Web Attacks? Why Do We Need WAF and Shield?

When your website is on the internet, anyone can send requests to it — including attackers. Common attacks include:

- **SQL Injection**: Attacker sends `'; DROP TABLE users;--` in a login form to delete your database
- **XSS (Cross-Site Scripting)**: Attacker injects malicious JavaScript that steals user cookies
- **DDoS (Distributed Denial of Service)**: Millions of fake requests flood your server until it crashes

**AWS WAF is like a bouncer at a nightclub** — it checks every request against your rules and blocks the bad ones before they reach your application. **AWS Shield is like an anti-flood barrier** — it protects against massive traffic floods (DDoS attacks).

**Simple setup:** You put WAF in front of your ALB or CloudFront, add managed rule sets (AWS provides ready-made rules for common attacks), and you're protected.

AWS WAF protects web applications from common exploits (SQL injection, XSS, etc.). AWS Shield provides DDoS protection. Together they form the front-line defense for your web applications.

```
What you'll learn:
├── WAF Fundamentals (Web ACLs, rules, conditions)
├── Creating a Web ACL (full portal walkthrough)
├── Rules: Managed, custom, rate-based, bot control
├── AWS Shield Standard vs Advanced
├── AWS Firewall Manager (multi-account WAF)
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: AWS WAF Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW AWS WAF WORKS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Internet → WAF (Web ACL) → Protected Resource                     │
│              │                  │                                    │
│              │ Inspect request  ├── CloudFront                     │
│              │ Match rules      ├── ALB                            │
│              │ Allow/Block/Count├── API Gateway                    │
│              │                  ├── AppSync                        │
│              └──────────────────├── Cognito                        │
│                                 └── App Runner                     │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Web ACL: Container for rules (associated with resource)     │
│ ├── Rule: Condition + action (allow, block, count, CAPTCHA)     │
│ ├── Rule Group: Reusable collection of rules                    │
│ ├── Managed Rule Group: Pre-built by AWS or marketplace         │
│ ├── Statement: Match condition (IP, string, regex, etc.)        │
│ └── WCU: Web ACL Capacity Units (rule complexity budget)        │
│                                                                       │
│ Rule evaluation order:                                              │
│ ├── Rules evaluated in priority order (lowest number first)     │
│ ├── First matching rule's action is taken                        │
│ ├── Default action if no rule matches (allow or block)          │
│ └── Count: Logs match but continues evaluating                  │
│                                                                       │
│ Pricing:                                                             │
│ ├── Web ACL: $5/month                                            │
│ ├── Rule: $1/month per rule                                      │
│ ├── Requests: $0.60/million requests                            │
│ ├── Bot Control: $10/month + $1/million requests                │
│ └── Managed rules: Varies ($1-$20/month per group)              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Web ACL (Full Portal Walkthrough)

```
Console → WAF & Shield → Web ACLs → Create web ACL

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: WEB ACL DETAILS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [prod-web-acl]                                           │
│ Description: [Production web application firewall]             │
│                                                                   │
│ Resource type:                                                  │
│ ● Regional resources (ALB, API Gateway, AppSync, etc.)       │
│ ○ Amazon CloudFront distributions                             │
│ → ⚡ CloudFront WAF must be in us-east-1                      │
│                                                                   │
│ Region: [US East (N. Virginia) ▼]                             │
│                                                                   │
│ Associated AWS resources: [Add AWS resources]                  │
│ ☑ ALB: prod-web-alb                                           │
│ ☑ API Gateway: prod-api (stage: prod)                         │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: ADD RULES AND RULE GROUPS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ [Add rules ▼]                                                  │
│ ├── Add managed rule groups                                   │
│ ├── Add my own rules and rule groups                          │
│ └── Add IP set                                                 │
│                                                                   │
│ ── AWS Managed Rule Groups (⚡ start here) ──                  │
│                                                                   │
│ Free rule groups:                                              │
│ ☑ AWS-AWSManagedRulesCommonRuleSet (⚡ essential)             │
│   → Protects against OWASP Top 10                            │
│   → SQL injection, XSS, bad bots, etc.                      │
│   WCU: 700                                                    │
│                                                                   │
│ ☑ AWS-AWSManagedRulesKnownBadInputsRuleSet                   │
│   → Log4j, Java deserialization, etc.                        │
│   WCU: 200                                                    │
│                                                                   │
│ ☑ AWS-AWSManagedRulesSQLiRuleSet                              │
│   → SQL injection patterns                                    │
│   WCU: 200                                                    │
│                                                                   │
│ ☐ AWS-AWSManagedRulesLinuxRuleSet                             │
│   → Linux-specific vulnerabilities                           │
│                                                                   │
│ ☐ AWS-AWSManagedRulesPHPRuleSet                               │
│   → PHP-specific vulnerabilities                             │
│                                                                   │
│ ☐ AWS-AWSManagedRulesWordPressRuleSet                         │
│   → WordPress-specific protections                           │
│                                                                   │
│ Paid rule groups:                                              │
│ ☐ AWS-AWSManagedRulesBotControlRuleSet ($10/month)           │
│   → Bot detection and mitigation                             │
│   → Verify bots, manage scrapers, block bad bots            │
│                                                                   │
│ ☐ AWS-AWSManagedRulesATPRuleSet ($10/month)                  │
│   → Account Takeover Prevention                              │
│   → Login page protection, credential stuffing               │
│                                                                   │
│ ☐ AWS-AWSManagedRulesACFPRuleSet ($10/month)                 │
│   → Account Creation Fraud Prevention                        │
│   → Sign-up page protection                                  │
│                                                                   │
│ ── Custom Rules ──                                              │
│                                                                   │
│ Rule type:                                                     │
│ ○ Regular rule (match condition → action)                     │
│ ● Rate-based rule (rate limit)                                │
│                                                                   │
│ Rate-based rule:                                               │
│ Name: [rate-limit-per-ip]                                     │
│ Rate limit: [2000] requests per 5-minute window              │
│ → Requests per IP. Block if exceeded.                       │
│ → ⚡ Essential DDoS / brute-force protection                 │
│                                                                   │
│ Regular rule conditions:                                       │
│ If a request:                                                  │
│ ├── Originates from IP [IP set ▼]                            │
│ ├── Has header [User-Agent] matching [regex ▼]               │
│ ├── Has URI path matching [/admin*]                          │
│ ├── Query string contains [<script>]                         │
│ ├── Body contains [SQL injection patterns]                   │
│ ├── Comes from country [geo match ▼]                         │
│ ├── Has label [awswaf:managed:aws:core-rule-set:...]        │
│ └── Request size exceeds [8192] bytes                        │
│                                                                   │
│ Action: ● Block ○ Allow ○ Count ○ CAPTCHA ○ Challenge       │
│                                                                   │
│ Default web ACL action (no rule match):                       │
│ ● Allow  ○ Block                                              │
│ → ⚡ Default Allow: Only block known-bad traffic             │
│ → Default Block: Only allow known-good (strict, may break)  │
│                                                                   │
│ Web ACL capacity: 1392 / 1500 WCU used                       │
│ → Max 1500 WCU per Web ACL                                   │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: SET RULE PRIORITY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Priority (top = evaluated first):                              │
│ 0: rate-limit-per-ip (rate-based)                             │
│ 1: AWS-AWSManagedRulesCommonRuleSet                           │
│ 2: AWS-AWSManagedRulesSQLiRuleSet                             │
│ 3: AWS-AWSManagedRulesKnownBadInputsRuleSet                  │
│                                                                   │
│ → ⚡ Rate limiting first (cheapest to evaluate)               │
│ → Then managed rules                                          │
│ → Then custom rules                                           │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 4: CONFIGURE METRICS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ CloudWatch metrics: ☑ Enabled for each rule                   │
│ Sampled requests: ☑ Enable                                    │
│ → View sample of matched requests in console                 │
│                                                                   │
│ Logging:                                                       │
│ ● CloudWatch Logs (⚡ recommended)                            │
│ ○ S3 bucket                                                   │
│ ○ Kinesis Data Firehose                                      │
│ Log destination: [aws-waf-logs-prod ▼]                       │
│ → ⚡ Always enable logging for troubleshooting                │
│                                                                   │
│                    [Next → Review → Create]                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: WAF Rules & Rule Groups

```
┌─────────────────────────────────────────────────────────────────────┐
│           RULE TYPES                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. IP Set rules:                                                    │
│    → Allow/block specific IP ranges                               │
│    → Whitelist known office IPs                                   │
│    → Block known malicious IPs                                    │
│                                                                       │
│ 2. Rate-based rules:                                                │
│    → Limit requests per IP per 5-minute window                   │
│    → Auto-blocks IPs exceeding threshold                         │
│    → Unblocks when rate drops below threshold                    │
│    → ⚡ Essential for API protection                              │
│                                                                       │
│ 3. Geo-match rules:                                                 │
│    → Allow/block by country                                       │
│    → e.g., Block traffic from countries you don't serve          │
│                                                                       │
│ 4. String/Regex match:                                              │
│    → Inspect: URI, query string, body, headers, cookies          │
│    → Match: Contains, starts with, ends with, exact, regex      │
│    → Text transformation: lowercase, URL decode, HTML decode    │
│                                                                       │
│ 5. Size constraint:                                                  │
│    → Block requests exceeding size thresholds                    │
│    → Protect against oversized payloads                          │
│                                                                       │
│ 6. Label-based rules:                                               │
│    → Managed rules add labels to matching requests              │
│    → Create custom rules based on those labels                  │
│    → Example: Label from bot detection → CAPTCHA action         │
│                                                                       │
│ Actions:                                                             │
│ ├── Allow: Let request through                                   │
│ ├── Block: Return 403 (customizable response)                   │
│ ├── Count: Log but don't block (testing mode)                   │
│ ├── CAPTCHA: Show CAPTCHA challenge                              │
│ └── Challenge: Silent browser challenge (JS verification)       │
│                                                                       │
│ ⚡ Start with Count mode → review logs → switch to Block       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: AWS Shield (Standard & Advanced)

```
┌─────────────────────────────────────────────────────────────────────┐
│           AWS SHIELD                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Shield Standard (FREE, automatic):                                  │
│ ├── Enabled by default on ALL AWS accounts                       │
│ ├── Protects against Layer 3/4 DDoS (network/transport)        │
│ ├── SYN floods, UDP reflection attacks                          │
│ ├── Inline detection and mitigation                              │
│ └── No additional cost                                           │
│                                                                       │
│ Shield Advanced ($3,000/month + data transfer):                    │
│ ├── Enhanced DDoS protection (Layer 3/4 AND Layer 7)           │
│ ├── Protects: CloudFront, ALB, NLB, Elastic IP, Global Accel. │
│ ├── 24/7 access to Shield Response Team (SRT)                  │
│ ├── DDoS cost protection (credits for scaling costs)           │
│ ├── Advanced metrics and reporting                               │
│ ├── Automatic application layer mitigation (WAF rules)         │
│ ├── Health-based detection (Route 53 health checks)            │
│ └── 1-year commitment                                            │
│                                                                       │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ Standard      │ Advanced             │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Cost                 │ Free          │ $3,000/month         │   │
│ │ Layer 3/4            │ ✅            │ ✅ (enhanced)        │   │
│ │ Layer 7              │ ❌            │ ✅ (via WAF)         │   │
│ │ DDoS Response Team   │ ❌            │ ✅ 24/7              │   │
│ │ Cost protection      │ ❌            │ ✅ (scaling credits) │   │
│ │ WAF included         │ ❌            │ ✅ (free WAF usage)  │   │
│ │ Visibility           │ Basic         │ Advanced dashboards  │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│                                                                       │
│ ⚡ Shield Advanced includes free WAF usage for protected         │
│   resources — significant cost savings if you use WAF           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: AWS Firewall Manager

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIREWALL MANAGER                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Centrally manage WAF, Shield, Security Groups, and          │
│ Network Firewall across multiple accounts in AWS Organizations.   │
│                                                                       │
│ Console → WAF & Shield → Firewall Manager → Security policies    │
│                                                                       │
│ Policy types:                                                        │
│ ├── AWS WAF: Deploy Web ACLs across accounts                    │
│ ├── Shield Advanced: Enable Shield on resources                 │
│ ├── Security Group: Manage SG rules across VPCs                │
│ ├── Network Firewall: Deploy network firewall policies         │
│ ├── DNS Firewall: Route 53 Resolver DNS firewall               │
│ └── Third-party firewall (e.g., Palo Alto)                     │
│                                                                       │
│ Benefits:                                                            │
│ ├── Single pane of glass for all accounts                       │
│ ├── Auto-apply to new resources/accounts                        │
│ ├── Compliance: Ensure WAF on every ALB                        │
│ ├── Non-compliant resource detection                            │
│ └── Auto-remediation                                             │
│                                                                       │
│ ⚡ Required for enterprise multi-account WAF management          │
│ Pricing: $100/month per policy per region                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & CLI Examples

```hcl
resource "aws_wafv2_web_acl" "main" {
  name        = "prod-web-acl"
  scope       = "REGIONAL"  # or CLOUDFRONT
  description = "Production WAF"

  default_action { allow {} }

  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSetMetric"
    }
  }

  rule {
    name     = "RateLimit"
    priority = 0
    action { block {} }
    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitMetric"
    }
  }

  visibility_config {
    sampled_requests_enabled   = true
    cloudwatch_metrics_enabled = true
    metric_name                = "ProdWebACL"
  }
}

resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.web.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

```bash
# List Web ACLs
aws wafv2 list-web-acls --scope REGIONAL --region us-east-1

# Get Web ACL details
aws wafv2 get-web-acl --name prod-web-acl --scope REGIONAL --id web-acl-id

# Get sampled requests
aws wafv2 get-sampled-requests \
  --web-acl-arn arn:aws:wafv2:us-east-1:123456:regional/webacl/prod-web-acl/id \
  --rule-metric-name CommonRuleSetMetric \
  --scope REGIONAL \
  --time-window StartTime=2024-01-01T00:00:00Z,EndTime=2024-01-02T00:00:00Z \
  --max-items 100
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Standard Web App Protection                              │
│ ├── AWS Common Rule Set (OWASP Top 10)                           │
│ ├── SQL injection rule set                                        │
│ ├── Known bad inputs rule set                                     │
│ ├── Rate limiting: 2000 req/5min per IP                         │
│ ├── Geo-block: Countries you don't serve                        │
│ └── Logging to CloudWatch Logs                                   │
│                                                                       │
│ Pattern 2: API Protection                                           │
│ ├── Rate limiting per IP (strict: 500 req/5min)                 │
│ ├── API key validation (header check)                            │
│ ├── Request body size limit                                       │
│ ├── SQL injection + XSS protection                               │
│ └── Count mode first → review → block                           │
│                                                                       │
│ Pattern 3: Login Page Protection                                    │
│ ├── Account Takeover Prevention (ATP) rule set                  │
│ ├── Rate limit login endpoint (/login, /auth)                   │
│ ├── CAPTCHA on suspicious login attempts                         │
│ └── IP reputation lists                                           │
│                                                                       │
│ ⚡ Deployment strategy:                                              │
│ 1. Deploy all rules in COUNT mode                                 │
│ 2. Monitor for 1-2 weeks                                          │
│ 3. Review false positives                                         │
│ 4. Exclude legitimate patterns                                    │
│ 5. Switch to BLOCK mode                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
WAF & Shield Quick Reference:
├── WAF: Layer 7 firewall (HTTP/HTTPS inspection)
├── Protects: CloudFront, ALB, API Gateway, AppSync
├── Web ACL: Container for rules (max 1500 WCU)
├── Managed rules: ⚡ Start with Common Rule Set
├── Rate-based: Limit requests per IP (DDoS/brute-force)
├── Actions: Allow, Block, Count, CAPTCHA, Challenge
├── Shield Standard: Free, auto Layer 3/4 DDoS protection
├── Shield Advanced: $3K/month, Layer 7, SRT, cost protection
├── Firewall Manager: Multi-account WAF management
├── Logging: CloudWatch Logs, S3, or Kinesis Firehose
├── ⚡ Always start in COUNT mode before blocking
└── ⚡ Common Rule Set + Rate Limit = minimum protection
```

---

## What's Next?

In **Chapter 43: GuardDuty & Security Hub**, we'll cover threat detection, security findings aggregation, and centralized security management across your AWS accounts.
