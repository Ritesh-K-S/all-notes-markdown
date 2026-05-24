# Cloud Elasticity — AWS Auto Scaling, Azure VMSS, GCP MIG

> **What you'll learn**: How each major cloud provider implements auto scaling at the infrastructure level — AWS Auto Scaling Groups, Azure Virtual Machine Scale Sets, and GCP Managed Instance Groups — their architectures, unique features, pricing models, and how to choose between them for your workload.

---

## Real-Life Analogy — Three Car Rental Companies

Imagine you need rental cars that automatically adjust to demand:

| Cloud Provider | Car Rental Analogy |
|---------------|-------------------|
| **AWS ASG** | The industry pioneer — most rental locations (regions), widest car selection (instance types), most add-on services. Complex pricing menu. |
| **Azure VMSS** | The enterprise favorite — great fleet management, seamless if you already own the brand (Microsoft ecosystem). Best hybrid option (park some cars in your own garage). |
| **GCP MIG** | The tech-savvy option — fastest car delivery, smartest auto-pilot (live migration, preemptible discounts). Simplest pricing. |

All three fundamentally do the same thing: **automatically add/remove virtual machines based on demand.** The differences are in features, integrations, and operational details.

---

## Core Concept Explained Step-by-Step

### Step 1: Shared Architecture — What All Three Have

```
COMMON ARCHITECTURE ACROSS ALL CLOUD PROVIDERS:

┌──────────────────────────────────────────────────────────────────┐
│                    AUTO SCALING ARCHITECTURE                       │
│                                                                  │
│  ┌─────────────────────────────────────────┐                    │
│  │         TEMPLATE / IMAGE                │                    │
│  │  (What each instance looks like)         │                    │
│  │  • OS image (AMI / VM Image / Image)     │                    │
│  │  • Instance size (CPU, RAM)              │                    │
│  │  • Startup script                        │                    │
│  │  • Network/security configuration        │                    │
│  └──────────────────┬──────────────────────┘                    │
│                     │ creates from template                      │
│                     ▼                                            │
│  ┌─────────────────────────────────────────┐                    │
│  │         SCALING GROUP                    │                    │
│  │  (The group of identical instances)      │                    │
│  │  • Min / Desired / Max capacity          │                    │
│  │  • Distribution across zones             │                    │
│  │  • Health check configuration            │                    │
│  │  • Scaling policies                      │                    │
│  └──────────────────┬──────────────────────┘                    │
│                     │ monitored by                               │
│                     ▼                                            │
│  ┌─────────────────────────────────────────┐                    │
│  │         SCALING POLICIES                 │                    │
│  │  (When and how to scale)                 │                    │
│  │  • Target tracking                       │                    │
│  │  • Step scaling                          │                    │
│  │  • Scheduled scaling                     │                    │
│  │  • Predictive scaling                    │                    │
│  └──────────────────┬──────────────────────┘                    │
│                     │ connected to                               │
│                     ▼                                            │
│  ┌─────────────────────────────────────────┐                    │
│  │         LOAD BALANCER                    │                    │
│  │  (Distributes traffic to instances)      │                    │
│  │  • Auto-registers new instances          │                    │
│  │  • Auto-deregisters terminated ones      │                    │
│  │  • Health checking                       │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Step 2: AWS Auto Scaling Groups (ASG)

```
AWS AUTO SCALING GROUP — THE MARKET LEADER:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  KEY COMPONENTS:                                                     │
│                                                                      │
│  Launch Template (what to create):                                   │
│  ├── AMI (Amazon Machine Image)                                     │
│  ├── Instance type(s) — can specify MULTIPLE for flexibility       │
│  ├── Key pair, security groups, IAM role                            │
│  ├── User data (startup script)                                     │
│  └── EBS volumes, network interfaces                                │
│                                                                      │
│  Auto Scaling Group (how to manage):                                │
│  ├── VPC + Subnets (across multiple AZs)                           │
│  ├── Min: 2 / Desired: 4 / Max: 20                                 │
│  ├── Health check: EC2 (basic) or ELB (load balancer-based)        │
│  ├── Target Groups (ALB integration)                                │
│  └── Instance refresh (rolling update strategy)                     │
│                                                                      │
│  UNIQUE AWS FEATURES:                                                │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Mixed Instances Policy                  │                        │
│  │ "Use MULTIPLE instance types"           │                        │
│  │                                         │                        │
│  │ Primary: m5.xlarge (On-Demand)          │                        │
│  │ Fallback: m5a.xlarge, m4.xlarge         │                        │
│  │ Spot: c5.xlarge, c5a.xlarge (70% off!)  │                        │
│  │                                         │                        │
│  │ Split: 30% On-Demand + 70% Spot         │                        │
│  │ If Spot unavailable → fall back to OD   │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Warm Pool                               │                        │
│  │ "Pre-initialized instances waiting"     │                        │
│  │                                         │                        │
│  │ Instances in Stopped state (not billed  │                        │
│  │ for compute, only storage).             │                        │
│  │ When ASG needs to scale out:            │                        │
│  │ START existing warm pool instance       │                        │
│  │ (seconds!) vs launching new (minutes).  │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Predictive Scaling                      │                        │
│  │ "ML-based scaling before traffic comes" │                        │
│  │                                         │                        │
│  │ Analyzes 14 days of historical data.    │                        │
│  │ Predicts next 48 hours of traffic.      │                        │
│  │ Pre-scales capacity BEFORE peak.        │                        │
│  │ Covered in Chapter 7.6.                 │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  SCALING SPEED:                                                      │
│  • New instance launch: 1-3 minutes (from AMI)                      │
│  • Warm pool instance: 10-30 seconds (already initialized)          │
│  • Spot instance: 30-90 seconds (pre-allocated capacity)            │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Step 3: Azure Virtual Machine Scale Sets (VMSS)

