# Stream Processing (Kafka Streams, Apache Flink)

> **What you'll learn**: How to process millions of events per second in real-time using stream processing frameworks — including the difference between batch and stream processing, windowing, exactly-once semantics, stateful processing, and how Netflix, LinkedIn, and Uber use stream processing at planet-scale.

---

## Real-Life Analogy

Imagine you're running a **massive highway toll system** for an entire country.

**Batch Processing** (the old way):
- Every night at midnight, a truck collects all the paper toll receipts from every booth
- Takes them to a warehouse, sorts them, calculates totals
- By morning, you know yesterday's revenue
- Problem: You only know what happened YESTERDAY. If a toll booth broke at 9 AM, you don't know until the NEXT morning.

**Stream Processing** (the modern way):
- Every toll booth has an electronic sensor
- As each car passes, the event is processed IMMEDIATELY
- You know revenue in REAL-TIME, detect anomalies INSTANTLY
- If a booth stops reporting, you know within SECONDS
- You can react: dynamically open more lanes during rush hour, flag suspicious vehicles

Stream processing = **analyzing data as it flows**, not after it's stored.

---

## Core Concept Explained Step-by-Step

### Step 1: Batch Processing vs Stream Processing

```
┌─────────────────────────────────────────────────────────────────────┐
│              BATCH vs STREAM PROCESSING                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  BATCH PROCESSING                    STREAM PROCESSING               │
│  ════════════════                    ════════════════                 │
│                                                                      │
│  • Process data in large chunks      • Process data as it arrives    │
│  • Minutes/hours latency             • Milliseconds/seconds latency  │
│  • Bounded datasets                  • Unbounded (infinite) streams  │
│  • High throughput                   • Low latency                   │
│  • Simple to reason about            • Complex (ordering, state)     │
│                                                                      │
│  Examples:                           Examples:                        │
│  - Nightly reports                   - Fraud detection               │
│  - Monthly billing                   - Real-time dashboards          │
│  - Data warehouse ETL                - Live recommendations          │
│  - Hadoop MapReduce                  - IoT sensor monitoring         │
│                                                                      │
│  Tools:                              Tools:                           │
│  - Apache Spark (batch mode)         - Apache Kafka Streams          │
│  - Apache Hadoop                     - Apache Flink                  │
│  - AWS Glue                          - Apache Spark Streaming        │
│  - Google Dataflow (batch)           - AWS Kinesis                   │
│                                      - Google Dataflow (streaming)   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 2: What is a Stream?

A **stream** is an unbounded, continuously flowing sequence of events.

```
Time ──────────────────────────────────────────────────────▶

Events:  [e1]  [e2]  [e3]  [e4]  [e5]  [e6]  [e7] ... (never ends)

Examples of streams:
─────────────────────────────────────────────────────
│ Source              │ Event                          │
─────────────────────────────────────────────────────
│ E-commerce site     │ User clicked "Add to Cart"     │
│ Payment system      │ Transaction of ₹499 processed  │
│ IoT sensor          │ Temperature reading: 42°C      │
│ Social media        │ New tweet posted                │
│ Ride-sharing app    │ Driver location updated         │
│ Gaming platform     │ Player scored 100 points        │
─────────────────────────────────────────────────────
```

### Step 3: Stream Processing Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                  STREAM PROCESSING ARCHITECTURE                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────┐    ┌─────────────┐    ┌──────────────┐    ┌─────────┐ │
│  │ Sources │    │  Message     │    │   Stream     │    │  Sinks  │ │
│  │         │───▶│  Broker      │───▶│  Processor   │───▶│         │ │
│  │(Producers)   │(Kafka/Kinesis)    │(Flink/KStreams)   │(Outputs)│ │
│  └─────────┘    └─────────────┘    └──────────────┘    └─────────┘ │
│                                                                       │
│  Sources:           Broker:           Processor:        Sinks:        │
│  • Web apps         • Buffers events  • Transforms     • Database    │
│  • Mobile apps      • Guarantees      • Filters        • Dashboard   │
│  • IoT devices        delivery        • Aggregates     • Alert       │
│  • Databases (CDC)  • Partitions      • Joins          • Another     │
│  • Log files          for scale       • Windows          topic       │
│                                       • Enriches                      │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Step 4: Kafka Streams — Stream Processing Inside Your App

Kafka Streams is a **library** (not a separate cluster) that lets you process Kafka topics within your application.

```
┌────────────────────────────────────────────────────────────┐
│              KAFKA STREAMS ARCHITECTURE                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                YOUR APPLICATION (JVM)                  │  │
│  │                                                        │  │
│  │  ┌────────────────────────────────────────────────┐   │  │
│  │  │            Kafka Streams Library                 │   │  │
│  │  │                                                  │   │  │
│  │  │  Input Topic ──▶ [Filter] ──▶ [Map] ──▶ Output  │   │  │
│  │  │  "orders"        │            │         Topic    │   │  │
│  │  │                  │            │       "alerts"   │   │  │
│  │  │                  ▼            ▼                   │   │  │
│  │  │              State Store (RocksDB)                │   │  │
│  │  │              (for aggregations, joins)            │   │  │
│  │  └────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Key Benefits:                                              │
│  • No separate cluster needed (just a library!)            │
│  • Exactly-once processing                                  │
│  • Scales by running more app instances                     │
│  • Fault-tolerant (state backed up to Kafka)               │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

