# Time-Series Databases (InfluxDB, TimescaleDB)

> **What you'll learn**: How time-series databases are optimized for timestamped data, why regular databases struggle with metrics and IoT data, how they achieve massive write throughput with efficient compression, and when to use InfluxDB vs TimescaleDB.

---

## Real-Life Analogy

Imagine you have a **fitness tracker** that records your heart rate every second. In one day, that's 86,400 readings. In a year, 31.5 million readings. Now imagine a hospital monitoring 1,000 patients simultaneously — 31.5 BILLION readings per year, for ONE hospital.

A regular database would choke on this because:
- You're **always appending** new data (never updating old readings)
- You **always query by time ranges** ("show me last 24 hours")
- Old data becomes **less valuable** (you care about recent trends)
- You need **aggregations** ("average heart rate per hour")

A **time-series database** is specifically built for this pattern — like a specialized tape recorder that writes continuously, compresses old recordings, and can instantly fast-forward/rewind to any time range.

---

## Core Concept Explained Step-by-Step

### What is Time-Series Data?

**Time-series data** = measurements collected over time, where each point has:
- **Timestamp** (when)
- **Value(s)** (what was measured)
- **Tags/Labels** (metadata: which server, which sensor, which region)

```
TIME                    │ METRIC        │ HOST        │ VALUE
────────────────────────┼───────────────┼─────────────┼──────────
2024-01-15 10:00:00     │ cpu_usage     │ server-1    │ 45.2%
2024-01-15 10:00:00     │ cpu_usage     │ server-2    │ 67.8%
2024-01-15 10:00:01     │ cpu_usage     │ server-1    │ 46.1%
2024-01-15 10:00:01     │ cpu_usage     │ server-2    │ 68.3%
2024-01-15 10:00:02     │ cpu_usage     │ server-1    │ 44.9%
...
(Millions of rows per day, per metric, per host)
```

### Why Regular Databases Fail at Time-Series

```
REGULAR DATABASE (PostgreSQL):            TIME-SERIES DATABASE (InfluxDB):
┌─────────────────────────────┐          ┌─────────────────────────────────┐
│ Row-oriented storage         │          │ Column-oriented, time-sorted    │
│ B-Tree index (random I/O)   │          │ LSM-Tree (sequential writes)    │
│ No built-in downsampling     │          │ Automatic downsampling          │
│ No time-based retention      │          │ Built-in retention policies     │
│ No compression for sequences │          │ Delta/Gorilla compression       │
│ General-purpose optimizer    │          │ Optimized for time-range scans  │
│                              │          │                                 │
│ 50K writes/sec (struggle)    │          │ 1M+ writes/sec (easy)           │
│ 100GB → 100GB               │          │ 100GB → ~10GB (compression)    │
└─────────────────────────────┘          └─────────────────────────────────┘
```

### The Time-Series Data Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TIME-SERIES DATA MODEL                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  MEASUREMENT (like a table):  "cpu_usage"                           │
│                                                                       │
│  TAGS (indexed metadata, low cardinality):                          │
│    host = "server-1"                                                 │
│    region = "us-east"                                                │
│    env = "production"                                                │
│                                                                       │
│  FIELDS (actual values, not indexed):                                │
│    usage_percent = 45.2                                              │
│    cores_active = 6                                                  │
│                                                                       │
│  TIMESTAMP:  2024-01-15T10:00:00.000Z                              │
│                                                                       │
│  Series = Unique combination of measurement + tags                   │
│  "cpu_usage,host=server-1,region=us-east"  ← ONE time series        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Storage Architecture (InfluxDB / TSM Engine)

