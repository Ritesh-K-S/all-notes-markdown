# Auto Scaling — Let the System Scale Itself

> **What you'll learn**: How auto scaling automatically adds servers when traffic increases and removes them when traffic decreases, the feedback loop that makes it work, cooldown periods, scaling groups, and the infrastructure components that make hands-free scaling possible.

---

## Real-Life Analogy — The Smart Parking Lot

Imagine you manage a parking garage for a shopping mall:

**Without auto scaling (manual):**
- You built 100 parking spots
- Black Friday: 500 cars show up → 400 cars drive away angry (lost customers!)
- Tuesday at 2 AM: 3 cars parked → 97 spots wasted (paying for empty space!)
- You must GUESS demand weeks in advance

**With auto scaling (smart):**
- You have a magical expanding parking lot
- More cars arriving? The lot GROWS automatically — new levels appear
- Cars leaving? The lot SHRINKS — empty levels disappear
- You only pay for the spots that are actually being used
- A sensor (metrics) tells the system when to grow or shrink

```
WITHOUT AUTO SCALING:                WITH AUTO SCALING:

Traffic spike at 2 PM:               Traffic spike at 2 PM:
┌─────────────────────┐              ┌─────────────────────┐
│  Server capacity:   │              │  Servers: 3 → 8     │
│  ████████████ FULL! │              │  █████████          │
│  Users waiting...   │              │  (auto-added 5 more)│
│  Some get errors    │              │  All users served!  │
└─────────────────────┘              └─────────────────────┘

Traffic drops at 11 PM:              Traffic drops at 11 PM:
┌─────────────────────┐              ┌─────────────────────┐
│  Server capacity:   │              │  Servers: 8 → 2     │
│  ██                 │              │  ██                  │
│  Paying for idle!   │              │  (removed 6 servers)│
│  Wasting money 💰   │              │  Only pay for 2! 💰 │
└─────────────────────┘              └─────────────────────┘
```

---

## Core Concept Explained Step-by-Step

### Step 1: The Auto Scaling Feedback Loop

Auto scaling is a **closed-loop control system** — it monitors, decides, and acts:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  AUTO SCALING FEEDBACK LOOP                           │
│                                                                     │
│                    ┌──────────────┐                                 │
│              ┌────▶│   MONITOR    │─────┐                           │
│              │     │  (metrics)   │     │                           │
│              │     └──────────────┘     │                           │
│              │                          ▼                           │
│     ┌────────┴─────┐          ┌──────────────┐                     │
│     │    EXECUTE   │          │    DECIDE    │                     │
│     │  (add/remove │◀─────────│  (compare to │                     │
│     │   servers)   │          │  thresholds) │                     │
│     └──────────────┘          └──────────────┘                     │
│                                                                     │
│  1. MONITOR: Collect CPU, memory, request count, latency            │
│  2. DECIDE:  "CPU > 70% for 3 minutes? → SCALE OUT"               │
│  3. EXECUTE: Launch new server instances                            │
│  4. MONITOR: Check if the problem is resolved                      │
│  5. Repeat continuously (every 30-60 seconds)                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 2: Auto Scaling Group — The Core Component

