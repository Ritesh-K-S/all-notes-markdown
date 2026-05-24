# Predictive Scaling & Scheduled Scaling — Scale Before the Storm

> **What you'll learn**: How machine learning analyzes historical traffic patterns to predict future demand and pre-scale capacity hours ahead, how scheduled scaling handles known events (Black Friday, game launches, TV ads), and how pre-warming ensures new instances are ready to serve traffic the instant they're needed — eliminating the scaling lag that reactive approaches can't avoid.

---

## Real-Life Analogy — Weather Forecasting for Fire Departments

**Reactive scaling** = Sending fire trucks AFTER the fire starts (always late, damage already done).

**Scheduled scaling** = "July 4th always has fireworks fires — schedule extra trucks every year."

**Predictive scaling** = A weather model says "Thursday will be 105°F, dry, windy — historically that means 3x more fires." Deploy extra trucks Wednesday night, BEFORE the fires start.

```
REACTIVE vs SCHEDULED vs PREDICTIVE:

Traffic timeline (24 hours):
                    PEAK
                   ╱    ╲
                  ╱      ╲
                 ╱        ╲
        ────────╱          ╲──────
        ▲                   ▲
        │                   │
    Off-peak              Off-peak

REACTIVE (Chapter 7.2-7.3):
┌─────────────────────────────────────────────────────────┐
│         PEAK hits → alarm → scale → instances ready     │
│         ████████ ← USERS WAITING (3-5 min lag!)        │
│                   ████████████████ capacity catches up  │
│  "We react AFTER the damage starts"                     │
└─────────────────────────────────────────────────────────┘

SCHEDULED (this chapter):
┌─────────────────────────────────────────────────────────┐
│    8 AM: scale up (we KNOW peak is at 9 AM)            │
│    ████████████████████████ capacity ready BEFORE peak  │
│    10 PM: scale down (peak is over)                     │
│  "We know our daily pattern and prepare in advance"     │
└─────────────────────────────────────────────────────────┘

PREDICTIVE (this chapter):
┌─────────────────────────────────────────────────────────┐
│  ML model: "Tomorrow's peak will be 40% higher than     │
│  usual — there's a marketing campaign"                  │
│  Pre-scale to 140% capacity by 7 AM                     │
│  ████████████████████████████████ perfectly sized!      │
│  "We PREDICT the future and prepare automatically"      │
└─────────────────────────────────────────────────────────┘
```

---

## Core Concept Explained Step-by-Step

### Step 1: The Problem with Reactive-Only Scaling

```
WHY REACTIVE SCALING ISN'T ENOUGH:

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  The GAP between "traffic arrives" and "capacity is ready":     │
│                                                                  │
│  Timeline of a traffic spike:                                    │
│                                                                  │
│  00:00 - Traffic begins increasing                               │
│  01:00 - Metrics cross threshold                                 │
│  03:00 - Evaluation period passes (3 min to confirm)            │
│  03:01 - Scaling decision made                                   │
│  03:30 - Instance launching (cloud API call)                    │
│  04:30 - Instance running, app starting                          │
│  05:30 - App warmed up (connections, caches loaded)             │
│  06:00 - Health check passes → serving traffic                  │
│                                                                  │
│  Total: 6 MINUTES of degraded service!                          │
│                                                                  │
│  Users │       ████ degraded experience                         │
│        │      ╱████╲                                            │
│  Traffic│     ╱ ████ ╲                                           │
│        │    ╱  ████  ╲                                          │
│        │───╱   ████   ╲───                                      │
│        └────────────────────── time                              │
│             ↑    ↑                                               │
│          spike  capacity                                         │
│          starts  catches up                                      │
│             │    │                                               │
│             └─6m─┘  (users suffered here!)                      │
│                                                                  │
│  FOR PREDICTABLE PATTERNS, THIS LAG IS UNNECESSARY!             │
│  If you KNOW the spike comes every day at 9 AM...               │
│  ...why wait for it to happen before reacting?                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Step 2: Scheduled Scaling — Known Patterns

```
SCHEDULED SCALING: "Scale at pre-determined times"

WHEN TO USE:
├── Daily patterns: peak at 9 AM, quiet at midnight
├── Weekly patterns: busy weekdays, quiet weekends
├── Known events: Black Friday, product launch, TV commercial
├── Batch jobs: nightly reports at 2 AM need extra compute
└── Geographic peaks: follow-the-sun across time zones