```
WRITE PATH:
┌────────────┐     ┌──────────────┐     ┌─────────────────────┐
│ Incoming   │────▶│   WAL        │────▶│    In-Memory Cache   │
│ Data Point │     │(Write-Ahead  │     │  (Sorted by time)    │
│            │     │ Log - Disk)  │     │                      │
└────────────┘     └──────────────┘     └──────────┬──────────┘
                                                    │ Flush when full
                                                    ▼
                                        ┌─────────────────────┐
                                        │  TSM File (on disk)  │
                                        │  Compressed, sorted  │
                                        │  by series + time    │
                                        └──────────┬──────────┘
                                                    │ Background
                                                    ▼
                                        ┌─────────────────────┐
                                        │   Compaction         │
                                        │  (Merge small files  │
                                        │   into larger ones)  │
                                        └─────────────────────┘

READ PATH:
Query: "SELECT mean(cpu) WHERE host='srv-1' AND time > now()-1h GROUP BY time(5m)"
         │
         ├──▶ Check in-memory cache (most recent data)
         ├──▶ Check TSM files (on disk, compressed)
         └──▶ Merge results, apply aggregation
```

### Time-Series Compression (Gorilla Encoding)

Real CPU data: `45.2, 45.3, 45.1, 45.4, 45.2, 45.5`

```
NAIVE STORAGE:                    GORILLA COMPRESSION:
Each value: 8 bytes (float64)     First value: 8 bytes (full)
6 values = 48 bytes               Subsequent: XOR with previous
                                  45.2 XOR 45.3 = very few bits differ!
                                  Store only the XOR difference
                                  6 values ≈ 12 bytes (75% savings!)

TIMESTAMPS:
Naive: 8 bytes each               Delta-of-deltas encoding:
                                  Timestamps are usually evenly spaced:
                                  10:00, 10:01, 10:02, 10:03...
                                  Deltas: 60, 60, 60, 60 (all same!)
                                  Delta-of-deltas: 0, 0, 0, 0
                                  Store: 1 bit per timestamp!

Result: 10-20x compression on typical metrics data
```

### TimescaleDB Architecture (PostgreSQL Extension)

```
┌─────────────────────────────────────────────────────────────────┐
│              TimescaleDB (on top of PostgreSQL)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   HYPERTABLE                               │  │
│  │  (Looks like one big table, actually many chunks)         │  │
│  │                                                           │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │  │
│  │  │ Chunk 1  │ │ Chunk 2  │ │ Chunk 3  │ │ Chunk 4  │   │  │
│  │  │ Jan 1-7  │ │ Jan 8-14 │ │Jan 15-21 │ │Jan 22-28 │   │  │
│  │  │ (cold)   │ │ (cold)   │ │ (warm)   │ │ (hot)    │   │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │  │
│  │                                                           │  │
│  │  Each chunk = Regular PostgreSQL table                    │  │
│  │  Auto-partitioned by time (default: 7 days)              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Benefits:                                                       │
│  • Full SQL support (JOINs, subqueries, window functions)       │
│  • PostgreSQL ecosystem (pg_dump, replication, extensions)       │
│  • Chunk-level operations (compress old chunks, drop old data)   │
│  • Continuous aggregations (auto-maintained materialized views)  │
└─────────────────────────────────────────────────────────────────┘
```

### Retention Policies & Downsampling

```
TIME ──────────────────────────────────────────────────────▶

│ Last 24h        │ Last 7 days      │ Last 90 days     │ Older    │
│ RAW DATA        │ 1-MIN AVERAGES   │ 1-HOUR AVERAGES  │ DELETED  │
│ (every second)  │ (60x less data)  │ (3600x less)     │          │
│ Fast SSD        │ SSD              │ HDD / Cold       │          │

Automatic downsampling:
- Keep raw data for 24 hours (for debugging)
- After 24h: aggregate to 1-minute intervals
- After 7 days: aggregate to 1-hour intervals
- After 90 days: delete (or archive to S3)
```

---

## Code Examples

### Python (InfluxDB)