```
AUTO SCALING GROUP (ASG):
A group of identical server instances that grow and shrink together.

Configuration:
┌──────────────────────────────────────────────────────────────┐
│  Auto Scaling Group: "web-servers"                            │
│                                                              │
│  ┌──────────────────────────────────────┐                   │
│  │  Launch Template:                     │                   │
│  │  ├── Instance type: t3.xlarge         │                   │
│  │  ├── AMI: ami-myapp-v2.3             │                   │
│  │  ├── Security groups: [sg-web]       │                   │
│  │  ├── User data: startup-script.sh    │                   │
│  │  └── IAM role: web-server-role       │                   │
│  └──────────────────────────────────────┘                   │
│                                                              │
│  ┌──────────────────────────────────────┐                   │
│  │  Scaling Parameters:                  │                   │
│  │  ├── Minimum instances:  2  (always) │                   │
│  │  ├── Desired instances:  4  (normal) │                   │
│  │  ├── Maximum instances: 20  (peak)   │                   │
│  │  └── Health check grace: 300s        │                   │
│  └──────────────────────────────────────┘                   │
│                                                              │
│  Current State (3 PM, normal traffic):                      │
│  ┌──┐ ┌──┐ ┌──┐ ┌──┐                                      │
│  │S1│ │S2│ │S3│ │S4│  ← 4 instances (desired count)       │
│  └──┘ └──┘ └──┘ └──┘                                      │
│                                                              │
│  During Traffic Spike:                                       │
│  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐                │
│  │S1│ │S2│ │S3│ │S4│ │S5│ │S6│ │S7│ │S8│  ← 8 instances │
│  └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘  (scaled out!)  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Step 3: The Lifecycle of an Auto Scaling Event

```
SCALE-OUT EVENT (adding servers):

Time 0:00 - Traffic begins increasing
├── Current: 4 instances, CPU avg = 45%
│
Time 0:30 - Metrics alarm triggers
├── CPU avg = 75% (above 70% threshold!)
├── But wait... evaluation period = 3 minutes
│
Time 3:00 - Evaluation period passes, still high
├── CPU avg = 78% → ALARM STATE!
├── Auto scaler decides: "Add 2 instances"
│
Time 3:01 - New instances launching
├── Cloud provider allocating resources
├── VM booting, OS starting
│
Time 3:45 - Instances running
├── Application starting, warming up
├── Health checks not yet passing
│
Time 4:15 - Health checks pass
├── Load balancer begins sending traffic
├── New instances receiving requests
│
Time 4:30 - Traffic redistributed
├── Now 6 instances, CPU avg = 50%
├── System is healthy again!
│
Time 4:30+ - COOLDOWN PERIOD starts (300 seconds)
├── No more scaling decisions for 5 minutes
├── Prevents oscillation/flapping
└── After cooldown: resume monitoring

TOTAL TIME: ~4.5 minutes from spike to relief
```

### Step 4: Scale-In (Removing Servers)

```
SCALE-IN EVENT (removing servers):

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Current: 8 instances, traffic has decreased                │
│  CPU avg: 25% (below 30% threshold for 10 minutes)        │
│                                                             │
│  Auto scaler decides: "Remove 2 instances"                 │
│                                                             │
│  WHICH servers to terminate?                               │
│  Policies:                                                  │
│  ├── Oldest instance first (replace aging hardware)        │
│  ├── Newest instance first (preserve stable ones)          │
│  ├── Closest to billing hour (save money)                  │
│  └── Instance with least connections (minimize disruption) │
│                                                             │
│  GRACEFUL SHUTDOWN:                                         │
│  1. Remove instance from load balancer                     │
│  2. Wait for in-flight requests to complete (drain)        │
│  3. Send SIGTERM to application                            │
│  4. Wait for graceful shutdown (30-90 seconds)             │
│  5. Terminate instance                                      │
│                                                             │
│  Before: ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐       │
│          │S1│ │S2│ │S3│ │S4│ │S5│ │S6│ │S7│ │S8│       │
│          └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘       │
│                                                             │
│  After:  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐                  │
│          │S1│ │S2│ │S3│ │S4│ │S5│ │S6│                  │
│          └──┘ └──┘ └──┘ └──┘ └──┘ └──┘                  │
│                                              (S7, S8 gone) │
└─────────────────────────────────────────────────────────────┘
```

### Step 5: Cooldown Periods — Preventing Oscillation

```
WHY COOLDOWN MATTERS:

WITHOUT cooldown (BAD):
Time 0: CPU high → scale out (+2 servers)
Time 1: CPU drops (new servers helping) → scale in (-2 servers)
Time 2: CPU high again (removed too soon!) → scale out (+2)
Time 3: CPU drops → scale in (-2)
...forever oscillating! Instances launching and dying constantly.

