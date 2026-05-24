# FinOps — Managing Cloud Costs at Scale

> **What you'll learn**: How organizations with massive cloud bills ($1M+/year) implement financial governance, build cost-aware engineering cultures, and use sophisticated practices to optimize spending without sacrificing reliability or performance.

---

## Real-Life Analogy

Imagine a large company where every employee has a corporate credit card with no spending limit:

- **Without FinOps**: Everyone buys whatever they want. Some people order first-class flights for a 1-hour trip. Others leave the office lights on 24/7. Nobody knows who's spending what until the monthly bill arrives — and it's 3x the budget. Fingers are pointed, budgets are slashed blindly, and projects get killed.

- **With FinOps**: Everyone can still spend, but there's visibility. Each department sees their spending in real-time. A "travel advisor" helps people find cheaper flights. Teams that optimize get recognized. There's a shared understanding: *spend wisely, not less* — every dollar should create value.

**FinOps is the practice of bringing financial accountability to cloud spending** — giving engineering teams the information and incentives to make cost-efficient decisions without slowing them down.

---

## Core Concept Explained Step-by-Step

### What is FinOps?

**FinOps = Finance + DevOps**. It's a cultural practice and operational model for managing cloud costs where variable spending (pay-as-you-go cloud) is managed collaboratively between engineering, finance, and business teams.

```
┌──────────────────────────────────────────────────────────────────┐
│                     FINOPS DEFINITION                              │
│                                                                  │
│  "FinOps is an evolving cloud financial management discipline    │
│   and cultural practice that enables organizations to get        │
│   maximum business value by helping engineering, finance,        │
│   and business teams collaborate on data-driven spending         │
│   decisions."                                                    │
│                                        — FinOps Foundation       │
│                                                                  │
│  Key Principles:                                                 │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 1. Teams need to collaborate                          │       │
│  │ 2. Everyone takes ownership of their cloud usage      │       │
│  │ 3. A centralized team drives FinOps                   │       │
│  │ 4. Reports should be accessible and timely            │       │
│  │ 5. Decisions are driven by business value of cloud    │       │
│  │ 6. Take advantage of the variable cost model          │       │
│  └──────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────┘
```

### The Three Phases of FinOps

```
┌─────────────────────────────────────────────────────────────────┐
│              THE FINOPS LIFECYCLE                                 │
│                                                                 │
│          ┌──────────┐                                           │
│          │  INFORM  │ ← Where is money going?                   │
│          └────┬─────┘                                           │
│               │                                                 │
│               ▼                                                 │
│          ┌──────────┐                                           │
│          │ OPTIMIZE │ ← How can we spend less?                  │
│          └────┬─────┘                                           │
│               │                                                 │
│               ▼                                                 │
│          ┌──────────┐                                           │
│          │ OPERATE  │ ← How do we sustain this?                 │
│          └────┬─────┘                                           │
│               │                                                 │
│               └───────────── REPEAT ────────────────────────    │
│                                                                 │
│  Phase 1 — INFORM:                                              │
│  • Tag all resources with owner/team/project/environment        │
│  • Build dashboards showing cost per team/service               │
│  • Allocate shared costs (networking, support, platform)        │
│  • Report anomalies and trends                                  │
│                                                                 │
│  Phase 2 — OPTIMIZE:                                            │
│  • Right-size over-provisioned instances                        │
│  • Purchase Reserved Instances / Savings Plans                  │
│  • Use Spot instances for fault-tolerant workloads              │
│  • Delete unused resources (zombie instances, old snapshots)    │
│  • Architect for cost (serverless, arm64, tiered storage)       │
│                                                                 │
│  Phase 3 — OPERATE:                                             │
│  • Set budgets and alerts per team                              │
│  • Automate cost policies (auto-shutdown dev at night)          │
│  • Continuous improvement culture                               │
│  • Cost as a non-functional requirement in design reviews       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The FinOps Team Structure

```
┌──────────────────────────────────────────────────────────────┐
│             FINOPS ORGANIZATIONAL MODEL                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │            FINOPS TEAM (Central)                       │   │
│  │  • FinOps Practitioner / Cloud Economist              │   │
│  │  • Sets policies, builds tools, reports to leadership │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                              │                               │
│         ┌────────────────────┼────────────────────┐         │
│         │                    │                    │          │
│         ▼                    ▼                    ▼          │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   │
│  │  Engineering │   │   Finance    │   │  Business /  │   │
│  │  Teams       │   │              │   │  Product     │   │
│  ├──────────────┤   ├──────────────┤   ├──────────────┤   │
│  │ • Own their  │   │ • Budgets    │   │ • Define     │   │
│  │   spending   │   │ • Forecasts  │   │   business   │   │
│  │ • Right-size │   │ • Contracts  │   │   value      │   │
│  │ • Optimize   │   │ • Chargebacks│   │ • Approve    │   │
│  │   code       │   │ • Reporting  │   │   trade-offs │   │
│  └──────────────┘   └──────────────┘   └──────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Cost Allocation & Tagging

