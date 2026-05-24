# Real-Time Analytics Architecture

> **What you'll learn**: How to build end-to-end systems that ingest millions of events per second, process them in real-time, and serve live dashboards and alerts with sub-second latency — the way Netflix, Uber, and LinkedIn do it at planet-scale.

---

## Real-Life Analogy

Imagine you're the **director of a live TV news broadcast** during election night.

### Batch Analytics (The Morning Newspaper)

A newspaper publishes results the *next day*. They wait for all votes to be counted, write articles, print millions of copies, and deliver them to doorsteps at 6 AM. The data is accurate but **12 hours old**.

### Real-Time Analytics (Live TV)

You're broadcasting LIVE. Every minute:
- New vote counts stream in from 10,000 polling stations
- Your graphics team updates the on-screen tally **within seconds**
- Analysts announce "State X just flipped!" the moment it happens
- Viewers see real-time progress bars filling up

You can't wait for all data to arrive. You can't batch-process at midnight. You must:
1. **Ingest** data the instant it arrives
2. **Process** it (aggregate, filter, enrich) in milliseconds
3. **Serve** updated results to millions of viewers simultaneously

**That's real-time analytics** — seeing what's happening NOW, not what happened yesterday.

---

## Core Concept Explained Step-by-Step

### Step 1: What Makes Analytics "Real-Time"?

```
┌─────────────────────────────────────────────────────────────────┐
│              ANALYTICS LATENCY SPECTRUM                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ◀─── BATCH ──────────── NEAR REAL-TIME ─────── REAL-TIME ───▶  │
│                                                                   │
│  Hours/Days           Minutes              Seconds/Milliseconds   │
│                                                                   │
│  "Yesterday's         "What happened        "What's happening     │
│   revenue was $5M"    in the last 5 min?"   RIGHT NOW?"           │
│                                                                   │
│  Technology:          Technology:           Technology:            │
│  • Airflow + BQ       • Micro-batch Spark   • Kafka + Flink       │
│  • Daily ETL          • Kafka + 5min windows• Druid / ClickHouse  │
│  • Traditional DW     • Near-RT dashboards  • Custom streaming    │
│                                                                   │
│  Use Case:            Use Case:            Use Case:              │
│  • Monthly reports    • Live dashboards    • Fraud detection       │
│  • ML training        • Anomaly detection  • Dynamic pricing      │
│  • Compliance         • Alerting           • Gaming leaderboards  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: The Three Layers of Real-Time Analytics

Every real-time analytics system has three distinct layers:

```
┌─────────────────────────────────────────────────────────────────────────┐
│              REAL-TIME ANALYTICS: THREE LAYERS                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Layer 1: INGESTION                                                      │
│  ─────────────────                                                      │
│  "Collect events at massive scale, buffer them reliably"                 │
│  Tools: Apache Kafka, Amazon Kinesis, Google Pub/Sub                     │
│  Scale: Millions of events/second                                        │
│                                                                          │
│  Layer 2: PROCESSING                                                     │
│  ────────────────────                                                   │
│  "Compute aggregations, detect patterns, enrich in real-time"            │
│  Tools: Apache Flink, Kafka Streams, Spark Structured Streaming          │
│  Operations: Window, aggregate, join, filter, alert                      │
│                                                                          │
│  Layer 3: SERVING                                                        │
│  ────────────────                                                       │
│  "Store pre-computed results for instant query responses"                │
│  Tools: Apache Druid, ClickHouse, Apache Pinot, Rockset                  │
│  Latency: Sub-second queries over billions of rows                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Step 3: The Complete Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 REAL-TIME ANALYTICS ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  EVENT SOURCES           INGESTION        PROCESSING       SERVING           │
│                                                                              │
│  ┌─────────────┐      ┌──────────┐     ┌──────────┐     ┌──────────────┐   │
│  │ Web/Mobile   │─────▶│          │     │          │     │              │   │
│  │ Clickstream  │      │          │     │  Apache  │     │ Apache Druid │   │
│  └─────────────┘      │          │     │  Flink   │     │ / ClickHouse │   │
│                       │  Apache  │────▶│          │────▶│ / Pinot      │   │
│  ┌─────────────┐      │  Kafka   │     │  Window  │     │              │   │
│  │ IoT Sensors  │─────▶│          │     │  Agg.    │     │ Pre-computed │   │
│  │ (millions)   │      │  Topics  │     │  Join    │     │ aggregates   │   │
│  └─────────────┘      │          │     │  Filter  │     └──────┬───────┘   │
│                       │          │     └──────────┘            │            │
│  ┌─────────────┐      │          │                             │            │
│  │ DB Changes   │─────▶│          │                             ▼            │
│  │ (CDC)        │      └──────────┘            ┌────────────────────────┐   │
│  └─────────────┘                               │  Dashboard / API       │   │
│                                                │  (Grafana, Superset,   │   │
│  ┌─────────────┐                               │   custom apps)         │   │
│  │ Server Logs  │─────▶  (Kafka)               └────────────────────────┘   │
│  └─────────────┘                                                            │
│                                                                              │
│  End-to-End Latency: Event occurs → visible on dashboard in < 5 seconds     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Step 4: Lambda vs Kappa Architecture