```
AZURE VMSS — ENTERPRISE & HYBRID CHAMPION:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  KEY COMPONENTS:                                                     │
│                                                                      │
│  VM Image:                                                           │
│  ├── Azure Marketplace image OR custom image                        │
│  ├── VM size (Standard_D4s_v3, etc.)                                │
│  ├── Managed disks configuration                                    │
│  └── Cloud-init or custom script extension                          │
│                                                                      │
│  Scale Set Configuration:                                            │
│  ├── Availability Zones (spread across 1, 2, or 3 zones)          │
│  ├── Fault domains (within a zone, across racks)                   │
│  ├── Orchestration mode: Uniform (identical) or Flexible (mixed)   │
│  └── Upgrade policy: Automatic, Rolling, or Manual                 │
│                                                                      │
│  UNIQUE AZURE FEATURES:                                              │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Flexible Orchestration                  │                        │
│  │ "Mix different VM sizes in one set"     │                        │
│  │                                         │                        │
│  │ Unlike Uniform (all identical):         │                        │
│  │ ├── Mix D4s, D8s, D16s in same group   │                        │
│  │ ├── Add existing VMs to the set         │                        │
│  │ └── More control over placement         │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Spot VMs with Eviction Policy           │                        │
│  │ "Cheap VMs that might be reclaimed"     │                        │
│  │                                         │                        │
│  │ Eviction: Deallocate or Delete           │                        │
│  │ Max price: Set ceiling (or -1 for market)│                        │
│  │ 60-90% savings vs On-Demand             │                        │
│  │ + Try/Restore: auto-restore when avail  │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Automatic Instance Repair               │                        │
│  │ "Self-healing without manual action"    │                        │
│  │                                         │                        │
│  │ Health probe fails → wait grace period  │                        │
│  │ Still unhealthy? → DELETE and recreate  │                        │
│  │ Fully automatic, no intervention needed │                        │
│  │ Grace period prevents false positives   │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Azure Arc Integration (HYBRID!)         │                        │
│  │ "Scale across cloud AND on-premises"    │                        │
│  │                                         │                        │
│  │ Manage on-prem VMs alongside cloud      │                        │
│  │ Single control plane for everything     │                        │
│  │ Best for enterprises with data centers  │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  SCALING SPEED:                                                      │
│  • New VM allocation: 2-5 minutes                                   │
│  • With custom image + Ephemeral OS disk: 1-2 minutes              │
│  • Spot VM (if capacity available): 30-90 seconds                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Step 4: GCP Managed Instance Groups (MIG)

```
GCP MIG — FAST, SIMPLE, TECH-FORWARD:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  KEY COMPONENTS:                                                     │
│                                                                      │
│  Instance Template:                                                  │
│  ├── Machine type (n2-standard-4, etc.)                             │
│  ├── Boot disk image (from Image Family for auto-updates!)          │
│  ├── Startup script (metadata)                                      │
│  └── Network tags, service account                                  │
│                                                                      │
│  Managed Instance Group:                                             │
│  ├── Regional (multi-zone, recommended) OR Zonal (single zone)     │
│  ├── Target size (desired count)                                    │
│  ├── Autoscaler attached (separate resource)                        │
│  └── Named ports (for load balancer backend)                        │
│                                                                      │
│  UNIQUE GCP FEATURES:                                                │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Live Migration                          │                        │
│  │ "Move VMs without any downtime"         │                        │
│  │                                         │                        │
│  │ GCP can transparently move your VM      │                        │
│  │ to another host for maintenance.        │                        │
│  │ No reboot, no interruption!             │                        │
│  │ Other providers: maintenance = reboot.  │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Preemptible / Spot VMs                  │                        │
│  │ "60-91% cheaper, max 24hr lifetime"     │                        │
│  │                                         │                        │
│  │ Can be reclaimed with 30s notice.       │                        │
│  │ Perfect for batch, ML training, CI/CD.  │                        │
│  │ Spot VMs: same discount, no 24hr limit. │                        │
│  │ MIG auto-replaces terminated Spots.     │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Per-Instance Configuration              │                        │
│  │ "Customize individual instances"        │                        │
│  │                                         │                        │
│  │ Apply specific metadata or disk configs │                        │
│  │ to individual instances in the group.   │                        │
│  │ Useful for sharded databases,           │                        │
│  │ partition-assigned workers.             │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  ┌────────────────────────────────────────┐                        │
│  │ Autoscaler Modes                        │                        │
│  │ "ON / OFF / ONLY_SCALE_OUT"             │                        │
│  │                                         │                        │
│  │ ONLY_SCALE_OUT: Will add instances but  │                        │
│  │ never remove them automatically.        │                        │
│  │ Perfect for: stateful workloads where   │                        │
│  │ you want to grow but control shrinking. │                        │
│  └────────────────────────────────────────┘                        │
│                                                                      │
│  SCALING SPEED:                                                      │
│  • Standard instance: 20-90 seconds (fastest of the three!)        │
│  • With Container-Optimized OS: 30-60 seconds                      │
│  • Spot VM: 10-30 seconds (pre-allocated pools)                    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Step 5: Head-to-Head Comparison