**Java Example — Kafka Streams:**

```java
// Fraud Detection: Flag orders > ₹50,000 in real-time
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders");

// Filter high-value orders
KStream<String, Order> highValueOrders = orders
    .filter((key, order) -> order.getAmount() > 50000);

// Count orders per customer in last 5 minutes
KTable<Windowed<String>, Long> orderCounts = orders
    .groupBy((key, order) -> order.getCustomerId())
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
    .count();

// Flag suspicious: more than 3 high-value orders in 5 mins
orderCounts
    .toStream()
    .filter((windowedKey, count) -> count > 3)
    .to("fraud-alerts");

// Build and start
KafkaStreams streams = new KafkaStreams(builder.build(), config);
streams.start();
```

**Python Example — Faust (Kafka Streams for Python):**

```python
import faust

app = faust.App('fraud-detector', broker='kafka://localhost:9092')

class Order(faust.Record):
    order_id: str
    customer_id: str
    amount: float
    timestamp: float

orders_topic = app.topic('orders', value_type=Order)

# Sliding window table: count orders per customer in last 5 minutes
order_counts = app.Table(
    'order_counts',
    default=int,
).tumbling(300)  # 5-minute window

@app.agent(orders_topic)
async def detect_fraud(orders):
    async for order in orders:
        if order.amount > 50000:
            order_counts[order.customer_id] += 1
            
            if order_counts[order.customer_id].now() > 3:
                print(f"🚨 FRAUD ALERT: Customer {order.customer_id} "
                      f"placed {order_counts[order.customer_id].now()} "
                      f"high-value orders in 5 minutes!")
                await fraud_alerts_topic.send(value=order)
```

### Step 5: Apache Flink — The Most Powerful Stream Processor

Apache Flink is a **distributed stream processing framework** that runs as its own cluster.

```
┌────────────────────────────────────────────────────────────────┐
│                 APACHE FLINK ARCHITECTURE                        │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────┐     ┌────────────────────────────────────┐      │
│  │  Client   │     │         Flink Cluster               │      │
│  │  (Submit  │────▶│                                      │      │
│  │   Job)    │     │  ┌──────────────┐                   │      │
│  └───────────┘     │  │ Job Manager  │ (coordinator)     │      │
│                     │  │  • Schedules │                   │      │
│                     │  │  • Checkpoints│                  │      │
│                     │  │  • Recovery   │                  │      │
│                     │  └──────┬───────┘                   │      │
│                     │         │                            │      │
│                     │    ┌────┼────┐                       │      │
│                     │    ▼    ▼    ▼                       │      │
│                     │  ┌───┐┌───┐┌───┐                    │      │
│                     │  │TM1││TM2││TM3│ (Task Managers)    │      │
│                     │  │   ││   ││   │                    │      │
│                     │  │ ○ ││ ○ ││ ○ │ ○ = operator slot  │      │
│                     │  │ ○ ││ ○ ││ ○ │                    │      │
│                     │  └───┘└───┘└───┘                    │      │
│                     └────────────────────────────────────┘      │
│                                                                 │
│  Key Benefits:                                                  │
│  • True streaming (not micro-batching like Spark)              │
│  • Exactly-once state consistency                               │
│  • Event time processing (handles late data)                   │
│  • Savepoints (pause, resume, scale jobs)                      │
│  • SQL support (Flink SQL)                                      │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

**Java Example — Flink Fraud Detection:**

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// Enable exactly-once checkpointing every 60 seconds
env.enableCheckpointing(60000, CheckpointingMode.EXACTLY_ONCE);

// Read from Kafka
DataStream<Order> orders = env
    .addSource(new FlinkKafkaConsumer<>("orders", new OrderSchema(), kafkaProps));

// Detect fraud: same card used from 2 different countries within 5 minutes
DataStream<FraudAlert> alerts = orders
    .keyBy(Order::getCardNumber)
    .window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
    .process(new ProcessWindowFunction<Order, FraudAlert, String, TimeWindow>() {
        @Override
        public void process(String cardNumber, Context ctx, 
                          Iterable<Order> orders, Collector<FraudAlert> out) {
            Set<String> countries = new HashSet<>();
            for (Order order : orders) {
                countries.add(order.getCountry());
            }
            if (countries.size() > 1) {
                out.collect(new FraudAlert(cardNumber, countries, ctx.window()));
            }
        }
    });

alerts.addSink(new FlinkKafkaProducer<>("fraud-alerts", new AlertSchema(), kafkaProps));
env.execute("Fraud Detection Job");
```

