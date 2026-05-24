# Scaling Policies — CPU, Memory, Request Count, Custom Metrics

> **What you'll learn**: How to choose the RIGHT metric to trigger auto scaling, configure alarm thresholds that avoid false positives, design multi-metric policies that catch different types of bottlenecks, and build custom metrics that reflect your application's actual health better than generic system metrics.

---

## Real-Life Analogy — Fire Station Deployment

A city's fire department decides how many fire trucks to have ready based on multiple signals:

| Signal | What it means | Action |
|--------|--------------|--------|
| **Temperature outside** (CPU) | Hot days = more fires | Staff extra trucks when temp > 95°F |
| **Number of 911 calls** (Request count) | More calls = more incidents | Add trucks when calls > 50/hour |
| **Average response time** (Latency) | If taking too long, need more trucks nearby | Add if avg > 8 minutes |
| **Festival happening** (Custom event) | Known high-risk event | Pre-deploy extra trucks |

Using ONLY temperature would miss arson waves. Using ONLY call count would miss false alarms. The best strategy? **Multiple signals combined.**

Same for auto scaling — no single metric tells the whole story.

---

## Core Concept Explained Step-by-Step

### Step 1: The Four Standard Metrics

```
┌─────────────────────────────────────────────────────────────────────┐
│                  THE FOUR PILLARS OF SCALING METRICS                  │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐                       │
│  │    CPU Usage     │  │  Memory Usage    │                       │
│  │                  │  │                  │                       │
│  │  "How hard is   │  │  "How full is    │                       │
│  │   the brain     │  │   the workspace  │                       │
│  │   working?"     │  │   (RAM)?"        │                       │
│  │                  │  │                  │                       │
│  │  Best for:      │  │  Best for:       │                       │
│  │  Compute-heavy  │  │  In-memory apps, │                       │
│  │  workloads      │  │  caching servers │                       │
│  └──────────────────┘  └──────────────────┘                       │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐                       │
│  │  Request Count   │  │  Custom Metrics  │                       │
│  │                  │  │                  │                       │
│  │  "How many      │  │  "What does YOUR │                       │
│  │   people are    │  │   app actually   │                       │
│  │   asking for    │  │   care about?"   │                       │
│  │   things?"      │  │                  │                       │
│  │                  │  │  Best for:       │                       │
│  │  Best for:      │  │  Business-aware  │                       │
│  │  Web APIs,      │  │  scaling (queue  │                       │
│  │  HTTP servers   │  │  depth, latency) │                       │
│  └──────────────────┘  └──────────────────┘                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 2: CPU-Based Scaling — The Default Choice

```
CPU SCALING: "Scale when servers are working too hard"

How it works:
┌─────────────────────────────────────────────────────┐
│  Metric: Average CPU across all instances in ASG    │
│  Target: 60%                                         │
│                                                     │
│  Example with 4 instances:                          │
│  Server 1: 55% CPU                                  │
│  Server 2: 70% CPU                                  │
│  Server 3: 65% CPU                                  │
│  Server 4: 50% CPU                                  │
│  Average: (55+70+65+50)/4 = 60% ← right on target │
│                                                     │
│  If average rises to 75%:                           │
│  Need instances = current × (actual/target)        │
│  Need = 4 × (75/60) = 5 instances                  │
│  → Launch 1 more instance                           │
└─────────────────────────────────────────────────────┘

When CPU-based scaling works WELL:
├── Compute-heavy applications (image processing, ML inference)
├── Applications with linear CPU-to-throughput relationship
└── Well-distributed request processing

When CPU-based scaling FAILS:
├── I/O bound apps (waiting on DB, not using CPU)
│   └── CPU stays at 20% but responses are slow!
├── Apps with GC pauses (CPU spikes from GC, not real load)
├── Apps that cache heavily (memory is bottleneck, not CPU)
└── WebSocket servers (connections open but idle CPU)

