# Kubernetes Autoscaling — HPA, VPA & Cluster Autoscaler

> **What you'll learn**: How Kubernetes scales at three distinct levels — scaling the number of Pods (HPA), scaling the resources of individual Pods (VPA), and scaling the underlying Nodes (Cluster Autoscaler) — plus how KEDA brings event-driven scaling to Kubernetes workloads.

---

## Real-Life Analogy — A Restaurant Kitchen

Think of a Kubernetes cluster as a restaurant:

| Kubernetes | Restaurant |
|-----------|-----------|
| **Node** | A kitchen station (stove + counter space) |
| **Pod** | A chef working at a station |
| **HPA** (Horizontal Pod Autoscaler) | Hiring more chefs when orders pile up |
| **VPA** (Vertical Pod Autoscaler) | Giving each chef a bigger station with better tools |
| **Cluster Autoscaler** | Building more kitchen stations when all are occupied |

```
SCALING LEVELS IN KUBERNETES:

Level 3: CLUSTER AUTOSCALER
"Need more physical machines (Nodes)"
┌───────────────────────────────────────────────────────────────┐
│  Node 1          Node 2          Node 3 (NEW!)               │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                │
│  │          │   │          │   │          │                │
│  │ Level 2: VPA changes pod resource requests/limits         │
│  │          │   │          │   │          │                │
│  │ ┌──┐┌──┐│   │ ┌──┐┌──┐│   │ ┌──┐     │                │
│  │ │P1││P2││   │ │P3││P4││   │ │P5│     │                │
│  │ └──┘└──┘│   │ └──┘└──┘│   │ └──┘     │  ← Level 1:    │
│  │          │   │          │   │          │    HPA adds     │
│  └──────────┘   └──────────┘   └──────────┘    more Pods   │
│                                                              │
└───────────────────────────────────────────────────────────────┘
     ↑ Level 3 added this Node because Pods couldn't fit!
```

---

## Core Concept Explained Step-by-Step

### Step 1: HPA — Horizontal Pod Autoscaler

```
HPA: "Add more Pods when existing Pods are overloaded"

HOW HPA WORKS:

1. HPA controller runs every 15 seconds (configurable)
2. Queries Metrics Server for current Pod resource usage
3. Calculates desired replica count
4. Updates Deployment's replica count

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  HPA Formula:                                                    │
│                                                                  │
│  desiredReplicas = ceil[                                         │
│    currentReplicas × (currentMetric / desiredMetric)            │
│  ]                                                               │
│                                                                  │
│  Example:                                                        │
│  • Current replicas: 3                                          │
│  • Current CPU (avg): 75%                                       │
│  • Target CPU: 50%                                              │
│  • Desired = ceil(3 × 75/50) = ceil(4.5) = 5                   │
│  • Action: Scale from 3 → 5 Pods                                │
│                                                                  │
│  CONTROL LOOP:                                                   │
│                                                                  │
│  ┌────────────┐   query    ┌────────────────┐                   │
│  │   HPA      │──────────▶│ Metrics Server │                   │
│  │ Controller │◀──────────│ (or Prometheus)│                   │
│  └─────┬──────┘   metrics  └────────────────┘                   │
│        │                                                         │
│        │ scale                                                   │
│        ▼                                                         │
│  ┌────────────┐   creates/   ┌────────────────┐                │
│  │ Deployment │──deletes───▶│ Pods (replicas)│                │
│  │ (spec)     │              │  P1  P2  P3... │                │
│  └────────────┘              └────────────────┘                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**HPA Scaling Behavior Configuration:**

```
SCALE-UP vs SCALE-DOWN BEHAVIOR:

Kubernetes 1.18+ allows fine-grained control:

Scale UP (react quickly to load):
├── Stabilization window: 0 seconds (scale immediately)
├── Policies: Add up to 4 Pods OR 100% of current, every 15s
└── Select: Max (use most aggressive policy)

