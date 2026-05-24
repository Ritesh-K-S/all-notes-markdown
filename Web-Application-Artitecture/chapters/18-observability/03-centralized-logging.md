# Centralized Logging — ELK Stack, Fluentd & Loki

> **What you'll learn**: How to collect logs from hundreds of services into one place where you can search, filter, and analyze them — using the ELK Stack, Fluentd, and Grafana Loki.

---

## Real-Life Analogy

Imagine you run a chain of **500 restaurants** across the country. Every restaurant keeps its own logbook (complaints, orders, incidents). If a food poisoning outbreak happens, how do you investigate?

**Without centralized logging:** You call each restaurant, ask them to dig through their physical logbooks, read entries over the phone, and try to find patterns. This takes DAYS.

**With centralized logging:** Every restaurant sends a photo of each logbook entry to a central headquarters instantly. At HQ, you type "chicken" + "complaint" + "last 48 hours" and instantly see all matching entries across all 500 locations. You find the pattern in MINUTES.

That's exactly what centralized logging does for your servers. Instead of SSH-ing into 200 servers and running `grep`, you search all logs from one UI.

---

## The Problem: Logs Are Scattered Everywhere

```
WITHOUT CENTRALIZED LOGGING:

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Server 1 │  │ Server 2 │  │ Server 3 │  │ Server 4 │
│          │  │          │  │          │  │          │
│ /var/log │  │ /var/log │  │ /var/log │  │ /var/log │
│  app.log │  │  app.log │  │  app.log │  │  app.log │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
     ?              ?              ?              ?

Developer: "I need to find why user X got an error..."
  → SSH into server 1... grep... not here
  → SSH into server 2... grep... not here
  → SSH into server 3... grep... found something!
  → Wait, the request also hit server 4...
  → This takes 30 minutes for ONE investigation 😫
```

```
WITH CENTRALIZED LOGGING:

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Server 1 │  │ Server 2 │  │ Server 3 │  │ Server 4 │
│          │  │          │  │          │  │          │
│ Log Agent│  │ Log Agent│  │ Log Agent│  │ Log Agent│
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │              │
     └──────────────┼──────────────┼──────────────┘
                    │              │
                    ▼              ▼
            ┌─────────────────────────┐
            │   CENTRALIZED LOGGING   │
            │      SYSTEM             │
            │                         │
            │  Search: user_id="X"    │
            │  → All results in 2sec  │
            └─────────────────────────┘
```

---

## The Centralized Logging Architecture

Every centralized logging solution follows this pattern:

```
┌─────────────────────────────────────────────────────────────────┐
│                  CENTRALIZED LOGGING PIPELINE                     │
│                                                                   │
│  ┌─────────┐    ┌───────────┐    ┌─────────┐    ┌───────────┐  │
│  │ Sources │──▶│ Collection│──▶│ Storage │──▶│   Query    │  │
│  │         │    │& Transform│    │& Index  │    │& Visualize│  │
│  └─────────┘    └───────────┘    └─────────┘    └───────────┘  │
│                                                                   │
│  • App logs      • Parse          • Index for    • Search UI   │
│  • System logs   • Enrich           fast search  • Dashboards  │
│  • Access logs   • Filter         • Compress     • Alerts      │
│  • Audit logs    • Route          • Retention                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Option 1: The ELK Stack (Elasticsearch + Logstash + Kibana)

The most popular open-source logging stack, used by thousands of companies.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ELK STACK                                  │
│                                                                   │
│  ┌──────────┐     ┌──────────┐     ┌──────────────┐            │
│  │  Your    │     │          │     │              │            │
│  │  Apps    │────▶│ Logstash │────▶│Elasticsearch │            │
│  │          │     │          │     │              │            │
│  └──────────┘     └──────────┘     └──────┬───────┘            │
│                                           │                     │
│  What it does:    What it does:    What it does:               │
│  • Generate       • Collect        • Store logs                │
│    log entries    • Parse          • Full-text index           │
│                   • Transform      • Fast search               │
│                   • Route                                       │
│                                           │                     │
│                                           ▼                     │
│                                    ┌──────────────┐            │
│                                    │    Kibana    │            │
│                                    │              │            │
│                                    │ • Search UI  │            │
│                                    │ • Dashboards │            │
│                                    │ • Alerts     │            │
│                                    └──────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### Modern Variant: EFK (Elasticsearch + Filebeat + Kibana)

In production, **Filebeat** (lightweight) replaces Logstash (heavy) on each server:

```
Server 1: [App] → [Filebeat] ─┐
Server 2: [App] → [Filebeat] ─┤──▶ [Logstash] ──▶ [Elasticsearch] ──▶ [Kibana]
Server 3: [App] → [Filebeat] ─┘
                                    (optional central processing)

