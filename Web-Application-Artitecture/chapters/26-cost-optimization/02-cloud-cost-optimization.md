# Cost Optimization in Cloud — Reserved, Spot & On-Demand

> **What you'll learn**: How cloud pricing models work, how to save 30-70% on your infrastructure bill, and how to build a smart purchasing strategy that balances cost with reliability.

---

## Real-Life Analogy

Think of cloud computing like booking hotel rooms:

- **On-Demand** = Walk-in rate. Show up, get a room, pay full price. No commitment, maximum flexibility. Expensive per night.
- **Reserved Instances** = Annual lease. You sign a 1-year or 3-year contract for a specific room. You get 30-60% off, but you pay even if you don't show up.
- **Spot Instances** = Last-minute deal on an app. The hotel has empty rooms and offers them at 70-90% off. Amazing price, BUT they can kick you out with 2 minutes notice if a full-price guest shows up.
- **Savings Plans** = Loyalty program. You commit to spending $X per hour on any room type. You get a discount without being locked to a specific room.

**The smartest guests (companies) use a mix of all strategies** — a fixed lease for rooms they always need, walk-in for occasional trips, and last-minute deals for flexible stays.

---

## Core Concept Explained Step-by-Step

### The Cloud Pricing Landscape

```
┌────────────────────────────────────────────────────────────────┐
│              CLOUD PRICING MODELS                               │
├─────────────────┬──────────┬────────────┬─────────────────────┤
│ Model           │ Discount │ Commitment │ Interruption Risk   │
├─────────────────┼──────────┼────────────┼─────────────────────┤
│ On-Demand       │ 0%       │ None       │ None                │
│ Reserved (1yr)  │ 30-40%   │ 1 year     │ None                │
│ Reserved (3yr)  │ 50-60%   │ 3 years    │ None                │
│ Savings Plans   │ 30-60%   │ 1-3 years  │ None                │
│ Spot/Preemptible│ 60-90%   │ None       │ HIGH (2 min notice) │
│ Free Tier       │ 100%     │ None       │ None (limited)      │
└─────────────────┴──────────┴────────────┴─────────────────────┘
```

### How On-Demand Pricing Works

On-Demand is the "default" — you pay per hour (or per second) for exactly what you use.

```
┌──────────────────────────────────────────────────┐
│           ON-DEMAND PRICING                       │
│                                                  │
│  You use it ──▶ You pay for it                   │
│  You stop   ──▶ You stop paying                  │
│                                                  │
│  Example: m6i.xlarge (4 CPU, 16 GB)             │
│  AWS US-East: ~$0.192/hour = $140/month         │
│                                                  │
│  ✅ Best for: Unpredictable workloads,           │
│     short-term projects, dev/test                │
│  ❌ Expensive for: Steady 24/7 workloads         │
└──────────────────────────────────────────────────┘
```

### How Reserved Instances (RIs) Work

You commit to using a specific instance type in a specific region for 1 or 3 years.

```
                RESERVED INSTANCE FLOW
                
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │ You commit  │     │ AWS gives   │     │ You pay     │
    │ to 1 or 3  │────▶│ 30-60%      │────▶│ upfront,    │
    │ years       │     │ discount    │     │ monthly, or │
    │             │     │             │     │ hybrid      │
    └─────────────┘     └─────────────┘     └─────────────┘
    
    Payment Options:
    ┌─────────────────┬──────────┬──────────────────────┐
    │ Option          │ Discount │ Cash Flow            │
    ├─────────────────┼──────────┼──────────────────────┤
    │ All Upfront     │ Maximum  │ Pay everything now   │
    │ Partial Upfront │ Medium   │ Half now, half monthly│
    │ No Upfront      │ Minimum  │ All monthly          │
    └─────────────────┴──────────┴──────────────────────┘
```

**RI Types:**

| Type | Flexibility | Discount |
|------|------------|----------|
| **Standard RI** | Locked to specific instance type | Higher discount (up to 72%) |
| **Convertible RI** | Can change instance type/family | Lower discount (up to 54%) |

### How Spot Instances Work

Cloud providers have massive spare capacity. Instead of letting it sit idle, they sell it cheap — but can reclaim it anytime.

