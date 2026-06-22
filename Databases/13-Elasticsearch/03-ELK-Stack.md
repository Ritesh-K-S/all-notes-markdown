# 📊 Chapter 3F.3 — ELK Stack: Elasticsearch + Logstash + Kibana

> **"If Elasticsearch is the brain, Logstash is the nervous system, and Kibana is the eyes. Together, they give you X-ray vision into your entire infrastructure."**

> **Level:** 🟡 Intermediate | 🔥 High Demand
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 3F.1 (ES Architecture), Chapter 3F.2 (Query DSL)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand the **ELK Stack** (now called **Elastic Stack**) and how each component fits
- Build a complete **log ingestion pipeline** with Logstash
- Know how **Beats** (Filebeat, Metricbeat, etc.) simplify data collection
- Create **Kibana dashboards** for real-time monitoring
- Manage index growth with **Index Lifecycle Management (ILM)**
- Design production-grade **observability pipelines** used by Netflix, Uber, and LinkedIn

---

## 🌍 1. The ELK Stack — Big Picture

### 1.1 — What Problem Does It Solve?

Imagine you run **50 microservices** across **200 servers**. Something breaks at 3 AM:

```
❌ Without ELK:
  → SSH into server 1... grep logs... nothing
  → SSH into server 2... grep logs... nothing
  → SSH into server 3... grep logs... maybe?
  → SSH into server 4...
  → 45 minutes later, still searching 😩
  → Meanwhile, thousands of users are affected 🔥

✅ With ELK:
  → Open Kibana dashboard
  → Filter: level=ERROR, last 15 minutes
  → See: payment-service on prod-server-12 threw NullPointerException
  → Click → Full stack trace, correlated logs, timeline
  → Root cause found in 30 seconds ⚡
  → Fix deployed, users happy 🎉
```

### 1.2 — The Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                        THE ELASTIC STACK                             │
│              (formerly ELK Stack, now with Beats)                   │
│                                                                      │
│  ┌──────────┐   ┌──────────┐   ┌───────────────┐   ┌────────────┐ │
│  │          │   │          │   │               │   │            │ │
│  │  BEATS   │──▶│ LOGSTASH │──▶│ELASTICSEARCH  │──▶│  KIBANA    │ │
│  │          │   │          │   │               │   │            │ │
│  │ Collect  │   │ Process  │   │ Store & Search│   │ Visualize  │ │
│  │          │   │ Transform│   │               │   │            │ │
│  └──────────┘   └──────────┘   └───────────────┘   └────────────┘ │
│                                                                      │
│  Lightweight      Heavy-duty      The search       Dashboards,      │
│  data shippers    ETL pipeline    & analytics      alerts, maps     │
│  (on every        (central        engine           & exploration    │
│   server)         processing)                                       │
│                                                                      │
│  🏭 ALTERNATIVE FLOW (simpler):                                     │
│  Beats ─────────────────────────▶ Elasticsearch ──▶ Kibana          │
│  (Beats can send directly to ES — skip Logstash for simple cases)   │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 — When Do You Need What?

| Component | When to Use | When to Skip |
|-----------|------------|--------------|
| **Beats** | Always. Lightweight agents on every server | Never skip — it's the data collector |
| **Logstash** | Complex transformations, multiple sources, enrichment | Simple log forwarding (Beats → ES directly) |
| **Elasticsearch** | Always. It's the core storage and search engine | Never skip |
| **Kibana** | Always. Visualization, dashboards, exploration | If you only use the API programmatically |

---

## 🏭 2. Beats — Lightweight Data Shippers

> **"Beats are single-purpose data shippers. Install them on every server, and they silently collect data."**

### 2.1 — The Beats Family

```
┌─────────────────────────────────────────────────────────────────┐
│                        BEATS FAMILY                              │
│                                                                   │
│  ┌────────────────┐                                              │
│  │   FILEBEAT     │  📄 Collects LOG FILES                       │
│  │                │  → Reads /var/log/*, application logs        │
│  │   Most Popular │  → Tails files like "tail -f" but smarter   │
│  └────────────────┘  → Handles log rotation, multiline events   │
│                                                                   │
│  ┌────────────────┐                                              │
│  │   METRICBEAT   │  📊 Collects SYSTEM & SERVICE METRICS        │
│  │                │  → CPU, memory, disk, network                │
│  │                │  → MySQL, PostgreSQL, MongoDB, Redis metrics │
│  └────────────────┘  → Docker, Kubernetes container stats       │
│                                                                   │
│  ┌────────────────┐                                              │
│  │   PACKETBEAT   │  🌐 Collects NETWORK PACKET DATA             │
│  │                │  → HTTP, MySQL, DNS, TLS traffic             │
│  └────────────────┘  → Like Wireshark but automated             │
│                                                                   │
│  ┌────────────────┐                                              │
│  │   HEARTBEAT    │  💓 UPTIME MONITORING                        │
│  │                │  → Pings URLs, TCP ports, ICMP               │
│  └────────────────┘  → "Is my service alive?" checks            │
│                                                                   │
│  ┌────────────────┐                                              │
│  │   AUDITBEAT    │  🔒 SECURITY AUDIT DATA                      │
│  │                │  → File integrity monitoring                 │
│  └────────────────┘  → Linux audit framework events             │
│                                                                   │
│  ┌────────────────┐                                              │
│  │   WINLOGBEAT   │  🪟 WINDOWS EVENT LOGS                       │
│  │                │  → Windows Event Log collection              │
│  └────────────────┘  → Security, Application, System logs       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 — Filebeat Configuration (Most Common)

```yaml
# filebeat.yml — Ship application logs to Elasticsearch