EXAMPLE: Why CPU alone fails for I/O-bound apps:
┌────────────────────────────────────────────────────────────┐
│ Web app making database calls:                             │
│                                                            │
│ Request: GET /api/users (takes 200ms, mostly DB wait)      │
│                                                            │
│ Server timeline for one request:                           │
│ [CPU: 5ms][WAIT for DB: 190ms][CPU: 5ms]                  │
│                                                            │
│ CPU utilization: 10ms / 200ms = 5%                        │
│                                                            │
│ Server handles 100 concurrent requests:                    │
│ CPU: still only ~30% (waiting on DB most of the time!)    │
│ But response time: 2000ms (DB connection pool exhausted!)  │
│                                                            │
│ Auto scaler: "CPU is 30%, no need to scale"  ← WRONG!    │
│ Users: "The app is so slow!"                              │
└────────────────────────────────────────────────────────────┘
```

### Step 3: Memory-Based Scaling

```
MEMORY SCALING: "Scale when servers run out of workspace"

┌─────────────────────────────────────────────────────┐
│  When to use:                                        │
│  ├── In-memory caching applications                 │
│  ├── Java apps with large heaps                     │
│  ├── Applications that buffer data in RAM           │
│  └── ML models loaded into memory                   │
│                                                     │
│  Typical threshold: Scale when memory > 75%          │
│                                                     │
│  CAUTION: Memory often doesn't decrease after       │
│  traffic drops (memory leaks, GC not reclaiming).   │
│  This makes scale-IN unreliable with memory metric! │
│                                                     │
│  Memory usage pattern:                              │
│  ────────────────────────────────────────────       │
│  100%│        ╱───────────── memory never drops     │
│   75%│───────╱──── threshold (scale!)               │
│   50%│    ╱                                         │
│   25%│  ╱                                           │
│     0│╱                                             │
│      └────────────────────────── time               │
│                                                     │
│  vs CPU pattern (naturally drops when load drops):  │
│  100%│     ╱╲                                       │
│   75%│   ╱    ╲                                     │
│   50%│  ╱      ╲╱╲                                  │
│   25%│╱            ╲                                │
│     0│               ╲───                           │
│      └────────────────────────── time               │
└─────────────────────────────────────────────────────┘
```

### Step 4: Request Count-Based Scaling

```
REQUEST COUNT: "Scale when too many people are asking"

This is often the BEST metric for web applications:

┌─────────────────────────────────────────────────────────────────┐
│  Metric: Requests per instance per minute (from Load Balancer)  │
│  Target: 1000 requests/instance/minute                          │
│                                                                 │
│  Why it's better than CPU:                                      │
│  ├── Directly measures LOAD on your application                │
│  ├── Works for I/O-bound AND CPU-bound apps                    │
│  ├── Easy to reason about: "each server handles 1000 req/min"  │
│  └── Predictable: 5000 req/min ÷ 1000 = need 5 servers       │
│                                                                 │
│  HOW IT WORKS WITH ALB (AWS):                                  │
│                                                                 │
│  ALB tracks: Total requests hitting the target group            │
│  Divided by: Number of healthy instances                        │
│  = ALBRequestCountPerTarget                                     │
│                                                                 │
│  Example:                                                       │
│  6000 requests/min hitting ALB                                  │
│  Current: 4 instances                                           │
│  Per instance: 6000/4 = 1500 req/instance/min                  │
│  Target: 1000 req/instance/min                                  │
│  Need: 6000/1000 = 6 instances                                  │
│  Action: Add 2 more instances!                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Step 5: Custom Metrics — The Expert Choice

```
CUSTOM METRICS: "Scale on what actually matters to YOUR app"

┌─────────────────────────────────────────────────────────────────────┐
│                    EXAMPLES OF CUSTOM METRICS                         │
│                                                                     │
│  ┌─────────────────────────────────────┐                           │
│  │ Queue Depth (Message Queues)        │                           │
│  │ "How many messages are waiting?"     │                           │
│  │ If queue > 1000 → add workers        │                           │
│  │                                     │                           │
│  │ Best for: background job processors │                           │
│  │ (email senders, image processors)   │                           │
│  └─────────────────────────────────────┘                           │
│                                                                     │
│  ┌─────────────────────────────────────┐                           │
│  │ P99 Latency                         │                           │
│  │ "How slow are the slowest requests?" │                           │
│  │ If P99 > 500ms → add servers         │                           │
│  │                                     │                           │
│  │ Best for: user-facing APIs where     │                           │
│  │ latency = user experience            │                           │
│  └─────────────────────────────────────┘                           │
│                                                                     │
│  ┌─────────────────────────────────────┐                           │
│  │ Active Connections                   │                           │
│  │ "How many users are connected?"      │                           │
│  │ If connections/server > 5000 → scale │                           │
│  │                                     │                           │
│  │ Best for: WebSocket apps, real-time  │                           │
│  │ chat, gaming servers                 │                           │
│  └─────────────────────────────────────┘                           │
│                                                                     │
│  ┌─────────────────────────────────────┐                           │
│  │ Error Rate                          │                           │
│  │ "What % of requests are failing?"    │                           │
│  │ If error_rate > 5% → add capacity    │                           │
│  │                                     │                           │
│  │ Best for: detecting overload before  │                           │
│  │ traditional metrics catch it         │                           │
│  └─────────────────────────────────────┘                           │
│                                                                     │
│  ┌─────────────────────────────────────┐                           │
│  │ Business Metrics                    │                           │
│  │ "Concurrent video streams"           │                           │
│  │ "Active game sessions"               │                           │
│  │ "Orders being processed"             │                           │
│  │                                     │                           │
│  │ Best for: domain-specific scaling    │                           │
│  │ (Netflix, gaming, e-commerce)        │                           │
│  └─────────────────────────────────────┘                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 6: Multi-Metric Policies — Belt AND Suspenders

```
COMBINING MULTIPLE METRICS:

You can set MULTIPLE scaling policies on the same ASG.
The most aggressive policy wins (if ANY says "scale out", it scales out).

┌──────────────────────────────────────────────────────────────┐
│  Auto Scaling Group: "web-servers"                            │
│                                                              │
│  Policy 1: Target CPU = 60%                                  │
│  Policy 2: Target Request Count = 1000/instance/min          │
│  Policy 3: Scale out if P99 latency > 500ms                 │
│                                                              │
│  Scenario A: Black Friday sale                               │
│  ├── CPU: 40% (requests are fast I/O lookups)               │
│  ├── Requests: 3000/instance/min ← TRIGGERS POLICY 2!      │
│  └── Scale out because requests too high                    │
│                                                              │
│  Scenario B: ML model update causing slow responses          │
│  ├── CPU: 90% ← TRIGGERS POLICY 1!                         │
│  ├── Requests: 500/instance/min (low because things are slow)│
│  └── Scale out because CPU too high                         │
│                                                              │
│  Scenario C: Database slowdown                               │
│  ├── CPU: 20% (servers waiting on DB, not computing)        │
│  ├── Requests: 800/instance/min (below threshold)           │
│  ├── P99 latency: 2000ms ← TRIGGERS POLICY 3!             │
│  └── Scale out because latency too high                     │
│                                                              │
│  With all three, you catch EVERY type of overload!          │
└──────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### CloudWatch Alarm Anatomy

```
ALARM STRUCTURE (AWS CloudWatch):

┌──────────────────────────────────────────────────────────────────┐
│  Alarm: "web-cpu-high"                                           │
│                                                                  │
│  Metric:     AWS/EC2 → CPUUtilization                           │
│  Statistic:  Average                                             │
│  Period:     60 seconds (evaluate every 1 min)                  │
│  Threshold:  > 70%                                               │
│  Datapoints: 3 out of 5 (must breach 3 of last 5 periods)      │
│                                                                  │
│  WHAT "3 out of 5" MEANS:                                       │
│                                                                  │
│  Time:    T1    T2    T3    T4    T5    Decision                │
│  CPU:     75%   68%   80%   72%   76%                           │
│  Breach?  YES   NO    YES   YES   YES   3/5 → ALARM! 🚨        │
│                                                                  │
│  vs:                                                             │
│  Time:    T1    T2    T3    T4    T5    Decision                │
│  CPU:     75%   50%   72%   45%   60%                           │
│  Breach?  YES   NO    YES   NO    NO    2/5 → OK (no alarm)    │
│                                                                  │
│  This prevents one spike from triggering unnecessary scaling!   │
│                                                                  │
│  ALARM STATES:                                                   │
│  ┌────┐ breach  ┌───────┐ resolve  ┌────┐                      │
│  │ OK │────────▶│ ALARM │─────────▶│ OK │                      │
│  └────┘         └───────┘          └────┘                      │
│                    │                                             │
│                    └── Triggers scaling action                   │
└──────────────────────────────────────────────────────────────────┘
```

### Metric Aggregation Across Instances