Filebeat = lightweight log shipper (uses 5MB RAM)
Logstash = heavy log processor (uses 1GB+ RAM)
```

### How Elasticsearch Indexes Logs

```
Log entry arrives:
  {"timestamp": "2024-03-15T14:32:01Z", "level": "ERROR", "service": "payment", 
   "message": "Card declined for user_42", "trace_id": "abc-123"}

Elasticsearch creates an INVERTED INDEX:
  "card"     → [doc_1, doc_47, doc_892]
  "declined" → [doc_1, doc_53]
  "user_42"  → [doc_1, doc_234, doc_567]
  "payment"  → [doc_1, doc_2, doc_3, ..., doc_500]

Searching "card declined user_42":
  → Intersect posting lists → doc_1 found in 2ms!
```

---

## Option 2: Fluentd / Fluent Bit

**Fluentd** is a unified log collector (part of the CNCF — same org as Kubernetes).

```
┌─────────────────────────────────────────────────────────────────┐
│                      FLUENTD ARCHITECTURE                         │
│                                                                   │
│  ┌──────────────────────────────────────────────────┐           │
│  │  Fluentd collects from MANY sources:              │           │
│  │                                                    │           │
│  │  • Application log files (tail)                   │           │
│  │  • Docker container stdout                        │           │
│  │  • System logs (syslog)                          │           │
│  │  • HTTP endpoints                                 │           │
│  │  • TCP/UDP streams                               │           │
│  └──────────────────────┬───────────────────────────┘           │
│                          │                                       │
│                          ▼                                       │
│  ┌──────────────────────────────────────────────────┐           │
│  │  Process & Transform:                             │           │
│  │  • Parse JSON, regex, CSV                        │           │
│  │  • Add fields (hostname, environment)            │           │
│  │  • Filter (drop debug logs in prod)              │           │
│  │  • Buffer (batch before sending)                 │           │
│  └──────────────────────┬───────────────────────────┘           │
│                          │                                       │
│                          ▼                                       │
│  ┌──────────────────────────────────────────────────┐           │
│  │  Output to MANY destinations:                     │           │
│  │                                                    │           │
│  │  • Elasticsearch                                  │           │
│  │  • Amazon S3 (cheap long-term storage)           │           │
│  │  • Kafka (stream processing)                     │           │
│  │  • Datadog / Splunk / Loki                       │           │
│  │  • stdout (for Kubernetes)                       │           │
│  └──────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

**Fluent Bit** = ultra-lightweight version (2MB, written in C). Perfect for:
- Kubernetes sidecars
- IoT devices
- Edge computing

```
Fluent Bit (2MB RAM) → collects & forwards
Fluentd (40MB RAM)   → collects, processes & routes

Common pattern in Kubernetes:
  [Pod] → stdout → [Fluent Bit DaemonSet] → [Fluentd Aggregator] → [Elasticsearch]
```

---

## Option 3: Grafana Loki — "Like Prometheus, but for Logs"

Loki takes a radically different approach: it does NOT index log content. It only indexes **labels** (metadata).

```
┌─────────────────────────────────────────────────────────────────┐
│                  LOKI vs ELASTICSEARCH                            │
│                                                                   │
│  ELASTICSEARCH:                                                  │
│  Indexes EVERY WORD in every log                                │
│  → Super fast full-text search                                  │
│  → But: VERY expensive storage & compute                        │
│  → 1 TB of logs = ~1 TB of index data                          │
│                                                                   │
│  LOKI:                                                           │
│  Indexes ONLY labels (service, level, env)                      │
│  Stores log content as compressed chunks                        │
│  → Search by labels first, then grep within results             │
│  → 10-100x cheaper than Elasticsearch                           │
│  → 1 TB of logs = ~50 GB stored                                │
└─────────────────────────────────────────────────────────────────┘
```