### Step 6: Windowing — Grouping Infinite Streams into Finite Chunks

Since streams are infinite, you need **windows** to perform aggregations.

```
┌─────────────────────────────────────────────────────────────────┐
│                    TYPES OF WINDOWS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. TUMBLING WINDOW (fixed, non-overlapping)                    │
│  ═══════════════════════════════════════════                     │
│                                                                  │
│  Time: ──|──────────|──────────|──────────|──▶                  │
│           Window 1    Window 2    Window 3                       │
│          [e1,e2,e3]  [e4,e5]    [e6,e7,e8]                     │
│                                                                  │
│  Use: "Count orders every 5 minutes"                            │
│                                                                  │
│  ─────────────────────────────────────────────────              │
│                                                                  │
│  2. SLIDING WINDOW (fixed, overlapping)                         │
│  ═══════════════════════════════════════                         │
│                                                                  │
│  Time: ──|────────────────|──▶                                  │
│           |──────────────|                                       │
│              |──────────────|                                    │
│           Window 1                                               │
│              Window 2                                            │
│                 Window 3                                         │
│                                                                  │
│  Use: "Average latency over last 5 min, updated every 1 min"   │
│                                                                  │
│  ─────────────────────────────────────────────────              │
│                                                                  │
│  3. SESSION WINDOW (dynamic, gap-based)                         │
│  ═══════════════════════════════════════                         │
│                                                                  │
│  Time: ──[e1 e2 e3]─────gap─────[e4 e5]────gap────[e6]──▶     │
│           Session 1               Session 2         Session 3   │
│                                                                  │
│  Use: "User activity sessions (new session after 30 min idle)" │
│                                                                  │
│  ─────────────────────────────────────────────────              │
│                                                                  │
│  4. GLOBAL WINDOW (all events in one window)                    │
│  ═══════════════════════════════════════════                     │
│                                                                  │
│  Time: ──[e1 e2 e3 e4 e5 e6 e7 ... forever]──▶                │
│                                                                  │
│  Use: "Running count/sum (needs custom trigger)"                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 7: Stateful Stream Processing

Many stream operations need **state** — remembering information across events.

```
┌──────────────────────────────────────────────────────────────┐
│              STATEFUL vs STATELESS OPERATIONS                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  STATELESS (no memory needed):                               │
│  • Filter: remove events where amount < 100                  │
│  • Map: convert USD to INR                                   │
│  • FlatMap: split one event into multiple                    │
│                                                               │
│  STATEFUL (needs memory):                                    │
│  • Count: "How many orders this hour?"                       │
│  • Sum: "Total revenue today"                                │
│  • Join: "Match click events with purchase events"           │
│  • Dedup: "Remove duplicate events"                          │
│  • Pattern: "Alert if 3 failed logins in 5 minutes"         │
│                                                               │
│  State Storage:                                               │
│  ┌────────────────────────────────────────────────────┐      │
│  │  Kafka Streams → RocksDB (embedded, local)         │      │
│  │  Flink → RocksDB or Heap (checkpointed to S3/HDFS)│      │
│  │  Spark → In-memory (checkpointed to HDFS)          │      │
│  └────────────────────────────────────────────────────┘      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Step 8: Event Time vs Processing Time

