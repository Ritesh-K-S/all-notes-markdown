# MLOps Fundamentals

## Table of Contents
- [What is MLOps](#what-is-mlops)
- [Why MLOps Matters](#why-mlops-matters)
- [The ML Lifecycle](#the-ml-lifecycle)
- [DevOps vs MLOps](#devops-vs-mlops)
- [MLOps Maturity Levels](#mlops-maturity-levels)
- [Key Components of MLOps](#key-components-of-mlops)
- [MLOps Team Roles](#mlops-team-roles)
- [Tools Landscape](#tools-landscape)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is MLOps

### Simple Explanation

Imagine you built an amazing recipe (ML model) in your kitchen (Jupyter notebook). It tastes great at home. But now you want to open a restaurant that serves 10,000 customers daily, keeps the food quality consistent, adapts to changing tastes, and never shuts down.

**MLOps** (Machine Learning Operations) is the set of practices, tools, and culture that takes your ML model from a "cool experiment" to a **reliable, scalable, production system** that delivers real business value.

### Formal Definition

> **MLOps** is a set of practices at the intersection of Machine Learning, DevOps, and Data Engineering that aims to deploy and maintain ML systems in production reliably and efficiently.

### The Core Problem MLOps Solves

```
┌─────────────────────────────────────────────────────────┐
│                    THE ML GAP                            │
│                                                         │
│   Notebook Prototype ──── ??? ────► Production System   │
│   (works on my machine)           (serves millions)     │
│                                                         │
│   87% of ML projects NEVER make it to production        │
│   (Source: Gartner, VentureBeat Research)               │
└─────────────────────────────────────────────────────────┘
```

This gap exists because:
1. **ML code is only ~5-10%** of a production ML system
2. The remaining 90% is infrastructure, monitoring, data pipelines, and operations
3. Traditional software engineering practices don't fully address ML-specific challenges

### The Hidden Technical Debt in ML Systems

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌──────────┐ ┌───────────────┐ ┌──────────────────────────┐  │
│  │ Config   │ │ Data          │ │ Feature                  │  │
│  │ Mgmt     │ │ Collection    │ │ Extraction               │  │
│  ├──────────┤ ├───────────────┤ ├──────────────────────────┤  │
│  │ Data     │ │               │ │ Process                  │  │
│  │ Verify   │ │  ML CODE      │ │ Management               │  │
│  ├──────────┤ │  (tiny box)   │ ├──────────────────────────┤  │
│  │ Machine  │ │               │ │ Serving                  │  │
│  │ Resource │ ├───────────────┤ │ Infrastructure           │  │
│  │ Mgmt     │ │ Analysis      │ ├──────────────────────────┤  │
│  ├──────────┤ │ Tools         │ │ Monitoring               │  │
│  │ Testing  │ │               │ │                          │  │
│  └──────────┘ └───────────────┘ └──────────────────────────┘  │
│                                                                │
│  (From Google's "Hidden Technical Debt in ML Systems" paper)   │
└────────────────────────────────────────────────────────────────┘
```

---

## Why MLOps Matters

### Real-World Relevance

| Scenario | Without MLOps | With MLOps |
|----------|---------------|------------|
| Model update | Manual notebook re-run, manual deploy (days) | Automated pipeline, one-click deploy (hours) |
| Data quality issue | Discovered after model degrades | Caught by automated data validation |
| Model performance drop | Users complain weeks later | Alert fires within minutes |
| Compliance audit | "I think we used this dataset..." | Full lineage and version tracking |
| Team collaboration | "Which model version is in prod?" | Clear registry with metadata |
| Scaling | Crashes under load | Auto-scales based on traffic |

### Business Impact

- **Uber**: Manages 10,000+ models in production using Michelangelo (internal MLOps platform)
- **Netflix**: Deploys hundreds of models for recommendations, A/B tests continuously
- **Spotify**: Updates recommendation models daily with automated pipelines
- **Google**: Serves billions of predictions per day across products

### When You'd Use MLOps

✅ **Use MLOps when:**
- Your model needs to serve real users/systems
- Data changes over time (most real-world scenarios)
- Multiple team members work on ML projects
- You need reproducibility and auditability
- Business decisions depend on model predictions

❌ **Don't over-engineer when:**
- You're doing one-time analysis/research
- Proof of concept stage (but plan for it!)
- Small personal projects with no production needs

---

## The ML Lifecycle

### How It Works

The ML lifecycle is not linear — it's a continuous loop. Unlike traditional software where you "build and ship," ML systems need constant care because the world (data) keeps changing.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ML LIFECYCLE (Iterative)                      │
│                                                                     │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │
│    │ Business │    │  Data    │    │  Model   │    │  Model   │   │
│    │ Problem  │───►│ Pipeline │───►│ Develop  │───►│ Evaluate │   │
│    │ Framing  │    │          │    │          │    │          │   │
│    └──────────┘    └──────────┘    └──────────┘    └─────┬────┘   │
│         ▲                                                 │        │
│         │                                                 ▼        │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │
│    │ Retrain  │    │ Monitor  │    │  Model   │    │  Model   │   │
│    │ Trigger  │◄───│ & Alert  │◄───│ Serving  │◄───│ Deploy   │   │
│    │          │    │          │    │          │    │          │   │
│    └──────────┘    └──────────┘    └──────────┘    └──────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Phase Breakdown

#### Phase 1: Business Problem Framing
- Define success metrics (not just accuracy — business KPIs)
- Determine latency requirements (real-time vs batch)
- Identify data sources and constraints

#### Phase 2: Data Pipeline
- Data ingestion from multiple sources
- Data validation and quality checks
- Feature engineering and transformation
- Data versioning

#### Phase 3: Model Development
- Experiment with algorithms
- Hyperparameter tuning
- Feature selection
- Track all experiments

#### Phase 4: Model Evaluation
- Offline evaluation (test sets, cross-validation)
- Fairness and bias checks
- Comparison with baseline/current production model

#### Phase 5: Model Deployment
- Package model for serving
- A/B testing or canary deployment
- Integration with existing systems

#### Phase 6: Model Serving
- Real-time inference (REST API, gRPC)
- Batch inference (scheduled predictions)
- Edge deployment (mobile, IoT)

#### Phase 7: Monitoring & Alerting
- Track prediction quality
- Detect data drift and concept drift
- Monitor system health (latency, throughput)

#### Phase 8: Retraining Trigger
- Scheduled retraining
- Triggered by drift detection
- Triggered by new data availability

### Code Example: Simple ML Pipeline Structure

```python
"""
A simple MLOps-aware project structure demonstrating 
the lifecycle stages in code.
"""

# project_structure.py - Shows how an MLOps project is organized
import os

# ============================================================
# STANDARD MLOPS PROJECT STRUCTURE
# ============================================================
project_structure = """
my-ml-project/
│
├── data/
│   ├── raw/                  # Original, immutable data
│   ├── processed/            # Cleaned, transformed data
│   └── features/             # Feature-engineered data
│
├── src/
│   ├── data/                 # Data pipeline scripts
│   │   ├── ingestion.py      # Pull data from sources
│   │   ├── validation.py     # Data quality checks
│   │   └── preprocessing.py  # Clean and transform
│   │
│   ├── features/             # Feature engineering
│   │   ├── build_features.py
│   │   └── feature_store.py
│   │
│   ├── models/               # Model training & evaluation
│   │   ├── train.py          # Training logic
│   │   ├── evaluate.py       # Evaluation metrics
│   │   ├── predict.py        # Inference logic
│   │   └── hyperparameter_tuning.py
│   │
│   ├── serving/              # Model serving
│   │   ├── api.py            # REST API (FastAPI/Flask)
│   │   └── batch_predict.py  # Batch inference jobs
│   │
│   └── monitoring/           # Production monitoring
│       ├── drift_detection.py
│       └── performance_tracking.py
│
├── pipelines/                # Orchestration (Airflow, Kubeflow)
│   ├── training_pipeline.py
│   └── inference_pipeline.py
│
├── tests/                    # Testing
│   ├── unit/
│   ├── integration/
│   └── model/                # Model-specific tests
│
├── configs/                  # Configuration files
│   ├── model_config.yaml
│   ├── data_config.yaml
│   └── serving_config.yaml
│
├── notebooks/                # Exploration (NOT production)
│   └── exploration.ipynb
│
├── Dockerfile                # Containerization
├── docker-compose.yml
├── requirements.txt          # Dependencies
├── setup.py                  # Package setup
├── Makefile                  # Common commands
├── .github/workflows/        # CI/CD
│   └── ml_pipeline.yml
└── README.md
"""

print(project_structure)
```

### Code Example: Basic ML Pipeline with MLOps Practices

```python
"""
Demonstrates a minimal but MLOps-aware training pipeline.
This shows how even simple projects can follow MLOps principles.
"""

import json
import hashlib
import datetime
from pathlib import Path
from typing import Dict, Tuple, Any

import numpy as np
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, classification_report
import joblib

# ============================================================
# CONFIGURATION (Externalized - not hardcoded)
# ============================================================
CONFIG = {
    "data": {
        "test_size": 0.2,
        "random_state": 42,
    },
    "model": {
        "n_estimators": 100,
        "max_depth": 5,
        "random_state": 42,
    },
    "paths": {
        "model_dir": "models/",
        "metrics_dir": "metrics/",
        "data_dir": "data/",
    }
}


def compute_data_hash(X: np.ndarray, y: np.ndarray) -> str:
    """
    Compute a hash of the dataset for versioning.
    This helps detect when data has changed between runs.
    """
    data_bytes = X.tobytes() + y.tobytes()
    return hashlib.md5(data_bytes).hexdigest()


def load_and_validate_data() -> Tuple[np.ndarray, np.ndarray]:
    """
    Load data with basic validation checks.
    In production, this would pull from a data warehouse
    and run comprehensive data quality tests.
    """
    # Load dataset
    iris = load_iris()
    X, y = iris.data, iris.target
    
    # ---- DATA VALIDATION ----
    # Check for NaN values
    assert not np.isnan(X).any(), "Data contains NaN values!"
    
    # Check expected shape
    assert X.shape[1] == 4, f"Expected 4 features, got {X.shape[1]}"
    
    # Check label distribution (detect class imbalance)
    unique, counts = np.unique(y, return_counts=True)
    print(f"Class distribution: {dict(zip(unique, counts))}")
    
    # Check for reasonable value ranges
    assert X.min() >= 0, "Negative values detected (unexpected for Iris)"
    assert X.max() <= 10, "Abnormally large values detected"
    
    # Compute and log data hash for version tracking
    data_hash = compute_data_hash(X, y)
    print(f"Data hash: {data_hash}")
    
    return X, y


def train_model(
    X_train: np.ndarray, 
    y_train: np.ndarray, 
    config: Dict[str, Any]
) -> RandomForestClassifier:
    """
    Train model with configuration from external config.
    Never hardcode hyperparameters inside training logic.
    """
    model = RandomForestClassifier(
        n_estimators=config["model"]["n_estimators"],
        max_depth=config["model"]["max_depth"],
        random_state=config["model"]["random_state"],
    )
    model.fit(X_train, y_train)
    return model


def evaluate_model(
    model: RandomForestClassifier, 
    X_test: np.ndarray, 
    y_test: np.ndarray
) -> Dict[str, Any]:
    """
    Evaluate model and return structured metrics.
    Always return metrics as a dictionary for easy logging/comparison.
    """
    y_pred = model.predict(X_test)
    
    metrics = {
        "accuracy": accuracy_score(y_test, y_pred),
        "classification_report": classification_report(
            y_test, y_pred, output_dict=True
        ),
        "n_test_samples": len(y_test),
    }
    
    print(f"Accuracy: {metrics['accuracy']:.4f}")
    return metrics


def save_artifacts(
    model: RandomForestClassifier, 
    metrics: Dict[str, Any], 
    config: Dict[str, Any]
) -> str:
    """
    Save model and metadata together.
    Always save: model, metrics, config, timestamp, data hash.
    """
    # Create timestamped run directory
    timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    run_dir = Path(config["paths"]["model_dir"]) / f"run_{timestamp}"
    run_dir.mkdir(parents=True, exist_ok=True)
    
    # Save model
    model_path = run_dir / "model.joblib"
    joblib.dump(model, model_path)
    
    # Save metrics
    metrics_path = run_dir / "metrics.json"
    # Convert numpy types to Python native for JSON serialization
    with open(metrics_path, "w") as f:
        json.dump(metrics, f, indent=2, default=str)
    
    # Save config (reproducibility!)
    config_path = run_dir / "config.json"
    with open(config_path, "w") as f:
        json.dump(config, f, indent=2)
    
    # Save run metadata
    metadata = {
        "timestamp": timestamp,
        "python_version": "3.9",  # In prod: sys.version
        "model_type": "RandomForestClassifier",
        "config": config,
        "metrics_summary": {"accuracy": metrics["accuracy"]},
    }
    with open(run_dir / "metadata.json", "w") as f:
        json.dump(metadata, f, indent=2)
    
    print(f"Artifacts saved to: {run_dir}")
    return str(run_dir)


def run_pipeline():
    """
    Main pipeline orchestrator.
    Each step is a separate function for testability and reuse.
    """
    print("=" * 60)
    print("STARTING ML TRAINING PIPELINE")
    print("=" * 60)
    
    # Step 1: Load and validate data
    print("\n[Step 1/4] Loading and validating data...")
    X, y = load_and_validate_data()
    
    # Step 2: Split data
    print("\n[Step 2/4] Splitting data...")
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, 
        test_size=CONFIG["data"]["test_size"],
        random_state=CONFIG["data"]["random_state"],
        stratify=y,  # Maintain class distribution
    )
    print(f"Train: {len(X_train)} samples, Test: {len(X_test)} samples")
    
    # Step 3: Train model
    print("\n[Step 3/4] Training model...")
    model = train_model(X_train, y_train, CONFIG)
    
    # Step 4: Evaluate
    print("\n[Step 4/4] Evaluating model...")
    metrics = evaluate_model(model, X_test, y_test)
    
    # Step 5: Save artifacts
    print("\n[Step 5/5] Saving artifacts...")
    run_dir = save_artifacts(model, metrics, CONFIG)
    
    print("\n" + "=" * 60)
    print("PIPELINE COMPLETED SUCCESSFULLY")
    print("=" * 60)
    
    return model, metrics


# Run the pipeline
if __name__ == "__main__":
    model, metrics = run_pipeline()
```

**Output:**
```
============================================================
STARTING ML TRAINING PIPELINE
============================================================

[Step 1/4] Loading and validating data...
Class distribution: {0: 50, 1: 50, 2: 50}
Data hash: 3dc9e8c46c2e09e77ede42e8c459a3fc

[Step 2/4] Splitting data...
Train: 120 samples, Test: 30 samples

[Step 3/4] Training model...

[Step 4/4] Evaluating model...
Accuracy: 0.9667

[Step 5/5] Saving artifacts...
Artifacts saved to: models/run_20240115_143022

============================================================
PIPELINE COMPLETED SUCCESSFULLY
============================================================
```

---

## DevOps vs MLOps

### How It Works

MLOps extends DevOps principles but adds ML-specific challenges. Think of it this way:

> **DevOps** = Code → Test → Deploy → Monitor
> **MLOps** = Code + Data + Model → Test → Deploy → Monitor + Retrain

### Key Differences

| Aspect | DevOps | MLOps |
|--------|--------|-------|
| **What changes** | Code | Code + Data + Model |
| **Testing** | Unit/integration tests | + Data tests, model tests, bias tests |
| **Versioning** | Code (Git) | Code + Data + Model + Config |
| **CI trigger** | Code commit | Code commit + Data change + Schedule |
| **CD target** | Application binary | Model artifact + Serving infra |
| **Monitoring** | App metrics (latency, errors) | + Model performance, data drift |
| **Rollback** | Previous code version | Previous model version (plus data!) |
| **Build time** | Seconds to minutes | Minutes to hours (training) |
| **Reproducibility** | Same code → Same binary | Same code ≠ Same model (randomness, data) |

### The Three Axes of ML Change

```
                    ┌─── CODE (algorithms, features, preprocessing)
                    │
ML System ──────────┼─── DATA (new data, distribution shift)
                    │
                    └─── MODEL (hyperparameters, architecture)
                    
Traditional Software only has ONE axis: CODE
```

### Why Standard DevOps Fails for ML

```python
"""
Demonstration of why ML needs more than standard DevOps.
This shows the unique challenges ML introduces.
"""

# ============================================================
# CHALLENGE 1: Non-determinism
# ============================================================
from sklearn.neural_network import MLPClassifier
from sklearn.datasets import make_classification
import numpy as np

X, y = make_classification(n_samples=100, random_state=42)

# Same code, different results each time (without random_state)!
model1 = MLPClassifier(max_iter=100)  # No random_state
model1.fit(X, y)
score1 = model1.score(X, y)

model2 = MLPClassifier(max_iter=100)  # Same code!
model2.fit(X, y)
score2 = model2.score(X, y)

print(f"Run 1 accuracy: {score1:.4f}")
print(f"Run 2 accuracy: {score2:.4f}")
print(f"Same code, same data, different results: {score1 != score2}")
# This breaks the DevOps assumption: same input → same output

# ============================================================
# CHALLENGE 2: Data Dependencies
# ============================================================
# In DevOps: dependencies are libraries (requirements.txt)
# In MLOps: dependencies include DATA which can change silently

# Example: Feature distribution shift
training_data_stats = {"mean_age": 35, "std_age": 10}
production_data_stats = {"mean_age": 45, "std_age": 15}  # Shifted!

# The model code didn't change, but performance will degrade
# DevOps monitoring won't catch this!

# ============================================================
# CHALLENGE 3: Gradual Degradation
# ============================================================
# Software bugs: Binary (works or crashes)
# ML bugs: Gradual (slowly gets worse, hard to detect)

# Simulating model degradation over time
np.random.seed(42)
days = range(1, 31)
# Model accuracy slowly drops as data drifts
accuracy_over_time = [0.95 - 0.005 * day + np.random.normal(0, 0.01) 
                      for day in days]

print("\nModel accuracy over 30 days (simulated drift):")
print(f"Day 1:  {accuracy_over_time[0]:.3f}")
print(f"Day 15: {accuracy_over_time[14]:.3f}")
print(f"Day 30: {accuracy_over_time[29]:.3f}")
print("→ No crash, no error, just slowly getting worse!")
```

### Analogy: Restaurant vs Chemistry Lab

| DevOps (Restaurant) | MLOps (Chemistry Lab) |
|---------------------|----------------------|
| Recipe (code) is fixed | Formula evolves with new ingredients |
| Ingredients (inputs) are consistent | Raw materials vary by batch |
| Same cook → Same dish | Same process → Different results |
| Quality check: taste test | Quality check: multiple experiments |
| Scale up: hire more cooks | Scale up: need bigger equipment |
| Bad dish? Redo immediately | Bad batch? Investigate root cause |

---

## MLOps Maturity Levels

### How It Works

Google defined MLOps maturity levels (0-2) to help organizations assess where they are and where they need to go. Think of it like a video game: you start at Level 0 and progress as your ML practice matures.

### Level 0: Manual Process

```
┌─────────────────────────────────────────────────────────────┐
│                    LEVEL 0: MANUAL                           │
│                                                             │
│  Data Scientist                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Jupyter Notebook                                    │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐     │   │
│  │  │ Data │→│ Prep │→│Train │→│ Eval │→│Model │     │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──┬───┘     │   │
│  └──────────────────────────────────────────┼──────────┘   │
│                                              │              │
│                     Manual handoff           │              │
│                     (email, Slack, USB?)      ▼              │
│                                         ┌──────────┐        │
│                     ML Engineer          │ Deploy   │        │
│                     manually deploys     │ (manual) │        │
│                                         └──────────┘        │
│                                                             │
│  Characteristics:                                           │
│  • No automation                                            │
│  • Infrequent releases (quarterly?)                         │
│  • No monitoring                                            │
│  • No reproducibility guarantees                            │
│  • "It worked on my laptop"                                 │
└─────────────────────────────────────────────────────────────┘
```

**Typical at:** Startups, early ML adoption, research teams

### Level 1: ML Pipeline Automation

```
┌─────────────────────────────────────────────────────────────┐
│                    LEVEL 1: AUTOMATED PIPELINE               │
│                                                             │
│  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐    │
│  │ Data │──►│ Prep │──►│Train │──►│ Eval │──►│Deploy│    │
│  │Ingest│   │      │   │      │   │      │   │      │    │
│  └──────┘   └──────┘   └──────┘   └──────┘   └──────┘    │
│      │           │          │          │          │         │
│      ▼           ▼          ▼          ▼          ▼         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │          Orchestrator (Airflow, Kubeflow)           │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                      │
│      ▼                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │     Metadata Store (experiments, metrics, lineage)  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Characteristics:                                           │
│  • Automated training pipeline                              │
│  • Continuous training on new data                          │
│  • Experiment tracking                                      │
│  • Model registry                                           │
│  • BUT: Pipeline code changes are still manual              │
└─────────────────────────────────────────────────────────────┘
```

**Typical at:** Mid-size companies, mature ML teams

### Level 2: CI/CD Pipeline Automation

```
┌─────────────────────────────────────────────────────────────┐
│                    LEVEL 2: FULL CI/CD                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Source Control                    │    │
│  │         (Code, Pipeline Definitions, Config)        │    │
│  └───────────────────────┬─────────────────────────────┘    │
│                          │ triggers                          │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                CI Pipeline                          │    │
│  │  • Build pipeline components                        │    │
│  │  • Run unit tests + integration tests               │    │
│  │  • Test data validation logic                       │    │
│  │  • Test model training logic                        │    │
│  └───────────────────────┬─────────────────────────────┘    │
│                          │ deploys                           │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              CD Pipeline                            │    │
│  │  • Deploy ML pipeline to production                 │    │
│  │  • ML pipeline runs → produces model                │    │
│  │  • Model validated → pushed to registry             │    │
│  │  • Model deployed to serving infrastructure         │    │
│  └───────────────────────┬─────────────────────────────┘    │
│                          │ monitors                          │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Monitoring + Trigger                      │    │
│  │  • Performance monitoring                           │    │
│  │  • Drift detection → auto-retrain                   │    │
│  │  • A/B testing results → auto-promote               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Characteristics:                                           │
│  • Full automation of pipeline deployment                   │
│  • Rapid experimentation + rapid deployment                │
│  • Automated testing at every stage                        │
│  • Multiple models can be trained/deployed per day         │
└─────────────────────────────────────────────────────────────┘
```

**Typical at:** Tech giants (Google, Netflix, Uber), mature ML organizations

### Maturity Assessment Code

```python
"""
MLOps Maturity Assessment Tool.
Use this to evaluate where your organization stands.
"""

from dataclasses import dataclass
from typing import List


@dataclass
class MaturityQuestion:
    """A single assessment question."""
    category: str
    question: str
    level_0: str  # What Level 0 looks like
    level_1: str  # What Level 1 looks like
    level_2: str  # What Level 2 looks like


# Define assessment questions
ASSESSMENT = [
    MaturityQuestion(
        category="Training",
        question="How is model training triggered?",
        level_0="Manually in notebooks",
        level_1="Automated pipeline on schedule/trigger",
        level_2="Automated + CI/CD deploys new pipeline versions",
    ),
    MaturityQuestion(
        category="Deployment",
        question="How are models deployed?",
        level_0="Manual copy/deploy",
        level_1="Automated deployment from registry",
        level_2="Canary/A-B deploy with auto-rollback",
    ),
    MaturityQuestion(
        category="Monitoring",
        question="How is model performance tracked?",
        level_0="Not tracked / ad-hoc checks",
        level_1="Basic metrics dashboard",
        level_2="Automated drift detection + alerting + retraining",
    ),
    MaturityQuestion(
        category="Testing",
        question="What testing exists?",
        level_0="Manual testing in notebook",
        level_1="Unit tests for pipeline components",
        level_2="Full test suite: unit + integration + model + data",
    ),
    MaturityQuestion(
        category="Versioning",
        question="What is versioned?",
        level_0="Only code (maybe)",
        level_1="Code + Model + some config",
        level_2="Code + Data + Model + Config + Pipeline + Environment",
    ),
    MaturityQuestion(
        category="Reproducibility",
        question="Can you reproduce a past result?",
        level_0="Unlikely / depends on who ran it",
        level_1="Yes, with some manual setup",
        level_2="Yes, fully automated from any commit",
    ),
]


def assess_maturity():
    """Run a quick maturity assessment."""
    print("=" * 60)
    print("       MLOps MATURITY ASSESSMENT")
    print("=" * 60)
    
    scores = []
    
    for i, q in enumerate(ASSESSMENT, 1):
        print(f"\n{'─' * 60}")
        print(f"Q{i}. [{q.category}] {q.question}")
        print(f"  Level 0: {q.level_0}")
        print(f"  Level 1: {q.level_1}")
        print(f"  Level 2: {q.level_2}")
        # In interactive mode, you'd collect input here
        # For demo, assume Level 0 for all
        scores.append(0)
    
    avg_score = sum(scores) / len(scores)
    print(f"\n{'=' * 60}")
    print(f"Overall Maturity Level: {avg_score:.1f}")
    
    if avg_score < 0.5:
        print("Status: Level 0 (Manual)")
        print("Next step: Automate your training pipeline")
    elif avg_score < 1.5:
        print("Status: Level 1 (Pipeline Automation)")
        print("Next step: Add CI/CD for pipeline code")
    else:
        print("Status: Level 2 (Full CI/CD)")
        print("Next step: Optimize and scale")


assess_maturity()
```

---

## Key Components of MLOps

### Component Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                    MLOps COMPONENT STACK                            │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    ORCHESTRATION LAYER                        │  │
│  │         (Airflow, Kubeflow Pipelines, Prefect)               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐   │
│  │ Data    │ │Experiment│ │ Model   │ │ Model   │ │ Monitor- │   │
│  │ Version │ │ Tracking │ │Registry │ │ Serving │ │ ing      │   │
│  │         │ │          │ │         │ │         │ │          │   │
│  │ • DVC   │ │ • MLflow │ │ • MLflow│ │ • Fast- │ │ •Evidently│  │
│  │ • Lake- │ │ • W&B   │ │ • Bento-│ │   API   │ │ •Prometheus│ │
│  │   FS    │ │ • Comet │ │   ML    │ │ • Triton│ │ •Grafana  │  │
│  │ • Delta │ │         │ │ • Vertex│ │ • Seldon│ │ •WhyLabs  │  │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └──────────┘   │
│                              │                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                 INFRASTRUCTURE LAYER                          │  │
│  │       (Kubernetes, Docker, Cloud Services, Terraform)        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    CI/CD LAYER                                │  │
│  │         (GitHub Actions, GitLab CI, Jenkins, CML)            │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

### Component Details

| Component | Purpose | Key Tools |
|-----------|---------|-----------|
| **Data Versioning** | Track data changes, ensure reproducibility | DVC, LakeFS, Delta Lake |
| **Experiment Tracking** | Log params, metrics, artifacts for every run | MLflow, W&B, Neptune |
| **Model Registry** | Central store for production-ready models | MLflow Registry, Vertex AI |
| **Feature Store** | Consistent feature computation train/serve | Feast, Tecton, Hopsworks |
| **Model Serving** | Expose models as APIs | FastAPI, TF Serving, Triton |
| **Monitoring** | Detect drift, track performance | Evidently, WhyLabs, Arize |
| **Orchestration** | Schedule and coordinate pipeline steps | Airflow, Kubeflow, Prefect |
| **CI/CD** | Automate testing and deployment | GitHub Actions, CML, Jenkins |
| **Infrastructure** | Compute, storage, networking | K8s, Docker, Terraform |

---

## MLOps Team Roles

### Role Breakdown

```
┌────────────────────────────────────────────────────────────────┐
│                    WHO DOES WHAT IN MLOps                       │
│                                                                │
│  Data Scientist          ML Engineer          Platform/DevOps  │
│  ┌──────────────┐       ┌──────────────┐    ┌──────────────┐  │
│  │• Experiments │       │• Pipelines   │    │• Infra       │  │
│  │• Feature Eng │       │• Model Optim │    │• Kubernetes  │  │
│  │• Model Dev   │       │• Serving     │    │• CI/CD       │  │
│  │• Notebooks   │       │• Testing     │    │• Monitoring  │  │
│  │• Research    │       │• Production  │    │• Security    │  │
│  │              │       │  Code        │    │• Scaling     │  │
│  └──────┬───────┘       └──────┬───────┘    └──────┬───────┘  │
│         │                      │                    │          │
│         └──────────────────────┼────────────────────┘          │
│                                │                               │
│                    All collaborate on:                          │
│                    • Defining metrics                           │
│                    • Setting quality gates                      │
│                    • Incident response                          │
│                    • Architecture decisions                     │
└────────────────────────────────────────────────────────────────┘
```

---

## Tools Landscape

### Code Example: MLOps Tool Comparison Helper

```python
"""
MLOps tools categorized by function.
Use this as a reference when building your MLOps stack.
"""

mlops_tools = {
    "Experiment Tracking": {
        "tools": [
            {"name": "MLflow", "cost": "Free/OSS", "best_for": "All-in-one, self-hosted"},
            {"name": "Weights & Biases", "cost": "Free tier/Paid", "best_for": "Deep learning, collaboration"},
            {"name": "Neptune.ai", "cost": "Free tier/Paid", "best_for": "Enterprise, metadata management"},
            {"name": "Comet ML", "cost": "Free tier/Paid", "best_for": "Code tracking, diff visualization"},
        ],
    },
    "Data Versioning": {
        "tools": [
            {"name": "DVC", "cost": "Free/OSS", "best_for": "Git-like data versioning"},
            {"name": "LakeFS", "cost": "Free/OSS", "best_for": "Data lake versioning at scale"},
            {"name": "Delta Lake", "cost": "Free/OSS", "best_for": "ACID transactions on data lakes"},
            {"name": "Pachyderm", "cost": "Free tier/Paid", "best_for": "Data lineage + versioning"},
        ],
    },
    "Model Serving": {
        "tools": [
            {"name": "FastAPI", "cost": "Free/OSS", "best_for": "Simple REST APIs, prototypes"},
            {"name": "TF Serving", "cost": "Free/OSS", "best_for": "TensorFlow models at scale"},
            {"name": "TorchServe", "cost": "Free/OSS", "best_for": "PyTorch models"},
            {"name": "Triton", "cost": "Free/OSS", "best_for": "Multi-framework, GPU optimization"},
            {"name": "BentoML", "cost": "Free/OSS", "best_for": "Easy packaging + deployment"},
            {"name": "Seldon Core", "cost": "OSS/Enterprise", "best_for": "Kubernetes-native, A/B testing"},
        ],
    },
    "Orchestration": {
        "tools": [
            {"name": "Apache Airflow", "cost": "Free/OSS", "best_for": "General workflow orchestration"},
            {"name": "Kubeflow Pipelines", "cost": "Free/OSS", "best_for": "K8s-native ML pipelines"},
            {"name": "Prefect", "cost": "Free tier/Paid", "best_for": "Modern Python-native workflows"},
            {"name": "Dagster", "cost": "Free tier/Paid", "best_for": "Data-aware orchestration"},
            {"name": "ZenML", "cost": "Free/OSS", "best_for": "MLOps framework, portable pipelines"},
        ],
    },
    "Monitoring": {
        "tools": [
            {"name": "Evidently AI", "cost": "Free/OSS", "best_for": "Data/model drift detection"},
            {"name": "WhyLabs", "cost": "Free tier/Paid", "best_for": "Real-time monitoring"},
            {"name": "Arize AI", "cost": "Free tier/Paid", "best_for": "LLM + ML observability"},
            {"name": "Prometheus+Grafana", "cost": "Free/OSS", "best_for": "Infrastructure metrics"},
            {"name": "NannyML", "cost": "Free/OSS", "best_for": "Performance estimation w/o labels"},
        ],
    },
    "End-to-End Platforms": {
        "tools": [
            {"name": "AWS SageMaker", "cost": "Pay-per-use", "best_for": "AWS ecosystem"},
            {"name": "GCP Vertex AI", "cost": "Pay-per-use", "best_for": "Google ecosystem"},
            {"name": "Azure ML", "cost": "Pay-per-use", "best_for": "Microsoft ecosystem"},
            {"name": "Databricks", "cost": "Pay-per-use", "best_for": "Unified data + ML"},
        ],
    },
}

# Print formatted comparison
for category, data in mlops_tools.items():
    print(f"\n{'='*60}")
    print(f"  {category}")
    print(f"{'='*60}")
    print(f"  {'Tool':<20} {'Cost':<18} {'Best For'}")
    print(f"  {'─'*18} {'─'*16} {'─'*30}")
    for tool in data["tools"]:
        print(f"  {tool['name']:<20} {tool['cost']:<18} {tool['best_for']}")
```

### Choosing Your Stack (Decision Framework)

```python
"""
Decision framework for choosing MLOps tools.
Answer these questions to find the right stack for your team.
"""

def recommend_stack(
    team_size: str,        # "solo", "small" (2-5), "medium" (5-20), "large" (20+)
    cloud_provider: str,   # "aws", "gcp", "azure", "multi", "on-prem"
    ml_maturity: str,      # "beginner", "intermediate", "advanced"
    budget: str,           # "minimal", "moderate", "enterprise"
) -> dict:
    """Recommend an MLOps stack based on constraints."""
    
    recommendations = {"stack": [], "reasoning": []}
    
    # Experiment Tracking
    if budget == "minimal" or ml_maturity == "beginner":
        recommendations["stack"].append(("Experiment Tracking", "MLflow"))
        recommendations["reasoning"].append("MLflow: Free, self-hosted, good starting point")
    else:
        recommendations["stack"].append(("Experiment Tracking", "Weights & Biases"))
        recommendations["reasoning"].append("W&B: Better collaboration, visualization")
    
    # Orchestration
    if team_size in ["solo", "small"]:
        recommendations["stack"].append(("Orchestration", "Prefect / simple cron"))
        recommendations["reasoning"].append("Small teams don't need Airflow complexity")
    else:
        recommendations["stack"].append(("Orchestration", "Airflow or Kubeflow"))
        recommendations["reasoning"].append("Larger teams benefit from DAG visualization")
    
    # Model Serving
    if ml_maturity == "beginner":
        recommendations["stack"].append(("Serving", "FastAPI + Docker"))
        recommendations["reasoning"].append("FastAPI: Easy to learn, sufficient for most cases")
    elif team_size in ["medium", "large"]:
        recommendations["stack"].append(("Serving", "Triton or Seldon Core"))
        recommendations["reasoning"].append("Scale needs dedicated serving infrastructure")
    
    # Monitoring
    if budget == "minimal":
        recommendations["stack"].append(("Monitoring", "Evidently AI"))
        recommendations["reasoning"].append("Evidently: Free, good drift reports")
    else:
        recommendations["stack"].append(("Monitoring", "WhyLabs or Arize"))
        recommendations["reasoning"].append("Managed monitoring reduces ops burden")
    
    return recommendations


# Example usage
result = recommend_stack(
    team_size="small",
    cloud_provider="aws",
    ml_maturity="intermediate",
    budget="moderate",
)

print("Recommended MLOps Stack:")
print("-" * 40)
for component, tool in result["stack"]:
    print(f"  {component}: {tool}")
print("\nReasoning:")
for reason in result["reasoning"]:
    print(f"  • {reason}")
```

---

## Common Mistakes

### 1. Starting with Tools Instead of Processes

❌ **Wrong:** "Let's set up Kubeflow, MLflow, and Airflow!"
✅ **Right:** "What's our biggest pain point? Let's solve that first."

> **Pro Tip:** Start with the simplest thing that works. A cron job + shell script > over-engineered Kubernetes pipeline that nobody maintains.

### 2. Treating ML Models Like Regular Software

❌ **Wrong:** Only version-controlling the code
✅ **Right:** Versioning code + data + model + config + environment

### 3. No Baseline Comparison

❌ **Wrong:** "Our model has 95% accuracy, ship it!"
✅ **Right:** "Our model has 95% accuracy, which is 3% better than the current production model AND 10% better than a simple heuristic baseline."

### 4. Ignoring Data Quality

❌ **Wrong:** Focusing only on model architecture
✅ **Right:** Spending 80% of effort on data quality (garbage in = garbage out)

### 5. Over-Engineering Too Early

❌ **Wrong:** Building a full MLOps platform for your first model
✅ **Right:** Start simple, add complexity as pain points emerge

```
Maturity Journey:
Notebook → Script → Pipeline → Automated Pipeline → Full MLOps

DO NOT skip steps! Each step teaches you what you need next.
```

### 6. No Monitoring in Production

❌ **Wrong:** Deploy and forget
✅ **Right:** Monitor from day 1 (at minimum: prediction distribution, latency, error rate)

### 7. Manual Feature Engineering in Production

❌ **Wrong:** Different feature code in notebook vs production
✅ **Right:** Single feature computation shared between training and serving (feature store pattern)

---

## Interview Questions

### Conceptual Questions

**Q1: What is MLOps and why is it needed?**
> **A:** MLOps is a set of practices combining ML, DevOps, and Data Engineering to reliably deploy and maintain ML systems in production. It's needed because ML systems have unique challenges: they degrade silently (data drift), depend on data that changes, are non-deterministic, and require continuous retraining. Without MLOps, 87% of ML projects never reach production.

**Q2: Explain the difference between DevOps and MLOps.**
> **A:** DevOps manages code deployments; MLOps manages code + data + model deployments. Key differences: (1) ML has three axes of change (code, data, model) vs one (code), (2) ML requires experiment tracking and model versioning, (3) ML needs data validation and drift monitoring, (4) ML builds are non-deterministic, (5) ML systems degrade gradually rather than failing suddenly.

**Q3: What are the MLOps maturity levels?**
> **A:** Level 0 (Manual): Everything done manually in notebooks, no automation. Level 1 (Pipeline Automation): Automated training pipeline, experiment tracking, model registry, continuous training. Level 2 (CI/CD): Full automation including pipeline deployment, automated testing, canary deployments, automated retraining triggered by monitoring.

**Q4: What is data drift and concept drift?**
> **A:** Data drift: input feature distributions change over time (e.g., customer demographics shift). Concept drift: the relationship between features and target changes (e.g., pandemic changes spending patterns). Both cause model performance degradation without any code changes.

**Q5: How would you design an ML system for a company that's never deployed ML before?**
> **A:** Start at Level 0 with clear goals: (1) Identify the business problem and success metric, (2) Build a simple model with proper project structure, (3) Add experiment tracking (MLflow), (4) Deploy with FastAPI + Docker, (5) Add basic monitoring (prediction distribution, latency), (6) Iterate — add automation as pain points emerge. Don't over-engineer upfront.

### Scenario-Based Questions

**Q6: Your model's accuracy dropped from 95% to 88% over two weeks. How do you diagnose this?**
> **A:** Systematic approach: (1) Check data pipeline — is input data quality degraded? (2) Check feature distributions — has data drift occurred? (3) Check prediction distribution — is the model outputting different patterns? (4) Check if the target variable definition changed. (5) Compare recent data samples manually. (6) If data drifted, retrain on recent data. (7) If concept drifted, may need model architecture change.

**Q7: How do you ensure reproducibility of ML experiments?**
> **A:** Version everything: (1) Code — Git, (2) Data — DVC or data versioning, (3) Environment — Docker/conda lock files, (4) Config — externalized hyperparameters in YAML, (5) Random seeds — set and log them, (6) Hardware info — log GPU/CPU specs, (7) Use experiment tracking (MLflow) to link all these together for each run.

**Q8: Your team has 3 data scientists all training models. How do you prevent chaos?**
> **A:** (1) Shared experiment tracking (MLflow/W&B) so everyone sees all experiments, (2) Model registry with approval workflow, (3) Git branching strategy for model code, (4) Shared feature store to prevent duplicate feature engineering, (5) Clear promotion path: experiment → staging → production, (6) Code review for pipeline changes.

---

## Quick Reference

### MLOps Cheat Sheet

| Concept | One-Line Summary |
|---------|-----------------|
| MLOps | DevOps + Data + Models = Production ML |
| ML Lifecycle | Continuous loop: Build → Deploy → Monitor → Retrain |
| Level 0 | Manual everything (most teams start here) |
| Level 1 | Automated training pipelines |
| Level 2 | Full CI/CD for pipelines + automated monitoring |
| Data Drift | Input distributions change over time |
| Concept Drift | Feature-target relationships change |
| Model Registry | Central store for versioned, approved models |
| Feature Store | Shared feature computation for train + serve |
| A/B Testing | Compare new model against current production |
| Canary Deploy | Gradually shift traffic to new model |
| Shadow Deploy | New model runs in parallel, predictions not served |

### Key Principles

1. **Automate gradually** — Don't build a platform before you have a model
2. **Version everything** — Code, data, model, config, environment
3. **Test ML-specifically** — Data tests, model tests, not just unit tests
4. **Monitor from day 1** — At minimum: predictions, latency, input distributions
5. **Reproducibility is non-negotiable** — If you can't reproduce it, you can't debug it
6. **Start simple** — FastAPI > Kubernetes for your first deployment
7. **Data quality > Model complexity** — A simple model on clean data beats complex model on dirty data

### Quick Command Reference

```bash
# Initialize MLflow tracking
mlflow ui --port 5000

# Initialize DVC in a project
dvc init
dvc remote add -d storage s3://my-bucket/dvc-store

# Build Docker image for ML model
docker build -t my-model:v1.0 .
docker run -p 8000:8000 my-model:v1.0

# Quick model serving with FastAPI
uvicorn app:app --host 0.0.0.0 --port 8000

# Run data validation with Great Expectations
great_expectations checkpoint run my_checkpoint
```

---

## Summary

MLOps is not a tool — it's a **mindset and practice** for treating ML systems as production software. The key insight is that ML systems are fundamentally different from traditional software because they depend on data that changes, produce non-deterministic outputs, and degrade silently.

**Start where you are, not where you want to be.** Move from Level 0 → Level 1 → Level 2 incrementally, solving real pain points at each step.

---

*Next: [02-Experiment-Tracking.md](02-Experiment-Tracking.md) — Deep dive into MLflow, W&B, and experiment organization*