```
FEATURE COMPARISON TABLE:

┌─────────────────────────┬──────────────┬──────────────┬──────────────┐
│ Feature                 │ AWS ASG      │ Azure VMSS   │ GCP MIG      │
├─────────────────────────┼──────────────┼──────────────┼──────────────┤
│ Max instances per group │ 500 (soft)   │ 1000         │ 2000         │
│ Scale-out speed         │ 1-3 min      │ 2-5 min      │ 20-90 sec    │
│ Scale-to-zero           │ Yes (min=0)  │ Yes (min=0)  │ Yes (min=0)  │
│ Mixed instance types    │ Yes (policy) │ Yes (flex)   │ No (1 template│
│ Warm pool              │ Yes ✓        │ No           │ No            │
│ Predictive scaling      │ Yes (ML)     │ Yes (preview)│ No (manual)   │
│ Spot/preemptible        │ Yes (70% off)│ Yes (90% off)│ Yes (91% off) │
│ Live migration         │ No           │ Limited      │ Yes ✓         │
│ Hybrid (on-prem)       │ Outposts     │ Arc ✓        │ Anthos        │
│ Container-native        │ ECS/EKS     │ ACI/AKS      │ GKE (best)   │
│ Rolling updates         │ Instance     │ Rolling/Auto │ Proactive     │
│                         │ refresh      │              │ updater       │
│ Metrics integration     │ CloudWatch   │ Azure Monitor│ Cloud         │
│                         │              │              │ Monitoring    │
│ Scheduling support      │ Yes ✓        │ Yes ✓        │ Yes ✓         │
│ Instance protection     │ Yes          │ Yes          │ No (use       │
│                         │              │              │ taints)       │
└─────────────────────────┴──────────────┴──────────────┴──────────────┘
```