Scale DOWN (be conservative to avoid flapping):
├── Stabilization window: 300 seconds (wait 5 min)
├── Policies: Remove max 1 Pod every 60 seconds
└── Select: Min (use least aggressive policy)

WHY ASYMMETRIC?
┌────────────────────────────────────────────────────────────┐
│  Scale UP fast:                                            │
│  - Users are waiting right now! Every second matters.      │
│  - Over-provisioning briefly costs little.                 │
│                                                            │
│  Scale DOWN slow:                                          │
│  - Removing too fast → traffic returns → scale up again    │
│  - "Flapping" = worse than keeping an extra Pod            │
│  - Stabilization window: only scale down if metric was     │
│    CONSISTENTLY low for the entire window                  │
└────────────────────────────────────────────────────────────┘
```

### Step 2: VPA — Vertical Pod Autoscaler

```
VPA: "Right-size Pod resource requests and limits"

THE PROBLEM VPA SOLVES:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  When you first deploy, you GUESS resource requirements:      │
│                                                                │
│  resources:                                                    │
│    requests:                                                   │
│      cpu: "500m"      ← Is this right? Who knows!            │
│      memory: "256Mi"  ← Too much? Too little?                │
│    limits:                                                     │
│      cpu: "1000m"                                             │
│      memory: "512Mi"                                          │
│                                                                │
│  TOO HIGH: Wasting resources (nodes look full but aren't)    │
│  TOO LOW: Pods get OOM killed or CPU throttled               │
│                                                                │
│  VPA watches actual usage over time and adjusts:             │
│  "This Pod uses 120m CPU on average, peak 300m.              │
│   Recommend: request=200m, limit=400m"                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘

VPA MODES:
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  1. "Off" (Recommend only)                                   │
│     └── VPA calculates recommendations, doesn't apply them  │
│     └── You review and manually update your Deployment      │
│     └── SAFEST — start here!                                │
│                                                              │
│  2. "Initial" (Apply to new Pods only)                       │
│     └── When Pods are created, use VPA recommendations      │
│     └── Existing Pods are NOT restarted                     │
│     └── Good for initial rollouts                           │
│                                                              │
│  3. "Auto" (Full control)                                    │
│     └── VPA evicts Pods that are incorrectly sized          │
│     └── Pod restarts with new resource settings             │
│     └── CAUTION: Causes Pod restarts! Need good disruption  │
│         budgets (PDBs) to avoid downtime                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘

VPA WORKFLOW:
┌──────────┐  observes   ┌───────────┐  recommends  ┌──────────┐
│ Pod with │────────────▶│    VPA    │─────────────▶│ Updated  │
│ cpu: 500m│             │Recommender│              │ cpu: 200m│
│ mem: 256M│             │           │              │ mem: 128M│
└──────────┘             └───────────┘              └──────────┘
     ↑                                                    │
     └────── evicts & recreates with new resources ◀──────┘
                    (in "Auto" mode)
```

### Step 3: Cluster Autoscaler — Node Level Scaling

```
CLUSTER AUTOSCALER: "Add/remove physical machines (Nodes)"

WHEN DOES CLUSTER AUTOSCALER ACT?

Scale UP scenario:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  HPA says: "I need 10 Pods of web-server"                    │
│  But only 6 can fit on existing Nodes!                        │
│                                                                │
│  Node 1 (8GB RAM):              Node 2 (8GB RAM):            │
│  ┌──────────────────────┐      ┌──────────────────────┐     │
│  │ Pod1(2GB) Pod2(2GB)  │      │ Pod3(2GB) Pod4(2GB)  │     │
│  │ Pod5(2GB) Pod6(2GB)  │      │ system(2GB) ░░FULL░░ │     │
│  │ system(1GB) ░░FULL░░ │      └──────────────────────┘     │
│  └──────────────────────┘                                    │
│                                                                │
│  Scheduler: "Pod 7, 8, 9, 10 are PENDING — no room!"        │
│  Cluster Autoscaler sees Pending Pods → adds Node 3          │
│                                                                │
│  Node 3 (8GB RAM) — NEWLY ADDED:                             │
│  ┌──────────────────────┐                                    │
│  │ Pod7(2GB) Pod8(2GB)  │                                    │
│  │ Pod9(2GB) Pod10(2GB) │                                    │
│  │ system(1GB)  ░░OK░░  │  ← All Pods now scheduled!        │
│  └──────────────────────┘                                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘

Scale DOWN scenario:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Traffic dropped. HPA scaled Pods from 10 → 4.               │
│  Node 3 has no Pods (or only moveable Pods).                  │
│                                                                │
│  Cluster Autoscaler checks (every 10 seconds):               │
│  "Is this Node underutilized for > 10 minutes?"             │
│  "Can all its Pods be moved to other Nodes?"                 │
│                                                                │
│  If YES to both:                                              │
│  1. Cordon the Node (prevent new scheduling)                 │
│  2. Drain the Node (gracefully evict all Pods)               │
│  3. Terminate the Node (return to cloud provider)            │
│                                                                │
│  CONSTRAINTS — Cluster Autoscaler will NOT remove a Node if: │
│  ├── It has Pods with local storage                          │
│  ├── It has Pods not managed by a controller                 │
│  ├── It has Pods with restrictive PodDisruptionBudget        │
│  └── It was scaled up less than 10 minutes ago               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Step 4: How All Three Work Together

```
THE THREE AUTOSCALERS IN CONCERT:

Traffic Spike Scenario (10x normal):

Time 0:00 — Traffic increases
├── HPA detects: CPU avg 80% (target: 50%)
├── HPA decides: Scale from 5 Pods → 8 Pods
│
Time 0:15 — HPA acts
├── Scheduler tries to place 3 new Pods
├── Nodes are FULL — Pods go to "Pending" state
│
Time 0:30 — Cluster Autoscaler acts
├── Detects Pending Pods
├── Calculates: need 2 more Nodes to fit them
├── Requests 2 new Nodes from cloud provider
│
Time 2:00 — New Nodes ready
├── Nodes join cluster, pass health checks
├── Scheduler places Pending Pods on new Nodes
│
Time 2:30 — Pods running
├── 8 Pods serving traffic across 4 Nodes
├── CPU avg drops to 50% — HPA is satisfied
│
Time later — Traffic decreases
├── HPA scales Pods: 8 → 3
├── Cluster Autoscaler: Nodes 3 & 4 are empty
├── After 10 min idle: removes extra Nodes
│
TIMELINE:
│ 0s         15s        30s        2min       2.5min
│  │          │          │          │          │
│  ▼          ▼          ▼          ▼          ▼
│ Traffic  HPA adds   CA adds    Nodes      Pods
│ spike    Pods       Nodes      ready      serving
│                     (pending                 
│                      Pods)                   
│
│ Total time from spike to relief: ~2.5 minutes
│ (Most time spent waiting for Nodes to boot!)
│
```

### Step 5: KEDA — Event-Driven Autoscaling

```
KEDA (Kubernetes Event-Driven Autoscaling):

Standard HPA limitation:
├── Only scales on CPU, memory, or custom metrics API
├── Minimum replica = 1 (can't scale to zero!)
└── Complex to set up for external metrics

KEDA extends HPA:
├── Scale on external events (Kafka lag, SQS queue, Redis streams)
├── Scale TO ZERO (huge cost savings for bursty workloads!)
└── 60+ built-in scalers for popular systems

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  WITHOUT KEDA:                                                   │
│  Queue empty → 1 Pod sitting idle (minimum 1) → paying for it  │
│                                                                  │
│  WITH KEDA:                                                      │
│  Queue empty → 0 Pods! → no cost!                               │
│  Message arrives → KEDA triggers → 1 Pod spins up → processes  │
│  Queue grows → KEDA scales → 10 Pods processing in parallel    │
│  Queue empty again → KEDA scales to 0 → back to no cost       │
│                                                                  │
│  KEDA Architecture:                                              │
│  ┌──────────────┐      ┌──────────────┐     ┌──────────┐      │
│  │ Event Source │      │    KEDA      │     │   HPA    │      │
│  │(Kafka/SQS/  │─────▶│  Operator    │────▶│(standard)│      │
│  │ Redis/etc)  │      │              │     │          │      │
│  └──────────────┘      └──────────────┘     └────┬─────┘      │
│                                                   │             │
│                                              scales│             │
│                                                   ▼             │
│                                           ┌──────────────┐     │
│                                           │  Deployment  │     │
│                                           │  (0→N Pods)  │     │
│                                           └──────────────┘     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Metrics Pipeline in Kubernetes

```
HOW METRICS FLOW FROM POD TO HPA:

Layer 1: Resource Metrics (CPU, Memory)
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Pod (cAdvisor) → Kubelet → Metrics Server → HPA Controller   │
│                                                                 │
│  cAdvisor: embedded in kubelet, tracks container resource usage │
│  Metrics Server: aggregates metrics from all kubelets           │
│  HPA: queries Metrics Server every 15 seconds                  │
│                                                                 │
│  API: metrics.k8s.io/v1beta1                                   │
│  kubectl top pods → uses this same path                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Layer 2: Custom Metrics (app-specific)
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  App → Prometheus → Prometheus Adapter → HPA Controller        │
│                                                                 │
│  App exposes /metrics endpoint (requests_per_second, etc.)     │
│  Prometheus scrapes and stores                                  │
│  Prometheus Adapter exposes as custom.metrics.k8s.io API       │
│  HPA queries this API for scaling decisions                    │
│                                                                 │
│  API: custom.metrics.k8s.io/v1beta1                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Layer 3: External Metrics (outside cluster)
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  External System (SQS/CloudWatch) → External Adapter → HPA    │
│                                                                 │
│  Metrics from systems outside the cluster                      │
│  (cloud queue depths, external API metrics)                    │
│  KEDA provides adapters for 60+ external systems               │
│                                                                 │
│  API: external.metrics.k8s.io/v1beta1                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Cluster Autoscaler Decision Logic

```
CLUSTER AUTOSCALER INTERNALS:

Every 10 seconds, the Cluster Autoscaler runs:

┌──────────────────────────────────────────────────────────────────┐
│ SCALE UP CHECK:                                                  │
│                                                                  │
│ 1. Find all "Unschedulable" (Pending) Pods                      │
│ 2. For each Node Group, simulate: "Would this Node type fit     │
│    the pending Pods?"                                            │
│ 3. Choose the Node Group that satisfies the most pending Pods   │
│ 4. Check: would adding a Node exceed the Node Group's max?      │
│ 5. If OK → request new Node from cloud provider                 │
│                                                                  │
│ EXPANDER STRATEGIES (when multiple groups could work):          │
│ ├── random: pick any valid group randomly                       │
│ ├── most-pods: pick group that schedules the most pods          │
│ ├── least-waste: pick group with least unused resources after   │
│ ├── price: pick cheapest node type                              │
│ └── priority: use user-defined priority order                   │
│                                                                  │
│ SCALE DOWN CHECK:                                                │
│                                                                  │
│ 1. For each Node, calculate utilization:                        │
│    (sum of Pod requests) / (Node allocatable resources)         │
│ 2. If utilization < 50% for > 10 minutes → candidate           │
│ 3. Simulate: "Can all Pods on this Node move elsewhere?"        │
│ 4. Check PodDisruptionBudgets — would eviction violate them?   │
│ 5. If safe → cordon, drain, terminate                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Flask App with Prometheus Metrics for HPA

```python
# app.py — Expose custom metrics for Kubernetes HPA to use
from flask import Flask, request
from prometheus_client import Counter, Histogram, Gauge, generate_latest
import time

app = Flask(__name__)

# Define metrics that HPA can scale on
REQUEST_COUNT = Counter('http_requests_total', 'Total requests', ['method', 'endpoint'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'Request latency',
                            buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0])
ACTIVE_REQUESTS = Gauge('http_active_requests', 'Currently processing requests')
QUEUE_SIZE = Gauge('task_queue_size', 'Number of tasks waiting in queue')

@app.before_request
def before_request():
    request.start_time = time.time()
    ACTIVE_REQUESTS.inc()

@app.after_request
def after_request(response):
    latency = time.time() - request.start_time
    REQUEST_COUNT.labels(method=request.method, endpoint=request.path).inc()
    REQUEST_LATENCY.observe(latency)
    ACTIVE_REQUESTS.dec()
    return response

@app.route('/metrics')
def metrics():
    """Prometheus scrapes this endpoint every 15 seconds."""
    return generate_latest(), 200, {'Content-Type': 'text/plain'}

@app.route('/api/process', methods=['POST'])
def process():
    # Simulate work
    time.sleep(0.1)
    return {'status': 'processed'}

# HPA will scale based on http_requests_total rate
# or http_active_requests going above threshold
```

### Java — Spring Boot App with Micrometer for HPA

```java
// MetricsConfig.java — Expose custom metrics for Kubernetes HPA scaling
import io.micrometer.core.instrument.*;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import javax.servlet.http.*;
import java.util.concurrent.atomic.AtomicInteger;

@Component
public class MetricsConfig implements HandlerInterceptor {
    
    private final MeterRegistry registry;
    private final AtomicInteger activeRequests;
    private final AtomicInteger queueDepth;
    
    public MetricsConfig(MeterRegistry registry) {
        this.registry = registry;
        // Gauge: current active requests (HPA can scale on this)
        this.activeRequests = registry.gauge("app.active.requests",
            new AtomicInteger(0));
        // Gauge: background job queue depth
        this.queueDepth = registry.gauge("app.queue.depth",
            new AtomicInteger(0));
    }
    
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res,
                            Object handler) {
        activeRequests.incrementAndGet();
        req.setAttribute("startTime", System.nanoTime());
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res,
                               Object handler, Exception ex) {
        activeRequests.decrementAndGet();
        long duration = System.nanoTime() - (long) req.getAttribute("startTime");
        
        // Record request latency histogram (HPA scales if P99 exceeds threshold)
        registry.timer("app.request.duration",
            Tags.of("method", req.getMethod(), "uri", req.getRequestURI()))
            .record(duration, java.util.concurrent.TimeUnit.NANOSECONDS);
        
        // Count errors (HPA can scale on error rate)
        if (res.getStatus() >= 500) {
            registry.counter("app.errors.total",
                Tags.of("status", String.valueOf(res.getStatus()))).increment();
        }
    }
    
    // Kubernetes Prometheus scrapes /actuator/prometheus
    // Prometheus Adapter exposes metrics to HPA via custom.metrics.k8s.io
}
```

---

## Infrastructure Examples

### HPA Configuration — Basic and Advanced

```yaml
# hpa-basic.yaml — Simple CPU-based HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
  # Scale on CPU (50% target)
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  # Also scale on memory (70% target)
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  # Custom metric: requests per second per pod
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"  # 100 req/s per pod max
  # Scale-up/down behavior
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0  # React immediately
      policies:
      - type: Percent
        value: 100        # Can double pod count
        periodSeconds: 15
      - type: Pods
        value: 4          # Or add 4 pods at a time
        periodSeconds: 15
      selectPolicy: Max   # Use whichever adds MORE
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before removing
      policies:
      - type: Pods
        value: 1          # Remove only 1 pod at a time
        periodSeconds: 60 # Every 60 seconds max
      selectPolicy: Min   # Use least aggressive
