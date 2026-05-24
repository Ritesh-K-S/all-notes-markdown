# Chapter 55 — Dataflow

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Dataflow Fundamentals](#part-1--dataflow-fundamentals)
- [Part 2: Apache Beam Model](#part-2--apache-beam-model)
- [Part 3: PCollections & Transforms](#part-3--pcollections--transforms)
- [Part 4: ParDo & DoFn](#part-4--pardo--dofn)
- [Part 5: Windowing](#part-5--windowing)
- [Part 6: Triggers & Accumulation](#part-6--triggers--accumulation)
- [Part 7: Side Inputs & Outputs](#part-7--side-inputs--outputs)
- [Part 8: I/O Connectors](#part-8--io-connectors)
- [Part 9: Streaming Pipelines](#part-9--streaming-pipelines)
- [Part 10: Batch Pipelines](#part-10--batch-pipelines)
- [Part 11: Dataflow Templates](#part-11--dataflow-templates)
- [Part 12: Flex Templates](#part-12--flex-templates)
- [Part 13: Monitoring & Troubleshooting](#part-13--monitoring--troubleshooting)
- [Part 14: Console Walkthrough — Dataflow Job Creation](#part-14-console-walkthrough--dataflow-job-creation)
- [Part 15: Terraform & gcloud CLI Reference](#part-15--terraform--gcloud-cli-reference)
- [Part 16: Real-World Patterns](#part-16--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Dataflow is Google Cloud's fully managed service for stream and batch data processing, built on the open-source Apache Beam SDK. It provides auto-scaling, dynamic work rebalancing, and exactly-once processing semantics. You write a pipeline once in Apache Beam (Python, Java, or Go) and run it on Dataflow — the same code works for both batch and streaming without modification.

---

## Part 1 — Dataflow Fundamentals

### Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│         DATAFLOW ARCHITECTURE                                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  You write: Apache Beam Pipeline (SDK)                              │
│  Dataflow provides: Execution engine (Runner)                      │
│                                                                      │
│  ┌────────────────────────────────────────────────────────┐        │
│  │  Apache Beam SDK (Python / Java / Go)                   │        │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐          │        │
│  │  │ Source    │──►│ Transform│──►│ Sink     │          │        │
│  │  │ (Read)   │   │ (Process)│   │ (Write)  │          │        │
│  │  └──────────┘   └──────────┘   └──────────┘          │        │
│  └────────────────────────┬───────────────────────────────┘        │
│                           │ submit pipeline                         │
│                           ▼                                         │
│  ┌────────────────────────────────────────────────────────┐        │
│  │  Dataflow Service (Runner)                              │        │
│  │                                                          │        │
│  │  • Optimizes execution graph (fusion, combining)       │        │
│  │  • Provisions worker VMs automatically                 │        │
│  │  • Auto-scales workers up/down                         │        │
│  │  • Dynamic work rebalancing (liquid sharding)          │        │
│  │  • Manages checkpointing (streaming)                   │        │
│  │  • Handles retries and fault tolerance                 │        │
│  │                                                          │        │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │        │
│  │  │Worker 1│ │Worker 2│ │Worker 3│ │Worker N│        │        │
│  │  │ (VM)   │ │ (VM)   │ │ (VM)   │ │ (VM)   │        │        │
│  │  └────────┘ └────────┘ └────────┘ └────────┘        │        │
│  └────────────────────────────────────────────────────────┘        │
│                                                                      │
│  Processing modes:                                                  │
│  • Batch: bounded data (files), runs to completion                │
│  • Streaming: unbounded data (Pub/Sub), runs continuously         │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Dataflow | AWS Kinesis Data Analytics | Azure Stream Analytics |
|---------|-------------|---------------------------|----------------------|
| Engine | Apache Beam | Apache Flink | Proprietary |
| Open source | Yes (Beam SDK) | Yes (Flink) | No |
| Languages | Python, Java, Go | Java, SQL | SQL, C# |
| Batch + Stream | Same code | Separate services | Separate |
| Auto-scaling | Yes (automatic) | Yes (Flink auto) | Yes |
| Serverless | Yes | Managed Flink | Yes |
| Exactly-once | Yes | Yes | At-least-once |

### Pricing

| Component | Cost |
|-----------|------|
| Worker vCPU | $0.056/vCPU-hr (batch), $0.069/vCPU-hr (streaming) |
| Worker memory | $0.003557/GB-hr (batch), $0.0035/GB-hr (streaming) |
| Persistent disk | $0.000054/GB-hr (HDD), $0.000298/GB-hr (SSD) |
| Dataflow Shuffle | $0.011/GB shuffled |
| Streaming Engine | $0.018/hr per unit |
| Dataflow Prime | vCPU + memory + shuffle (unified pricing) |

---

## Part 2 — Apache Beam Model

### Core Concepts

```
┌────────────────────────────────────────────────────────────────────┐
│         APACHE BEAM MODEL                                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Pipeline: the entire data processing job                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  PCollection ──► Transform ──► PCollection ──► Transform  │     │
│  │  (input)         (ParDo)       (intermediate)   (GroupBy) │     │
│  │                                      │                     │     │
│  │                                      ▼                     │     │
│  │                                PCollection ──► Transform  │     │
│  │                                (grouped)       (Write)    │     │
│  │                                                    │       │     │
│  │                                                    ▼       │     │
│  │                                              Output       │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  PCollection:                                                       │
│  • Distributed dataset (immutable)                                 │
│  • Bounded (batch) or unbounded (streaming)                       │
│  • Elements can be any serializable type                          │
│                                                                      │
│  Transform:                                                         │
│  • ParDo — element-wise processing (like map)                     │
│  • GroupByKey — group by key                                      │
│  • CoGroupByKey — join multiple PCollections                      │
│  • Combine — aggregation (sum, count, avg)                        │
│  • Flatten — merge PCollections                                   │
│  • Partition — split PCollection                                  │
│  • Window — assign windows for streaming                          │
│                                                                      │
│  Runner: executes the pipeline                                     │
│  • DirectRunner — local testing                                   │
│  • DataflowRunner — Google Cloud Dataflow                        │
│  • FlinkRunner, SparkRunner — other runners                      │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 3 — PCollections & Transforms

### Basic Pipeline (Python)

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

options = PipelineOptions([
    '--runner=DataflowRunner',
    '--project=my-project',
    '--region=us-central1',
    '--temp_location=gs://my-bucket/temp/',
    '--staging_location=gs://my-bucket/staging/',
])

with beam.Pipeline(options=options) as p:
    (
        p
        | 'Read' >> beam.io.ReadFromText('gs://my-bucket/input/*.csv')
        | 'Parse' >> beam.Map(lambda line: line.split(','))
        | 'Filter' >> beam.Filter(lambda fields: float(fields[3]) > 100)
        | 'Format' >> beam.Map(lambda fields: {
            'user_id': fields[0],
            'product': fields[1],
            'amount': float(fields[3]),
        })
        | 'Write' >> beam.io.WriteToBigQuery(
            'my-project:analytics.purchases',
            schema='user_id:STRING,product:STRING,amount:FLOAT',
            write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
        )
    )
```

### Common Transforms

```python
# Map — transform each element
| 'ToUpper' >> beam.Map(lambda x: x.upper())

# FlatMap — one-to-many transform
| 'SplitWords' >> beam.FlatMap(lambda line: line.split())

# Filter — keep matching elements
| 'OnlyErrors' >> beam.Filter(lambda log: log['level'] == 'ERROR')

# GroupByKey — group by key
| 'PairWithKey' >> beam.Map(lambda x: (x['region'], x))
| 'GroupByRegion' >> beam.GroupByKey()

# CombinePerKey — aggregate per key
| 'SumByRegion' >> beam.CombinePerKey(sum)

# Combine globally
| 'TotalCount' >> beam.combiners.Count.Globally()

# Flatten — merge multiple PCollections
(pcoll1, pcoll2, pcoll3) | beam.Flatten()

# Partition — split into multiple PCollections
def partition_fn(element, num_partitions):
    return 0 if element['priority'] == 'high' else 1

high, low = (
    pcoll | beam.Partition(partition_fn, 2)
)
```

---

## Part 4 — ParDo & DoFn

### Custom Processing Logic

```python
# DoFn — the processing function for ParDo
class EnrichEventFn(beam.DoFn):
    def setup(self):
        """Called once per worker at startup."""
        import redis
        self.cache = redis.Redis(host='10.0.0.5')

    def process(self, element):
        """Called for each element."""
        user_id = element['user_id']

        # Lookup user info from cache
        user_info = self.cache.get(f"user:{user_id}")
        if user_info:
            element['user_name'] = user_info.decode()

        element['processed_at'] = datetime.utcnow().isoformat()
        yield element

    def teardown(self):
        """Called once per worker at shutdown."""
        self.cache.close()

# Use in pipeline
| 'Enrich' >> beam.ParDo(EnrichEventFn())

# DoFn with multiple outputs (tagged)
class ClassifyFn(beam.DoFn):
    def process(self, element):
        if element['amount'] > 1000:
            yield beam.pvalue.TaggedOutput('high_value', element)
        else:
            yield beam.pvalue.TaggedOutput('normal', element)

results = (
    pcoll | beam.ParDo(ClassifyFn()).with_outputs('high_value', 'normal')
)
high_value = results.high_value
normal = results.normal
```

---

## Part 5 — Windowing

### Window Types

```
┌────────────────────────────────────────────────────────────────────┐
│         WINDOWING (for streaming)                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Fixed Windows (tumbling):                                      │
│  ┌──────┐┌──────┐┌──────┐┌──────┐                                │
│  │ 0-5m ││ 5-10m││10-15m││15-20m│   non-overlapping             │
│  └──────┘└──────┘└──────┘└──────┘                                │
│                                                                      │
│  2. Sliding Windows:                                                │
│  ┌──────────┐                                                      │
│  │   0-10m  │                                                      │
│  └──────────┘                                                      │
│     ┌──────────┐                                                   │
│     │  5-15m   │          overlapping (period = 5m, size = 10m)   │
│     └──────────┘                                                   │
│        ┌──────────┐                                                │
│        │ 10-20m   │                                                │
│        └──────────┘                                                │
│                                                                      │
│  3. Session Windows:                                                │
│  ┌────────┐     ┌──────────────┐  ┌───┐                          │
│  │ events │ gap │   events     │  │evt│  gap-based per key       │
│  └────────┘     └──────────────┘  └───┘                          │
│                                                                      │
│  4. Global Window (default for batch):                             │
│  ┌──────────────────────────────────────────────┐                 │
│  │ all elements in one window                    │                 │
│  └──────────────────────────────────────────────┘                 │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```python
from apache_beam import window

# Fixed windows — 5 minutes
| 'Window5Min' >> beam.WindowInto(window.FixedWindows(5 * 60))

# Sliding windows — 10 min window, 5 min period
| 'SlidingWindow' >> beam.WindowInto(window.SlidingWindows(10 * 60, 5 * 60))

# Session windows — 30 min gap
| 'SessionWindow' >> beam.WindowInto(window.Sessions(30 * 60))

# With allowed lateness
| 'WindowWithLateness' >> beam.WindowInto(
    window.FixedWindows(5 * 60),
    allowed_lateness=beam.utils.timestamp.Duration(seconds=3600),
)
```

---

## Part 6 — Triggers & Accumulation

### Trigger Types

```python
from apache_beam.transforms.trigger import (
    AfterWatermark, AfterProcessingTime, AfterCount,
    Repeatedly, AccumulationMode
)

# Default trigger: fire once when watermark passes end of window
| beam.WindowInto(
    window.FixedWindows(60),
    trigger=AfterWatermark(),
    accumulation_mode=AccumulationMode.DISCARDING,
)

# Fire early results every 30 seconds, final at watermark
| beam.WindowInto(
    window.FixedWindows(60),
    trigger=AfterWatermark(
        early=AfterProcessingTime(30),    # early speculative results
        late=AfterCount(1),               # fire for each late element
    ),
    accumulation_mode=AccumulationMode.ACCUMULATING,
    allowed_lateness=3600,
)

# Repeatedly fire every 100 elements
| beam.WindowInto(
    window.FixedWindows(60),
    trigger=Repeatedly(AfterCount(100)),
    accumulation_mode=AccumulationMode.DISCARDING,
)
```

| Accumulation Mode | Behavior |
|-------------------|----------|
| DISCARDING | Each pane contains only new elements since last fire |
| ACCUMULATING | Each pane contains ALL elements in window so far |

---

## Part 7 — Side Inputs & Outputs

### Side Inputs

```python
# Side input: small dataset available to all workers
exchange_rates = (
    p | 'ReadRates' >> beam.io.ReadFromText('gs://bucket/rates.json')
      | 'ParseRates' >> beam.Map(json.loads)
)

# Use as side input in ParDo
class ConvertCurrencyFn(beam.DoFn):
    def process(self, element, rates):
        rate = rates.get(element['currency'], 1.0)
        element['amount_usd'] = element['amount'] * rate
        yield element

| 'Convert' >> beam.ParDo(
    ConvertCurrencyFn(),
    rates=beam.pvalue.AsDict(exchange_rates)
)

# Side input types:
# AsSingleton — single value
# AsList — list of all elements
# AsDict — dictionary (key-value pairs)
# AsIter — iterable
```

---

## Part 8 — I/O Connectors

### Built-in Connectors

| Source/Sink | Read | Write |
|-------------|------|-------|
| Text files | `ReadFromText` | `WriteToText` |
| BigQuery | `ReadFromBigQuery` | `WriteToBigQuery` |
| Pub/Sub | `ReadFromPubSub` | `WriteToPubSub` |
| Avro | `ReadFromAvro` | `WriteToAvro` |
| Parquet | `ReadFromParquet` | `WriteToParquet` |
| TFRecord | `ReadFromTFRecord` | `WriteToTFRecord` |
| Bigtable | `ReadFromBigtable` | `WriteToBigtable` |
| Kafka | `ReadFromKafka` | `WriteToKafka` |
| JDBC | `ReadFromJdbc` | `WriteToJdbc` |

```python
# Read from Pub/Sub (streaming)
| beam.io.ReadFromPubSub(
    topic='projects/my-project/topics/events',
    # or subscription='projects/my-project/subscriptions/my-sub',
    with_attributes=True,
)

# Write to BigQuery (streaming)
| beam.io.WriteToBigQuery(
    'project:dataset.table',
    schema='user_id:STRING,event:STRING,ts:TIMESTAMP',
    write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
    create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED,
    method=beam.io.WriteToBigQuery.Method.STREAMING_INSERTS,
)
```

---

## Part 9 — Streaming Pipelines

### Pub/Sub → Dataflow → BigQuery

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions, StandardOptions
import json

options = PipelineOptions([
    '--runner=DataflowRunner',
    '--project=my-project',
    '--region=us-central1',
    '--temp_location=gs://my-bucket/temp/',
    '--streaming',                     # enable streaming mode
    '--enable_streaming_engine',       # use Streaming Engine
    '--autoscaling_algorithm=THROUGHPUT_BASED',
    '--max_num_workers=10',
])

with beam.Pipeline(options=options) as p:
    (
        p
        | 'ReadPubSub' >> beam.io.ReadFromPubSub(
            subscription='projects/my-project/subscriptions/events-sub'
        )
        | 'ParseJSON' >> beam.Map(json.loads)
        | 'Window' >> beam.WindowInto(beam.window.FixedWindows(60))
        | 'AddTimestamp' >> beam.Map(lambda e: beam.window.TimestampedValue(
            e, e['timestamp']
        ))
        | 'WriteBQ' >> beam.io.WriteToBigQuery(
            'my-project:analytics.events',
            schema='user_id:STRING,event_type:STRING,timestamp:TIMESTAMP,data:JSON',
            write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
            method=beam.io.WriteToBigQuery.Method.STREAMING_INSERTS,
        )
    )
```

---

## Part 10 — Batch Pipelines

### GCS → Transform → BigQuery

```python
import apache_beam as beam

options = PipelineOptions([
    '--runner=DataflowRunner',
    '--project=my-project',
    '--region=us-central1',
    '--temp_location=gs://my-bucket/temp/',
    '--num_workers=5',
    '--machine_type=n1-standard-4',
])

with beam.Pipeline(options=options) as p:
    (
        p
        | 'ReadCSV' >> beam.io.ReadFromText(
            'gs://data-bucket/sales/2024-*.csv',
            skip_header_lines=1
        )
        | 'Parse' >> beam.Map(lambda line: dict(
            zip(['order_id', 'product', 'quantity', 'price', 'date'],
                line.split(','))
        ))
        | 'CalcTotal' >> beam.Map(lambda r: {
            **r,
            'total': float(r['quantity']) * float(r['price'])
        })
        | 'FilterBig' >> beam.Filter(lambda r: r['total'] > 50)
        | 'WriteBQ' >> beam.io.WriteToBigQuery(
            'my-project:sales.orders',
            schema='order_id:STRING,product:STRING,quantity:INTEGER,'
                   'price:FLOAT,date:DATE,total:FLOAT',
            write_disposition=beam.io.BigQueryDisposition.WRITE_TRUNCATE,
        )
    )
```

---

## Part 11 — Dataflow Templates

### Google-Provided Templates

```bash
# Pub/Sub to BigQuery (streaming)
gcloud dataflow jobs run pubsub-to-bq \
    --gcs-location=gs://dataflow-templates/latest/PubSub_to_BigQuery \
    --region=us-central1 \
    --parameters \
inputTopic=projects/my-project/topics/events,\
outputTableSpec=my-project:analytics.events,\
outputDeadletterTable=my-project:analytics.events_deadletter

# GCS Text to BigQuery (batch)
gcloud dataflow jobs run gcs-to-bq \
    --gcs-location=gs://dataflow-templates/latest/GCS_Text_to_BigQuery \
    --region=us-central1 \
    --parameters \
inputFilePattern=gs://data-bucket/input/*.csv,\
JSONPath=gs://data-bucket/schema.json,\
outputTable=my-project:dataset.table,\
bigQueryLoadingTemporaryDirectory=gs://my-bucket/temp/

# BigQuery to GCS (Parquet)
gcloud dataflow jobs run bq-export \
    --gcs-location=gs://dataflow-templates/latest/Cloud_BigQuery_to_GCS_Parquet \
    --region=us-central1 \
    --parameters \
tableRef=my-project:dataset.table,\
bucket=gs://export-bucket/output/
```

---

## Part 12 — Flex Templates

### Custom Templates

```bash
# Build Flex Template
gcloud dataflow flex-template build \
    gs://my-bucket/templates/my-pipeline.json \
    --image-gcr-path=gcr.io/my-project/my-pipeline:latest \
    --sdk-language=PYTHON \
    --flex-template-base-image=PYTHON3 \
    --metadata-file=metadata.json \
    --py-path=.

# Run Flex Template
gcloud dataflow flex-template run my-job \
    --template-file-gcs-location=gs://my-bucket/templates/my-pipeline.json \
    --region=us-central1 \
    --parameters input_topic=projects/my-project/topics/events \
    --parameters output_table=my-project:analytics.events
```

```json
// metadata.json
{
  "name": "My Pipeline",
  "description": "Processes events from Pub/Sub to BigQuery",
  "parameters": [
    {
      "name": "input_topic",
      "label": "Pub/Sub input topic",
      "helpText": "Full topic path",
      "isOptional": false
    },
    {
      "name": "output_table",
      "label": "BigQuery output table",
      "helpText": "project:dataset.table",
      "isOptional": false
    }
  ]
}
```

---

## Part 13 — Monitoring & Troubleshooting

### Key Metrics

```
┌────────────────────────────────────────────────────────────────────┐
│         MONITORING                                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Dataflow console shows:                                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Job Graph: visual DAG of pipeline steps                  │     │
│  │  Step metrics: elements in/out, time, errors              │     │
│  │  Worker metrics: CPU, memory, disk                        │     │
│  │  Autoscaling: current/target workers                      │     │
│  │  Watermark: how far behind real-time (streaming)          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Key metrics to watch:                                              │
│  • system_lag: delay between event time and processing            │
│  • data_watermark: oldest unprocessed event timestamp             │
│  • elements_produced: throughput per step                         │
│  • wall_time: time spent in each step                             │
│  • autoscaling/target_workers: if maxed out, need more quota     │
│                                                                      │
│  Common issues:                                                     │
│  • Hot keys → uneven distribution (one worker overloaded)        │
│  • Data skew → use Reshuffle or Combine before GroupByKey        │
│  • Memory errors → increase machine type or reduce batch size    │
│  • Watermark stuck → late data or unacked Pub/Sub messages       │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# List jobs
gcloud dataflow jobs list --region=us-central1

# Describe job
gcloud dataflow jobs describe JOB_ID --region=us-central1

# Cancel job (batch)
gcloud dataflow jobs cancel JOB_ID --region=us-central1

# Drain job (streaming — finish in-flight, stop new input)
gcloud dataflow jobs drain JOB_ID --region=us-central1

# Update streaming job (in-place pipeline update)
python pipeline.py --update --job_name=existing-job-name \
    --runner=DataflowRunner --region=us-central1
```

---

## Part 14: Console Walkthrough — Dataflow Job Creation

### Creating a Job from Template

```
Console → Dataflow → Jobs → CREATE JOB FROM TEMPLATE

┌─────────────────────────────────────────────────────────────────┐
│           CREATE JOB FROM TEMPLATE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Job name ──                                                   │
│ Job name: [gcs-to-bigquery-daily]                               │
│                                                                   │
│ ── Regional endpoint ──                                         │
│ Region: [us-central1 ▼]                                         │
│                                                                   │
│ ── Dataflow template ──                                         │
│ Template:                                                       │
│ ● Google-provided templates                                    │
│   Category: [Process Data in Bulk (Batch) ▼]                  │
│   Template: [Cloud Storage Text to BigQuery ▼]                │
│                                                                   │
│ ○ Custom template                                              │
│   GCS path: [gs://my-bucket/templates/my-template]            │
│                                                                   │
│ ── Required parameters ──                                       │
│ (parameters vary by template)                                  │
│                                                                   │
│ Input file pattern:                                             │
│   [gs://my-data-bucket/input/*.csv]                            │
│                                                                   │
│ JSON schema path:                                               │
│   [gs://my-data-bucket/schema.json]                            │
│                                                                   │
│ Output BigQuery table:                                          │
│   [my-project:my_dataset.my_table]                             │
│                                                                   │
│ Temporary BigQuery directory:                                   │
│   [gs://my-data-bucket/temp/]                                  │
│                                                                   │
│ ── Optional parameters ──                                       │
│ [▸ Show optional parameters]                                   │
│                                                                   │
│ Temp location: [gs://my-data-bucket/dataflow-temp/]            │
│ Max workers: [10]                                               │
│ Machine type: [n2-standard-4 ▼]                                 │
│ Service account: [dataflow-sa@proj.iam.gserviceaccount.com]    │
│ Network: [default ▼]                                            │
│ Subnetwork: [regions/us-central1/subnetworks/default ▼]        │
│ ☑ Use private IPs (no public IP on workers)                    │
│ Encryption: [● Google-managed | ○ CMEK]                        │
│                                                                   │
│                           [RUN JOB]                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Monitoring a Running Job

```
Console → Dataflow → Jobs → Click job name

┌─────────────────────────────────────────────────────────────────┐
│           JOB DETAILS                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Status: Running ● | Elapsed: 00:12:34                           │
│                                                                   │
│ ── Job Graph tab ──                                             │
│ Visual DAG showing pipeline stages:                             │
│ [Read] → [Parse] → [Transform] → [Write to BQ]                │
│ ⚡ Click any stage to see:                                       │
│   ├── Elements processed                                       │
│   ├── Estimated size                                            │
│   ├── Wall time                                                 │
│   └── Errors (if any)                                           │
│                                                                   │
│ ── Tabs ──                                                       │
│ [Job Graph] [Execution details] [Job Metrics] [Job Logs]       │
│                                                                   │
│ ── Actions ──                                                    │
│ [STOP] → Cancel (batch) or Drain (streaming)                   │
│ ⚡ Drain: finish in-flight elements, stop accepting new input.  │
│ ⚡ Cancel: stop immediately (may lose in-flight data).          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 15 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Dataflow Job (Batch — Classic Template) ─────────────────
resource "google_dataflow_job" "batch_job" {
  name              = "gcs-to-bigquery"
  template_gcs_path = "gs://dataflow-templates/latest/GCS_Text_to_BigQuery"
  temp_gcs_location = "gs://my-bucket/temp/"
  region            = var.region
  project           = var.project_id

  parameters = {
    inputFilePattern                  = "gs://data-bucket/input/*.csv"
    JSONPath                          = "gs://data-bucket/schema.json"
    outputTable                       = "${var.project_id}:dataset.table"
    bigQueryLoadingTemporaryDirectory = "gs://my-bucket/bq-temp/"
  }

  machine_type = "n1-standard-4"
  max_workers  = 10

  on_delete = "cancel"
}

# ─── Dataflow Flex Template Job ───────────────────────────────
resource "google_dataflow_flex_template_job" "streaming" {
  name                    = "pubsub-to-bq-streaming"
  container_spec_gcs_path = "gs://my-bucket/templates/pipeline.json"
  region                  = var.region
  project                 = var.project_id

  parameters = {
    input_topic  = google_pubsub_topic.events.id
    output_table = "${var.project_id}:analytics.events"
  }

  machine_type             = "n1-standard-2"
  max_workers              = 20
  enable_streaming_engine  = true
  autoscaling_algorithm    = "THROUGHPUT_BASED"

  on_delete = "drain"
}

# ─── Service Account ─────────────────────────────────────────
resource "google_service_account" "dataflow" {
  account_id   = "dataflow-worker"
  display_name = "Dataflow Worker SA"
  project      = var.project_id
}

resource "google_project_iam_member" "dataflow_worker" {
  project = var.project_id
  role    = "roles/dataflow.worker"
  member  = "serviceAccount:${google_service_account.dataflow.email}"
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# JOBS
# ═══════════════════════════════════════════════════════════════
gcloud dataflow jobs list --region=R
gcloud dataflow jobs describe JOB_ID --region=R
gcloud dataflow jobs cancel JOB_ID --region=R
gcloud dataflow jobs drain JOB_ID --region=R
gcloud dataflow jobs show JOB_ID --region=R

# ═══════════════════════════════════════════════════════════════
# RUN CLASSIC TEMPLATE
# ═══════════════════════════════════════════════════════════════
gcloud dataflow jobs run JOB_NAME \
    --gcs-location=gs://dataflow-templates/latest/TEMPLATE \
    --region=R \
    --parameters KEY=VALUE,KEY=VALUE

# ═══════════════════════════════════════════════════════════════
# FLEX TEMPLATES
# ═══════════════════════════════════════════════════════════════
gcloud dataflow flex-template build gs://bucket/template.json \
    --image-gcr-path=gcr.io/project/image:tag \
    --sdk-language=PYTHON

gcloud dataflow flex-template run JOB_NAME \
    --template-file-gcs-location=gs://bucket/template.json \
    --region=R \
    --parameters KEY=VALUE

# ═══════════════════════════════════════════════════════════════
# SNAPSHOTS (streaming)
# ═══════════════════════════════════════════════════════════════
gcloud dataflow snapshots create --job-id=JOB_ID --region=R
gcloud dataflow snapshots list --region=R
gcloud dataflow snapshots describe SNAP_ID --region=R
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Real-Time Event Processing

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: REAL-TIME ANALYTICS                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────────────┐        │
│  │ Web/App  │──►│ Pub/Sub  │──►│ Dataflow (streaming)      │        │
│  │ Events   │   │ topic    │   │                            │        │
│  └──────────┘   └──────────┘   │ 1. Parse JSON             │        │
│                                 │ 2. Validate schema        │        │
│                                 │ 3. Enrich (user lookup)   │        │
│                                 │ 4. Window: 1-min fixed    │        │
│                                 │ 5. Aggregate per window   │        │
│                                 └──────┬──────────┬─────────┘        │
│                                        │          │                   │
│                                   ┌────▼────┐ ┌──▼──────────┐       │
│                                   │BigQuery │ │ Pub/Sub     │       │
│                                   │(raw +   │ │ (alerts for │       │
│                                   │ agg)    │ │  anomalies) │       │
│                                   └─────────┘ └─────────────┘       │
│                                                                        │
│  Dead-letter: failed records → separate BQ table for debugging     │
│  Streaming Engine: off-loads state to Dataflow service              │
│  Auto-scaling: based on Pub/Sub backlog                             │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: ETL Data Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: BATCH ETL                                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Nightly batch ETL pipeline:                                         │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Cloud Scheduler (cron: 0 2 * * *)                        │        │
│  │       │                                                    │        │
│  │       ▼ trigger                                           │        │
│  │  Cloud Function → launches Dataflow Flex Template        │        │
│  │       │                                                    │        │
│  │       ▼                                                    │        │
│  │  Dataflow batch job:                                      │        │
│  │  ├── Read from Cloud SQL (JDBC connector)                │        │
│  │  ├── Read from GCS (CSV/Parquet files)                   │        │
│  │  ├── Join and deduplicate                                │        │
│  │  ├── Transform (clean, normalize, derive columns)        │        │
│  │  ├── Write to BigQuery (partitioned table)               │        │
│  │  └── Write errors to GCS dead-letter bucket              │        │
│  │                                                            │        │
│  │  On completion → Pub/Sub notification                    │        │
│  │  → Triggers downstream dbt models in BigQuery            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Streaming CDC (Change Data Capture)

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: CHANGE DATA CAPTURE                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐        │
│  │ Cloud SQL   │  │ Debezium     │  │ Pub/Sub             │        │
│  │ (source DB) │─►│ (CDC agent)  │─►│ (change events)     │        │
│  └─────────────┘  └──────────────┘  └──────────┬──────────┘        │
│                                                  │                    │
│                                        ┌─────────▼──────────┐       │
│                                        │ Dataflow streaming  │       │
│                                        │ ├── Parse CDC event │       │
│                                        │ ├── Apply upsert    │       │
│                                        │ │   logic            │       │
│                                        │ ├── Handle deletes  │       │
│                                        │ └── Write to BQ     │       │
│                                        │     (MERGE/upsert)  │       │
│                                        └─────────────────────┘       │
│                                                                        │
│  Result: BigQuery table mirrors source DB in near real-time         │
│  Latency: seconds (not hours like batch ETL)                        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command / Code |
|--------|---------------|
| Run classic template | `gcloud dataflow jobs run NAME --gcs-location=gs://... --parameters K=V` |
| Run flex template | `gcloud dataflow flex-template run NAME --template-file-gcs-location=gs://...` |
| Build flex template | `gcloud dataflow flex-template build gs://... --image-gcr-path=gcr.io/...` |
| List jobs | `gcloud dataflow jobs list --region=R` |
| Cancel (batch) | `gcloud dataflow jobs cancel JOB_ID --region=R` |
| Drain (streaming) | `gcloud dataflow jobs drain JOB_ID --region=R` |
| Enable streaming engine | `--enable_streaming_engine` |
| Set max workers | `--max_num_workers=N` |
| Fixed window | `beam.WindowInto(window.FixedWindows(300))` |
| Read Pub/Sub | `beam.io.ReadFromPubSub(topic=T)` |
| Write BigQuery | `beam.io.WriteToBigQuery(table, schema=S)` |

---

## What is Data Processing? (Beginner Explanation)

If you're new to data processing, here's the simplest way to think about it:

```
┌────────────────────────────────────────────────────────────────────┐
│         DATA PROCESSING — TWO MODES                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  BATCH Processing:                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Like processing a pile of documents at night.            │     │
│  │                                                            │     │
│  │  You collect all the invoices from the day, put them in   │     │
│  │  a big stack, and process them all at once overnight.     │     │
│  │                                                            │     │
│  │  Example: "Every night at 2 AM, take all today's sales   │     │
│  │           data from files and load it into BigQuery."     │     │
│  │                                                            │     │
│  │  ✓ Has a start and an end (job completes)                │     │
│  │  ✓ Data is already sitting there, waiting to be processed│     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  STREAM Processing:                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Like processing mail as it arrives.                       │     │
│  │                                                            │     │
│  │  Instead of waiting for all mail to pile up, you open     │     │
│  │  and handle each letter the moment it hits your mailbox.  │     │
│  │                                                            │     │
│  │  Example: "As each user clicks on our website, process   │     │
│  │           the event immediately and update dashboards."   │     │
│  │                                                            │     │
│  │  ✓ Runs continuously (never "finishes")                  │     │
│  │  ✓ Data arrives in real time, processed within seconds   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### What is Apache Beam?

Apache Beam is an **open-source, portable data processing framework**. Think of it as a universal language for writing data pipelines:

- **You write your pipeline once** using the Beam SDK (Python, Java, or Go)
- **You choose where to run it** — Dataflow (Google Cloud), Apache Flink, Apache Spark, or even your laptop
- **The same code works for both batch and streaming** — no rewriting needed

```
  Your code (Apache Beam SDK)
       │
       ├──► Run on Dataflow  (Google Cloud — fully managed)
       ├──► Run on Flink     (open-source runner)
       ├──► Run on Spark     (open-source runner)
       └──► Run locally      (DirectRunner — for testing)
```

> **Analogy:** Apache Beam is like writing a recipe. Dataflow is one of the kitchens where you can cook that recipe. The recipe (Beam code) stays the same regardless of which kitchen (runner) you choose.

### When to Use Dataflow

| Use Dataflow When... | Don't Use Dataflow When... |
|----------------------|---------------------------|
| You need real-time streaming from Pub/Sub | You just need to run SQL queries (use BigQuery instead) |
| You want unified batch + streaming code | You already have Spark/Hadoop jobs (consider Dataproc) |
| You need exactly-once processing guarantees | You need a simple one-time file copy (use `gsutil` or Transfer Service) |
| You want fully managed auto-scaling with no clusters | You need interactive notebook exploration (use Dataproc + Jupyter) |
| You need complex event-time windowing and triggers | Your data fits in memory on a single machine |

---

## Console Walkthrough: Running Template Jobs

### Running a Google-Provided Template from Console

```
Console → Dataflow → Jobs → CREATE JOB FROM TEMPLATE

┌─────────────────────────────────────────────────────────────────┐
│           STEP-BY-STEP: RUN A TEMPLATE JOB                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Navigate                                                 │
│ ─────────────────                                                │
│ Console → search "Dataflow" → Click "Dataflow" → "Jobs"       │
│ → Click "CREATE JOB FROM TEMPLATE" button at the top            │
│                                                                   │
│ Step 2: Basic Configuration                                     │
│ ─────────────────────────                                        │
│ • Job name: [my-first-dataflow-job]                             │
│   (lowercase letters, numbers, hyphens only)                    │
│                                                                   │
│ • Regional endpoint: [us-central1 ▼]                            │
│   (choose the region closest to your data)                      │
│                                                                   │
│ Step 3: Select Template                                          │
│ ───────────────────                                              │
│ • Template type: ● Google-provided templates                   │
│ • Category: select one:                                         │
│   - "Process Data in Bulk (Batch)" — for file-based jobs       │
│   - "Process Data Continuously (Stream)" — for Pub/Sub jobs    │
│ • Template: e.g. "Cloud Storage Text to BigQuery"               │
│                                                                   │
│ Step 4: Fill Required Parameters                                 │
│ ────────────────────────────                                     │
│ (these change based on the template you chose)                  │
│ Example for "Cloud Storage Text to BigQuery":                   │
│ • Input file pattern: [gs://my-bucket/data/*.csv]               │
│ • JSON schema path:   [gs://my-bucket/schema.json]              │
│ • Output BQ table:    [project:dataset.table]                   │
│ • Temp directory:     [gs://my-bucket/temp/]                    │
│                                                                   │
│ Step 5: (Optional) Advanced Options                              │
│ ───────────────────────────────                                  │
│ Click "Show optional parameters" to set:                        │
│ • Max workers, machine type, service account                   │
│ • Network/subnetwork, encryption settings                       │
│                                                                   │
│ Step 6: Click [RUN JOB]                                         │
│ → Job starts provisioning workers                               │
│ → You're taken to the job details page                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Monitoring a Running Job from Console

```
Console → Dataflow → Jobs → Click on your job name

┌─────────────────────────────────────────────────────────────────┐
│           MONITORING A RUNNING JOB                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Top Bar ──                                                    │
│ Status: ● Running  |  Elapsed: 00:05:23  |  Type: Batch        │
│                                                                   │
│ ── Job Graph Tab (default view) ──                              │
│ Shows a visual DAG (directed acyclic graph) of your pipeline:  │
│                                                                   │
│   [ReadFromGCS] ──► [ParseCSV] ──► [Transform] ──► [WriteBQ]  │
│        ▲                                                        │
│   Click any box to see:                                         │
│   • Elements added (input count)                                │
│   • Elements produced (output count)                            │
│   • Estimated byte size                                         │
│   • Wall time (how long this step took)                        │
│   • Error count (if any)                                        │
│                                                                   │
│ ── Execution Details Tab ──                                     │
│ Shows worker-level details:                                     │
│ • Number of workers (current and target)                       │
│ • Worker CPU/memory usage                                       │
│ • Autoscaling events                                            │
│                                                                   │
│ ── Job Metrics Tab ──                                           │
│ Charts for throughput, latency, watermark (streaming)           │
│                                                                   │
│ ── Job Logs Tab ──                                              │
│ • Filter by severity: Info, Warning, Error                     │
│ • Shows worker logs, pipeline errors, and Dataflow messages    │
│ • Click "View in Logs Explorer" for advanced filtering         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Draining or Cancelling a Job from Console

```
Console → Dataflow → Jobs → Click on your running job

┌─────────────────────────────────────────────────────────────────┐
│           STOP A RUNNING JOB                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Click the [STOP] button at the top of the job details page.    │
│ You'll see two options:                                         │
│                                                                   │
│ ── Option 1: Cancel ──                                          │
│ • Stops the job immediately                                    │
│ • In-flight data may be lost                                   │
│ • Use for: batch jobs, or when you don't care about data loss  │
│ • Workers are released right away                              │
│                                                                   │
│ ── Option 2: Drain (streaming jobs only) ──                    │
│ • Stops accepting new input from sources (e.g., Pub/Sub)      │
│ • Finishes processing all in-flight elements                  │
│ • Writes all pending output to sinks (e.g., BigQuery)         │
│ • Then shuts down gracefully                                   │
│ • Use for: streaming jobs where you want zero data loss        │
│ • May take several minutes to complete                         │
│                                                                   │
│ ── After Stopping ──                                            │
│ Status changes to: "Cancelled" or "Drained"                   │
│ Workers are automatically released and billing stops.          │
│ You can still view the job graph, logs, and metrics.           │
│                                                                   │
│ 💡 Tip: You CANNOT restart a stopped job. You must create a    │
│    new job. For streaming, consider using "Update" instead of  │
│    drain/cancel to modify a running pipeline in-place.         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 56: Dataproc** → `56-dataproc.md`