The foundation of FinOps is knowing **who is spending what**:

```
┌──────────────────────────────────────────────────────────────┐
│             TAGGING STRATEGY                                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Required tags on EVERY resource:                            │
│                                                              │
│  ┌─────────────────┬────────────────────────────────────┐   │
│  │ Tag Key         │ Example Values                      │   │
│  ├─────────────────┼────────────────────────────────────┤   │
│  │ team            │ payments, search, platform          │   │
│  │ service         │ payment-api, search-indexer         │   │
│  │ environment     │ production, staging, development    │   │
│  │ cost-center     │ CC-1234, CC-5678                    │   │
│  │ project         │ checkout-v2, recommendation-engine  │   │
│  │ owner           │ alice@company.com                   │   │
│  │ managed-by      │ terraform, manual, cdk             │   │
│  └─────────────────┴────────────────────────────────────┘   │
│                                                              │
│  Without tags → "Unallocated costs" → nobody takes ownership │
│  Target: < 5% untagged/unallocated spending                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### The Unit Economics Model

The most mature FinOps teams measure **cost per unit of business value**, not just total spend:

```
┌──────────────────────────────────────────────────────────────┐
│           UNIT ECONOMICS EXAMPLES                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  E-Commerce:                                                 │
│  ├── Cost per order processed      = $0.12                   │
│  ├── Cost per search query         = $0.001                  │
│  ├── Infra cost as % of revenue    = 4.2%                    │
│  └── Cost per 1000 page views      = $0.45                   │
│                                                              │
│  SaaS:                                                       │
│  ├── Cost per active user/month    = $1.20                   │
│  ├── Cost per API call             = $0.00003                │
│  ├── Infra cost per $1 revenue     = $0.18                   │
│  └── Cost per GB stored            = $0.023                  │
│                                                              │
│  Streaming:                                                  │
│  ├── Cost per hour of video served = $0.003                  │
│  ├── Cost per subscriber/month     = $0.85                   │
│  └── Encoding cost per minute      = $0.012                  │
│                                                              │
│  WHY THIS MATTERS:                                           │
│  Total cloud bill went up 40%? PANIC!                        │
│  But revenue grew 60% and cost-per-order dropped 15%?        │
│  Actually that's GREAT! — efficiency improved.               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Showback vs Chargeback

Two models for cost accountability:

```
┌──────────────────────────────────────────────────────────────┐
│         SHOWBACK vs CHARGEBACK                                │
├───────────────────────┬──────────────────────────────────────┤
│      SHOWBACK         │       CHARGEBACK                     │
├───────────────────────┼──────────────────────────────────────┤
│ "Here's what you      │ "This is deducted from              │
│  spent. FYI."         │  your team's budget."               │
│                       │                                      │
│ Informational only    │ Actual financial impact              │
│ No budget impact      │ Teams feel the pain of waste        │
│ Lower friction        │ Stronger incentive to optimize      │
│ Good starting point   │ Requires mature tagging             │
│                       │                                      │
│ Risk: Teams may       │ Risk: Teams may cut corners         │
│ ignore the reports    │ on reliability to save money        │
└───────────────────────┴──────────────────────────────────────┘

Most companies start with SHOWBACK and evolve to CHARGEBACK
as their tagging and allocation mature.
```

### Anomaly Detection

At scale, humans can't spot every cost issue. Automated detection is essential:

```
┌──────────────────────────────────────────────────────────────┐
│         COST ANOMALY DETECTION                                │
│                                                              │
│  Normal daily spend: $5,000 ± $500                           │
│                                                              │
│  $                                                           │
│  │                                                           │
│  │                              ╱╲ ← ANOMALY! $12K day      │
│  │                             ╱  ╲                          │
│  │    ╱╲   ╱╲   ╱╲   ╱╲     ╱    ╲   ╱╲                    │
│  │   ╱  ╲ ╱  ╲ ╱  ╲ ╱  ╲  ╱      ╲ ╱  ╲                   │
│  │  ╱    ╳    ╳    ╳    ╳ ╱        ╳    ╲                   │
│  │ ╱                      ╱                                  │
│  └────────────────────────────────────────── Days            │
│                                                              │
│  Common anomaly causes:                                      │
│  • Forgotten load test running in production account         │
│  • Auto-scaling triggered by a bug (infinite loop)          │
│  • Accidental deployment of expensive GPU instances          │
│  • Data transfer spike from a misconfigured backup           │
│  • S3 request spike from bot/scraper traffic                 │
│                                                              │
│  Tools: AWS Cost Anomaly Detection, CloudHealth,             │
│         Kubecost, custom Prometheus alerts                    │
└──────────────────────────────────────────────────────────────┘
```

### The FinOps Maturity Model

```
┌──────────────────────────────────────────────────────────────┐
│          FINOPS MATURITY LEVELS                                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CRAWL (Level 1):                                            │
│  ├── Basic cost visibility (billing console)                 │
│  ├── Some resources tagged                                   │
│  ├── Monthly bill review by 1 person                         │
│  ├── Reactive: "Why is the bill so high?"                    │
│  └── Savings: 0-10%                                          │
│                                                              │
│  WALK (Level 2):                                             │
│  ├── Full tagging policy enforced                            │
│  ├── Per-team cost dashboards                                │
│  ├── Reserved Instances purchased                            │
│  ├── Right-sizing reviews quarterly                          │
│  ├── Budget alerts configured                                │
│  └── Savings: 20-35%                                         │
│                                                              │
│  RUN (Level 3):                                              │
│  ├── Unit economics tracked (cost per transaction)           │
│  ├── Automated anomaly detection                             │
│  ├── Chargeback to teams (budget ownership)                  │
│  ├── Cost is part of architecture reviews                    │
│  ├── Spot/preemptible used extensively                       │
│  ├── Automated right-sizing and scheduling                   │
│  └── Savings: 35-50%                                         │
│                                                              │
│  FLY (Level 4 — Planet Scale):                               │
│  ├── Real-time cost per request visibility                   │
│  ├── ML-driven optimization recommendations                 │
│  ├── Custom pricing negotiations with cloud providers        │
│  ├── Multi-cloud arbitrage (cheapest provider per workload)  │
│  ├── Cost efficiency as engineering KPI                      │
│  ├── Automated policy enforcement (block non-compliant)      │
│  └── Savings: 50-70%                                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — FinOps Cost Allocation Engine

```python
# finops_allocator.py — Allocate cloud costs to teams by tags

from dataclasses import dataclass, field
from typing import Dict, List
from collections import defaultdict

@dataclass
class CloudResource:
    resource_id: str
    service: str           # EC2, RDS, S3, etc.
    monthly_cost: float
    tags: Dict[str, str]   # team, service, environment, etc.

@dataclass
class CostReport:
    total_cost: float = 0.0
    allocated_cost: float = 0.0
    unallocated_cost: float = 0.0
    by_team: Dict[str, float] = field(default_factory=lambda: defaultdict(float))
    by_environment: Dict[str, float] = field(default_factory=lambda: defaultdict(float))
    by_service: Dict[str, float] = field(default_factory=lambda: defaultdict(float))