Two competing philosophies for real-time + batch analytics:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LAMBDA ARCHITECTURE                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Events ──▶ Kafka ──┬──▶ [BATCH LAYER: Spark, daily jobs]               │
│                     │       Store in data lake                           │
│                     │       Accurate but slow (hours)                    │
│                     │              │                                     │
│                     │              ▼                                     │
│                     │    ┌──────────────────┐                            │
│                     │    │  SERVING LAYER    │ ◀── Dashboard queries     │
│                     │    │  (Merge batch +   │                           │
│                     │    │   speed results)  │                           │
│                     │    └──────────────────┘                            │
│                     │              ▲                                     │
│                     │              │                                     │
│                     └──▶ [SPEED LAYER: Flink, real-time]                 │
│                            Approximate, fast (seconds)                   │
│                                                                          │
│  Problem: Two codebases doing the same logic. Hard to maintain.          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                    KAPPA ARCHITECTURE (Simpler)                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Events ──▶ Kafka ──▶ [SINGLE STREAM PROCESSOR: Flink]                   │
│                              │                                           │
│                              ▼                                           │
│                    ┌──────────────────┐                                   │
│                    │  SERVING LAYER    │ ◀── Dashboard queries            │
│                    └──────────────────┘                                   │
│                                                                          │
│  Everything goes through ONE path. If you need to reprocess,             │
│  replay events from Kafka's log from the beginning.                      │
│                                                                          │
│  Simpler! One codebase. One processing model. Replay for corrections.    │
└─────────────────────────────────────────────────────────────────────────┘
```

**Industry trend**: Kappa Architecture is winning. Flink is powerful enough to handle both real-time AND historical reprocessing.

---

## How It Works Internally

### Stream Processing: Windowing

The core challenge in real-time analytics: events arrive continuously, but you need to compute aggregates (counts, sums, averages) over time periods:

```
┌─────────────────────────────────────────────────────────────────┐
│                  WINDOWING STRATEGIES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  TUMBLING WINDOW (Fixed, non-overlapping):                        │
│  ─────────────────────────────────────────                       │
│  "Count orders per minute"                                        │
│                                                                   │
│  Events: ─e1──e2──e3──|──e4──e5──|──e6──e7──e8──|──▶ time       │
│  Windows: [  Window 1  ][  Window 2  ][  Window 3  ]             │
│  Result:  count=3        count=2       count=3                    │
│                                                                   │
│  ─────────────────────────────────────────────────────────────── │
│                                                                   │
│  SLIDING WINDOW (Overlapping):                                    │
│  ─────────────────────────────                                   │
│  "Average order value over last 5 minutes, updated every minute"  │
│                                                                   │
│  Events: ─e1──e2──e3──e4──e5──e6──e7──e8──▶ time                │
│  Windows: [──────── 5 min ────────]                               │
│                [──────── 5 min ────────]                          │
│                     [──────── 5 min ────────]                     │
│  Result: Windows overlap, each emits its own aggregate            │
│                                                                   │
│  ─────────────────────────────────────────────────────────────── │
│                                                                   │
│  SESSION WINDOW (Activity-based):                                 │
│  ────────────────────────────────                                │
│  "Group user actions until 30 min of inactivity"                  │
│                                                                   │
│  Events: ─e1─e2─e3─────────────────────e4─e5──────────────▶     │
│  Windows: [Session 1: e1,e2,e3]  gap   [Session 2: e4,e5]       │
│           (user active)         (idle)  (user returns)            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### OLAP Serving Layer: How Druid/Pinot Handle Real-Time Queries

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 APACHE DRUID ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Real-Time Ingestion          Historical Data                            │
│  ────────────────────          ───────────────                           │
│                                                                          │
│  Kafka ──▶ ┌─────────────┐                                              │
│            │ Real-Time    │    ┌─────────────┐                           │
│            │ Nodes        │    │ Historical   │                           │
│            │              │    │ Nodes        │                           │
│            │ Latest data  │    │              │                           │
│            │ (last hour)  │    │ Older data   │                           │
│            │ In memory    │    │ (days-years) │                           │
│            └──────┬───────┘    │ On disk/S3   │                           │
│                   │            └──────┬───────┘                           │
│                   │                   │                                   │
│                   └─────────┬─────────┘                                   │
│                             │                                             │
│                             ▼                                             │
│                   ┌─────────────────┐                                     │
│                   │  Broker Nodes    │ ◀── SQL Queries from dashboards    │
│                   │                 │                                     │
│                   │  Merges results  │                                     │
│                   │  from real-time  │                                     │
│                   │  + historical    │                                     │
│                   └─────────────────┘                                     │
│                                                                          │
│  Result: Query across ALL data (real-time + historical)                  │
│          returns in < 1 second, even over billions of rows               │
│                                                                          │
│  Secret sauce:                                                           │
│  • Columnar storage with bitmap indexes                                  │
│  • Pre-aggregation at ingestion (rollup)                                 │
│  • Segment-level parallelism                                             │
│  • Approximate algorithms (HyperLogLog, quantile sketches)               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Late-Arriving Events (Watermarks)

