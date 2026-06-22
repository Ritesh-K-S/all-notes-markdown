# ⏱️ Chapter 3G.3 — InfluxDB & Time-Series Databases

> **Level:** 🟡 Intermediate | 🔥 High Demand
> **Time to Master:** ~4–5 hours
> **Prerequisites:** Chapter 3A.1 (NoSQL Overview), Chapter 1.7 (Indexing Deep Dive)

> **"Every second, the world generates 127 new IoT devices. Every one of them produces time-stamped data. Welcome to the Time-Series revolution."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **what Time-Series data is** and why it needs a specialized database
- Know **why traditional databases FAIL** at time-series workloads
- Master **InfluxDB architecture** — the most popular open-source TSDB
- Write queries in **InfluxQL** (SQL-like) and **Flux** (functional)
- Understand **Retention Policies** — auto-delete old data
- Master **Downsampling** — keep summaries, discard raw data
- Apply time-series patterns to **IoT, monitoring, finance, and analytics**
- Compare InfluxDB with **TimescaleDB, Prometheus, QuestDB, and others**

---

## 1. What Is Time-Series Data? — It's Everywhere

### The Definition

**Time-Series Data** = Data points indexed by **time**, collected at regular (or irregular) intervals, usually **append-only** (you rarely update old data).

```
Think of it as an infinite stream of (timestamp, value) pairs:

  2024-06-15 10:00:00  →  CPU: 45%    Memory: 72%    Disk: 60%
  2024-06-15 10:00:01  →  CPU: 47%    Memory: 72%    Disk: 60%
  2024-06-15 10:00:02  →  CPU: 52%    Memory: 73%    Disk: 60%
  2024-06-15 10:00:03  →  CPU: 89%    Memory: 78%    Disk: 60%   ← SPIKE!
  2024-06-15 10:00:04  →  CPU: 91%    Memory: 80%    Disk: 60%   ← ALERT!
  ...
  (This goes on FOREVER. 86,400 data points per metric per day.)
  (With 1000 servers, that's 86.4 MILLION points/day for ONE metric.)
```

### Time-Series Data Is EVERYWHERE

```
┌─────────────────────────────────────────────────────────────────┐
│              Where Time-Series Data Lives                       │
├──────────────────┬──────────────────────────────────────────────┤
│                  │                                              │
│  🖥️ DevOps &     │  CPU, Memory, Disk, Network traffic,       │
│     Monitoring   │  Request latency, Error rates, Uptime       │
│                  │                                              │
│  🏭 IoT &        │  Temperature, Pressure, Humidity, Vibration,│
│     Industrial   │  Power consumption, Machine status          │
│                  │                                              │
│  📈 Finance      │  Stock prices, Trade volumes, Exchange       │
│                  │  rates, Order book snapshots                 │
│                  │                                              │
│  🌤️ Weather      │  Temperature, Wind speed, Precipitation,    │
│                  │  Air quality index, UV levels               │
│                  │                                              │
│  📱 App Analytics│  Active users, Page views, Click rates,     │
│                  │  Session duration, Conversion funnels       │
│                  │                                              │
│  🏥 Healthcare   │  Heart rate, Blood pressure, ECG readings,  │
│                  │  Patient vitals over time                   │
│                  │                                              │
│  🚗 Autonomous   │  GPS position, Speed, LIDAR readings,      │
│     Vehicles     │  Brake/accelerator events per millisecond   │
│                  │                                              │
│  ⚡ Smart Grid   │  Energy production/consumption per second,  │
│                  │  Grid frequency, Voltage levels             │
└──────────────────┴──────────────────────────────────────────────┘
```

---

## 2. Why Traditional Databases Fail at Time-Series

### The Experiment — Let's Use PostgreSQL

```sql
-- "Let's just use PostgreSQL. How hard can it be?"

CREATE TABLE metrics (
    timestamp TIMESTAMPTZ NOT NULL,
    host VARCHAR(255),
    metric_name VARCHAR(255),
    value DOUBLE PRECISION
);

-- Insert 1 second of data (10 servers × 5 metrics = 50 rows)
INSERT INTO metrics VALUES ('2024-06-15 10:00:00', 'web-01', 'cpu', 45.2);
INSERT INTO metrics VALUES ('2024-06-15 10:00:00', 'web-01', 'memory', 72.1);
-- ... 48 more rows

-- Now do this EVERY SECOND, 24/7, for 1000 servers, 50 metrics...
```

```
After 1 day:   50 metrics × 1000 servers × 86400 seconds = 4.32 BILLION rows
After 1 week:  30.24 BILLION rows
After 1 month: 129.6 BILLION rows 😱

Problems with PostgreSQL at this scale:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ INSERT Performance:                                         │
│     B-Tree index updates on every insert → gets slower          │
│     MVCC overhead → dead tuples pile up                        │
│     VACUUM can't keep up → table bloat                         │
│                                                                 │
│  ❌ QUERY Performance:                                         │
│     "Average CPU last 1 hour" → scans millions of rows         │
│     No built-in downsampling → must aggregate manually         │
│     JOINs make sense for relational data, not metrics          │
│                                                                 │
│  ❌ STORAGE:                                                    │
│     Row-based storage → stores "host", "metric" for EVERY row  │
│     Each row has 23-byte tuple header (PostgreSQL overhead)     │
│     No automatic compression for repeating values              │
│                                                                 │
│  ❌ LIFECYCLE:                                                  │
│     "Delete data older than 30 days" → massive DELETE           │
│     operation that locks the table and takes hours             │
│     No built-in retention policies                             │
│                                                                 │
│  BOTTOM LINE: PostgreSQL WORKS but is 10-100x less efficient   │
│  than a purpose-built time-series database for this workload.  │
└─────────────────────────────────────────────────────────────────┘
```

### What a Time-Series Database Does Differently