class FinOpsAllocator:
    """Allocates cloud costs to teams and generates reports."""
    
    def __init__(self, resources: List[CloudResource]):
        self.resources = resources
    
    def generate_report(self) -> CostReport:
        """Allocate costs and generate a full breakdown."""
        report = CostReport()
        
        for resource in self.resources:
            report.total_cost += resource.monthly_cost
            report.by_service[resource.service] += resource.monthly_cost
            
            # Allocate by team tag
            team = resource.tags.get("team")
            if team:
                report.by_team[team] += resource.monthly_cost
                report.allocated_cost += resource.monthly_cost
            else:
                report.unallocated_cost += resource.monthly_cost
            
            # Track by environment
            env = resource.tags.get("environment", "unknown")
            report.by_environment[env] += resource.monthly_cost
        
        return report
    
    def find_waste(self) -> List[Dict]:
        """Identify potential waste (dev resources running 24/7, etc.)."""
        waste = []
        
        for r in self.resources:
            env = r.tags.get("environment", "")
            
            # Dev/staging running expensive instances
            if env in ("development", "staging") and r.monthly_cost > 500:
                waste.append({
                    "resource": r.resource_id,
                    "reason": f"{env} resource costs ${r.monthly_cost:.0f}/mo",
                    "suggestion": "Schedule shutdown outside work hours (save ~65%)",
                    "potential_savings": r.monthly_cost * 0.65
                })
            
            # Untagged resources (likely forgotten)
            if not r.tags.get("team"):
                waste.append({
                    "resource": r.resource_id,
                    "reason": "No team tag — likely orphaned",
                    "suggestion": "Identify owner or terminate",
                    "potential_savings": r.monthly_cost
                })
        
        return waste
    
    def print_report(self):
        """Print a formatted cost allocation report."""
        report = self.generate_report()
        alloc_pct = (report.allocated_cost / report.total_cost * 100 
                     if report.total_cost > 0 else 0)
        
        print("\n" + "=" * 60)
        print("         MONTHLY FINOPS COST REPORT")
        print("=" * 60)
        print(f"  Total Cloud Spend:    ${report.total_cost:,.2f}")
        print(f"  Allocated:            ${report.allocated_cost:,.2f} ({alloc_pct:.0f}%)")
        print(f"  Unallocated:          ${report.unallocated_cost:,.2f}")
        
        print("\n  --- COST BY TEAM ---")
        for team, cost in sorted(report.by_team.items(), 
                                  key=lambda x: -x[1]):
            pct = cost / report.total_cost * 100
            bar = "█" * int(pct / 2)
            print(f"  {team:<20} ${cost:>10,.2f}  {pct:4.1f}% {bar}")
        
        print("\n  --- COST BY ENVIRONMENT ---")
        for env, cost in sorted(report.by_environment.items(), 
                                 key=lambda x: -x[1]):
            print(f"  {env:<20} ${cost:>10,.2f}")
        
        # Waste detection
        waste = self.find_waste()
        if waste:
            total_savings = sum(w["potential_savings"] for w in waste)
            print(f"\n  --- WASTE DETECTED ({len(waste)} items) ---")
            print(f"  Potential monthly savings: ${total_savings:,.2f}")
            for w in waste[:5]:  # Show top 5
                print(f"  • {w['resource']}: {w['reason']}")
        
        print("=" * 60)


# Example usage with sample data
resources = [
    CloudResource("i-abc123", "EC2", 2400, {"team": "payments", "environment": "production"}),
    CloudResource("i-def456", "EC2", 1800, {"team": "search", "environment": "production"}),
    CloudResource("rds-prod1", "RDS", 3200, {"team": "payments", "environment": "production"}),
    CloudResource("i-dev789", "EC2", 800, {"team": "search", "environment": "development"}),
    CloudResource("i-orphan1", "EC2", 600, {"environment": "production"}),  # No team tag!
    CloudResource("s3-logs", "S3", 450, {"team": "platform", "environment": "production"}),
    CloudResource("i-staging", "EC2", 1200, {"team": "payments", "environment": "staging"}),
]

allocator = FinOpsAllocator(resources)
allocator.print_report()
```

### Java — Budget Alert System

```java
// BudgetAlertSystem.java — Monitor team budgets and send alerts

import java.util.*;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

public class BudgetAlertSystem {
    
    enum AlertSeverity { INFO, WARNING, CRITICAL }
    
    record TeamBudget(
        String teamName,
        double monthlyBudget,      // Total monthly budget ($)
        double currentSpend,       // Spend so far this month ($)
        double forecastedSpend     // Projected month-end spend ($)
    ) {
        // What percentage of the month has passed?
        double monthProgress() {
            int dayOfMonth = LocalDate.now().getDayOfMonth();
            int daysInMonth = LocalDate.now().lengthOfMonth();
            return (double) dayOfMonth / daysInMonth;
        }
        
        // Are we spending faster than budget allows?
        double burnRate() {
            double expectedSpend = monthlyBudget * monthProgress();
            return expectedSpend > 0 ? currentSpend / expectedSpend : 0;
        }
        
        double forecastOverage() {
            return forecastedSpend - monthlyBudget;
        }
    }
    