```python
from influxdb_client import InfluxDBClient, Point, WritePrecision
from influxdb_client.client.write_api import SYNCHRONOUS
from datetime import datetime, timedelta
import random

# Connect to InfluxDB
client = InfluxDBClient(
    url="http://localhost:8086",
    token="my-token",
    org="my-org"
)
write_api = client.write_api(write_options=SYNCHRONOUS)
query_api = client.query_api()

# Write metrics (simulating server monitoring)
for i in range(100):
    point = Point("server_metrics") \
        .tag("host", "web-server-1") \
        .tag("region", "ap-south-1") \
        .tag("env", "production") \
        .field("cpu_percent", random.uniform(30, 80)) \
        .field("memory_percent", random.uniform(50, 90)) \
        .field("disk_io_mbps", random.uniform(10, 200)) \
        .time(datetime.utcnow() - timedelta(seconds=100-i))
    
    write_api.write(bucket="monitoring", record=point)

# Query: Average CPU per 5-minute window over last hour
query = '''
    from(bucket: "monitoring")
        |> range(start: -1h)
        |> filter(fn: (r) => r._measurement == "server_metrics")
        |> filter(fn: (r) => r._field == "cpu_percent")
        |> filter(fn: (r) => r.host == "web-server-1")
        |> aggregateWindow(every: 5m, fn: mean, createEmpty: false)
        |> yield(name: "mean_cpu")
'''
tables = query_api.query(query)
for table in tables:
    for record in table.records:
        print(f"{record.get_time()}: CPU = {record.get_value():.1f}%")

# Alert query: Find when CPU > 90% for more than 5 minutes
alert_query = '''
    from(bucket: "monitoring")
        |> range(start: -1h)
        |> filter(fn: (r) => r._field == "cpu_percent")
        |> aggregateWindow(every: 5m, fn: mean)
        |> filter(fn: (r) => r._value > 90.0)
'''
```

### Python (TimescaleDB — using standard SQL!)

```python
import psycopg2
from datetime import datetime, timedelta

conn = psycopg2.connect("host=localhost dbname=metrics user=admin password=secret")
cursor = conn.cursor()

# Create hypertable (TimescaleDB magic!)
cursor.execute("""
    CREATE TABLE IF NOT EXISTS server_metrics (
        time        TIMESTAMPTZ NOT NULL,
        host        TEXT NOT NULL,
        cpu         DOUBLE PRECISION,
        memory      DOUBLE PRECISION,
        disk_io     DOUBLE PRECISION
    );
    -- Convert to hypertable (auto-partitions by time!)
    SELECT create_hypertable('server_metrics', 'time', if_not_exists => TRUE);
""")

# Insert data (same as regular PostgreSQL!)
cursor.execute("""
    INSERT INTO server_metrics (time, host, cpu, memory, disk_io)
    VALUES (NOW(), 'web-1', 45.2, 67.8, 120.5)
""")

# Time-bucket query (TimescaleDB function)
cursor.execute("""
    SELECT 
        time_bucket('5 minutes', time) AS bucket,
        host,
        AVG(cpu) AS avg_cpu,
        MAX(cpu) AS max_cpu,
        AVG(memory) AS avg_memory
    FROM server_metrics
    WHERE time > NOW() - INTERVAL '1 hour'
    GROUP BY bucket, host
    ORDER BY bucket DESC
""")

for row in cursor.fetchall():
    print(f"{row[0]} | {row[1]} | CPU: {row[2]:.1f}% (max: {row[3]:.1f}%) | MEM: {row[4]:.1f}%")

# Continuous aggregation (auto-maintained materialized view)
cursor.execute("""
    CREATE MATERIALIZED VIEW IF NOT EXISTS hourly_metrics
    WITH (timescaledb.continuous) AS
    SELECT 
        time_bucket('1 hour', time) AS hour,
        host,
        AVG(cpu) AS avg_cpu,
        percentile_cont(0.95) WITHIN GROUP (ORDER BY cpu) AS p95_cpu
    FROM server_metrics
    GROUP BY hour, host;
""")

# Retention policy: auto-delete data older than 30 days
cursor.execute("""
    SELECT add_retention_policy('server_metrics', INTERVAL '30 days');
""")

conn.commit()
```