```
┌──────────────────────┬────────────────────┬────────────────────┐
│  Feature             │  Traditional DB    │  Time-Series DB    │
├──────────────────────┼────────────────────┼────────────────────┤
│  Write Pattern       │  Random writes     │  Append-only       │
│  Write Speed         │  ~10K rows/sec     │  ~1M+ points/sec  │
│  Storage Format      │  Row-based         │  Columnar/Custom   │
│  Compression         │  Generic           │  Time-aware        │
│                      │  (LZ4, zstd)       │  (Gorilla, Delta,  │
│                      │                    │   XOR encoding)    │
│  Compression Ratio   │  2-5x              │  10-50x            │
│  Time-range Queries  │  Full index scan   │  O(1) chunk lookup │
│  Aggregation         │  Manual GROUP BY   │  Built-in functions│
│  Data Retention      │  Manual DELETE     │  Auto-expiry!      │
│  Downsampling        │  Manual ETL jobs   │  Built-in!         │
│  Cardinality         │  Unlimited         │  Optimized for tags│
└──────────────────────┴────────────────────┴────────────────────┘
```

### How Time-Series Databases Achieve 10-50x Compression

```
Traditional Row Storage (PostgreSQL):
──────────────────────────────────────
Row 1: [timestamp=2024-06-15T10:00:00] [host=web-01] [cpu=45.2]
Row 2: [timestamp=2024-06-15T10:00:01] [host=web-01] [cpu=45.5]
Row 3: [timestamp=2024-06-15T10:00:02] [host=web-01] [cpu=45.3]
Row 4: [timestamp=2024-06-15T10:00:03] [host=web-01] [cpu=45.8]

→ "web-01" stored 4 times. Timestamp fully stored each time.
→ ~100 bytes per row × billions of rows = TERABYTES

Time-Series Columnar Storage:
──────────────────────────────
Series Key: {host=web-01, metric=cpu}     ← Stored ONCE

Timestamps: [10:00:00, 10:00:01, 10:00:02, 10:00:03]
  → Delta encoding: [base, +1s, +1s, +1s]     ← 4 bytes instead of 32!

Values: [45.2, 45.5, 45.3, 45.8]
  → XOR encoding: [45.2, XOR-diff, XOR-diff, XOR-diff]
  → Similar values compress to just a few bits!

→ Same data: ~8 bytes instead of 400 bytes = 50x compression! 🚀
```

---

## 3. InfluxDB — The #1 Open-Source Time-Series Database

### What Is InfluxDB?

```
┌─────────────────────────────────────────────────────────────────┐
│                       InfluxDB                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  • Purpose-built time-series database                          │
│  • Written in Go (fast, single binary)                         │
│  • Open-source (MIT license)                                   │
│  • Part of the TICK Stack (Telegraf, InfluxDB, Chronograf,     │
│    Kapacitor) — now evolved to InfluxDB 3.x                   │
│  • Handles millions of writes per second                       │
│  • Built-in retention policies & downsampling                  │
│  • Two query languages: InfluxQL (SQL-like) & Flux (functional)│
│  • Created by InfluxData (founded 2012)                        │
│                                                                 │
│  Used By:                                                      │
│  • eBay, Cisco, IBM, Tesla, Airbus, PayPal, Siemens           │
│  • Any company doing monitoring, IoT, or real-time analytics  │
│                                                                 │
│  DB-Engines Ranking: #1 Time-Series Database (since 2016)     │
└─────────────────────────────────────────────────────────────────┘
```

### InfluxDB Versions — Know the Differences

```
┌────────────────┬────────────────────┬────────────────────────────┐
│  Version       │  Query Language    │  Storage Engine            │
├────────────────┼────────────────────┼────────────────────────────┤
│  InfluxDB 1.x  │  InfluxQL (SQL)   │  TSM (Time-Structured     │
│  (Legacy)      │                    │  Merge Tree)               │
│                │                    │                            │
│  InfluxDB 2.x  │  Flux (functional)│  TSM + unified platform    │
│  (Current OSS) │  + InfluxQL       │  (tasks, dashboards, API)  │
│                │                    │                            │
│  InfluxDB 3.x  │  SQL + InfluxQL   │  Apache Arrow + Parquet   │
│  (Cloud/Edge)  │  + Flux           │  (columnar, blazing fast)  │
│                │                    │                            │
├────────────────┴────────────────────┴────────────────────────────┤
│  💡 This chapter covers InfluxDB 2.x (most widely deployed)    │
│     with notes on 1.x and 3.x where relevant.                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. InfluxDB Data Model — Think Differently

### The Core Concepts

```
In SQL you think:    Database → Table → Row → Column
In InfluxDB:         Bucket → Measurement → Point → Field/Tag

┌─────────────────────────────────────────────────────────────────┐
│                 InfluxDB Data Model                             │
├──────────────┬──────────────────────────────────────────────────┤
│  Bucket      │  Container for data (like a database)           │
│              │  Has a retention policy attached                 │
│              │                                                  │
│  Measurement │  Like a SQL table — logical grouping of data    │
│              │  Example: "cpu", "memory", "temperature"        │
│              │                                                  │
│  Tag         │  Indexed metadata (key-value string pairs)      │
│  (TAG SET)   │  Used for filtering & grouping                  │
│              │  Example: host=web-01, region=us-east           │
│              │  ⚡ INDEXED — fast to query!                     │
│              │                                                  │
│  Field       │  The actual data values (key-value pairs)       │
│  (FIELD SET) │  Example: cpu_usage=45.2, memory_used=72.1     │
│              │  ❌ NOT INDEXED — can't efficiently filter      │
│              │                                                  │
│  Timestamp   │  When the data point was recorded               │
│              │  Nanosecond precision!                          │
│              │                                                  │
│  Point       │  A single data record (measurement + tags +    │
│              │  fields + timestamp)                            │
│              │                                                  │
│  Series      │  Unique combination of measurement + tag set   │
│              │  Example: cpu{host=web-01, region=us-east}      │
│              │  This is the fundamental unit of storage        │
└──────────────┴──────────────────────────────────────────────────┘
```

### Visual Example

```
Measurement: "server_metrics"