    record Alert(
        String teamName,
        AlertSeverity severity,
        String message,
        double currentSpend,
        double budget
    ) {}
    
    private final List<TeamBudget> budgets = new ArrayList<>();
    
    public void addTeamBudget(TeamBudget budget) {
        budgets.add(budget);
    }
    
    public List<Alert> checkBudgets() {
        List<Alert> alerts = new ArrayList<>();
        
        for (TeamBudget tb : budgets) {
            double burnRate = tb.burnRate();
            double utilizationPct = (tb.currentSpend / tb.monthlyBudget) * 100;
            
            // Critical: Already over budget
            if (tb.currentSpend > tb.monthlyBudget) {
                alerts.add(new Alert(tb.teamName, AlertSeverity.CRITICAL,
                    String.format("OVER BUDGET! Spent $%.0f of $%.0f budget (%.0f%%)",
                        tb.currentSpend, tb.monthlyBudget, utilizationPct),
                    tb.currentSpend, tb.monthlyBudget));
            }
            // Critical: Forecasted to exceed by >20%
            else if (tb.forecastedSpend > tb.monthlyBudget * 1.2) {
                alerts.add(new Alert(tb.teamName, AlertSeverity.CRITICAL,
                    String.format("Forecasted to exceed budget by $%.0f (%.0f%% over)",
                        tb.forecastOverage(), (tb.forecastOverage() / tb.monthlyBudget) * 100),
                    tb.currentSpend, tb.monthlyBudget));
            }
            // Warning: Burn rate > 1.3x (spending 30% faster than planned)
            else if (burnRate > 1.3) {
                alerts.add(new Alert(tb.teamName, AlertSeverity.WARNING,
                    String.format("High burn rate: %.1fx budget pace. " +
                        "Projected overage: $%.0f", burnRate, tb.forecastOverage()),
                    tb.currentSpend, tb.monthlyBudget));
            }
            // Info: > 80% budget used
            else if (utilizationPct > 80) {
                alerts.add(new Alert(tb.teamName, AlertSeverity.INFO,
                    String.format("%.0f%% of budget consumed with %.0f%% " +
                        "of month remaining",
                        utilizationPct, (1 - tb.monthProgress()) * 100),
                    tb.currentSpend, tb.monthlyBudget));
            }
        }
        
        return alerts;
    }
    
    public void printAlertReport() {
        List<Alert> alerts = checkBudgets();
        
        System.out.println("╔══════════════════════════════════════════════════╗");
        System.out.println("║          FINOPS BUDGET ALERT REPORT             ║");
        System.out.println("╠══════════════════════════════════════════════════╣");
        
        if (alerts.isEmpty()) {
            System.out.println("║  ✅ All teams within budget.                    ║");
        } else {
            for (Alert a : alerts) {
                String icon = switch (a.severity) {
                    case CRITICAL -> "🔴";
                    case WARNING  -> "🟡";
                    case INFO     -> "🔵";
                };
                System.out.printf("║ %s [%s] %s%n", icon, a.teamName, a.message);
            }
        }
        
        System.out.println("╚══════════════════════════════════════════════════╝");
    }
    
    public static void main(String[] args) {
        BudgetAlertSystem system = new BudgetAlertSystem();
        
        // Mid-month check (day 15 of 30)
        system.addTeamBudget(new TeamBudget("payments", 25000, 18000, 32000));
        system.addTeamBudget(new TeamBudget("search", 15000, 7200, 14800));
        system.addTeamBudget(new TeamBudget("platform", 40000, 22000, 43000));
        system.addTeamBudget(new TeamBudget("ml-team", 30000, 35000, 55000));
        
        system.printAlertReport();
    }
}
```

---

## Infrastructure Examples

### AWS Cost Management Setup with Terraform

```hcl
# finops-infrastructure.tf — FinOps governance infrastructure