### Java (InfluxDB)

```java
import com.influxdb.client.InfluxDBClient;
import com.influxdb.client.InfluxDBClientFactory;
import com.influxdb.client.WriteApiBlocking;
import com.influxdb.client.domain.WritePrecision;
import com.influxdb.client.write.Point;
import java.time.Instant;

public class TimeSeriesExample {
    public static void main(String[] args) {
        InfluxDBClient client = InfluxDBClientFactory.create(
            "http://localhost:8086", "my-token".toCharArray(), "my-org", "monitoring");

        WriteApiBlocking writeApi = client.getWriteApiBlocking();

        // Write server metrics
        Point point = Point.measurement("server_metrics")
            .addTag("host", "web-server-1")
            .addTag("region", "ap-south-1")
            .addField("cpu_percent", 67.5)
            .addField("memory_percent", 82.3)
            .addField("request_count", 1523)
            .time(Instant.now(), WritePrecision.MS);

        writeApi.writePoint(point);

        // Query with Flux language
        String query = """
            from(bucket: "monitoring")
                |> range(start: -1h)
                |> filter(fn: (r) => r._measurement == "server_metrics")
                |> filter(fn: (r) => r.host == "web-server-1")
                |> aggregateWindow(every: 5m, fn: mean)
        """;

        client.getQueryApi().query(query).forEach(table -> {
            table.getRecords().forEach(record -> {
                System.out.printf("%s: %s = %.2f%n",
                    record.getTime(), record.getField(), (Double) record.getValue());
            });
        });

        client.close();
    }
}
```

---

## Infrastructure Examples

### InfluxDB + Grafana Stack (Docker Compose)

```yaml
version: '3.8'
services:
  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=secretpassword
      - DOCKER_INFLUXDB_INIT_ORG=myorg
      - DOCKER_INFLUXDB_INIT_BUCKET=monitoring
    volumes:
      - influxdb-data:/var/lib/influxdb2

  telegraf:
    image: telegraf:1.29
    volumes:
      - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - influxdb

  grafana:
    image: grafana/grafana:10.2
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - influxdb

volumes:
  influxdb-data:
  grafana-data:
```

### TimescaleDB Production Setup

```yaml
version: '3.8'
services:
  timescaledb:
    image: timescale/timescaledb:latest-pg16
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=metrics
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=secret
    volumes:
      - timescale-data:/var/lib/postgresql/data
    command: >
      postgres 
        -c shared_buffers=4GB
        -c effective_cache_size=12GB
        -c timescaledb.max_background_workers=8
    deploy:
      resources:
        limits:
          memory: 16G

volumes:
  timescale-data:
```

---

## InfluxDB vs TimescaleDB

```
┌───────────────────────────────────────────────────────────────────────┐
│                 InfluxDB vs TimescaleDB                                │
├──────────────────────┬────────────────────────┬───────────────────────┤
│ Feature              │ InfluxDB               │ TimescaleDB            │
├──────────────────────┼────────────────────────┼───────────────────────┤
│ Query Language       │ Flux (custom)          │ Full SQL               │
│ Based On             │ Custom engine          │ PostgreSQL extension   │
│ JOINs               │ Limited                │ Full SQL JOINs         │
│ Ecosystem            │ Standalone             │ PostgreSQL ecosystem   │
│ Write Performance    │ Excellent              │ Very Good              │
│ Compression          │ 10-20x                 │ 5-10x                 │
│ Learning Curve       │ New language (Flux)    │ Standard SQL           │
│ Best For             │ Pure metrics/IoT       │ Metrics + relational   │
│ Managed Service      │ InfluxDB Cloud         │ Timescale Cloud        │
└──────────────────────┴────────────────────────┴───────────────────────┘
```

