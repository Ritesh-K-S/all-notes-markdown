# SLIs, SLOs & SLAs — Defining Reliability

> **What you'll learn**: How to formally define and measure your system's reliability using Service Level Indicators (SLIs), Service Level Objectives (SLOs), and Service Level Agreements (SLAs) — the framework used by Google, Amazon, and every serious engineering organization.

---

## Real-Life Analogy

Imagine you run a **pizza delivery service**:

- **SLI (Service Level Indicator)** = What you MEASURE
  "What percentage of pizzas are delivered in under 30 minutes?"
  → Last month: 94.2% of deliveries were under 30 min

- **SLO (Service Level Objective)** = Your INTERNAL TARGET
  "We aim to deliver 95% of pizzas in under 30 minutes"
  → 94.2% < 95% target → We're not meeting our goal! Fix something!

- **SLA (Service Level Agreement)** = Your PROMISE TO CUSTOMERS (with consequences)
  "If we deliver less than 90% of pizzas in 30 minutes, you get a refund"
  → 94.2% > 90% SLA → No refunds needed, but we're close to our internal target

```
Strictness:  SLA (90%)  ←──────────→  SLO (95%)
                                         ↑
                                    Internal target
                                    (stricter than SLA
                                     so you catch problems
                                     BEFORE customers notice)
```

---

## The Relationship Between SLI, SLO, and SLA

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  SLI (Indicator)     SLO (Objective)      SLA (Agreement)       │
│  ────────────────    ────────────────     ─────────────────     │
│                                                                   │
│  WHAT you measure    WHAT you aim for     WHAT you promise       │
│                                           (with legal/financial   │
│  "Our metric"        "Our target"          consequences)         │
│                                                                   │
│  ┌───────────┐      ┌───────────┐       ┌───────────────┐      │
│  │ % of      │      │ 99.9% of  │       │ If uptime     │      │
│  │ successful│ ───▶ │ requests  │ ───▶  │ drops below   │      │
│  │ requests  │      │ must      │       │ 99.5%, we     │      │
│  │           │      │ succeed   │       │ give credits  │      │
│  └───────────┘      └───────────┘       └───────────────┘      │
│                                                                   │
│  Technical metric    Engineering goal    Business contract       │
│  (engineers own)     (engineers own)     (legal/business own)    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

KEY INSIGHT: SLO is always stricter than SLA!
  SLA: 99.5% (promise to customers)
  SLO: 99.9% (internal target — leaves margin for error)
  
  If you only target 99.5%, you'll sometimes breach it.
  By targeting 99.9%, you rarely breach 99.5%.
```

---

## SLI — Service Level Indicator

### What Makes a Good SLI?

An SLI is a **quantitative measure of service quality** as experienced by users.

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON SLIs                                    │
│                                                                   │
│  ┌──────────────────┬────────────────────────────────────────┐  │
│  │ Category         │ SLI Measurement                         │  │
│  ├──────────────────┼────────────────────────────────────────┤  │
│  │ AVAILABILITY     │ % of requests that succeed              │  │
│  │                  │ (non-5xx responses / total responses)   │  │
│  ├──────────────────┼────────────────────────────────────────┤  │
│  │ LATENCY          │ % of requests faster than threshold    │  │
│  │                  │ (requests < 300ms / total requests)    │  │
│  ├──────────────────┼────────────────────────────────────────┤  │
│  │ THROUGHPUT       │ Requests processed per second           │  │
│  │                  │ (used less often as SLI)               │  │
│  ├──────────────────┼────────────────────────────────────────┤  │
│  │ CORRECTNESS      │ % of responses with correct data       │  │
│  │                  │ (correct results / total results)      │  │
│  ├──────────────────┼────────────────────────────────────────┤  │
│  │ FRESHNESS        │ % of data updated within threshold     │  │
│  │                  │ (data age < 1min / total queries)      │  │
│  ├──────────────────┼────────────────────────────────────────┤  │
│  │ DURABILITY       │ % of data that's not lost              │  │
│  │                  │ (objects retrievable / objects stored)  │  │
│  └──────────────────┴────────────────────────────────────────┘  │
│                                                                   │
│  Formula (general):                                              │
│                                                                   │
│         good events                                              │
│  SLI = ─────────────── × 100%                                   │
│         total events                                             │
│                                                                   │
│  Example:                                                        │
│  999,700 successful requests / 1,000,000 total = 99.97%         │
└─────────────────────────────────────────────────────────────────┘
```