EXAMPLE — E-commerce Daily Pattern:
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Schedule:                                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 6:00 AM  → Set min=4, desired=6   (morning ramp)     │  │
│  │ 8:00 AM  → Set min=8, desired=12  (work hours)       │  │
│  │ 12:00 PM → Set min=10, desired=15 (lunch shopping)    │  │
│  │ 6:00 PM  → Set min=12, desired=18 (evening peak)     │  │
│  │ 10:00 PM → Set min=6, desired=8   (wind down)        │  │
│  │ 1:00 AM  → Set min=2, desired=3   (overnight)        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  Capacity over 24 hours:                                     │
│                                                              │
│  18│           ┌──────┐                                      │
│  15│      ┌────┘      │                                      │
│  12│  ┌───┘           │                                      │
│   8│──┘               └────┐                                │
│   6│                       └──┐                             │
│   3│                          └──────────────┐              │
│   2│                                         └───           │
│    └──────────────────────────────────────────────           │
│    6AM  8AM  12PM  6PM  10PM  1AM  6AM                      │
│                                                              │
│  + Reactive scaling STILL active on top of scheduled!       │
│  Scheduled sets the MINIMUM; reactive handles unexpected.   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Step 3: Predictive Scaling — ML-Powered Forecasting

```
PREDICTIVE SCALING: "Let AI predict your traffic 48 hours ahead"

HOW IT WORKS:
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. LEARN (historical analysis)                                  │
│     ├── Collects 14 days of metric history                      │
│     ├── Identifies patterns: daily, weekly, seasonal            │
│     ├── Learns correlations between metrics and load            │
│     └── Builds a time-series forecasting model                  │
│                                                                  │
│  2. PREDICT (forecast future)                                    │
│     ├── Generates forecast for next 48 hours                    │
│     ├── Calculates required capacity at each point              │
│     └── Updates forecast every 24 hours (or more frequently)    │
│                                                                  │
│  3. PRE-SCALE (act before demand)                               │
│     ├── Launches instances BEFORE predicted spike              │
│     ├── Allows warm-up time before traffic arrives             │
│     └── Instances ready and healthy when peak hits             │
│                                                                  │
│  ┌─────────────────────────────────────────────────────┐       │
│  │                                                     │       │
│  │ Historical:  ╱╲  ╱╲  ╱╲  ╱╲  ╱╲  ╱╲  ╱╲          │       │
│  │             ╱  ╲╱  ╲╱  ╲╱  ╲╱  ╲╱  ╲╱  ╲         │       │
│  │             │  past 14 days  │                      │       │
│  │                              │                      │       │
│  │ Prediction:                  │  ╱╲  ╱╲             │       │
│  │                              │ ╱  ╲╱  ╲            │       │
│  │                              │╱        ╲           │       │
│  │                              │next 48h  │          │       │
│  │                              ▲                      │       │
│  │                              │                      │       │
│  │                          TODAY                      │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                  │
│  THE ML MODEL CAPTURES:                                          │
│  ├── Daily cycles (quiet at night, busy during day)             │
│  ├── Weekly cycles (weekdays vs weekends)                       │
│  ├── Trend (overall growth: 5% more users each week)           │
│  └── Anomalies it's seen before (Monday morning spikes)        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Step 4: Predictive Scaling Modes

```
PREDICTIVE SCALING OPERATING MODES:

MODE 1: FORECAST ONLY (Observe)
┌────────────────────────────────────────────────────────────────┐
│  ML generates predictions but does NOT take action.            │
│  You compare predictions vs actual in CloudWatch.             │
│  Use this for 2 weeks to validate accuracy before enabling.   │
│                                                                │
│  Predicted:  ─ ─ ─ ─ ─ ─ ─                                   │
│  Actual:     ━━━━━━━━━━━━━━                                   │
│  If they align well → enable "Forecast and Scale"             │
└────────────────────────────────────────────────────────────────┘

MODE 2: FORECAST AND SCALE (Active)
┌────────────────────────────────────────────────────────────────┐
│  ML generates predictions AND scales capacity proactively.    │
│  Works alongside reactive scaling (higher of the two wins).   │
│                                                                │
│  Effective capacity = MAX(predictive, reactive)               │
│                                                                │
│  Why MAX? Safety net:                                          │
│  ├── If prediction underestimates → reactive catches it      │
│  ├── If prediction overestimates → extra capacity, no harm   │
│  └── If unexpected spike → reactive still works normally     │
└────────────────────────────────────────────────────────────────┘