---

## Real-World Example

### Uber — Time-Series for Real-Time Monitoring

```
┌─────────────────────────────────────────────────────────────────┐
│            Uber's Metrics Platform (M3)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Scale:                                                          │
│  • 500 million+ unique time series                               │
│  • 30 billion+ data points ingested per day                      │
│  • Sub-second query latency on any time range                    │
│                                                                   │
│  Architecture:                                                   │
│  ┌──────────┐    ┌──────────┐    ┌───────────────────────┐     │
│  │ Services │───▶│ M3       │───▶│ M3DB (Custom TSDB)   │     │
│  │ (emit    │    │Aggregator│    │ • Distributed         │     │
│  │ metrics) │    │          │    │ • Write-optimized     │     │
│  └──────────┘    └──────────┘    │ • Compressed storage  │     │
│                                   │ • Retention tiers     │     │
│                                   └───────────┬───────────┘     │
│                                               │                  │
│                                               ▼                  │
│                                   ┌───────────────────────┐     │
│                                   │ Grafana (Dashboards)  │     │
│                                   │ + Alerting            │     │
│                                   └───────────────────────┘     │
│                                                                   │
│  Metrics tracked:                                                │
│  • Trip request latency (p50, p95, p99)                         │
│  • Driver location update frequency                              │
│  • Service error rates per endpoint                              │
│  • Database query latencies                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| High-cardinality tags | Creates millions of time series (memory explosion) | Don't use user_id or request_id as tags |
| Not setting retention | Disk fills up forever | Set retention policies from day one |
| Storing events, not metrics | Time-series DBs are for aggregated metrics | Use Kafka/Elasticsearch for events |
| Querying raw data at large ranges | Scanning billions of points is slow | Use downsampled/continuous aggregations |
| Not pre-aggregating | Every dashboard query hits raw data | Create materialized views for common queries |
| Wrong time precision | Nanosecond precision when seconds suffice | Use appropriate precision (saves storage) |

---

## When to Use / When NOT to Use

### ✅ Use Time-Series Databases When:
- Data is **timestamped** and **append-only** (metrics, logs, sensor data)
- Queries are **always by time range** ("last 1 hour", "last 7 days")
- You need **automatic downsampling** and retention
- **High write throughput** is required (millions of points/second)
- Data naturally **compresses well** (sequential values)
- Built-in **aggregation functions** are needed (moving averages, percentiles)

### ❌ Do NOT Use Time-Series Databases When:
- Data requires **frequent updates** (TSDB is append-only)
- You need **complex relationships** between entities (use Graph/SQL)
- **Full-text search** is required (use Elasticsearch)
- Data is **not time-based** (use regular databases)
- You need **ACID transactions** (use PostgreSQL)
- Dataset is small (regular PostgreSQL with indexes handles it fine)

---

## Key Takeaways

1. **Time-series databases** are purpose-built for timestamped, append-only data with time-range queries — achieving 10-100x better performance than general databases for these workloads.
2. **Compression** (Gorilla, delta-of-deltas) achieves 10-20x storage reduction because time-series data has natural patterns.
3. **Retention policies** automatically age out old data — keep raw data short-term, aggregates long-term.
4. **TimescaleDB** = PostgreSQL with time-series superpowers (full SQL, JOINs, ecosystem). Best when you need time-series + relational.
5. **InfluxDB** = Purpose-built TSDB with best compression and write performance. Best for pure metrics/IoT workloads.
6. **High-cardinality tags** (like user_id) are the #1 performance killer — design tag schemas carefully.
7. **Downsampling and continuous aggregations** are essential for querying large time ranges efficiently.

---

## What's Next?

Next, we'll explore **Search Engines as Databases (Elasticsearch, OpenSearch)** (Chapter 9.8), powerful systems built for full-text search, log analytics, and finding needles in haystacks of unstructured data.