### SLI Examples for Different Services

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  API Service:                                                    │
│    SLI = requests returning 2xx or 4xx / total requests         │
│    (4xx = client error, not our fault → counts as "good")       │
│                                                                   │
│  Database:                                                       │
│    SLI = queries completing in < 100ms / total queries          │
│                                                                   │
│  Video Streaming:                                                │
│    SLI = streams with < 1% buffering / total streams            │
│                                                                   │
│  Search Engine:                                                  │
│    SLI = searches returning results in < 500ms / total searches │
│                                                                   │
│  Payment Service:                                                │
│    SLI = transactions completed successfully / total attempted  │
│                                                                   │
│  Email Delivery:                                                 │
│    SLI = emails delivered within 5 min / total emails sent      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## SLO — Service Level Objective

### Setting the Right Target

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT DIFFERENT SLOs MEAN IN PRACTICE                │
│                                                                   │
│  SLO       │ Downtime/month │ Downtime/year │ What it feels like│
│  ──────────┼────────────────┼───────────────┼───────────────────│
│  99%       │ 7.2 hours      │ 3.65 days     │ Unreliable       │
│  99.9%     │ 43.2 minutes   │ 8.76 hours    │ Good for most    │
│  99.95%    │ 21.6 minutes   │ 4.38 hours    │ Very reliable    │
│  99.99%    │ 4.3 minutes    │ 52.6 minutes  │ Extremely        │
│  99.999%   │ 26 seconds     │ 5.26 minutes  │ Ultra-reliable   │
│                                                                   │
│  IMPORTANT: Each additional "9" is 10x harder and 10x costlier!│
│                                                                   │
│  99% → 99.9%:    Add redundancy, basic monitoring              │
│  99.9% → 99.99%: Multi-region, auto-failover, chaos testing   │
│  99.99% → 99.999%: Massive investment, bespoke solutions       │
└─────────────────────────────────────────────────────────────────┘
```

### SLO Examples

```
SERVICE: Payment API
  SLO 1 (Availability): 99.95% of requests succeed (non-5xx)
  SLO 2 (Latency): 99% of requests complete in < 500ms
  SLO 3 (Latency): 99.9% of requests complete in < 2000ms

SERVICE: User Dashboard
  SLO 1 (Availability): 99.9% of page loads succeed
  SLO 2 (Latency): 95% of pages load in < 2 seconds
  
SERVICE: Data Pipeline  
  SLO 1 (Freshness): 99% of data is less than 5 minutes old
  SLO 2 (Correctness): 99.99% of records processed without error
```

---

## Error Budget — The Key Insight

The **error budget** is the ALLOWED amount of unreliability. It's what makes SLOs practical.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ERROR BUDGET                                    │
│                                                                   │
│  If SLO = 99.9% availability                                    │
│  Then Error Budget = 100% - 99.9% = 0.1%                       │
│                                                                   │
│  In a 30-day month:                                              │
│  0.1% × 30 days × 24 hours × 60 min = 43.2 minutes            │
│                                                                   │
│  You're ALLOWED 43.2 minutes of downtime this month!            │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │        ERROR BUDGET USAGE THIS MONTH                      │   │
│  │                                                            │   │
│  │  Total Budget: 43.2 minutes                               │   │
│  │  Used:         12 min (deploy caused 5xx) + 8 min (DB)   │   │
│  │  Remaining:    23.2 minutes                               │   │
│  │                                                            │   │
│  │  ████████████████░░░░░░░░░░░░░░░░░░░░░░ 46% used        │   │
│  │                                                            │   │
│  │  Status: ✅ Safe to deploy new features                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  WHEN ERROR BUDGET IS EXHAUSTED:                                │
│  ████████████████████████████████████████████ 100% used         │
│                                                                   │
│  → FREEZE all feature deployments                               │
│  → Focus exclusively on reliability improvements                │
│  → Only deploy bug fixes and reliability patches                │
│  → Resume features when budget recovers next period             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Why Error Budgets Are Brilliant

```
WITHOUT Error Budget:
  Engineers: "We need to deploy this feature!"
  SREs: "No! It might cause an outage!"
  → Constant conflict between velocity and reliability

WITH Error Budget:
  Budget available? → Deploy freely! Move fast!
  Budget exhausted? → Stop. Fix reliability. No arguing.
  → Clear, objective decision framework. No politics.