HOW PREDICTIVE + REACTIVE WORK TOGETHER:
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Capacity                                                        │
│  20│          ┌─────┐                                           │
│  18│    ╱─────┘     │  ← Predictive pre-scaled                 │
│  15│   ╱            │                                           │
│  12│──╱             └──┐                                        │
│   8│                   │                                        │
│   6│                   └────── ← Predictive scales down        │
│   4│─────                                                       │
│    └──────────────────────────────── time                       │
│                                                                  │
│  What if an UNEXPECTED spike happens on top of predicted load? │
│                                                                  │
│  Capacity                                                        │
│  25│              ┌──┐ ← Reactive adds MORE on top!            │
│  20│          ┌───┘  └───┐                                      │
│  18│    ╱─────┘          │ ← Predictive already pre-scaled    │
│  15│   ╱                 │                                      │
│  12│──╱                  └──                                    │
│    └──────────────────────────── time                           │
│                                                                  │
│  Predictive handled 90% of the load.                            │
│  Reactive handled the unexpected 10% remainder.                 │
│  USERS EXPERIENCED ZERO DEGRADATION!                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Step 5: Pre-Warming — Preparing Instances Before Traffic

```
PRE-WARMING: "Instances that are READY, not just RUNNING"

THE COLD-START PROBLEM:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  A new instance is "running" but NOT "ready":                 │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Instance starts (OS boots)           │  30 seconds   │    │
│  │ Application loads                     │  15 seconds   │    │
│  │ Dependencies connect (DB, cache)     │  10 seconds   │    │
│  │ JIT compilation / warm-up requests    │  30 seconds   │    │
│  │ Cache population                      │  60 seconds   │    │
│  │ Health check passes (3 checks × 10s) │  30 seconds   │    │
│  ├──────────────────────────────────────┼──────────────┤    │
│  │ TOTAL: "running" to "serving"         │  ~3 minutes  │    │
│  └──────────────────────────────────────┴──────────────┘    │
│                                                                │
│  DURING THOSE 3 MINUTES:                                      │
│  • Instance is BILLED but NOT USEFUL                          │
│  • First requests hit a cold cache → slow responses           │
│  • Connection pools are empty → latency spikes               │
│  • JVM hasn't JIT-compiled hot paths → slow execution        │
│                                                                │
└────────────────────────────────────────────────────────────────┘

PRE-WARMING STRATEGIES:
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Strategy 1: Warm Pool (AWS)                                    │
│  ├── Instances pre-initialized, then STOPPED                   │
│  ├── When needed: START (10-30 seconds vs 3 minutes)           │
│  ├── Not billed for compute while stopped                      │
│  └── Still billed for EBS storage (cheap)                      │
│                                                                  │
│  Strategy 2: Lifecycle Hooks                                    │
│  ├── Instance launches but pauses before joining LB            │
│  ├── Custom script runs: populate cache, warm connections      │
│  ├── Script sends "ready" signal → instance joins LB          │
│  └── Ensures first real request hits a warm instance           │
│                                                                  │
│  Strategy 3: Synthetic Traffic                                  │
│  ├── After launch, send fake requests to warm JIT/caches      │
│  ├── Hit all major API endpoints with realistic data           │
│  ├── Prime connection pools to DB, Redis, external APIs        │
│  └── Only then register with load balancer                     │
│                                                                  │
│  Strategy 4: Over-Provisioning                                  │
│  ├── Always keep 1-2 extra instances beyond current need       │
│  ├── "Buffer" instances absorb spikes instantly                │
│  ├── Simple but costs money (paying for idle capacity)         │
│  └── Math: 2 extra × $0.10/hr = $144/month for instant scale │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Step 6: Handling Known Events — Event-Based Scaling

```
EVENT-BASED SCALING: "I KNOW something big is coming"

EXAMPLES OF KNOWN EVENTS:
├── Marketing: TV commercial airs at 8:05 PM → traffic spike in 60 seconds
├── Product: New feature launch Tuesday at 10 AM
├── Cultural: Super Bowl halftime → tweet storm → 50x API calls
├── Gaming: New season drops at midnight → 100x login attempts
├── Retail: Black Friday at midnight → 20x normal traffic
└── Streaming: New show drops Friday at midnight → 5x concurrent streams