┌──────────────────────────────────────────────────────────────────┐
│ Timestamp              │ Tags                │ Fields            │
│                        │ (indexed)           │ (not indexed)     │
├────────────────────────┼─────────────────────┼───────────────────┤
│ 2024-06-15T10:00:00Z   │ host=web-01         │ cpu=45.2          │
│                        │ region=us-east      │ memory=72.1       │
│                        │ env=production      │ disk=60.0         │
├────────────────────────┼─────────────────────┼───────────────────┤
│ 2024-06-15T10:00:00Z   │ host=web-02         │ cpu=62.8          │
│                        │ region=us-east      │ memory=80.5       │
│                        │ env=production      │ disk=45.0         │
├────────────────────────┼─────────────────────┼───────────────────┤
│ 2024-06-15T10:00:01Z   │ host=web-01         │ cpu=47.1          │
│                        │ region=us-east      │ memory=72.3       │
│                        │ env=production      │ disk=60.0         │
└────────────────────────┴─────────────────────┴───────────────────┘

Series 1: server_metrics{host=web-01, region=us-east, env=production}
Series 2: server_metrics{host=web-02, region=us-east, env=production}

Each series is stored contiguously on disk → sequential reads → FAST!
```

### Tags vs Fields — The Critical Decision

```
┌──────────────────────────────────────────────────────────────────┐
│          Tags vs Fields — GET THIS RIGHT!                       │
├──────────────────┬────────────────────┬──────────────────────────┤
│  Aspect          │  Tags              │  Fields                  │
├──────────────────┼────────────────────┼──────────────────────────┤
│  Indexed?        │  ✅ YES            │  ❌ NO                  │
│  Data Type       │  String only       │  Float, Int, String, Bool│
│  Used in WHERE?  │  ✅ Fast filter    │  ⚠️ Slow (full scan)    │
│  Used in GROUP?  │  ✅ Yes            │  ❌ No                  │
│  Cardinality     │  Keep LOW          │  Can be anything         │
│  Example         │  host, region,     │  cpu_usage, temperature, │
│                  │  sensor_id, env    │  response_time, count    │
├──────────────────┴────────────────────┴──────────────────────────┤
│                                                                  │
│  RULE OF THUMB:                                                  │
│  • Will you WHERE/GROUP BY this value? → TAG                    │
│  • Is it metadata that describes the source? → TAG              │
│  • Is it the actual measurement/number? → FIELD                 │
│  • Does it have millions of unique values? → FIELD (not tag!)   │
│                                                                  │
│  ⚠️ HIGH CARDINALITY TAGS = PERFORMANCE KILLER!                 │
│     tag: user_id (millions of values) → BAD! Use as field.     │
│     tag: host (hundreds of values) → GOOD!                     │
│     tag: region (5-10 values) → PERFECT!                       │
└──────────────────────────────────────────────────────────────────┘
```

> ⚠️ **The #1 InfluxDB Mistake**: Using high-cardinality values (like user IDs, email addresses, or UUIDs) as tags. Each unique tag combination creates a new **series**. Millions of series = memory explosion and query slowdown. This is called the **"cardinality bomb"**.

---

## 5. Writing Data to InfluxDB

### Line Protocol — The Wire Format

InfluxDB uses a compact text-based format called **Line Protocol** for ingestion:

```
Format:
  measurement,tag1=val1,tag2=val2 field1=value1,field2=value2 timestamp

Examples:
  cpu,host=web-01,region=us-east usage=45.2,idle=54.8 1718438400000000000
  │    │                         │                     │
  │    │                         │                     └── Timestamp (nanoseconds)
  │    │                         └── Fields (the data)
  │    └── Tags (indexed metadata)
  └── Measurement name

Rules:
  • Measurement: no spaces (use underscores)
  • Tags: comma-separated, no spaces around = or ,
  • Fields: comma-separated, strings must be "quoted"
  • Timestamp: optional (defaults to server time), nanosecond Unix epoch
  • At least ONE field is required (tags are optional)
```

### Writing Data — Multiple Methods

```bash
# ── Method 1: HTTP API (InfluxDB 2.x) ──
curl -X POST "http://localhost:8086/api/v2/write?org=myorg&bucket=mybucket" \
  -H "Authorization: Token my-token" \
  -H "Content-Type: text/plain" \
  --data-raw '
cpu,host=web-01,region=us-east usage=45.2,idle=54.8 1718438400000000000
cpu,host=web-02,region=us-east usage=62.8,idle=37.2 1718438400000000000
memory,host=web-01 used=72.1,available=27.9 1718438400000000000
memory,host=web-02 used=80.5,available=19.5 1718438400000000000
'

# ── Method 2: InfluxDB CLI ──
influx write \
  --bucket mybucket \
  --org myorg \
  "cpu,host=web-01 usage=45.2 1718438400000000000"
```

```python
# ── Method 3: Python Client ──
from influxdb_client import InfluxDBClient, Point, WritePrecision
from influxdb_client.client.write_api import SYNCHRONOUS
from datetime import datetime

client = InfluxDBClient(
    url="http://localhost:8086",
    token="my-token",
    org="myorg"
)
write_api = client.write_api(write_options=SYNCHRONOUS)

# Using Point builder (recommended)
point = Point("server_metrics") \
    .tag("host", "web-01") \
    .tag("region", "us-east") \
    .tag("env", "production") \
    .field("cpu_usage", 45.2) \
    .field("memory_used", 72.1) \
    .field("disk_used", 60.0) \
    .time(datetime.utcnow(), WritePrecision.NS)

write_api.write(bucket="mybucket", record=point)

# Batch writing (high performance)
points = []
for i in range(10000):
    p = Point("sensor_data") \
        .tag("sensor_id", f"sensor-{i % 100}") \
        .tag("building", "HQ") \
        .field("temperature", 22.5 + (i % 10) * 0.1) \
        .field("humidity", 45.0 + (i % 5) * 0.5)
    points.append(p)