```
┌──────────────────────────────────────────────────────────────┐
│                 SPOT INSTANCE LIFECYCLE                        │
│                                                              │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│   │ Request  │──▶│ Running  │──▶│ Interrupt │               │
│   │ Spot     │   │ (cheap!) │   │ (2 min   │               │
│   │ Instance │   │          │   │ warning) │               │
│   └──────────┘   └──────────┘   └──────────┘               │
│        │                              │                      │
│        │              ┌───────────────┘                      │
│        ▼              ▼                                      │
│   ┌──────────┐   ┌──────────┐                               │
│   │ No       │   │ Instance │                               │
│   │ capacity │   │ terminated│                               │
│   │ (wait)   │   │ gracefully│                               │
│   └──────────┘   └──────────┘                               │
│                                                              │
│   Discount: 60-90% off On-Demand                            │
│   Risk: Can be taken away with 2-minute notice              │
└──────────────────────────────────────────────────────────────┘
```

### How Savings Plans Work (AWS)

A newer, more flexible alternative to RIs. You commit to a **dollar amount per hour** of compute usage.

```
┌──────────────────────────────────────────────────────────────┐
│                 SAVINGS PLANS                                  │
│                                                              │
│  "I commit to spending $10/hour on compute for 1 year"      │
│                                                              │
│  Types:                                                      │
│  ┌─────────────────────┬────────────────────────────────┐   │
│  │ Compute Savings Plan │ Any instance, any region,      │   │
│  │                      │ any OS. Maximum flexibility.   │   │
│  ├─────────────────────┼────────────────────────────────┤   │
│  │ EC2 Savings Plan     │ Specific instance family +     │   │
│  │                      │ region. Higher discount.       │   │
│  └─────────────────────┴────────────────────────────────┘   │
│                                                              │
│  How it works:                                               │
│  - Usage up to your commitment → discounted rate            │
│  - Usage beyond your commitment → on-demand rate            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### The Optimal Cost Strategy: Mix & Match

The best teams use a layered approach:

```
                    COST PYRAMID
                    
           ┌─────────────────┐
           │   ON-DEMAND     │  ← Spike/burst traffic (10-20%)
           │   (Expensive)   │
           ├─────────────────┤
           │   SPOT          │  ← Fault-tolerant workloads (20-30%)
           │   (Cheapest)    │
           ├─────────────────┤
           │   SAVINGS       │  ← Flexible committed usage (20-30%)
           │   PLANS         │
           ├─────────────────┤
           │   RESERVED      │  ← Steady-state baseline (30-50%)
           │   INSTANCES     │
           └─────────────────┘
           
    ← Most cost effective at the bottom
    ← Most flexible at the top
```

### Spot Instance Strategy — Diversification

The key to using Spot reliably is **diversification** — spread across multiple instance types and availability zones:

```
┌────────────────────────────────────────────────────────────┐
│         SPOT DIVERSIFICATION STRATEGY                       │
│                                                            │
│  Instead of:                                               │
│    10 × c5.xlarge in us-east-1a (ALL eggs in one basket)  │
│                                                            │
│  Do this:                                                  │
│    3 × c5.xlarge   in us-east-1a                          │
│    2 × c5a.xlarge  in us-east-1b    (AMD variant)         │
│    2 × c6i.xlarge  in us-east-1c    (newer gen)           │
│    3 × m5.xlarge   in us-east-1a    (different family)    │
│                                                            │
│  Why? Different instance pools have different              │
│  interruption rates. Diversification = more stability.    │
└────────────────────────────────────────────────────────────┘
```

### Understanding Cloud Cost Breakdown

For a typical web application, costs break down like:

```
┌─────────────────────────────────────────────────────┐
│        TYPICAL CLOUD COST BREAKDOWN                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Compute (EC2/VMs)        ████████████████  40%    │
│  Databases (RDS/DynamoDB) ██████████        25%    │
│  Data Transfer            ██████            15%    │
│  Storage (S3/EBS)         ████              10%    │
│  Other (LB, DNS, etc.)   ████              10%    │
│                                                     │
│  💡 Compute is usually the biggest bill.            │
│     Focus optimization efforts there first.         │
└─────────────────────────────────────────────────────┘
```

### Data Transfer — The Hidden Cost

Data transfer is often called the "cloud tax" because it's not obvious until the bill arrives:

```
┌─────────────────────────────────────────────────────────────┐
│              DATA TRANSFER COSTS                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Inbound (Internet → AWS)        FREE                      │
│  Within same AZ                  FREE                      │
│  Between AZs (same region)       $0.01/GB                  │
│  Between regions                 $0.02/GB                  │
│  Outbound (AWS → Internet)       $0.09/GB (first 10 TB)   │
│                                                             │
│  💡 A service doing 100 TB/month outbound:                 │
│     100,000 GB × $0.09 = $9,000/month JUST for transfer!  │
│                                                             │
│  Solutions:                                                 │
│  • Use CloudFront CDN (cheaper egress rates)               │
│  • Compress responses (gzip/brotli)                        │
│  • Keep chatty services in the same AZ                     │
│  • Use VPC endpoints for AWS service access                │
└─────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Cloud Cost Analyzer