### Loki Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      GRAFANA LOKI STACK                           │
│                                                                   │
│  ┌─────────┐      ┌─────────┐      ┌──────────┐               │
│  │  Apps   │─────▶│ Promtail│─────▶│   Loki   │               │
│  │         │      │ (agent) │      │          │               │
│  └─────────┘      └─────────┘      └────┬─────┘               │
│                                          │                      │
│  Promtail:                               │                      │
│  • Discovers log files                   │                      │
│  • Adds labels                           ▼                      │
│  • Ships to Loki                  ┌──────────────┐             │
│                                    │   Grafana    │             │
│  Loki stores:                      │              │             │
│  • Labels → Index (tiny)          │ • LogQL UI   │             │
│  • Content → Chunks (compressed)  │ • Dashboards │             │
│                                    │ • Alerts     │             │
│                                    └──────────────┘             │
└─────────────────────────────────────────────────────────────────┘

LogQL query example:
  {service="payment", level="error"} |= "timeout" | json | duration > 5s
```

---

## How It Works Internally

### Log Pipeline Deep Dive

```
Step 1: APPLICATION generates a log
  logger.error("Payment failed", extra={"user": "alice", "amount": 99.99})

Step 2: LOG is written to stdout/file as JSON
  {"ts":"2024-03-15T14:32:01Z","level":"ERROR","msg":"Payment failed",
   "user":"alice","amount":99.99,"trace_id":"abc-123"}

Step 3: AGENT picks it up (Filebeat/Promtail/Fluent Bit)
  • Reads the file (tail -f equivalent)
  • Adds metadata: hostname, container_id, pod_name
  • Batches entries (send every 5s or every 1000 lines)

Step 4: PROCESSOR transforms (optional - Logstash/Fluentd)
  • Parses JSON
  • Extracts fields
  • Masks sensitive data (credit card → "****")
  • Routes: errors → fast index, debug → cold storage

Step 5: STORAGE indexes and stores
  • Elasticsearch: full inverted index
  • Loki: label index + compressed chunks
  • S3: raw archives for compliance

Step 6: QUERY at investigation time
  • "Show me all ERROR logs from payment-service in the last hour"
  • Result in <2 seconds across millions of log entries
```

### Buffering and Reliability

```
What happens if the logging backend is down?

┌──────────┐                    ┌──────────────┐
│   App    │──logs──▶ ┌──────┐ │  Logging     │
│          │          │Buffer│─┤  Backend     │ ← DOWN!
└──────────┘          │(disk)│ │              │
                      └──────┘ └──────────────┘
                         │
                         │ Buffer holds logs on disk
                         │ Retries when backend recovers
                         │ No log loss!
                         ▼
              ┌──────────────────┐
              │ Backend recovers │
              │ Buffer flushes   │
              │ All logs arrive  │
              └──────────────────┘
```

---

## Code Examples

### Python: Structured Logging with JSON

```python
# structured_logging.py
# Production-ready structured logging in Python

import structlog
import logging

# Configure structlog for JSON output
structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,          # Add "level" field
        structlog.stdlib.add_logger_name,        # Add "logger" field
        structlog.processors.TimeStamper(fmt="iso"),  # Add timestamp
        structlog.processors.StackInfoRenderer(),     # Add stack on error
        structlog.processors.JSONRenderer()       # Output as JSON
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

logger = structlog.get_logger("payment-service")

def process_payment(user_id: str, amount: float, currency: str):
    """Example showing proper structured logging."""
    
    # Bind context that appears in ALL subsequent logs
    log = logger.bind(
        user_id=user_id,
        amount=amount,
        currency=currency,
        trace_id="abc-123"  # From request context
    )
    
    log.info("payment.started")
    # Output: {"event":"payment.started","user_id":"user_42",
    #          "amount":99.99,"currency":"USD","trace_id":"abc-123",
    #          "level":"info","timestamp":"2024-03-15T14:32:01Z"}
    
    try:
        # ... payment logic ...
        log.info("payment.succeeded", provider="stripe", latency_ms=234)
    except Exception as e:
        log.error("payment.failed", error=str(e), error_type=type(e).__name__)
        raise