```
┌──────────────────────────────────────────────────────────────────┐
│              EVENT TIME vs PROCESSING TIME                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Event Time = When the event actually happened (in the source)   │
│  Processing Time = When the event is processed by the system     │
│                                                                   │
│  Why does it matter?                                             │
│                                                                   │
│  Mobile app user offline → events queued → sent later:           │
│                                                                   │
│  Event:    [A:9:00] [B:9:01] [C:9:02] [D:9:03]                 │
│  Arrives:  [A:9:00] [B:9:01] [D:9:03] [C:10:00] ← C arrived   │
│                                              late (network issue) │
│                                                                   │
│  If using Processing Time:                                       │
│    Window [9:00-9:05] = {A, B, D}     ← C missed! Wrong count! │
│    Window [10:00-10:05] = {C}          ← C counted in wrong hr  │
│                                                                   │
│  If using Event Time:                                            │
│    Window [9:00-9:05] = {A, B, C, D}  ← Correct! (waited for C)│
│                                                                   │
│  Handling Late Data:                                             │
│  • Watermarks: "I believe all events up to time T have arrived" │
│  • Allowed Lateness: "Wait extra 5 minutes for stragglers"      │
│  • Side Output: "Send super-late events to a special stream"    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Step 9: Exactly-Once Processing

The hardest challenge in stream processing: **processing each event exactly once**.

```
┌──────────────────────────────────────────────────────────────────┐
│              DELIVERY GUARANTEES IN STREAM PROCESSING              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  At-Most-Once:  Fire and forget. May lose events.               │
│                  (Fastest, least reliable)                        │
│                                                                   │
│  At-Least-Once: Retry on failure. May process duplicates.       │
│                  (Need idempotent operations)                     │
│                                                                   │
│  Exactly-Once:  Each event affects state exactly once.           │
│                  (Hardest to achieve!)                            │
│                                                                   │
│  How Flink achieves Exactly-Once:                                │
│  ─────────────────────────────────                               │
│  1. Distributed Snapshots (Chandy-Lamport algorithm)            │
│  2. Aligned Checkpoints:                                         │
│                                                                   │
│     Source ──▶ [Op1] ──▶ [Op2] ──▶ [Op3] ──▶ Sink             │
│                  │          │          │                          │
│                  ▼          ▼          ▼                          │
│             [State1]   [State2]   [State3]                       │
│                  │          │          │                          │
│                  └──────────┴──────────┘                         │
│                             │                                     │
│                    Checkpoint Barrier                             │
│                    (snapshot all states atomically)               │
│                             │                                     │
│                             ▼                                     │
│                    ┌─────────────────┐                           │
│                    │  S3 / HDFS      │                           │
│                    │  (durable       │                           │
│                    │   checkpoint)   │                           │
│                    └─────────────────┘                           │
│                                                                   │
│  On failure → restore from last checkpoint → replay from Kafka  │
│  Result: Each event counted exactly once in the output          │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Step 10: Kafka Streams vs Apache Flink — When to Use What

```
┌────────────────────────────────────────────────────────────────────┐
│         KAFKA STREAMS vs APACHE FLINK COMPARISON                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Feature            │ Kafka Streams         │ Apache Flink          │
│  ═══════════════════│═══════════════════════│═══════════════════════│
│  Deployment         │ Library (in your app) │ Separate cluster      │
│  Infrastructure     │ Just Kafka needed     │ Flink + ZooKeeper     │
│  Complexity         │ Simple                │ More complex          │
│  Source/Sink        │ Kafka only            │ Kafka, HDFS, DBs, etc │
│  SQL Support        │ KSQL (separate)       │ Flink SQL (built-in)  │
│  Event Time         │ Yes                   │ Yes (more advanced)   │
│  Exactly-Once       │ Yes                   │ Yes                   │
│  Batch + Stream     │ Stream only           │ Unified batch+stream  │
│  State Size         │ Limited by disk       │ Huge (TB-scale)       │
│  Late Data          │ Basic                 │ Advanced (side output)│
│  Savepoints         │ No                    │ Yes (pause/resume)    │
│  Best For           │ Simple transformations│ Complex event process │
│                     │ Kafka-centric apps    │ Multi-source ETL      │
│                     │ Microservices         │ Large-scale analytics │
│                                                                     │
│  Decision Guide:                                                    │
│  • Already using Kafka + simple processing → Kafka Streams         │
│  • Need multi-source, complex CEP, SQL → Flink                     │
│  • Want minimal ops overhead → Kafka Streams                        │
│  • Need batch + streaming unified → Flink                           │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Examples

### How LinkedIn Uses Kafka Streams

```
┌──────────────────────────────────────────────────────────────────┐
│              LINKEDIN — REAL-TIME ACTIVITY TRACKING                │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1 TRILLION events/day processed through Kafka                   │
│                                                                   │
│  ┌──────────┐     ┌──────────┐     ┌──────────────────┐        │
│  │ User     │     │  Kafka   │     │  Kafka Streams   │        │
│  │ Activity │────▶│  Topics  │────▶│  Applications    │        │
│  │          │     │          │     │                    │        │
│  │ • Views  │     │ • page-  │     │ • "Who viewed    │        │
│  │ • Clicks │     │   views  │     │    your profile" │        │
│  │ • Shares │     │ • clicks │     │ • Feed ranking   │        │
│  │ • Msgs   │     │ • shares │     │ • Recommendations│        │
│  └──────────┘     └──────────┘     │ • Abuse detection│        │
│                                     └──────────────────┘        │
│                                                                   │
│  Key patterns:                                                    │
│  • Standardized event schema (Avro)                              │
│  • 7-day retention in Kafka                                      │
│  • Stream-table joins for enrichment                             │
│  • Tiered processing (real-time → near-real-time → batch)       │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### How Uber Uses Apache Flink

