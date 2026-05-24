# Metrics & Dashboards — Prometheus & Grafana

> **What you'll learn**: How to collect numeric measurements from your systems using Prometheus, store them as time-series data, and visualize them in real-time dashboards using Grafana.

---

## Real-Life Analogy

Think of a **hospital ICU**. Every patient is connected to monitors that continuously display:
- Heart rate (60-100 bpm ✓, 200 bpm → ALARM!)
- Blood pressure (120/80 ✓, 200/120 → ALARM!)
- Oxygen saturation (98% ✓, 85% → ALARM!)
- Temperature (37°C ✓, 40°C → ALARM!)

The nurses don't read these numbers one by one. They look at **dashboards** — screens that show all vital signs with colors and graphs. Green = healthy. Yellow = warning. Red = act NOW.

That's exactly what Prometheus + Grafana do for your servers:
- **Prometheus** = the heart monitors (collects the numbers continuously)
- **Grafana** = the ICU dashboard screens (shows everything visually)

---

## What Are Metrics?

Metrics are **numeric measurements collected over time**. Every 15 seconds (or whatever interval you choose), your system reports numbers like:

```
At 14:32:00:
  cpu_usage = 45%
  memory_used = 6.2 GB
  http_requests_per_second = 1,247
  error_rate = 0.3%
  response_time_p99 = 420ms
  active_connections = 89

At 14:32:15 (15 seconds later):
  cpu_usage = 47%
  memory_used = 6.3 GB
  http_requests_per_second = 1,312
  error_rate = 0.2%
  response_time_p99 = 395ms
  active_connections = 91
```

Plot these over time and you get a **time-series graph**:

```
CPU Usage Over 24 Hours:
100%|
 90%|                          ╱╲
 80%|                    ╱╲  ╱  ╲    ← Peak at 2 PM
 70%|              ╱╲  ╱  ╲╱    ╲
 60%|        ╱╲  ╱  ╲╱           ╲
 50%|──╱╲──╱  ╲╱                   ╲──
 40%|╱    ╱                           ╲
 30%|                                    ╲──── ← Night traffic drop
 20%|
    └─────────────────────────────────────────
     00:00  04:00  08:00  12:00  16:00  20:00
```

---

## Prometheus — The Metrics Collector

### What is Prometheus?

Prometheus is an open-source monitoring system created at **SoundCloud** in 2012 and donated to the CNCF. It's now the industry standard for metrics collection in cloud-native environments.

### How Prometheus Works (Pull Model)

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS ARCHITECTURE                        │
│                                                                   │
│  ┌──────────┐  /metrics    ┌─────────────────────────────────┐  │
│  │ App #1   │◀────────────│                                 │  │
│  │ :8080    │              │                                 │  │
│  └──────────┘              │       PROMETHEUS SERVER         │  │
│                            │                                 │  │
│  ┌──────────┐  /metrics   │  ┌───────────┐  ┌───────────┐  │  │
│  │ App #2   │◀────────────│  │  Scraper  │  │   TSDB    │  │  │
│  │ :8081    │              │  │(pulls     │  │(Time-Series│  │  │
│  └──────────┘              │  │ every 15s)│  │ Database) │  │  │
│                            │  └───────────┘  └───────────┘  │  │
│  ┌──────────┐  /metrics   │                                 │  │
│  │ Database │◀────────────│  ┌───────────┐  ┌───────────┐  │  │
│  │ Exporter │              │  │   Rules   │  │  Alerting │  │  │
│  │ :9187    │              │  │  Engine   │  │  Manager  │  │  │
│  └──────────┘              │  └───────────┘  └───────────┘  │  │
│                            │                                 │  │
│  ┌──────────┐  /metrics   └──────────────────┬──────────────┘  │
│  │ Node     │◀────────────                   │                  │
│  │ Exporter │              Pulls metrics      │ Sends alerts    │
│  │ :9100    │              every 15 seconds   ▼                  │
│  └──────────┘                          ┌──────────────┐         │
│                                        │  AlertManager│         │
│                                        │  → Slack     │         │
│                                        │  → PagerDuty │         │
│                                        └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### The /metrics Endpoint

Every service exposes a `/metrics` endpoint that Prometheus scrapes:

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/users",status="200"} 15234
http_requests_total{method="GET",endpoint="/api/users",status="500"} 23
http_requests_total{method="POST",endpoint="/api/orders",status="201"} 4521