write_api.write(bucket="mybucket", record=points)  # Sent in efficient batches
```

### Telegraf — The Data Collection Agent

```
┌─────────────────────────────────────────────────────────────────┐
│                     Telegraf                                    │
│                                                                 │
│  Telegraf is InfluxDB's official data collection agent.         │
│  It collects metrics from 300+ sources and writes to InfluxDB. │
│                                                                 │
│  ┌──────────────┐                                              │
│  │  System      │──┐                                           │
│  │  (CPU/Mem)   │  │                                           │
│  ├──────────────┤  │    ┌──────────┐     ┌──────────┐         │
│  │  Docker      │──┼───►│ Telegraf │────►│ InfluxDB │         │
│  ├──────────────┤  │    │ (Agent)  │     │          │         │
│  │  NGINX       │──┤    └──────────┘     └──────────┘         │
│  ├──────────────┤  │                                           │
│  │  PostgreSQL  │──┤    Telegraf handles:                      │
│  ├──────────────┤  │    • Collection interval                  │
│  │  MQTT/Kafka  │──┘    • Buffering & batching                │
│  └──────────────┘       • Transformation & filtering          │
│                          • Authentication & retry              │
│                                                                │
│  Zero code! Just configure telegraf.conf:                      │
└─────────────────────────────────────────────────────────────────┘
```

```toml
# ── telegraf.conf example ──

# Collect CPU metrics every 10 seconds
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false

# Collect memory metrics
[[inputs.mem]]

# Collect disk metrics
[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs"]

# Collect Docker container metrics
[[inputs.docker]]
  endpoint = "unix:///var/run/docker.sock"

# Write to InfluxDB
[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "my-token"
  organization = "myorg"
  bucket = "system_metrics"
```

---

## 6. Querying InfluxDB — InfluxQL & Flux

### InfluxQL — The SQL-Like Language (InfluxDB 1.x / 2.x)

```sql
-- ── Basic query ──
SELECT cpu_usage, memory_used 
FROM server_metrics 
WHERE host = 'web-01' 
  AND time > now() - 1h

-- ── Aggregation with time grouping (the bread and butter!) ──
SELECT MEAN(cpu_usage) AS avg_cpu,
       MAX(cpu_usage) AS max_cpu,
       MIN(cpu_usage) AS min_cpu
FROM server_metrics
WHERE host = 'web-01'
  AND time > now() - 24h
GROUP BY time(5m)              -- Average every 5 minutes!
                                -- 86400 seconds → 288 data points
                                -- Instead of 86400! (300x reduction)

-- ── Group by tags ──
SELECT MEAN(cpu_usage) 
FROM server_metrics
WHERE time > now() - 1h
GROUP BY host, region           -- Per-host, per-region averages

-- ── Fill missing data ──
SELECT MEAN(temperature) 
FROM sensor_data
WHERE time > now() - 6h
GROUP BY time(10m) fill(linear)  -- Interpolate gaps!
                                  -- Options: null, none, 0, linear, previous

-- ── Moving Average ──
SELECT MOVING_AVERAGE(MEAN(cpu_usage), 3) 
FROM server_metrics
WHERE time > now() - 1h
GROUP BY time(5m)

-- ── Top/Bottom N ──
SELECT TOP(cpu_usage, host, 5) FROM server_metrics  -- Top 5 hottest hosts
SELECT BOTTOM(cpu_usage, host, 5) FROM server_metrics -- Top 5 coldest hosts

-- ── Percentiles ──
SELECT PERCENTILE(response_time, 95) AS p95,
       PERCENTILE(response_time, 99) AS p99
FROM api_metrics
WHERE time > now() - 1h
GROUP BY time(5m)

-- ── Regular Expressions in WHERE ──
SELECT MEAN(cpu_usage) 
FROM server_metrics
WHERE host =~ /^web-/          -- All hosts starting with "web-"
  AND time > now() - 1h
GROUP BY host

-- ── Subqueries ──
SELECT MAX(mean_cpu)
FROM (
    SELECT MEAN(cpu_usage) AS mean_cpu
    FROM server_metrics
    WHERE time > now() - 24h
    GROUP BY time(1h), host
)
GROUP BY time(1h)              -- Hottest host each hour
```

### InfluxQL Aggregation Functions

```
┌────────────────────────────────────────────────────────────────┐
│            InfluxQL Functions Reference                        │
├──────────────┬─────────────────────────────────────────────────┤
│  Aggregation │  COUNT, DISTINCT, MEAN, MEDIAN, MODE, SUM,     │
│              │  SPREAD (max-min), STDDEV                      │
│              │                                                 │
│  Selection   │  FIRST, LAST, MAX, MIN, TOP, BOTTOM,          │
│              │  PERCENTILE, SAMPLE                            │
│              │                                                 │
│  Transform   │  ABS, ACOS, ASIN, ATAN, CEIL, COS, CUMSUM,   │
│              │  DERIVATIVE, DIFFERENCE, ELAPSED, EXP, FLOOR,  │
│              │  LN, LOG2, LOG10, MOVING_AVERAGE, NON_NEGATIVE │
│              │  _DERIVATIVE, NON_NEGATIVE_DIFFERENCE, POW,    │
│              │  ROUND, SIN, SQRT, TAN                         │
│              │                                                 │
│  Predictor   │  HOLT_WINTERS (time-series forecasting!)       │
└──────────────┴─────────────────────────────────────────────────┘
```

### Flux — The Functional Query Language (InfluxDB 2.x)

```javascript
// ── Basic Flux query ──
from(bucket: "mybucket")
  |> range(start: -1h)                              // Time range
  |> filter(fn: (r) => r._measurement == "server_metrics")
  |> filter(fn: (r) => r.host == "web-01")
  |> filter(fn: (r) => r._field == "cpu_usage")

// ── Aggregation with windowing ──
from(bucket: "mybucket")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "server_metrics")
  |> filter(fn: (r) => r._field == "cpu_usage")
  |> aggregateWindow(every: 5m, fn: mean)           // 5-minute averages
  |> yield(name: "avg_cpu")

// ── Multiple aggregations ──
data = from(bucket: "mybucket")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "server_metrics")
  |> filter(fn: (r) => r._field == "cpu_usage")