---

## How It Works Internally

### Scaling Decision Flow (All Providers)

```
INTERNAL SCALING FLOW (simplified):

┌───────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Every 30-60 seconds:                                            │
│                                                                   │
│  ┌────────────────┐                                              │
│  │ Metrics Service│  CloudWatch / Azure Monitor / Cloud Monitoring│
│  │   collects     │                                              │
│  └───────┬────────┘                                              │
│          │ aggregated metrics                                    │
│          ▼                                                        │
│  ┌────────────────┐                                              │
│  │  Scaling Engine│  Evaluates policies against metrics          │
│  │   evaluates    │                                              │
│  └───────┬────────┘                                              │
│          │ desired capacity change                               │
│          ▼                                                        │
│  ┌────────────────┐                                              │
│  │  Capacity      │  Checks zone balance, Spot availability,    │
│  │  Planner       │  instance type inventory                    │
│  └───────┬────────┘                                              │
│          │ placement decisions                                   │
│          ▼                                                        │
│  ┌────────────────┐                                              │
│  │  Instance      │  Calls compute API to create/terminate       │
│  │  Lifecycle     │  instances, manages startup/shutdown         │
│  └───────┬────────┘                                              │
│          │ instance ready                                        │
│          ▼                                                        │
│  ┌────────────────┐                                              │
│  │  Load Balancer │  Registers new instance, starts health      │
│  │  Registration  │  checks, begins routing traffic             │
│  └────────────────┘                                              │
│                                                                   │
│  ZONE BALANCING:                                                 │
│  All providers distribute instances evenly across zones.         │
│                                                                   │
│  6 instances across 3 zones:                                     │
│  Zone A: ██  (2 instances)                                       │
│  Zone B: ██  (2 instances)                                       │
│  Zone C: ██  (2 instances)                                       │
│                                                                   │
│  If Zone A fails (entire data center down):                      │
│  Zone B: ████ (2 + 2 new instances)                              │
│  Zone C: ████ (2 + 2 new instances)                              │
│  Auto scaling immediately compensates!                           │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Instance Replacement Strategies

```
ROLLING UPDATE STRATEGIES:

AWS Instance Refresh:
┌────────────────────────────────────────────────────────────┐
│  Strategy: Replace 25% at a time                           │
│  Health check: Wait until instance passes LB health check  │
│                                                            │
│  Phase 1: ████████████████  (16 instances, old version)   │
│  Phase 2: ████████████░░░░  (12 old + 4 new launching)   │
│  Phase 3: ████████░░░░████  (8 old + 4 ready + 4 new)    │
│  Phase 4: ████░░░░████████  (4 old + 12 new)             │
│  Phase 5: ░░░░████████████  (0 old + 16 new, complete!)  │
└────────────────────────────────────────────────────────────┘

Azure Rolling Upgrade:
┌────────────────────────────────────────────────────────────┐
│  maxBatchInstancePercent: 20%                              │
│  pauseTimeBetweenBatches: PT60S (1 minute)               │
│  maxUnhealthyUpgradedInstancePercent: 20%                 │
│                                                            │
│  Rolls batch by batch, pauses between batches.            │
│  If too many unhealthy: STOPS the rollout!               │
└────────────────────────────────────────────────────────────┘

GCP Proactive Update:
┌────────────────────────────────────────────────────────────┐
│  type: PROACTIVE (auto-replace) or OPPORTUNISTIC (manual) │
│  maxSurge: 3 (extra instances during update)              │
│  maxUnavailable: 0 (never reduce below desired)           │
│                                                            │
│  Creates new instances FIRST, then removes old.           │
│  Zero-downtime by default!                                │
└────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Multi-Cloud Auto Scaling Manager