┌────────────────────────────────────────────────────────────┐
│  Instances ──▶  ^^^^  oscillating = waste + instability!   │
│                ╱    ╲╱    ╲╱    ╲                          │
│  CPU ─────▶  ─────────────────────                        │
└────────────────────────────────────────────────────────────┘

WITH cooldown (GOOD):
Time 0: CPU high → scale out (+2 servers)
Time 1: COOLDOWN (5 min) — no decisions allowed
Time 5: Cooldown ends. CPU is stable at 50%. Do nothing.
Result: Stable! New servers had time to absorb load.

┌────────────────────────────────────────────────────────────┐
│  Instances ──▶  ┌───────── stable ─────────────           │
│                 │                                           │
│  CPU ─────▶  ───┘  drops and stays low                     │
│              ↑                                             │
│           scale-out                                        │
│           + cooldown                                       │
└────────────────────────────────────────────────────────────┘

TYPICAL COOLDOWN VALUES:
├── Scale-out cooldown: 60-180 seconds (act fast on spikes)
├── Scale-in cooldown: 300-600 seconds (be conservative removing)
└── Why longer for scale-in? Removing too aggressively = worse than keeping extra
```

---

## How It Works Internally

### The Auto Scaler Decision Engine

```
INTERNAL DECISION PROCESS:

Every 30 seconds, the auto scaler runs this loop:

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. COLLECT METRICS                                             │
│     ├── Pull from CloudWatch/Prometheus/monitoring system       │
│     ├── Aggregate across all instances (avg, max, min)         │
│     └── Current: avg_cpu=75%, request_count=5000/min           │
│                                                                 │
│  2. EVALUATE ALARMS                                             │
│     ├── Is avg_cpu > 70%? → YES                               │
│     ├── Has it been > 70% for 3+ minutes? → YES               │
│     └── ALARM STATE TRIGGERED!                                  │
│                                                                 │
│  3. CHECK COOLDOWN                                              │
│     ├── Is cooldown period active? → NO (last scale was 10m ago)│
│     └── Proceed with scaling action                             │
│                                                                 │
│  4. CALCULATE DESIRED CAPACITY                                  │
│     ├── Policy: "Add 2 instances" (simple scaling)             │
│     │   OR                                                      │
│     ├── Policy: "Scale to maintain 50% CPU" (target tracking)  │
│     │   Current: 4 instances @ 75% CPU                         │
│     │   Needed: 4 × (75/50) = 6 instances                     │
│     └── New desired = 6 (but max=20, so OK)                    │
│                                                                 │
│  5. EXECUTE                                                     │
│     ├── Call cloud API: "Launch 2 new instances"               │
│     ├── Use launch template for configuration                  │
│     └── Register with load balancer target group               │
│                                                                 │
│  6. START COOLDOWN                                              │
│     └── No more scaling for next 180 seconds                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Instance Lifecycle States

```
INSTANCE LIFECYCLE IN AUTO SCALING:

┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Pending  │───▶│InService │───▶│Terminating│───▶│Terminated│
│          │    │          │    │           │    │          │
│ Booting, │    │ Healthy, │    │ Draining, │    │ Gone,    │
│ starting │    │ receiving│    │ shutting  │    │ resources│
│ app      │    │ traffic  │    │ down      │    │ freed    │
└──────────┘    └──────────┘    └───────────┘    └──────────┘
     │                │
     │                │          ┌──────────┐
     │                └─────────▶│Standby   │ (maintenance)
     │                           └──────────┘
     │
     └── Lifecycle hooks can pause here
         (for custom initialization, pulling config, etc.)
```

### Target Tracking vs Step Scaling