Real-world events don't always arrive in order. A user's click at 12:00:00 might arrive at your system at 12:00:05 due to network delays:

```
┌─────────────────────────────────────────────────────────────────┐
│                  HANDLING LATE EVENTS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Event Time vs Processing Time:                                   │
│                                                                   │
│  Event Time:       12:00  12:01  12:02  12:03  12:04             │
│  Arrival Time:     12:00  12:02  12:01  12:05  12:04             │
│                              ▲       ▲                            │
│                              │       │                            │
│                    Arrived   Arrived                              │
│                    late!     out of order!                        │
│                                                                   │
│  Watermark: "I've seen all events up to time T"                   │
│                                                                   │
│  Processing Time: ──────────────────────────────────▶            │
│  Watermark:       ─────────W(12:00)──W(12:01)───────▶           │
│                                                                   │
│  When watermark passes window end → emit window result            │
│  Late events after watermark → either discard or update           │
│                                                                   │
│  Flink allows:                                                    │
│  • allowedLateness(5 minutes) → keep window open for late events  │
│  • Side outputs → route late events to separate stream            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: Real-Time Analytics with Apache Flink (PyFlink)

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment, EnvironmentSettings
from pyflink.table.window import Tumble
from pyflink.table.expressions import col, lit

# ═══════════════════════════════════════════════════════
# Real-Time Analytics: Compute orders-per-minute from Kafka
# Using Apache Flink's Table API
# ═══════════════════════════════════════════════════════

def build_realtime_analytics_pipeline():
    """
    Flink job that reads order events from Kafka,
    computes per-minute aggregations, and writes to Druid.
    """
    env = StreamExecutionEnvironment.get_execution_environment()
    env.set_parallelism(4)  # 4 parallel workers
    
    t_env = StreamTableEnvironment.create(env)
    
    # Define source: Kafka topic with order events
    t_env.execute_sql("""
        CREATE TABLE order_events (
            order_id BIGINT,
            user_id BIGINT,
            product_id INT,
            amount DECIMAL(10,2),
            category STRING,
            city STRING,
            event_time TIMESTAMP(3),
            WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
        ) WITH (
            'connector' = 'kafka',
            'topic' = 'orders.created',
            'properties.bootstrap.servers' = 'kafka:9092',
            'format' = 'json',
            'scan.startup.mode' = 'latest-offset'
        )
    """)
    
    # Define sink: Real-time OLAP (Druid/ClickHouse)
    t_env.execute_sql("""
        CREATE TABLE realtime_metrics (
            window_start TIMESTAMP(3),
            window_end TIMESTAMP(3),
            category STRING,
            city STRING,
            order_count BIGINT,
            total_revenue DECIMAL(12,2),
            avg_order_value DECIMAL(10,2),
            unique_users BIGINT
        ) WITH (
            'connector' = 'jdbc',
            'url' = 'jdbc:clickhouse://clickhouse:8123/analytics',
            'table-name' = 'realtime_order_metrics'
        )
    """)
    
    # Real-time aggregation: 1-minute tumbling windows
    t_env.execute_sql("""
        INSERT INTO realtime_metrics
        SELECT
            TUMBLE_START(event_time, INTERVAL '1' MINUTE) AS window_start,
            TUMBLE_END(event_time, INTERVAL '1' MINUTE) AS window_end,
            category,
            city,
            COUNT(*) AS order_count,
            SUM(amount) AS total_revenue,
            AVG(amount) AS avg_order_value,
            COUNT(DISTINCT user_id) AS unique_users
        FROM order_events
        GROUP BY
            TUMBLE(event_time, INTERVAL '1' MINUTE),
            category,
            city
    """)


def build_fraud_detection_pipeline():
    """
    Real-time fraud detection: Alert if a user makes > 5 orders
    in any 10-minute sliding window.
    """
    # This pattern detects rapid repeated purchases (potential fraud)
    t_env.execute_sql("""
        INSERT INTO fraud_alerts
        SELECT
            user_id,
            HOP_START(event_time, INTERVAL '1' MINUTE, INTERVAL '10' MINUTE) AS window_start,
            COUNT(*) AS order_count,
            SUM(amount) AS total_amount
        FROM order_events
        GROUP BY
            user_id,
            HOP(event_time, INTERVAL '1' MINUTE, INTERVAL '10' MINUTE)
        HAVING COUNT(*) > 5 OR SUM(amount) > 10000
    """)
```