# ════════════════════ INPUTS ════════════════════
filebeat.inputs:

  # Application logs
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.log                    # Your app logs
      - /var/log/nginx/access.log               # Nginx access logs
    fields:
      service: "my-app"                         # Add custom fields
      environment: "production"
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'    # Java stack traces
    multiline.negate: true                       # (multiline handling)
    multiline.match: after

  # System logs
  - type: log
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/auth.log
    fields:
      service: "system"

# ════════════════════ MODULES ════════════════════
# Pre-built parsers for common log formats
filebeat.modules:
  - module: nginx
    access:
      enabled: true
      var.paths: ["/var/log/nginx/access.log"]
    error:
      enabled: true
      var.paths: ["/var/log/nginx/error.log"]

  - module: mysql
    slowlog:
      enabled: true
      var.paths: ["/var/log/mysql/slow.log"]

# ════════════════════ OUTPUT ════════════════════
# Option A: Send directly to Elasticsearch
output.elasticsearch:
  hosts: ["https://es-node1:9200", "https://es-node2:9200"]
  username: "elastic"
  password: "${ES_PASSWORD}"                    # Use env variable!
  index: "logs-myapp-%{+yyyy.MM.dd}"           # Daily indices

# Option B: Send to Logstash (for processing)
# output.logstash:
#   hosts: ["logstash-server:5044"]

# ════════════════════ PROCESSORS ════════════════════
# Lightweight transformations (runs on the Beat itself)
processors:
  - add_host_metadata: ~                        # Add hostname, OS info
  - add_cloud_metadata: ~                       # Add AWS/GCP/Azure info
  - drop_event:
      when:
        contains:
          message: "healthcheck"                # Drop noisy health checks
```

### 2.3 — Metricbeat Configuration

```yaml
# metricbeat.yml — Collect system and service metrics

metricbeat.modules:

  # System metrics (CPU, memory, disk, network)
  - module: system
    period: 10s                                 # Collect every 10 seconds
    metricsets:
      - cpu
      - memory
      - disk
      - network
      - process
    process.include_top_n:
      by_cpu: 5                                 # Top 5 CPU-consuming processes
      by_memory: 5

  # Docker container metrics
  - module: docker
    period: 10s
    hosts: ["unix:///var/run/docker.sock"]
    metricsets:
      - container
      - cpu
      - memory
      - network

  # MySQL metrics
  - module: mysql
    period: 30s
    hosts: ["tcp(127.0.0.1:3306)/"]
    username: monitor_user
    password: "${MYSQL_PASSWORD}"
    metricsets:
      - status                                  # SHOW GLOBAL STATUS
      - galera_status

  # Redis metrics
  - module: redis
    period: 10s
    hosts: ["127.0.0.1:6379"]
    metricsets:
      - info
      - keyspace

output.elasticsearch:
  hosts: ["https://es-node1:9200"]
  username: "elastic"
  password: "${ES_PASSWORD}"
```

---

## 🔧 3. Logstash — The Data Processing Pipeline

> **"Logstash is your data's personal chef — it takes raw ingredients, cleans them, seasons them, and serves them perfectly plated."**

### 3.1 — Logstash Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      LOGSTASH PIPELINE                           │
│                                                                   │
│   ┌─────────┐      ┌──────────────┐      ┌──────────┐          │
│   │         │      │              │      │          │          │
│   │  INPUT  │─────▶│   FILTER     │─────▶│  OUTPUT  │          │
│   │         │      │              │      │          │          │
│   └─────────┘      └──────────────┘      └──────────┘          │
│                                                                   │
│   Where data       Transform,             Where data            │
│   comes FROM       parse, enrich          goes TO               │
│                                                                   │
│   • Beats          • grok (regex)         • Elasticsearch       │
│   • Kafka          • mutate (rename)      • Kafka               │
│   • TCP/UDP        • date (parse dates)   • S3                  │
│   • HTTP           • geoip (location)     • File                │
│   • File           • json (parse JSON)    • Stdout (debug)      │
│   • JDBC           • dissect (fast parse) • Email/PagerDuty     │
│   • S3             • ruby (custom code)   • Multiple outputs!   │
│   • Syslog         • drop/if-else                               │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 — Basic Logstash Config

```ruby
# /etc/logstash/conf.d/pipeline.conf