```
HOW METRICS ARE AGGREGATED:

4 instances reporting CPU every 60 seconds:

Time T1:
  Instance 1: 45%
  Instance 2: 80%
  Instance 3: 55%
  Instance 4: 60%

Aggregation options:
┌──────────────────────────────────────────────────────────────┐
│  Average:  (45+80+55+60)/4 = 60%                            │
│            Good for: general load assessment                 │
│            Risk: masks one overloaded instance               │
│                                                              │
│  Maximum:  max(45,80,55,60) = 80%                           │
│            Good for: catching ANY overloaded instance        │
│            Risk: too sensitive, may trigger on outliers      │
│                                                              │
│  Minimum:  min(45,80,55,60) = 45%                           │
│            Good for: scale-IN decisions                      │
│            (only scale in when ALL instances are underloaded)│
│                                                              │
│  Sum:      45+80+55+60 = 240 (total CPU seconds)           │
│            Good for: total workload measurement              │
│                                                              │
│  p90/p99:  Statistical percentile                           │
│            Good for: latency metrics                        │
│            "90% of requests complete within X ms"           │
└──────────────────────────────────────────────────────────────┘

RECOMMENDATION:
├── Scale OUT on: Average (catches overall overload)
├── Scale IN on: Maximum (only shrink when even the busiest is idle)
└── Alert on: p99 latency (catches tail latency problems)
```

---

## Code Examples

### Python — Custom Metric Publisher with CloudWatch

```python
# custom_metrics.py — Publish application-specific metrics for auto scaling
import boto3
import time
import threading
from collections import deque

cloudwatch = boto3.client('cloudwatch')

class AppMetrics:
    """Collect and publish custom metrics for auto scaling decisions."""
    
    def __init__(self, service_name: str):
        self.service_name = service_name
        self.request_latencies = deque(maxlen=1000)  # Last 1000 requests
        self.active_connections = 0
        self.error_count = 0
        self.request_count = 0
        self._start_publisher()
    
    def record_request(self, latency_ms: float, success: bool):
        """Call this after every request completes."""
        self.request_latencies.append(latency_ms)
        self.request_count += 1
        if not success:
            self.error_count += 1
    
    def _calculate_p99(self) -> float:
        """Calculate 99th percentile latency."""
        if not self.request_latencies:
            return 0
        sorted_latencies = sorted(self.request_latencies)
        idx = int(len(sorted_latencies) * 0.99)
        return sorted_latencies[min(idx, len(sorted_latencies) - 1)]
    
    def _publish(self):
        """Publish metrics every 60 seconds to CloudWatch."""
        while True:
            p99 = self._calculate_p99()
            error_rate = (self.error_count / max(self.request_count, 1)) * 100
            
            cloudwatch.put_metric_data(
                Namespace=f'MyApp/{self.service_name}',
                MetricData=[
                    {'MetricName': 'P99Latency', 'Value': p99, 'Unit': 'Milliseconds'},
                    {'MetricName': 'ErrorRate', 'Value': error_rate, 'Unit': 'Percent'},
                    {'MetricName': 'ActiveConnections', 'Value': self.active_connections,
                     'Unit': 'Count'},
                ]
            )
            # Reset counters for next period
            self.error_count = 0
            self.request_count = 0
            time.sleep(60)
    
    def _start_publisher(self):
        thread = threading.Thread(target=self._publish, daemon=True)
        thread.start()

# Usage in your web application:
metrics = AppMetrics("order-service")

@app.route('/api/orders', methods=['POST'])
def create_order():
    start = time.time()
    try:
        result = process_order(request.json)
        metrics.record_request((time.time() - start) * 1000, success=True)
        return result
    except Exception as e:
        metrics.record_request((time.time() - start) * 1000, success=False)
        raise
```

### Java — Queue-Based Auto Scaling Metric