### Python: Simple Real-Time Dashboard Backend

```python
from kafka import KafkaConsumer
from collections import defaultdict
from datetime import datetime, timedelta
import json
import threading

# ═══════════════════════════════════════════════════════
# Lightweight real-time metrics aggregator
# For small-scale use without Flink (< 10K events/sec)
# ═══════════════════════════════════════════════════════

class RealTimeAggregator:
    """
    In-memory aggregator for real-time metrics.
    Maintains rolling 1-minute windows.
    Serves instant query responses for dashboards.
    """
    
    def __init__(self):
        # Window: minute → category → metrics
        self.windows = defaultdict(lambda: defaultdict(lambda: {
            'count': 0, 'revenue': 0.0, 'users': set()
        }))
        self.lock = threading.Lock()
    
    def process_event(self, event: dict):
        """Process a single order event into the current window."""
        event_time = datetime.fromisoformat(event['timestamp'])
        window_key = event_time.strftime('%Y-%m-%d %H:%M')  # 1-min window
        category = event['category']
        
        with self.lock:
            window = self.windows[window_key][category]
            window['count'] += 1
            window['revenue'] += event['amount']
            window['users'].add(event['user_id'])
    
    def get_current_metrics(self) -> dict:
        """Return metrics for the last 5 minutes (dashboard query)."""
        now = datetime.now()
        results = []
        
        with self.lock:
            for i in range(5):
                minute = (now - timedelta(minutes=i)).strftime('%Y-%m-%d %H:%M')
                if minute in self.windows:
                    for category, metrics in self.windows[minute].items():
                        results.append({
                            'window': minute,
                            'category': category,
                            'orders': metrics['count'],
                            'revenue': metrics['revenue'],
                            'unique_users': len(metrics['users']),
                        })
        
        return {'timestamp': now.isoformat(), 'metrics': results}
    
    def cleanup_old_windows(self):
        """Remove windows older than 10 minutes to prevent memory growth."""
        cutoff = (datetime.now() - timedelta(minutes=10)).strftime('%Y-%m-%d %H:%M')
        with self.lock:
            old_keys = [k for k in self.windows if k < cutoff]
            for k in old_keys:
                del self.windows[k]


def start_consumer(aggregator: RealTimeAggregator):
    """Kafka consumer that feeds events into the aggregator."""
    consumer = KafkaConsumer(
        'orders.created',
        bootstrap_servers=['localhost:9092'],
        value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        group_id='realtime-dashboard',
    )
    
    for message in consumer:
        aggregator.process_event(message.value)
```

