# Capacity Planning — Preparing for Growth

> **What you'll learn**: How to forecast future infrastructure needs, plan for traffic growth, and ensure your system can handle 10x or 100x load without emergency firefighting or surprise outages.

---

## Real-Life Analogy

Imagine you run a chain of coffee shops. You need to plan ahead:

- **How many cups of coffee will you sell next month?** (Traffic forecast)
- **Do you have enough baristas for the morning rush?** (Compute capacity)
- **Is your coffee bean storage large enough for the holiday season?** (Storage growth)
- **If a celebrity tweets about your shop and 50x people show up tomorrow, will you survive?** (Burst capacity)

You can't hire baristas the same morning a rush happens — hiring takes time. Similarly, you can't provision infrastructure the second traffic spikes — planning, provisioning, and testing take time.

**Capacity planning is the discipline of predicting future demand and ensuring your system is ready BEFORE the demand arrives.**

---

## Core Concept Explained Step-by-Step

### What Capacity Planning Covers

```
┌─────────────────────────────────────────────────────────────────┐
│                CAPACITY PLANNING SCOPE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   COMPUTE    │    │   STORAGE    │    │   NETWORK    │      │
│  │  CPU / RAM   │    │  Disk / DB   │    │  Bandwidth   │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │  DATABASE    │    │    CACHE     │    │   PEOPLE     │      │
│  │  Connections │    │  Hit Rates   │    │  (On-Call,   │      │
│  │  / IOPS     │    │  / Memory    │    │   DevOps)    │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                                 │
│  Time horizons:                                                 │
│  • Short-term: Next 1-4 weeks (upcoming events, releases)      │
│  • Medium-term: Next 1-6 months (growth trends)                │
│  • Long-term: 6-24 months (strategic planning, contracts)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The Capacity Planning Process

```
┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│  MEASURE  │────▶│  FORECAST │────▶│   PLAN    │────▶│  EXECUTE  │
│           │     │           │     │           │     │           │
│ Current   │     │ Predict   │     │ Decide    │     │ Provision │
│ usage &   │     │ future    │     │ what to   │     │ & test    │
│ trends    │     │ demand    │     │ provision │     │           │
└───────────┘     └───────────┘     └───────────┘     └───────────┘
      │                                                      │
      │                                                      │
      └──────────────────── REPEAT ──────────────────────────┘
                    (Monthly / Quarterly)
```

### Step 1: Measure Current State

Before forecasting, you need a baseline:

```
┌──────────────────────────────────────────────────────────────┐
│           BASELINE METRICS TO TRACK                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Traffic:                                                    │
│  ├── Peak RPS (requests per second)                          │
│  ├── Average RPS                                             │
│  ├── Peak-to-average ratio                                   │
│  └── DAU / MAU (daily/monthly active users)                  │
│                                                              │
│  Compute:                                                    │
│  ├── CPU utilization (avg, p95, max)                         │
│  ├── Memory utilization (avg, p95, max)                      │
│  └── Number of instances running                             │
│                                                              │
│  Storage:                                                    │
│  ├── Total data size (DB, files, logs)                       │
│  ├── Growth rate (GB/day or GB/month)                        │
│  └── IOPS utilization (read + write)                         │
│                                                              │
│  Network:                                                    │
│  ├── Bandwidth utilization                                   │
│  ├── Connection count                                        │
│  └── Latency (p50, p95, p99)                                │
│                                                              │
│  Application:                                                │
│  ├── Response time (p50, p95, p99)                           │
│  ├── Error rate                                              │
│  └── Queue depth (if using message queues)                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Step 2: Forecast Future Demand

Use multiple signals to predict growth:

```
┌──────────────────────────────────────────────────────────────┐
│           FORECASTING METHODS                                 │
├────────────────────┬─────────────────────────────────────────┤
│ Method             │ Description                              │
├────────────────────┼─────────────────────────────────────────┤
│ Trend Extrapolation│ Extend historical growth curve forward   │
│ Business Input     │ Sales forecasts, marketing campaigns     │
│ Seasonal Patterns  │ Holiday spikes, end-of-month peaks       │
│ Event-Based        │ Product launches, TV ads, viral events   │
│ Benchmark          │ Compare with similar companies at scale  │
└────────────────────┴─────────────────────────────────────────┘

Example Growth Projection:

     Traffic (RPS)
     │
3000 │                                          ╱ Optimistic (+50%)
     │                                       ╱╱
2000 │                                   ╱╱╱    ── Expected (+30%)
     │                              ╱╱╱╱
1000 │         ╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱       ╲╲── Conservative (+15%)
     │    ╱╱╱╱╱
 500 │╱╱╱╱
     └────────────────────────────────────────── Time
      Jan    Mar    Jun    Sep    Dec
```

### Step 3: Calculate Headroom

**The Golden Rule: Never operate above 70% capacity at peak.**

```
┌──────────────────────────────────────────────────────────────┐
│           CAPACITY HEADROOM MODEL                             │
│                                                              │
│  100% ┤ ████████████████████████████████ ← System crashes   │
│       │                                                      │
│   80% ┤ ████████████████████████ ← DANGER ZONE              │
│       │                           (latency spikes, errors)   │
│   70% ┤ ████████████████████ ← TARGET MAX at peak           │
│       │                                                      │
│   50% ┤ █████████████████ ← Comfortable operating range     │
│       │                                                      │
│   30% ┤ ████████████ ← Average utilization (healthy)        │
│       │                                                      │
│    0% ┤                                                      │
│       └──────────────────────────────────────────────────    │
│                                                              │
│  Headroom = (100% - Peak Utilization) = buffer for spikes   │
│  Target: 30-40% headroom at PEAK traffic                    │
└──────────────────────────────────────────────────────────────┘
```

### Step 4: Define Scaling Triggers

```
┌────────────────────────────────────────────────────────────┐
│          CAPACITY THRESHOLDS & ACTIONS                       │
├──────────────┬─────────────────────────────────────────────┤
│ Utilization  │ Action                                       │
├──────────────┼─────────────────────────────────────────────┤
│  < 30%       │ Consider downsizing (over-provisioned)      │
│  30-60%      │ Healthy — no action needed                  │
│  60-70%      │ Start planning expansion (2-4 weeks out)    │
│  70-80%      │ URGENT — provision more capacity NOW        │
│  > 80%       │ EMERGENCY — risk of degradation/outage      │
└──────────────┴─────────────────────────────────────────────┘
```

---

## How It Works Internally

### The Capacity Planning Formula

```
Required Capacity = (Forecasted Peak Traffic × Per-Request Cost)
                   / Target Utilization

Where:
  Forecasted Peak = Current Peak × Growth Rate × Spike Factor
  Target Utilization = 0.6 to 0.7 (60-70%)
  Spike Factor = 1.5 to 3.0 (for unexpected bursts)
```

**Detailed Example:**

```
Current state:
  - Peak RPS: 5,000
  - Each request uses: 50ms CPU, 10 MB RAM, 20 KB response
  - Current servers: 10 × m6i.xlarge (4 CPU, 16 GB each)
  - Current peak utilization: 65%

Growth forecast (6 months):
  - User growth: +40%
  - Feature additions that increase per-request cost: +20%
  - Upcoming marketing campaign: +30% spike

Required capacity in 6 months:
  Peak RPS = 5,000 × 1.4 (growth) = 7,000
  With campaign spike = 7,000 × 1.3 = 9,100 RPS
  Per-request CPU cost increases 20%: effectively 9,100 × 1.2 = 10,920 RPS equivalent

  Current capacity at 70% target: 5,000 / 0.65 × 0.70 = 5,385 RPS
  (We're already near capacity!)

  Needed: 10,920 / 0.70 = 15,600 RPS capacity
  = 15,600 / 5,385 × 10 servers ≈ 29 servers

  Action: Need ~19 more servers (or equivalent with larger instances)
```