THE STRATEGY:
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  BEFORE EVENT:                                                   │
│  ├── T-24h: Increase max capacity (remove ceiling)             │
│  ├── T-4h:  Pre-scale to predicted peak capacity               │
│  ├── T-2h:  Warm all instances (connections, caches)           │
│  ├── T-1h:  Verify health of all instances                     │
│  ├── T-30m: Lower scaling thresholds (more sensitive)          │
│  └── T-0:   Event begins, capacity is ready!                   │
│                                                                  │
│  DURING EVENT:                                                   │
│  ├── Monitor closely (war room)                                │
│  ├── Reactive scaling handles unexpected excess                │
│  ├── Manual override button available if needed                │
│  └── Disable non-critical features if extreme load             │
│                                                                  │
│  AFTER EVENT:                                                    │
│  ├── T+1h:  Observe traffic returning to normal               │
│  ├── T+2h:  Begin gradual scale-down (not aggressive!)        │
│  ├── T+6h:  Restore normal scaling thresholds                 │
│  └── T+24h: Full post-mortem, tune for next time              │
│                                                                  │
│  CAPACITY TIMELINE:                                              │
│  25│    ┌──────────────────────┐                                │
│  20│    │    PRE-SCALED        │                                │
│  15│    │    CAPACITY          │                                │
│  10│────┘                      └─────┐                          │
│   5│                                 └─────── normal            │
│    └────┬──────────┬───────────┬─────────                       │
│      -4h    EVENT START     +2h     +6h                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### AWS Predictive Scaling — The ML Pipeline

```
AWS PREDICTIVE SCALING INTERNALS:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  DATA COLLECTION (Continuous):                                       │
│  ├── CloudWatch metrics for ASG (CPU, network, ALB requests)       │
│  ├── 14 days minimum history required                               │
│  ├── Collects at 1-hour granularity                                 │
│  └── Stores in internal ML training dataset                         │
│                                                                      │
│  MODEL TRAINING (Daily):                                            │
│  ├── Time-series decomposition:                                     │
│  │   ├── Trend component (long-term growth/decline)                │
│  │   ├── Daily seasonality (24-hour cycle)                         │
│  │   ├── Weekly seasonality (7-day cycle)                          │
│  │   └── Residual (unexplained variance)                           │
│  ├── Algorithm: similar to Facebook Prophet / exponential smoothing │
│  └── Handles missing data, outliers automatically                  │
│                                                                      │
│  FORECASTING (Every few hours):                                     │
│  ├── Generates point forecast for next 48 hours                    │
│  ├── Adds buffer: forecast × (1 + buffer_percentage)               │
│  │   Default buffer: 0% (can set to 10-50% for safety)            │
│  └── Converts metric forecast → required instance count            │
│                                                                      │
│  CAPACITY PLANNING (Translation):                                   │
│  ├── "If CPU target is 50% and predicted CPU load is 400%..."     │
│  ├── "...then I need 400%/50% = 8 instances"                      │
│  ├── Rounds UP (never under-provision)                             │
│  └── Respects ASG min/max boundaries                               │
│                                                                      │
│  EXECUTION:                                                          │
│  ├── Schedules capacity changes at exact timestamps                │
│  ├── Pre-launches instances ahead of predicted need                │
│  ├── Lead time: configurable (default varies by provider)          │
│  └── If reactive wants MORE → reactive wins (MAX behavior)        │
│                                                                      │
│  VISUALIZATION:                                                      │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  Predicted │    ┌─╲                                  │          │
│  │  Load      │   ╱   ╲──╲                              │          │
│  │            │  ╱        ╲──╲                           │          │
│  │            │─╱             ╲──                        │          │
│  │            └───────────────────────── time            │          │
│  │                                                      │          │
│  │  Planned   │    ┌───────┐                            │          │
│  │  Capacity  │   ╱│buffer │╲                           │          │
│  │            │──╱ └───────┘ ╲──                        │          │
│  │            └───────────────────────── time            │          │
│  │                  ↑                                   │          │
│  │            capacity includes safety buffer           │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Scheduled vs Predictive — When Each Triggers

```
INTERACTION BETWEEN SCHEDULED AND PREDICTIVE:

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  You can use BOTH on the same ASG:                              │
│                                                                  │
│  Scheduled: "Every weekday, min=10 from 8 AM to 6 PM"          │
│  Predictive: "ML predicts actual load patterns"                  │
│  Reactive: "CPU > 60% → add more"                               │
│                                                                  │
│  Effective capacity = MAX(scheduled, predictive, reactive)       │
│                                                                  │
│  Example scenario:                                               │
│  ┌──────────┬──────────┬───────────┬──────────────────────┐    │
│  │ Time     │ Scheduled│ Predictive│ Reactive │ EFFECTIVE  │    │
│  ├──────────┼──────────┼───────────┼──────────┼───────────┤    │
│  │ 2 AM     │ min=2    │ 3         │ 2        │ 3         │    │
│  │ 8 AM     │ min=10   │ 12        │ 8        │ 12        │    │
│  │ 12 PM    │ min=10   │ 15        │ 14       │ 15        │    │
│  │ 3 PM     │ min=10   │ 11        │ 16 (!)   │ 16        │    │
│  │ 10 PM    │ min=2    │ 5         │ 4        │ 5         │    │
│  └──────────┴──────────┴───────────┴──────────┴───────────┘    │
│                                                                  │
│  At 3 PM: Unexpected spike! Reactive > Predictive → reactive   │
│  wins. Predictive was wrong about this specific moment,         │
│  but reactive catches it. Belt AND suspenders!                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Predictive Scaling Configuration with Boto3