### Java: Kafka Streams Real-Time Aggregation

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.kstream.Materialized;
import org.apache.kafka.streams.state.QueryableStoreTypes;

import java.time.Duration;
import java.util.Properties;

public class RealTimeAnalyticsStream {
    
    /**
     * Kafka Streams application that computes real-time order metrics.
     * Results are queryable via Interactive Queries (REST API).
     */
    public KafkaStreams buildTopology() {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "realtime-analytics");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000); // Flush every second
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Read order events from Kafka
        KStream<String, OrderEvent> orders = builder.stream(
            "orders.created",
            Consumed.with(Serdes.String(), new OrderEventSerde())
        );
        
        // Real-time aggregation: orders per category per minute
        KTable<Windowed<String>, OrderMetrics> metrics = orders
            .groupBy(
                (key, order) -> order.getCategory(),  // Group by category
                Grouped.with(Serdes.String(), new OrderEventSerde())
            )
            .windowedBy(
                TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1))  // 1-min tumbling
            )
            .aggregate(
                OrderMetrics::new,                   // Initial value
                (category, order, agg) -> {          // Aggregation logic
                    agg.incrementCount();
                    agg.addRevenue(order.getAmount());
                    agg.addUser(order.getUserId());
                    return agg;
                },
                Materialized.as("order-metrics-store")  // Queryable state store
                    .withKeySerde(Serdes.String())
                    .withValueSerde(new OrderMetricsSerde())
            );
        
        // Write aggregated results to output topic (for downstream consumers)
        metrics.toStream()
            .map((windowedKey, value) -> KeyValue.pair(
                windowedKey.key() + "@" + windowedKey.window().start(),
                value.toJson()
            ))
            .to("realtime.order.metrics", Produced.with(Serdes.String(), Serdes.String()));
        
        // Also: detect anomalies (sudden spike in orders)
        orders
            .groupBy((key, order) -> "global", Grouped.with(Serdes.String(), new OrderEventSerde()))
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
            .count(Materialized.as("order-count-store"))
            .toStream()
            .filter((window, count) -> count > 10000)  // Anomaly: > 10K orders/min
            .mapValues(count -> "ALERT: " + count + " orders in last minute!")
            .to("alerts.high-volume");
        
        return new KafkaStreams(builder.build(), props);
    }
    
    /**
     * Interactive Query: Serve metrics via REST API without external DB.
     * Kafka Streams stores state locally — queryable in-process!
     */
    public OrderMetrics queryCurrentMetrics(KafkaStreams streams, String category) {
        var store = streams.store(
            StoreQueryParameters.fromNameAndType(
                "order-metrics-store",
                QueryableStoreTypes.windowStore()
            )
        );
        
        // Fetch latest window for this category
        var iterator = store.backwardFetch(
            category,
            Instant.now().minus(Duration.ofMinutes(5)),
            Instant.now()
        );
        
        if (iterator.hasNext()) {
            return iterator.next().value;
        }
        return new OrderMetrics(); // Empty
    }
}
```

---

## Infrastructure Examples

### Complete Real-Time Analytics Stack (Docker Compose)

```yaml
# docker-compose.yml — Full real-time analytics stack
version: '3.8'