### Database Capacity Planning

Databases are the hardest component to scale, so plan furthest ahead:

```
┌──────────────────────────────────────────────────────────────┐
│         DATABASE CAPACITY DIMENSIONS                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. CONNECTION CAPACITY                                      │
│     Current: 150 connections / Max: 200                      │
│     Growth: +10 connections/month                            │
│     Runway: 5 months before hitting limit                    │
│     Action: Upgrade to larger instance or add PgBouncer      │
│                                                              │
│  2. STORAGE CAPACITY                                         │
│     Current: 450 GB used / 1 TB provisioned                  │
│     Growth: 30 GB/month                                      │
│     Runway: 18 months before full                            │
│     Action: Monitor; resize at 70% (700 GB)                  │
│                                                              │
│  3. IOPS CAPACITY                                            │
│     Current: 6,000 IOPS peak / 10,000 provisioned           │
│     Growth: +500 IOPS/month                                  │
│     Runway: 8 months before saturation                       │
│     Action: Plan upgrade to io2 volume at 8,000 IOPS        │
│                                                              │
│  4. QUERY PERFORMANCE                                        │
│     Current p99 latency: 45ms                                │
│     Acceptable threshold: 100ms                              │
│     Growth in query complexity: +15%/quarter                 │
│     Action: Add read replicas when p99 > 70ms               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Load Testing for Capacity Validation

Capacity plans must be validated with load tests:

```
        LOAD TESTING STRATEGY

        ┌─────────────────────────────────┐
        │          TARGET LOAD            │
        │    (Forecasted peak × 1.5)      │
        └────────────────┬────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  RAMP TEST   │ │  SPIKE TEST  │ │  SOAK TEST   │
│              │ │              │ │              │
│ Gradually    │ │ Sudden 10x   │ │ Sustained    │
│ increase     │ │ traffic      │ │ load for     │
│ load to find │ │ burst for    │ │ 24-48 hours  │
│ breaking     │ │ 5-10 minutes │ │              │
│ point        │ │              │ │ Find: memory │
│              │ │ Find: auto-  │ │ leaks, conn  │
│ Find: max    │ │ scaling      │ │ exhaustion,  │
│ throughput   │ │ speed,       │ │ disk fill    │
│              │ │ error rate   │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

## Code Examples

### Python — Capacity Forecasting with Linear Regression