```python
# predictive_scaling.py — Configure AWS Predictive Scaling
import boto3
from datetime import datetime, timedelta

autoscaling = boto3.client('autoscaling')

# Step 1: Create predictive scaling policy
autoscaling.put_scaling_policy(
    AutoScalingGroupName='web-production',
    PolicyName='predictive-cpu-scaling',
    PolicyType='PredictiveScaling',
    PredictiveScalingConfiguration={
        # Which metric to predict and target
        'MetricSpecifications': [{
            'TargetValue': 50.0,  # Target 50% CPU
            'PredefinedMetricPairSpecification': {
                'PredefinedMetricType': 'ASGCPUUtilization',
            },
            # Or use custom metrics:
            # 'CustomizedLoadMetricSpecification': {
            #     'MetricDataQueries': [{
            #         'Id': 'load',
            #         'MetricStat': {...},
            #     }]
            # },
        }],
        # Mode: ForecastOnly (observe) or ForecastAndScale (active)
        'Mode': 'ForecastAndScale',
        # How far ahead to schedule capacity
        'SchedulingBufferTime': 300,  # 5 minutes before predicted need
        # Safety buffer above prediction
        'MaxCapacityBreachBehavior': 'HonorMaxCapacity',
        # Buffer above prediction (0 = exact, add 10-20% for safety)
        'MaxCapacityBuffer': 10,  # 10% buffer above predicted need
    }
)
print("Predictive scaling enabled! ML will learn traffic patterns over 14 days.")

# Step 2: Create scheduled scaling for known events
autoscaling.put_scheduled_update_group_action(
    AutoScalingGroupName='web-production',
    ScheduledActionName='weekday-morning-ramp',
    Recurrence='0 7 * * MON-FRI',  # 7 AM weekdays (cron format)
    MinSize=8,
    DesiredCapacity=12,
    MaxSize=50,
)

autoscaling.put_scheduled_update_group_action(
    AutoScalingGroupName='web-production',
    ScheduledActionName='evening-peak',
    Recurrence='0 17 * * MON-FRI',  # 5 PM weekdays
    MinSize=15,
    DesiredCapacity=20,
    MaxSize=50,
)

autoscaling.put_scheduled_update_group_action(
    AutoScalingGroupName='web-production',
    ScheduledActionName='overnight-scale-down',
    Recurrence='0 23 * * *',  # 11 PM every day
    MinSize=2,
    DesiredCapacity=4,
    MaxSize=50,
)

# Step 3: One-time event scaling (e.g., product launch)
launch_date = datetime(2024, 3, 15, 8, 0, 0)  # March 15 at 8 AM
autoscaling.put_scheduled_update_group_action(
    AutoScalingGroupName='web-production',
    ScheduledActionName='product-launch-pre-scale',
    StartTime=launch_date - timedelta(hours=2),  # 2 hours before
    MinSize=30,
    DesiredCapacity=40,
    MaxSize=100,  # Remove ceiling for the event
)
print(f"Pre-scaling to 40 instances at {launch_date - timedelta(hours=2)}")
```

### Java — Custom Pre-Warming with Lifecycle Hooks