# ════════════════════ INPUT ════════════════════
input {
  # Receive from Filebeat
  beats {
    port => 5044
  }

  # Or read from Kafka
  # kafka {
  #   bootstrap_servers => "kafka1:9092,kafka2:9092"
  #   topics => ["app-logs"]
  #   group_id => "logstash-consumers"
  #   codec => json
  # }
}

# ════════════════════ FILTER ════════════════════
filter {

  # Parse JSON logs
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }

  # Parse unstructured logs with Grok
  else {
    grok {
      match => {
        "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} \[%{DATA:service}\] %{GREEDYDATA:log_message}"
      }
    }
    # Input:  "2024-06-15T10:30:45.123Z ERROR [payment-service] Connection timeout to DB"
    # Output: timestamp=2024-06-15T10:30:45.123Z, level=ERROR,
    #         service=payment-service, log_message=Connection timeout to DB
  }

  # Parse the timestamp
  date {
    match => ["timestamp", "ISO8601", "yyyy-MM-dd HH:mm:ss"]
    target => "@timestamp"
  }

  # Add geo data from IP addresses
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geo"
      # Adds: geo.country_name, geo.city_name, geo.location (lat/lon)
    }
  }

  # Add custom fields
  mutate {
    add_field => { "pipeline" => "main" }
    rename => { "host" => "hostname" }
    remove_field => ["agent", "ecs"]           # Remove noisy fields
    lowercase => ["level"]                      # Normalize log level
  }

  # Drop health check noise
  if [log_message] =~ /healthcheck|ping|favicon/ {
    drop { }
  }

  # Conditional routing
  if [level] == "error" {
    mutate {
      add_tag => ["alert"]
    }
  }
}

# ════════════════════ OUTPUT ════════════════════
output {
  # Primary: Elasticsearch
  elasticsearch {
    hosts => ["https://es-node1:9200", "https://es-node2:9200"]
    user => "elastic"
    password => "${ES_PASSWORD}"
    index => "logs-%{[service]}-%{+YYYY.MM.dd}"   # logs-payment-service-2024.06.15
    # This creates daily indices per service!
  }

  # Alert critical errors to Slack/PagerDuty
  if "alert" in [tags] {
    http {
      url => "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
      http_method => "post"
      format => "json"
      mapping => {
        "text" => "🚨 ERROR in %{service}: %{log_message}"
      }
    }
  }

  # Debug: Also print to console
  # stdout { codec => rubydebug }
}
```

### 3.3 — Grok — The Pattern-Matching Superpower

> **Grok is Logstash's regex-on-steroids. It has 120+ pre-built patterns for common log formats.**

```ruby
# Common Grok patterns:
# %{IP:client_ip}           → 192.168.1.1
# %{TIMESTAMP_ISO8601:ts}   → 2024-06-15T10:30:45Z
# %{LOGLEVEL:level}         → ERROR, WARN, INFO
# %{NUMBER:status_code}     → 200, 404, 500
# %{GREEDYDATA:message}     → Everything else
# %{WORD:method}             → GET, POST, PUT
# %{URIPATH:path}            → /api/v2/users
# %{DATA:service}            → Non-greedy match

# ── Example: Parse Nginx access log ──
# Raw log:
# 192.168.1.5 - john [15/Jun/2024:10:30:45 +0000] "GET /api/users HTTP/1.1" 200 1234

grok {
  match => {
    "message" => '%{IPORHOST:client_ip} - %{DATA:user} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATH:path} HTTP/%{NUMBER:http_version}" %{NUMBER:status_code} %{NUMBER:bytes}'
  }
}

# Result:
# client_ip: 192.168.1.5
# user: john
# method: GET
# path: /api/users
# status_code: 200
# bytes: 1234

# ── Example: Parse custom app log ──
# Raw: [2024-06-15 10:30:45] [ERROR] [OrderService] Failed to process order #12345 - insufficient stock