```python
# multi_cloud_scaler.py — Unified interface for multi-cloud auto scaling
from abc import ABC, abstractmethod
import boto3  # AWS

class AutoScalerBase(ABC):
    """Common interface for all cloud auto scaling operations."""
    
    @abstractmethod
    def get_current_capacity(self) -> dict:
        pass
    
    @abstractmethod
    def set_desired_capacity(self, count: int):
        pass
    
    @abstractmethod
    def get_scaling_activities(self) -> list:
        pass


class AWSAutoScaler(AutoScalerBase):
    """AWS Auto Scaling Group manager."""
    
    def __init__(self, asg_name: str, region: str = 'us-east-1'):
        self.client = boto3.client('autoscaling', region_name=region)
        self.asg_name = asg_name
    
    def get_current_capacity(self) -> dict:
        response = self.client.describe_auto_scaling_groups(
            AutoScalingGroupNames=[self.asg_name])
        asg = response['AutoScalingGroups'][0]
        return {
            'min': asg['MinSize'],
            'desired': asg['DesiredCapacity'],
            'max': asg['MaxSize'],
            'current': len([i for i in asg['Instances'] 
                          if i['LifecycleState'] == 'InService']),
        }
    
    def set_desired_capacity(self, count: int):
        self.client.set_desired_capacity(
            AutoScalingGroupName=self.asg_name,
            DesiredCapacity=count,
            HonorCooldown=True  # Respect cooldown periods
        )
    
    def get_scaling_activities(self) -> list:
        response = self.client.describe_scaling_activities(
            AutoScalingGroupName=self.asg_name, MaxRecords=10)
        return [{'time': a['StartTime'], 'cause': a['Cause'],
                 'status': a['StatusCode']} for a in response['Activities']]
    
    def configure_warm_pool(self, min_size: int = 2):
        """Pre-initialize instances for faster scale-out."""
        self.client.put_warm_pool(
            AutoScalingGroupName=self.asg_name,
            MinSize=min_size,
            PoolState='Stopped',  # Not billed for compute!
        )
        print(f"Warm pool configured: {min_size} pre-initialized instances")


# Usage:
scaler = AWSAutoScaler('web-production-asg')
capacity = scaler.get_current_capacity()
print(f"Current: {capacity['current']}/{capacity['max']} instances")
# If manual intervention needed (e.g., before a big event):
# scaler.set_desired_capacity(20)
```

### Java — Azure VMSS Management

```java
// AzureVmssManager.java — Manage Azure Virtual Machine Scale Sets
import com.azure.resourcemanager.AzureResourceManager;
import com.azure.resourcemanager.compute.models.*;
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.core.management.profile.AzureProfile;
import com.azure.core.management.AzureEnvironment;

public class AzureVmssManager {
    
    private final AzureResourceManager azure;
    private final String resourceGroup;
    private final String vmssName;
    
    public AzureVmssManager(String subscriptionId, String resourceGroup, 
                           String vmssName) {
        this.azure = AzureResourceManager.authenticate(
            new DefaultAzureCredentialBuilder().build(),
            new AzureProfile(AzureEnvironment.AZURE))
            .withSubscription(subscriptionId);
        this.resourceGroup = resourceGroup;
        this.vmssName = vmssName;
    }
    
    public void printCapacity() {
        VirtualMachineScaleSet vmss = azure.virtualMachineScaleSets()
            .getByResourceGroup(resourceGroup, vmssName);
        
        System.out.printf("VMSS: %s%n", vmss.name());
        System.out.printf("  Capacity: %d instances%n", vmss.capacity());
        System.out.printf("  SKU: %s%n", vmss.sku().name());
        
        // List instance health
        vmss.virtualMachines().list().forEach(vm -> {
            System.out.printf("  Instance %s: %s%n",
                vm.instanceId(), vm.powerState());
        });
    }
    
    public void scaleOut(int newCapacity) {
        VirtualMachineScaleSet vmss = azure.virtualMachineScaleSets()
            .getByResourceGroup(resourceGroup, vmssName);
        
        if (newCapacity > vmss.capacity()) {
            vmss.update()
                .withCapacity(newCapacity)
                .apply();
            System.out.printf("Scaled from %d → %d instances%n",
                vmss.capacity(), newCapacity);
        }
    }
    
    public void configureAutoScale() {
        // Azure auto scale is configured via Monitor Autoscale Settings
        // This is a conceptual representation of the API call
        azure.autoscaleSettings().define("web-autoscale")
            .withRegion("eastus")
            .withExistingResourceGroup(resourceGroup)
            .withTargetResource(getVmssResourceId())
            .defineAutoscaleProfile("default-profile")
                .withMetricBasedScale(2, 20, 4)  // min, max, default
                .defineScaleRule()
                    .withMetricSource(getVmssResourceId())
                    .withMetricName("Percentage CPU")
                    .withTimeGrain(java.time.Duration.ofMinutes(1))
                    .withStatistic(MetricStatisticType.AVERAGE)
                    .withCondition(TimeAggregationType.AVERAGE,
                        ComparisonOperationType.GREATER_THAN, 70)
                    .withScaleAction(ScaleDirection.INCREASE, ScaleType.CHANGE_COUNT, 
                        2, java.time.Duration.ofMinutes(5))
                    .attach()
                .attach()
            .create();
    }
    
    private String getVmssResourceId() {
        return String.format("/subscriptions/%s/resourceGroups/%s" +
            "/providers/Microsoft.Compute/virtualMachineScaleSets/%s",
            azure.subscriptionId(), resourceGroup, vmssName);
    }
}
```

