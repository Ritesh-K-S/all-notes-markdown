# Chapter 11: TFX (TensorFlow Extended) — Production ML Pipelines

## Table of Contents
- [11.1 Introduction to TFX](#111-introduction-to-tfx)
- [11.2 TFX Pipeline Architecture](#112-tfx-pipeline-architecture)
- [11.3 ExampleGen — Data Ingestion](#113-examplegen--data-ingestion)
- [11.4 StatisticsGen & SchemaGen — Data Understanding](#114-statisticsgen--schemagen--data-understanding)
- [11.5 ExampleValidator — Data Validation (TFDV)](#115-examplevalidator--data-validation-tfdv)
- [11.6 Transform — Feature Engineering (TFT)](#116-transform--feature-engineering-tft)
- [11.7 Trainer — Model Training](#117-trainer--model-training)
- [11.8 Tuner — Hyperparameter Tuning](#118-tuner--hyperparameter-tuning)
- [11.9 Evaluator — Model Evaluation (TFMA)](#119-evaluator--model-evaluation-tfma)
- [11.10 Pusher — Model Deployment](#1110-pusher--model-deployment)
- [11.11 ML Metadata (MLMD)](#1111-ml-metadata-mlmd)
- [11.12 Orchestrators (Airflow, Kubeflow, Vertex AI)](#1112-orchestrators-airflow-kubeflow-vertex-ai)
- [11.13 End-to-End TFX Pipeline Example](#1113-end-to-end-tfx-pipeline-example)
- [11.14 Common Mistakes](#1114-common-mistakes)
- [11.15 Interview Questions](#1115-interview-questions)
- [11.16 Quick Reference](#1116-quick-reference)

---

## 11.1 Introduction to TFX

### What It Is
TFX (TensorFlow Extended) is Google's **production-grade ML platform** for building end-to-end machine learning pipelines. Think of it as an **assembly line for ML** — raw data goes in one end, and a validated, deployable model comes out the other end, with every step automated, tracked, and reproducible.

If regular ML is like cooking a single meal, TFX is like running a restaurant kitchen — you need consistent recipes (pipelines), quality control (validation), inventory tracking (metadata), and food safety checks (model evaluation) at every step.

### Why It Matters
- **Reproducibility**: Every experiment is tracked and reproducible
- **Automation**: Retrain models automatically when new data arrives
- **Quality Gates**: Data and model validation prevent bad models from reaching production
- **Scale**: Handle terabytes of data with Apache Beam
- **Google-proven**: Used internally at Google for virtually all production ML
- **Industry standard**: Required knowledge for ML Engineering roles

### The ML Pipeline Problem

```
WITHOUT TFX (Ad-hoc ML):
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Jupyter  │──→│ Train in │──→│ Save .h5 │──→│ Deploy   │
│ Notebook │   │ Notebook │   │ Manually │   │ Manually │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
Problems: No validation, no tracking, not reproducible, breaks at scale

WITH TFX (Production ML):
┌─────────┐  ┌──────┐  ┌──────┐  ┌─────────┐  ┌───────┐  ┌─────────┐  ┌───────┐  ┌──────┐
│Example  │→ │Stats │→ │Schema│→ │Validator│→ │Trans- │→ │Trainer  │→ │Evalu- │→ │Pusher│
│Gen      │  │Gen   │  │Gen   │  │         │  │form   │  │         │  │ator   │  │      │
└─────────┘  └──────┘  └──────┘  └─────────┘  └───────┘  └─────────┘  └───────┘  └──────┘
     │           │          │          │            │           │           │          │
     └───────────┴──────────┴──────────┴────────────┴───────────┴───────────┴──────────┘
                              ML Metadata Store (tracks everything)
```

### Installation

```python
# Install TFX (includes TensorFlow, TFDV, TFT, TFMA)
# pip install tfx

# Individual components (if you need them separately)
# pip install tensorflow-data-validation   # TFDV
# pip install tensorflow-transform         # TFT
# pip install tensorflow-model-analysis    # TFMA
# pip install ml-metadata                  # MLMD

import tfx
print(f"TFX version: {tfx.__version__}")
```

---

## 11.2 TFX Pipeline Architecture

### Core Concepts

```
TFX PIPELINE ARCHITECTURE:

┌─────────────────────────────────────────────────────────────────┐
│                        TFX Pipeline                             │
│                                                                 │
│  Components (what runs):                                        │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
│  │ Component │→ │ Component │→ │ Component │→ │ Component │   │
│  │  Driver   │  │  Executor │  │  Publisher │  │           │   │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘   │
│                                                                 │
│  Artifacts (what flows between components):                     │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐               │
│  │ Examples │ ──→ │Statistics│ ──→ │  Schema  │ ──→ ...       │
│  │(tf.data) │     │ (proto)  │     │ (proto)  │               │
│  └──────────┘     └──────────┘     └──────────┘               │
│                                                                 │
│  ML Metadata Store (SQLite/MySQL):                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Tracks: artifacts, executions, lineage, parameters      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Orchestrator (what runs the pipeline):                         │
│  Apache Beam (local) | Airflow | Kubeflow | Vertex AI          │
└─────────────────────────────────────────────────────────────────┘
```

### Component Anatomy

Every TFX component has three parts:

```
┌─────────────────────────────────────┐
│           TFX Component             │
│                                     │
│  1. Driver                          │
│     - Resolves input artifacts      │
│     - Checks if execution needed    │
│     (caching — skip if already done)│
│                                     │
│  2. Executor                        │
│     - Does the actual work          │
│     - Data processing, training,    │
│       validation, etc.              │
│                                     │
│  3. Publisher                        │
│     - Registers output artifacts    │
│     - Updates ML Metadata store     │
│                                     │
│  Inputs → [Driver→Executor→Publisher] → Outputs
│                    ↕                │
│            ML Metadata Store        │
└─────────────────────────────────────┘
```

### Standard TFX Components

| Component | Purpose | Input | Output |
|-----------|---------|-------|--------|
| `ExampleGen` | Data ingestion | CSV, TFRecord, BigQuery | tf.Example |
| `StatisticsGen` | Compute statistics | Examples | DatasetFeatureStatistics |
| `SchemaGen` | Infer schema | Statistics | Schema |
| `ExampleValidator` | Validate data | Statistics + Schema | Anomalies |
| `Transform` | Feature engineering | Examples + Schema | Transformed Examples + Transform Graph |
| `Trainer` | Train model | Transformed Examples + Schema | SavedModel |
| `Tuner` | Hyperparameter tuning | Transformed Examples | Best Hyperparameters |
| `Evaluator` | Evaluate model | Examples + Model | Evaluation Results + Blessing |
| `Pusher` | Deploy model | Model + Blessing | Deployed Model |

---

## 11.3 ExampleGen — Data Ingestion

### What It Is
ExampleGen is the **first component** in any TFX pipeline. It ingests raw data from various sources and converts it into `tf.Example` format — a standard serialized format that all downstream components understand.

Think of it as the **receiving dock** of a warehouse — whatever format the shipments arrive in, they get unpacked and reorganized into standard containers.

### Why It Matters
- Standardizes data format across the pipeline
- Handles **train/eval splitting** automatically
- Supports **incremental data** (process only new data)
- Works with CSV, TFRecord, BigQuery, Parquet, and custom sources

### Code Examples

```python
from tfx.components import CsvExampleGen
from tfx.proto import example_gen_pb2
import os

# ─── Basic CSV Ingestion ───
# Reads all CSV files from the specified directory
csv_example_gen = CsvExampleGen(
    input_base=os.path.join(DATA_ROOT, 'data')
)

# ─── Custom Train/Eval Split ───
# Default is 2:1 (66% train, 33% eval)
# Custom: 80% train, 20% eval
output_config = example_gen_pb2.Output(
    split_config=example_gen_pb2.SplitConfig(
        splits=[
            example_gen_pb2.SplitConfig.Split(name='train', hash_buckets=8),
            example_gen_pb2.SplitConfig.Split(name='eval', hash_buckets=2)
        ]
    )
)

csv_example_gen = CsvExampleGen(
    input_base=os.path.join(DATA_ROOT, 'data'),
    output_config=output_config
)

# ─── Pre-split Data (train/ and eval/ directories) ───
input_config = example_gen_pb2.Input(
    splits=[
        example_gen_pb2.Input.Split(name='train', pattern='train/*'),
        example_gen_pb2.Input.Split(name='eval', pattern='eval/*')
    ]
)

csv_example_gen = CsvExampleGen(
    input_base=os.path.join(DATA_ROOT, 'data'),
    input_config=input_config
)
```

### Other Data Sources

```python
# ─── TFRecord ───
from tfx.components import ImportExampleGen

tfrecord_example_gen = ImportExampleGen(
    input_base=os.path.join(DATA_ROOT, 'tfrecords')
)

# ─── BigQuery ───
from tfx.extensions.google_cloud_big_query.example_gen.component import BigQueryExampleGen

query = """
    SELECT *
    FROM `project.dataset.table`
    WHERE date >= '2024-01-01'
"""

bq_example_gen = BigQueryExampleGen(query=query)

# ─── Parquet Files ───
from tfx.components import FileBasedExampleGen
from tfx.components.example_gen.custom_executors import parquet_executor

parquet_example_gen = FileBasedExampleGen(
    input_base=os.path.join(DATA_ROOT, 'parquet_data'),
    custom_executor_spec=parquet_executor.Executor
)
```

### Span and Version (Incremental Ingestion)

```python
# When new data arrives daily, use spans to process only new data
# Directory structure:
# data/
#   span-01/   ← Day 1 data
#   span-02/   ← Day 2 data (new)
#   span-03/   ← Day 3 data (newer)

input_config = example_gen_pb2.Input(
    splits=[
        example_gen_pb2.Input.Split(
            name='train',
            pattern='span-{SPAN}/*'   # {SPAN} is replaced automatically
        )
    ]
)

# Range config to process specific spans
range_config = example_gen_pb2.RangeConfig(
    static_range=example_gen_pb2.StaticRange(
        start_span_number=1,
        end_span_number=3
    )
)

csv_example_gen = CsvExampleGen(
    input_base=DATA_ROOT,
    input_config=input_config,
    range_config=range_config
)
```

---

## 11.4 StatisticsGen & SchemaGen — Data Understanding

### What They Are
- **StatisticsGen**: Computes descriptive statistics over your dataset (mean, median, distribution, missing values, etc.)
- **SchemaGen**: Infers a **schema** (data types, value ranges, expected features) from the statistics

Think of StatisticsGen as a **health checkup** for your data, and SchemaGen as writing down what "healthy" looks like so you can detect problems in future data.

### Why They Matter
- Understand your data before training
- Detect data drift and anomalies automatically
- Schema serves as a "contract" — new data must conform to it
- Statistics can be visualized with TFDV (TensorFlow Data Validation)

### StatisticsGen

```python
from tfx.components import StatisticsGen

# Compute statistics on the output of ExampleGen
statistics_gen = StatisticsGen(
    examples=csv_example_gen.outputs['examples']
)

# ─── Using TFDV Directly (for exploration) ───
import tensorflow_data_validation as tfdv

# Generate statistics from a CSV
stats = tfdv.generate_statistics_from_csv(
    data_location='data/train.csv',
    delimiter=','
)

# Visualize (works in Jupyter notebook)
tfdv.visualize_statistics(stats)

# Generate from a TFRecord
stats = tfdv.generate_statistics_from_tfrecord(
    data_location='data/train.tfrecord'
)

# Compare two datasets (e.g., train vs eval)
train_stats = tfdv.generate_statistics_from_csv('data/train.csv')
eval_stats = tfdv.generate_statistics_from_csv('data/eval.csv')

tfdv.visualize_statistics(
    lhs_statistics=train_stats,
    rhs_statistics=eval_stats,
    lhs_name='Train',
    rhs_name='Eval'
)
```

### What Statistics Include

```
Feature: "age"
┌─────────────────────────────────────┐
│ Type: INT                           │
│ Count: 50,000                       │
│ Missing: 123 (0.25%)                │
│ Mean: 38.5                          │
│ Std Dev: 13.2                       │
│ Min: 17                             │
│ Median: 37                          │
│ Max: 90                             │
│ Quantiles: [25, 33, 42, 55]         │
│ Histogram: ████▇▆▄▃▂▁              │
│ Unique values: 73                   │
│ Top values: 31(5.2%), 28(4.8%), ... │
└─────────────────────────────────────┘

Feature: "occupation"
┌─────────────────────────────────────┐
│ Type: STRING                        │
│ Count: 50,000                       │
│ Missing: 0 (0%)                     │
│ Unique: 15                          │
│ Top: "Prof-specialty" (25.3%)       │
│ Avg Length: 12.5 chars              │
│ Distribution: ██▇▅▃▂▁▁▁▁▁▁▁▁▁     │
└─────────────────────────────────────┘
```

### SchemaGen

```python
from tfx.components import SchemaGen

# Infer schema from statistics
schema_gen = SchemaGen(
    statistics=statistics_gen.outputs['statistics'],
    infer_feature_shape=True  # Infer tensor shapes from data
)

# ─── Using TFDV Directly ───
import tensorflow_data_validation as tfdv

# Infer schema
schema = tfdv.infer_schema(statistics=train_stats)

# Display schema
tfdv.display_schema(schema)

# ─── Modify Schema Manually ───
# Set a feature as required (no missing values allowed)
tfdv.get_feature(schema, 'age').presence.min_fraction = 1.0

# Set valid range for numeric feature
tfdv.set_domain(schema, 'age', 
                tfdv.schema_pb2.IntDomain(min=0, max=120))

# Set valid values for categorical feature
tfdv.set_domain(schema, 'sex',
                tfdv.schema_pb2.StringDomain(
                    value=['Male', 'Female']
                ))

# Mark feature as optional
tfdv.get_feature(schema, 'middle_name').presence.min_fraction = 0.0

# Save schema for version control
tfdv.write_schema_text(schema, 'schema/schema.pbtxt')

# Load schema
loaded_schema = tfdv.load_schema_text('schema/schema.pbtxt')
```

### Schema Example (Protocol Buffer Text Format)

```
feature {
  name: "age"
  type: INT
  presence { min_fraction: 1.0 }
  shape { dim { size: 1 } }
  int_domain { min: 0  max: 120 }
}
feature {
  name: "income"
  type: FLOAT
  presence { min_fraction: 0.95 }
  shape { dim { size: 1 } }
  float_domain { min: 0.0 }
}
feature {
  name: "occupation"
  type: BYTES
  presence { min_fraction: 1.0 }
  string_domain {
    value: "Engineer"
    value: "Doctor"
    value: "Teacher"
    ...
  }
}
```

---

## 11.5 ExampleValidator — Data Validation (TFDV)

### What It Is
ExampleValidator compares your **actual data statistics** against the **expected schema** and flags **anomalies**. It's the quality control inspector — if the data doesn't match expectations, it raises alerts before you waste time training on bad data.

### Why It Matters
- Catches **data drift** (distribution changes over time)
- Detects **schema violations** (new categories, wrong types, missing features)
- Prevents **garbage-in-garbage-out** scenarios
- Saves hours of debugging "why did my model suddenly get worse?"

### How Data Drift Works

```
TRAINING DATA (historical):           NEW DATA (production):
age distribution:                     age distribution:
  20-30: ████████ (40%)                 20-30: ██ (10%)        ← DRIFT!
  30-40: ██████ (30%)                   30-40: ██████ (30%)
  40-50: ████ (20%)                     40-50: ████████ (40%)  ← DRIFT!
  50+:   ██ (10%)                       50+:   ████ (20%)      ← DRIFT!

Schema says "sex" has values: [Male, Female]
New data has: [Male, Female, Non-binary]    ← NEW VALUE ANOMALY!

Schema says "income" is never missing
New data has 5% missing income              ← MISSING VALUE ANOMALY!
```

### Code Examples

```python
from tfx.components import ExampleValidator

# ─── In TFX Pipeline ───
example_validator = ExampleValidator(
    statistics=statistics_gen.outputs['statistics'],
    schema=schema_gen.outputs['schema']
)

# ─── Using TFDV Directly ───
import tensorflow_data_validation as tfdv

# Load schema and compute new data statistics
schema = tfdv.load_schema_text('schema/schema.pbtxt')
new_stats = tfdv.generate_statistics_from_csv('data/new_data.csv')

# Validate
anomalies = tfdv.validate_statistics(
    statistics=new_stats,
    schema=schema
)

# Display anomalies
tfdv.display_anomalies(anomalies)
```

### Detecting Data Drift and Skew

```python
import tensorflow_data_validation as tfdv
from tensorflow_data_validation.utils.schema_util import (
    get_feature, set_domain
)

# ─── Configure Drift Detection ───
# Compare serving data against training data

# For numeric features: use L-infinity distance
# (max absolute difference between distributions)
schema = tfdv.load_schema_text('schema/schema.pbtxt')

# Set drift threshold for 'age' feature
tfdv.get_feature(schema, 'age').drift_comparator.infinity_norm.threshold = 0.05
# If distribution shifts by more than 5%, flag as drift

# For categorical features: use L-infinity on distribution
tfdv.get_feature(schema, 'occupation').drift_comparator.infinity_norm.threshold = 0.1

# ─── Detect Training-Serving Skew ───
# Compare statistics of training data vs serving data
tfdv.get_feature(schema, 'age').skew_comparator.infinity_norm.threshold = 0.05

# Validate with both training and serving stats
anomalies = tfdv.validate_statistics(
    statistics=serving_stats,
    schema=schema,
    previous_statistics=training_stats  # For drift detection
)

tfdv.display_anomalies(anomalies)
```

### Common Anomaly Types

| Anomaly | Description | Action |
|---------|-------------|--------|
| `SCHEMA_MISSING_COLUMN` | Feature exists in schema but not in data | Check data pipeline |
| `SCHEMA_NEW_COLUMN` | Feature in data not in schema | Add to schema or remove |
| `INT_TYPE_SMALL_INT` | Integer column has unexpected small values | Verify data source |
| `ENUM_TYPE_UNEXPECTED_STRING_VALUES` | Categorical has new values | Update schema or fix data |
| `FLOAT_TYPE_HAS_NAN` | NaN values in float column | Add imputation |
| `FEATURE_MISSING_VALUE` | Missing rate exceeds threshold | Investigate data source |
| `DATASET_HIGH_NUM_EXAMPLES` | Unexpected data volume change | Check ingestion pipeline |

> **Pro Tip**: Always version-control your schema file (`schema.pbtxt`). When the schema needs to change (e.g., new feature added), update it explicitly rather than re-inferring — this prevents accidental acceptance of data quality issues.

---

## 11.6 Transform — Feature Engineering (TFT)

### What It Is
The Transform component performs **feature engineering** using TensorFlow Transform (TFT). The key insight: your preprocessing logic is saved as a **TensorFlow graph** that is deployed **alongside your model**. This guarantees that the same transformations applied during training are applied during serving.

### Why It Matters
- **Training-serving skew prevention**: Preprocessing is part of the model graph
- **Full-pass transformations**: Can compute vocabulary, min/max over the ENTIRE dataset (not just a batch)
- **Scalable**: Runs on Apache Beam (can process terabytes)
- **Reusable**: Transform graph is saved and reused in production

### The Training-Serving Skew Problem

```
WITHOUT TFT (dangerous):
┌──────────┐              ┌──────────┐
│ Training │              │ Serving  │
│          │              │          │
│ Python   │              │ Java     │  ← Different language!
│ sklearn  │              │ custom   │  ← Different code!
│ normalize│              │ normalize│  ← Might differ!
│          │              │          │
│ Result:  │              │ Result:  │
│ age=0.5  │              │ age=0.3  │  ← DIFFERENT! Bug!
└──────────┘              └──────────┘

WITH TFT (safe):
┌──────────┐              ┌──────────┐
│ Training │              │ Serving  │
│          │              │          │
│ TFT      │──── SAME ───│ TFT      │  ← Same TF graph
│ Graph    │    GRAPH     │ Graph    │
│          │              │          │
│ Result:  │              │ Result:  │
│ age=0.5  │              │ age=0.5  │  ← IDENTICAL! ✓
└──────────┘              └──────────┘
```

### TFT Preprocessing Functions

```python
import tensorflow_transform as tft

def preprocessing_fn(inputs):
    """Preprocessing function for TFT.
    
    This function is run on the ENTIRE dataset (not per-batch).
    TFT analyzers (tft.mean, tft.vocabulary, etc.) compute
    full-dataset statistics in a first pass, then apply
    transformations in a second pass.
    
    Args:
        inputs: Dict of feature tensors
    Returns:
        Dict of transformed feature tensors
    """
    outputs = {}
    
    # ─── Numerical Features ───
    
    # Z-score normalization: (x - mean) / std
    # tft.scale_to_z_score computes mean/std over ENTIRE dataset
    outputs['age_normalized'] = tft.scale_to_z_score(inputs['age'])
    
    # Min-max scaling to [0, 1]
    outputs['income_scaled'] = tft.scale_to_0_1(inputs['income'])
    
    # Bucketize (bin) continuous values
    outputs['age_bucket'] = tft.bucketize(inputs['age'], num_buckets=10)
    
    # Bucketize with explicit boundaries
    outputs['income_bucket'] = tft.apply_buckets(
        inputs['income'],
        bucket_boundaries=[[20000, 40000, 60000, 80000, 100000]]
    )
    
    # Log transformation
    outputs['log_income'] = tf.math.log1p(tf.cast(inputs['income'], tf.float32))
    
    # ─── Categorical Features ───
    
    # Convert string to integer index
    # tft.compute_and_apply_vocabulary scans ALL data for unique values
    outputs['occupation_index'] = tft.compute_and_apply_vocabulary(
        inputs['occupation'],
        top_k=100,           # Keep top 100 values
        num_oov_buckets=1    # 1 bucket for out-of-vocabulary values
    )
    
    # Hash categorical (for high-cardinality features)
    outputs['zipcode_hash'] = tft.hash_strings(
        inputs['zipcode'],
        hash_buckets=1000
    )
    
    # ─── Text Features ───
    
    # Bag of words
    outputs['text_bow'] = tft.bag_of_words(
        inputs['text'],
        top_k=5000
    )
    
    # TF-IDF
    outputs['text_tfidf'] = tft.tfidf(
        inputs['text'],
        vocab_size=5000
    )
    
    # N-grams
    outputs['text_ngrams'] = tft.ngrams(
        inputs['text'],
        ngram_range=(1, 3),
        top_k=10000
    )
    
    # ─── Pass through (no transformation) ───
    outputs['label'] = inputs['label']
    
    return outputs
```

### TFX Transform Component

```python
from tfx.components import Transform

transform = Transform(
    examples=csv_example_gen.outputs['examples'],
    schema=schema_gen.outputs['schema'],
    module_file=os.path.abspath('preprocessing.py')  # Contains preprocessing_fn
)

# Output artifacts:
# - transform.outputs['transformed_examples']  → Transformed data
# - transform.outputs['transform_graph']       → Saved preprocessing graph
```

### Key TFT Analyzers

| Analyzer | Purpose | Example |
|----------|---------|---------|
| `tft.mean()` | Compute mean over entire dataset | Centering |
| `tft.var()` | Compute variance | Normalization |
| `tft.min()` / `tft.max()` | Min/max values | Scaling |
| `tft.scale_to_z_score()` | Z-score normalize | Standard scaling |
| `tft.scale_to_0_1()` | Min-max scale | [0,1] range |
| `tft.bucketize()` | Equal-width binning | Discretization |
| `tft.quantiles()` | Compute quantile boundaries | Equal-frequency bins |
| `tft.compute_and_apply_vocabulary()` | String → index | Categorical encoding |
| `tft.hash_strings()` | String → hash bucket | High-cardinality categoricals |
| `tft.tfidf()` | TF-IDF vectorization | Text features |
| `tft.pca()` | Principal Component Analysis | Dimensionality reduction |

### Two-Phase Processing

```
PHASE 1: ANALYZE (full pass over data)
┌─────────────────────────────────────────┐
│ Read ALL data → Compute statistics      │
│                                         │
│ tft.mean(age) → mean=38.5              │
│ tft.var(age)  → var=174.2              │
│ tft.vocabulary(occupation) → [Eng, Dr]  │
│ tft.quantiles(income, 10) → [20k..100k]│
│                                         │
│ These become CONSTANTS in the graph     │
└─────────────────────────────────────────┘
                    │
                    ▼
PHASE 2: TRANSFORM (apply to each example)
┌─────────────────────────────────────────┐
│ For each example:                       │
│                                         │
│ age=45 → (45 - 38.5) / √174.2 = 0.49  │
│ occupation="Engineer" → index=0         │
│ income=75000 → bucket=7                │
│                                         │
│ Same constants used in serving! ✓       │
└─────────────────────────────────────────┘
```

> **Pro Tip**: Only use TFT for transformations that need full-dataset statistics (normalization, vocabulary). For simple per-example transforms (e.g., string splitting, type casting), use `tf.data.Dataset.map()` instead — it's simpler and faster.

---

## 11.7 Trainer — Model Training

### What It Is
The Trainer component trains your TensorFlow/Keras model using the transformed data from the Transform component. It integrates with the pipeline's schema, transform graph, and hyperparameters.

### Why It Matters
- Automatic integration with Transform artifacts
- Distributed training support
- Warm-starting from previous models
- Hyperparameter integration from Tuner component

### Trainer Module File

```python
# trainer_module.py — The model definition file

import tensorflow as tf
import tensorflow_transform as tft
from tfx.components.trainer.fn_args_utils import FnArgs

# ─── Feature Configuration ───
NUMERIC_FEATURES = ['age', 'income', 'hours_per_week']
CATEGORICAL_FEATURES = ['occupation', 'education', 'marital_status']
LABEL_KEY = 'label'

def _get_serve_tf_examples_fn(model, tf_transform_output):
    """Creates a serving function that applies transform graph."""
    
    # Get the transform graph
    model.tft_layer = tf_transform_output.transform_features_layer()
    
    @tf.function(input_signature=[
        tf.TensorSpec(shape=[None], dtype=tf.string, name='examples')
    ])
    def serve_tf_examples_fn(serialized_tf_examples):
        """Serving function that preprocesses and predicts."""
        # Parse raw tf.Example
        feature_spec = tf_transform_output.raw_feature_spec()
        feature_spec.pop(LABEL_KEY)  # Remove label from serving
        
        parsed_features = tf.io.parse_example(
            serialized_tf_examples, feature_spec
        )
        
        # Apply same transforms as training
        transformed_features = model.tft_layer(parsed_features)
        
        # Predict
        return model(transformed_features)
    
    return serve_tf_examples_fn


def _build_keras_model(tf_transform_output):
    """Build a Keras model for the pipeline."""
    
    feature_columns = []
    
    # Numeric features (already normalized by TFT)
    for feature in NUMERIC_FEATURES:
        feature_columns.append(
            tf.feature_column.numeric_column(
                feature + '_normalized', shape=(1,)
            )
        )
    
    # Categorical features (already indexed by TFT)
    for feature in CATEGORICAL_FEATURES:
        vocab_size = tf_transform_output.vocabulary_size_by_name(feature)
        feature_columns.append(
            tf.feature_column.embedding_column(
                tf.feature_column.categorical_column_with_identity(
                    feature + '_index',
                    num_buckets=vocab_size + 1  # +1 for OOV
                ),
                dimension=min(vocab_size // 2, 64)
            )
        )
    
    # Build model
    feature_layer = tf.keras.layers.DenseFeatures(feature_columns)
    
    model = tf.keras.Sequential([
        feature_layer,
        tf.keras.layers.Dense(256, activation='relu'),
        tf.keras.layers.Dropout(0.3),
        tf.keras.layers.Dense(128, activation='relu'),
        tf.keras.layers.Dropout(0.2),
        tf.keras.layers.Dense(1, activation='sigmoid')
    ])
    
    model.compile(
        optimizer=tf.keras.optimizers.Adam(learning_rate=0.001),
        loss='binary_crossentropy',
        metrics=[
            tf.keras.metrics.BinaryAccuracy(),
            tf.keras.metrics.AUC(name='auc')
        ]
    )
    
    return model


def _input_fn(file_pattern, tf_transform_output, batch_size=64):
    """Create tf.data.Dataset from transformed examples."""
    
    transformed_feature_spec = (
        tf_transform_output.transformed_feature_spec().copy()
    )
    
    dataset = tf.data.experimental.make_batched_features_dataset(
        file_pattern=file_pattern,
        batch_size=batch_size,
        features=transformed_feature_spec,
        reader=tf.data.TFRecordDataset,
        label_key=LABEL_KEY
    )
    
    return dataset.prefetch(tf.data.AUTOTUNE)


def run_fn(fn_args: FnArgs):
    """Main training function called by TFX Trainer."""
    
    # Load transform output
    tf_transform_output = tft.TFTransformOutput(fn_args.transform_output)
    
    # Create datasets
    train_dataset = _input_fn(
        fn_args.train_files,
        tf_transform_output,
        batch_size=64
    )
    
    eval_dataset = _input_fn(
        fn_args.eval_files,
        tf_transform_output,
        batch_size=64
    )
    
    # Build model
    model = _build_keras_model(tf_transform_output)
    
    # Callbacks
    callbacks = [
        tf.keras.callbacks.EarlyStopping(
            monitor='val_auc',
            patience=5,
            restore_best_weights=True,
            mode='max'
        ),
        tf.keras.callbacks.TensorBoard(
            log_dir=fn_args.model_run_dir,
            update_freq='batch'
        )
    ]
    
    # Use hyperparameters from Tuner (if available)
    if fn_args.hyperparameters:
        hparams = fn_args.hyperparameters.get('values', {})
        learning_rate = hparams.get('learning_rate', 0.001)
        model.optimizer.learning_rate = learning_rate
    
    # Train
    model.fit(
        train_dataset,
        validation_data=eval_dataset,
        epochs=fn_args.custom_config.get('epochs', 50),
        callbacks=callbacks
    )
    
    # Save model with serving signature
    signatures = {
        'serving_default': _get_serve_tf_examples_fn(
            model, tf_transform_output
        ).get_concrete_function(
            tf.TensorSpec(shape=[None], dtype=tf.string, name='examples')
        )
    }
    
    model.save(
        fn_args.serving_model_dir,
        save_format='tf',
        signatures=signatures
    )
```

### Trainer Component in Pipeline

```python
from tfx.components import Trainer
from tfx.proto import trainer_pb2

trainer = Trainer(
    module_file=os.path.abspath('trainer_module.py'),
    examples=transform.outputs['transformed_examples'],
    transform_graph=transform.outputs['transform_graph'],
    schema=schema_gen.outputs['schema'],
    train_args=trainer_pb2.TrainArgs(num_steps=10000),  # Training steps
    eval_args=trainer_pb2.EvalArgs(num_steps=1000),     # Eval steps
    custom_config={'epochs': 50}
)

# With hyperparameters from Tuner:
trainer = Trainer(
    module_file=os.path.abspath('trainer_module.py'),
    examples=transform.outputs['transformed_examples'],
    transform_graph=transform.outputs['transform_graph'],
    schema=schema_gen.outputs['schema'],
    hyperparameters=tuner.outputs['best_hyperparameters'],  # From Tuner
    train_args=trainer_pb2.TrainArgs(num_steps=10000),
    eval_args=trainer_pb2.EvalArgs(num_steps=1000)
)
```

---

## 11.8 Tuner — Hyperparameter Tuning

### What It Is
The Tuner component performs automated hyperparameter search using Keras Tuner. It finds the best combination of hyperparameters (learning rate, units, dropout, etc.) and passes them to the Trainer.

### Code Example

```python
# tuner_module.py

import keras_tuner as kt
import tensorflow as tf
import tensorflow_transform as tft
from tfx.components.trainer.fn_args_utils import FnArgs


def _build_model_for_tuning(hp, tf_transform_output):
    """Build model with tunable hyperparameters."""
    
    inputs = {}
    encoded_features = []
    
    # Numeric features
    for feature in NUMERIC_FEATURES:
        inp = tf.keras.layers.Input(shape=(1,), name=feature + '_normalized')
        inputs[feature + '_normalized'] = inp
        encoded_features.append(inp)
    
    # Concatenate all features
    x = tf.keras.layers.Concatenate()(encoded_features)
    
    # Tunable architecture
    num_layers = hp.Int('num_layers', min_value=1, max_value=4, step=1)
    
    for i in range(num_layers):
        units = hp.Int(f'units_{i}', min_value=32, max_value=512, step=32)
        x = tf.keras.layers.Dense(units, activation='relu')(x)
        
        dropout_rate = hp.Float(f'dropout_{i}', 0.0, 0.5, step=0.1)
        x = tf.keras.layers.Dropout(dropout_rate)(x)
    
    output = tf.keras.layers.Dense(1, activation='sigmoid')(x)
    
    model = tf.keras.Model(inputs=inputs, outputs=output)
    
    # Tunable learning rate
    learning_rate = hp.Float(
        'learning_rate',
        min_value=1e-5,
        max_value=1e-2,
        sampling='LOG'
    )
    
    model.compile(
        optimizer=tf.keras.optimizers.Adam(learning_rate=learning_rate),
        loss='binary_crossentropy',
        metrics=[tf.keras.metrics.AUC(name='auc')]
    )
    
    return model


def tuner_fn(fn_args: FnArgs):
    """TFX Tuner entry point."""
    
    tf_transform_output = tft.TFTransformOutput(fn_args.transform_output)
    
    tuner = kt.Hyperband(
        hypermodel=lambda hp: _build_model_for_tuning(hp, tf_transform_output),
        objective=kt.Objective('val_auc', direction='max'),
        max_epochs=30,
        factor=3,
        hyperband_iterations=2,
        directory=fn_args.working_dir,
        project_name='hyperband_tuning'
    )
    
    return tfx.components.TunerFnResult(
        tuner=tuner,
        fit_kwargs={
            'x': _input_fn(fn_args.train_files, tf_transform_output, 64),
            'validation_data': _input_fn(fn_args.eval_files, tf_transform_output, 64),
            'callbacks': [
                tf.keras.callbacks.EarlyStopping(
                    monitor='val_auc', patience=3, mode='max'
                )
            ]
        }
    )
```

### Tuner Component

```python
from tfx.components import Tuner

tuner = Tuner(
    module_file=os.path.abspath('tuner_module.py'),
    examples=transform.outputs['transformed_examples'],
    transform_graph=transform.outputs['transform_graph'],
    schema=schema_gen.outputs['schema'],
    train_args=trainer_pb2.TrainArgs(num_steps=1000),
    eval_args=trainer_pb2.EvalArgs(num_steps=500)
)

# Output: tuner.outputs['best_hyperparameters']
# Pass this to Trainer component
```

---

## 11.9 Evaluator — Model Evaluation (TFMA)

### What It Is
The Evaluator component uses **TensorFlow Model Analysis (TFMA)** to perform deep evaluation of your model. Unlike simple accuracy numbers, TFMA gives you **sliced metrics** — performance broken down by feature values, time periods, or any grouping you define.

The Evaluator also acts as a **gatekeeper**: it compares your new model against a baseline (usually the currently deployed model) and only "blesses" the new model if it's better.

### Why It Matters
- A model with 95% overall accuracy might have 40% accuracy on minority groups
- Sliced evaluation reveals hidden performance problems
- Automatic model validation prevents regressions
- Required for responsible AI and fairness assessment

### How Model Blessing Works

```
MODEL EVALUATION GATE:

New Model (candidate):
┌──────────────────────────┐
│ Overall AUC: 0.92        │
│ Slice "age>60": AUC 0.88 │
│ Slice "male": AUC 0.93   │
│ Slice "female": AUC 0.91 │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│       EVALUATOR          │
│                          │
│ Check 1: AUC > 0.85? ✅  │
│ Check 2: Better than     │
│   current model? ✅      │
│ Check 3: All slices      │
│   above threshold? ✅    │
│                          │
│ Result: BLESSED ✅        │
└──────────┬───────────────┘
           │
           ▼
        Pusher deploys the model

If ANY check fails → NOT BLESSED ❌ → Model is NOT deployed
```

### TFMA Code Examples

```python
import tensorflow_model_analysis as tfma

# ─── Define Evaluation Configuration ───
eval_config = tfma.EvalConfig(
    model_specs=[
        tfma.ModelSpec(
            signature_name='serving_default',
            label_key='label',
            preprocessing_function_names=['transform_features']
        )
    ],
    
    # Slicing specs — evaluate on these subgroups
    slicing_specs=[
        tfma.SlicingSpec(),                              # Overall
        tfma.SlicingSpec(feature_keys=['sex']),           # By gender
        tfma.SlicingSpec(feature_keys=['race']),          # By race
        tfma.SlicingSpec(feature_keys=['age_bucket']),    # By age group
        tfma.SlicingSpec(                                 # Cross-slice
            feature_keys=['sex', 'race']
        ),
    ],
    
    # Metrics to compute
    metrics_specs=[
        tfma.MetricsSpec(
            metrics=[
                tfma.MetricConfig(class_name='BinaryAccuracy'),
                tfma.MetricConfig(class_name='AUC'),
                tfma.MetricConfig(class_name='Precision'),
                tfma.MetricConfig(class_name='Recall'),
                tfma.MetricConfig(class_name='FairnessIndicators',
                                  config='{"thresholds": [0.3, 0.5, 0.7]}'),
            ],
            # Thresholds for model validation
            thresholds={
                'auc': tfma.MetricThreshold(
                    value_threshold=tfma.GenericValueThreshold(
                        lower_bound={'value': 0.85}       # Absolute threshold
                    ),
                    change_threshold=tfma.GenericChangeThreshold(
                        direction=tfma.MetricDirection.HIGHER_IS_BETTER,
                        absolute={'value': -0.01}          # Can't drop more than 1%
                    )
                ),
                'binary_accuracy': tfma.MetricThreshold(
                    value_threshold=tfma.GenericValueThreshold(
                        lower_bound={'value': 0.80}
                    )
                )
            }
        )
    ]
)
```

### Evaluator Component

```python
from tfx.components import Evaluator

evaluator = Evaluator(
    examples=csv_example_gen.outputs['examples'],
    model=trainer.outputs['model'],
    baseline_model=model_resolver.outputs['model'],  # Current production model
    eval_config=eval_config
)

# Outputs:
# - evaluator.outputs['evaluation']  → Detailed metrics
# - evaluator.outputs['blessing']    → BLESSED or NOT_BLESSED
```

### Using TFMA Directly

```python
import tensorflow_model_analysis as tfma

# ─── Run Evaluation ───
eval_result = tfma.run_model_analysis(
    eval_shared_model=tfma.default_eval_shared_model(
        eval_saved_model_path='saved_model/',
        eval_config=eval_config
    ),
    data_location='data/eval.tfrecord',
    eval_config=eval_config,
    output_path='eval_output/'
)

# ─── Visualize in Jupyter ───
# Overall metrics
tfma.view.render_slicing_metrics(eval_result)

# Specific slice
tfma.view.render_slicing_metrics(
    eval_result,
    slicing_spec=tfma.SlicingSpec(feature_keys=['sex'])
)

# Time series (compare across model versions)
tfma.view.render_time_series(
    eval_result,
    tfma.SlicingSpec(),
    display_full_path=True
)

# Fairness indicators
from tensorflow_model_analysis.addons.fairness.view import widget_view
widget_view.render_fairness_indicator(eval_result)

# ─── Programmatic Access to Metrics ───
metric_value = tfma.load_eval_result(output_path='eval_output/')
for slice_key, metrics in metric_value.slicing_metrics:
    print(f"Slice: {slice_key}")
    print(f"  AUC: {metrics['']['']['auc']['doubleValue']:.4f}")
    print(f"  Accuracy: {metrics['']['']['binary_accuracy']['doubleValue']:.4f}")
```

> **Pro Tip**: Always include fairness slices (gender, race, age) in your eval config. Even if your use case seems unrelated to demographics, biased models can cause legal and ethical issues. It's much cheaper to catch this before deployment.

---

## 11.10 Pusher — Model Deployment

### What It Is
The Pusher component deploys a **blessed** model to a serving infrastructure. It only pushes if the Evaluator has blessed the model (i.e., the model passed all quality checks).

### Deployment Targets

```python
from tfx.components import Pusher
from tfx.proto import pusher_pb2

# ─── Deploy to Local Filesystem ───
pusher = Pusher(
    model=trainer.outputs['model'],
    model_blessing=evaluator.outputs['blessing'],
    push_destination=pusher_pb2.PushDestination(
        filesystem=pusher_pb2.PushDestination.Filesystem(
            base_directory='serving_model/'
        )
    )
)

# ─── Deploy to TF Serving ───
pusher = Pusher(
    model=trainer.outputs['model'],
    model_blessing=evaluator.outputs['blessing'],
    push_destination=pusher_pb2.PushDestination(
        filesystem=pusher_pb2.PushDestination.Filesystem(
            base_directory='/models/my_model'
            # TF Serving watches this directory for new versions
        )
    )
)

# ─── Deploy to Google Cloud AI Platform ───
from tfx.extensions.google_cloud_ai_platform.pusher import executor as ai_platform_pusher_executor

pusher = Pusher(
    model=trainer.outputs['model'],
    model_blessing=evaluator.outputs['blessing'],
    custom_executor_spec=ai_platform_pusher_executor.Executor,
    custom_config={
        'ai_platform_serving_args': {
            'model_name': 'my_model',
            'project_id': 'my-gcp-project',
            'regions': ['us-central1']
        }
    }
)
```

### Versioned Model Directory Structure

```
serving_model/
├── 1/                    ← Version 1 (timestamp-based)
│   ├── saved_model.pb
│   ├── variables/
│   │   ├── variables.data-00000-of-00001
│   │   └── variables.index
│   └── assets/
│       └── vocab.txt
├── 2/                    ← Version 2 (newer, auto-deployed)
│   ├── saved_model.pb
│   ├── variables/
│   └── assets/
└── 3/                    ← Version 3 (latest blessed model)
    ├── saved_model.pb
    ├── variables/
    └── assets/

TF Serving automatically loads the latest version (highest number)
```

---

## 11.11 ML Metadata (MLMD)

### What It Is
ML Metadata is the **backbone** of TFX — a metadata store that tracks every artifact, execution, and relationship in your pipeline. It answers questions like:
- "Which dataset was used to train model v3?"
- "What preprocessing was applied to the data?"
- "Which models passed evaluation?"

### Why It Matters
- **Reproducibility**: Re-create any experiment exactly
- **Lineage**: Trace any model back to its training data
- **Debugging**: When a model fails, trace what changed
- **Compliance**: Audit trail for regulated industries (healthcare, finance)

### MLMD Concepts

```
ML Metadata Store:

ARTIFACTS (things):
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Dataset v1      │  │ Model v3        │  │ Schema v2       │
│ type: Examples  │  │ type: Model     │  │ type: Schema    │
│ uri: gs://...   │  │ uri: gs://...   │  │ uri: gs://...   │
│ state: LIVE     │  │ state: LIVE     │  │ state: LIVE     │
└─────────────────┘  └─────────────────┘  └─────────────────┘

EXECUTIONS (runs):
┌─────────────────┐  ┌─────────────────┐
│ Training Run #5 │  │ Eval Run #5     │
│ type: Trainer   │  │ type: Evaluator │
│ state: COMPLETE │  │ state: COMPLETE │
│ params: lr=0.01 │  │ result: BLESSED │
└─────────────────┘  └─────────────────┘

EVENTS (connections):
Dataset v1 ──INPUT──→ Training Run #5 ──OUTPUT──→ Model v3
Model v3 ──INPUT──→ Eval Run #5 ──OUTPUT──→ Blessing
```

### Querying MLMD

```python
import ml_metadata as mlmd
from ml_metadata.proto import metadata_store_pb2

# ─── Connect to Metadata Store ───
# SQLite (local development)
connection_config = metadata_store_pb2.ConnectionConfig()
connection_config.sqlite.filename_uri = 'metadata/mlmd.db'
store = mlmd.MetadataStore(connection_config)

# MySQL (production)
# connection_config.mysql.host = 'mysql-server'
# connection_config.mysql.port = 3306
# connection_config.mysql.database = 'mlmd'
# connection_config.mysql.user = 'root'
# connection_config.mysql.password = 'password'

# ─── Query Artifacts ───
# Get all artifacts
all_artifacts = store.get_artifacts()
for artifact in all_artifacts:
    print(f"ID: {artifact.id}, Type: {artifact.type_id}, URI: {artifact.uri}")

# Get artifacts by type
model_type = store.get_artifact_type('Model')
models = store.get_artifacts_by_type('Model')
for model in models:
    print(f"Model: {model.uri}, State: {model.state}")

# ─── Query Executions ───
executions = store.get_executions()
for execution in executions:
    print(f"Execution: {execution.id}, Type: {execution.type_id}")

# ─── Lineage Tracking ───
# Find what artifacts were used to produce a specific model
events = store.get_events_by_artifact_ids([model_artifact_id])
for event in events:
    if event.type == metadata_store_pb2.Event.OUTPUT:
        # This execution produced this model
        execution = store.get_executions_by_id([event.execution_id])[0]
        
        # Find inputs to this execution
        input_events = store.get_events_by_execution_ids([execution.id])
        for input_event in input_events:
            if input_event.type == metadata_store_pb2.Event.INPUT:
                input_artifact = store.get_artifacts_by_id(
                    [input_event.artifact_id]
                )[0]
                print(f"Input: {input_artifact.uri}")
```

### Lineage Example

```
"Show me the lineage for production model v5"

Dataset (Jan 2024)
    └──→ ExampleGen (run #12)
            └──→ Examples (split: train/eval)
                    ├──→ StatisticsGen (run #12)
                    │       └──→ Statistics
                    │               └──→ SchemaGen (run #8)
                    │                       └──→ Schema v2
                    ├──→ ExampleValidator (run #12) ✅ No anomalies
                    └──→ Transform (run #12)
                            └──→ Transformed Examples
                                    └──→ Trainer (run #12, lr=0.001, epochs=50)
                                            └──→ Model v5
                                                    └──→ Evaluator (run #12)
                                                            ├──→ Evaluation (AUC=0.94) ✅
                                                            └──→ Blessing ✅
                                                                    └──→ Pusher (run #12)
                                                                            └──→ Deployed! 🚀
```

---

## 11.12 Orchestrators (Airflow, Kubeflow, Vertex AI)

### What They Are
Orchestrators manage the execution of your TFX pipeline — they decide when to run each component, handle dependencies, manage resources, and retry failures.

### Comparison

| Orchestrator | Best For | Complexity | Scale |
|-------------|----------|------------|-------|
| **Local (BeamDag)** | Development, testing | Low | Single machine |
| **Apache Airflow** | Batch processing, cron jobs | Medium | Medium clusters |
| **Kubeflow Pipelines** | Kubernetes-based ML | High | Large clusters |
| **Vertex AI Pipelines** | Google Cloud native | Medium | Auto-scaling |

### Local Orchestrator (Development)

```python
from tfx.orchestration.experimental.interactive.interactive_context import InteractiveContext

# ─── Interactive (Jupyter Notebook) ───
context = InteractiveContext()

# Run components one by one
context.run(csv_example_gen)
context.run(statistics_gen)
context.run(schema_gen)
context.run(example_validator)
context.run(transform)
context.run(trainer)
context.run(evaluator)
context.run(pusher)
```

### Apache Airflow Orchestrator

```python
from tfx.orchestration.airflow.airflow_dag_runner import AirflowDagRunner
from tfx.orchestration.airflow.airflow_dag_runner import AirflowPipelineConfig

# Define pipeline
def create_pipeline():
    return tfx.dsl.Pipeline(
        pipeline_name='my_pipeline',
        pipeline_root='gs://my-bucket/pipeline',
        components=[
            csv_example_gen,
            statistics_gen,
            schema_gen,
            example_validator,
            transform,
            trainer,
            evaluator,
            pusher
        ],
        metadata_connection_config=tfx.orchestration.metadata.sqlite_metadata_connection_config(
            'metadata/mlmd.db'
        )
    )

# Create Airflow DAG
airflow_config = AirflowPipelineConfig(
    schedule_interval='0 0 * * *',  # Daily at midnight
    start_date=datetime(2024, 1, 1)
)

DAG = AirflowDagRunner(airflow_config).run(create_pipeline())
```

### Kubeflow Pipelines Orchestrator

```python
from tfx.orchestration.kubeflow.v2 import kubeflow_v2_dag_runner

# Define pipeline
pipeline = create_pipeline()

# Run on Kubeflow
runner = kubeflow_v2_dag_runner.KubeflowV2DagRunner(
    config=kubeflow_v2_dag_runner.KubeflowV2DagRunnerConfig(),
    output_dir='pipeline_output/',
    output_filename='pipeline.json'
)

runner.run(pipeline)
```

### Vertex AI Pipelines (Google Cloud)

```python
from tfx.orchestration.kubeflow.v2 import kubeflow_v2_dag_runner

runner = kubeflow_v2_dag_runner.KubeflowV2DagRunner(
    config=kubeflow_v2_dag_runner.KubeflowV2DagRunnerConfig(),
    output_dir='gs://my-bucket/pipeline',
    output_filename='vertex_pipeline.json'
)

runner.run(create_pipeline())

# Submit to Vertex AI
from google.cloud import aiplatform

aiplatform.init(project='my-project', location='us-central1')

job = aiplatform.PipelineJob(
    display_name='my-tfx-pipeline',
    template_path='gs://my-bucket/pipeline/vertex_pipeline.json',
    pipeline_root='gs://my-bucket/pipeline-runs'
)

job.submit()
```

---

## 11.13 End-to-End TFX Pipeline Example

### Complete Pipeline Code

```python
"""
complete_pipeline.py
A full TFX pipeline for a binary classification task.
"""

import os
import tensorflow as tf
import tfx
from tfx.components import (
    CsvExampleGen,
    StatisticsGen,
    SchemaGen,
    ExampleValidator,
    Transform,
    Trainer,
    Evaluator,
    Pusher
)
from tfx.proto import example_gen_pb2, trainer_pb2, pusher_pb2
from tfx.dsl.components.common.resolver import Resolver
from tfx.dsl.input_resolution.strategies import latest_blessed_model_strategy
from tfx.types import Channel
from tfx.types.standard_artifacts import Model, ModelBlessing
import tensorflow_model_analysis as tfma

# ─── Pipeline Configuration ───
PIPELINE_NAME = 'census_income_classifier'
PIPELINE_ROOT = os.path.join('pipelines', PIPELINE_NAME)
DATA_ROOT = os.path.join('data', 'census')
METADATA_PATH = os.path.join('metadata', PIPELINE_NAME, 'metadata.db')
SERVING_MODEL_DIR = os.path.join('serving_model', PIPELINE_NAME)
MODULE_DIR = os.path.join('modules')


def create_pipeline(
    pipeline_name: str,
    pipeline_root: str,
    data_root: str,
    module_dir: str,
    serving_model_dir: str,
    metadata_path: str,
) -> tfx.dsl.Pipeline:
    """Create a TFX pipeline."""
    
    # ─── 1. Data Ingestion ───
    example_gen = CsvExampleGen(
        input_base=data_root,
        output_config=example_gen_pb2.Output(
            split_config=example_gen_pb2.SplitConfig(
                splits=[
                    example_gen_pb2.SplitConfig.Split(name='train', hash_buckets=4),
                    example_gen_pb2.SplitConfig.Split(name='eval', hash_buckets=1)
                ]
            )
        )
    )
    
    # ─── 2. Statistics ───
    statistics_gen = StatisticsGen(
        examples=example_gen.outputs['examples']
    )
    
    # ─── 3. Schema ───
    schema_gen = SchemaGen(
        statistics=statistics_gen.outputs['statistics'],
        infer_feature_shape=True
    )
    
    # ─── 4. Validation ───
    example_validator = ExampleValidator(
        statistics=statistics_gen.outputs['statistics'],
        schema=schema_gen.outputs['schema']
    )
    
    # ─── 5. Transform ───
    transform = Transform(
        examples=example_gen.outputs['examples'],
        schema=schema_gen.outputs['schema'],
        module_file=os.path.join(module_dir, 'preprocessing.py')
    )
    
    # ─── 6. Training ───
    trainer = Trainer(
        module_file=os.path.join(module_dir, 'trainer_module.py'),
        examples=transform.outputs['transformed_examples'],
        transform_graph=transform.outputs['transform_graph'],
        schema=schema_gen.outputs['schema'],
        train_args=trainer_pb2.TrainArgs(num_steps=5000),
        eval_args=trainer_pb2.EvalArgs(num_steps=1000)
    )
    
    # ─── 7. Model Resolution (get baseline for comparison) ───
    model_resolver = Resolver(
        strategy_class=latest_blessed_model_strategy.LatestBlessedModelStrategy,
        model=Channel(type=Model),
        model_blessing=Channel(type=ModelBlessing)
    ).with_id('latest_blessed_model_resolver')
    
    # ─── 8. Evaluation ───
    eval_config = tfma.EvalConfig(
        model_specs=[
            tfma.ModelSpec(label_key='label')
        ],
        slicing_specs=[
            tfma.SlicingSpec(),
            tfma.SlicingSpec(feature_keys=['sex']),
            tfma.SlicingSpec(feature_keys=['race'])
        ],
        metrics_specs=[
            tfma.MetricsSpec(
                metrics=[
                    tfma.MetricConfig(class_name='BinaryAccuracy'),
                    tfma.MetricConfig(class_name='AUC'),
                    tfma.MetricConfig(class_name='Precision'),
                    tfma.MetricConfig(class_name='Recall'),
                ],
                thresholds={
                    'auc': tfma.MetricThreshold(
                        value_threshold=tfma.GenericValueThreshold(
                            lower_bound={'value': 0.85}
                        ),
                        change_threshold=tfma.GenericChangeThreshold(
                            direction=tfma.MetricDirection.HIGHER_IS_BETTER,
                            absolute={'value': -0.005}
                        )
                    )
                }
            )
        ]
    )
    
    evaluator = Evaluator(
        examples=example_gen.outputs['examples'],
        model=trainer.outputs['model'],
        baseline_model=model_resolver.outputs['model'],
        eval_config=eval_config
    )
    
    # ─── 9. Deployment ───
    pusher = Pusher(
        model=trainer.outputs['model'],
        model_blessing=evaluator.outputs['blessing'],
        push_destination=pusher_pb2.PushDestination(
            filesystem=pusher_pb2.PushDestination.Filesystem(
                base_directory=serving_model_dir
            )
        )
    )
    
    # ─── Assemble Pipeline ───
    components = [
        example_gen,
        statistics_gen,
        schema_gen,
        example_validator,
        transform,
        trainer,
        model_resolver,
        evaluator,
        pusher
    ]
    
    return tfx.dsl.Pipeline(
        pipeline_name=pipeline_name,
        pipeline_root=pipeline_root,
        components=components,
        metadata_connection_config=(
            tfx.orchestration.metadata.sqlite_metadata_connection_config(
                metadata_path
            )
        )
    )


# ─── Run the Pipeline ───
if __name__ == '__main__':
    from tfx.orchestration.local.local_dag_runner import LocalDagRunner
    
    pipeline = create_pipeline(
        pipeline_name=PIPELINE_NAME,
        pipeline_root=PIPELINE_ROOT,
        data_root=DATA_ROOT,
        module_dir=MODULE_DIR,
        serving_model_dir=SERVING_MODEL_DIR,
        metadata_path=METADATA_PATH
    )
    
    LocalDagRunner().run(pipeline)
    print("Pipeline completed successfully!")
```

### Pipeline Visualization

```
COMPLETE TFX PIPELINE FLOW:

    ┌─────────────┐
    │  Raw Data   │
    │  (CSV/BQ)   │
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │ ExampleGen  │ ── Split train/eval
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │StatisticsGen│ ── Compute stats
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │  SchemaGen  │ ── Infer schema
    └──────┬──────┘
           │
    ┌──────▼──────────┐
    │ExampleValidator │ ── Check for anomalies
    └──────┬──────────┘
           │
    ┌──────▼──────┐
    │  Transform  │ ── Feature engineering
    └──────┬──────┘
           │
    ┌──────▼──────┐    ┌───────────┐
    │   Trainer   │◄───│   Tuner   │ (optional)
    └──────┬──────┘    └───────────┘
           │
    ┌──────▼──────┐    ┌─────────────────┐
    │  Evaluator  │◄───│ Model Resolver  │ (baseline)
    └──────┬──────┘    └─────────────────┘
           │
     blessed? ──── NO ──→ Stop (don't deploy)
           │
          YES
           │
    ┌──────▼──────┐
    │   Pusher    │ ── Deploy to serving
    └─────────────┘
```

---

## 11.14 Common Mistakes

### Mistake 1: Preprocessing Outside TFT
```python
# ❌ WRONG: Preprocessing in pandas (training-serving skew!)
df['age_normalized'] = (df['age'] - df['age'].mean()) / df['age'].std()

# ✅ CORRECT: Preprocessing in TFT (same transform at serving time)
def preprocessing_fn(inputs):
    outputs = {}
    outputs['age_normalized'] = tft.scale_to_z_score(inputs['age'])
    return outputs
```

### Mistake 2: No Model Validation Before Deployment
```python
# ❌ WRONG: Deploy immediately after training
pusher = Pusher(
    model=trainer.outputs['model'],
    push_destination=push_destination
    # No model_blessing — deploys every model regardless of quality!
)

# ✅ CORRECT: Only deploy blessed models
pusher = Pusher(
    model=trainer.outputs['model'],
    model_blessing=evaluator.outputs['blessing'],  # Gatekeeper!
    push_destination=push_destination
)
```

### Mistake 3: Ignoring Data Skew and Drift
```python
# ❌ WRONG: Skip validation
# "The data is always fine"

# ✅ CORRECT: Always validate, configure drift detection
schema = tfdv.load_schema_text('schema.pbtxt')
tfdv.get_feature(schema, 'age').drift_comparator.infinity_norm.threshold = 0.05
anomalies = tfdv.validate_statistics(new_stats, schema, previous_statistics=old_stats)
```

### Mistake 4: Not Using Sliced Evaluation
```python
# ❌ WRONG: Only check overall metrics
eval_config = tfma.EvalConfig(
    slicing_specs=[tfma.SlicingSpec()],  # Only overall
    ...
)

# ✅ CORRECT: Evaluate on important subgroups
eval_config = tfma.EvalConfig(
    slicing_specs=[
        tfma.SlicingSpec(),                            # Overall
        tfma.SlicingSpec(feature_keys=['gender']),     # By gender
        tfma.SlicingSpec(feature_keys=['age_group']),   # By age
        tfma.SlicingSpec(feature_keys=['region']),      # By region
    ],
    ...
)
```

### Mistake 5: Hardcoding Paths
```python
# ❌ WRONG: Hardcoded absolute paths
data_root = '/home/user/data/train.csv'

# ✅ CORRECT: Use relative paths and configuration
import os
DATA_ROOT = os.path.join(os.environ.get('DATA_DIR', 'data'), 'train')
PIPELINE_ROOT = os.path.join(os.environ.get('PIPELINE_DIR', 'pipelines'), PIPELINE_NAME)
```

### Mistake 6: No Metadata Store
```python
# ❌ WRONG: No metadata tracking
# "I'll just remember what I did"

# ✅ CORRECT: Always configure metadata store
pipeline = tfx.dsl.Pipeline(
    ...,
    metadata_connection_config=(
        tfx.orchestration.metadata.sqlite_metadata_connection_config(
            os.path.join('metadata', 'mlmd.db')
        )
    )
)
```

---

## 11.15 Interview Questions

### Q1: What is TFX and why would you use it over a Jupyter notebook?
**Answer**: TFX (TensorFlow Extended) is Google's production ML platform for building end-to-end ML pipelines. While notebooks are great for exploration, they're terrible for production: they're not reproducible, don't scale, have no validation, and can't be automated. TFX provides: automated data validation (catch bad data before training), consistent preprocessing (no training-serving skew), model evaluation with fairness analysis, metadata tracking (full lineage), and orchestration (run on schedule). Any ML system serving real users needs these capabilities.

### Q2: What is training-serving skew and how does TFT prevent it?
**Answer**: Training-serving skew occurs when preprocessing logic differs between training and serving, causing silent accuracy degradation. For example, if you normalize age using pandas mean/std during training but use different values in the Java serving system, predictions will be wrong. TFT (TensorFlow Transform) solves this by encoding preprocessing as a TensorFlow graph — the exact same computation runs during both training and serving. Analyzers (like `tft.scale_to_z_score`) compute statistics over the full dataset in a first pass, then bake those constants into the graph.

### Q3: Explain the role of the Evaluator component and model blessing.
**Answer**: The Evaluator performs deep model analysis using TFMA (TensorFlow Model Analysis). It computes metrics not just overall, but sliced by feature values (e.g., accuracy by gender, age group). It compares the candidate model against a baseline (usually the current production model) using configurable thresholds. If the candidate meets all thresholds (absolute and relative to baseline), it's "blessed" — marked as safe to deploy. The Pusher only deploys blessed models. This prevents deploying models that are worse than what's currently in production or that fail on specific subgroups.

### Q4: How does ML Metadata enable reproducibility?
**Answer**: MLMD tracks three types of entities: Artifacts (data, models, schemas), Executions (pipeline runs), and Events (connections between artifacts and executions). Every artifact has a URI, type, and state. Every execution records its parameters and timestamps. Events record which artifacts were inputs/outputs of each execution. This creates a complete lineage graph — you can trace any model back to its training data, schema, preprocessing, and hyperparameters. To reproduce an experiment, query MLMD for the execution's parameters and input artifacts.

### Q5: What data sharding strategies does TFX support for distributed training?
**Answer**: TFX supports four auto-sharding policies via `tf.data.experimental.AutoShardPolicy`:
- **AUTO**: TF decides based on dataset structure (default)
- **FILE**: Each worker reads different input files (best for multiple TFRecord files)
- **DATA**: Each worker gets different elements from the same data source (best for in-memory data)
- **OFF**: No automatic sharding — user handles it manually

### Q6: How would you set up a TFX pipeline for continuous training?
**Answer**: Use ExampleGen with span-based ingestion to process incremental data. Set up an orchestrator (Airflow/Kubeflow) with a schedule (e.g., daily/weekly). The pipeline: ingests new data (ExampleGen with span), validates against existing schema (ExampleValidator), transforms with existing transform graph, trains new model (optionally warm-starting from previous), evaluates against current production model (Evaluator with change thresholds), and only deploys if the new model is better (Pusher with blessing gate). Use BackupAndRestore for fault tolerance and MLMD for tracking.

### Q7: What is the difference between TFDV, TFT, and TFMA?
**Answer**:
- **TFDV** (TensorFlow Data Validation): Validates data quality — computes statistics, infers schemas, detects anomalies and drift. Used before training.
- **TFT** (TensorFlow Transform): Performs feature engineering as a TF graph — normalization, vocabulary creation, bucketization. Prevents training-serving skew.
- **TFMA** (TensorFlow Model Analysis): Evaluates trained models with sliced metrics and fairness indicators. Compares against baselines. Used after training, before deployment.

### Q8: How do you handle schema evolution in a TFX pipeline?
**Answer**: When your data changes legitimately (new features, changed categories), you need to update the schema. Best practice: (1) Run SchemaGen on new data to infer updated schema. (2) Review changes manually — don't blindly accept. (3) Version-control the schema file. (4) Update the Transform preprocessing_fn if new features need processing. (5) Use ExampleValidator with the new schema to validate. (6) Retrain and evaluate the model. The schema should be treated as a first-class artifact, versioned alongside code.

---

## 11.16 Quick Reference

### TFX Component Summary

| Component | Library | Key Function | Input | Output |
|-----------|---------|-------------|-------|--------|
| ExampleGen | TFX | Data ingestion | Raw data | tf.Examples |
| StatisticsGen | TFDV | Compute stats | Examples | Statistics |
| SchemaGen | TFDV | Infer schema | Statistics | Schema |
| ExampleValidator | TFDV | Validate data | Stats + Schema | Anomalies |
| Transform | TFT | Feature engineering | Examples + Schema | Transformed Examples |
| Trainer | TFX | Train model | Transformed Examples | SavedModel |
| Tuner | Keras Tuner | HPO | Transformed Examples | Best HParams |
| Evaluator | TFMA | Evaluate model | Examples + Model | Blessing |
| Pusher | TFX | Deploy model | Model + Blessing | Deployed Model |

### Key TFT Functions

| Function | Purpose |
|----------|---------|
| `tft.scale_to_z_score(x)` | Z-score normalization |
| `tft.scale_to_0_1(x)` | Min-max scaling |
| `tft.bucketize(x, n)` | Equal-width binning |
| `tft.compute_and_apply_vocabulary(x)` | String → integer index |
| `tft.hash_strings(x, n)` | Hash to n buckets |
| `tft.tfidf(x, vocab_size)` | TF-IDF vectorization |
| `tft.ngrams(x, range, k)` | N-gram extraction |
| `tft.pca(x, output_dim)` | PCA dimensionality reduction |
| `tft.mean(x)` | Full-dataset mean |
| `tft.var(x)` | Full-dataset variance |

### TFDV Quick Reference

```python
import tensorflow_data_validation as tfdv

# Generate statistics
stats = tfdv.generate_statistics_from_csv('data.csv')

# Infer schema
schema = tfdv.infer_schema(stats)

# Validate data
anomalies = tfdv.validate_statistics(stats, schema)

# Visualize
tfdv.visualize_statistics(stats)
tfdv.display_schema(schema)
tfdv.display_anomalies(anomalies)

# Compare datasets
tfdv.visualize_statistics(train_stats, eval_stats, 'Train', 'Eval')

# Save/Load schema
tfdv.write_schema_text(schema, 'schema.pbtxt')
schema = tfdv.load_schema_text('schema.pbtxt')
```

### TFMA Quick Reference

```python
import tensorflow_model_analysis as tfma

# Define eval config
eval_config = tfma.EvalConfig(
    model_specs=[tfma.ModelSpec(label_key='label')],
    slicing_specs=[
        tfma.SlicingSpec(),                          # Overall
        tfma.SlicingSpec(feature_keys=['feature']),   # By feature
    ],
    metrics_specs=[
        tfma.MetricsSpec(
            metrics=[tfma.MetricConfig(class_name='AUC')],
            thresholds={
                'auc': tfma.MetricThreshold(
                    value_threshold=tfma.GenericValueThreshold(
                        lower_bound={'value': 0.85}
                    )
                )
            }
        )
    ]
)

# Run evaluation
eval_result = tfma.run_model_analysis(
    eval_shared_model=tfma.default_eval_shared_model(
        eval_saved_model_path='model/', eval_config=eval_config
    ),
    data_location='eval.tfrecord',
    eval_config=eval_config
)

# Visualize
tfma.view.render_slicing_metrics(eval_result)
```

### Pipeline Orchestrator Cheatsheet

```python
# Local (development)
from tfx.orchestration.local.local_dag_runner import LocalDagRunner
LocalDagRunner().run(pipeline)

# Interactive (Jupyter)
from tfx.orchestration.experimental.interactive.interactive_context import InteractiveContext
context = InteractiveContext()
context.run(component)

# Airflow
from tfx.orchestration.airflow.airflow_dag_runner import AirflowDagRunner
DAG = AirflowDagRunner(config).run(pipeline)

# Kubeflow / Vertex AI
from tfx.orchestration.kubeflow.v2 import kubeflow_v2_dag_runner
runner = kubeflow_v2_dag_runner.KubeflowV2DagRunner(config)
runner.run(pipeline)
```

### Installation Commands

```bash
# Full TFX installation
pip install tfx

# Individual components
pip install tensorflow-data-validation    # TFDV
pip install tensorflow-transform          # TFT
pip install tensorflow-model-analysis     # TFMA
pip install ml-metadata                   # MLMD

# Extras
pip install keras-tuner                   # For Tuner component
pip install apache-beam[gcp]              # For GCP runners
```