```
TWO MAIN SCALING APPROACHES:

TARGET TRACKING (Recommended — Simpler):
"Keep average CPU at 50%"
┌──────────────────────────────────────────────────────────────┐
│  You say: "Target CPU = 50%"                                 │
│  Auto scaler figures out how many instances needed.          │
│                                                              │
│  CPU at 75%? → Add instances until CPU ≈ 50%               │
│  CPU at 30%? → Remove instances until CPU ≈ 50%            │
│                                                              │
│  Like a thermostat: "Keep temperature at 72°F"              │
│  The system does the math.                                   │
└──────────────────────────────────────────────────────────────┘

STEP SCALING (More control — Complex):
"At X% CPU, add Y instances. At Z% CPU, add W instances."
┌──────────────────────────────────────────────────────────────┐
│  You define multiple steps:                                  │
│                                                              │
│  CPU 60-70%: Add 1 instance     (gentle response)          │
│  CPU 70-80%: Add 2 instances    (moderate response)         │
│  CPU 80-90%: Add 4 instances    (aggressive response)       │
│  CPU > 90%:  Add 8 instances    (emergency response!)       │
│                                                              │
│  Like stair steps — bigger response at higher load.         │
│                                                              │
│  Instances                                                   │
│  added: 8 │              ┌─────                             │
│         4 │         ┌────┘                                  │
│         2 │    ┌────┘                                       │
│         1 │ ┌──┘                                            │
│           └──────────────────────── CPU%                    │
│             60  70  80  90                                   │
└──────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Auto Scaling Configuration with Boto3 (AWS)

```python
# auto_scaling_setup.py — Configure AWS Auto Scaling with boto3
import boto3

autoscaling = boto3.client('autoscaling')
cloudwatch = boto3.client('cloudwatch')

# Step 1: Create launch template (defines what instances look like)
ec2 = boto3.client('ec2')
ec2.create_launch_template(
    LaunchTemplateName='web-server-template',
    LaunchTemplateData={
        'ImageId': 'ami-0123456789abcdef0',  # Your app's AMI
        'InstanceType': 't3.xlarge',
        'SecurityGroupIds': ['sg-web-servers'],
        'UserData': 'IyEvYmluL2Jhc2gKZG9ja2VyIHJ1biAteWFwcA==',  # base64 startup
    }
)

# Step 2: Create Auto Scaling Group
autoscaling.create_auto_scaling_group(
    AutoScalingGroupName='web-asg',
    LaunchTemplate={'LaunchTemplateName': 'web-server-template', 'Version': '$Latest'},
    MinSize=2,          # Never fewer than 2 (high availability)
    MaxSize=20,         # Never more than 20 (cost protection)
    DesiredCapacity=4,  # Start with 4
    TargetGroupARNs=['arn:aws:elasticloadbalancing:...:targetgroup/web/...'],
    AvailabilityZones=['us-east-1a', 'us-east-1b', 'us-east-1c'],
    HealthCheckType='ELB',
    HealthCheckGracePeriod=300,  # 5 min to warm up before health checks
    Tags=[{'Key': 'Name', 'Value': 'web-server', 'PropagateAtLaunch': True}],
)