```

---

## SLA — Service Level Agreement

### What's in an SLA?

An SLA is a **legal/business contract** with customers that specifies:
1. What service levels you promise
2. How they're measured
3. What happens if you breach them (credits, refunds)

```
┌─────────────────────────────────────────────────────────────────┐
│              EXAMPLE: AWS SLA FOR EC2                             │
│                                                                   │
│  AWS promises:                                                   │
│  • Monthly Uptime Percentage: 99.99%                            │
│                                                                   │
│  If they breach:                                                 │
│  ┌──────────────────────────────┬────────────────────────────┐  │
│  │ Monthly Uptime               │ Service Credit             │  │
│  ├──────────────────────────────┼────────────────────────────┤  │
│  │ < 99.99% but ≥ 99.0%       │ 10% credit                 │  │
│  │ < 99.0% but ≥ 95.0%        │ 30% credit                 │  │
│  │ < 95.0%                      │ 100% credit                │  │
│  └──────────────────────────────┴────────────────────────────┘  │
│                                                                   │
│  Note: Customer must REQUEST the credit (it's not automatic!)   │
└─────────────────────────────────────────────────────────────────┘
```

### SLA vs SLO in Practice

```
                    SLA: 99.9%      SLO: 99.95%
                    (customer       (internal
                     promise)        target)
                        │               │
  ◀────────────────────│───────────────│─────────────────────▶
  0%                   │               │                   100%
                       │               │
                       │  ← BUFFER →  │
                       │               │
  If you target 99.95% (SLO), you'll rarely breach 99.9% (SLA).
  The gap between SLO and SLA is your safety margin.
```

---

## How It Works Internally

### Measuring SLIs in Practice

```
┌─────────────────────────────────────────────────────────────────┐
│           HOW TO MEASURE SLIs WITH PROMETHEUS                    │
│                                                                   │
│  AVAILABILITY SLI:                                              │
│  ─────────────────                                              │
│  Good events = requests where status != 5xx                     │
│  Total events = all requests                                    │
│                                                                   │
│  PromQL:                                                         │
│  1 - (                                                           │
│    sum(rate(http_requests_total{status=~"5.."}[30d]))           │
│    / sum(rate(http_requests_total[30d]))                        │
│  )                                                               │
│  Result: 0.9997 → 99.97% availability                          │
│                                                                   │
│  LATENCY SLI:                                                   │
│  ─────────────                                                   │
│  Good events = requests completing in < 300ms                   │
│  Total events = all requests                                    │
│                                                                   │
│  PromQL:                                                         │
│  sum(rate(http_request_duration_seconds_bucket{le="0.3"}[30d])) │
│  / sum(rate(http_request_duration_seconds_count[30d]))          │
│  Result: 0.9850 → 98.5% of requests under 300ms               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Error Budget Calculation

```
CALCULATING REMAINING ERROR BUDGET:

Given:
  SLO = 99.9% availability (30-day rolling window)
  Total requests in 30 days = 100,000,000
  Failed requests so far = 80,000

Error budget (total allowed failures):
  100,000,000 × (1 - 0.999) = 100,000 failures allowed

Budget consumed:
  80,000 / 100,000 = 80% consumed

Budget remaining:
  20,000 more failures allowed this period
  = 20% remaining

  ████████████████████████████████████████░░░░░░░░░░ 80% used

Status: ⚠️ Caution — slow down risky deployments
```

### SLO-Based Alerting (Multi-Window Multi-Burn-Rate)

Instead of simple threshold alerts, SLO-based alerting detects when you're **burning through your error budget too fast**:

```
┌─────────────────────────────────────────────────────────────────┐
│            MULTI-WINDOW BURN RATE ALERTING                        │
│                                                                   │
│  Burn Rate = how fast you're consuming error budget              │
│                                                                   │
│  Burn Rate 1x = consuming budget at exactly the expected rate   │
│  Burn Rate 14x = consuming 14x faster → budget gone in ~2 days │
│  Burn Rate 6x = consuming 6x faster → budget gone in ~5 days  │
│                                                                   │
│  ALERT RULES:                                                    │
│  ─────────────                                                   │
│  Page (Critical):                                                │
│    burn_rate > 14x for the last 1 hour                          │
│    AND burn_rate > 14x for the last 5 minutes                   │
│    → "You'll exhaust your entire monthly budget in 2 days!"    │
│                                                                   │
│  Ticket (Warning):                                               │
│    burn_rate > 6x for the last 6 hours                          │
│    AND burn_rate > 6x for the last 30 minutes                   │
│    → "Budget being consumed faster than sustainable"            │
│                                                                   │
│  WHY TWO WINDOWS?                                                │
│  Short window (5m/30m): Confirms it's happening NOW             │
│  Long window (1h/6h): Confirms it's sustained, not a blip      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: SLO Monitoring and Error Budget Tracking

```python
# slo_monitor.py
# Track SLIs and calculate error budget consumption

from prometheus_client import Counter, Gauge, Histogram
from dataclasses import dataclass
from datetime import datetime, timedelta

# --- Metrics for SLI tracking ---
TOTAL_REQUESTS = Counter(
    'sli_requests_total',
    'Total requests (for SLI calculation)',
    ['service']
)
SUCCESSFUL_REQUESTS = Counter(
    'sli_requests_successful_total',
    'Successful requests (non-5xx)',
    ['service']
)
FAST_REQUESTS = Counter(
    'sli_requests_fast_total',
    'Requests completing under latency threshold',
    ['service']
)

# Error budget gauge (for Grafana dashboard)
ERROR_BUDGET_REMAINING = Gauge(
    'slo_error_budget_remaining_ratio',
    'Remaining error budget (0.0 = exhausted, 1.0 = full)',
    ['service', 'slo_name']
)

@dataclass
class SLOConfig:
    name: str
    target: float          # e.g., 0.999 for 99.9%
    window_days: int       # Rolling window (typically 30)
    
    @property
    def error_budget_ratio(self) -> float:
        """Maximum allowed failure ratio."""
        return 1.0 - self.target  # 0.001 for 99.9% SLO

# Define SLOs
AVAILABILITY_SLO = SLOConfig(name="availability", target=0.999, window_days=30)
LATENCY_SLO = SLOConfig(name="latency_300ms", target=0.99, window_days=30)

def calculate_error_budget(slo: SLOConfig, total_events: int, 
                           bad_events: int) -> dict:
    """Calculate error budget consumption."""
    
    allowed_bad_events = int(total_events * slo.error_budget_ratio)
    budget_consumed_ratio = bad_events / allowed_bad_events if allowed_bad_events > 0 else 1.0
    budget_remaining_ratio = max(0, 1.0 - budget_consumed_ratio)
    
    # Calculate burn rate
    # If we're X days into the window, we should have consumed X/30 of budget
    days_elapsed = 15  # Example: halfway through the month
    expected_consumption = days_elapsed / slo.window_days
    burn_rate = budget_consumed_ratio / expected_consumption if expected_consumption > 0 else 0
    
    result = {
        "slo_name": slo.name,
        "slo_target": f"{slo.target * 100}%",
        "total_events": total_events,
        "bad_events": bad_events,
        "allowed_bad_events": allowed_bad_events,
        "budget_consumed": f"{budget_consumed_ratio * 100:.1f}%",
        "budget_remaining": f"{budget_remaining_ratio * 100:.1f}%",
        "burn_rate": f"{burn_rate:.1f}x",
        "status": "OK" if budget_remaining_ratio > 0.25 else 
                 "WARNING" if budget_remaining_ratio > 0 else "EXHAUSTED"
    }
    
    # Update Prometheus gauge
    ERROR_BUDGET_REMAINING.labels(
        service="payment-service",
        slo_name=slo.name
    ).set(budget_remaining_ratio)
    
    return result

# Example usage
budget = calculate_error_budget(
    slo=AVAILABILITY_SLO,
    total_events=50_000_000,  # 50M requests in 15 days
    bad_events=35_000         # 35K failures
)
print(budget)
# {'slo_target': '99.9%', 'allowed_bad_events': 50000,
#  'budget_consumed': '70.0%', 'burn_rate': '1.4x', 'status': 'WARNING'}
```

### Java: SLO Error Budget Calculator

```java
// SloMonitor.java
// Calculate and expose error budget metrics

import io.micrometer.core.instrument.*;
import java.time.Duration;
import java.time.Instant;

public class SloMonitor {
    
    private final MeterRegistry registry;
    
    // SLO Configuration
    record SloConfig(String name, double target, Duration window) {
        double errorBudgetRatio() { return 1.0 - target; }
    }
    
    record ErrorBudgetStatus(
        String sloName, double targetPercent,
        long totalEvents, long badEvents, long allowedBadEvents,
        double budgetConsumedPercent, double budgetRemainingPercent,
        double burnRate, String status
    ) {}
    
    private final SloConfig availabilitySlo = new SloConfig(
        "availability", 0.999, Duration.ofDays(30));
    
    private final SloConfig latencySlo = new SloConfig(
        "latency_p99_300ms", 0.99, Duration.ofDays(30));
    
    public SloMonitor(MeterRegistry registry) {
        this.registry = registry;
    }
    
    public ErrorBudgetStatus calculateBudget(SloConfig slo, 
                                              long totalEvents, 
                                              long badEvents,
                                              Duration elapsed) {
        long allowedBad = (long)(totalEvents * slo.errorBudgetRatio());
        double consumed = allowedBad > 0 ? (double) badEvents / allowedBad : 1.0;
        double remaining = Math.max(0, 1.0 - consumed);
        
        // Burn rate: are we consuming faster than expected?
        double expectedConsumption = (double) elapsed.toHours() / slo.window().toHours();
        double burnRate = expectedConsumption > 0 ? consumed / expectedConsumption : 0;
        
        String status = remaining > 0.25 ? "OK" : remaining > 0 ? "WARNING" : "EXHAUSTED";
        
        // Expose as Prometheus gauge
        Gauge.builder("slo.error_budget.remaining", () -> remaining)
            .tag("slo", slo.name())
            .tag("service", "payment-service")
            .register(registry);
        
        return new ErrorBudgetStatus(
            slo.name(), slo.target() * 100,
            totalEvents, badEvents, allowedBad,
            consumed * 100, remaining * 100,
            burnRate, status
        );
    }
    
    // Example: Check budget and decide if deployment is safe
    public boolean isSafeToD deploy() {
        var status = calculateBudget(availabilitySlo, 
            getTotalRequests(), getFailedRequests(), getElapsedTime());
        
        // Only deploy if we have > 25% budget remaining
        return status.budgetRemainingPercent() > 25.0;
    }
}
```

---

## Infrastructure Example: SLO Dashboard in Grafana

```yaml
# Prometheus recording rules for SLO calculations
# These pre-compute expensive queries so dashboards load instantly

groups:
  - name: slo_recording_rules
    interval: 1m
    rules:
      # Availability SLI (30-day rolling)
      - record: sli:availability:ratio_rate30d
        expr: |
          1 - (
            sum(increase(http_requests_total{status=~"5.."}[30d]))
            / sum(increase(http_requests_total[30d]))
          )
      
      # Latency SLI (% requests under 300ms, 30-day rolling)
      - record: sli:latency_300ms:ratio_rate30d
        expr: |
          sum(increase(http_request_duration_seconds_bucket{le="0.3"}[30d]))
          / sum(increase(http_request_duration_seconds_count[30d]))
      
      # Error budget remaining (availability)
      - record: slo:availability:error_budget_remaining
        expr: |
          1 - (
            (1 - sli:availability:ratio_rate30d)
            / (1 - 0.999)
          )
        # Result: 1.0 = full budget, 0.0 = exhausted
      
      # Burn rate (1-hour window)
      - record: slo:availability:burn_rate_1h
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[1h]))
          / sum(rate(http_requests_total[1h]))
          / (1 - 0.999)
        # Result: 1.0 = normal burn, >14 = page-worthy

  - name: slo_alerts
    rules:
      # Page: burning budget 14x too fast (will exhaust in ~2 days)
      - alert: SLOBudgetBurnHigh
        expr: |
          slo:availability:burn_rate_1h > 14
          and slo:availability:burn_rate_1h > 14
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error budget burning 14x too fast — will exhaust in ~2 days"
      
      # Ticket: budget nearly exhausted
      - alert: SLOBudgetLow
        expr: slo:availability:error_budget_remaining < 0.25
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Error budget < 25% remaining — slow down risky changes"
```

---

## Real-World Example

### How Google Uses SLOs (from the SRE Book)

```
┌─────────────────────────────────────────────────────────────────┐
│              GOOGLE'S SLO FRAMEWORK                               │
│                                                                   │
│  Every service at Google has:                                    │
│  • Clear SLIs (availability, latency, correctness)              │
│  • Published SLOs (reviewed quarterly)                          │
│  • Error budget policies (what happens when exhausted)          │
│                                                                   │
│  Error Budget Policy Example (Google Search):                    │
│  ─────────────────────────────────────────────                  │
│  Budget remaining > 50%:                                        │
│    → Deploy freely, run experiments, launch features            │
│                                                                   │
│  Budget remaining 25-50%:                                       │
│    → Deploy with extra caution, smaller batches                 │
│    → No risky experiments                                       │
│                                                                   │
│  Budget remaining < 25%:                                        │
│    → Feature freeze (only reliability fixes)                    │
│    → Incident review, root cause analysis                      │
│                                                                   │
│  Budget exhausted (0%):                                         │
│    → Complete code freeze                                       │
│    → All hands on reliability                                   │
│    → Postmortem required for every incident                    │
│    → Changes reviewed by VP                                     │
│                                                                   │
│  Result:                                                        │
│  • Product teams understand the cost of unreliability          │
│  • SRE team has objective "stop" mechanism                     │
│  • No arguing about "should we deploy?" — data decides         │
└─────────────────────────────────────────────────────────────────┘
```

### How Amazon Defines SLAs for AWS Services

```
AWS Service SLAs (public):