grok {
  match => {
    "message" => '\[%{TIMESTAMP_ISO8601:timestamp}\] \[%{LOGLEVEL:level}\] \[%{DATA:service}\] %{GREEDYDATA:log_message}'
  }
}
```

> 💡 **Pro Tip:** Test your Grok patterns at [grokdebugger.com](https://grokdebugger.com) or in Kibana's **Dev Tools → Grok Debugger** before putting them in production.

### 3.4 — Logstash vs Beats — When to Use What

```
┌──────────────────────────────────────────────────────────────┐
│                   DECISION FLOW                               │
│                                                               │
│  Do you need to transform/enrich data?                       │
│  ├── NO  → Beats → Elasticsearch (skip Logstash)             │
│  └── YES → Do you need complex processing?                   │
│            ├── Light processing → Beats processors → ES      │
│            └── Heavy processing → Beats → Logstash → ES      │
│                                                               │
│  Are you ingesting from non-file sources (Kafka, DB, S3)?    │
│  └── YES → Logstash (Beats only does files/metrics/packets)  │
│                                                               │
│  Do you need multiple outputs (ES + S3 + Kafka)?             │
│  └── YES → Logstash (can fan out to multiple destinations)   │
└──────────────────────────────────────────────────────────────┘
```

| Feature | Beats | Logstash |
|---------|-------|----------|
| **Resource Usage** | ⚡ ~10-30 MB RAM | 💪 ~1-4 GB RAM (JVM) |
| **Processing** | Light (basic processors) | Heavy (grok, geoip, ruby, etc.) |
| **Input Sources** | Files, metrics, packets | Anything (Kafka, DB, HTTP, S3...) |
| **Output Targets** | ES, Logstash, Kafka | Anything (ES, S3, Kafka, email...) |
| **Where to Run** | On every server (agent) | Centralized (1-3 instances) |
| **Scaling** | Scales per server | Scale with load balancing |

---

## 📊 4. Kibana — Visualization & Exploration

> **"Kibana turns your data into stories. Numbers become charts. Logs become timelines. Chaos becomes clarity."**

### 4.1 — Kibana Features Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        KIBANA                                    │
│                                                                   │
│  ┌───────────────────┐  ┌───────────────────┐                   │
│  │    DISCOVER       │  │    DASHBOARD      │                   │
│  │                   │  │                   │                   │
│  │  Explore raw logs │  │  Multi-panel      │                   │
│  │  Search & filter  │  │  dashboards with  │                   │
│  │  View documents   │  │  charts, maps,    │                   │
│  │  Quick analysis   │  │  tables, metrics  │                   │
│  └───────────────────┘  └───────────────────┘                   │
│                                                                   │
│  ┌───────────────────┐  ┌───────────────────┐                   │
│  │    VISUALIZE      │  │      LENS         │                   │
│  │                   │  │                   │                   │
│  │  Build individual │  │  Drag-and-drop    │                   │
│  │  charts: bar,     │  │  smart chart      │                   │
│  │  line, pie, map,  │  │  builder          │                   │
│  │  heatmap, gauge   │  │  (easiest way!)   │                   │
│  └───────────────────┘  └───────────────────┘                   │
│                                                                   │
│  ┌───────────────────┐  ┌───────────────────┐                   │
│  │    DEV TOOLS      │  │    ALERTS         │                   │
│  │                   │  │                   │                   │
│  │  Console: Run ES  │  │  Automated        │                   │
│  │  queries directly │  │  alerting via     │                   │
│  │  Grok debugger    │  │  email, Slack,    │                   │
│  │  Search profiler  │  │  PagerDuty        │                   │
│  └───────────────────┘  └───────────────────┘                   │
│                                                                   │
│  ┌───────────────────┐  ┌───────────────────┐                   │
│  │    MAPS           │  │    CANVAS         │                   │
│  │                   │  │                   │                   │
│  │  Geospatial data  │  │  Pixel-perfect    │                   │
│  │  visualized on    │  │  infographic      │                   │
│  │  real maps        │  │  dashboards       │                   │
│  └───────────────────┘  └───────────────────┘                   │
│                                                                   │
│  ┌───────────────────┐  ┌───────────────────┐                   │
│  │    OBSERVABILITY  │  │    SECURITY       │                   │
│  │                   │  │                   │                   │
│  │  APM: traces,     │  │  SIEM: threat     │                   │
│  │  errors, latency  │  │  detection,       │                   │
│  │  Uptime monitoring│  │  investigation    │                   │
│  └───────────────────┘  └───────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 — Setting Up Kibana (Step-by-Step)

```
Step 1: Access Kibana
  → Open browser: http://localhost:5601

Step 2: Create Data View (Index Pattern)
  → Management → Data Views → Create data view
  → Name: "logs-*"
  → Time field: @timestamp
  → This tells Kibana which ES indices to query

Step 3: Explore in Discover
  → Click "Discover" in sidebar
  → Select your data view "logs-*"
  → Set time range (last 24h, last 7d, etc.)
  → See your logs flowing in real-time!

Step 4: Add filters & search
  → Search bar: "level: ERROR AND service: payment-service"
  → Add field filters from the sidebar
  → Click on any document to see full details

Step 5: Build Visualizations
  → Click "Visualize Library" → Create → Lens
  → Drag fields to build charts
  → Save to a dashboard