# Enforce tagging policy — reject untagged resources
resource "aws_organizations_policy" "require_tags" {
  name        = "require-cost-tags"
  description = "Require team, environment, and service tags"
  type        = "TAG_POLICY"
  content     = jsonencode({
    tags = {
      team = {
        tag_key = { "@@assign" = "team" }
        enforced_for = { "@@assign" = [
          "ec2:instance", "rds:db", "s3:bucket",
          "elasticache:cluster", "lambda:function"
        ]}
      }
      environment = {
        tag_key = { "@@assign" = "environment" }
        tag_value = { "@@assign" = ["production", "staging", "development"] }
        enforced_for = { "@@assign" = [
          "ec2:instance", "rds:db"
        ]}
      }
    }
  })
}

# Budget alerts per team
resource "aws_budgets_budget" "team_budgets" {
  for_each = {
    payments = 25000
    search   = 15000
    platform = 40000
    ml       = 30000
  }
  
  name         = "${each.key}-monthly-budget"
  budget_type  = "COST"
  limit_amount = each.value
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  
  cost_filter {
    name   = "TagKeyValue"
    values = ["user:team$${each.key}"]
  }
  
  # Alert at 80% actual spend
  notification {
    comparison_operator = "GREATER_THAN"
    threshold           = 80
    threshold_type      = "PERCENTAGE"
    notification_type   = "ACTUAL"
    subscriber_email_addresses = ["${each.key}-leads@company.com"]
  }
  
  # Alert at 100% forecasted spend
  notification {
    comparison_operator = "GREATER_THAN"
    threshold           = 100
    threshold_type      = "PERCENTAGE"
    notification_type   = "FORECASTED"
    subscriber_email_addresses = [
      "${each.key}-leads@company.com",
      "finops@company.com"
    ]
  }
}

# Cost anomaly detection
resource "aws_ce_anomaly_monitor" "service_monitor" {
  name              = "service-level-anomaly-monitor"
  monitor_type      = "DIMENSIONAL"
  monitor_dimension = "SERVICE"
}

resource "aws_ce_anomaly_subscription" "alert" {
  name = "cost-anomaly-alerts"
  
  monitor_arn_list = [aws_ce_anomaly_monitor.service_monitor.arn]
  
  threshold_expression {
    dimension {
      key           = "ANOMALY_TOTAL_IMPACT_ABSOLUTE"
      values        = ["100"]  # Alert if anomaly > $100
      match_options = ["GREATER_THAN_OR_EQUAL"]
    }
  }
  
  subscriber {
    type    = "SNS"
    address = aws_sns_topic.finops_alerts.arn
  }
}
```

### Kubernetes Cost Monitoring with Kubecost

```yaml
# kubecost-values.yaml — Deploy Kubecost for K8s cost visibility
# Helm: helm install kubecost kubecost/cost-analyzer -f kubecost-values.yaml

kubecostModel:
  # Enable container-level cost allocation
  containerStatsEnabled: true
  
  # Cloud pricing integration
  cloudCost:
    enabled: true
    provider: aws
    
  # Allocation settings
  allocation:
    # How to split shared costs (istio, monitoring, kube-system)
    sharedNamespaces:
      - kube-system
      - istio-system
      - monitoring
    sharedOverhead: 2000  # $2000/month platform overhead to distribute

# Configure budget alerts per namespace (team)
budgets:
  - name: payments-team
    namespace: payments
    monthly: 8000      # $8000/month
    alertThreshold: 80 # Alert at 80%
    
  - name: search-team
    namespace: search
    monthly: 5000
    alertThreshold: 80
    
  - name: ml-pipelines
    namespace: ml
    monthly: 12000
    alertThreshold: 70  # Lower threshold — ML costs spike easily

# Right-sizing recommendations
savings:
  # Recommend container right-sizing
  containerRightSizing:
    enabled: true
    # Look at 7 days of data for recommendations
    window: "7d"
    # Recommend based on p95 usage (not max)
    targetUtilization: 0.80
    
  # Identify abandoned workloads
  abandonedWorkloads:
    enabled: true
    # Flag pods with < 5% CPU utilization for 7+ days
    cpuThreshold: 0.05
    memoryThreshold: 0.10
    window: "7d"
```

### Automated Dev Environment Shutdown

```python
# auto_shutdown.py — Shut down dev/staging instances outside work hours
# Runs as a Lambda function on a CloudWatch schedule

import boto3
from datetime import datetime
import pytz

ec2 = boto3.client('ec2')