# HELP http_request_duration_seconds Request duration histogram
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{endpoint="/api/users",le="0.01"} 12000
http_request_duration_seconds_bucket{endpoint="/api/users",le="0.05"} 14500
http_request_duration_seconds_bucket{endpoint="/api/users",le="0.1"} 15000
http_request_duration_seconds_bucket{endpoint="/api/users",le="0.5"} 15200
http_request_duration_seconds_bucket{endpoint="/api/users",le="1"} 15234
http_request_duration_seconds_bucket{endpoint="/api/users",le="+Inf"} 15234

# HELP process_cpu_seconds_total Total CPU time
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 1234.56
```

---

## PromQL — Prometheus Query Language

PromQL is how you ask questions about your metrics:

```
┌─────────────────────────────────────────────────────────────────┐
│                     PROMQL EXAMPLES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  BASIC QUERIES:                                                  │
│  ─────────────                                                   │
│  http_requests_total                    All request counters     │
│  http_requests_total{status="500"}      Only 500 errors          │
│  up{job="api-server"}                   Is the service alive?    │
│                                                                   │
│  RATE (how fast a counter is growing):                          │
│  ─────────────                                                   │
│  rate(http_requests_total[5m])          Requests per second      │
│                                          (averaged over 5 min)   │
│                                                                   │
│  AGGREGATION:                                                    │
│  ─────────────                                                   │
│  sum(rate(http_requests_total[5m]))     Total RPS across all     │
│                                          instances               │
│                                                                   │
│  PERCENTILES (from histograms):                                 │
│  ─────────────                                                   │
│  histogram_quantile(0.99,               p99 latency              │
│    rate(http_request_duration_seconds_bucket[5m]))               │
│                                                                   │
│  ALERTING EXPRESSIONS:                                          │
│  ─────────────                                                   │
│  rate(http_requests_total{status="500"}[5m])                    │
│    / rate(http_requests_total[5m]) > 0.05                       │
│  → "Error rate > 5% in the last 5 minutes"                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Grafana — The Visualization Layer

### What is Grafana?

Grafana is an open-source dashboard platform that connects to Prometheus (and 50+ other data sources) and creates beautiful, interactive dashboards.

```
┌─────────────────────────────────────────────────────────────────┐
│                    GRAFANA DASHBOARD                              │
│                                                                   │
│  ┌─────────────────────────┐  ┌─────────────────────────┐      │
│  │    Request Rate (RPS)   │  │     Error Rate (%)      │      │
│  │         ╱╲              │  │                          │      │
│  │   ╱╲  ╱  ╲             │  │  ─────────── 0.2%       │      │
│  │  ╱  ╲╱    ╲ 1,247 RPS  │  │       (healthy)         │      │
│  │ ╱                       │  │                          │      │
│  └─────────────────────────┘  └─────────────────────────┘      │
│                                                                   │
│  ┌─────────────────────────┐  ┌─────────────────────────┐      │
│  │  Latency Percentiles   │  │     CPU / Memory        │      │
│  │                          │  │                          │      │
│  │  p99: ─── 420ms        │  │  CPU: ████████░░ 78%    │      │
│  │  p95: ─── 180ms        │  │  RAM: ██████░░░░ 62%    │      │
│  │  p50: ─── 45ms         │  │  Disk: ███░░░░░░░ 31%   │      │
│  └─────────────────────────┘  └─────────────────────────┘      │
│                                                                   │
│  ┌───────────────────────────────────────────────────────┐      │
│  │              Active Connections Over Time              │      │
│  │  120|         ╱╲                                      │      │
│  │  100|    ╱╲  ╱  ╲  ╱╲                               │      │
│  │   80|───╱  ╲╱    ╲╱  ╲──────                        │      │
│  │   60|                                                 │      │
│  │      └───────────────────────────────────────────     │      │
│  └───────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### Grafana Data Sources

```
Grafana can pull data from 50+ sources:

  ┌──────────────────────────────────────┐
  │            GRAFANA                     │
  │                                        │
  │  ┌──────────────────────────────┐    │
  │  │      Dashboard Engine        │    │
  │  └──────────┬───────────────────┘    │
  │             │                         │
  │  ┌──────────┼──────────────────┐     │
  │  │          │                   │     │
  │  ▼          ▼                   ▼     │
  │ Prometheus  Elasticsearch    Loki    │
  │ (metrics)   (logs)          (logs)   │
  │                                        │
  │ CloudWatch  InfluxDB        MySQL    │
  │ Datadog     Tempo           Jaeger   │
  └──────────────────────────────────────┘