```

### 4.3 — Essential Dashboard Panels

Here's what a production monitoring dashboard typically includes:

```
┌─────────────────────────────────────────────────────────────────┐
│  🖥️ PRODUCTION MONITORING DASHBOARD                             │
│  Time Range: Last 24 hours        Auto-refresh: 30 seconds     │
├─────────────────────────┬───────────────────────────────────────┤
│                         │                                       │
│  📊 LOG VOLUME          │  🚦 ERROR RATE                        │
│  ───────────            │  ──────────                           │
│  ▁▂▃▅▇█▇▅▃▂▁▂▃▄▅▇     │  Errors/min: 12  (⬆️ +340%)          │
│  ~~~~~~~~~~~~^~~        │  [LINE CHART: errors over time]       │
│           spike!        │  ████ 🔴 payment-svc: 8/min          │
│                         │  ██   🟡 auth-svc: 3/min             │
│  Total: 2.4M events    │  █    🟢 user-svc: 1/min             │
│                         │                                       │
├─────────────────────────┼───────────────────────────────────────┤
│                         │                                       │
│  📈 RESPONSE TIMES (P95)│  🌍 REQUEST GEO MAP                   │
│  ───────────────────    │  ─────────────────                    │
│  payment-svc:  450ms 🔴│                                       │
│  auth-svc:     120ms 🟢│  [WORLD MAP with dots showing         │
│  user-svc:      80ms 🟢│   request origins by country]         │
│  order-svc:    200ms 🟡│                                       │
│                         │  US: 45% | EU: 30% | Asia: 25%       │
│                         │                                       │
├─────────────────────────┼───────────────────────────────────────┤
│                         │                                       │
│  📋 TOP ERRORS          │  🔄 HTTP STATUS CODES                  │
│  ──────────             │  ────────────────────                 │
│  1. NullPointerException│  [PIE CHART]                          │
│     (payment-svc) x 847 │  🟢 2xx: 94.2%                       │
│  2. ConnectionTimeout   │  🟡 3xx: 2.1%                        │
│     (db-pool) x 234     │  🟠 4xx: 3.0%                        │
│  3. TokenExpired        │  🔴 5xx: 0.7%                        │
│     (auth-svc) x 156    │                                       │
│                         │                                       │
├─────────────────────────┴───────────────────────────────────────┤
│                                                                  │
│  📜 LATEST ERROR LOGS (Live Stream)                              │
│  ───────────────────────────────────                            │
│  10:30:45 ERROR [payment-svc] Connection timeout to DB (30s)    │
│  10:30:44 ERROR [payment-svc] Failed to process order #78921    │
│  10:30:42 WARN  [auth-svc]   Token refresh failed for user_123  │
│  10:30:41 ERROR [payment-svc] Connection timeout to DB (30s)    │
│  10:30:38 ERROR [payment-svc] NullPointerException at Checkout  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 — Kibana Query Language (KQL)

```
# KQL — Kibana's search language (simpler than Query DSL)

# Simple field match
level: ERROR

# Multiple conditions (AND)
level: ERROR AND service: payment-service

# OR conditions
service: payment-service OR service: auth-service

# Wildcards
service: pay*

# Ranges
response_time_ms > 500
price >= 100 AND price <= 500

# NOT
NOT level: DEBUG

# Nested fields
geo.country_name: "United States"

# Exists check
tags: *

# Complex query
level: ERROR AND service: payment-service AND response_time_ms > 1000 AND NOT message: "healthcheck"
```

---

## 🔄 5. Index Lifecycle Management (ILM)

> **"Without ILM, your indices grow forever, eat all your disk, and your cluster dies. ILM is your janitor."**

### 5.1 — The Problem

```
Day 1:    logs-2024-06-01  → 5 GB    (hot, actively searched)
Day 30:   logs-2024-06-01  → 5 GB    (warm, occasionally searched)
Day 90:   logs-2024-06-01  → 5 GB    (cold, rarely accessed)
Day 365:  logs-2024-06-01  → 5 GB    (frozen, almost never accessed)
Day 400:  logs-2024-06-01  → 5 GB    (⚠️ WHY is this still here?!)

After 1 year: 365 × 5 GB = 1.8 TB of logs 😱
After 3 years: 5.4 TB → ❌ Cluster is full, alerts firing!
```

### 5.2 — ILM Phases

```
┌──────────────────────────────────────────────────────────────────┐
│                    INDEX LIFECYCLE PHASES                         │
│                                                                   │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐     │
│  │          │   │          │   │          │   │          │     │
│  │   HOT    │──▶│   WARM   │──▶│   COLD   │──▶│  DELETE  │     │
│  │          │   │          │   │          │   │          │     │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘     │
│                                                                   │
│  🔥 Active       🟡 Less         🧊 Rarely       🗑️ Goodbye!    │
│  indexing &      frequent        accessed.                       │
│  searching.      searches.       Compressed.     Deleted after   │
│  Fast SSDs.      Can shrink      Cheap storage.  retention.     │
│  Full replicas.  replicas.       Frozen option.                  │
│                                                                   │
│  Day 0-7         Day 7-30        Day 30-90       Day 90+        │
│  (example)       (example)       (example)       (example)      │
└──────────────────────────────────────────────────────────────────┘
```

### 5.3 — Creating an ILM Policy