# Work hours: 8 AM - 8 PM IST, Monday-Friday
TIMEZONE = pytz.timezone('Asia/Kolkata')
WORK_START = 8
WORK_END = 20
WORK_DAYS = range(0, 5)  # Monday = 0, Friday = 4

def lambda_handler(event, context):
    """
    Stop dev/staging instances outside work hours.
    Start them when work hours begin.
    
    Saves ~65% on dev/staging compute costs!
    """
    now = datetime.now(TIMEZONE)
    is_work_hours = (now.weekday() in WORK_DAYS and 
                     WORK_START <= now.hour < WORK_END)
    
    # Find all dev/staging instances with auto-shutdown tag
    filters = [
        {'Name': 'tag:auto-shutdown', 'Values': ['true']},
        {'Name': 'tag:environment', 'Values': ['development', 'staging']}
    ]
    
    response = ec2.describe_instances(Filters=filters)
    instances = []
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            instances.append({
                'id': instance['InstanceId'],
                'state': instance['State']['Name'],
                'name': next(
                    (t['Value'] for t in instance.get('Tags', []) 
                     if t['Key'] == 'Name'), 'unnamed'
                )
            })
    
    if is_work_hours:
        # Start stopped instances
        to_start = [i['id'] for i in instances if i['state'] == 'stopped']
        if to_start:
            ec2.start_instances(InstanceIds=to_start)
            print(f"Started {len(to_start)} instances (work hours)")
    else:
        # Stop running instances
        to_stop = [i['id'] for i in instances if i['state'] == 'running']
        if to_stop:
            ec2.stop_instances(InstanceIds=to_stop)
            print(f"Stopped {len(to_stop)} instances (outside work hours)")
    
    return {
        'statusCode': 200,
        'body': f"Processed {len(instances)} instances. "
                f"Work hours: {is_work_hours}"
    }
```

---

## Real-World Example

### How Spotify Runs FinOps at Scale

```
┌──────────────────────────────────────────────────────────────┐
│           SPOTIFY'S FINOPS PRACTICE                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Scale: ~$100M+ annual cloud spend on GCP                   │
│                                                              │
│  Structure:                                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Central "Cost Engineering" team (5-8 people)           │ │
│  │ • Builds internal cost dashboard ("Backstage" plugin)  │ │
│  │ • Sets policies and guidelines                         │ │
│  │ • Negotiates with GCP on enterprise pricing            │ │
│  │ • Publishes weekly "Cost Digest" to all engineers      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Key Practices:                                              │
│                                                              │
│  1. COST PER STREAM                                          │
│     Primary unit metric. As long as cost-per-stream          │
│     decreases, total spend growth is acceptable.             │
│                                                              │
│  2. "GOLDEN SIGNALS" FOR COST                                │
│     Every team dashboard shows:                              │
│     • Cost trend (7-day, 30-day)                            │
│     • Cost per unit metric                                   │
│     • Anomaly indicators                                     │
│     • Right-sizing recommendations                           │
│                                                              │
│  3. SQUAD-LEVEL OWNERSHIP                                    │
│     Each squad (team) owns their cloud bill.                 │
│     Quarterly "cost reviews" are part of OKRs.              │
│                                                              │
│  4. COMMITTED USE DISCOUNTS (CUDs)                           │
│     Central team manages GCP CUDs (equivalent to RIs).      │
│     Purchased based on aggregate baseline across all teams.  │
│                                                              │
│  5. PREEMPTIBLE VMs FOR BATCH                                │
│     All data pipelines, ML training, CI/CD run on           │
│     preemptible VMs. Saves ~80% on those workloads.         │
│                                                              │
│  Results:                                                    │
│  • 40% reduction in cost-per-stream over 3 years            │
│  • < 3% unallocated costs                                   │
│  • Engineering teams proactively optimize                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### How Airbnb Built Their FinOps Practice