```java
// InstancePreWarmer.java — Warm up new instances before serving traffic
import software.amazon.awssdk.services.autoscaling.AutoScalingClient;
import software.amazon.awssdk.services.autoscaling.model.*;
import java.net.http.*;
import java.net.URI;
import java.time.Duration;
import java.util.List;

public class InstancePreWarmer {
    
    private final AutoScalingClient autoScaling;
    private final HttpClient httpClient;
    private final List<String> warmupEndpoints;
    
    public InstancePreWarmer() {
        this.autoScaling = AutoScalingClient.create();
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .build();
        // Endpoints to hit for warming caches/JIT
        this.warmupEndpoints = List.of(
            "/api/health",
            "/api/products?category=popular",
            "/api/users/profile",
            "/api/search?q=test",
            "/api/recommendations"
        );
    }
    
    /**
     * Called by lifecycle hook when a new instance launches.
     * Warms up the instance before allowing traffic.
     */
    public void warmInstance(String instanceId, String hookName, String token) {
        String instanceIp = getInstancePrivateIp(instanceId);
        System.out.printf("Warming instance %s at %s%n", instanceId, instanceIp);
        
        try {
            // Wait for application to be reachable
            waitForApp(instanceIp, 120);  // Max 2 minutes
            
            // Send synthetic requests to warm caches and JIT
            for (int round = 0; round < 3; round++) {
                for (String endpoint : warmupEndpoints) {
                    HttpRequest request = HttpRequest.newBuilder()
                        .uri(URI.create("http://" + instanceIp + ":8080" + endpoint))
                        .header("X-Warmup", "true")  // App can log differently
                        .GET().build();
                    httpClient.send(request, HttpResponse.BodyHandlers.discarding());
                }
            }
            System.out.printf("Instance %s warmed successfully!%n", instanceId);
            
            // Tell ASG: "This instance is ready to receive real traffic"
            autoScaling.completeLifecycleAction(CompleteLifecycleActionRequest.builder()
                .autoScalingGroupName("web-production")
                .lifecycleHookName(hookName)
                .lifecycleActionToken(token)
                .lifecycleActionResult("CONTINUE")  // Ready!
                .build());
                
        } catch (Exception e) {
            System.err.printf("Warmup failed for %s: %s%n", instanceId, e.getMessage());
            // ABANDON = terminate this instance, try a new one
            autoScaling.completeLifecycleAction(CompleteLifecycleActionRequest.builder()
                .autoScalingGroupName("web-production")
                .lifecycleHookName(hookName)
                .lifecycleActionToken(token)
                .lifecycleActionResult("ABANDON")
                .build());
        }
    }
    
    private void waitForApp(String ip, int maxSeconds) throws Exception {
        long deadline = System.currentTimeMillis() + (maxSeconds * 1000L);
        while (System.currentTimeMillis() < deadline) {
            try {
                HttpRequest req = HttpRequest.newBuilder()
                    .uri(URI.create("http://" + ip + ":8080/api/health"))
                    .timeout(Duration.ofSeconds(2)).GET().build();
                HttpResponse<String> resp = httpClient.send(req, 
                    HttpResponse.BodyHandlers.ofString());
                if (resp.statusCode() == 200) return;  // App is up!
            } catch (Exception ignored) { /* Not ready yet */ }
            Thread.sleep(2000);  // Retry every 2 seconds
        }
        throw new RuntimeException("App did not become healthy within " + maxSeconds + "s");
    }
    
    private String getInstancePrivateIp(String instanceId) {
        // Query EC2 for instance private IP (implementation omitted for brevity)
        return "10.0.1.42";  // Placeholder
    }
}
```

---

## Infrastructure Examples

### AWS — Complete Predictive + Scheduled + Reactive Setup

```hcl
# predictive-scaling.tf — Three layers of scaling for production

# Layer 1: Predictive Scaling (ML-based, 48h forecast)
resource "aws_autoscaling_policy" "predictive" {
  name                   = "predictive-scaling"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "PredictiveScaling"

  predictive_scaling_configuration {
    mode                          = "ForecastAndScale"
    scheduling_buffer_time        = 300  # 5 min pre-launch buffer
    max_capacity_breach_behavior  = "HonorMaxCapacity"
    max_capacity_buffer           = 10   # 10% safety buffer

    metric_specification {
      target_value = 50.0  # Keep CPU at 50%
      predefined_metric_pair_specification {
        predefined_metric_type = "ASGCPUUtilization"
      }
    }
  }
}

# Layer 2: Scheduled Scaling (known patterns)
resource "aws_autoscaling_schedule" "morning_scale_up" {
  scheduled_action_name  = "weekday-morning"
  autoscaling_group_name = aws_autoscaling_group.web.name
  recurrence             = "0 7 * * MON-FRI"
  min_size               = 8
  desired_capacity       = 12
  max_size               = 50
}

resource "aws_autoscaling_schedule" "evening_peak" {
  scheduled_action_name  = "evening-peak"
  autoscaling_group_name = aws_autoscaling_group.web.name
  recurrence             = "0 17 * * MON-FRI"
  min_size               = 15
  desired_capacity       = 20
  max_size               = 50
}

resource "aws_autoscaling_schedule" "overnight" {
  scheduled_action_name  = "overnight"
  autoscaling_group_name = aws_autoscaling_group.web.name
  recurrence             = "0 23 * * *"
  min_size               = 2
  desired_capacity       = 4
  max_size               = 50
}

# Layer 3: Reactive Scaling (catches unexpected spikes)
resource "aws_autoscaling_policy" "reactive_cpu" {
  name                   = "reactive-cpu"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value       = 60.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# Pre-warming: Lifecycle hook for warm-up
resource "aws_autoscaling_lifecycle_hook" "warmup" {
  name                   = "instance-warmup"
  autoscaling_group_name = aws_autoscaling_group.web.name
  lifecycle_transition   = "autoscaling:EC2_INSTANCE_LAUNCHING"
  heartbeat_timeout      = 300  # 5 min max for warmup
  default_result         = "CONTINUE"  # If no response, allow anyway

  notification_target_arn = aws_sns_topic.scaling_events.arn
  role_arn                = aws_iam_role.lifecycle_hook.arn
}

# Warm pool for instant scaling
resource "aws_autoscaling_group" "web" {
  # ... (other config from Chapter 7.5) ...

  warm_pool {
    pool_state                  = "Stopped"
    min_size                    = 4
    max_group_prepared_capacity = 10
    instance_reuse_policy {
      reuse_on_scale_in = true
    }
  }
}
```