```python
# cloud_cost_analyzer.py — Analyze and optimize cloud spending

from dataclasses import dataclass
from typing import List

@dataclass
class Instance:
    name: str
    vcpu: int
    memory_gb: int
    on_demand_hourly: float    # $/hour
    reserved_1yr_hourly: float # $/hour (no upfront)
    spot_hourly: float         # $/hour (average)
    utilization_pct: float     # Current avg CPU utilization

class CostOptimizer:
    """Analyzes instances and recommends pricing model."""
    
    HOURS_PER_MONTH = 730
    HOURS_PER_YEAR = 8760
    
    def __init__(self, instances: List[Instance]):
        self.instances = instances
    
    def analyze(self):
        """Analyze each instance and recommend optimal pricing."""
        total_current = 0
        total_optimized = 0
        
        print(f"{'Instance':<20} {'Current $/mo':<15} {'Recommended':<15} "
              f"{'Optimized $/mo':<15} {'Savings':<10}")
        print("-" * 75)
        
        for inst in self.instances:
            current_monthly = inst.on_demand_hourly * self.HOURS_PER_MONTH
            recommendation, optimized = self._recommend(inst)
            savings_pct = (1 - optimized / current_monthly) * 100
            
            print(f"{inst.name:<20} ${current_monthly:<14.2f} "
                  f"{recommendation:<15} ${optimized:<14.2f} {savings_pct:.0f}%")
            
            total_current += current_monthly
            total_optimized += optimized
        
        print("-" * 75)
        print(f"{'TOTAL':<20} ${total_current:<14.2f} "
              f"{'—':<15} ${total_optimized:<14.2f} "
              f"{(1 - total_optimized/total_current)*100:.0f}%")
    
    def _recommend(self, inst: Instance):
        """Recommend pricing model based on utilization."""
        monthly_on_demand = inst.on_demand_hourly * self.HOURS_PER_MONTH
        monthly_reserved = inst.reserved_1yr_hourly * self.HOURS_PER_MONTH
        monthly_spot = inst.spot_hourly * self.HOURS_PER_MONTH
        
        # High, steady utilization → Reserved
        if inst.utilization_pct > 60:
            return "Reserved (1yr)", monthly_reserved
        # Low utilization → Right-size first, then consider spot
        elif inst.utilization_pct < 20:
            # Suggest downsizing (rough: use spot price as proxy)
            return "Downsize+Spot", monthly_spot
        # Medium utilization, variable → Savings Plan
        else:
            # Savings plan gives ~30% discount with flexibility
            return "Savings Plan", monthly_on_demand * 0.7


# Example: Analyze a fleet of servers
fleet = [
    Instance("api-server-1", 8, 32, 0.384, 0.245, 0.115, 72),
    Instance("api-server-2", 8, 32, 0.384, 0.245, 0.115, 68),
    Instance("worker-1", 4, 16, 0.192, 0.123, 0.058, 15),
    Instance("cache-server", 8, 64, 0.504, 0.322, 0.151, 85),
    Instance("ml-batch", 16, 64, 0.768, 0.491, 0.230, 35),
]

optimizer = CostOptimizer(fleet)
optimizer.analyze()
```

### Java — Spot Instance Handler with Graceful Shutdown