```json
// Create ILM policy: "logs-policy"
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",                 // Create new index when current hits 50GB
            "max_age": "1d"                      // OR when it's 1 day old
          },
          "set_priority": {
            "priority": 100                      // Highest priority for recovery
          }
        }
      },
      "warm": {
        "min_age": "7d",                         // Move to warm after 7 days
        "actions": {
          "shrink": {
            "number_of_shards": 1                // Reduce shards (save resources)
          },
          "forcemerge": {
            "max_num_segments": 1                // Merge segments (save disk)
          },
          "allocate": {
            "number_of_replicas": 1              // Keep 1 replica
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",                        // Move to cold after 30 days
        "actions": {
          "allocate": {
            "number_of_replicas": 0,             // No replicas (save disk)
            "require": {
              "data": "cold"                     // Move to cold-tier nodes
            }
          },
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",                        // DELETE after 90 days
        "actions": {
          "delete": {}                           // 🗑️ Goodbye!
        }
      }
    }
  }
}
```

### 5.4 — Applying ILM to Indices

```json
// Step 1: Create an index template with the ILM policy
PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",            // Attach ILM policy
      "index.lifecycle.rollover_alias": "logs-active"   // Alias for rollover
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "service": { "type": "keyword" },
        "message": { "type": "text" }
      }
    }
  }
}

// Step 2: Create the initial index with the rollover alias
PUT /logs-000001
{
  "aliases": {
    "logs-active": {
      "is_write_index": true                           // Write to this index
    }
  }
}

// Step 3: All writes go to the alias
POST /logs-active/_doc
{
  "@timestamp": "2024-06-15T10:30:00Z",
  "level": "ERROR",
  "service": "payment-service",
  "message": "Connection timeout"
}
// ILM automatically rolls over, moves phases, and deletes old data! 🎉
```

### 5.5 — Monitor ILM Status

```json
// Check ILM status for an index
GET /logs-000001/_ilm/explain

// Response:
{
  "indices": {
    "logs-000001": {
      "index": "logs-000001",
      "managed": true,
      "policy": "logs-policy",
      "lifecycle_date_millis": 1718441400000,
      "phase": "warm",                          // Currently in warm phase
      "phase_time_millis": 1718527800000,
      "action": "complete",
      "step": "complete"
    }
  }
}

// View all ILM policies
GET /_ilm/policy
```

---

## 🏗️ 6. Production Architecture Patterns

### 6.1 — Small Setup (Startups, < 10 GB/day)

```
┌──────────────────────────────────────────────────────┐
│                SMALL SETUP                            │
│                                                       │
│  App Servers (3)          Elastic Stack (3 nodes)     │
│  ┌──────────┐            ┌───────────────────────┐   │
│  │ App + FB │──────────▶│ ES Node 1 (master+data)│   │
│  └──────────┘            ├───────────────────────┤   │
│  ┌──────────┐            │ ES Node 2 (master+data)│   │
│  │ App + FB │──────────▶├───────────────────────┤   │
│  └──────────┘            │ ES Node 3 (master+data)│   │
│  ┌──────────┐            └───────────┬───────────┘   │
│  │ App + FB │──────────▶            │               │
│  └──────────┘                       ▼               │
│                              ┌──────────┐            │
│  FB = Filebeat               │  Kibana  │            │
│  No Logstash needed!         └──────────┘            │
│                                                       │
│  Cost: ~$500-1000/month (cloud)                      │
│  Beats send directly to ES (simple, efficient)       │
└──────────────────────────────────────────────────────┘
```

### 6.2 — Medium Setup (Mid-size, 10-100 GB/day)

```
┌────────────────────────────────────────────────────────────────┐
│                    MEDIUM SETUP                                 │
│                                                                 │
│  App Servers (20)     Logstash (2)        ES Cluster (6)       │
│  ┌───────┐           ┌──────────┐        ┌────────────────┐   │
│  │App+FB │──┐        │ Logstash │───────▶│ Master × 3     │   │
│  └───────┘  │   ┌───▶│ (Parse,  │        │ (dedicated)    │   │
│  ┌───────┐  ├───┤    │  enrich) │        ├────────────────┤   │
│  │App+FB │──┤   │    └──────────┘        │ Data × 5       │   │
│  └───────┘  │   │    ┌──────────┐   ┌───▶│ (hot + warm)   │   │
│  ┌───────┐  │   │    │ Logstash │───┘    └───────┬────────┘   │
│  │App+FB │──┘   └───▶│ (HA/LB)  │               │            │
│  └───────┘           └──────────┘               ▼            │
│  ... × 20                                 ┌──────────┐        │
│                                           │  Kibana  │        │
│  + Metricbeat on all servers              │ (HA × 2) │        │
│  + Heartbeat for uptime                   └──────────┘        │
│                                                                 │
│  Cost: ~$3,000-8,000/month (cloud)                             │
└────────────────────────────────────────────────────────────────┘
```

### 6.3 — Large Setup (Enterprise, 100+ GB/day)