data |> aggregateWindow(every: 5m, fn: mean) |> yield(name: "mean")
data |> aggregateWindow(every: 5m, fn: max)  |> yield(name: "max")
data |> aggregateWindow(every: 5m, fn: min)  |> yield(name: "min")

// ── Percentile calculation ──
from(bucket: "mybucket")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "api_metrics")
  |> filter(fn: (r) => r._field == "response_time")
  |> aggregateWindow(every: 5m, fn: (tables=<-, column) =>
      tables |> quantile(q: 0.95, column: column)
  )

// ── Join two measurements ──
cpu = from(bucket: "mybucket")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu")
  |> filter(fn: (r) => r._field == "usage")

mem = from(bucket: "mybucket")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "memory")
  |> filter(fn: (r) => r._field == "used_percent")

join(tables: {cpu: cpu, mem: mem}, on: ["_time", "host"])

// ── Alert: CPU above 90% for 5 minutes ──
from(bucket: "mybucket")
  |> range(start: -10m)
  |> filter(fn: (r) => r._measurement == "server_metrics")
  |> filter(fn: (r) => r._field == "cpu_usage")
  |> aggregateWindow(every: 1m, fn: mean)
  |> filter(fn: (r) => r._value > 90.0)             // Only high-CPU points
  |> count()
  |> filter(fn: (r) => r._value >= 5)               // At least 5 minutes
  // → Trigger alert!

// ── Downsampling task (automatic!) ──
option task = {name: "downsample_cpu", every: 1h}

from(bucket: "raw_metrics")
  |> range(start: -task.every)
  |> filter(fn: (r) => r._measurement == "cpu")
  |> aggregateWindow(every: 5m, fn: mean)
  |> to(bucket: "downsampled_metrics")
```

### InfluxQL vs Flux — Quick Comparison

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│  Feature         │  InfluxQL            │  Flux                │
├──────────────────┼──────────────────────┼──────────────────────┤
│  Style           │  SQL-like            │  Functional/Pipeline │
│  Learning Curve  │  Easy (know SQL?)    │  Medium              │
│  Joins           │  Limited             │  Full support        │
│  Math Operations │  Basic               │  Rich                │
│  Alerting        │  External (Kapacitor)│  Built-in tasks      │
│  Downsampling    │  CQ (cont. queries)  │  Tasks               │
│  Cross-bucket    │  ❌                  │  ✅                  │
│  Variables       │  ❌                  │  ✅                  │
│  String Ops      │  Limited             │  Full                │
│  Recommended     │  Simple queries,     │  Complex analytics,  │
│                  │  InfluxDB 1.x compat │  InfluxDB 2.x native │
└──────────────────┴──────────────────────┴──────────────────────┘
```

---

## 7. Retention Policies — Auto-Delete Old Data

### The Problem

```
Day 1:   "We'll keep EVERYTHING! Disk is cheap!"
Day 30:  "Hmm, we've used 500GB..."
Day 90:  "2TB of metrics... storage costs are climbing..."
Day 365: "12TB. We're paying more for storage than our engineers."

Reality: You don't need per-second data from 6 months ago.
  • Last 24 hours → Raw data (every second)
  • Last 7 days → 1-minute averages
  • Last 30 days → 5-minute averages
  • Last 1 year → 1-hour averages
  • Older → Delete or archive
```

### InfluxDB Retention Policies

```bash
# ── InfluxDB 1.x: Retention Policies ──

# Create a retention policy
CREATE RETENTION POLICY "one_week" ON "mydb" DURATION 7d REPLICATION 1
CREATE RETENTION POLICY "one_month" ON "mydb" DURATION 30d REPLICATION 1
CREATE RETENTION POLICY "one_year" ON "mydb" DURATION 365d REPLICATION 1

# Set default retention policy
ALTER RETENTION POLICY "one_week" ON "mydb" DEFAULT

# Data older than the duration is AUTOMATICALLY DELETED!
# No manual cleanup. No cron jobs. No table locks.

# ── InfluxDB 2.x: Bucket = Database + Retention Policy ──
# When you create a bucket, you set the retention:
influx bucket create \
  --name raw_metrics \
  --retention 7d           # Auto-delete after 7 days

influx bucket create \
  --name downsampled_5m \
  --retention 30d          # Keep 30 days of 5-minute averages

influx bucket create \
  --name downsampled_1h \
  --retention 365d         # Keep 1 year of hourly averages
```

### Downsampling — Keep Summaries, Discard Details

```
┌──────────────────────────────────────────────────────────────┐
│              Downsampling Pipeline                            │
│                                                              │
│  Raw Data (1-sec)  ──┐                                      │
│  Retention: 7 days   │                                      │
│                      │  Aggregate every 5 min                │
│                      ▼                                      │
│  5-min Averages    ──┐                                      │
│  Retention: 30 days  │                                      │
│                      │  Aggregate every 1 hour               │
│                      ▼                                      │
│  1-hour Averages   ──┐                                      │
│  Retention: 1 year   │                                      │
│                      │  Aggregate every 1 day                │
│                      ▼                                      │
│  Daily Summaries                                            │
│  Retention: Forever                                         │
│                                                              │
│  Result: Recent data is detailed. Old data is summarized.   │
│          Storage stays manageable. Queries stay fast.        │
└──────────────────────────────────────────────────────────────┘
```

```javascript
// ── InfluxDB 2.x: Downsampling Task ──
// This runs automatically every hour!

option task = {
  name: "downsample_to_5m",
  every: 1h
}

from(bucket: "raw_metrics")
  |> range(start: -task.every)                    // Last hour of raw data
  |> filter(fn: (r) => r._measurement == "server_metrics")
  |> aggregateWindow(every: 5m, fn: mean)         // 5-minute averages
  |> to(bucket: "downsampled_5m",                 // Write to different bucket
       org: "myorg")
```

---

## 8. InfluxDB Storage Engine — TSM (Time-Structured Merge Tree)