```

### VPA Configuration

```yaml
# vpa.yaml — Vertical Pod Autoscaler
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-server-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-server
  updatePolicy:
    updateMode: "Off"  # Start with recommendations only!
  resourcePolicy:
    containerPolicies:
    - containerName: web-server
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "4"
        memory: "8Gi"
      controlledResources: ["cpu", "memory"]
---
# Check VPA recommendations:
# kubectl describe vpa web-server-vpa
#
# Output:
#   Recommendation:
#     Container Recommendations:
#       Container Name: web-server
#       Lower Bound:    Cpu: 200m, Memory: 256Mi
#       Target:         Cpu: 500m, Memory: 512Mi  ← Use this!
#       Upper Bound:    Cpu: 1200m, Memory: 1Gi
#       Uncapped Target: Cpu: 500m, Memory: 512Mi
```

### Cluster Autoscaler Configuration

```yaml
# cluster-autoscaler-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    spec:
      containers:
      - name: cluster-autoscaler
        image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste       # Choose cheapest node type
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled
        - --balance-similar-node-groups  # Keep AZs balanced
        - --scale-down-delay-after-add=10m   # Wait 10 min after adding
        - --scale-down-unneeded-time=10m     # Node idle for 10 min → remove
        - --scale-down-utilization-threshold=0.5  # Below 50% = underutilized
        - --max-node-provision-time=15m      # Give up if node takes > 15 min