```java
// QueueDepthMetricPublisher.java — Scale workers based on queue depth
import software.amazon.awssdk.services.cloudwatch.CloudWatchClient;
import software.amazon.awssdk.services.cloudwatch.model.*;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.*;
import java.time.Instant;
import java.util.concurrent.*;

public class QueueDepthMetricPublisher {
    
    private final SqsClient sqs;
    private final CloudWatchClient cloudWatch;
    private final String queueUrl;
    
    public QueueDepthMetricPublisher(String queueUrl) {
        this.sqs = SqsClient.create();
        this.cloudWatch = CloudWatchClient.create();
        this.queueUrl = queueUrl;
    }
    
    public void startPublishing() {
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
        scheduler.scheduleAtFixedRate(this::publishQueueDepth, 0, 60, TimeUnit.SECONDS);
    }
    
    private void publishQueueDepth() {
        // Get current queue depth from SQS
        GetQueueAttributesResponse attrs = sqs.getQueueAttributes(
            GetQueueAttributesRequest.builder()
                .queueUrl(queueUrl)
                .attributeNames(QueueAttributeName.APPROXIMATE_NUMBER_OF_MESSAGES)
                .build()
        );
        
        int queueDepth = Integer.parseInt(
            attrs.attributes().get(QueueAttributeName.APPROXIMATE_NUMBER_OF_MESSAGES));
        
        // Publish to CloudWatch for auto scaling to use
        cloudWatch.putMetricData(PutMetricDataRequest.builder()
            .namespace("MyApp/Workers")
            .metricData(MetricDatum.builder()
                .metricName("QueueDepth")
                .value((double) queueDepth)
                .unit(StandardUnit.COUNT)
                .timestamp(Instant.now())
                .dimensions(Dimension.builder()
                    .name("QueueName")
                    .value("order-processing")
                    .build())
                .build())
            .build());
        
        System.out.printf("Queue depth: %d messages waiting%n", queueDepth);
        // Auto scaling policy: if QueueDepth > 100, add worker instances
        // if QueueDepth < 10 for 10 minutes, remove workers
    }
}
```

---

## Infrastructure Examples

### AWS — Multi-Metric Scaling Policy (Terraform)

```hcl
# Multi-metric scaling: covers CPU, requests, AND latency

# Policy 1: CPU-based (catches compute bottlenecks)
resource "aws_autoscaling_policy" "cpu" {
  name                   = "cpu-scaling"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"
  
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60.0
  }
}

# Policy 2: Request count (catches traffic increases)
resource "aws_autoscaling_policy" "requests" {
  name                   = "request-count-scaling"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"
  
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.web.arn_suffix}/${aws_lb_target_group.web.arn_suffix}"
    }
    target_value = 1000.0  # 1000 requests per instance per period
  }
}

# Policy 3: Custom metric — P99 latency (catches I/O bottlenecks)
resource "aws_autoscaling_policy" "latency" {
  name                   = "latency-scaling"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "StepScaling"
  adjustment_type        = "ChangeInCapacity"
  
  step_adjustment {
    scaling_adjustment          = 2   # Add 2 instances
    metric_interval_lower_bound = 0   # When latency exceeds threshold
    metric_interval_upper_bound = 200 # Up to 200ms above threshold
  }
  step_adjustment {
    scaling_adjustment          = 4   # Add 4 instances (emergency)
    metric_interval_lower_bound = 200 # More than 200ms above threshold
  }
}

# Alarm for latency policy
resource "aws_cloudwatch_metric_alarm" "high_latency" {
  alarm_name          = "web-high-p99-latency"
  namespace           = "MyApp/WebServers"
  metric_name         = "P99Latency"
  statistic           = "Average"
  period              = 60
  evaluation_periods  = 3
  threshold           = 500  # 500ms P99 latency
  comparison_operator = "GreaterThanThreshold"
  alarm_actions       = [aws_autoscaling_policy.latency.arn]
}
```

### Queue-Based Worker Scaling

```hcl
# Worker auto scaling based on SQS queue depth
resource "aws_autoscaling_policy" "queue_workers" {
  name                   = "queue-depth-scaling"
  autoscaling_group_name = aws_autoscaling_group.workers.name
  policy_type            = "TargetTrackingScaling"
  
  target_tracking_configuration {
    customized_metric_specification {
      metric_dimension {
        name  = "QueueName"
        value = "order-processing"
      }
      metric_name = "ApproximateNumberOfMessagesVisible"
      namespace   = "AWS/SQS"
      statistic   = "Average"
    }
    target_value = 10.0  # Target: 10 messages per worker instance
    # If queue has 100 messages and target is 10 → need 10 workers
    # If queue has 5 messages → can scale down to 1 worker
  }
}
```

---

## Real-World Example

### How Uber Scales on Custom Metrics