process_payment("user_42", 99.99, "USD")
```

### Java: Structured Logging with Logback + JSON

```java
// PaymentService.java
// Structured JSON logging in Java with SLF4J + MDC

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import net.logstash.logback.argument.StructuredArguments;
import static net.logstash.logback.argument.StructuredArguments.*;

public class PaymentService {
    
    private static final Logger logger = LoggerFactory.getLogger(PaymentService.class);
    
    public void processPayment(String userId, double amount, String currency) {
        // MDC (Mapped Diagnostic Context) adds fields to ALL logs in this thread
        MDC.put("user_id", userId);
        MDC.put("trace_id", "abc-123");
        
        try {
            logger.info("Payment started", 
                kv("amount", amount),        // {"amount": 99.99}
                kv("currency", currency));   // {"currency": "USD"}
            
            // Output JSON:
            // {"timestamp":"2024-03-15T14:32:01Z","level":"INFO",
            //  "message":"Payment started","user_id":"user_42",
            //  "trace_id":"abc-123","amount":99.99,"currency":"USD"}
            
            // ... payment processing ...
            
            logger.info("Payment succeeded",
                kv("provider", "stripe"),
                kv("latency_ms", 234));
                
        } catch (Exception e) {
            logger.error("Payment failed", 
                kv("error", e.getMessage()),
                kv("error_type", e.getClass().getSimpleName()), e);
        } finally {
            MDC.clear();  // Always clean up MDC
        }
    }
}
```

```xml
<!-- logback.xml - Output JSON to stdout for container environments -->
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <!-- Automatically includes MDC fields, timestamp, level -->
    </encoder>
  </appender>
  
  <root level="INFO">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

---

## Infrastructure Example: Complete ELK Stack with Docker

```yaml
# docker-compose-elk.yml
version: '3.8'
services:
  # Elasticsearch - stores and indexes logs
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data

  # Kibana - web UI for searching logs
  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  # Filebeat - lightweight log collector (runs on each server)
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.12.0
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

```yaml
# filebeat.yml - Collect Docker container logs
filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'
    processors:
      - decode_json_fields:
          fields: ["message"]   # Parse JSON log messages
          target: ""
      - add_docker_metadata: ~  # Add container name, image, etc.

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "app-logs-%{+yyyy.MM.dd}"  # Daily indices for easy retention
```

---

## Infrastructure Example: Loki Stack with Kubernetes

```yaml
# Promtail DaemonSet - collects logs from every pod
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
spec:
  template:
    spec:
      containers:
        - name: promtail
          image: grafana/promtail:2.9.0
          args:
            - -config.file=/etc/promtail/config.yaml
          volumeMounts:
            - name: pods-logs
              mountPath: /var/log/pods
              readOnly: true
      volumes:
        - name: pods-logs
          hostPath:
            path: /var/log/pods
```

```yaml
# promtail-config.yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Use pod labels as Loki labels
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
    pipeline_stages:
      - json:
          expressions:
            level: level
            msg: message
      - labels:
          level:   # Extract log level as a Loki label