### Kubernetes — Scheduled HPA with CronJobs

```yaml
# scheduled-hpa.yaml — Scale Kubernetes deployments on schedule
# Using kubernetes-cronhpa-controller

apiVersion: autoscaling.alibabacloud.com/v1beta1
kind: CronHorizontalPodAutoscaler
metadata:
  name: web-scheduled-scaling
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-server
  jobs:
  - name: morning-scale-up
    schedule: "0 7 * * MON-FRI"
    targetSize: 10
    runOnce: false
  - name: evening-peak
    schedule: "0 17 * * MON-FRI"  
    targetSize: 20
    runOnce: false
  - name: overnight-scale-down
    schedule: "0 23 * * *"
    targetSize: 3
    runOnce: false
  - name: black-friday-pre-scale
    schedule: "0 20 28 11 *"  # Nov 28 at 8 PM (before midnight sales)
    targetSize: 50
    runOnce: true  # Only this year!
---
# Regular HPA still active for unexpected spikes
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-reactive-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-server
  minReplicas: 3   # Will be overridden by scheduled scaling above
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

## Real-World Example

### How Disney+ Handles Marvel Movie Premiere Night

```
DISNEY+ PREMIERE NIGHT SCALING:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  SCENARIO: "Avengers: New Movie" drops at midnight Pacific          │
│  Expected: 20-50x normal concurrent streams in first 10 minutes    │
│                                                                      │
│  THE SCALING PLAN (T = premiere time, midnight):                    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ T-7 days: PLANNING                                        │      │
│  │ ├── Analyze previous premieres (Mandalorian, Loki data)  │      │
│  │ ├── Model: "Avengers will be 3x bigger than Loki"        │      │
│  │ ├── Set max capacity: 100x normal (no ceiling)           │      │
│  │ └── Reserve cloud capacity with provider (guaranteed)     │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ T-24h: PRE-STAGING                                        │      │
│  │ ├── Deploy latest code to all regions                     │      │
│  │ ├── Pre-warm CDN (video segments pre-cached at edge)      │      │
│  │ ├── Scale database read replicas: 5 → 20                 │      │
│  │ └── Scale Redis clusters: 3 nodes → 12 nodes            │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ T-4h: PRE-SCALE                                           │      │
│  │ ├── API servers: 50 → 500 instances (10x)                │      │
│  │ ├── Video streaming: 100 → 1000 instances                │      │
│  │ ├── Auth service: 20 → 200 instances                     │      │
│  │ ├── Run synthetic load test at 50% expected peak         │      │
│  │ └── Verify all instances healthy and warm                 │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ T-1h: FINAL PREPARATION                                   │      │
│  │ ├── Engineering war room activated (50+ engineers)        │      │
│  │ ├── Disable non-critical features (recommendations off)  │      │
│  │ ├── Enable "premiere mode" (simplified UI, less JS)      │      │
│  │ ├── Set reactive scaling thresholds very low (20% CPU)   │      │
│  │ └── All hands on deck with manual override access        │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ T=0: PREMIERE!                                            │      │
│  │ ├── 15M concurrent streams in first 5 minutes!           │      │
│  │ ├── Pre-scaled capacity handles 90% of load              │      │
│  │ ├── Reactive scaling catches remaining 10%               │      │
│  │ ├── CDN serves 85% of traffic (pre-warmed)              │      │
│  │ └── P99 latency stays below 200ms (success!)            │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ T+3h: GRADUAL SCALE-DOWN                                 │      │
│  │ ├── Initial binge viewers settle into sustained watching  │      │
│  │ ├── Traffic stabilizes at 5x normal (still high!)        │      │
│  │ ├── Scale down from 1000 → 300 streaming instances       │      │
│  │ └── Over next 48h: gradual return to normal              │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  RESULT: Zero buffering, zero downtime, 99.99% availability        │
│  COST: Massive for 4 hours, but revenue per subscriber × 15M      │
│         makes it insignificant.                                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Enabling predictive scaling without "Forecast Only" first | ML model might be inaccurate for your workload initially | Run in "Forecast Only" mode for 2 weeks, validate predictions, then enable |
| Not enough history (< 14 days) | ML can't find patterns without data | Wait for 14+ days of metric history before enabling predictive |
| Scheduled scaling without reactive backup | If actual traffic exceeds scheduled capacity, no safety net | ALWAYS keep reactive policies active alongside scheduled |
| Pre-scaling too late for events | Even with pre-scale, instances need time to warm up | Pre-scale at least 30 minutes before event; 2-4 hours for large events |
| Not accounting for warmup in predictions | Predictive scaling launches at predicted time, but instance needs 3-5 min | Set scheduling_buffer_time (AWS) to account for boot + warmup |
| Over-relying on predictions for irregular traffic | ML predicts recurring patterns, not one-off viral events | Predictive for daily patterns + reactive for anomalies + scheduled for known events |
| Scaling down too fast after events | Immediate aftermath often has secondary spikes (social media buzz) | Scale down gradually over hours, not minutes |