```
UBER'S MULTI-METRIC SCALING:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  Uber can't scale on CPU alone:                                     │
│  - During a rainstorm: ride requests spike 5x in minutes           │
│  - CPU might be fine, but request queue is growing!                 │
│  - Need to scale IMMEDIATELY, not wait for CPU to rise             │
│                                                                      │
│  Uber's custom scaling metrics:                                     │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ Service: Ride Matching                                      │    │
│  │ Scale on: pending_ride_requests / available_drivers         │    │
│  │ Target: < 3 (max 3 pending requests per available driver)   │    │
│  │ If ratio > 5: EMERGENCY scale, add 50% more capacity       │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ Service: ETA Calculation                                    │    │
│  │ Scale on: p99_eta_calculation_time                          │    │
│  │ Target: < 200ms                                             │    │
│  │ If P99 > 500ms: scale out (users seeing "calculating...")   │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ Service: Surge Pricing Engine                               │    │
│  │ Scale on: events_per_second in geographic zone              │    │
│  │ Target: < 10000 events/sec/instance                         │    │
│  │ Spikes during concerts, sports events, New Year's Eve       │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  KEY INSIGHT: Uber scales on BUSINESS metrics, not system metrics. │
│  "Pending rides per driver" tells more than CPU ever could.        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Scaling only on CPU for I/O-bound apps | CPU stays low while requests timeout | Add request count or latency policies |
| Threshold too low (e.g., CPU > 40%) | Constant scaling up/down, wasted money | Set higher (60-70% CPU) with proper evaluation periods |
| Threshold too high (e.g., CPU > 90%) | Instances already overloaded before scaling triggers | Scale at 70% to leave headroom for spike absorption |
| Single datapoint triggers scaling | One spike (GC pause, health check) causes unnecessary scaling | Use "3 of 5" evaluation periods |
| Not accounting for instance startup time | By the time new instance is ready, users already got errors | Scale earlier (lower threshold) or use predictive scaling |
| Using Average for scale-in | Average is low because new instances are idle | Use Maximum for scale-in decisions |
| Forgetting to scale the database | App servers scale but DB becomes bottleneck | Scale DB independently or use connection pooling |

---

## When to Use / When NOT to Use Each Metric

```
METRIC SELECTION GUIDE:

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  What's your application type?                                 │
│                                                                 │
│  CPU-heavy (ML inference, video encoding, crypto):             │
│  └── PRIMARY: CPU utilization ✓                                │
│                                                                 │
│  I/O-heavy (REST APIs, database-backed services):              │
│  └── PRIMARY: Request count per target ✓                       │
│  └── SECONDARY: P99 latency ✓                                 │
│                                                                 │
│  Real-time (WebSocket, gaming, chat):                          │
│  └── PRIMARY: Active connections per instance ✓                │
│                                                                 │
│  Queue workers (background jobs, email, notifications):        │
│  └── PRIMARY: Queue depth / backlog ✓                          │
│                                                                 │
│  Memory-intensive (caching, data processing):                  │
│  └── PRIMARY: Memory utilization ✓                             │
│  └── NOTE: Hard to scale IN on memory (doesn't release easily)│
│                                                                 │
│  ALL APPS should add:                                          │
│  └── SAFETY NET: Error rate > 5% → emergency scale-out        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **CPU alone is insufficient** for most web applications — I/O-bound apps can have low CPU but terrible response times. Always use multiple metrics.

2. **Request count per target is often the best primary metric** for web APIs — it directly measures load and gives predictable, linear scaling.

3. **Custom metrics are the expert choice** — queue depth, P99 latency, error rate, and business metrics give the most accurate scaling signals for your specific application.

4. **Use multiple policies simultaneously** — they work independently, and the most aggressive one wins. This catches different types of bottlenecks.

5. **Evaluation periods prevent false alarms** — require the threshold to be breached for multiple consecutive periods (e.g., 3 of 5 minutes) before acting.

6. **Scale OUT aggressively, scale IN conservatively** — it's better to have one extra server than to be caught short during a traffic spike.

7. **The right metric depends on your bottleneck** — compute-bound apps need CPU metrics; I/O-bound need latency/request count; queue workers need queue depth.

---

## What's Next?

You've mastered scaling policies for VM-based auto scaling. But what about containers? In **Chapter 7.4: Kubernetes HPA, VPA & Cluster Autoscaler**, we'll explore how Kubernetes handles auto scaling at multiple levels — scaling pods within a node, scaling pod resources, and scaling the nodes themselves.