```
┌──────────────────────────────────────────────────────────────────┐
│              UBER — REAL-TIME SURGE PRICING                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Problem: Calculate supply/demand ratio per area every 30 sec    │
│                                                                   │
│  ┌───────────┐                                                   │
│  │ Driver    │──"DriverLocationUpdated"──┐                       │
│  │ Locations │                            │                       │
│  └───────────┘                            ▼                       │
│                                    ┌─────────────┐               │
│  ┌───────────┐                     │ Apache Flink │              │
│  │ Rider     │──"RideRequested"───▶│             │──▶ Surge     │
│  │ Requests  │                     │ • Geo-hash   │    Multiplier│
│  └───────────┘                     │   grouping   │              │
│                                    │ • 30s tumbling│              │
│  ┌───────────┐                     │   windows    │              │
│  │ Completed │──"RideCompleted"───▶│ • Supply/    │              │
│  │ Rides     │                     │   Demand calc│              │
│  └───────────┘                     └─────────────┘               │
│                                                                   │
│  Flink Job:                                                       │
│  1. Key by geohash (geographic area)                             │
│  2. Tumbling window of 30 seconds                                │
│  3. Count drivers (supply) and requests (demand)                 │
│  4. If demand/supply > threshold → surge pricing                 │
│  5. Emit surge multiplier to pricing service                     │
│                                                                   │
│  Scale: 1M+ events/second, sub-second latency                   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### How Netflix Uses Flink for Real-Time Personalization

```
┌──────────────────────────────────────────────────────────────────┐
│              NETFLIX — REAL-TIME CONTENT RECOMMENDATIONS           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────┐    ┌────────┐    ┌──────────┐    ┌────────────┐   │
│  │ User    │    │ Kafka  │    │  Flink   │    │ Feature    │   │
│  │ Actions │───▶│ Topics │───▶│ Pipeline │───▶│ Store      │   │
│  │         │    │        │    │          │    │ (Redis)    │   │
│  │• Play   │    │        │    │• Session │    │            │   │
│  │• Pause  │    │        │    │  features│    │ • Last 5   │   │
│  │• Skip   │    │        │    │• Real-   │    │   shows    │   │
│  │• Browse │    │        │    │  time    │    │ • Watch    │   │
│  │• Search │    │        │    │  user    │    │   patterns │   │
│  └─────────┘    └────────┘    │  profile │    │ • Interests│   │
│                                └──────────┘    └─────┬──────┘   │
│                                                       │          │
│                                                       ▼          │
│                                              ┌──────────────┐    │
│                                              │ Recommendation│   │
│                                              │ Model (ML)    │   │
│                                              │               │   │
│                                              │ "Because you  │   │
│                                              │  watched X,   │   │
│                                              │  try Y"       │   │
│                                              └──────────────┘    │
│                                                                   │
│  Result: Recommendations update within seconds of user activity  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Other Stream Processing Frameworks