---

## Infrastructure Examples

### AWS — Production ASG with Mixed Instances & Warm Pool

```hcl
# aws-production-asg.tf

resource "aws_autoscaling_group" "production" {
  name                = "web-production"
  vpc_zone_identifier = var.private_subnet_ids
  min_size            = 4
  max_size            = 40
  desired_capacity    = 8

  # Mixed instances: On-Demand base + Spot for savings
  mixed_instances_policy {
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.web.id
        version            = "$Latest"
      }
      override {
        instance_type = "m5.xlarge"
      }
      override {
        instance_type = "m5a.xlarge"  # AMD alternative (cheaper)
      }
      override {
        instance_type = "m6i.xlarge"  # Newer gen
      }
    }

    instances_distribution {
      on_demand_base_capacity                  = 4   # First 4 are always On-Demand
      on_demand_percentage_above_base_capacity = 25  # Then 25% OD, 75% Spot
      spot_allocation_strategy                 = "capacity-optimized"
    }
  }

  # Warm pool for faster scaling
  warm_pool {
    pool_state                  = "Stopped"
    min_size                    = 4
    max_group_prepared_capacity = 10
    instance_reuse_policy {
      reuse_on_scale_in = true  # Return to warm pool instead of terminating
    }
  }

  # Health checking
  health_check_type         = "ELB"
  health_check_grace_period = 300

  # Instance refresh for zero-downtime deployments
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 75
      instance_warmup        = 180
    }
  }

  target_group_arns = [aws_lb_target_group.web.arn]
}
```

### Azure — VMSS with Auto-Repair and Spot VMs

```hcl
# azure-vmss.tf

resource "azurerm_linux_virtual_machine_scale_set" "web" {
  name                = "web-vmss"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard_D4s_v3"
  instances           = 4
  
  zones              = ["1", "2", "3"]  # Spread across 3 AZs
  zone_balance       = true

  # Spot configuration for cost savings
  priority        = "Spot"
  eviction_policy = "Deallocate"  # Keep disk, just stop VM
  max_bid_price   = 0.08         # Max price per hour

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  # Auto-repair unhealthy instances
  automatic_instance_repair {
    enabled      = true
    grace_period = "PT30M"  # 30 minute grace before considering unhealthy
  }

  # Rolling upgrades
  upgrade_mode = "Rolling"
  rolling_upgrade_policy {
    max_batch_instance_percent              = 20
    max_unhealthy_instance_percent          = 20
    max_unhealthy_upgraded_instance_percent = 5
    pause_time_between_batches              = "PT60S"
  }

  # Health probe
  health_probe_id = azurerm_lb_probe.web.id
}

# Autoscale settings
resource "azurerm_monitor_autoscale_setting" "web" {
  name                = "web-autoscale"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  target_resource_id  = azurerm_linux_virtual_machine_scale_set.web.id

  profile {
    name = "default"
    capacity {
      default = 4
      minimum = 2
      maximum = 40
    }

    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_linux_virtual_machine_scale_set.web.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        time_aggregation   = "Average"
        operator           = "GreaterThan"
        threshold          = 70
      }
      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "2"
        cooldown  = "PT5M"
      }
    }
  }
}
```