```java
// SpotInstanceHandler.java — Handle spot interruption gracefully

import java.net.HttpURLConnection;
import java.net.URL;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Monitors for EC2 Spot termination notices and triggers
 * graceful shutdown when interrupted.
 * 
 * AWS provides a 2-minute warning via instance metadata.
 */
public class SpotInstanceHandler implements Runnable {
    
    // EC2 instance metadata endpoint for spot termination
    private static final String TERMINATION_URL = 
        "http://169.254.169.254/latest/meta-data/spot/instance-action";
    
    private final AtomicBoolean shutdownRequested = new AtomicBoolean(false);
    private final Runnable gracefulShutdownAction;
    
    public SpotInstanceHandler(Runnable gracefulShutdownAction) {
        this.gracefulShutdownAction = gracefulShutdownAction;
    }
    
    public boolean isShutdownRequested() {
        return shutdownRequested.get();
    }
    
    @Override
    public void run() {
        System.out.println("[Spot Monitor] Watching for termination notices...");
        
        while (!shutdownRequested.get()) {
            try {
                if (checkTerminationNotice()) {
                    System.out.println("[Spot Monitor] ⚠️ TERMINATION NOTICE RECEIVED!");
                    System.out.println("[Spot Monitor] Initiating graceful shutdown...");
                    shutdownRequested.set(true);
                    gracefulShutdownAction.run();
                }
                Thread.sleep(5000); // Check every 5 seconds
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    private boolean checkTerminationNotice() {
        try {
            URL url = new URL(TERMINATION_URL);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.setConnectTimeout(1000);
            conn.setReadTimeout(1000);
            conn.setRequestMethod("GET");
            
            // 200 = termination notice exists, 404 = no notice
            return conn.getResponseCode() == 200;
        } catch (Exception e) {
            return false; // Metadata not available or error
        }
    }
    
    // Usage example
    public static void main(String[] args) {
        SpotInstanceHandler handler = new SpotInstanceHandler(() -> {
            // Graceful shutdown actions:
            System.out.println("1. Stop accepting new requests");
            System.out.println("2. Finish in-progress requests");
            System.out.println("3. Deregister from load balancer");
            System.out.println("4. Flush caches to disk");
            System.out.println("5. Signal orchestrator to replace this instance");
        });
        
        new Thread(handler).start();
    }
}
```

---

## Infrastructure Examples

### Terraform — Mixed Pricing Strategy

```hcl
# main.tf — Deploy with a mix of On-Demand, Reserved, and Spot

# Baseline: Reserved capacity (always running)
resource "aws_instance" "baseline" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "m6i.xlarge"
  
  # These will be covered by Reserved Instance purchases
  # (RIs are bought separately in AWS console/CLI)
  
  tags = {
    Name     = "api-baseline-${count.index}"
    Pricing  = "reserved"
    Role     = "api-server"
  }
}

# Elastic: Spot instances for variable load
resource "aws_autoscaling_group" "spot_fleet" {
  name                = "api-spot-fleet"
  min_size            = 0
  max_size            = 20
  desired_capacity    = 3
  vpc_zone_identifier = var.subnet_ids
  
  mixed_instances_policy {
    instances_distribution {
      on_demand_base_capacity                  = 0    # No on-demand in this ASG
      on_demand_percentage_above_base_capacity = 0    # 100% spot
      spot_allocation_strategy                 = "capacity-optimized"
    }
    
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.api.id
      }
      
      # Diversify across instance types for stability
      override {
        instance_type = "m6i.xlarge"
      }
      override {
        instance_type = "m5.xlarge"
      }
      override {
        instance_type = "m5a.xlarge"
      }
      override {
        instance_type = "c6i.xlarge"
      }
    }
  }
  
  tag {
    key                 = "Pricing"
    value               = "spot"
    propagate_at_launch = true
  }
}

# Spike protection: On-demand fallback
resource "aws_autoscaling_group" "on_demand_overflow" {
  name             = "api-on-demand-overflow"
  min_size         = 0
  max_size         = 10
  desired_capacity = 0  # Only scales up when spot can't fulfill
  
  launch_template {
    id = aws_launch_template.api.id
  }
  
  tag {
    key                 = "Pricing"
    value               = "on-demand"
    propagate_at_launch = true
  }
}
```

### Kubernetes — Spot Node Pools with Priority

```yaml
# kube-spot-nodepool.yaml — Mix spot and on-demand nodes in K8s

# Priority class for critical workloads (stays on on-demand nodes)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-workload
value: 1000000
globalDefault: false
description: "Critical pods that must not run on spot instances"

---
# Priority class for fault-tolerant workloads (can use spot)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: spot-tolerant
value: 100
globalDefault: true
description: "Pods that can tolerate spot interruptions"

---
# Deploy critical API on on-demand nodes only
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service  # Critical — never use spot!
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: critical-workload
      nodeSelector:
        node-lifecycle: on-demand  # Only on-demand nodes
      containers:
        - name: payment
          image: myapp/payment:latest
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"

---
# Deploy batch workers on spot nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-processor  # Fault-tolerant — great for spot
spec:
  replicas: 10
  template:
    spec:
      priorityClassName: spot-tolerant
      tolerations:
        - key: "spot"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      nodeSelector:
        node-lifecycle: spot
      containers:
        - name: processor
          image: myapp/image-processor:latest
          resources:
            requests:
              cpu: "4"
              memory: "8Gi"
```

---

## Real-World Example

### How Spotify Saves Millions with Spot Instances

