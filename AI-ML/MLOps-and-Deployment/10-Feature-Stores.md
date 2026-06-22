# Feature Stores

## Table of Contents
- [What is a Feature Store?](#what-is-a-feature-store)
- [Why It Matters](#why-it-matters)
- [How It Works](#how-it-works)
- [Online vs Offline Feature Stores](#online-vs-offline-feature-stores)
- [Feast — Open Source Feature Store](#feast--open-source-feature-store)
- [Tecton — Managed Feature Platform](#tecton--managed-feature-platform)
- [Feature Pipelines and Engineering](#feature-pipelines-and-engineering)
- [Feature Store Design Patterns](#feature-store-design-patterns)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is a Feature Store?

### Simple Explanation

Imagine you're running a factory that makes different products (ML models). Every product needs raw materials (data), and before using them you need to process them into components (features):
- **Raw material:** "User clicked at 2024-01-15 14:30:22 UTC"
- **Processed feature:** "user_clicks_last_7_days = 42"

Without a Feature Store, every product line (team/model) processes raw materials independently → waste, inconsistency, bugs.

A **Feature Store** is like a **central warehouse** for processed components:
- Compute features once, share everywhere
- Same feature definition used in training AND serving
- Track versions, lineage, and freshness
- Serve features at millisecond latency for real-time predictions

### Formal Definition

A Feature Store is a centralized data system that manages the lifecycle of ML features — from computation and storage to serving and monitoring. It ensures **consistency** between training and serving (avoiding training-serving skew), enables **reuse** across teams and models, and provides both **batch** (offline) and **low-latency** (online) access to features.

---

## Why It Matters

### The Problem Feature Stores Solve

```
WITHOUT A FEATURE STORE:
                                                           
  Data Scientist A                Data Scientist B          
  ┌──────────────┐               ┌──────────────┐          
  │ Computes     │               │ Computes     │          
  │ user_age as  │               │ user_age as  │          
  │ 2024 - birth │               │ today - DOB  │  ← Different logic!
  │              │               │ (includes    │          
  │ (integer)    │               │  months)     │          
  └──────────────┘               └──────────────┘          
         │                              │                   
    Trains model                   Trains model             
    with BATCH                     with BATCH               
    features                       features                 
         │                              │                   
  ┌──────────────┐               ┌──────────────┐          
  │ Serves model │               │ Serves model │          
  │ with LIVE    │               │ with LIVE    │          
  │ features     │               │ features     │          
  │ (different   │               │ (different   │          
  │  pipeline!)  │ ← SKEW!      │  pipeline!)  │ ← SKEW! 
  └──────────────┘               └──────────────┘          

Problems:
1. Duplicated work (same feature computed differently)
2. Training-serving skew (batch vs real-time pipelines differ)
3. No reuse (each model has its own feature pipeline)
4. No governance (who uses what? what changed?)
```

```
WITH A FEATURE STORE:
                                                           
  ┌──────────────────────────────────────────────┐         
  │              FEATURE STORE                     │         
  │                                                │         
  │  user_age = DATEDIFF(today, birth_date)        │         
  │  user_clicks_7d = COUNT(clicks WHERE t > -7d)  │         
  │  user_avg_order = AVG(orders.amount)            │         
  │                                                │         
  │  ┌──────────┐         ┌──────────┐            │         
  │  │ OFFLINE  │         │  ONLINE  │            │         
  │  │ Store    │ ──sync──│  Store   │            │         
  │  │(training)│         │(serving) │            │         
  │  └──────────┘         └──────────┘            │         
  │                                                │         
  └──────────┬──────────────────┬─────────────────┘         
             │                  │                            
    ┌────────┴────┐    ┌───────┴──────┐                    
    │ Model A     │    │ Model B      │                    
    │ Training    │    │ Real-time    │                    
    │ (same       │    │ Serving      │                    
    │  features!) │    │ (same        │                    
    └─────────────┘    │  features!)  │                    
                       └──────────────┘                    
                                                           
✓ Single source of truth                                   
✓ No training-serving skew                                 
✓ Feature reuse across teams                               
✓ Governance and lineage                                   
```

### Real-World Impact

| Problem | Without Feature Store | With Feature Store |
|---------|---------------------|-------------------|
| **Training-serving skew** | 30-40% of ML bugs | Eliminated by design |
| **Feature development time** | 2-4 weeks per feature | Hours (reuse existing) |
| **Team collaboration** | Each team builds own pipeline | Shared feature catalog |
| **Feature freshness** | Manual checks | Automated monitoring |
| **Point-in-time correctness** | Easy to leak future data | Built-in time travel |
| **Serving latency** | 50-200ms (compute on-the-fly) | 1-5ms (pre-computed) |

### When You Need a Feature Store

- **Multiple models** sharing features (fraud + recommendations + pricing)
- **Real-time serving** requiring pre-computed features at low latency
- **Regulated industries** needing feature lineage and audit trails
- **Teams > 3 data scientists** to avoid duplicated work
- **Training-serving skew** has caused production bugs

### When You DON'T Need One

- **1-2 simple models** with few features
- **Batch-only predictions** with no real-time requirements
- **Solo practitioner** or very small team
- **Prototype/MVP stage** — adds unnecessary complexity

---

## How It Works

### Feature Store Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     FEATURE STORE ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  DATA SOURCES              FEATURE PIPELINES                     │
│  ┌──────────┐             ┌──────────────────┐                  │
│  │Databases │──┐          │ Batch Pipeline   │                  │
│  │(Postgres)│  │          │ (Spark/SQL)      │──┐               │
│  └──────────┘  │          │ Runs: hourly/    │  │               │
│  ┌──────────┐  ├────────▶│ daily            │  │               │
│  │Event     │  │          └──────────────────┘  │               │
│  │Streams   │──┤          ┌──────────────────┐  │               │
│  │(Kafka)   │  │          │ Stream Pipeline  │  │               │
│  └──────────┘  │          │ (Flink/Spark SS) │  │               │
│  ┌──────────┐  │          │ Runs: real-time  │──┤               │
│  │Data Lake │──┘          └──────────────────┘  │               │
│  │(S3/GCS)  │                                    │               │
│  └──────────┘                                    │               │
│                                                   ▼               │
│                           ┌──────────────────────────────────┐  │
│                           │        FEATURE REGISTRY           │  │
│                           │  • Feature definitions            │  │
│                           │  • Metadata & documentation       │  │
│                           │  • Lineage & dependencies         │  │
│                           │  • Access control                 │  │
│                           └───────────┬──────────────────────┘  │
│                                       │                          │
│                          ┌────────────┼────────────┐            │
│                          ▼            ▼            ▼            │
│                   ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│                   │ OFFLINE  │ │  ONLINE  │ │ FEATURE  │       │
│                   │  STORE   │ │  STORE   │ │MONITORING│       │
│                   │          │ │          │ │          │       │
│                   │ BigQuery │ │  Redis   │ │ Drift    │       │
│                   │ Snowflake│ │ DynamoDB │ │ Freshness│       │
│                   │ S3/GCS   │ │ Cassandra│ │ Quality  │       │
│                   └────┬─────┘ └────┬─────┘ └──────────┘       │
│                        │            │                            │
│                        ▼            ▼                            │
│                   ┌──────────┐ ┌──────────┐                     │
│                   │ Training │ │ Serving  │                     │
│                   │ (Batch   │ │ (Real-   │                     │
│                   │  retrieval│ │ time API)│                     │
│                   └──────────┘ └──────────┘                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Concepts

#### 1. Feature Definition

```python
# A feature definition specifies:
# - Name: unique identifier
# - Entity: what the feature describes (user, product, transaction)
# - Type: data type (float, int, string, list)
# - Source: where raw data comes from
# - Transformation: how to compute the feature
# - Freshness: how often it should be updated

# Example:
feature_definition = {
    "name": "user_avg_purchase_30d",
    "entity": "user_id",
    "type": "float64",
    "description": "Average purchase amount in the last 30 days",
    "source": "orders_table",
    "transformation": "AVG(amount) WHERE date > NOW() - 30 DAYS GROUP BY user_id",
    "freshness": "1 hour",
    "owner": "fraud-team",
    "tags": ["monetary", "behavioral"]
}
```

#### 2. Entity

An entity is the "key" that features are associated with (like a primary key in a database).

```python
# Common entities:
# - user_id     → user-level features (age, total_purchases, avg_session)
# - product_id  → product features (price, category, avg_rating)
# - driver_id   → driver features (rating, trips_completed, acceptance_rate)
# - store_id    → store features (revenue, order_count, avg_delivery_time)

# Compound entities:
# - (user_id, product_id) → user-product interaction features
# - (user_id, merchant_id) → user-merchant transaction features
```

#### 3. Point-in-Time Correctness

```
WHY THIS MATTERS — Preventing Data Leakage

Timeline:
Jan 1    Jan 15    Feb 1    Feb 15    Mar 1 (prediction)
  │        │         │        │         │
  ▼        ▼         ▼        ▼         ▼
Order    Order     Order    [Label]   Predict
$50      $100      $75      Churned?  Will churn?

WRONG: When training, if you compute avg_purchase = ($50+$100+$75)/3 = $75
       You're using Feb 1 data to predict Jan 15 → DATA LEAKAGE!

RIGHT (Point-in-time): At prediction time Feb 15, only use data available
       up to Feb 15: avg_purchase = ($50+$100+$75)/3 = $75 ✓
       At label time Jan 15: avg_purchase = ($50)/1 = $50

Feature stores handle this automatically via event timestamps!
```

---

## Online vs Offline Feature Stores

### Comparison

```
┌──────────────────────────┬──────────────────────────┐
│     OFFLINE STORE         │      ONLINE STORE         │
├──────────────────────────┼──────────────────────────┤
│                            │                            │
│  Purpose: Training         │  Purpose: Serving          │
│                            │                            │
│  Latency: Seconds-minutes  │  Latency: 1-10 ms          │
│                            │                            │
│  Storage: Data warehouse   │  Storage: Key-value store  │
│  (BigQuery, Snowflake, S3) │  (Redis, DynamoDB)         │
│                            │                            │
│  Data: Historical (months) │  Data: Latest values only  │
│                            │                            │
│  Access: Batch queries     │  Access: Point lookups     │
│  SELECT * WHERE date       │  GET user_features(id=123) │
│  BETWEEN '2024-01' AND    │                            │
│  '2024-06'                 │                            │
│                            │                            │
│  Size: TB-PB               │  Size: GB-TB               │
│                            │                            │
│  Cost: Low (cold storage)  │  Cost: High (always-on)    │
│                            │                            │
└──────────────────────────┴──────────────────────────┘

SYNC: Offline store → materialization job → Online store
      (Ensures training and serving use the same features)
```

### When to Use Each

| Scenario | Store Type | Why |
|----------|-----------|-----|
| Model training | Offline | Need historical data for train/test splits |
| Batch predictions | Offline | Processing large volumes, latency doesn't matter |
| Real-time API predictions | Online | Need features in <10ms |
| A/B testing analysis | Offline | Historical analysis of model performance |
| Feature exploration | Offline | Data scientists browsing features |
| Fraud detection (live) | Online | Must decide in milliseconds |
| Recommendation engine | Online | Real-time personalization |

---

## Feast — Open Source Feature Store

### What is Feast?

Feast (Feature Store) is the most popular open-source feature store. It provides:
- Feature definitions as code (version controlled)
- Offline store (BigQuery, Snowflake, Redshift, file-based)
- Online store (Redis, DynamoDB, SQLite)
- Point-in-time correct training data generation
- Feature serving API

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    FEAST ARCHITECTURE                      │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  feature_repo/                                           │
│  ├── feature_store.yaml    ← Configuration               │
│  ├── features.py           ← Feature definitions         │
│  └── data/                 ← Data sources                │
│                                                           │
│  ┌─────────────┐    feast apply    ┌───────────────┐    │
│  │ Feature     │ ──────────────▶  │   Registry     │    │
│  │ Definitions │                   │  (metadata)    │    │
│  │ (Python)    │                   └───────────────┘    │
│  └─────────────┘                          │              │
│                                           │              │
│       feast materialize          ┌────────┴──────┐      │
│  ┌──────────┐ ──────────────▶  │  Online Store  │      │
│  │ Offline   │                  │  (Redis)       │      │
│  │ Store     │                  └────────────────┘      │
│  │ (S3/BQ)  │                                           │
│  └──────────┘                                           │
│       │                                                   │
│       │  get_historical_features()                       │
│       ▼                                                   │
│  Training Data                                           │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### Code: Complete Feast Workflow

```python
# ============================================
# STEP 0: Install Feast
# ============================================
# pip install feast[redis]
# For cloud: pip install feast[gcp] or feast[aws]

# ============================================
# STEP 1: Initialize Feature Repository
# ============================================
# Run in terminal: feast init my_feature_repo
# This creates the directory structure

# ============================================
# STEP 2: Define Data Sources and Features
# ============================================

# feature_repo/features.py
"""
Feature definitions for a fraud detection system.
"""
from datetime import timedelta
from feast import (
    Entity, 
    Feature, 
    FeatureView, 
    FileSource,
    ValueType,
    Field
)
from feast.types import Float32, Float64, Int64, String

# --- Define Entities ---
# An entity is the "join key" for features

user = Entity(
    name="user_id",
    description="Unique identifier for a user",
    join_keys=["user_id"]
)

merchant = Entity(
    name="merchant_id",
    description="Unique identifier for a merchant",
    join_keys=["merchant_id"]
)


# --- Define Data Sources ---
# Where the raw/pre-computed feature data lives

user_features_source = FileSource(
    path="data/user_features.parquet",    # Parquet file with feature data
    timestamp_field="event_timestamp",     # When the feature was computed
    created_timestamp_column="created_timestamp"  # When the row was inserted
)

transaction_stats_source = FileSource(
    path="data/transaction_stats.parquet",
    timestamp_field="event_timestamp"
)

merchant_features_source = FileSource(
    path="data/merchant_features.parquet",
    timestamp_field="event_timestamp"
)


# --- Define Feature Views ---
# A FeatureView groups related features from the same source

user_profile_fv = FeatureView(
    name="user_profile",
    entities=[user],
    ttl=timedelta(days=365),    # How long features are valid
    schema=[
        Field(name="user_age", dtype=Int64),
        Field(name="account_age_days", dtype=Int64),
        Field(name="country", dtype=String),
        Field(name="is_verified", dtype=Int64),
    ],
    source=user_features_source,
    tags={"team": "user-platform", "priority": "high"}
)

user_transaction_stats_fv = FeatureView(
    name="user_transaction_stats",
    entities=[user],
    ttl=timedelta(hours=24),    # Features expire after 24h (need fresh data)
    schema=[
        Field(name="txn_count_7d", dtype=Int64),
        Field(name="txn_count_30d", dtype=Int64),
        Field(name="avg_txn_amount_7d", dtype=Float64),
        Field(name="avg_txn_amount_30d", dtype=Float64),
        Field(name="max_txn_amount_7d", dtype=Float64),
        Field(name="std_txn_amount_30d", dtype=Float64),
        Field(name="unique_merchants_7d", dtype=Int64),
        Field(name="txn_frequency_change", dtype=Float64),  # This week vs last week
    ],
    source=transaction_stats_source,
    online=True,     # Materialize to online store for real-time serving
    tags={"team": "fraud", "refresh": "hourly"}
)

merchant_profile_fv = FeatureView(
    name="merchant_profile",
    entities=[merchant],
    ttl=timedelta(days=30),
    schema=[
        Field(name="merchant_category", dtype=String),
        Field(name="merchant_risk_score", dtype=Float64),
        Field(name="avg_txn_amount", dtype=Float64),
        Field(name="chargeback_rate", dtype=Float64),
    ],
    source=merchant_features_source,
    online=True,
    tags={"team": "fraud"}
)


# --- Define Feature Service ---
# A Feature Service bundles feature views for a specific model

from feast import FeatureService

fraud_detection_service = FeatureService(
    name="fraud_detection",
    features=[
        user_profile_fv,
        user_transaction_stats_fv,
        merchant_profile_fv
    ],
    description="Features for the fraud detection model v2",
    tags={"model": "fraud-v2", "owner": "fraud-team"}
)
```

### Feature Store Configuration

```yaml
# feature_repo/feature_store.yaml
project: fraud_detection
provider: local  # Options: local, gcp, aws

registry: data/registry.db  # Where feature metadata is stored

# Offline store configuration
offline_store:
  type: file  # Options: file, bigquery, snowflake, redshift

# Online store configuration  
online_store:
  type: redis     # Options: sqlite, redis, dynamodb, datastore
  connection_string: "localhost:6379"
  # For production:
  # type: redis
  # connection_string: "redis-cluster.example.com:6379,password=xxx"

# For GCP:
# provider: gcp
# offline_store:
#   type: bigquery
#   project: my-gcp-project
#   dataset: feast_features
# online_store:
#   type: datastore
#   project: my-gcp-project

# For AWS:
# provider: aws
# offline_store:
#   type: redshift
#   cluster_id: feast-cluster
#   region: us-east-1
#   database: feast
#   user: admin
# online_store:
#   type: dynamodb
#   region: us-east-1
```

### Code: Using Feast for Training

```python
# training_with_feast.py
"""
Use Feast to generate point-in-time correct training datasets.
"""
import pandas as pd
from feast import FeatureStore
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report

# ============================================
# STEP 1: Connect to Feature Store
# ============================================

# Initialize Feast (points to feature_store.yaml)
store = FeatureStore(repo_path="feature_repo/")


# ============================================
# STEP 2: Create Entity DataFrame
# ============================================

# This defines WHAT entities and WHEN you want features for
# The "event_timestamp" is crucial for point-in-time correctness

entity_df = pd.DataFrame({
    "user_id": [1001, 1002, 1003, 1004, 1005, 1001, 1002],
    "merchant_id": [501, 502, 501, 503, 504, 502, 503],
    "event_timestamp": pd.to_datetime([
        "2024-01-15 10:00:00",  # Feature values as of this time
        "2024-01-15 11:00:00",
        "2024-01-16 09:00:00",
        "2024-01-16 14:00:00",
        "2024-01-17 08:00:00",
        "2024-02-01 10:00:00",  # Same user, different time → different features
        "2024-02-01 11:00:00",
    ]),
    "label": [0, 1, 0, 0, 1, 0, 1]  # 0=legitimate, 1=fraud
})

print(f"Entity DataFrame:\n{entity_df}")


# ============================================
# STEP 3: Get Historical Features (Training)
# ============================================

# This joins features to the entity DataFrame with point-in-time correctness
# For each row, it looks up features that were available AT that timestamp
# (no future data leakage!)

training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        # Specify which features to retrieve
        "user_profile:user_age",
        "user_profile:account_age_days",
        "user_profile:is_verified",
        "user_transaction_stats:txn_count_7d",
        "user_transaction_stats:avg_txn_amount_7d",
        "user_transaction_stats:max_txn_amount_7d",
        "user_transaction_stats:unique_merchants_7d",
        "user_transaction_stats:txn_frequency_change",
        "merchant_profile:merchant_risk_score",
        "merchant_profile:chargeback_rate",
    ]
).to_df()

# Or use the Feature Service (recommended):
# training_df = store.get_historical_features(
#     entity_df=entity_df,
#     features=store.get_feature_service("fraud_detection")
# ).to_df()

print(f"\nTraining DataFrame shape: {training_df.shape}")
print(f"Columns: {training_df.columns.tolist()}")
print(training_df.head())


# ============================================
# STEP 4: Train Model
# ============================================

# Prepare features
feature_columns = [
    'user_age', 'account_age_days', 'is_verified',
    'txn_count_7d', 'avg_txn_amount_7d', 'max_txn_amount_7d',
    'unique_merchants_7d', 'txn_frequency_change',
    'merchant_risk_score', 'chargeback_rate'
]

# Handle missing values (some features might not exist for all timestamps)
training_df[feature_columns] = training_df[feature_columns].fillna(0)

X = training_df[feature_columns]
y = training_df['label']

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# Train
model = GradientBoostingClassifier(n_estimators=100, random_state=42)
model.fit(X_train, y_train)

# Evaluate
y_pred = model.predict(X_test)
print(f"\n{classification_report(y_test, y_pred)}")


# ============================================
# STEP 5: Materialize to Online Store
# ============================================

# This copies features from offline store → online store
# So they can be served at low latency for real-time predictions

from datetime import datetime

store.materialize(
    start_date=datetime(2024, 1, 1),
    end_date=datetime(2024, 12, 31)
)
# Or materialize incrementally (only new data):
# store.materialize_incremental(end_date=datetime.now())

print("Features materialized to online store!")
```

### Code: Using Feast for Real-Time Serving

```python
# serving_with_feast.py
"""
Serve features in real-time for online predictions.
"""
from feast import FeatureStore
import numpy as np

store = FeatureStore(repo_path="feature_repo/")

# ============================================
# Real-time Feature Retrieval
# ============================================

def predict_fraud(user_id: int, merchant_id: int, model) -> dict:
    """
    Real-time fraud prediction using features from the online store.
    
    Args:
        user_id: User making the transaction
        merchant_id: Merchant receiving the transaction
        model: Trained ML model
        
    Returns:
        Prediction result with probability
    """
    # Get features from online store (millisecond latency!)
    feature_vector = store.get_online_features(
        features=[
            "user_profile:user_age",
            "user_profile:account_age_days",
            "user_profile:is_verified",
            "user_transaction_stats:txn_count_7d",
            "user_transaction_stats:avg_txn_amount_7d",
            "user_transaction_stats:max_txn_amount_7d",
            "user_transaction_stats:unique_merchants_7d",
            "user_transaction_stats:txn_frequency_change",
            "merchant_profile:merchant_risk_score",
            "merchant_profile:chargeback_rate",
        ],
        entity_rows=[
            {"user_id": user_id, "merchant_id": merchant_id}
        ]
    ).to_dict()
    
    # Convert to model input
    features = np.array([[
        feature_vector["user_age"][0] or 0,
        feature_vector["account_age_days"][0] or 0,
        feature_vector["is_verified"][0] or 0,
        feature_vector["txn_count_7d"][0] or 0,
        feature_vector["avg_txn_amount_7d"][0] or 0.0,
        feature_vector["max_txn_amount_7d"][0] or 0.0,
        feature_vector["unique_merchants_7d"][0] or 0,
        feature_vector["txn_frequency_change"][0] or 0.0,
        feature_vector["merchant_risk_score"][0] or 0.5,
        feature_vector["chargeback_rate"][0] or 0.0,
    ]])
    
    # Predict
    probability = model.predict_proba(features)[0][1]
    prediction = int(probability > 0.5)
    
    return {
        "is_fraud": prediction,
        "fraud_probability": round(float(probability), 4),
        "features_used": {k: v[0] for k, v in feature_vector.items()},
        "user_id": user_id,
        "merchant_id": merchant_id
    }


# FastAPI integration
"""
from fastapi import FastAPI
import joblib

app = FastAPI()
model = joblib.load("models/fraud_model.pkl")
store = FeatureStore(repo_path="feature_repo/")

@app.post("/predict")
async def predict(user_id: int, merchant_id: int, amount: float):
    result = predict_fraud(user_id, merchant_id, model)
    return result
"""
```

### CLI Commands

```bash
# Initialize a new feature repository
feast init my_feature_repo

# Apply feature definitions (register with the registry)
cd my_feature_repo
feast apply

# List all registered features
feast feature-views list
feast entities list
feast feature-services list

# Materialize features to the online store
feast materialize 2024-01-01T00:00:00 2024-12-31T23:59:59

# Materialize only new data since last materialization
feast materialize-incremental $(date -u +"%Y-%m-%dT%H:%M:%S")

# View feature view schema
feast feature-views describe user_transaction_stats

# Serve features via HTTP (starts a REST server)
feast serve -h 0.0.0.0 -p 6566

# Tear down (remove all infrastructure)
feast teardown
```

---

## Tecton — Managed Feature Platform

### What is Tecton?

Tecton is an enterprise-grade managed feature platform (built by the creators of Uber's Michelangelo). It handles the hard parts that open-source Feast doesn't:
- **Managed infrastructure** — no Redis/Spark to manage
- **Stream processing** — real-time features from Kafka/Kinesis
- **Feature monitoring** — automated drift/quality alerts
- **Governance** — access control, audit logs, compliance

### Tecton vs Feast

| Aspect | Feast (Open Source) | Tecton (Managed) |
|--------|-------------------|-----------------|
| **Cost** | Free (+ infra costs) | Paid subscription |
| **Setup effort** | Medium-High | Low |
| **Stream features** | Basic/manual | Built-in (Spark/Flink) |
| **Monitoring** | DIY | Built-in dashboards |
| **Governance** | Basic | Enterprise-grade (RBAC, audit) |
| **Scale** | Depends on your infra | Handles automatically |
| **Support** | Community | Enterprise support |
| **Best for** | Small teams, startups | Enterprise, regulated industries |

### Tecton Example (Conceptual)

```python
# Tecton feature definitions (conceptual — requires Tecton SDK)
"""
from tecton import Entity, BatchSource, StreamSource, FeatureView, Aggregation
from tecton.types import Float64, String, Timestamp

# Entity
user = Entity(name="user", join_keys=["user_id"])

# Stream source (real-time events from Kafka)
transactions_stream = StreamSource(
    name="transactions",
    stream_config=KafkaConfig(
        kafka_bootstrap_servers="kafka:9092",
        topics="transactions",
        timestamp_field="timestamp"
    ),
    batch_config=HiveConfig(
        database="analytics",
        table="transactions",
        timestamp_field="timestamp"
    )
)

# Real-time aggregation feature view
user_txn_features = FeatureView(
    name="user_txn_features",
    entities=[user],
    mode="streaming",
    source=transactions_stream,
    aggregation_interval=timedelta(minutes=5),  # Update every 5 min
    aggregations=[
        Aggregation(column="amount", function="mean", time_windows=["1h", "24h", "7d"]),
        Aggregation(column="amount", function="max", time_windows=["1h", "24h"]),
        Aggregation(column="amount", function="count", time_windows=["1h", "24h", "7d"]),
        Aggregation(column="merchant_id", function="approx_count_distinct", time_windows=["24h"]),
    ],
    online=True,
    offline=True,
    feature_start_time=datetime(2024, 1, 1),
    tags={"team": "fraud", "tier": "critical"},
    owner="fraud-team@company.com"
)

# Get online features
features = tecton.get_online_features(
    feature_service_name="fraud_detection",
    join_keys={"user_id": "12345"},
    request_context_map={"transaction_amount": 500.0}
).to_dict()
"""
```

### Other Feature Store Alternatives

| Platform | Type | Best For |
|----------|------|----------|
| **Feast** | Open Source | Startups, custom deployments |
| **Tecton** | Managed | Enterprise, real-time features |
| **Databricks Feature Store** | Managed | Databricks users |
| **SageMaker Feature Store** | Managed | AWS-native teams |
| **Vertex AI Feature Store** | Managed | GCP-native teams |
| **Hopsworks** | Open Source / Managed | Python-first teams |
| **Featureform** | Open Source | Infrastructure abstraction |

---

## Feature Pipelines and Engineering

### Feature Pipeline Patterns

```
PATTERN 1: BATCH FEATURES (most common)
─────────────────────────────────────────
  Raw Data → SQL/Spark Job → Feature Table → Online Store
  Schedule: Hourly / Daily
  Example: avg_purchase_30d, user_segment, credit_score

PATTERN 2: STREAMING FEATURES (real-time)
─────────────────────────────────────────
  Event Stream → Flink/Spark → Aggregation → Online Store
  Latency: Seconds to minutes
  Example: txn_count_last_5min, current_session_pages

PATTERN 3: ON-DEMAND FEATURES (computed at request time)
─────────────────────────────────────────
  Request arrives → Compute on-the-fly → Combine with stored features
  Latency: Added at serving time
  Example: time_since_last_login, distance_to_merchant

PATTERN 4: HYBRID (most production systems)
─────────────────────────────────────────
  Batch features (daily) + Streaming features (real-time) 
  + On-demand features (request-time)
  → Combined feature vector → Model prediction
```

### Code: Building Feature Pipelines

```python
# feature_pipelines.py
"""
Feature pipeline implementations for different patterns.
"""
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
from typing import Dict, List, Optional

# ============================================
# BATCH FEATURE PIPELINE
# ============================================

class BatchFeaturePipeline:
    """
    Batch feature computation pipeline.
    Runs on schedule (hourly/daily) to pre-compute features.
    """
    
    def __init__(self, db_connection):
        self.db = db_connection
    
    def compute_user_features(self, as_of_date: datetime) -> pd.DataFrame:
        """
        Compute user-level features as of a specific date.
        
        Point-in-time correctness: Only uses data available BEFORE as_of_date.
        """
        query = f"""
        WITH user_transactions AS (
            SELECT
                user_id,
                amount,
                merchant_id,
                transaction_date,
                is_fraud
            FROM transactions
            WHERE transaction_date <= '{as_of_date.isoformat()}'
        ),
        
        user_stats_7d AS (
            SELECT
                user_id,
                COUNT(*) as txn_count_7d,
                AVG(amount) as avg_txn_amount_7d,
                MAX(amount) as max_txn_amount_7d,
                STDDEV(amount) as std_txn_amount_7d,
                COUNT(DISTINCT merchant_id) as unique_merchants_7d
            FROM user_transactions
            WHERE transaction_date >= '{(as_of_date - timedelta(days=7)).isoformat()}'
            GROUP BY user_id
        ),
        
        user_stats_30d AS (
            SELECT
                user_id,
                COUNT(*) as txn_count_30d,
                AVG(amount) as avg_txn_amount_30d,
                SUM(amount) as total_spend_30d,
                COUNT(DISTINCT merchant_id) as unique_merchants_30d
            FROM user_transactions
            WHERE transaction_date >= '{(as_of_date - timedelta(days=30)).isoformat()}'
            GROUP BY user_id
        ),
        
        user_fraud_history AS (
            SELECT
                user_id,
                SUM(CASE WHEN is_fraud = 1 THEN 1 ELSE 0 END) as fraud_count_total,
                AVG(CASE WHEN is_fraud = 1 THEN 1.0 ELSE 0.0 END) as fraud_rate
            FROM user_transactions
            GROUP BY user_id
        )
        
        SELECT
            u7.user_id,
            '{as_of_date.isoformat()}' as event_timestamp,
            
            -- 7-day features
            COALESCE(u7.txn_count_7d, 0) as txn_count_7d,
            COALESCE(u7.avg_txn_amount_7d, 0) as avg_txn_amount_7d,
            COALESCE(u7.max_txn_amount_7d, 0) as max_txn_amount_7d,
            COALESCE(u7.std_txn_amount_7d, 0) as std_txn_amount_7d,
            COALESCE(u7.unique_merchants_7d, 0) as unique_merchants_7d,
            
            -- 30-day features
            COALESCE(u30.txn_count_30d, 0) as txn_count_30d,
            COALESCE(u30.avg_txn_amount_30d, 0) as avg_txn_amount_30d,
            COALESCE(u30.total_spend_30d, 0) as total_spend_30d,
            COALESCE(u30.unique_merchants_30d, 0) as unique_merchants_30d,
            
            -- Derived features
            CASE 
                WHEN COALESCE(u30.txn_count_30d, 0) > 0 
                THEN COALESCE(u7.txn_count_7d, 0) * 4.0 / u30.txn_count_30d
                ELSE 1.0
            END as txn_frequency_change,  -- This week vs monthly average
            
            -- Fraud history
            COALESCE(uf.fraud_count_total, 0) as fraud_count_total,
            COALESCE(uf.fraud_rate, 0) as fraud_rate
            
        FROM user_stats_7d u7
        LEFT JOIN user_stats_30d u30 ON u7.user_id = u30.user_id
        LEFT JOIN user_fraud_history uf ON u7.user_id = uf.user_id
        """
        
        return pd.read_sql(query, self.db)
    
    def run(self, as_of_date: Optional[datetime] = None):
        """Execute the batch pipeline."""
        if as_of_date is None:
            as_of_date = datetime.now()
        
        print(f"Computing features as of {as_of_date}")
        
        # Compute features
        user_features = self.compute_user_features(as_of_date)
        
        # Write to offline store (e.g., parquet files or data warehouse)
        output_path = f"data/user_features_{as_of_date.strftime('%Y%m%d')}.parquet"
        user_features.to_parquet(output_path, index=False)
        
        print(f"Wrote {len(user_features)} feature rows to {output_path}")
        return user_features


# ============================================
# STREAMING FEATURE PIPELINE (Conceptual)
# ============================================

"""
# Using Apache Flink for real-time feature computation

from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment

env = StreamExecutionEnvironment.get_execution_environment()
t_env = StreamTableEnvironment.create(env)

# Define Kafka source
t_env.execute_sql('''
    CREATE TABLE transactions (
        user_id BIGINT,
        amount DOUBLE,
        merchant_id BIGINT,
        event_time TIMESTAMP(3),
        WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
    ) WITH (
        'connector' = 'kafka',
        'topic' = 'transactions',
        'properties.bootstrap.servers' = 'kafka:9092',
        'format' = 'json'
    )
''')

# Compute streaming aggregations
t_env.execute_sql('''
    CREATE TABLE user_features_sink (
        user_id BIGINT,
        window_start TIMESTAMP,
        txn_count_5min BIGINT,
        avg_amount_5min DOUBLE,
        max_amount_5min DOUBLE,
        PRIMARY KEY (user_id) NOT ENFORCED
    ) WITH (
        'connector' = 'upsert-kafka',  -- or 'redis'
        'topic' = 'user-features',
        'key.format' = 'json',
        'value.format' = 'json'
    )
''')

# Tumbling window aggregation
t_env.execute_sql('''
    INSERT INTO user_features_sink
    SELECT
        user_id,
        TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_start,
        COUNT(*) as txn_count_5min,
        AVG(amount) as avg_amount_5min,
        MAX(amount) as max_amount_5min
    FROM transactions
    GROUP BY
        user_id,
        TUMBLE(event_time, INTERVAL '5' MINUTE)
''')
"""


# ============================================
# ON-DEMAND FEATURE COMPUTATION
# ============================================

class OnDemandFeatures:
    """
    Features computed at request time.
    These depend on the request context (not pre-computable).
    """
    
    @staticmethod
    def compute(
        request_data: Dict,
        stored_features: Dict
    ) -> Dict:
        """
        Compute on-demand features at prediction time.
        
        Args:
            request_data: Current transaction details
            stored_features: Pre-computed features from feature store
        """
        on_demand = {}
        
        # Time-based features
        current_hour = datetime.now().hour
        on_demand['is_night_transaction'] = int(current_hour < 6 or current_hour > 22)
        on_demand['is_weekend'] = int(datetime.now().weekday() >= 5)
        
        # Transaction vs history comparison
        avg_amount = stored_features.get('avg_txn_amount_30d', 0)
        current_amount = request_data.get('amount', 0)
        
        if avg_amount > 0:
            on_demand['amount_vs_avg_ratio'] = current_amount / avg_amount
        else:
            on_demand['amount_vs_avg_ratio'] = 1.0
        
        # Is this amount unusually high?
        max_amount = stored_features.get('max_txn_amount_7d', 0)
        on_demand['exceeds_7d_max'] = int(current_amount > max_amount)
        
        # Velocity check (requests in last N minutes — from a cache/counter)
        on_demand['requests_last_5min'] = request_data.get('recent_request_count', 0)
        
        return on_demand


# ============================================
# COMBINED FEATURE RETRIEVAL
# ============================================

def get_full_feature_vector(
    user_id: int,
    merchant_id: int,
    transaction: Dict,
    feature_store,  # Feast FeatureStore
    model
) -> Dict:
    """
    Combine all feature sources for a complete prediction.
    
    This is what happens in production at prediction time:
    1. Fetch pre-computed features from online store (fast)
    2. Compute on-demand features from request context
    3. Combine into a single feature vector
    4. Run model prediction
    """
    # Step 1: Get stored features (batch + streaming)
    stored_features = feature_store.get_online_features(
        features=[
            "user_transaction_stats:txn_count_7d",
            "user_transaction_stats:avg_txn_amount_7d",
            "user_transaction_stats:max_txn_amount_7d",
            "user_transaction_stats:unique_merchants_7d",
            "merchant_profile:merchant_risk_score",
            "merchant_profile:chargeback_rate",
        ],
        entity_rows=[{"user_id": user_id, "merchant_id": merchant_id}]
    ).to_dict()
    
    # Flatten: {feature_name: [value]} → {feature_name: value}
    stored = {k: v[0] for k, v in stored_features.items()}
    
    # Step 2: Compute on-demand features
    on_demand = OnDemandFeatures.compute(transaction, stored)
    
    # Step 3: Combine all features
    full_features = {**stored, **on_demand}
    
    return full_features
```

---

## Feature Store Design Patterns

### Pattern 1: Shared Feature Catalog

```
Multiple models reuse the same features:

Feature Catalog:
├── user_profile (age, country, verified)
│   ├── Used by: fraud_model ✓
│   ├── Used by: recommendation_model ✓
│   └── Used by: churn_model ✓
│
├── user_transaction_stats (txn_count, avg_amount)
│   ├── Used by: fraud_model ✓
│   └── Used by: credit_risk_model ✓
│
└── merchant_profile (risk_score, category)
    └── Used by: fraud_model ✓

Benefit: Compute once, use everywhere.
One definition = consistency across all models.
```

### Pattern 2: Feature Versioning

```python
# Version features when the computation logic changes

# v1: Simple average
user_spend_v1 = FeatureView(
    name="user_spend_v1",
    # avg_spend = mean(all transactions)
    ...
)

# v2: Weighted average (more recent transactions weighted higher)
user_spend_v2 = FeatureView(
    name="user_spend_v2",
    # avg_spend = weighted_mean(transactions, weights=recency)
    ...
)

# Models can pin to specific versions:
# fraud_model_v1 uses user_spend_v1
# fraud_model_v2 uses user_spend_v2
# Both can run simultaneously during A/B testing
```

### Pattern 3: Feature Freshness Tiers

```
TIER 1: Real-time (seconds)
  ├── txn_count_last_5min (streaming pipeline)
  ├── current_session_pages (streaming pipeline)
  └── Stored in: Redis with 5-min TTL

TIER 2: Near-real-time (minutes to hours)
  ├── txn_count_7d (hourly batch job)
  ├── avg_purchase_amount (hourly batch job)
  └── Stored in: Redis with 1-hour TTL

TIER 3: Batch (daily)
  ├── user_segment (daily batch job)
  ├── credit_score (daily batch job)
  └── Stored in: Redis with 24-hour TTL

TIER 4: Slow-changing (weekly/monthly)
  ├── user_age (profile update)
  ├── account_type (rarely changes)
  └── Stored in: Redis with 7-day TTL
```

---

## Common Mistakes

### 1. Training-Serving Skew (The #1 Feature Store Problem)

```python
# ❌ WRONG: Different feature computation in training vs serving

# Training (Jupyter notebook):
df['avg_amount'] = df.groupby('user_id')['amount'].transform('mean')
# Uses Pandas groupby — includes ALL data (future leakage!)

# Serving (production API):
avg_amount = redis.get(f"user:{user_id}:avg_amount")
# Uses pre-computed value from different pipeline — may differ!

# ✅ RIGHT: Use the feature store for BOTH training and serving
# Training:
training_df = store.get_historical_features(entity_df, features=...)
# Serving:
serving_features = store.get_online_features(features=..., entity_rows=...)
# Same computation, same code path → no skew!
```

### 2. Not Handling Feature Freshness

```python
# ❌ WRONG: Serving stale features without checking

features = store.get_online_features(
    features=["user_stats:txn_count_7d"],
    entity_rows=[{"user_id": 123}]
)
# What if this feature hasn't been updated in 3 days?
# Your model thinks user had 5 transactions, but they actually had 50!

# ✅ RIGHT: Check freshness and handle stale features
features = store.get_online_features(...)
metadata = features.metadata  # Check when features were last updated

if (datetime.now() - metadata.last_updated).hours > 2:
    # Option A: Use fallback/default values
    # Option B: Compute on-the-fly
    # Option C: Return a lower-confidence prediction
    # Option D: Alert the ops team
    log.warning(f"Stale features for user {user_id}, last updated {metadata.last_updated}")
```

### 3. Too Many Features in One Feature View

```python
# ❌ WRONG: Monolithic feature view with 100 features
all_user_features = FeatureView(
    name="all_user_features",
    schema=[
        Field(name="feature_1", ...),
        # ... 100 features with different update schedules
        Field(name="feature_100", ...),
    ],
    ttl=timedelta(hours=1)  # All features on same schedule? Bad!
)

# ✅ RIGHT: Split by domain and freshness
user_profile_fv = FeatureView(name="user_profile", ttl=timedelta(days=7), ...)       # Slow-changing
user_activity_fv = FeatureView(name="user_activity", ttl=timedelta(hours=1), ...)     # Hourly
user_realtime_fv = FeatureView(name="user_realtime", ttl=timedelta(minutes=5), ...)   # Near real-time
```

### 4. Ignoring Point-in-Time Correctness

```python
# ❌ WRONG: Just join features on user_id without considering time
training_df = users_df.merge(features_df, on='user_id')
# This may use FUTURE features for past labels → data leakage!

# ✅ RIGHT: Always use point-in-time joins
training_df = store.get_historical_features(
    entity_df=entity_df_with_timestamps,  # Must include event_timestamp!
    features=["user_stats:txn_count_7d"]
)
# Feast automatically ensures features are from BEFORE the event_timestamp
```

### 5. Not Planning for Schema Evolution

```python
# ❌ WRONG: Renaming/removing features without migration plan
# Monday: "user_income" feature exists, 5 models use it
# Tuesday: Rename to "user_annual_income" 
# Result: 5 models break in production!

# ✅ RIGHT: Versioning and deprecation strategy
# 1. Add new feature alongside old one
# 2. Migrate models one by one
# 3. Deprecate old feature (add warning)
# 4. Remove old feature after all models migrated

user_profile_v2 = FeatureView(
    name="user_profile_v2",
    schema=[
        Field(name="user_annual_income", dtype=Float64),   # New name
        Field(name="user_income", dtype=Float64),           # Keep old (deprecated)
    ],
    tags={"deprecated_fields": "user_income"}
)
```

---

## Interview Questions

### Conceptual

**Q1: What is a feature store and why do we need one?**
> **A:** A feature store is a centralized system for managing ML features across their lifecycle — computation, storage, serving, and monitoring. We need one because: (1) It eliminates training-serving skew by using the same feature definitions for both, (2) It enables feature reuse across models and teams, (3) It provides point-in-time correct training data (preventing data leakage), (4) It serves pre-computed features at low latency for real-time predictions, (5) It provides governance — who computed what, when, and which models use which features.

**Q2: Explain the difference between online and offline feature stores.**
> **A:** The offline store (BigQuery, S3, Snowflake) holds historical feature values for training — supports large batch queries over months/years of data with seconds-to-minutes latency. The online store (Redis, DynamoDB) holds only the latest feature values for real-time serving — supports point lookups by entity key with 1-10ms latency. They're synced via "materialization" — a process that copies the latest feature values from offline to online store. Both use the same feature definitions, ensuring consistency.

**Q3: What is training-serving skew and how does a feature store prevent it?**
> **A:** Training-serving skew occurs when features used during training differ from those used during serving. Common causes: (1) Different code paths — Pandas in notebooks vs SQL in production, (2) Different data sources — training uses data warehouse, serving computes on-the-fly, (3) Different timing — training uses batch data, serving uses real-time data. A feature store prevents this by having a single feature definition used for both `get_historical_features()` (training) and `get_online_features()` (serving). Same computation logic, same data source.

**Q4: What is point-in-time correctness and why does it matter?**
> **A:** Point-in-time correctness means that when creating training data, you only use features that were actually available at the time of each training example. Without it, you get data leakage — using future information to predict the past, which inflates training metrics but fails in production. For example, if predicting whether a user will churn on Jan 15, you should only use features computed from data available before Jan 15, not after. Feature stores handle this automatically through timestamp-based joins.

**Q5: Compare Feast, Tecton, and cloud-provider feature stores.**
> **A:** Feast is open-source, free, and flexible — good for teams that want control and don't need managed streaming features. Tecton is enterprise-managed with built-in streaming, monitoring, and governance — ideal for organizations with real-time feature needs and compliance requirements. Cloud feature stores (SageMaker, Vertex AI) are tightly integrated with their ecosystems — best when you're already all-in on that cloud. Feast for flexibility and cost, Tecton for enterprise features, cloud-native for ecosystem integration.

### Scenario-Based

**Q6: You're building a fraud detection system. Design the feature store architecture.**
> **A:** Three feature tiers: (1) Batch features (daily): user demographics, historical fraud rate, account age — stored in BigQuery + Redis. (2) Near-real-time features (5-min windows): transaction count last hour, average amount last hour, unique merchants last hour — Spark Streaming → Redis. (3) On-demand features: time since last transaction, distance from last transaction location, amount vs. historical average — computed at request time. Use Feast or Tecton with Redis online store. Materialization runs every 5 minutes for streaming features, daily for batch. Monitor feature freshness with alerts if >10 minutes stale.

**Q7: Your team has 5 data scientists building 10 models. How do you set up a feature store?**
> **A:** (1) Start with Feast (open source, low barrier). (2) Establish a shared feature repository in Git — all feature definitions are code-reviewed. (3) Create Feature Services per model — each model declares which features it needs. (4) Set up a materialization pipeline (Airflow/cron) to keep online store fresh. (5) CI/CD: Feature definition changes go through PR review, `feast apply` runs in CI. (6) Documentation: Each feature has description, owner, and freshness SLA. (7) Governance: Dashboard showing feature usage across models (which models break if a feature changes).

---

## Quick Reference

### Feature Store Comparison Table

| Feature | Feast | Tecton | SageMaker FS | Vertex AI FS |
|---------|-------|--------|-------------|-------------|
| **Type** | Open Source | Managed | Managed | Managed |
| **Offline Store** | File/BQ/Snowflake/Redshift | Databricks/Snowflake | S3 | BigQuery |
| **Online Store** | Redis/DynamoDB/SQLite | DynamoDB/Redis | DynamoDB | Bigtable |
| **Streaming** | Basic | Built-in (Spark/Flink) | Via Kinesis | Via Dataflow |
| **Point-in-Time** | Yes | Yes | Yes | Yes |
| **Monitoring** | DIY | Built-in | Basic | Basic |
| **RBAC** | Basic | Enterprise | IAM | IAM |
| **Best For** | Startups, custom | Enterprise | AWS teams | GCP teams |

### Feature Engineering Cheat Sheet

| Feature Category | Examples | Update Frequency |
|-----------------|----------|-----------------|
| **Identity** | user_id, account_type | Rarely changes |
| **Demographic** | age, country, income_bracket | Monthly |
| **Behavioral (batch)** | avg_purchase_30d, login_frequency | Daily/Hourly |
| **Behavioral (stream)** | txn_count_5min, pages_last_session | Real-time |
| **Temporal** | is_weekend, hour_of_day, is_holiday | On-demand |
| **Aggregate** | total_spend, lifetime_orders | Daily |
| **Ratio/Derived** | amount_vs_avg, frequency_change | Computed from others |
| **Cross-entity** | user_merchant_txn_count | Daily |

### Feast Commands Quick Reference

| Command | Purpose |
|---------|---------|
| `feast init` | Create new feature repo |
| `feast apply` | Register feature definitions |
| `feast materialize` | Sync offline → online store |
| `feast materialize-incremental` | Sync only new data |
| `feast feature-views list` | List all feature views |
| `feast serve` | Start feature serving HTTP server |
| `feast teardown` | Remove all infrastructure |

### Key Formulas

**Feature Freshness SLA:**

$$\text{Freshness} = t_{current} - t_{last\_materialized}$$

$$\text{SLA Met} = \text{Freshness} \leq \text{TTL}$$

**Feature Coverage:**

$$\text{Coverage} = \frac{\text{Entities with non-null features}}{\text{Total entities requested}} \times 100\%$$

Target: >95% coverage for critical features.

**Feature Drift (Population Stability Index):**

$$PSI = \sum_{i=1}^{n} (P_i - Q_i) \times \ln\left(\frac{P_i}{Q_i}\right)$$

Where $P$ = expected distribution, $Q$ = actual distribution.
- PSI < 0.1: No significant drift
- 0.1 ≤ PSI < 0.2: Moderate drift (investigate)
- PSI ≥ 0.2: Significant drift (retrain)

---

> **Pro Tip:** Start simple. Many teams try to build a feature store before they need one. If you have < 3 models and < 5 data scientists, a well-organized set of SQL queries + a Redis cache may be all you need. Graduate to Feast when you feel the pain of duplicated feature logic and training-serving skew.