```

---

## How It Works Internally

### Prometheus Time-Series Database (TSDB)

```
How Prometheus stores metrics internally:

TIME SERIES = metric name + label set + timestamp + value

Example time series:
  http_requests_total{method="GET", status="200"}

Stored as:
  ┌─────────────────────────────────────────┐
  │ Timestamp         │ Value               │
  ├───────────────────┼─────────────────────┤
  │ 1710510720        │ 15000               │
  │ 1710510735        │ 15023  (+23 in 15s) │
  │ 1710510750        │ 15051  (+28 in 15s) │
  │ 1710510765        │ 15078  (+27 in 15s) │
  └───────────────────┴─────────────────────┘

Storage format:
  ┌────────────────────────────────────────────────────┐
  │  Blocks (2-hour chunks):                           │
  │                                                     │
  │  [Block 1: 00:00-02:00] [Block 2: 02:00-04:00]   │
  │                                                     │
  │  Each block contains:                              │
  │  ├── index (inverted index for label lookup)      │
  │  ├── chunks (compressed time-series data)          │
  │  └── tombstones (deleted data markers)            │
  │                                                     │
  │  Compression: ~1.3 bytes per sample (very compact)│
  │  1 million samples = ~1.3 MB                      │
  └────────────────────────────────────────────────────┘
```

### Cardinality: The Silent Killer

```
CARDINALITY = number of unique time series

LOW CARDINALITY (good):
  http_requests_total{method="GET", status="200"}   ← Only a few values
  http_requests_total{method="POST", status="201"}
  → Maybe 20-30 unique combinations

HIGH CARDINALITY (dangerous!):
  http_requests_total{user_id="user_1", ...}
  http_requests_total{user_id="user_2", ...}
  http_requests_total{user_id="user_3", ...}
  → 1 million users = 1 million time series = PROMETHEUS CRASH

RULE: Never use unbounded values as labels!
  ❌ user_id, email, request_id, IP address
  ✅ method, status_code, service_name, region
```

---

## Code Examples

### Python: Custom Metrics with Prometheus Client

```python
# metrics_example.py
# Full example of instrumenting a Python service with Prometheus metrics

from prometheus_client import (
    Counter, Histogram, Gauge, Summary, 
    start_http_server, REGISTRY
)
from flask import Flask, request
import time
import random

app = Flask(__name__)

# --- COUNTER: Total requests (always goes up) ---
REQUESTS = Counter(
    'app_requests_total',
    'Total application requests',
    ['method', 'endpoint', 'status']
)