```
┌─────────────────────────────────────────────────────────────────┐
│              TSM Storage Engine Architecture                    │
│                                                                 │
│  Write Path:                                                    │
│  ┌──────────┐                                                  │
│  │  Writes  │                                                  │
│  └────┬─────┘                                                  │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────┐    ┌──────────┐                                 │
│  │   WAL    │───►│  Cache   │   (In-memory sorted data)       │
│  │ (Write-  │    │ (sorted  │                                  │
│  │  Ahead   │    │  by time │                                  │
│  │  Log)    │    │  + key)  │                                  │
│  └──────────┘    └────┬─────┘                                  │
│                       │  When cache is full (or time trigger)  │
│                       ▼                                        │
│                  ┌──────────┐                                  │
│                  │ TSM File │   (Compressed, immutable,        │
│                  │ (on disk)│    sorted by time + series key)  │
│                  └──────────┘                                  │
│                       │                                        │
│                       ▼  Background compaction                 │
│              ┌──────────────────┐                              │
│              │  Compacted TSM   │   (Merged & optimized)       │
│              │  Files           │                               │
│              └──────────────────┘                              │
│                                                                 │
│  Read Path:                                                    │
│  1. Check Cache (in-memory) → newest data                     │
│  2. Check TSM Files (on disk) → use index to find time range  │
│  3. Merge results → return to client                          │
│                                                                 │
│  Key Optimizations:                                            │
│  • Time-sorted storage → sequential disk reads                │
│  • Compression: Gorilla (timestamps), Snappy/XOR (values)    │
│  • Series index → O(1) lookup for a specific series          │
│  • Compaction → merge small files into larger, optimized ones │
└─────────────────────────────────────────────────────────────────┘
```

### Compression Deep Dive

```
Timestamps (Gorilla compression):
──────────────────────────────────
  Raw:      [1718438400, 1718438401, 1718438402, 1718438403]
  Delta:    [base,       +1,         +1,         +1]
  Delta-of-Delta: [base, +1, 0, 0, 0]  ← mostly zeros!
  
  Regular intervals compress to nearly ZERO bytes per point!
  (Because Δ-of-Δ is 0 for constant-interval data)

Values (XOR + variable-length encoding):
─────────────────────────────────────────
  Raw:      [45.2, 45.5, 45.3, 45.8]
  XOR:      [45.2, XOR(45.2,45.5), XOR(45.5,45.3), XOR(45.3,45.8)]
  
  Similar float values differ in only a few bits.
  XOR encoding stores only the differing bits!
  
  Result: 8 bytes/value → often <1 byte/value after compression!
```

---

## 9. Real-World IoT Example — End to End

### Smart Building Monitoring System

```
Architecture:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Floor 1          Floor 2          Floor 3                     │
│  ┌─────┐          ┌─────┐          ┌─────┐                    │
│  │Temp │          │Temp │          │Temp │                     │
│  │Humid│          │Humid│          │Humid│    200 sensors      │
│  │CO2  │          │CO2  │          │CO2  │    reporting        │
│  │Light│          │Light│          │Light│    every 10 sec     │
│  └──┬──┘          └──┬──┘          └──┬──┘                     │
│     │                │                │                        │
│     └────────┬───────┘────────┬───────┘                        │
│              │                │                                │
│              ▼                ▼                                │
│         ┌──────────┐    ┌──────────┐                          │
│         │ Telegraf  │    │ MQTT     │                          │
│         │ Agent     │◄───│ Broker   │                          │
│         └────┬─────┘    └──────────┘                          │
│              │                                                 │
│              ▼                                                 │
│         ┌──────────┐                                          │
│         │ InfluxDB │   Raw: 7 days                            │
│         │          │   5-min avg: 90 days                      │
│         │          │   1-hr avg: 2 years                       │
│         └────┬─────┘                                          │
│              │                                                 │
│              ▼                                                 │
│         ┌──────────┐                                          │
│         │ Grafana  │   Real-time dashboards                   │
│         │          │   Alert on anomalies                     │
│         └──────────┘                                          │
└────────────────────────────────────────────────────────────────┘
```

```
Data Volume Calculation:
  200 sensors × 4 metrics × 6 points/min × 60 min × 24 hr
  = 6,912,000 data points per day
  = ~200 million points per month
  
  With time-series compression: ~2 GB/month (instead of ~50 GB!)
```

```python
# ── Writing sensor data ──
from influxdb_client import InfluxDBClient, Point
from datetime import datetime

client = InfluxDBClient(url="http://localhost:8086", token="my-token", org="myorg")
write_api = client.write_api()

# Simulate sensor reading
point = Point("building_environment") \
    .tag("building", "HQ") \
    .tag("floor", "3") \
    .tag("room", "3A-Conference") \
    .tag("sensor_id", "SENS-301") \
    .field("temperature", 23.5) \
    .field("humidity", 45.2) \
    .field("co2_ppm", 420) \
    .field("light_lux", 350) \
    .time(datetime.utcnow())

write_api.write(bucket="building_metrics", record=point)
```

```javascript
// ── Querying: "Show average temperature per floor, last 6 hours" ──
// Flux query:
from(bucket: "building_metrics")
  |> range(start: -6h)
  |> filter(fn: (r) => r._measurement == "building_environment")
  |> filter(fn: (r) => r._field == "temperature")
  |> aggregateWindow(every: 15m, fn: mean)
  |> group(columns: ["floor"])

// ── Alert: "CO2 above 1000 ppm for 10 minutes" ──
from(bucket: "building_metrics")
  |> range(start: -15m)
  |> filter(fn: (r) => r._field == "co2_ppm")
  |> aggregateWindow(every: 1m, fn: mean)
  |> filter(fn: (r) => r._value > 1000)
  |> count()
  |> filter(fn: (r) => r._value >= 10)
  // → Open windows! Turn on ventilation!
```

---

## 10. Time-Series Database Landscape — Comparison