```python
# capacity_forecast.py — Predict when you'll run out of capacity

import numpy as np
from datetime import datetime, timedelta

class CapacityForecaster:
    """Forecasts resource usage and predicts capacity exhaustion dates."""
    
    def __init__(self, metric_name: str, max_capacity: float):
        self.metric_name = metric_name
        self.max_capacity = max_capacity
        self.timestamps = []
        self.values = []
    
    def add_data_point(self, timestamp: datetime, value: float):
        """Add a historical measurement."""
        self.timestamps.append(timestamp.timestamp())
        self.values.append(value)
    
    def forecast(self, target_utilization_pct: float = 70.0):
        """
        Predict when capacity will be exhausted.
        Returns the estimated date and growth rate.
        """
        if len(self.values) < 2:
            raise ValueError("Need at least 2 data points")
        
        # Simple linear regression: y = mx + b
        x = np.array(self.timestamps)
        y = np.array(self.values)
        
        # Normalize x for numerical stability
        x_mean = x.mean()
        x_norm = x - x_mean
        
        # Calculate slope (m) and intercept (b)
        m = np.sum(x_norm * (y - y.mean())) / np.sum(x_norm ** 2)
        b = y.mean() - m * x_mean
        
        # When will we hit the target threshold?
        threshold = self.max_capacity * (target_utilization_pct / 100)
        
        if m <= 0:
            return None, m  # Usage is flat or decreasing
        
        # Solve: threshold = m * t + b → t = (threshold - b) / m
        exhaustion_timestamp = (threshold - b) / m
        exhaustion_date = datetime.fromtimestamp(exhaustion_timestamp)
        
        # Growth rate per day
        daily_growth = m * 86400  # seconds in a day
        
        return exhaustion_date, daily_growth
    
    def print_report(self):
        """Print a capacity forecast report."""
        current_value = self.values[-1]
        current_pct = (current_value / self.max_capacity) * 100
        
        print(f"\n{'='*55}")
        print(f"  CAPACITY FORECAST: {self.metric_name}")
        print(f"{'='*55}")
        print(f"  Current usage:    {current_value:.1f} / {self.max_capacity:.1f} "
              f"({current_pct:.1f}%)")
        
        date_70, growth = self.forecast(70)
        date_80, _ = self.forecast(80)
        date_100, _ = self.forecast(100)
        
        if date_70 and date_70 > datetime.now():
            days_to_70 = (date_70 - datetime.now()).days
            print(f"  70% threshold:    {date_70.strftime('%Y-%m-%d')} "
                  f"({days_to_70} days)")
        
        if date_80 and date_80 > datetime.now():
            days_to_80 = (date_80 - datetime.now()).days
            print(f"  80% threshold:    {date_80.strftime('%Y-%m-%d')} "
                  f"({days_to_80} days)")
        
        if date_100 and date_100 > datetime.now():
            days_to_100 = (date_100 - datetime.now()).days
            print(f"  100% exhaustion:  {date_100.strftime('%Y-%m-%d')} "
                  f"({days_to_100} days)")
        
        print(f"  Daily growth rate: {growth:.2f} units/day")
        print(f"{'='*55}")


# Example: Track disk usage growth over 6 months
forecaster = CapacityForecaster("PostgreSQL Disk (GB)", max_capacity=1000)

base_date = datetime(2024, 1, 1)
# Simulate monthly data points showing growth
monthly_values = [420, 448, 479, 512, 549, 590]

for i, value in enumerate(monthly_values):
    forecaster.add_data_point(base_date + timedelta(days=30*i), value)

forecaster.print_report()
```

### Java — Capacity Monitoring with Threshold Alerts

```java
// CapacityMonitor.java — Monitor resources and alert before exhaustion

import java.time.Instant;
import java.time.Duration;
import java.util.*;

public class CapacityMonitor {
    
    enum AlertLevel { OK, WARNING, CRITICAL, EMERGENCY }
    
    record ResourceMetric(
        String name,
        double currentValue,
        double maxCapacity,
        double growthRatePerDay,  // units per day
        Instant lastUpdated
    ) {
        double utilizationPct() {
            return (currentValue / maxCapacity) * 100;
        }
        
        // Days until reaching threshold percentage
        long daysUntil(double thresholdPct) {
            double threshold = maxCapacity * (thresholdPct / 100);
            double remaining = threshold - currentValue;
            if (growthRatePerDay <= 0 || remaining <= 0) return -1;
            return (long) (remaining / growthRatePerDay);
        }
    }
    
    private final List<ResourceMetric> resources = new ArrayList<>();
    private final Map<AlertLevel, Double> thresholds = Map.of(
        AlertLevel.OK, 60.0,
        AlertLevel.WARNING, 70.0,
        AlertLevel.CRITICAL, 80.0,
        AlertLevel.EMERGENCY, 90.0
    );
    
    public void addResource(ResourceMetric metric) {
        resources.add(metric);
    }
    
    public AlertLevel getAlertLevel(ResourceMetric metric) {
        double util = metric.utilizationPct();
        if (util >= 90) return AlertLevel.EMERGENCY;
        if (util >= 80) return AlertLevel.CRITICAL;
        if (util >= 70) return AlertLevel.WARNING;
        return AlertLevel.OK;
    }
    
    public void printCapacityReport() {
        System.out.println("╔══════════════════════════════════════════════════════╗");
        System.out.println("║           CAPACITY PLANNING REPORT                  ║");
        System.out.println("╠══════════════════════════════════════════════════════╣");
        
        for (ResourceMetric r : resources) {
            AlertLevel level = getAlertLevel(r);
            String icon = switch (level) {
                case EMERGENCY -> "🔴";
                case CRITICAL  -> "🟠";
                case WARNING   -> "🟡";
                case OK        -> "🟢";
            };
            
            System.out.printf("║ %s %-20s  %5.1f%% used  (%,.0f / %,.0f)%n",
                icon, r.name(), r.utilizationPct(), 
                r.currentValue(), r.maxCapacity());
            System.out.printf("║    Growth: %.1f/day | 70%% in %d days | "
                + "100%% in %d days%n",
                r.growthRatePerDay(), r.daysUntil(70), r.daysUntil(100));
            System.out.println("║");
        }
        
        System.out.println("╚══════════════════════════════════════════════════════╝");
    }
    
    public static void main(String[] args) {
        CapacityMonitor monitor = new CapacityMonitor();
        
        monitor.addResource(new ResourceMetric(
            "DB Disk (GB)", 680, 1000, 3.2, Instant.now()));
        monitor.addResource(new ResourceMetric(
            "DB Connections", 145, 200, 0.8, Instant.now()));
        monitor.addResource(new ResourceMetric(
            "Redis Memory (GB)", 22, 64, 0.3, Instant.now()));
        monitor.addResource(new ResourceMetric(
            "Kafka Partitions", 180, 256, 1.5, Instant.now()));
        
        monitor.printCapacityReport();
    }
}
```