```

---

## Comparison: ELK vs Loki vs Cloud Solutions

| Feature | ELK Stack | Grafana Loki | Cloud (CloudWatch/GCP) |
|---------|-----------|--------------|------------------------|
| **Cost** | High (indexing is expensive) | Low (10-100x cheaper) | Medium-High (per-GB pricing) |
| **Search Speed** | Fastest (full-text index) | Medium (label + grep) | Medium |
| **Setup Complexity** | Complex (manage cluster) | Simple | Zero (managed) |
| **Best For** | Full-text search heavy use | Cost-sensitive, Grafana users | Small teams, multi-cloud |
| **Scale** | Petabytes (with $$$) | Petabytes (cheaply) | Auto-scales |
| **Kubernetes Native** | Needs configuration | Excellent | Vendor-specific |
| **Open Source** | Yes (with paid features) | Yes (Apache 2.0) | No |

---

## Real-World Example

### How Uber Handles 200TB of Logs Per Day

```
┌─────────────────────────────────────────────────────────┐
│                UBER'S LOGGING PIPELINE                    │
│                                                          │
│  Source:                                                │
│  • 4,000+ microservices                                 │
│  • 200 TB of logs generated per day                    │
│                                                          │
│  Pipeline:                                              │
│  [Services] → [Kafka] → [Flink Processing] → Storage  │
│                              │                          │
│                    ┌─────────┼─────────┐                │
│                    ▼         ▼         ▼                │
│              [Elasticsearch] [HDFS]  [Real-time         │
│              (7-day hot)    (archive) Analytics]        │
│                                                          │
│  Key Decisions:                                         │
│  • Only index ERROR/WARN in Elasticsearch (costs)      │
│  • Archive ALL logs to HDFS/S3 (compliance)           │
│  • Sample DEBUG logs (keep 1% in production)           │
│  • Kafka decouples producers from consumers            │
└─────────────────────────────────────────────────────────┘
```

### How Shopify Uses Fluentd at Scale

Shopify processes **80,000 requests/second** during Black Friday:
- Every request generates 5-10 log entries
- Fluentd DaemonSets on every Kubernetes node
- Logs flow to **Google Cloud Logging** for hot search
- Archives to **BigQuery** for long-term analytics
- Total: ~50 billion log entries per day during peak

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Logging PII (emails, credit cards)** | GDPR/compliance violation, security risk | Mask sensitive fields in the pipeline |
| **No log retention policy** | Storage costs grow forever | Set TTL: 7 days hot, 30 days warm, 90 days cold |
| **Unstructured text logs** | Can't filter, aggregate, or alert on them | Always use structured JSON logging |
| **Single Elasticsearch node** | Single point of failure, data loss | Minimum 3 nodes with replication |
| **Not rate-limiting log volume** | One chatty service overwhelms the pipeline | Set per-service log quotas |
| **Logging in the hot path** | Synchronous logging adds latency to requests | Use async logging (write to buffer, ship later) |
| **No correlation IDs** | Can't trace a request across services | Add trace_id to every log entry |

---

## When to Use / When NOT to Use

### Use ELK Stack When:
- ✅ You need powerful full-text search across logs
- ✅ You have complex query requirements (regex, aggregations)
- ✅ Your team already knows Elasticsearch
- ✅ You have budget for infrastructure (3+ nodes minimum)

### Use Grafana Loki When:
- ✅ You already use Grafana for dashboards
- ✅ Cost is a primary concern
- ✅ You mostly filter by labels (service, level, environment)
- ✅ You're running on Kubernetes

### Use Cloud Logging (CloudWatch, GCP, Azure) When:
- ✅ You're a small team and don't want to manage infrastructure
- ✅ You're already in that cloud ecosystem
- ✅ Log volume is moderate (<100 GB/day)
- ❌ Avoid for multi-cloud (vendor lock-in)

---

## Key Takeaways

1. **Centralized logging is mandatory** once you have more than one server — you cannot SSH into 50 machines to debug an issue.

2. **ELK is the gold standard** for full-text log search but is expensive to run at scale.

3. **Loki is the modern cost-effective choice** — 10-100x cheaper by only indexing labels, not content.

4. **Always use structured logging (JSON)** — it enables filtering, alerting, and machine processing.

5. **Your logging pipeline needs buffering** — if the backend is down, you must not lose logs. Disk-backed buffers solve this.

6. **Set retention policies from day one** — logs grow fast. 7 days hot, 30 days warm, archive the rest.

7. **Correlation IDs (trace_id) are critical** — they connect logs to traces and let you follow a request across 20 services.

---

## What's Next?

With logging covered, let's move to the second pillar — **Metrics and Dashboards** — where we'll deep-dive into Prometheus for collecting metrics and Grafana for visualizing them.

**Next: [04-metrics-and-dashboards.md](./04-metrics-and-dashboards.md)** — Metrics & Dashboards (Prometheus, Grafana)