```
┌──────────────────────────────────────────────────────────────────────┐
│              Time-Series Database Comparison                         │
├──────────────┬──────────┬──────────┬──────────┬──────────┬──────────┤
│  Feature     │ InfluxDB │Timescale │Prometheus│ QuestDB  │ClickHouse│
│              │          │   DB     │          │          │          │
├──────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│  Based On    │ Custom   │PostgreSQL│ Custom   │ Custom   │ Custom   │
│  Query Lang  │ Flux/SQL │ SQL!!    │ PromQL   │ SQL      │ SQL      │
│  Best For    │ IoT,     │ Already  │ Infra    │ High-    │ Analytics│
│              │ General  │ use PG   │ Monitor  │ ingest   │ OLAP     │
│  Write Speed │ ~1M/s    │ ~500K/s  │ ~1M/s   │ ~4M/s   │ ~2M/s   │
│  Compression │ 10-50x   │ ~10x    │ ~12x    │ ~20x    │ ~10-40x │
│  SQL Support │ Limited  │ Full!   │ ❌ (PQL) │ Full    │ Full     │
│  JOINs       │ Flux only│ Full    │ Limited │ Full    │ Full     │
│  Managed     │ Cloud    │ Cloud   │ Grafana │ Cloud   │ Cloud    │
│  License     │ MIT/Comm │ Apache  │ Apache  │ Apache  │ Apache   │
│  Language    │ Go       │ C (PG)  │ Go      │ Java/C++│ C++      │
├──────────────┴──────────┴──────────┴──────────┴──────────┴──────────┤
│                                                                      │
│  Decision Guide:                                                     │
│  • Already using PostgreSQL? → TimescaleDB (extension, not new DB!) │
│  • Kubernetes monitoring? → Prometheus + Grafana (industry standard) │
│  • IoT / general time-series? → InfluxDB (most popular, rich API)   │
│  • Need maximum write speed? → QuestDB (built for throughput)       │
│  • Heavy analytics on time-series? → ClickHouse (columnar beast)   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### TimescaleDB — Special Mention

```
┌─────────────────────────────────────────────────────────────────┐
│              TimescaleDB — Time-Series ON PostgreSQL             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TimescaleDB is NOT a separate database.                       │
│  It's a PostgreSQL EXTENSION that adds time-series superpowers. │
│                                                                 │
│  What you keep:                    What you gain:               │
│  • Full SQL (JOINs, CTEs, etc.)    • Hypertables (auto-partition│
│  • All PG extensions (PostGIS!)      by time)                  │
│  • Existing tools (pg_dump, etc.)  • 10-20x compression        │
│  • ACID transactions               • Continuous aggregates     │
│  • All your PostgreSQL knowledge   • Data retention policies   │
│                                    • Columnar compression      │
│                                    • Time-series functions     │
│                                                                 │
│  Perfect for: Teams already using PostgreSQL who need          │
│  time-series capabilities without learning a new database.     │
│                                                                 │
│  CREATE EXTENSION IF NOT EXISTS timescaledb;                   │
│  SELECT create_hypertable('metrics', 'time');  -- That's it!   │
└─────────────────────────────────────────────────────────────────┘
```

### Prometheus — The Kubernetes Standard

```
┌─────────────────────────────────────────────────────────────────┐
│              Prometheus — Infrastructure Monitoring             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Prometheus is THE monitoring system for Kubernetes and         │
│  cloud-native infrastructure.                                  │
│                                                                 │
│  Key Differences from InfluxDB:                                │
│  • PULL model (Prometheus scrapes targets, not push)           │
│  • PromQL query language (not SQL or Flux)                     │
│  • Built for infrastructure monitoring, not general TSDB       │
│  • Native Kubernetes service discovery                         │
│  • Alert Manager for notification routing                      │
│  • Grafana integration (industry standard combo)               │
│                                                                 │
│  When to use Prometheus:                                       │
│  • Kubernetes cluster monitoring                               │
│  • Microservices metrics (request rates, error rates)          │
│  • Infrastructure alerting                                     │
│                                                                 │
│  When to use InfluxDB instead:                                 │
│  • IoT sensor data (push model, higher write rates)           │
│  • Custom application metrics (richer data model)              │
│  • Long-term storage with downsampling                        │
│  • SQL-like querying                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. Best Practices & Common Mistakes

### ✅ Do This

```
1. DESIGN TAGS CAREFULLY: Low cardinality only (host, region, env).
   Never use user_id, email, or IP address as a tag.

2. USE RETENTION POLICIES: Don't keep per-second data forever.
   Raw → 7 days, 5-min avg → 90 days, 1-hr avg → forever.

3. BATCH WRITES: Send 1000-5000 points per API call, not one at a time.
   InfluxDB is optimized for batch ingestion.

4. USE TELEGRAF: Don't reinvent the wheel. Telegraf has 300+ input 
   plugins for every data source imaginable.

5. TIMESTAMP IN DATA: Always include timestamps from the source.
   Server-time can be inaccurate for distributed systems.

6. SCHEMA ON WRITE: Plan your measurement names, tags, and fields
   before you start writing. Changing schema later is painful.

7. USE GRAFANA: InfluxDB + Grafana = the gold standard for
   time-series visualization. Both are free and open source.

8. MONITOR CARDINALITY: Check series cardinality regularly.
   influx inspect report-tsm --print-tsm  (InfluxDB 1.x)
   
9. SEPARATE HIGH/LOW-FREQUENCY DATA: IoT sensors (every second) and
   daily business metrics shouldn't share the same bucket.

10. TEST QUERIES ON SMALL TIME RANGES FIRST: Always start with 
    |> range(start: -5m) before querying a year of data.
```

### ❌ Don't Do This