services:
  # Event Ingestion
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      CLUSTER_ID: "analytics-cluster-001"

  # Stream Processing
  flink-jobmanager:
    image: flink:1.18
    ports:
      - "8081:8081"  # Flink Web UI
    command: jobmanager
    environment:
      FLINK_PROPERTIES: |
        jobmanager.rpc.address: flink-jobmanager
        state.backend: rocksdb
        state.checkpoints.dir: s3://checkpoints/flink

  flink-taskmanager:
    image: flink:1.18
    command: taskmanager
    environment:
      FLINK_PROPERTIES: |
        jobmanager.rpc.address: flink-jobmanager
        taskmanager.numberOfTaskSlots: 4
        taskmanager.memory.process.size: 4g
    deploy:
      replicas: 3  # 3 workers × 4 slots = 12 parallel tasks

  # Serving Layer (Real-Time OLAP)
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse_data:/var/lib/clickhouse

  # Visualization
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  clickhouse_data:
  grafana_data:
```

### ClickHouse Table for Real-Time Metrics

```sql
-- ClickHouse: Materialized View for real-time aggregation
-- Events flow in from Kafka → auto-aggregated

-- Raw events table (fed from Kafka)
CREATE TABLE order_events_raw (
    event_time    DateTime64(3),
    order_id      UInt64,
    user_id       UInt64,
    product_id    UInt32,
    category      LowCardinality(String),
    city          LowCardinality(String),
    amount        Decimal(10,2)
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list = 'orders.created',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow';

-- Materialized View: Auto-aggregates into per-minute metrics
CREATE MATERIALIZED VIEW order_metrics_mv
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(window_start)
ORDER BY (window_start, category, city)
AS SELECT
    toStartOfMinute(event_time) AS window_start,
    category,
    city,
    countState() AS order_count,
    sumState(amount) AS total_revenue,
    avgState(amount) AS avg_order_value,
    uniqState(user_id) AS unique_users
FROM order_events_raw
GROUP BY window_start, category, city;

-- Query the real-time metrics (sub-second response!)
SELECT
    window_start,
    category,
    countMerge(order_count) AS orders,
    sumMerge(total_revenue) AS revenue,
    avgMerge(avg_order_value) AS avg_value,
    uniqMerge(unique_users) AS users
FROM order_metrics_mv
WHERE window_start >= now() - INTERVAL 5 MINUTE
GROUP BY window_start, category
ORDER BY window_start DESC, revenue DESC;
```

### Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "Real-Time Order Analytics",
    "panels": [
      {
        "title": "Orders Per Minute (Live)",
        "type": "timeseries",
        "datasource": "ClickHouse",
        "targets": [{
          "rawSql": "SELECT window_start AS time, countMerge(order_count) AS orders FROM order_metrics_mv WHERE $__timeFilter(window_start) GROUP BY window_start ORDER BY time",
          "refId": "A"
        }],
        "fieldConfig": {
          "defaults": { "unit": "short" }
        },
        "options": {
          "tooltip": { "mode": "multi" }
        }
      },
      {
        "title": "Revenue by Category (Last 5 min)",
        "type": "piechart",
        "datasource": "ClickHouse",
        "targets": [{
          "rawSql": "SELECT category, sumMerge(total_revenue) AS revenue FROM order_metrics_mv WHERE window_start >= now() - INTERVAL 5 MINUTE GROUP BY category ORDER BY revenue DESC"
        }]
      }
    ],
    "refresh": "5s"
  }
}
```

---

## Real-World Example

### LinkedIn's Real-Time Analytics (Apache Pinot)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LINKEDIN'S REAL-TIME ANALYTICS                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Use Cases:                                                              │
│  • "Who viewed your profile" (updates in seconds)                        │
│  • Ad campaign performance (real-time impressions/clicks)                │
│  • Feed ranking signals (trending posts)                                 │
│  • Abuse detection (spam, fake accounts)                                 │
│                                                                          │
│  Architecture:                                                           │
│                                                                          │
│  LinkedIn App (700M users)                                               │
│       │                                                                  │
│       │ Billions of events/day                                           │
│       ▼                                                                  │
│  ┌─────────────────┐                                                     │
│  │  Apache Kafka    │  (Unified event bus)                                │
│  │  2+ trillion     │                                                    │
│  │  messages/day    │                                                    │
│  └────────┬────────┘                                                     │
│           │                                                              │
│     ┌─────┴──────────────────────┐                                       │
│     ▼                            ▼                                       │
│  ┌────────────────┐    ┌────────────────────┐                            │
│  │ Apache Samza    │    │ Apache Pinot        │                            │
│  │ (Stream Proc.)  │    │ (Real-Time OLAP)    │                            │
│  │                │    │                    │                            │
│  │ • Enrichment   │    │ • 100K+ queries/sec │                            │
│  │ • Complex CEP  │    │ • Sub-second P99    │                            │
│  │ • Sessionize   │────▶│ • 100+ TB data     │                            │
│  └────────────────┘    │ • Hybrid tables     │                            │
│                        │   (real-time +      │                            │
│                        │    offline segments) │                            │
│                        └─────────┬──────────┘                            │
│                                  │                                       │
│                                  ▼                                       │
│                        ┌────────────────────┐                            │
│                        │  Applications       │                            │
│                        │  • Who Viewed       │                            │
│                        │  • Ad Analytics     │                            │
│                        │  • Feed Ranking     │                            │
│                        └────────────────────┘                            │
│                                                                          │
│  Scale:                                                                  │
│  • 1000+ Pinot tables                                                    │
│  • 100,000+ queries/second                                               │
│  • Sub-100ms P99 latency                                                 │
│  • Ingests millions of events/second in real-time                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Uber's Real-Time Analytics

- **Platform**: Custom built on Apache Pinot + Apache Flink
- **Use cases**: Surge pricing (recomputed every 30 seconds), real-time supply/demand, driver earnings dashboards
- **Scale**: Processes 10M+ events/sec, serves 100K+ queries/sec
- **Innovation**: "AresDB" — GPU-accelerated real-time analytics engine for geospatial queries

### Netflix

- **Tool**: Apache Druid for real-time streaming analytics
- **Use case**: Real-time viewing metrics (how many people are watching show X RIGHT NOW), A/B test monitoring, infrastructure health
- **Scale**: Ingests billions of events/day, dashboards update in seconds

---

## Comparison of Real-Time OLAP Engines

| Feature | Apache Druid | ClickHouse | Apache Pinot | Rockset |
|---------|-------------|------------|--------------|---------|
| **Origin** | Metamarkets → Apache | Yandex | LinkedIn → Apache | Ex-Facebook |
| **Best For** | Time-series analytics | General OLAP | User-facing analytics | Real-time SQL |
| **Ingestion** | Kafka native | Kafka, batch | Kafka native | CDC, streams |
| **Query Latency** | Sub-second | Sub-second | Sub-second | Sub-second |
| **Concurrency** | High (1000+ QPS) | Medium (100s QPS) | Very High (100K+ QPS) | High |
| **Storage** | Segments on deep storage | Local + S3 | Segments on S3 | Cloud-native |
| **SQL Support** | Yes (Druid SQL) | Full SQL | Yes (multi-stage) | Full SQL |
| **Updates** | Limited | Yes (ReplacingMergeTree) | Upsert support | Yes |
| **Cloud Managed** | Imply Cloud | ClickHouse Cloud | StarTree Cloud | Rockset Cloud |
| **Used By** | Netflix, Airbnb, eBay | Uber, Cloudflare | LinkedIn, Uber | Grubhub |

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Not handling late events | Window results become inaccurate as late data arrives | Use watermarks + allowed lateness in Flink |
| No backpressure handling | System crashes during traffic spikes | Configure Kafka consumer rate limits, Flink backpressure |
| Over-precise granularity | Storing per-second metrics for 3 years = massive storage cost | Pre-aggregate (1-min for recent, 1-hour for old data) |
| Not using approximate algorithms | Exact COUNT DISTINCT on billions = slow/expensive | Use HyperLogLog for unique counts, t-digest for percentiles |
| Single point of failure | Flink job crashes → dashboard goes dark | HA with checkpointing, standby jobs, monitoring |
| Ignoring event ordering | Out-of-order events cause incorrect aggregates | Use event-time processing with watermarks, not processing time |
| No replay capability | Bug in processing → corrupted results, can't fix | Store raw events in Kafka (7+ day retention), replay from offset |

---

## When to Use / When NOT to Use

### Use Real-Time Analytics When:
- ✅ Business decisions depend on up-to-the-second data (fraud, dynamic pricing)
- ✅ Operational dashboards need live monitoring (site reliability, DevOps)
- ✅ User-facing features require instant metrics ("3 people viewing this item")
- ✅ Anomaly detection must trigger alerts within seconds
- ✅ A/B tests need real-time statistical significance

### When NOT to Use Real-Time Analytics:
- ❌ Daily/weekly reports are sufficient (use batch — far simpler and cheaper)
- ❌ Your data volume is small (< 100K events/day) — just query your OLTP DB
- ❌ You don't have the team to operate Kafka + Flink + OLAP systems
- ❌ Approximate results are unacceptable (some real-time algorithms sacrifice precision)
- ❌ Cost is a primary constraint (real-time infra is expensive to run)

### Consider Near-Real-Time Instead:
- 🤔 If 5-minute latency is acceptable, micro-batch (Spark Structured Streaming with 5-min trigger) is much simpler
- 🤔 Many "real-time" requirements are actually "fresh enough" requirements

### The Decision Framework:

```
"How stale can the data be?"
│
├── Hours/days ──────▶ Batch (Airflow + BigQuery)     ← Simplest, cheapest
├── Minutes (1-15) ──▶ Micro-batch (Spark)            ← Middle ground
└── Seconds ─────────▶ True streaming (Flink + Druid) ← Most complex, expensive
```

---

## Key Takeaways

1. **Real-time analytics** = ingestion (Kafka) + processing (Flink) + serving (Druid/ClickHouse/Pinot). All three layers must be designed for low latency.

2. **Windowing** (tumbling, sliding, session) is the fundamental abstraction for computing aggregates over unbounded event streams.

3. **Kappa Architecture** (single streaming path) is replacing Lambda Architecture (separate batch + streaming). Flink is powerful enough to handle both.

4. **Real-time OLAP engines** (Druid, Pinot, ClickHouse) are purpose-built to serve sub-second queries over billions of rows with high concurrency — they're NOT general-purpose databases.

5. **Watermarks** solve the late-arriving events problem by defining when a window is "complete enough" to emit results.

6. **Approximate algorithms** (HyperLogLog, Count-Min Sketch, t-digest) are essential for real-time systems — exact answers at scale are too expensive.

7. **Don't over-engineer**: If hourly batch is good enough, use batch. Real-time analytics infrastructure (Kafka + Flink + Druid) is powerful but expensive to build and operate.

---

## What's Next?

Congratulations! You've completed Part 20: Data Engineering in Web Apps. You now understand:
- OLTP vs OLAP (Chapter 20.1)
- Data Pipelines — ETL & ELT (Chapter 20.2)
- Data Lakes & Warehouses (Chapter 20.3)
- Change Data Capture (Chapter 20.4)
- Real-Time Analytics Architecture (this chapter)

Next up is **Part 21: Service Discovery, Configuration & Coordination** — starting with [Chapter 21.1: Service Discovery — How Services Find Each Other](../21-service-discovery/01-service-discovery.md), where you'll learn how microservices dynamically locate and communicate with each other in large distributed systems.