# Step 3: Create target tracking scaling policy
autoscaling.put_scaling_policy(
    AutoScalingGroupName='web-asg',
    PolicyName='target-cpu-50',
    PolicyType='TargetTrackingScaling',
    TargetTrackingConfiguration={
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'ASGAverageCPUUtilization'
        },
        'TargetValue': 50.0,      # Keep CPU at 50%
        'ScaleInCooldown': 300,   # Wait 5 min before removing instances
        'ScaleOutCooldown': 60,   # Wait 1 min before adding more
    }
)
print("Auto scaling configured! System will maintain ~50% CPU utilization.")
```

### Java — Custom Metrics Publisher for Auto Scaling

```java
// CustomMetricsPublisher.java — Publish app metrics for auto scaling decisions
import software.amazon.awssdk.services.cloudwatch.CloudWatchClient;
import software.amazon.awssdk.services.cloudwatch.model.*;
import java.time.Instant;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class CustomMetricsPublisher {
    
    private final CloudWatchClient cloudWatch;
    private final AtomicInteger activeRequests = new AtomicInteger(0);
    private final AtomicInteger queueDepth = new AtomicInteger(0);
    
    public CustomMetricsPublisher() {
        this.cloudWatch = CloudWatchClient.create();
        // Publish metrics every 60 seconds
        Executors.newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(this::publishMetrics, 0, 60, TimeUnit.SECONDS);
    }
    
    // Call this when a request starts/ends
    public void requestStarted() { activeRequests.incrementAndGet(); }
    public void requestCompleted() { activeRequests.decrementAndGet(); }
    
    private void publishMetrics() {
        // Publish custom metric that auto scaler can use
        cloudWatch.putMetricData(PutMetricDataRequest.builder()
            .namespace("MyApp/WebServers")
            .metricData(
                MetricDatum.builder()
                    .metricName("ActiveRequestsPerInstance")
                    .value((double) activeRequests.get())
                    .unit(StandardUnit.COUNT)
                    .timestamp(Instant.now())
                    .build(),
                MetricDatum.builder()
                    .metricName("RequestQueueDepth")
                    .value((double) queueDepth.get())
                    .unit(StandardUnit.COUNT)
                    .timestamp(Instant.now())
                    .build()
            ).build());
        
        System.out.printf("Published metrics: requests=%d, queue=%d%n",
            activeRequests.get(), queueDepth.get());
    }
    
    // Auto scaler can now scale based on ActiveRequestsPerInstance > 100
    // instead of just CPU — much more accurate for web applications!
}
```

---

## Infrastructure Examples

### AWS Auto Scaling — Complete Terraform Setup

```hcl
# main.tf — Production auto scaling infrastructure

resource "aws_launch_template" "web" {
  name_prefix   = "web-server-"
  image_id      = "ami-0123456789abcdef0"
  instance_type = "t3.xlarge"

  network_interfaces {
    security_groups = [aws_security_group.web.id]
  }

  user_data = base64encode(<<-EOF
    #!/bin/bash
    docker pull myapp:latest
    docker run -d -p 8080:8080 myapp:latest
  EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = { Name = "web-server" }
  }
}

resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  min_size            = 2
  max_size            = 20
  desired_capacity    = 4
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [aws_lb_target_group.web.arn]
  health_check_type   = "ELB"
  health_check_grace_period = 300

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  # Distribute evenly across AZs
  enabled_metrics = ["GroupMinSize", "GroupMaxSize", "GroupDesiredCapacity",
                     "GroupInServiceInstances", "GroupTotalInstances"]
}