```

### KEDA — Scale on Kafka Consumer Lag

```yaml
# keda-kafka-scaler.yaml — Scale workers based on Kafka consumer lag
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: kafka-consumer-deployment
  minReplicaCount: 0   # Scale to zero when no messages!
  maxReplicaCount: 100
  pollingInterval: 10  # Check every 10 seconds
  cooldownPeriod: 300  # Wait 5 min before scaling to zero
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka-broker:9092
      consumerGroup: order-processors
      topic: orders
      lagThreshold: "50"  # Scale up when lag > 50 messages per partition
      # With 10 partitions and lag 500:
      # Desired = 500 / 50 = 10 pods (1 per partition, capped by partition count)
---
# KEDA also supports:
# - AWS SQS queue depth
# - Redis streams length
# - PostgreSQL query results (custom SQL!)
# - Prometheus metrics
# - Azure Service Bus
# - RabbitMQ queue length
# - Cron schedules (scale on time)
```

---

## Real-World Example

### How Spotify Scales with Kubernetes

```
SPOTIFY'S KUBERNETES AUTOSCALING:

┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  CHALLENGE:                                                          │
│  • 600+ microservices running on Kubernetes (GKE)                   │
│  • Peak: Monday morning commute (everyone hits "play")              │
│  • Off-peak: Late night (sleeping, not streaming)                   │
│  • Some services: constant load (recommendation engine)             │
│  • Some services: bursty (playlist generation, social features)     │
│                                                                      │
│  SOLUTION: Multi-level autoscaling                                  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ Service: Audio Streaming                                  │      │
│  │ HPA metric: active_streams_per_pod                        │      │
│  │ Target: 5000 concurrent streams per pod                   │      │
│  │ Min: 50 pods | Max: 500 pods                              │      │
│  │ Cluster Autoscaler: n2-highcpu-32 nodes (CPU-optimized)  │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ Service: Playlist Generator                               │      │
│  │ KEDA scaler: Pub/Sub message backlog                      │      │
│  │ Scales to ZERO when no playlists being generated         │      │
│  │ Bursts to 200 pods during "Your Daily Mix" refresh       │      │
│  │ (5 AM local time in each timezone)                        │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ Service: Search                                           │      │
│  │ HPA metric: p99_search_latency_ms (custom metric)        │      │
│  │ Target: p99 < 100ms                                       │      │
│  │ VPA mode: "Off" (recommendations reviewed weekly)        │      │
│  │ Result: right-sized from 2GB→512MB RAM (75% saving!)     │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  RESULTS:                                                            │
│  ├── 40% cost reduction from VPA right-sizing alone                │
│  ├── 60% fewer nodes at off-peak (Cluster Autoscaler)              │
│  ├── Zero manual scaling interventions                              │
│  └── P99 latency maintained below SLA targets                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using HPA + VPA on same resource (CPU) | They fight — HPA adds pods, VPA changes pod size, confusing each other | Use HPA for scaling out; VPA on different dimension (memory) OR VPA in "Off" mode for recommendations only |
| No resource requests set on Pods | HPA can't calculate utilization without requests; Cluster Autoscaler can't predict fit | ALWAYS set resource requests on every container |
| Min replicas = 1 for critical services | Single replica = downtime during updates or crashes | Min 2-3 for anything user-facing |
| Cluster Autoscaler max too low | Cluster can't scale during major incidents | Set max generously; use cost alerts for budget control |
| Not using PodDisruptionBudgets | Scale-down or VPA eviction can kill all pods simultaneously | Always set PDB: `minAvailable: 50%` or `maxUnavailable: 1` |
| HPA stabilization too short | Pods scale down, traffic comes back, scale up again (flapping) | scaleDown stabilization ≥ 300 seconds |
| Ignoring node startup time | Cluster Autoscaler takes 2-5 minutes to add nodes | Set HPA threshold lower to trigger earlier; use overprovisioning |