### GCP — Regional MIG with Autoscaler

```hcl
# gcp-mig.tf

resource "google_compute_instance_template" "web" {
  name_prefix  = "web-server-"
  machine_type = "n2-standard-4"

  disk {
    source_image = "projects/my-project/global/images/family/web-server"
    auto_delete  = true
    boot         = true
    disk_type    = "pd-ssd"
  }

  network_interface {
    subnetwork = google_compute_subnetwork.private.id
  }

  metadata_startup_script = <<-EOF
    #!/bin/bash
    docker run -d -p 8080:8080 gcr.io/my-project/web-app:latest
  EOF

  service_account {
    scopes = ["cloud-platform"]
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Regional MIG — automatically distributes across zones
resource "google_compute_region_instance_group_manager" "web" {
  name               = "web-mig"
  region             = "us-central1"
  base_instance_name = "web"
  target_size        = 4

  version {
    instance_template = google_compute_instance_template.web.id
  }

  named_port {
    name = "http"
    port = 8080
  }

  # Zero-downtime updates
  update_policy {
    type                  = "PROACTIVE"
    minimal_action        = "REPLACE"
    max_surge_fixed       = 3
    max_unavailable_fixed = 0  # Never go below desired count
  }

  # Auto-healing
  auto_healing_policies {
    health_check      = google_compute_health_check.web.id
    initial_delay_sec = 300
  }

  distribution_policy_zones = [
    "us-central1-a", "us-central1-b", "us-central1-c"
  ]
}

# Autoscaler
resource "google_compute_region_autoscaler" "web" {
  name   = "web-autoscaler"
  region = "us-central1"
  target = google_compute_region_instance_group_manager.web.id

  autoscaling_policy {
    min_replicas    = 2
    max_replicas    = 40
    cooldown_period = 60

    cpu_utilization {
      target = 0.6  # 60% CPU target
    }

    # Also scale on load balancing utilization
    load_balancing_utilization {
      target = 0.8  # 80% backend utilization
    }

    # Scale-in controls (be conservative)
    scale_in_control {
      max_scaled_in_replicas {
        fixed = 2  # Remove max 2 instances at a time
      }
      time_window_sec = 600  # Over 10-minute windows
    }
  }
}
```

---

## Real-World Example

### Multi-Cloud Strategy — How Shopify Handles Black Friday