┌────────────────────┬────────────────┬─────────────────────────┐
│ Service            │ SLA (Monthly)  │ Credit if Breached      │
├────────────────────┼────────────────┼─────────────────────────┤
│ EC2                │ 99.99%         │ 10-100% credit         │
│ S3                 │ 99.9%          │ 10-25% credit          │
│ RDS (Multi-AZ)    │ 99.95%         │ 10-100% credit         │
│ Lambda             │ 99.95%         │ 10-100% credit         │
│ DynamoDB           │ 99.999%        │ 10-25% credit          │
│ CloudFront         │ 99.9%          │ 10-25% credit          │
└────────────────────┴────────────────┴─────────────────────────┘

Note: Internal SLOs at AWS are much stricter than published SLAs!
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Setting SLO = 100%** | Impossible to achieve, prevents ANY changes | Set realistic targets (99.9% is very good) |
| **SLO same as SLA** | No safety margin — you'll breach the SLA | SLO should be 2-10x stricter than SLA |
| **Measuring internal metrics, not user experience** | Server says 200 but user sees blank page | Measure from the user's perspective |
| **Not using error budgets** | SLOs become meaningless targets nobody follows | Enforce consequences when budget is exhausted |
| **Too many SLOs** | Dilutes focus, hard to manage | 2-4 SLOs per service maximum |
| **Never revising SLOs** | Business changes, old SLOs become irrelevant | Review quarterly, adjust based on data |
| **Treating SLOs as goals to exceed** | 99.999% is wasteful if users only need 99.9% | Match SLO to actual user expectations |