# Target tracking: maintain 50% average CPU
resource "aws_autoscaling_policy" "cpu_target" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value       = 50.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# Also scale on request count per target
resource "aws_autoscaling_policy" "request_count" {
  name                   = "request-count-tracking"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.web.arn_suffix}/${aws_lb_target_group.web.arn_suffix}"
    }
    target_value = 1000  # Max 1000 requests per instance per minute
  }
}
```

---

## Real-World Example

### How Netflix Auto Scales for Global Streaming

```
NETFLIX AUTO SCALING ARCHITECTURE:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  CHALLENGE:                                                          │
│  • 250+ million subscribers worldwide                                │
│  • Traffic varies 3-5x between off-peak and peak (evening)          │
│  • Different peaks in different time zones                           │
│  • New show release = instant 2-3x traffic spike                    │
│                                                                      │
│  SOLUTION: Multi-layer auto scaling on AWS                          │
│                                                                      │
│  Layer 1: Regional Auto Scaling                                      │
│  ├── Each AWS region has its own Auto Scaling Groups                │
│  ├── US peaks at 8 PM EST → US region scales UP                    │
│  ├── US off-peak at 4 AM → US region scales DOWN                   │
│  └── Europe peaks at 8 PM CET → Europe scales UP                   │
│                                                                      │
│  Layer 2: Service-Level Auto Scaling                                 │
│  ├── Hundreds of microservices, EACH with own ASG                   │
│  ├── Recommendation service: scales with browsing traffic           │
│  ├── Streaming service: scales with concurrent viewers              │
│  └── Search service: scales with query volume                       │
│                                                                      │
│  Layer 3: Custom Metrics                                             │
│  ├── NOT just CPU! Netflix scales on:                               │
│  │   ├── Requests per second per instance                           │
│  │   ├── Stream starts per second                                   │
│  │   ├── P99 latency (if latency rises, add capacity)              │
│  │   └── Error rate (errors rising = need more instances)           │
│  └── Titus (Netflix's container orchestrator) handles execution     │
│                                                                      │
│  RESULTS:                                                            │
│  ├── Scales from thousands to tens of thousands of instances daily  │
│  ├── New show launch: pre-scales 2x BEFORE the release             │
│  ├── Saves millions in compute costs by scaling down at off-peak    │
│  └── Zero manual intervention needed                                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Only scaling on CPU | App might be I/O bound (CPU is low, but latency is high) | Also track request count, latency, queue depth |
| Cooldown too short | Instances oscillate up/down constantly | Scale-out cooldown: 60-120s; Scale-in: 300-600s |
| No health check grace period | New instances terminated before app starts | Set grace period > app startup time (typically 2-5 min) |
| Max capacity too low | During unexpected spike, hits ceiling and users get errors | Set max high enough for worst-case; use billing alerts instead |
| Not testing scale-out time | Assumed instant; actually takes 3-5 minutes | Know your lead time; pre-scale for predictable events |
| Scaling in too aggressively | Traffic returns, but servers already removed | Scale in slowly (remove 1-2 at a time, not all at once) |
| Not warming up new instances | Fresh instance gets traffic before cache is populated | Use lifecycle hooks to warm cache/connections before serving |

---

## When to Use / When NOT to Use

### ✅ Use Auto Scaling When:

- Traffic is **variable** (peaks and valleys throughout the day)
- You want to **minimize costs** (don't pay for idle servers)
- You need **high availability** (auto-replace failed instances)
- Traffic **spikes are unpredictable** (viral content, flash sales)
- Running in the **cloud** (easy to spin up/down instances)

### ❌ Auto Scaling May Not Be Needed When:

- Traffic is **constant** (steady 24/7 load)
- You're running on **bare metal** (can't add machines on-demand)
- Application **startup time is very long** (10+ minutes) — won't help for sudden spikes
- Cost of **always-on capacity** is acceptable and predictable
- Running a small project where **a single server handles everything**

### Decision Guide:

```
Does your traffic vary more than 2x between peak and off-peak?
├── YES → Auto scaling will save significant costs ✓
└── NO  → Fixed capacity may be simpler and sufficient

Can your application start and serve traffic within 5 minutes?
├── YES → Auto scaling is effective ✓
└── NO  → Consider predictive/scheduled scaling (Chapter 7.6)

Are you in the cloud (AWS, GCP, Azure)?
├── YES → Auto scaling is native and easy ✓
└── NO  → Consider Kubernetes autoscaling (Chapter 7.4)
```

---

## Key Takeaways

1. **Auto scaling is a feedback loop** — it monitors metrics, compares to thresholds, and adds or removes instances automatically to maintain your desired performance level.

2. **Auto Scaling Groups (ASGs)** define what instances look like (launch template) and how many should exist (min/desired/max capacity).

3. **Target tracking is the simplest approach** — set a target metric (e.g., 50% CPU) and let the auto scaler figure out how many instances are needed.

4. **Cooldown periods prevent oscillation** — after a scaling action, the system waits before making another decision. Scale-out cooldowns should be shorter than scale-in cooldowns.

5. **New instances take time** — from decision to serving traffic is typically 3-5 minutes. For instant spikes, pre-warming or over-provisioning is needed.

6. **Scale on the RIGHT metric** — CPU alone is often insufficient. Request count per instance, latency percentiles, and queue depth are often better indicators.

7. **Auto scaling saves money AND improves reliability** — fewer idle servers during off-peak, automatic replacement of failed instances, and automatic response to traffic spikes.

---

## What's Next?

You know what auto scaling is and how it works. But what METRIC should trigger scaling? CPU? Memory? Request count? Something custom? In **Chapter 7.3: Scaling Policies**, we'll dive deep into choosing the right metrics, configuring alarm thresholds, and building policies that respond correctly to different types of load patterns.