```
┌──────────────────────────────────────────────────────────────┐
│           AIRBNB'S FINOPS JOURNEY                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Cloud bill growing 3x faster than revenue          │
│                                                              │
│  Phase 1 (Crawl): Visibility                                 │
│  ├── Implemented mandatory tagging for all resources         │
│  ├── Built internal cost dashboard                           │
│  └── Monthly cost review with engineering VPs                │
│                                                              │
│  Phase 2 (Walk): Optimization                                │
│  ├── Right-sizing campaign saved 25% on compute             │
│  ├── Reserved Instances for all steady-state workloads       │
│  ├── Spot instances for search indexing jobs                  │
│  └── Storage tiering: moved cold data to cheaper storage     │
│                                                              │
│  Phase 3 (Run): Culture                                      │
│  ├── "Cost-per-booking" as an engineering KPI                │
│  ├── Cost estimate required in design documents              │
│  ├── Automated alerts per team when spend spikes             │
│  └── Quarterly "Frugality Awards" for best optimizers        │
│                                                              │
│  Phase 4 (Fly): Automation                                   │
│  ├── Auto right-sizing (recommendations auto-applied)        │
│  ├── Dynamic commitment management (RI/SP portfolio)        │
│  ├── Real-time cost attribution per API endpoint            │
│  └── ML-based anomaly detection and auto-remediation         │
│                                                              │
│  Total savings: ~$50M annually                               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | What To Do Instead |
|---------|-------------|-------------------|
| **No tagging strategy** | Can't allocate costs → no accountability | Enforce mandatory tags from day 1 |
| **Only finance looks at cloud bills** | Engineers make infra decisions but don't see cost impact | Give engineers real-time cost visibility |
| **Optimizing without measuring** | You might optimize the wrong thing | First instrument, then optimize |
| **Cutting costs blindly** | May break reliability, SLA violations cost more than savings | Optimize cost-per-unit, not total spend |
| **Over-committing to Reserved Instances** | Business changes, workloads shift, money is locked up | Start with 50% RI coverage, grow gradually |
| **Ignoring data transfer** | "Shadow tax" that grows with traffic | Architect for low transfer: CDN, edge, compression |
| **Making it adversarial** | "Finance vs Engineering" kills collaboration | Frame as shared goal: "maximum value per dollar" |
| **One-time optimization** | Drift happens — savings erode over weeks | FinOps is continuous, not a project |

---

## When to Use / When NOT to Use

### When to Invest in FinOps

| Signal | Action |
|--------|--------|
| Cloud bill > $50K/month | Hire or assign a FinOps practitioner |
| Cloud bill > $500K/month | Dedicated FinOps team (2-4 people) |
| Cloud bill > $5M/month | Full FinOps practice with tooling investment |
| Cloud growth > revenue growth | Urgent — efficiency declining |
| Engineers don't know their spend | Visibility problem — start with Inform phase |
| > 20% unallocated costs | Tagging/governance problem |

### When FinOps is Premature

- ❌ Very early startup (< $5K/month cloud bill) — just use general best practices
- ❌ Single small team that already has visibility — formal FinOps adds overhead
- ❌ When growth is the only priority and burn rate is funded — optimize later
- ❌ Short-lived projects (< 6 months) — not enough time for ROI

---

## Key Takeaways

1. **FinOps is a culture change, not just a tool** — it requires engineering, finance, and business teams collaborating on cost decisions. The tools support the culture, not replace it.

2. **Start with visibility (Inform), then optimize, then operate** — you can't fix what you can't see. Tagging and allocation come first, optimization second.

3. **Measure cost per unit of business value** — total cloud spend going up is fine IF cost-per-transaction is going down. Context matters more than absolutes.

4. **The optimal strategy uses a mix of pricing models** — 40-50% Reserved/Committed, 20-30% Spot/Preemptible, 20-30% On-Demand. One size never fits all.

5. **Automate everything possible** — scheduled shutdowns, right-sizing recommendations, anomaly detection, budget alerts. Manual reviews don't scale.

6. **Make cost a first-class engineering metric** — include cost estimates in architecture docs, show cost dashboards alongside performance dashboards, celebrate efficiency wins.

7. **FinOps is continuous, not a project** — cloud environments drift constantly. New services launch, traffic patterns shift, pricing changes. Build FinOps into your operational rhythm (weekly/monthly reviews).

---

## What's Next?

Congratulations! You've completed **Part 26: Cost Optimization & Capacity Planning**. You now understand how to:
- Size servers correctly for your workload
- Choose the right cloud pricing model to save 30-70%
- Plan capacity ahead of demand
- Build a FinOps practice that scales with your organization

Next up is **Part 27: Architecture Evolution — The Journey of a Real App**, where you'll see how a real application evolves from a single server serving 100 users all the way to a planet-scale system serving billions — applying everything you've learned across all previous chapters.