```
┌──────────────────────────────────────────────────────────────────┐
│              STREAM PROCESSING LANDSCAPE                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Framework            │ Model          │ Best For                 │
│  ═════════════════════│════════════════│═══════════════════════   │
│  Apache Flink         │ True streaming │ Complex, large-scale    │
│  Kafka Streams        │ True streaming │ Kafka-centric, simple   │
│  Apache Spark Stream. │ Micro-batch    │ Batch + stream unified  │
│  AWS Kinesis Data An. │ True streaming │ AWS-native workloads    │
│  Google Dataflow      │ True streaming │ GCP, Apache Beam        │
│  Apache Storm         │ True streaming │ Legacy (replaced by     │
│                       │                │  Flink in most cases)    │
│  Apache Pulsar Func.  │ True streaming │ Pulsar-centric          │
│  Redpanda             │ True streaming │ Kafka-compatible, C++   │
│  RisingWave           │ True streaming │ Streaming database      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Common Stream Processing Patterns

```
┌──────────────────────────────────────────────────────────────────┐
│              COMMON PATTERNS                                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. EVENT ENRICHMENT                                             │
│     Raw event + lookup data → enriched event                     │
│     "Order event" + "customer DB" → "Order with customer info"  │
│                                                                   │
│  2. STREAM-TABLE JOIN                                            │
│     Join streaming events with a slowly-changing reference table │
│     "Transaction stream" JOIN "exchange rates table"             │
│                                                                   │
│  3. COMPLEX EVENT PROCESSING (CEP)                               │
│     Detect patterns across multiple events                       │
│     "3 failed logins → account lock → different IP login"       │
│     → "Account compromised alert"                                │
│                                                                   │
│  4. DEDUPLICATION                                                │
│     Remove duplicate events (common with at-least-once delivery) │
│     Keep event IDs in state, drop if already seen               │
│                                                                   │
│  5. DYNAMIC ROUTING                                              │
│     Route events to different outputs based on content           │
│     High-priority → fast lane, Low-priority → batch lane        │
│                                                                   │
│  6. SESSIONIZATION                                               │
│     Group events into user sessions based on activity gaps       │
│     Web analytics: "pages viewed in one visit"                   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Production Considerations

```
┌──────────────────────────────────────────────────────────────────┐
│              PRODUCTION BEST PRACTICES                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. BACKPRESSURE HANDLING                                        │
│     • What: Consumer slower than producer                        │
│     • Flink: Automatic backpressure (TCP-based)                 │
│     • Kafka Streams: Consumer lag → auto-scales                 │
│     • Monitor: Consumer lag metric is critical!                  │
│                                                                   │
│  2. SCHEMA EVOLUTION                                             │
│     • Use Avro/Protobuf (not JSON) for events                   │
│     • Schema Registry for compatibility checks                   │
│     • Forward/backward compatible changes only                   │
│                                                                   │
│  3. STATE MANAGEMENT                                             │
│     • Keep state small (aggregate, don't accumulate)            │
│     • Set TTLs on state entries                                  │
│     • Monitor state size (can cause OOM)                        │
│                                                                   │
│  4. MONITORING                                                   │
│     • Consumer lag (how far behind?)                             │
│     • Throughput (events/second)                                 │
│     • Latency (event time → processing time gap)               │
│     • Checkpoint duration and size                               │
│     • Restart rate (are jobs crashing?)                          │
│                                                                   │
│  5. TESTING                                                       │
│     • TopologyTestDriver (Kafka Streams)                        │
│     • MiniCluster (Flink)                                        │
│     • Integration tests with embedded Kafka                      │
│     • Chaos testing: kill nodes, inject lag                      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

| Concept | Summary |
|---------|---------|
| Stream Processing | Analyzing data as it flows, not after storage |
| Kafka Streams | Library for stream processing within your Java/Python app |
| Apache Flink | Distributed cluster for complex, large-scale stream processing |
| Windowing | Grouping infinite streams into finite time-based chunks |
| Event Time | Processing based on when events happened, not when they arrived |
| Exactly-Once | Achieved via checkpointing and distributed snapshots |
| Stateful Processing | Operations that need memory (counts, joins, patterns) |
| Backpressure | System slowing producers when consumers can't keep up |

---

## What's Next?

You now understand how to process real-time data streams at scale. Next, we'll explore **Resilience & Fault Tolerance** — how to build systems that handle failures gracefully without losing data.

---

> **"The world is a stream of events. The question is not whether to do stream processing — it's whether you can afford to wait for batch."** — Jay Kreps, creator of Apache Kafka