---

## When to Use / When NOT to Use

```
DECISION GUIDE:

USE PREDICTIVE SCALING WHEN:
├── Traffic has clear daily/weekly patterns (most web apps do!)
├── You have 14+ days of metric history
├── Reactive scaling's 3-5 min lag causes user impact
├── Cost optimization is important (right-size capacity)
└── Running in AWS (best ML support) or have custom implementation

USE SCHEDULED SCALING WHEN:
├── You have KNOWN events (launches, campaigns, sales)
├── Traffic patterns are predictable by time of day
├── You want explicit control over minimum capacity
├── Operating across time zones (follow-the-sun)
└── ANY cloud provider (all support cron-based schedules)

USE PRE-WARMING WHEN:
├── Application cold-start time > 60 seconds
├── First requests to new instances are slow (cold cache)
├── JVM-based apps that need JIT warmup
├── Applications with connection pool establishment time
└── Critical SLA requirements (every request must be fast)

DO NOT USE PREDICTIVE SCALING WHEN:
├── Traffic is truly random with no pattern
├── Fewer than 14 days of history
├── Workload changes fundamentally week to week
├── Single-digit instances (noise overwhelms signal)
└── Cost of over-provisioning is negligible (just over-provision)
```

---

## Key Takeaways

1. **Reactive scaling alone has a 3-5 minute lag** — users suffer during this gap. Predictive and scheduled scaling eliminate this by having capacity ready BEFORE demand arrives.

2. **Predictive scaling uses ML** to analyze 14 days of history and forecast the next 48 hours. It automatically pre-scales capacity to match predicted demand.

3. **Scheduled scaling is for KNOWN patterns** — daily traffic curves, weekly cycles, and planned events. Set minimum capacity at specific times using cron expressions.

4. **The three layers work together**: Effective capacity = MAX(predictive, scheduled, reactive). If ANY layer says "scale up," it scales up. This provides defense-in-depth.

5. **Pre-warming is critical** — a "running" instance isn't a "ready" instance. Use lifecycle hooks, synthetic traffic, and warm pools to ensure new instances serve fast responses from their first real request.

6. **For major events, pre-scale aggressively** — it's far cheaper to have 20% over-capacity for 4 hours than to lose customers during a once-a-year event.

7. **Always validate predictions before activating** — run in "Forecast Only" mode for 2 weeks, compare predictions to actual, then enable auto-action once you trust the model.

---

## What's Next?

Congratulations! You've completed **Part 7: Auto Scaling & Elasticity**. You now understand how systems scale from a single server to planet-scale infrastructure — vertically, horizontally, reactively, predictively, and across multiple clouds.

In **Part 8: Databases & Storage**, we'll explore how to scale the hardest part of any system: the data layer. Databases can't simply be duplicated like stateless application servers — they require replication, sharding, consistency models, and careful architecture to handle millions of reads and writes per second.