```
┌──────────────────────────────────────────────────────────────────┐
│                      LARGE SETUP                                  │
│                                                                   │
│  App Servers      Kafka          Logstash        ES Cluster       │
│  (100+)           (Buffer)       (Workers)       (20+ nodes)      │
│                                                                   │
│  ┌─────┐        ┌─────────┐    ┌──────────┐   ┌──────────────┐  │
│  │FB/MB│───────▶│  Kafka  │───▶│ Logstash │──▶│ Coordinating │  │
│  └─────┘        │  Cluster│    │ × 4-8    │   │ × 2          │  │
│  ┌─────┐        │  (3-5   │    │ (parallel│   ├──────────────┤  │
│  │FB/MB│───────▶│  brokers)│    │  workers)│   │ Master × 3   │  │
│  └─────┘        └─────────┘    └──────────┘   │ (dedicated)  │  │
│  ... × 100                                     ├──────────────┤  │
│                                                │ Hot Data × 6 │  │
│  WHY KAFKA?                                    │ (NVMe SSD)   │  │
│  → Buffer spikes                               ├──────────────┤  │
│  → Logstash can restart                        │ Warm Data × 4│  │
│    without losing data                         │ (SSD)        │  │
│  → Multiple consumers                          ├──────────────┤  │
│    (ES + S3 + analytics)                       │ Cold Data × 4│  │
│                                                │ (HDD)        │  │
│                                                └──────┬───────┘  │
│                                                       │          │
│                                        ┌──────────┐   │          │
│                                        │  Kibana  │◀──┘          │
│                                        │  (HA)    │              │
│                                        └──────────┘              │
│                                                                   │
│  Cost: ~$15,000-50,000+/month (cloud)                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔒 7. Security Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│                 ELASTIC STACK SECURITY CHECKLIST              │
│                                                               │
│  ✅ Enable authentication (built-in since 8.x)              │
│     → xpack.security.enabled: true                          │
│     → Create separate users: admin, reader, writer          │
│                                                               │
│  ✅ Enable TLS/SSL everywhere                                │
│     → Node-to-node encryption (transport layer)             │
│     → Client-to-node encryption (HTTPS)                     │
│     → Beats → Logstash → ES (all encrypted)                │
│                                                               │
│  ✅ Role-Based Access Control (RBAC)                         │
│     → dev-team: read-only on dev-* indices                  │
│     → ops-team: read/write on logs-*, metrics-*             │
│     → admin: full access (use sparingly)                    │
│                                                               │
│  ✅ API Key authentication for services                      │
│     → Don't hardcode passwords in Beat/Logstash configs     │
│     → Use API keys or environment variables                 │
│                                                               │
│  ✅ Network security                                         │
│     → ES should NOT be exposed to the internet              │
│     → Use VPN/private network for cluster communication     │
│     → Kibana behind reverse proxy with auth                 │
│                                                               │
│  ✅ Audit logging                                            │
│     → Log who accessed what data and when                   │
│     → Required for compliance (GDPR, HIPAA, SOC2)          │
│                                                               │
│  ✅ Field-level security                                     │
│     → Hide PII fields (email, SSN) from certain roles      │
│     → Document-level security (see only your team's data)  │
└──────────────────────────────────────────────────────────────┘
```

---

## ⚡ 8. Performance Tuning Tips

### 8.1 — Indexing Performance

| Tip | Impact | How |
|-----|--------|-----|
| Use `_bulk` API | 🔥🔥🔥 | Batch 1000-5000 docs per bulk request |
| Increase `refresh_interval` | 🔥🔥 | Set to `30s` or `60s` for log indices |
| Disable replicas during bulk load | 🔥🔥 | `number_of_replicas: 0`, re-enable after |
| Use `_routing` for related docs | 🔥 | Co-locate related docs on same shard |
| Size bulk requests by MB, not count | 🔥 | Target 5-15 MB per bulk request |

### 8.2 — Search Performance

| Tip | Impact | How |
|-----|--------|-----|
| Use `filter` over `must` | 🔥🔥🔥 | Filters are cached and don't score |
| Avoid `wildcard` with leading `*` | 🔥🔥 | `*samsung` → full index scan! |
| Use `_source` filtering | 🔥🔥 | Return only fields you need |
| Set `size: 0` for agg-only queries | 🔥 | Skip returning search hits |
| Use `search_after` not `from/size` | 🔥 | For deep pagination (> 10K results) |
| Pre-warm caches with `terms` enum | 🔥 | Warm fielddata cache for heavy aggs |

### 8.3 — Cluster Health Tips

```
Rule of Thumb: Hardware Sizing

  Memory:  64 GB RAM per data node (31 GB to JVM heap, rest for OS cache)
  Storage: SSD for hot data, HDD OK for cold data
  CPU:     16+ cores for data nodes
  Network: 10 Gbps between nodes (minimum 1 Gbps)

  JVM Heap: NEVER exceed 50% of RAM, NEVER exceed 31 GB
    → 64 GB RAM → heap = 31 GB (sweet spot)
    → 32 GB RAM → heap = 16 GB
    → Rest goes to Lucene's OS page cache (critical!)
```

---

## 🆚 9. Elastic Stack vs Alternatives