---

## When to Use / When NOT to Use

```
WHICH AUTOSCALER DO YOU NEED?

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  "My pods are maxing out CPU/memory"                            │
│  └── HPA: Add more pods ✓                                      │
│                                                                  │
│  "My pods have too much/little resource allocated"              │
│  └── VPA: Right-size pod resources ✓                            │
│                                                                  │
│  "HPA wants more pods but nodes are full"                       │
│  └── Cluster Autoscaler: Add more nodes ✓                      │
│                                                                  │
│  "I have bursty workloads that sit idle 80% of the time"       │
│  └── KEDA: Scale to zero between bursts ✓                      │
│                                                                  │
│  MOST COMMON COMBO FOR PRODUCTION:                              │
│  HPA + Cluster Autoscaler + VPA (in "Off"/recommend mode)      │
│                                                                  │
│  DON'T USE KUBERNETES AUTOSCALING IF:                           │
│  ├── Not using Kubernetes (use cloud ASGs instead — Ch. 7.5)   │
│  ├── Running on bare metal with fixed capacity                  │
│  └── Stateful apps that can't tolerate restarts (use StatefulSet│
│      with manual scaling)                                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Kubernetes has THREE levels of autoscaling** — HPA (more pods), VPA (bigger pods), and Cluster Autoscaler (more nodes). Each solves a different problem.

2. **HPA is the most commonly used** — it scales pod replicas based on metrics. Use CPU for compute-bound, custom metrics for I/O-bound or business-specific workloads.

3. **VPA is best for right-sizing** — start in "Off" mode to get recommendations, then update your manifests. Saves 30-60% on resource waste.

4. **Cluster Autoscaler is essential in the cloud** — without it, HPA hits a ceiling when nodes are full. It automatically adds/removes nodes from your cloud provider.

5. **KEDA enables scale-to-zero** — critical for event-driven workloads (queue processors, scheduled jobs) where paying for idle pods is wasteful.

6. **Always set resource requests** — without them, HPA can't calculate utilization and Cluster Autoscaler can't predict pod placement.

7. **Scale up fast, scale down slow** — use asymmetric behavior policies. Users feel scale-up latency immediately; extra pods cost little.

---

## What's Next?

You've mastered Kubernetes-native autoscaling. But what about the underlying cloud infrastructure? In **Chapter 7.5: Cloud Elasticity — AWS ASG, Azure VMSS, GCP MIG**, we'll explore how each major cloud provider implements auto scaling at the infrastructure level, their unique features, and how to choose between them.