# --- HISTOGRAM: Response time distribution ---
LATENCY = Histogram(
    'app_request_latency_seconds',
    'Request latency in seconds',
    ['endpoint'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

# --- GAUGE: Current active connections ---
ACTIVE = Gauge(
    'app_active_connections',
    'Number of active connections'
)

# --- GAUGE: Last deployment timestamp ---
DEPLOY_TIME = Gauge(
    'app_last_deploy_timestamp',
    'Unix timestamp of last deployment'
)
DEPLOY_TIME.set_to_current_time()  # Set once at startup

# --- Business metric: Orders ---
ORDERS = Counter(
    'business_orders_total',
    'Total orders placed',
    ['product_category', 'payment_method']
)

REVENUE = Counter(
    'business_revenue_dollars_total',
    'Total revenue in dollars',
    ['currency']
)

@app.route('/api/users')
def get_users():
    ACTIVE.inc()
    start = time.time()
    
    try:
        # Simulate work
        time.sleep(random.uniform(0.01, 0.3))
        
        REQUESTS.labels('GET', '/api/users', '200').inc()
        return {"users": ["alice", "bob"]}
    except Exception:
        REQUESTS.labels('GET', '/api/users', '500').inc()
        raise
    finally:
        LATENCY.labels('/api/users').observe(time.time() - start)
        ACTIVE.dec()

@app.route('/api/orders', methods=['POST'])
def create_order():
    # Track business metrics
    ORDERS.labels(
        product_category='electronics',
        payment_method='credit_card'
    ).inc()
    REVENUE.labels(currency='USD').inc(99.99)
    
    return {"order_id": "12345"}, 201

if __name__ == '__main__':
    # Start Prometheus metrics server on port 8000
    start_http_server(8000)  # /metrics available here
    app.run(port=8080)
```

### Java: Metrics with Micrometer + Prometheus

```java
// MetricsConfig.java - Spring Boot with Micrometer
import io.micrometer.core.instrument.*;
import io.micrometer.core.instrument.binder.jvm.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> commonTags() {
        return registry -> {
            // Add common tags to ALL metrics
            registry.config().commonTags(
                "service", "order-service",
                "environment", "production",
                "region", "us-east-1"
            );
            
            // Register JVM metrics automatically
            new JvmMemoryMetrics().bindTo(registry);
            new JvmGcMetrics().bindTo(registry);
            new JvmThreadMetrics().bindTo(registry);
        };
    }
}

// OrderController.java - Custom business metrics
@RestController
public class OrderController {
    
    private final Counter orderCounter;
    private final Timer orderTimer;
    private final DistributionSummary orderValue;
    
    public OrderController(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created.total")
            .description("Total orders created")
            .tag("service", "order-api")
            .register(registry);
        
        this.orderTimer = Timer.builder("orders.processing.time")
            .description("Time to process an order")
            .publishPercentiles(0.5, 0.95, 0.99)  // p50, p95, p99
            .register(registry);
        
        this.orderValue = DistributionSummary.builder("orders.value.dollars")
            .description("Distribution of order values")
            .baseUnit("dollars")
            .publishPercentiles(0.5, 0.9, 0.99)
            .register(registry);
    }
    
    @PostMapping("/api/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest req) {
        return orderTimer.record(() -> {
            // Process order
            Order order = processOrder(req);
            
            // Record business metrics
            orderCounter.increment();
            orderValue.record(order.getTotalAmount());
            
            return ResponseEntity.status(201).body(order);
        });
    }
}
```

---

## Infrastructure Example: Complete Prometheus + Grafana Setup

### prometheus.yml — Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s       # Scrape targets every 15 seconds
  evaluation_interval: 15s   # Evaluate alert rules every 15 seconds

# Alert rules
rule_files:
  - "alert_rules.yml"

# Where to send alerts
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# What to scrape
scrape_configs:
  # Prometheus monitors itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  # Your application services
  - job_name: 'api-service'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['api-1:8080', 'api-2:8080', 'api-3:8080']
        labels:
          environment: 'production'
  
  # Auto-discovery in Kubernetes
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape=true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
  
  # Node Exporter (server hardware metrics)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
  
  # PostgreSQL metrics
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
  
  # Redis metrics
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

### alert_rules.yml — Alerting Rules

```yaml
# alert_rules.yml
groups:
  - name: application_alerts
    rules:
      # Alert if error rate > 5% for 5 minutes
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected ({{ $value | humanizePercentage }})"
          description: "Error rate is above 5% for the last 5 minutes"
      
      # Alert if p99 latency > 2 seconds
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency above 2s for {{ $labels.service }}"
      
      # Alert if service is down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is DOWN"
```

### Docker Compose — Full Stack

```yaml
# docker-compose-monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.50.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert_rules.yml:/etc/prometheus/alert_rules.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=10GB'

  grafana:
    image: grafana/grafana:10.3.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

  alertmanager:
    image: prom/alertmanager:v0.27.0
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml

  node-exporter:
    image: prom/node-exporter:v1.7.0
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro

volumes:
  prometheus-data:
  grafana-data:
```

---

## Real-World Example

### How GitLab Monitors with Prometheus + Grafana

```
┌─────────────────────────────────────────────────────────┐
│            GITLAB'S MONITORING APPROACH                   │
│                                                          │
│  Scale:                                                 │
│  • 30+ million registered users                        │
│  • 1,000+ microservices                                │
│  • 100+ million time series in Prometheus              │
│                                                          │
│  Architecture:                                          │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │ Services │──▶│  Prometheus  │──▶│   Thanos     │ │
│  │ (1000+)  │    │  (per-shard) │    │  (long-term  │ │
│  └──────────┘    └──────────────┘    │   storage)   │ │
│                                       └──────┬───────┘ │
│                                              │          │
│                                              ▼          │
│                                       ┌──────────────┐ │
│                                       │   Grafana    │ │
│                                       │ (dashboards) │ │
│                                       └──────────────┘ │
│                                                          │
│  Key Practices:                                         │
│  • Every service has a RED dashboard                   │
│    (Rate, Errors, Duration)                            │
│  • Prometheus sharded by team/domain                   │
│  • Thanos for cross-shard queries & long retention     │
│  • Automatic recording rules for expensive queries     │
└─────────────────────────────────────────────────────────┘
```

### Netflix's Atlas: Custom Metrics at Extreme Scale

Netflix collects **2+ billion metrics per minute** using Atlas:
- Custom dimensional time-series database
- Handles 1.5 billion unique time series
- In-memory queries return in < 100ms
- Uses "streaming aggregation" — pre-aggregates in real-time

---

## Scaling Prometheus: Thanos & Cortex

When a single Prometheus becomes too small:

```
┌──────────────────────────────────────────────────────────────────┐
│                 SCALING WITH THANOS                                │
│                                                                    │
│  Region: US-East              Region: EU-West                     │
│  ┌────────────────┐           ┌────────────────┐                 │
│  │  Prometheus 1  │           │  Prometheus 2  │                 │
│  │  (scrapes US   │           │  (scrapes EU   │                 │
│  │   services)    │           │   services)    │                 │
│  └───────┬────────┘           └───────┬────────┘                 │
│          │                            │                           │
│          │     ┌──────────────┐       │                           │
│          └────▶│ Thanos Query │◀──────┘                           │
│                │ (federated   │                                    │
│                │  queries)    │                                    │
│                └──────┬───────┘                                    │
│                       │                                            │
│                       ▼                                            │
│              ┌──────────────────┐                                 │
│              │  Object Storage  │  (S3/GCS — unlimited retention)│
│              │  (Thanos Store)  │                                 │
│              └──────────────────┘                                 │
│                                                                    │
│  Benefits:                                                        │
│  • Global view across all regions                                │
│  • Unlimited retention (cheap object storage)                    │
│  • Each Prometheus stays simple                                  │
│  • Downsampling for old data (5m, 1h averages)                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| **High-cardinality labels** (user_id, IP) | Prometheus runs out of memory and crashes | Only use bounded labels (method, status, region) |
| **Scraping too frequently** (every 1s) | Overwhelms both Prometheus and targets | 15-30s is fine for most use cases |
| **No retention limits** | Disk fills up, Prometheus stops | Set `--storage.tsdb.retention.time=30d` |
| **Only monitoring averages** | Hides tail latency problems | Always track p50, p95, p99 percentiles |
| **Too many dashboards** | Nobody knows which one to look at | One "golden signals" dashboard per service |
| **Alerting on symptoms, not impact** | Too many alerts → alert fatigue | Alert on user-facing impact (error rate, latency) |
| **Not using recording rules** | Expensive queries slow down Grafana | Pre-compute common aggregations as recording rules |

---

## When to Use / When NOT to Use

### Use Prometheus + Grafana When:
- ✅ You're running Kubernetes (native integration)
- ✅ You need powerful time-series queries (PromQL)
- ✅ You want open-source and vendor-neutral
- ✅ Your scale is under 10 million active time series per instance
- ✅ Your team likes "pull" model and service discovery

### Consider Alternatives When:
- ⚠️ You need 100+ days retention without Thanos → Consider InfluxDB or Mimir
- ⚠️ You have very short-lived jobs (functions, batch) → Consider Pushgateway or StatsD
- ⚠️ You want zero infrastructure management → Consider Datadog, New Relic (SaaS)
- ⚠️ You need high-resolution (sub-second) metrics → Consider InfluxDB or VictoriaMetrics

---

## Key Takeaways

1. **Prometheus is the standard** for cloud-native metrics collection — pull-based, PromQL, and deeply integrated with Kubernetes.

2. **Grafana is the standard** for visualization — connects to 50+ data sources, rich dashboards, free and open-source.

3. **Four Golden Signals** (Latency, Traffic, Errors, Saturation) should be on every service's dashboard.

4. **Watch your cardinality** — high-cardinality labels are the #1 cause of Prometheus crashes in production.

5. **Percentiles beat averages** — always track p50, p95, p99. A 50ms average can hide a 5-second p99.

6. **Use Thanos or Cortex for scale** — when you need cross-region queries, long retention, or more than one Prometheus can handle.

7. **Alert on impact, not symptoms** — "Error rate > 5% for users" is better than "CPU > 80% on one server."

---

## What's Next?

Now that you can collect and visualize metrics, let's explore the third pillar — **Distributed Tracing** — which lets you follow a single request as it travels across multiple services, databases, and queues.

**Next: [05-distributed-tracing.md](./05-distributed-tracing.md)** — Distributed Tracing (Jaeger, Zipkin, OpenTelemetry)