```
┌─────────────────────────────────────────────────────────────┐
│           SPOTIFY'S COST STRATEGY                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Spotify runs on GCP and uses a 3-tier approach:           │
│                                                             │
│  TIER 1: Reserved (Committed Use Discounts)                │
│  ┌─────────────────────────────────────────┐               │
│  │ • Core streaming services                │               │
│  │ • Databases (Bigtable, Spanner)         │               │
│  │ • Always-on infrastructure              │               │
│  │ • ~40% of total compute                 │               │
│  └─────────────────────────────────────────┘               │
│                                                             │
│  TIER 2: Preemptible/Spot VMs                              │
│  ┌─────────────────────────────────────────┐               │
│  │ • Batch processing (data pipelines)     │               │
│  │ • ML model training                     │               │
│  │ • Testing and CI/CD                     │               │
│  │ • ~35% of total compute                 │               │
│  └─────────────────────────────────────────┘               │
│                                                             │
│  TIER 3: On-Demand                                         │
│  ┌─────────────────────────────────────────┐               │
│  │ • Traffic spikes (new album releases)   │               │
│  │ • Emergency scaling                     │               │
│  │ • Short-lived development environments  │               │
│  │ • ~25% of total compute                 │               │
│  └─────────────────────────────────────────┘               │
│                                                             │
│  Result: ~45% reduction in compute costs                   │
│  compared to all on-demand                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | What To Do Instead |
|---------|-------------|-------------------|
| **Buying 3-year RIs for a new service** | If the service is deprecated or refactored, you're stuck paying | Start with 1-year or Convertible RIs for newer services |
| **Running everything on Spot** | Critical services will crash during interruptions | Only use Spot for stateless, fault-tolerant workloads |
| **Ignoring data transfer costs** | Can be 15-25% of bill for data-heavy apps | Use CDN, compress data, keep services co-located |
| **Over-provisioning "just in case"** | You're paying for idle resources 24/7 | Use auto-scaling + right-sizing reviews |
| **Not monitoring unused resources** | Forgotten instances, unattached EBS volumes, idle load balancers | Use AWS Cost Explorer, set up billing alerts |
| **Single AZ spot instances** | One pool runs out, all instances die together | Diversify across AZs and instance types |
| **Ignoring Graviton/ARM instances** | Missing 20-40% savings on compute | Test workloads on ARM — most modern stacks work fine |

---

## When to Use / When NOT to Use

### Reserved Instances / Savings Plans

| Use When | Don't Use When |
|----------|---------------|
| Workload runs 24/7 for 1+ years | Service might be deprecated soon |
| Predictable, steady resource needs | Rapid growth — needs change frequently |
| Databases, caches, core API servers | Dev/test environments |
| You have 6+ months of usage data | New, unproven workloads |

### Spot Instances

| Use When | Don't Use When |
|----------|---------------|
| Batch processing, ETL jobs | Databases or stateful services |
| CI/CD build workers | Single-instance deployments |
| ML training (checkpointing) | Real-time user-facing APIs (single instance) |
| Stateless web workers (with auto-scaling) | Jobs that can't be restarted |
| Big data (Spark, EMR) | Services requiring guaranteed SLA |

### On-Demand

| Use When | Don't Use When |
|----------|---------------|
| Unpredictable, bursty traffic | Steady 24/7 workloads (too expensive) |
| Short-lived workloads (hours/days) | Long-term production baseline |
| Emergency scaling / overflow | Cost is primary concern |
| New services still being validated | You have historical usage data to commit on |

---

## Key Takeaways

1. **Never run everything on On-Demand** — it's the most expensive option. Even a 1-year Savings Plan saves 30%+ with minimal risk.

2. **The optimal mix is typically: 40% Reserved + 30% Spot + 30% On-Demand** — adjust based on your workload's predictability and fault tolerance.

3. **Spot instances can save 60-90% but require architectural readiness** — your app must handle interruptions gracefully (stateless, checkpointing, auto-scaling replacement).

4. **Data transfer is the hidden cloud tax** — architect to minimize cross-region and egress traffic. Use CDNs and VPC endpoints.

5. **Right-size before you reserve** — buying a Reserved Instance for an oversized server locks in waste. Right-size first, then commit.

6. **Use Savings Plans over Standard RIs for flexibility** — unless you're certain about exact instance types for 3 years, Savings Plans give better value.

7. **Monitor and review monthly** — cloud spending drifts. Set up billing alerts at 50%, 80%, and 100% of budget. Review unused resources weekly.

---

## What's Next?

Next, we'll explore **Capacity Planning — Preparing for Growth** — where you'll learn how to forecast future resource needs, plan for traffic growth, and ensure your system can handle 10x or 100x traffic without emergency firefighting.