```
SHOPIFY'S BLACK FRIDAY SCALING STRATEGY:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  THE CHALLENGE:                                                      │
│  • Powers 2M+ online stores worldwide                               │
│  • Black Friday/Cyber Monday: traffic surges 4-10x in minutes      │
│  • $7.5 BILLION in sales over one weekend                           │
│  • Zero downtime tolerance (merchants lose money per second)        │
│                                                                      │
│  THE SOLUTION: Multi-cloud with aggressive pre-scaling             │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ PRIMARY: GCP (GKE + MIG)                                 │      │
│  │ • Kubernetes for microservices (HPA + Cluster Autoscaler)│      │
│  │ • MIGs for stateful components (MySQL, Redis)            │      │
│  │ • Fast instance provisioning (20-90 seconds)             │      │
│  │ • Regional MIGs for multi-zone HA                        │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ BFCM STRATEGY (3 weeks before):                          │      │
│  │ 1. Double baseline capacity (scheduled scaling)          │      │
│  │ 2. Pre-warm connection pools, caches, DNS                │      │
│  │ 3. Set max capacity 10x normal (remove ceiling)          │      │
│  │ 4. Reduce cooldown periods (react faster)                │      │
│  │ 5. Lower scaling thresholds (scale earlier)              │      │
│  │ 6. Enable "flash sale mode" (disable non-critical)      │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ DURING BFCM:                                              │      │
│  │ • War room with 100+ engineers monitoring                │      │
│  │ • Auto scaling handles 80% of adjustments                │      │
│  │ • Manual overrides available for emergencies             │      │
│  │ • Per-merchant throttling if needed (fairness)           │      │
│  │ • CDN absorbs 80%+ of read traffic                       │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  AFTER BFCM (gradual scale-down over 1 week):                      │
│  • Day 1-2: Reduce to 3x baseline (Cyber Monday, returns)         │
│  • Day 3-4: Reduce to 2x baseline (lingering sales)               │
│  • Day 5-7: Back to normal baseline                                │
│                                                                      │
│  RESULT: Zero downtime across 4 consecutive Black Fridays          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Single availability zone | Zone failure = complete outage | ALWAYS use multi-AZ (regional MIG, multi-subnet ASG) |
| 100% Spot/Preemptible instances | Mass eviction during capacity crunch = total outage | Use mixed: 30% On-Demand base + 70% Spot |
| Not testing instance startup time | Assumed 30 seconds, actually takes 5 minutes with app warmup | Measure end-to-end: launch → healthy → serving traffic |
| Same instance type only | If that type runs out of capacity, can't scale | Specify multiple instance types (Mixed Instances in AWS, same family alternatives) |
| No rolling update strategy | New AMI deployment = replace all at once = downtime | Configure rolling updates (25% batches, health checks between) |
| Ignoring Spot interruption handling | Instance terminated mid-request = user error | Handle SIGTERM gracefully, drain connections, use interruption notices |
| Max capacity = desired capacity | Can't scale out during spikes! | Set max 4-10x desired; use billing alerts for cost control |

---

## When to Use / When NOT to Use

### Decision Matrix: Which Cloud Provider for Auto Scaling?

```
CHOOSE AWS ASG IF:
├── Already invested in AWS ecosystem
├── Need Warm Pools for sub-minute scaling
├── Want Predictive Scaling (ML-based)
├── Require mixed On-Demand + Spot in same group
├── Need widest selection of instance types
└── Enterprise with complex networking (VPC, Transit Gateway)

CHOOSE AZURE VMSS IF:
├── Microsoft/Windows/.NET workloads
├── Hybrid cloud (Azure Arc for on-premises)
├── Enterprise with Active Directory requirements
├── Using Azure DevOps for CI/CD
├── Need Flexible orchestration (mix VM sizes)
└── Want Automatic Instance Repair built-in

CHOOSE GCP MIG IF:
├── Need fastest instance provisioning (20-90 sec)
├── Running Kubernetes-native workloads (GKE is best-in-class)
├── Want Live Migration (zero maintenance downtime)
├── Cost-sensitive (sustained use discounts automatic)
├── Prefer simplicity over feature count
└── Data/ML workloads (TPU access, BigQuery integration)
```

---

## Key Takeaways

1. **All three clouds solve the same problem** — automatically scaling compute resources based on demand. The fundamentals (template + group + policy + load balancer) are identical.

2. **AWS ASG is the most feature-rich** — warm pools, predictive scaling, mixed instances policy. Best for complex enterprise workloads needing every knob.

3. **Azure VMSS excels at hybrid** — Azure Arc integration, automatic repair, and Flexible orchestration make it best for enterprises bridging on-prem and cloud.

4. **GCP MIG is fastest and simplest** — 20-90 second provisioning, live migration, and straightforward autoscaler configuration. Best for greenfield cloud-native apps.

5. **Mix Spot and On-Demand** — never go 100% Spot for production. 30% On-Demand base + 70% Spot gives best cost-to-reliability ratio.

6. **Multi-zone is mandatory** — single zone = single point of failure. All providers support cross-zone distribution; always enable it.

7. **Measure total scale-out time** — from trigger to serving traffic includes: decision time + instance launch + OS boot + app startup + health check pass. Plan for the full duration.

---

## What's Next?

You've mastered cloud-provider auto scaling. But what about scaling BEFORE traffic arrives? In **Chapter 7.6: Predictive & Scheduled Scaling**, we'll explore how machine learning predicts traffic patterns 48 hours in advance, how scheduled scaling handles known events, and how pre-warming ensures your system is ready before the storm hits.