| Feature | ELK Stack | Splunk | Grafana + Loki | Datadog |
|---------|-----------|--------|----------------|---------|
| **Cost** | Free (self-managed) / Paid cloud | 💰💰💰 Expensive | Free (self-managed) | 💰💰 SaaS |
| **Search Power** | ⭐ Best (full Lucene) | ✅ Excellent | 🟡 Basic (label-based) | ✅ Good |
| **Scalability** | ✅ Horizontal | ✅ Horizontal | ✅ Horizontal | ✅ Managed |
| **Learning Curve** | 🟡 Moderate | 🟡 Moderate | 🟢 Easy | 🟢 Easy |
| **Setup Complexity** | 🔴 Complex (self) | 🟡 Moderate | 🟡 Moderate | 🟢 SaaS |
| **Full-Text Search** | ⭐ Unbeatable | ✅ Great | ❌ Limited | 🟡 Basic |
| **APM** | ✅ Elastic APM | ✅ Splunk APM | ✅ Tempo | ✅ Native |
| **Best For** | Search + Logs + SIEM | Enterprise + Compliance | Metrics + Light logs | SaaS simplicity |

---

## 🐳 10. Quick Start — Docker Compose (Full Stack)

```yaml
# docker-compose.yml — Complete ELK Stack in 5 minutes

version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false          # Disable for local dev
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"                             # Beats input
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

```bash
# Start the entire stack
docker-compose up -d

# Verify:
# Elasticsearch: http://localhost:9200
# Kibana:        http://localhost:5601

# Stop:
docker-compose down

# Stop + delete data:
docker-compose down -v
```

---

## 🔑 Key Takeaways

```
✅ ELK = Elasticsearch (store/search) + Logstash (process) + Kibana (visualize)
✅ Beats are lightweight agents on every server (Filebeat, Metricbeat, etc.)
✅ Beats can send directly to ES (simple) or through Logstash (complex processing)
✅ Logstash pipeline: Input → Filter (grok, mutate, geoip) → Output
✅ Grok patterns parse unstructured logs into structured fields
✅ Kibana provides Discover (explore), Visualize (charts), Dashboard (combine)
✅ KQL is Kibana's search language — simpler than full Query DSL
✅ ILM manages index lifecycle: Hot → Warm → Cold → Delete (saves disk $$$)
✅ Production: Kafka as buffer between Beats and Logstash for resilience
✅ Security: Always enable TLS, RBAC, audit logging in production
✅ JVM Heap: Never exceed 31 GB (compressed oops limit) or 50% of RAM
✅ Alternatives exist (Loki, Splunk, Datadog) but ELK remains the king of search
```

---

## ❓ Self-Check Questions

1. What are the 4 components of the Elastic Stack? What does each do?
2. When should you use Logstash vs sending Beats directly to Elasticsearch?
3. Write a Grok pattern to parse: `[2024-06-15 10:30:00] ERROR payment-svc: timeout`
4. What are the ILM phases? Give a realistic timeline for log data.
5. Why is Kafka often placed between Beats and Logstash in large deployments?
6. What's the maximum JVM heap size you should set for Elasticsearch and why?
7. How would you create a Kibana dashboard showing error rates per service over time?
8. What's the difference between `rollover` and `shrink` in ILM?
9. Name 3 security measures you'd implement for a production ELK deployment.
10. Your cluster is at 90% disk usage and growing. What ILM strategy would you implement?

---

## 🎯 Hands-On Lab

```
Lab 1: Deploy ELK with Docker Compose
  → Use the docker-compose.yml from Section 10
  → Verify all 3 services are running

Lab 2: Configure Filebeat to ship system logs
  → Install Filebeat → Point to /var/log/syslog
  → Output to your local Elasticsearch
  → Verify in Kibana Discover

Lab 3: Build a Logstash pipeline
  → Input: Read a sample CSV or JSON file
  → Filter: Parse with grok/json, add geoip, mutate fields
  → Output: Send to Elasticsearch
  → Verify fields are correctly parsed

Lab 4: Create a Kibana Dashboard
  → Panel 1: Line chart of log volume over time
  → Panel 2: Pie chart of log levels (INFO/WARN/ERROR)
  → Panel 3: Table of top 10 error messages
  → Panel 4: Metric showing total document count
  → Set auto-refresh to 30 seconds

Lab 5: Configure ILM
  → Create a policy: hot (7d) → warm (30d) → delete (90d)
  → Apply to an index template
  → Create a rollover alias
  → Verify with GET /_ilm/explain
```

---

> **🎉 Congratulations!** You've completed the Elasticsearch trilogy!
>
> You now understand:
> - **3F.1:** How ES works internally (inverted index, shards, mappings, analyzers)
> - **3F.2:** How to query anything (Query DSL, Bool, Aggregations, Suggesters)
> - **3F.3:** How to build a complete observability pipeline (Beats + Logstash + Kibana + ILM)
>
> **Next up in 3G:** [Other Important NoSQL & Specialty Databases](../14-Other-Databases/01-DynamoDB.md) — DynamoDB, Time-Series DBs, Vector DBs, and more!