```
1. DON'T use high-cardinality tags (user IDs, session IDs, trace IDs).
   This creates millions of series → memory explosion → crash!

2. DON'T store non-time-series data in InfluxDB (user profiles, config).
   Use PostgreSQL or MongoDB for that.

3. DON'T write one point at a time (high overhead per HTTP call).
   Always batch writes.

4. DON'T use fields for values you'll filter on frequently.
   Fields aren't indexed → full scan every time.

5. DON'T forget downsampling. Raw per-second data from 2 years ago
   is almost never needed. Save 95% storage with hourly averages.

6. DON'T use InfluxDB for complex relational queries.
   JOINs, transactions, and relational integrity are not its strength.

7. DON'T put the metric name in the field name:
   ❌ field: "cpu_web01_usage" (metrics × hosts = explosion)
   ✅ tag: host=web-01, field: cpu_usage

8. DON'T query without a time range. InfluxDB is optimized for
   time-bounded queries. Open-ended queries scan everything.

9. DON'T ignore disk I/O monitoring. TSM compaction needs fast disks.
   SSDs are recommended for production.

10. DON'T treat InfluxDB like a general-purpose database.
    It's a specialized tool — use it for what it's built for.
```

---

## 12. The TICK Stack / InfluxDB Platform

```
┌─────────────────────────────────────────────────────────────────┐
│          The TICK Stack (InfluxDB 1.x) / InfluxDB Platform 2.x │
│                                                                 │
│  InfluxDB 1.x — The TICK Stack:                                │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│  │Telegraf  │──►│InfluxDB  │──►│Chronograf│   │Kapacitor │   │
│  │(Collect) │   │(Store)   │   │(Visualize│   │(Alert    │   │
│  │          │   │          │◄──│ + Query) │   │ + Process│   │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   │
│                                                                 │
│  InfluxDB 2.x — Unified Platform:                              │
│                                                                 │
│  ┌──────────┐   ┌─────────────────────────────────────────┐   │
│  │Telegraf  │──►│         InfluxDB 2.x                     │   │
│  │(Collect) │   │  ┌─────────┬──────────┬──────────────┐  │   │
│  │          │   │  │ Store   │ Query    │ Alert/Tasks  │  │   │
│  └──────────┘   │  │ (TSM)   │ (Flux)   │ (Built-in!)  │  │   │
│                 │  ├─────────┼──────────┼──────────────┤  │   │
│  ┌──────────┐   │  │ Dashboards (Built-in Chronograf!) │  │   │
│  │ Grafana  │◄──│  │ API (Token-based auth)             │  │   │
│  │(optional)│   │  │ CLI (influx command)               │  │   │
│  └──────────┘   │  └──────────────────────────────────────┘  │   │
│                 └─────────────────────────────────────────────┘   │
│                                                                 │
│  2.x combined everything into ONE binary! No more managing     │
│  4 separate services.                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13. InfluxDB Quick-Start Setup

```bash
# ── Install InfluxDB 2.x (Ubuntu/Debian) ──
wget https://dl.influxdata.com/influxdb/releases/influxdb2-2.7.5-amd64.deb
sudo dpkg -i influxdb2-2.7.5-amd64.deb
sudo systemctl start influxdb

# ── Or Docker (easiest!) ──
docker run -d \
  --name influxdb \
  -p 8086:8086 \
  -v influxdb-data:/var/lib/influxdb2 \
  influxdb:2.7

# ── Initial setup (web UI) ──
# Open: http://localhost:8086
# Set: username, password, organization, initial bucket

# ── Or CLI setup ──
influx setup \
  --username admin \
  --password my-secure-password \
  --org myorg \
  --bucket mybucket \
  --retention 7d \
  --force

# ── Install Telegraf ──
sudo apt install telegraf
# Edit: /etc/telegraf/telegraf.conf
sudo systemctl start telegraf

# ── Verify ──
influx ping
# OK

influx bucket list
# ID                  Name            Retention
# 0a1b2c3d4e5f6g7h    mybucket        168h0m0s (7 days)
```

---

## 🧠 Chapter Summary — Key Takeaways

```
┌─────────────────────────────────────────────────────────────────┐
│     Time-Series Databases & InfluxDB — What You Must Remember  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. TIME-SERIES DATA IS EVERYWHERE: IoT, monitoring, finance,  │
│     analytics. If it has a timestamp, it's time-series.        │
│                                                                 │
│  2. TRADITIONAL DBs FAIL HERE: PostgreSQL works but is 10-100x│
│     less efficient than purpose-built TSDBs for write-heavy,   │
│     time-bounded workloads.                                    │
│                                                                 │
│  3. TAGS vs FIELDS: Tags are indexed (filter/group), fields    │
│     are not (store values). High-cardinality tags = disaster.  │
│                                                                 │
│  4. RETENTION + DOWNSAMPLING: Keep raw data for days, summaries│
│     for months/years. This is THE lifecycle management pattern.│
│                                                                 │
│  5. COMPRESSION IS MAGIC: Delta-of-delta for timestamps,      │
│     XOR for values → 10-50x compression for free.             │
│                                                                 │
│  6. INFLUXDB = #1 TSDB: Best ecosystem (Telegraf, Flux,       │
│     Grafana), easiest to start, rich query language.           │
│                                                                 │
│  7. KNOW THE ALTERNATIVES:                                     │
│     • Already on PostgreSQL? → TimescaleDB (extension)        │
│     • Kubernetes? → Prometheus + Grafana                      │
│     • Max write speed? → QuestDB                              │
│     • Heavy analytics? → ClickHouse                           │
│                                                                 │
│  8. NEVER: high-cardinality tags, unbounded queries without    │
│     time ranges, single-point writes, or treating TSDB as a   │
│     general-purpose database.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|-------------|-------|
| [3G.4 — HBase — Hadoop's Database](./04-HBase.md) | Wide-column store built on HDFS for big data |
| [3G.5 — Vector Databases](./05-Vector-Databases.md) | The AI/ML era — embeddings, similarity search, RAG |
| [3G.6 — Firebase](./06-Firebase.md) | Real-time sync for mobile & web apps |

---

> 💡 **Final Thought**: Time-series data is the **fastest-growing data category on the planet**. As IoT devices multiply, AI models need more telemetry, and companies demand real-time observability — time-series databases aren't optional anymore. They're foundational. Master InfluxDB, understand the alternatives, and you'll be the engineer who handles the data everyone else struggles with.

---

*Chapter 3G.3 — Complete. You now think in timestamps, tags, and retention policies. The time-series world is yours.* ⏱️