---

## Infrastructure Examples

### Grafana Dashboard — Capacity Planning Panels

```yaml
# grafana-capacity-dashboard.json (simplified YAML representation)
# Capacity Planning Dashboard with forecast panels

panels:
  - title: "CPU Capacity Runway"
    type: timeseries
    query: |
      # Show current usage + linear forecast
      avg(rate(container_cpu_usage_seconds_total[5m])) by (service)
      # Plus prediction line extending 90 days
    alert:
      condition: "when avg > 70% for 1h"
      severity: warning
      message: "Service {{ $labels.service }} CPU > 70% - plan expansion"
      
  - title: "Database Disk Usage Forecast"
    type: timeseries
    query: |
      pg_database_size_bytes{datname="production"}
    forecast:
      method: linear
      range: 90d       # Show 90-day projection
      threshold_lines:
        - label: "70% Warning"
          value: 716800000000  # 700 GB of 1 TB
        - label: "Resize Trigger"
          value: 858993459200  # 800 GB
          
  - title: "Capacity Exhaustion Countdown"
    type: stat
    queries:
      - name: "DB Disk"
        expr: "(1000 - pg_size_gb) / daily_growth_gb"
        unit: "days"
      - name: "Redis Memory"
        expr: "(64 - redis_used_memory_gb) / daily_growth_gb"
        unit: "days"
      - name: "Kafka Storage"
        expr: "(kafka_total_gb - kafka_used_gb) / daily_growth_gb"
        unit: "days"
    thresholds:
      - color: red
        value: 30    # Less than 30 days = red
      - color: orange
        value: 60    # 30-60 days = orange
      - color: green
        value: 90    # 60+ days = green
```

### AWS CloudWatch Alarm for Capacity Thresholds

```yaml
# cloudwatch-capacity-alarms.yaml (CloudFormation)
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  # Alert when RDS storage hits 70%
  RDSStorageAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: rds-storage-capacity-warning
      AlarmDescription: "DB storage > 70% - plan expansion"
      MetricName: FreeStorageSpace
      Namespace: AWS/RDS
      Statistic: Average
      Period: 3600          # Check hourly
      EvaluationPeriods: 3  # 3 consecutive hours
      Threshold: 300000000000  # 300 GB free (out of 1 TB)
      ComparisonOperator: LessThanThreshold
      AlarmActions:
        - !Ref CapacityPlanningTopic

  # Alert when ASG is running near max
  ASGCapacityAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: asg-capacity-near-max
      AlarmDescription: "ASG at 80% of max - increase max or add capacity"
      MetricName: GroupInServiceInstances
      Namespace: AWS/AutoScaling
      Dimensions:
        - Name: AutoScalingGroupName
          Value: api-production-asg
      Statistic: Average
      Period: 300
      EvaluationPeriods: 6  # 30 minutes sustained
      Threshold: 16         # 16 out of max 20 instances
      ComparisonOperator: GreaterThanOrEqualToThreshold
      AlarmActions:
        - !Ref CapacityPlanningTopic

  CapacityPlanningTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: capacity-planning-alerts
      Subscription:
        - Protocol: email
          Endpoint: platform-team@company.com
```