> **Important**: Being MORE reliable than necessary is a WASTE. If users are happy at 99.9%, engineering for 99.99% means spending 10x more for improvements nobody notices. That time could be spent on features.

---

## When to Use / When NOT to Use

### You Need SLOs When:
- ✅ You have production services with real users
- ✅ You need to balance feature velocity vs reliability
- ✅ Multiple teams depend on your service
- ✅ You want objective criteria for "is this good enough?"
- ✅ You need to decide whether to freeze deploys after an incident

### SLAs Are Needed When:
- ✅ You sell a service to paying customers
- ✅ Regulatory requirements demand guaranteed uptime
- ✅ Business contracts require defined service levels
- ❌ Internal services usually don't need formal SLAs (SLOs suffice)

### Start Simple:
```
Day 1 MVP:
  • Pick ONE SLI (availability or latency)
  • Set ONE SLO (start with 99.9% — adjust later)
  • Track error budget in a Grafana dashboard
  • Alert when burn rate is too high

Mature:
  • 2-3 SLIs per service
  • Error budget policies documented
  • Automated deployment gates based on budget
  • Quarterly SLO reviews
```

---

## Key Takeaways

1. **SLI = what you measure**, SLO = your internal target, SLA = your external promise with consequences. They're nested: SLA ⊂ SLO ⊂ SLI.

2. **Error budgets make SLOs actionable** — they tell you when to push features (budget available) and when to stop and fix (budget exhausted).

3. **100% reliability is wrong** — it's impossible AND wasteful. Set SLOs that match what users actually need.

4. **SLOs should be stricter than SLAs** — the gap is your safety margin. If SLA is 99.9%, target 99.95% internally.

5. **Measure from the user's perspective** — a server returning 200 OK doesn't mean the user had a good experience.

6. **Burn-rate alerting beats threshold alerting** — instead of "error rate > 5%", alert on "consuming budget 14x too fast."

7. **SLOs resolve the velocity vs reliability debate** — when budget is available, move fast. When it's not, stop and fix. Data decides, not opinions.

---

## What's Next?

Now that you understand how to define and measure reliability, let's look at the tools that tie everything together — **APM (Application Performance Monitoring)** tools like Datadog, New Relic, and Dynatrace that provide a unified view of your system's health.

**Next: [08-apm-tools.md](./08-apm-tools.md)** — APM Tools (Datadog, New Relic, Dynatrace)