### Load Test Configuration for Capacity Validation

```python
# locustfile.py — Load test to validate capacity plan
from locust import HttpUser, task, between

class CapacityValidationUser(HttpUser):
    """
    Simulates production traffic patterns to validate
    that provisioned capacity meets forecasted demand.
    
    Target: 10,000 RPS with p99 < 200ms
    """
    wait_time = between(0.5, 2.0)
    
    # Traffic distribution matching production
    @task(60)  # 60% of traffic
    def browse_products(self):
        self.client.get("/api/products?page=1&limit=20")
    
    @task(25)  # 25% of traffic
    def view_product_detail(self):
        product_id = self._random_product_id()
        self.client.get(f"/api/products/{product_id}")
    
    @task(10)  # 10% of traffic
    def search(self):
        self.client.get("/api/search?q=laptop&limit=10")
    
    @task(5)   # 5% of traffic (write-heavy)
    def add_to_cart(self):
        self.client.post("/api/cart/items", json={
            "product_id": self._random_product_id(),
            "quantity": 1
        })
    
    def _random_product_id(self):
        import random
        return random.randint(1, 100000)

# Run with:
# locust -f locustfile.py --host=https://staging.example.com
#        --users 10000 --spawn-rate 100 --run-time 30m
```

---

## Real-World Example

### How Amazon Plans for Prime Day

Amazon's Prime Day generates 10-100x normal traffic in a 48-hour window. Their capacity planning process starts **6 months ahead**:

```
┌──────────────────────────────────────────────────────────────┐
│          AMAZON PRIME DAY CAPACITY PLANNING                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  T-6 months: FORECAST                                        │
│  ├── Analyze previous Prime Days                             │
│  ├── Get business input (deals planned, marketing spend)    │
│  ├── Model traffic per service (checkout, search, catalog)  │
│  └── Add 30% buffer above forecast                          │
│                                                              │
│  T-3 months: PROVISION                                       │
│  ├── Reserve additional capacity in target regions          │
│  ├── Pre-warm databases and caches                          │
│  ├── Scale Elasticsearch clusters                           │
│  └── Increase Kafka partition count                         │
│                                                              │
│  T-1 month: VALIDATE                                         │
│  ├── "GameDay" load tests at 2x forecasted peak            │
│  ├── Chaos engineering (kill services at peak load)         │
│  ├── Verify auto-scaling triggers and speed                 │
│  └── Test failover to other regions                         │
│                                                              │
│  T-1 week: FREEZE & PREPARE                                 │
│  ├── Code freeze (no deployments)                           │
│  ├── All hands on deck (oncall war room)                    │
│  ├── Pre-scale to 60% of forecasted peak                   │
│  └── Verify monitoring and runbooks                         │
│                                                              │
│  Event day: MONITOR & REACT                                  │
│  ├── Real-time dashboards                                   │
│  ├── Auto-scaling handles variability                       │
│  ├── Manual intervention only if auto-scaling fails         │
│  └── Instant rollback plans for each service               │
│                                                              │
│  T+1 week: RETROSPECTIVE                                    │
│  ├── What was actual vs forecast?                           │
│  ├── What broke? What almost broke?                         │
│  ├── Update models for next event                           │
│  └── Scale back excess capacity                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Google's Borg Capacity Planning

Google manages capacity across 100+ data centers using **Borg** (precursor to Kubernetes):

```
┌──────────────────────────────────────────────────────────────┐
│         GOOGLE'S CAPACITY PRINCIPLES                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. N+2 Redundancy                                           │
│     Every service has capacity to lose 2 data centers        │
│     and still serve 100% of traffic                          │
│                                                              │
│  2. Bin Packing                                              │
│     Borg optimizes placement to maximize utilization         │
│     across the cluster (avg target: 60-70%)                  │
│                                                              │
│  3. Priority-Based Preemption                                │
│     Low-priority batch jobs fill gaps in capacity            │
│     High-priority services can evict them instantly          │
│                                                              │
│  4. Quota System                                             │
│     Each team has a resource quota (CPU, RAM, disk)          │
│     Quotas are planned annually, adjusted quarterly          │
│                                                              │
│  5. Utilization Targets                                      │
│     Under 40% → Team is over-provisioned (quota reduced)    │
│     40-70%    → Healthy                                      │
│     Over 70%  → Capacity risk (quota increased)             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | What To Do Instead |
|---------|-------------|-------------------|
| **Planning only for average load** | Systems crash during peaks, not averages | Plan for peak × 1.5-2x |
| **Ignoring database growth** | DBs are the hardest to scale quickly | Track storage/IOPS growth weekly |
| **No lead time in planning** | Provisioning takes weeks (RIs, hardware, contracts) | Plan 2-3 months ahead for major changes |
| **Never load testing** | Capacity plan is theoretical until validated | Load test quarterly at 2x expected peak |
| **Ignoring dependent services** | Your service is fine but the DB or queue becomes the bottleneck | Plan ALL components in the request path |
| **Treating peak as constant** | Flash spikes (10x in seconds) need different handling than sustained growth | Plan for burst capacity + auto-scaling speed |
| **Not tracking per-service metrics** | Aggregate cluster metrics hide individual service bottlenecks | Monitor capacity per service and dependency |

---

## When to Use / When NOT to Use

### When to Invest Heavily in Capacity Planning

- ✅ Running at > 50% utilization and growing
- ✅ Before major events (sales, launches, marketing campaigns)
- ✅ Managing databases (hardest to scale, longest lead times)
- ✅ Multi-region deployments (coordination complexity)
- ✅ Financial/compliance systems (downtime has legal consequences)
- ✅ When your cloud bill exceeds $50K/month (optimization ROI is high)

### When Capacity Planning is Less Critical

- ❌ Very early-stage startup (< 1000 users) — just auto-scale
- ❌ Fully serverless architecture (cloud handles capacity)
- ❌ Development/staging environments
- ❌ When auto-scaling works well and you have no hard limits

---

## Key Takeaways

1. **Capacity planning is about PREDICTING demand before it arrives** — not reacting after things break. The best time to plan is when you DON'T have a problem yet.

2. **Target 60-70% utilization at peak** — this gives you headroom for unexpected spikes without wasting money at low traffic.

3. **Databases are always the hardest constraint** — they're the slowest to scale and the most dangerous to change. Plan database capacity 3-6 months ahead.

4. **Use multiple forecasting inputs** — historical trends alone aren't enough. Combine with business plans (campaigns, features) and seasonal patterns.

5. **Validate plans with load tests** — a spreadsheet forecast is just a theory. Load testing at 2x expected peak proves whether your infrastructure can actually handle it.

6. **Build capacity reviews into your rhythm** — monthly reviews catch drift early. Quarterly planning meetings align with business cycles.

7. **Auto-scaling doesn't replace capacity planning** — auto-scaling handles minute-to-minute fluctuations. Capacity planning handles month-to-month growth and structural limits (max instances, DB connections, network bandwidth).

---

## What's Next?

Next, we'll explore **FinOps — Managing Cloud Costs at Scale** — where you'll learn how organizations with $1M+ cloud bills implement financial governance, build cost accountability cultures, and use sophisticated tooling to optimize spending without sacrificing performance.
