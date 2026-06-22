# Model Versioning and Registry

## Table of Contents
- [What is Model Versioning](#what-is-model-versioning)
- [Why Model Versioning Matters](#why-model-versioning-matters)
- [How Model Versioning Works](#how-model-versioning-works)
- [DVC — Data Version Control](#dvc--data-version-control)
- [MLflow Model Registry](#mlflow-model-registry)
- [Versioning Data](#versioning-data)
- [Versioning Models](#versioning-models)
- [Versioning Code + Config + Environment](#versioning-code--config--environment)
- [End-to-End Lineage](#end-to-end-lineage)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Model Versioning

### Simple Explanation

Imagine you're writing a book. You'd never work without "Save As" and "Undo." Now imagine you're writing a book where the **words** (code), the **paper** (data), and the **font** (model) all change independently — and any combination of the three could be "the right version."

**Model versioning** is like having a perfect time machine for your entire ML system. You can jump back to any point in time and see exactly:
- What code was used
- What data was used
- What model was produced
- What config/hyperparameters were set
- What results were achieved

### Formal Definition

> **Model versioning** is the practice of systematically tracking and managing different versions of ML artifacts — including trained models, datasets, configurations, and code — to enable reproducibility, comparison, auditing, and safe deployment.

### The Three Pillars of ML Versioning

```
┌─────────────────────────────────────────────────────────────────┐
│              THE THREE PILLARS OF ML VERSIONING                  │
│                                                                  │
│    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐       │
│    │              │   │              │   │              │       │
│    │    CODE      │   │    DATA      │   │    MODEL     │       │
│    │  Versioning  │   │  Versioning  │   │  Versioning  │       │
│    │              │   │              │   │              │       │
│    │  • Git       │   │  • DVC       │   │  • MLflow    │       │
│    │  • Commits   │   │  • LakeFS   │   │    Registry  │       │
│    │  • Branches  │   │  • Delta    │   │  • BentoML   │       │
│    │  • Tags      │   │    Lake     │   │  • W&B       │       │
│    │              │   │  • Pachyderm│   │    Artifacts  │       │
│    └──────┬───────┘   └──────┬───────┘   └──────┬───────┘       │
│           │                  │                   │               │
│           └──────────────────┼───────────────────┘               │
│                              │                                   │
│                     ┌────────▼────────┐                          │
│                     │   FULL LINEAGE  │                          │
│                     │   (Who, What,   │                          │
│                     │    When, Why)   │                          │
│                     └─────────────────┘                          │
│                                                                  │
│   Without ALL THREE → Cannot reproduce, cannot audit,            │
│                        cannot safely roll back                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Why Model Versioning Matters

### The Horror Stories Without Versioning

```
Scenario 1: "Which model is in production?"
─────────────────────────────────────────────
Alice: "The model in prod is acting weird."
Bob:   "Which model is in prod?"
Alice: "The one Carol deployed last week."
Carol: "I deployed model_final_v2.pkl."
Alice: "From which notebook?"
Carol: "I don't remember. Maybe the one in my Downloads folder?"
       → 3 days of detective work

Scenario 2: "Can we roll back?"
─────────────────────────────────────────────
PM:    "The new model is causing complaints. Roll back!"
Team:  "We can roll back the model file, but..."
       "...we also changed the feature pipeline."
       "...and the preprocessing was updated."
       "...and we don't know which data the old model used."
       → Can't safely roll back

Scenario 3: "Regulatory audit"
─────────────────────────────────────────────
Auditor: "Show me what data trained the model making this decision."
Team:    "We... think it was trained on data from Q3?"
         "The feature engineering might have changed since then."
         → Compliance failure, potential fine
```

### Real-World Impact

| Scenario | Without Versioning | With Versioning |
|----------|-------------------|-----------------|
| Roll back a bad model | Hours/days, risky | One command, safe |
| Reproduce a result | "It worked on my machine" | Deterministic reproduction |
| Compare model versions | Manual, error-prone | Automated, side-by-side |
| Regulatory audit | Panic, manual reconstruction | Click, full lineage |
| Onboard new team member | "Ask Alice, she knows" | Self-documenting history |
| Debug performance drop | Detective work | Diff between versions |
| A/B test models | Which model is which? | Clearly labeled versions |

### When You'd Use It

✅ **Always use model versioning when:**
- Any model goes to production
- Multiple people work on the same model
- You iterate on models over time
- Regulatory/compliance requirements exist
- You need reproducible results

✅ **Use data versioning when:**
- Data changes over time (most real-world cases)
- Training data is curated/labeled (expensive to recreate)
- Multiple datasets exist for the same problem
- Data pipelines transform raw data

---

## How Model Versioning Works

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                MODEL VERSIONING ARCHITECTURE                         │
│                                                                     │
│  Developer's Machine                                                │
│  ┌────────────────────────────────────────────────────┐             │
│  │                                                    │             │
│  │  Code (train.py, features.py)  ──► Git             │             │
│  │  Data (train.csv, images/)     ──► DVC ──► Remote  │             │
│  │  Config (params.yaml)          ──► Git             │             │
│  │  Model (model.pkl)             ──► Model Registry  │             │
│  │  Environment (Dockerfile)      ──► Git             │             │
│  │                                                    │             │
│  └────────────────────────────────────────────────────┘             │
│                                                                     │
│  What goes where:                                                   │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                                                                │ │
│  │  Git (small files, text)        DVC / Remote Storage (large)   │ │
│  │  ├── source code                ├── training data              │ │
│  │  ├── config files               ├── processed features        │ │
│  │  ├── pipeline definitions       ├── trained models (optional) │ │
│  │  ├── requirements.txt           ├── evaluation datasets       │ │
│  │  ├── Dockerfile                 └── large artifacts           │ │
│  │  ├── .dvc files (pointers)                                    │ │
│  │  └── dvc.lock (checksums)       Model Registry                │ │
│  │                                 ├── trained models             │ │
│  │                                 ├── model metadata             │ │
│  │                                 ├── stage transitions          │ │
│  │                                 └── approval history           │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Definition | Analogy |
|---------|------------|---------|
| **Version** | A specific snapshot of an artifact | A save point in a video game |
| **Registry** | Central store for model versions | A library catalog |
| **Stage** | Lifecycle phase (Staging, Production) | Draft → Review → Published |
| **Lineage** | Full history of how an artifact was created | A family tree |
| **Pointer file** | Small file in Git that references large data | A library card (points to the book) |
| **Remote storage** | Where large files actually live (S3, GCS) | The library shelves |

---

## DVC — Data Version Control

### What is DVC

DVC (Data Version Control) extends Git to handle large files, datasets, and ML pipelines. Think of it as **"Git for data."**

```
┌─────────────────────────────────────────────────────────────┐
│                    HOW DVC WORKS                             │
│                                                             │
│  Git Repository                  Remote Storage             │
│  ┌─────────────────────┐        ┌─────────────────┐        │
│  │ train.py             │        │ S3 / GCS / Azure│        │
│  │ features.py          │        │                 │        │
│  │ params.yaml          │        │ data/           │        │
│  │ data/train.csv.dvc ──┼───────►│  train.csv     │        │
│  │   (pointer: md5=abc) │  push  │  (actual data) │        │
│  │ models/model.pkl.dvc─┼───────►│ models/        │        │
│  │   (pointer: md5=def) │        │  model.pkl     │        │
│  │ dvc.yaml (pipeline)  │        │  (actual model)│        │
│  │ dvc.lock (checksums) │        │                 │        │
│  └─────────────────────┘        └─────────────────┘        │
│                                                             │
│  .dvc file contains:                                        │
│  ┌──────────────────────────────────┐                       │
│  │ outs:                           │                       │
│  │   - md5: abc123def456           │ ← Content hash       │
│  │     size: 104857600             │ ← File size          │
│  │     path: train.csv             │ ← Original path      │
│  └──────────────────────────────────┘                       │
│                                                             │
│  Small pointer in Git → Large file in remote storage        │
│  Git tracks the POINTER, DVC tracks the DATA                │
└─────────────────────────────────────────────────────────────┘
```

### DVC Setup and Basic Usage

```python
"""
DVC Setup and Core Commands
Run these in terminal (bash/cmd).
"""

# ============================================================
# INSTALLATION
# ============================================================
install_commands = """
# Install DVC
pip install dvc

# Install with cloud storage support
pip install dvc[s3]      # AWS S3
pip install dvc[gs]      # Google Cloud Storage
pip install dvc[azure]   # Azure Blob Storage
pip install dvc[all]     # All backends
"""

# ============================================================
# INITIALIZATION
# ============================================================
init_commands = """
# Initialize DVC in a Git repository
cd my-ml-project
git init
dvc init

# This creates:
# .dvc/           — DVC internal directory
# .dvc/config     — DVC configuration
# .dvcignore      — Files to ignore (like .gitignore)

# Commit DVC initialization
git add .dvc .dvcignore
git commit -m "Initialize DVC"
"""

# ============================================================
# CONFIGURE REMOTE STORAGE
# ============================================================
remote_commands = """
# Add remote storage (where large files actually live)

# AWS S3
dvc remote add -d myremote s3://my-bucket/dvc-store
dvc remote modify myremote region us-east-1

# Google Cloud Storage
dvc remote add -d myremote gs://my-bucket/dvc-store

# Azure Blob Storage
dvc remote add -d myremote azure://my-container/dvc-store

# Local / Network drive (for testing)
dvc remote add -d myremote /path/to/local/storage

# SSH Remote
dvc remote add -d myremote ssh://user@server/path/to/storage

# Save remote config to git
git add .dvc/config
git commit -m "Configure DVC remote storage"
"""

print("DVC Setup Commands")
print("=" * 50)
print(install_commands)
print(init_commands)
print(remote_commands)
```

### Tracking Data with DVC

```python
"""
DVC Data Tracking — The Core Workflow
"""

# ============================================================
# TRACKING FILES AND DIRECTORIES
# ============================================================
tracking_commands = """
# Track a single file
dvc add data/train.csv
# Creates: data/train.csv.dvc (pointer file)
# Adds to: .gitignore (so Git ignores the large file)

# Track a directory
dvc add data/images/
# Creates: data/images.dvc

# Commit the pointer files to Git
git add data/train.csv.dvc data/.gitignore
git commit -m "Track training data v1"

# Push data to remote storage
dvc push
# Uploads actual data to configured remote (S3, GCS, etc.)
"""

# ============================================================
# VERSION CONTROL WORKFLOW
# ============================================================
version_workflow = """
# === Version 1: Original data ===
dvc add data/train.csv
git add data/train.csv.dvc
git commit -m "Data v1: 10K samples"
git tag data-v1
dvc push

# === Version 2: More data collected ===
# (data/train.csv has been updated with new samples)
dvc add data/train.csv          # DVC detects the change
git add data/train.csv.dvc      # Updated pointer (new hash)
git commit -m "Data v2: 25K samples (added Jan 2024 data)"
git tag data-v2
dvc push

# === Switch between versions ===
# Go back to v1
git checkout data-v1 -- data/train.csv.dvc
dvc checkout
# Now data/train.csv is the v1 version!

# Go to latest
git checkout main -- data/train.csv.dvc
dvc checkout

# === Pull data on another machine ===
git clone https://github.com/user/ml-project.git
cd ml-project
dvc pull    # Downloads actual data from remote
"""

# ============================================================
# CHECKING STATUS
# ============================================================
status_commands = """
# See what's changed
dvc status

# See data file details
dvc diff          # Diff between current and last commit
dvc diff HEAD~2   # Diff between current and 2 commits ago

# List tracked files
dvc list . --dvc-only
"""

print("DVC Tracking Workflow")
print("=" * 50)
print(tracking_commands)
print("\nVersion Control Workflow:")
print(version_workflow)
```

### DVC Pipelines (Reproducible ML Workflows)

```python
"""
DVC Pipelines — Define and version your entire ML workflow.
This is one of DVC's most powerful features.
"""

# ============================================================
# FILE: dvc.yaml — Pipeline Definition
# ============================================================
dvc_yaml = """
# dvc.yaml — Defines the ML pipeline as a DAG (Directed Acyclic Graph)

stages:
  # Stage 1: Prepare data
  prepare:
    cmd: python src/prepare.py --input data/raw.csv --output data/prepared.csv
    deps:                      # Dependencies (re-run if changed)
      - src/prepare.py         # Code dependency
      - data/raw.csv           # Data dependency
    params:                    # Parameters from params.yaml
      - prepare.split_ratio
      - prepare.seed
    outs:                      # Outputs (tracked by DVC)
      - data/prepared.csv

  # Stage 2: Feature engineering
  featurize:
    cmd: python src/featurize.py --input data/prepared.csv --output data/features
    deps:
      - src/featurize.py
      - data/prepared.csv      # Output of previous stage!
    params:
      - featurize.max_features
      - featurize.ngram_range
    outs:
      - data/features/

  # Stage 3: Train model
  train:
    cmd: python src/train.py --features data/features --model models/model.pkl
    deps:
      - src/train.py
      - data/features/
    params:
      - train.n_estimators
      - train.learning_rate
      - train.max_depth
    outs:
      - models/model.pkl       # Trained model
    plots:                     # Plot outputs
      - plots/training_curve.csv:
          x: epoch
          y: loss

  # Stage 4: Evaluate
  evaluate:
    cmd: python src/evaluate.py --model models/model.pkl --data data/prepared.csv
    deps:
      - src/evaluate.py
      - models/model.pkl
      - data/prepared.csv
    metrics:                   # Metrics files (tracked, shown in dvc metrics)
      - metrics/scores.json:
          cache: false         # Always show latest
    plots:
      - plots/confusion_matrix.csv:
          x: predicted
          y: actual
"""

# ============================================================
# FILE: params.yaml — Parameters (versioned in Git)
# ============================================================
params_yaml = """
# params.yaml — All hyperparameters in one place

prepare:
  split_ratio: 0.2
  seed: 42

featurize:
  max_features: 5000
  ngram_range: [1, 2]

train:
  n_estimators: 200
  learning_rate: 0.01
  max_depth: 7
"""

# ============================================================
# PIPELINE COMMANDS
# ============================================================
pipeline_commands = """
# Run the entire pipeline
dvc repro
# DVC figures out which stages need re-running based on changes!
# Changed params.yaml train section? Only re-runs train + evaluate
# Changed data? Re-runs everything

# Run a specific stage
dvc repro train

# Visualize the pipeline DAG
dvc dag
# Output:
#   +---------+
#   | prepare |
#   +---------+
#        |
#   +-----------+
#   | featurize |
#   +-----------+
#        |
#   +-------+
#   | train |
#   +-------+
#        |
#   +----------+
#   | evaluate |
#   +----------+

# View metrics
dvc metrics show
# Output:
#   metrics/scores.json:
#     accuracy: 0.9534
#     f1_score: 0.9487

# Compare metrics across Git branches/tags
dvc metrics diff
# Output:
#   Path                Metric    HEAD     workspace  Change
#   metrics/scores.json accuracy  0.9534   0.9612     0.0078

# Compare params across versions
dvc params diff
# Output:
#   Path         Param              HEAD  workspace
#   params.yaml  train.n_estimators 200   300
#   params.yaml  train.max_depth    7     10

# View plots
dvc plots show
# Generates HTML plots you can open in browser
"""

print("DVC Pipeline Definition (dvc.yaml):")
print(dvc_yaml)
print("\nParams (params.yaml):")
print(params_yaml)
print("\nPipeline Commands:")
print(pipeline_commands)
```

### DVC Pipeline Implementation Files

```python
"""
Example implementation files for a DVC pipeline.
These are the actual Python scripts referenced in dvc.yaml.
"""

# ============================================================
# FILE: src/prepare.py
# ============================================================
prepare_py = '''
"""Data preparation stage."""
import argparse
import pandas as pd
from sklearn.model_selection import train_test_split
import yaml

def main():
    # Load parameters
    with open("params.yaml") as f:
        params = yaml.safe_load(f)["prepare"]
    
    # Parse arguments
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True)
    parser.add_argument("--output", required=True)
    args = parser.parse_args()
    
    # Load and prepare data
    df = pd.read_csv(args.input)
    
    # Basic cleaning
    df = df.dropna()
    df = df.drop_duplicates()
    
    # Split
    train_df, test_df = train_test_split(
        df,
        test_size=params["split_ratio"],
        random_state=params["seed"],
    )
    
    # Save
    train_df.to_csv(args.output.replace(".csv", "_train.csv"), index=False)
    test_df.to_csv(args.output.replace(".csv", "_test.csv"), index=False)
    print(f"Prepared: {len(train_df)} train, {len(test_df)} test samples")

if __name__ == "__main__":
    main()
'''

# ============================================================
# FILE: src/train.py
# ============================================================
train_py = '''
"""Training stage with parameter loading from params.yaml."""
import argparse
import yaml
import joblib
import json
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from pathlib import Path

def main():
    # Load parameters from params.yaml
    with open("params.yaml") as f:
        params = yaml.safe_load(f)["train"]
    
    parser = argparse.ArgumentParser()
    parser.add_argument("--features", required=True)
    parser.add_argument("--model", required=True)
    args = parser.parse_args()
    
    # Load features
    X_train = np.load(f"{args.features}/X_train.npy")
    y_train = np.load(f"{args.features}/y_train.npy")
    
    # Train with params from params.yaml (NOT hardcoded!)
    model = GradientBoostingClassifier(
        n_estimators=params["n_estimators"],
        learning_rate=params["learning_rate"],
        max_depth=params["max_depth"],
        random_state=42,
    )
    model.fit(X_train, y_train)
    
    # Save model
    Path(args.model).parent.mkdir(parents=True, exist_ok=True)
    joblib.dump(model, args.model)
    print(f"Model saved to {args.model}")

if __name__ == "__main__":
    main()
'''

# ============================================================
# FILE: src/evaluate.py
# ============================================================
evaluate_py = '''
"""Evaluation stage — outputs metrics in DVC-compatible format."""
import argparse
import json
import joblib
import numpy as np
from sklearn.metrics import accuracy_score, f1_score, precision_score, recall_score
from pathlib import Path

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--model", required=True)
    parser.add_argument("--data", required=True)
    args = parser.parse_args()
    
    # Load model and test data
    model = joblib.load(args.model)
    X_test = np.load(f"{args.data}/X_test.npy")
    y_test = np.load(f"{args.data}/y_test.npy")
    
    # Predict
    y_pred = model.predict(X_test)
    
    # Calculate metrics
    metrics = {
        "accuracy": float(accuracy_score(y_test, y_pred)),
        "f1_score": float(f1_score(y_test, y_pred, average="weighted")),
        "precision": float(precision_score(y_test, y_pred, average="weighted")),
        "recall": float(recall_score(y_test, y_pred, average="weighted")),
        "n_test_samples": int(len(y_test)),
    }
    
    # Save metrics (DVC reads this file)
    Path("metrics").mkdir(exist_ok=True)
    with open("metrics/scores.json", "w") as f:
        json.dump(metrics, f, indent=2)
    
    print(f"Evaluation Results:")
    for k, v in metrics.items():
        print(f"  {k}: {v}")

if __name__ == "__main__":
    main()
'''

print("Pipeline files defined:")
print("  • src/prepare.py  — Data preparation")
print("  • src/train.py    — Model training")
print("  • src/evaluate.py — Evaluation + metrics output")
```

### DVC Experiments (Built-in Experiment Tracking)

```python
"""
DVC Experiments — Run and compare experiments without Git branches.
A newer DVC feature that competes with MLflow for experiment tracking.
"""

dvc_experiments = """
# ============================================================
# DVC EXPERIMENTS — Quick Iterations
# ============================================================

# Run experiment with modified parameters (no git commit needed!)
dvc exp run --set-param train.n_estimators=300
dvc exp run --set-param train.learning_rate=0.05
dvc exp run --set-param train.n_estimators=500 --set-param train.max_depth=10

# Queue multiple experiments (run in parallel later)
dvc exp run --queue --set-param train.n_estimators=100
dvc exp run --queue --set-param train.n_estimators=200
dvc exp run --queue --set-param train.n_estimators=300
dvc exp run --run-all --parallel 4  # Run all queued, 4 at a time

# Show all experiments
dvc exp show
# Output (table):
# ┏━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━━━━━━┓
# ┃ Experiment     ┃ accuracy ┃ f1_score ┃ n_estimators   ┃
# ┡━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━━━━━━┩
# │ workspace      │ 0.9612   │ 0.9587   │ 300            │
# │ exp-abc123     │ 0.9534   │ 0.9487   │ 200            │
# │ exp-def456     │ 0.9601   │ 0.9576   │ 500            │
# │ exp-ghi789     │ 0.9489   │ 0.9445   │ 100            │
# └────────────────┴──────────┴──────────┴────────────────┘

# Compare two experiments
dvc exp diff exp-abc123 exp-def456

# Apply best experiment to workspace (make it the current state)
dvc exp apply exp-def456
git add .
git commit -m "Apply best experiment: 500 trees"

# Remove experiments you don't need
dvc exp remove exp-ghi789

# Push experiments to Git remote (share with team)
dvc exp push origin exp-def456

# Pull a colleague's experiment
dvc exp pull origin exp-abc123
"""

print(dvc_experiments)
```

---

## MLflow Model Registry

### What is a Model Registry

A **Model Registry** is a centralized store for managing ML models throughout their lifecycle. Think of it as a **"app store" for your organization's models** — a catalog where every model is registered, versioned, reviewed, and promoted through stages.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   MODEL REGISTRY LIFECYCLE                           │
│                                                                     │
│  Experiment                                                         │
│  Tracking        Model Registry         Production                  │
│  ┌──────┐       ┌─────────────────────────────────────┐  ┌──────┐  │
│  │Run 1 │       │                                     │  │      │  │
│  │Run 2 │──────►│  "fraud-detector"                   │  │ API  │  │
│  │Run 3 │ best  │  ┌───────────────────────────────┐  │  │Server│  │
│  │Run 4 │ model │  │ Version 1 ──► None (archived) │  │  │      │  │
│  │...   │       │  │ Version 2 ──► Staging          │──┼─►│      │  │
│  └──────┘       │  │ Version 3 ──► Production ──────│──┼─►│      │  │
│                 │  │ Version 4 ──► None (new)       │  │  │      │  │
│                 │  └───────────────────────────────┘  │  └──────┘  │
│                 │                                     │            │
│                 │  Metadata per version:               │            │
│                 │  • Source run ID                     │            │
│                 │  • Creation timestamp                │            │
│                 │  • Description                       │            │
│                 │  • Tags                              │            │
│                 │  • Stage transitions (audit log)     │            │
│                 └─────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

### Model Stages

```
┌──────────────────────────────────────────────────────────┐
│              MODEL STAGE TRANSITIONS                      │
│                                                          │
│   ┌──────┐     ┌─────────┐     ┌────────────┐           │
│   │ None │────►│ Staging │────►│ Production │           │
│   │      │     │         │     │            │           │
│   └──────┘     └─────────┘     └────────────┘           │
│       │             │               │                    │
│       │             │               │                    │
│       ▼             ▼               ▼                    │
│   ┌──────────┐ ┌──────────┐  ┌──────────┐              │
│   │ Archived │ │ Archived │  │ Archived │              │
│   └──────────┘ └──────────┘  └──────────┘              │
│                                                          │
│   None:       Just registered, not validated             │
│   Staging:    Being tested, A/B testing, validation      │
│   Production: Actively serving predictions               │
│   Archived:   Retired, kept for audit trail              │
└──────────────────────────────────────────────────────────┘
```

> **Note (MLflow 2.x+):** MLflow is transitioning from fixed stages (Staging/Production/Archived) to flexible **model aliases** (e.g., `champion`, `challenger`). Both approaches are shown below.

### MLflow Registry — Complete Workflow

```python
"""
MLflow Model Registry — Full Lifecycle Example
From experiment to production in code.
"""

import mlflow
import mlflow.sklearn
from mlflow.tracking import MlflowClient
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.datasets import load_wine
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
import time

# Setup
client = MlflowClient()
mlflow.set_experiment("wine-classification-registry-demo")

wine = load_wine()
X_train, X_test, y_train, y_test = train_test_split(
    wine.data, wine.target, test_size=0.2, random_state=42
)

# ============================================================
# STEP 1: Train and log a model (creates a run)
# ============================================================
with mlflow.start_run(run_name="gbm_production_candidate") as run:
    model = GradientBoostingClassifier(
        n_estimators=200, learning_rate=0.05, max_depth=5, random_state=42
    )
    model.fit(X_train, y_train)
    
    accuracy = accuracy_score(y_test, model.predict(X_test))
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_params({
        "n_estimators": 200,
        "learning_rate": 0.05,
        "max_depth": 5,
    })
    
    # Log the model
    mlflow.sklearn.log_model(
        model, 
        "model",
        registered_model_name="wine-classifier",  # Auto-registers!
    )
    
    run_id = run.info.run_id
    print(f"Run ID: {run_id}")
    print(f"Accuracy: {accuracy:.4f}")

# ============================================================
# STEP 2: Register a model (if not auto-registered above)
# ============================================================

# Method 1: Register from a run (already done above with registered_model_name)

# Method 2: Register manually from a run URI
# result = mlflow.register_model(
#     model_uri=f"runs:/{run_id}/model",
#     name="wine-classifier",
# )
# print(f"Registered version: {result.version}")

# Method 3: Create registered model first, then add versions
# client.create_registered_model(
#     name="wine-classifier-v2",
#     description="Wine quality classification model",
#     tags={"team": "data-science", "task": "classification"},
# )

# ============================================================
# STEP 3: Manage Model Versions
# ============================================================

# Get all versions of a model
model_name = "wine-classifier"
# versions = client.search_model_versions(f"name='{model_name}'")
# for v in versions:
#     print(f"Version {v.version}: stage={v.current_stage}, run_id={v.run_id}")

# Update description
# client.update_model_version(
#     name=model_name,
#     version=1,
#     description="GBM with 200 trees, trained on wine dataset v2"
# )

# Add tags to a version
# client.set_model_version_tag(model_name, version="1", key="validated", value="true")
# client.set_model_version_tag(model_name, version="1", key="dataset", value="wine-v2")

# ============================================================
# STEP 4: Stage Transitions (Legacy approach)
# ============================================================

# Promote to Staging
# client.transition_model_version_stage(
#     name=model_name,
#     version=1,
#     stage="Staging",
#     archive_existing_versions=False,  # Keep other Staging versions
# )

# After validation, promote to Production
# client.transition_model_version_stage(
#     name=model_name,
#     version=1,
#     stage="Production",
#     archive_existing_versions=True,  # Archive previous Production version
# )

# ============================================================
# STEP 4b: Model Aliases (Modern approach — MLflow 2.x+)
# ============================================================

# Set aliases (more flexible than fixed stages)
# client.set_registered_model_alias(model_name, alias="champion", version=1)
# client.set_registered_model_alias(model_name, alias="challenger", version=2)

# Load model by alias
# champion = mlflow.sklearn.load_model(f"models:/{model_name}@champion")
# challenger = mlflow.sklearn.load_model(f"models:/{model_name}@challenger")

# ============================================================
# STEP 5: Load Models from Registry for Serving
# ============================================================

# Load by version number
# model_v1 = mlflow.sklearn.load_model(f"models:/{model_name}/1")

# Load by stage (legacy)
# prod_model = mlflow.sklearn.load_model(f"models:/{model_name}/Production")

# Load by alias (modern)
# prod_model = mlflow.sklearn.load_model(f"models:/{model_name}@champion")

# ============================================================
# STEP 6: Archive/Delete old versions
# ============================================================

# Archive (keeps for audit, removes from active use)
# client.transition_model_version_stage(model_name, version=1, stage="Archived")

# Delete a version (careful — usually archive instead)
# client.delete_model_version(model_name, version=1)

# Delete entire registered model (all versions)
# client.delete_registered_model(model_name)

print("\nModel Registry workflow complete!")
print(f"Model '{model_name}' registered with version 1")
```

### Automated Model Promotion Pipeline

```python
"""
Automated model promotion — checks quality gates before promoting.
This is how mature teams handle model deployment.
"""

import mlflow
from mlflow.tracking import MlflowClient
from typing import Dict, Optional

client = MlflowClient()


def check_model_quality_gates(
    model_name: str,
    version: str,
    min_accuracy: float = 0.90,
    max_latency_ms: float = 100,
    min_test_samples: int = 1000,
) -> Dict[str, bool]:
    """
    Validate a model version against quality gates.
    ALL gates must pass for promotion to production.
    """
    # Get model version info
    model_version = client.get_model_version(model_name, version)
    run_id = model_version.run_id
    
    # Get run metrics
    run = client.get_run(run_id)
    metrics = run.data.metrics
    
    gates = {}
    
    # Gate 1: Minimum accuracy
    accuracy = metrics.get("accuracy", 0)
    gates["accuracy_gate"] = accuracy >= min_accuracy
    print(f"  Accuracy: {accuracy:.4f} >= {min_accuracy} → {'PASS' if gates['accuracy_gate'] else 'FAIL'}")
    
    # Gate 2: Inference latency
    latency = metrics.get("inference_latency_p95_ms", float("inf"))
    gates["latency_gate"] = latency <= max_latency_ms
    print(f"  Latency (p95): {latency:.1f}ms <= {max_latency_ms}ms → {'PASS' if gates['latency_gate'] else 'FAIL'}")
    
    # Gate 3: Sufficient test samples
    n_samples = metrics.get("n_test_samples", 0)
    gates["sample_gate"] = n_samples >= min_test_samples
    print(f"  Test samples: {n_samples} >= {min_test_samples} → {'PASS' if gates['sample_gate'] else 'FAIL'}")
    
    # Gate 4: Better than current production model
    gates["improvement_gate"] = True  # Default pass if no prod model
    try:
        # Get current production model's accuracy
        prod_versions = client.get_latest_versions(model_name, stages=["Production"])
        if prod_versions:
            prod_run = client.get_run(prod_versions[0].run_id)
            prod_accuracy = prod_run.data.metrics.get("accuracy", 0)
            gates["improvement_gate"] = accuracy > prod_accuracy
            print(f"  vs Production: {accuracy:.4f} > {prod_accuracy:.4f} → {'PASS' if gates['improvement_gate'] else 'FAIL'}")
        else:
            print(f"  vs Production: No current prod model → PASS")
    except Exception:
        print(f"  vs Production: Could not compare → PASS (default)")
    
    return gates


def promote_model(
    model_name: str,
    version: str,
    target_stage: str = "Production",
) -> bool:
    """
    Promote a model version after passing quality gates.
    Returns True if promotion succeeded.
    """
    print(f"\n{'='*60}")
    print(f"PROMOTION CHECK: {model_name} v{version} → {target_stage}")
    print(f"{'='*60}")
    
    # Run quality gates
    gates = check_model_quality_gates(model_name, version)
    
    all_passed = all(gates.values())
    
    if all_passed:
        print(f"\n✅ ALL GATES PASSED — Promoting to {target_stage}")
        # client.transition_model_version_stage(
        #     name=model_name,
        #     version=version,
        #     stage=target_stage,
        #     archive_existing_versions=True,
        # )
        
        # Add promotion metadata
        # client.set_model_version_tag(
        #     model_name, version, 
        #     "promoted_to", target_stage
        # )
        # client.set_model_version_tag(
        #     model_name, version,
        #     "promotion_timestamp", datetime.now().isoformat()
        # )
        return True
    else:
        failed = [k for k, v in gates.items() if not v]
        print(f"\n❌ PROMOTION BLOCKED — Failed gates: {failed}")
        return False


# Example usage
# promote_model("wine-classifier", version="3", target_stage="Production")
print("\nAutomated promotion pipeline ready!")
print("Quality gates: accuracy, latency, sample size, improvement over prod")
```

---

## Versioning Data

### Why Data Versioning is Different

```
┌─────────────────────────────────────────────────────────────┐
│            WHY GIT CAN'T VERSION DATA                        │
│                                                             │
│  Git is designed for:          Data needs:                  │
│  ✓ Small text files            ✗ Large binary files         │
│  ✓ Line-by-line diffs          ✗ Entire file diffs          │
│  ✓ < 100MB repos               ✗ GB/TB datasets             │
│  ✓ Full copy in each clone     ✗ Selective download          │
│  ✓ Merge conflicts             ✗ No meaningful merges        │
│                                                             │
│  Solution: Git stores POINTERS, external storage holds DATA │
│                                                             │
│  ┌─────┐    ┌──────────┐    ┌─────────────────────┐        │
│  │ Git │───►│ .dvc file│───►│ S3/GCS/Azure/local  │        │
│  │     │    │ (50 bytes│    │ (actual 5GB dataset) │        │
│  │     │    │  pointer)│    │                      │        │
│  └─────┘    └──────────┘    └─────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Data Versioning Approaches Compared

| Approach | Tool | Best For | Complexity |
|----------|------|----------|------------|
| **Pointer files** | DVC | Files/directories, any storage | Low |
| **Data lake versioning** | LakeFS | Large-scale data lakes, S3-compatible | Medium |
| **Delta tables** | Delta Lake | Tabular data, Spark ecosystem | Medium |
| **Database snapshots** | Custom | Relational data | High |
| **Hash-based** | Custom scripts | Simple projects | Low |

### Data Versioning with Hashing (DIY Approach)

```python
"""
DIY Data Versioning — When you can't use DVC.
Simple but effective hash-based approach.
"""

import hashlib
import json
import os
from pathlib import Path
from datetime import datetime
from typing import Dict, Optional
import shutil


class SimpleDataVersioner:
    """
    Lightweight data versioning using content hashing.
    Good for understanding the concept; use DVC in production.
    """
    
    def __init__(self, storage_dir: str = ".data_versions"):
        self.storage_dir = Path(storage_dir)
        self.storage_dir.mkdir(exist_ok=True)
        self.manifest_path = self.storage_dir / "manifest.json"
        self.manifest = self._load_manifest()
    
    def _load_manifest(self) -> Dict:
        """Load version manifest."""
        if self.manifest_path.exists():
            with open(self.manifest_path) as f:
                return json.load(f)
        return {"versions": {}}
    
    def _save_manifest(self):
        """Save version manifest."""
        with open(self.manifest_path, "w") as f:
            json.dump(self.manifest, f, indent=2)
    
    def _compute_hash(self, filepath: str) -> str:
        """Compute MD5 hash of a file (content-based addressing)."""
        md5 = hashlib.md5()
        with open(filepath, "rb") as f:
            for chunk in iter(lambda: f.read(8192), b""):
                md5.update(chunk)
        return md5.hexdigest()
    
    def version(
        self, 
        filepath: str, 
        tag: str, 
        description: str = ""
    ) -> str:
        """
        Create a version of a data file.
        
        Args:
            filepath: Path to the data file
            tag: Version tag (e.g., "v1.0", "2024-01-15")
            description: What changed in this version
            
        Returns:
            Version hash
        """
        file_hash = self._compute_hash(filepath)
        file_size = os.path.getsize(filepath)
        
        # Store in version directory
        version_dir = self.storage_dir / file_hash[:2] / file_hash[2:]
        version_dir.mkdir(parents=True, exist_ok=True)
        dest = version_dir / Path(filepath).name
        
        if not dest.exists():
            shutil.copy2(filepath, dest)
        
        # Record in manifest
        self.manifest["versions"][tag] = {
            "hash": file_hash,
            "filepath": str(filepath),
            "size_bytes": file_size,
            "timestamp": datetime.now().isoformat(),
            "description": description,
            "stored_at": str(dest),
        }
        self._save_manifest()
        
        print(f"Versioned: {filepath}")
        print(f"  Tag: {tag}")
        print(f"  Hash: {file_hash}")
        print(f"  Size: {file_size:,} bytes")
        
        return file_hash
    
    def restore(self, tag: str, output_path: Optional[str] = None) -> str:
        """Restore a specific version of data."""
        if tag not in self.manifest["versions"]:
            raise ValueError(f"Version '{tag}' not found")
        
        version_info = self.manifest["versions"][tag]
        stored_path = version_info["stored_at"]
        
        if output_path is None:
            output_path = version_info["filepath"]
        
        shutil.copy2(stored_path, output_path)
        print(f"Restored version '{tag}' to {output_path}")
        return output_path
    
    def list_versions(self):
        """List all versions."""
        print(f"\n{'Tag':<15} {'Hash':<12} {'Size':>12} {'Date':<20} Description")
        print("-" * 80)
        for tag, info in self.manifest["versions"].items():
            print(
                f"{tag:<15} {info['hash'][:10]:<12} "
                f"{info['size_bytes']:>10,}B "
                f"{info['timestamp'][:19]:<20} "
                f"{info.get('description', '')}"
            )
    
    def diff(self, tag1: str, tag2: str) -> bool:
        """Check if two versions have different content."""
        v1 = self.manifest["versions"].get(tag1, {})
        v2 = self.manifest["versions"].get(tag2, {})
        same = v1.get("hash") == v2.get("hash")
        print(f"Versions '{tag1}' and '{tag2}': {'IDENTICAL' if same else 'DIFFERENT'}")
        return not same


# Example usage
versioner = SimpleDataVersioner()

# Version initial dataset
# versioner.version("data/train.csv", tag="v1.0", description="Initial 10K samples")

# After collecting more data
# versioner.version("data/train.csv", tag="v2.0", description="Added Jan 2024 data, 25K samples")

# List all versions
# versioner.list_versions()

# Restore old version
# versioner.restore("v1.0", output_path="data/train_v1.csv")

print("SimpleDataVersioner ready!")
print("Use DVC for production — this is for learning the concept.")
```

---

## Versioning Models

### Model Versioning Strategies

```
┌──────────────────────────────────────────────────────────────────┐
│              MODEL VERSIONING STRATEGIES                          │
│                                                                  │
│  Strategy 1: File-based (simple)                                 │
│  models/                                                         │
│  ├── model_v1_20240115.pkl                                       │
│  ├── model_v2_20240201.pkl                                       │
│  └── model_v3_20240215.pkl                                       │
│  Problem: No metadata, no stage management, no lineage           │
│                                                                  │
│  Strategy 2: DVC-tracked (better)                                │
│  models/                                                         │
│  ├── model.pkl            (actual file — DVC tracked)            │
│  └── model.pkl.dvc        (pointer — Git tracked)                │
│  Better: Linked to Git history, versioned with code              │
│                                                                  │
│  Strategy 3: Model Registry (best)                               │
│  MLflow Registry:                                                │
│  ├── "fraud-detector" v1 → Archived                              │
│  ├── "fraud-detector" v2 → Production (@champion)                │
│  └── "fraud-detector" v3 → Staging (@challenger)                 │
│  Best: Full lifecycle management, approval workflows             │
└──────────────────────────────────────────────────────────────────┘
```

### Model Serialization Formats

```python
"""
How to save models in different formats.
The format matters for versioning, serving, and portability.
"""

import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_iris

# Train a sample model
iris = load_iris()
model = RandomForestClassifier(n_estimators=100, random_state=42)
model.fit(iris.data, iris.target)

# ============================================================
# FORMAT 1: Pickle / Joblib (Python-native)
# ============================================================
import joblib
import pickle

# Joblib (preferred for sklearn — handles numpy arrays better)
joblib.dump(model, "model_joblib.pkl")
loaded_model = joblib.load("model_joblib.pkl")

# Pickle (standard Python serialization)
with open("model_pickle.pkl", "wb") as f:
    pickle.dump(model, f)

# Pros: Simple, preserves full Python object
# Cons: Python-version dependent, security risks (arbitrary code exec),
#        not portable across languages

# ============================================================
# FORMAT 2: ONNX (Open Neural Network Exchange)
# ============================================================
# pip install skl2onnx onnxruntime
"""
from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType
import onnxruntime as rt

# Convert sklearn model to ONNX
initial_type = [('float_input', FloatTensorType([None, 4]))]
onnx_model = convert_sklearn(model, initial_types=initial_type)

# Save
with open("model.onnx", "wb") as f:
    f.write(onnx_model.SerializeToString())

# Load and run
sess = rt.InferenceSession("model.onnx")
pred = sess.run(None, {"float_input": iris.data[:5].astype(np.float32)})
"""

# Pros: Language-agnostic, optimized runtime, portable
# Cons: Not all operations supported, conversion can be lossy

# ============================================================
# FORMAT 3: MLflow Model Format (recommended for MLOps)
# ============================================================
"""
import mlflow.sklearn

# Save in MLflow format
mlflow.sklearn.save_model(model, "mlflow_model")

# This creates:
# mlflow_model/
# ├── MLmodel           (metadata: flavor, signature, etc.)
# ├── model.pkl         (actual model)
# ├── conda.yaml        (environment specification)
# ├── requirements.txt  (pip requirements)
# └── input_example.json (sample input)

# Load from MLflow format
loaded = mlflow.sklearn.load_model("mlflow_model")
"""

# Pros: Framework-agnostic loading, includes environment, 
#        integrates with registry
# Cons: MLflow dependency

# ============================================================
# FORMAT COMPARISON
# ============================================================
comparison = """
| Format      | Portable | Includes Env | Serving Ready | Size    |
|-------------|----------|-------------|---------------|---------|
| Pickle/Joblib| Python only | No       | No            | Small   |
| ONNX        | Any language | No       | Yes (optimized)| Medium  |
| MLflow      | Python (multi-framework) | Yes | Yes     | Medium  |
| SavedModel (TF) | Any language | No   | Yes (TF Serving)| Large |
| TorchScript | C++/Python | No        | Yes (TorchServe)| Medium |
| PMML        | Any language | No       | Yes           | Large   |
"""

print("Model Format Comparison:")
print(comparison)
```

---

## Versioning Code + Config + Environment

### The Complete Versioning Stack

```python
"""
Everything you need to version for full reproducibility.
"""

# ============================================================
# 1. CODE VERSIONING (Git — you already know this)
# ============================================================
code_versioning = """
# Standard Git workflow
git add src/train.py src/features.py
git commit -m "Add text feature extraction"
git tag model-v2.1

# Pro Tips:
# - Use conventional commits: "feat:", "fix:", "refactor:"
# - Tag model releases: git tag -a model-v2.1 -m "Added text features"
# - Use branches for experiments: git checkout -b experiment/bert-features
"""

# ============================================================
# 2. CONFIG VERSIONING (Git-tracked YAML/JSON)
# ============================================================
config_example = """
# configs/model_config.yaml
# This file is tracked in Git — every change is versioned

model:
  type: GradientBoosting
  hyperparameters:
    n_estimators: 200
    learning_rate: 0.05
    max_depth: 5
    subsample: 0.8

data:
  source: "s3://data-lake/fraud/v3/"
  train_split: 0.7
  val_split: 0.15
  test_split: 0.15
  
features:
  numerical:
    - amount
    - balance
    - transaction_count_30d
  categorical:
    - merchant_category
    - channel
  text:
    - description  # NEW in this version
    
preprocessing:
  scaling: standard
  handle_missing: median
  text_vectorizer: tfidf
  max_text_features: 5000

serving:
  batch_size: 64
  max_latency_ms: 50
  min_confidence: 0.7
"""

# ============================================================
# 3. ENVIRONMENT VERSIONING
# ============================================================

# Method 1: requirements.txt (basic)
requirements = """
# requirements.txt — Pin EXACT versions!
scikit-learn==1.3.2       # NOT scikit-learn>=1.3
pandas==2.1.4
numpy==1.24.3
mlflow==2.9.2
xgboost==2.0.3
"""

# Method 2: Conda environment (better — includes system deps)
conda_env = """
# environment.yml
name: fraud-detection
channels:
  - defaults
  - conda-forge
dependencies:
  - python=3.10.13
  - scikit-learn=1.3.2
  - pandas=2.1.4
  - numpy=1.24.3
  - pip:
    - mlflow==2.9.2
    - xgboost==2.0.3
"""

# Method 3: Docker (best — reproducible everything)
dockerfile = """
# Dockerfile — Complete environment reproducibility
FROM python:3.10.13-slim

# System dependencies
RUN apt-get update && apt-get install -y \\
    libgomp1 \\
    && rm -rf /var/lib/apt/lists/*

# Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application code
COPY src/ /app/src/
COPY configs/ /app/configs/
COPY models/ /app/models/

WORKDIR /app
CMD ["python", "src/serve.py"]
"""

# Method 4: pip freeze (snapshot current env)
freeze_commands = """
# Capture exact current environment
pip freeze > requirements-lock.txt

# Better: use pip-tools for dependency resolution
pip install pip-tools
pip-compile requirements.in   # Resolves and pins all deps
pip-sync requirements.txt     # Installs exact versions
"""

print("Versioning Stack:")
print("1. Code    → Git (commits, tags, branches)")
print("2. Config  → Git (YAML/JSON files)")  
print("3. Data    → DVC (pointer files in Git, data in remote)")
print("4. Model   → MLflow Registry (or DVC)")
print("5. Environ → Docker (or conda lock + requirements.txt)")
```

---

## End-to-End Lineage

### What is Model Lineage

**Model lineage** is the ability to trace any prediction back to exactly what code, data, config, and environment produced the model that made it.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   END-TO-END LINEAGE                                 │
│                                                                     │
│  Prediction: "This transaction is FRAUDULENT (confidence: 0.94)"    │
│                              │                                      │
│                              │ Which model?                         │
│                              ▼                                      │
│  Model: fraud-detector v3 (alias: @champion)                        │
│         Created: 2024-01-15 14:30:22                                │
│         MLflow Run ID: abc123                                       │
│                              │                                      │
│                              │ What code?                           │
│                              ▼                                      │
│  Code: Git commit a3f7b2c                                           │
│        Branch: main                                                 │
│        File: src/train.py (line 45-120)                             │
│                              │                                      │
│                              │ What data?                           │
│                              ▼                                      │
│  Data: DVC hash md5:9f86d08...                                      │
│        Source: s3://data-lake/fraud/v3/                              │
│        Size: 2.3M rows, 45 features                                 │
│        Date range: 2023-01 to 2023-12                               │
│                              │                                      │
│                              │ What config?                         │
│                              ▼                                      │
│  Config: params.yaml @ commit a3f7b2c                               │
│          n_estimators=200, lr=0.05, max_depth=5                     │
│                              │                                      │
│                              │ What environment?                    │
│                              ▼                                      │
│  Environment: Docker image fraud-model:a3f7b2c                      │
│               Python 3.10.13, sklearn 1.3.2, xgboost 2.0.3         │
│               GPU: None (CPU-only)                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementing Lineage Tracking

```python
"""
Practical lineage tracking — connecting all the dots.
"""

import json
import subprocess
import hashlib
from datetime import datetime
from pathlib import Path
from typing import Dict, Any


class LineageTracker:
    """
    Track the complete lineage of a model training run.
    Captures code, data, config, environment, and results.
    """
    
    def __init__(self, run_name: str):
        self.run_name = run_name
        self.lineage: Dict[str, Any] = {
            "run_name": run_name,
            "timestamp": datetime.now().isoformat(),
            "code": {},
            "data": {},
            "config": {},
            "environment": {},
            "model": {},
            "metrics": {},
        }
    
    def track_code(self):
        """Capture code version information."""
        try:
            git_hash = subprocess.check_output(
                ["git", "rev-parse", "HEAD"]
            ).decode().strip()
            
            git_branch = subprocess.check_output(
                ["git", "rev-parse", "--abbrev-ref", "HEAD"]
            ).decode().strip()
            
            # Check for uncommitted changes
            git_status = subprocess.check_output(
                ["git", "status", "--porcelain"]
            ).decode().strip()
            
            self.lineage["code"] = {
                "git_commit": git_hash,
                "git_branch": git_branch,
                "has_uncommitted_changes": len(git_status) > 0,
                "uncommitted_files": git_status.split("\n") if git_status else [],
            }
            
            if git_status:
                print("⚠️  WARNING: Uncommitted changes detected!")
                print("   Lineage may not be fully reproducible.")
                
        except subprocess.CalledProcessError:
            self.lineage["code"] = {"error": "Not a Git repository"}
    
    def track_data(self, data_paths: list):
        """Capture data version information."""
        data_info = []
        for path in data_paths:
            p = Path(path)
            if p.exists():
                # Compute content hash
                md5 = hashlib.md5()
                with open(p, "rb") as f:
                    for chunk in iter(lambda: f.read(8192), b""):
                        md5.update(chunk)
                
                data_info.append({
                    "path": str(p),
                    "hash": md5.hexdigest(),
                    "size_bytes": p.stat().st_size,
                    "modified": datetime.fromtimestamp(
                        p.stat().st_mtime
                    ).isoformat(),
                })
        
        self.lineage["data"] = data_info
    
    def track_config(self, config: Dict):
        """Capture configuration/hyperparameters."""
        self.lineage["config"] = config
    
    def track_environment(self):
        """Capture environment information."""
        import sys
        import platform
        
        # Get installed packages
        try:
            pip_list = subprocess.check_output(
                [sys.executable, "-m", "pip", "list", "--format=json"]
            ).decode()
            packages = {
                p["name"]: p["version"] 
                for p in json.loads(pip_list)
            }
        except Exception:
            packages = {}
        
        self.lineage["environment"] = {
            "python_version": sys.version,
            "platform": platform.platform(),
            "processor": platform.processor(),
            "key_packages": {
                k: packages.get(k, "not installed")
                for k in ["scikit-learn", "pandas", "numpy", 
                          "torch", "tensorflow", "xgboost", "mlflow"]
            },
        }
    
    def track_model(self, model_path: str, model_type: str):
        """Capture model artifact information."""
        p = Path(model_path)
        if p.exists():
            md5 = hashlib.md5(p.read_bytes()).hexdigest()
            self.lineage["model"] = {
                "path": str(p),
                "hash": md5,
                "size_bytes": p.stat().st_size,
                "type": model_type,
            }
    
    def track_metrics(self, metrics: Dict[str, float]):
        """Capture evaluation metrics."""
        self.lineage["metrics"] = metrics
    
    def save(self, output_dir: str = "lineage"):
        """Save complete lineage record."""
        output_path = Path(output_dir)
        output_path.mkdir(exist_ok=True)
        
        filename = f"{self.run_name}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
        filepath = output_path / filename
        
        with open(filepath, "w") as f:
            json.dump(self.lineage, f, indent=2, default=str)
        
        print(f"\nLineage saved to: {filepath}")
        return str(filepath)
    
    def display(self):
        """Print lineage summary."""
        print(f"\n{'='*60}")
        print(f"LINEAGE RECORD: {self.run_name}")
        print(f"{'='*60}")
        print(f"Timestamp: {self.lineage['timestamp']}")
        
        if self.lineage["code"]:
            print(f"\nCode:")
            print(f"  Git commit: {self.lineage['code'].get('git_commit', 'N/A')}")
            print(f"  Branch: {self.lineage['code'].get('git_branch', 'N/A')}")
        
        if self.lineage["data"]:
            print(f"\nData ({len(self.lineage['data'])} files):")
            for d in self.lineage["data"]:
                print(f"  {d['path']}: hash={d['hash'][:10]}... ({d['size_bytes']:,}B)")
        
        if self.lineage["config"]:
            print(f"\nConfig:")
            for k, v in self.lineage["config"].items():
                print(f"  {k}: {v}")
        
        if self.lineage["metrics"]:
            print(f"\nMetrics:")
            for k, v in self.lineage["metrics"].items():
                print(f"  {k}: {v}")
        
        print(f"{'='*60}")


# ============================================================
# USAGE EXAMPLE
# ============================================================

# Initialize tracker
tracker = LineageTracker("fraud_detector_v3")

# Track everything
tracker.track_code()
tracker.track_environment()
tracker.track_config({
    "n_estimators": 200,
    "learning_rate": 0.05,
    "max_depth": 5,
    "features": "all_v5",
})
tracker.track_metrics({
    "accuracy": 0.9534,
    "f1_score": 0.9487,
    "precision": 0.9512,
    "recall": 0.9463,
})

# Display and save
tracker.display()
# tracker.save()
```

---

## Common Mistakes

### 1. Only Versioning Code (Forgetting Data and Config)

❌ **Wrong:** Committing only `train.py` changes
✅ **Right:** Versioning code + data + config + environment together

> If you can't reproduce the exact data that trained a model, you can't truly reproduce the model.

### 2. Not Pinning Dependency Versions

❌ **Wrong:** `requirements.txt` with `scikit-learn>=1.0`
✅ **Right:** `scikit-learn==1.3.2` (exact version!)

```
# Real horror story:
# sklearn 1.3 changed default parameters for some estimators.
# Same code + same data + different sklearn version = different model!
```

### 3. Storing Large Files in Git

❌ **Wrong:** `git add data/training_set.csv` (500MB file)
✅ **Right:** `dvc add data/training_set.csv` → Git only stores the pointer

> Git stores EVERY version of every file. A 500MB dataset with 10 versions = 5GB Git repo. Your colleagues will hate you.

### 4. No Clear Stage/Promotion Process

❌ **Wrong:** Anyone can push any model to production
✅ **Right:** Defined quality gates: None → Staging (automated tests) → Production (approval)

### 5. Forgetting to Version the Feature Pipeline

❌ **Wrong:** Versioning the model but not the feature engineering code
✅ **Right:** The model is useless without its matching feature pipeline — version them together

```
Common disaster:
1. Train model with features computed by feature_v2.py
2. Deploy model
3. Production pipeline still uses feature_v1.py
4. Model receives wrong features → garbage predictions
5. Nobody knows why because "the model is the same"
```

### 6. Manual Version Naming

❌ **Wrong:** `model_final.pkl`, `model_final_v2.pkl`, `model_REALLY_final.pkl`
✅ **Right:** Automated versioning via registry (v1, v2, v3...) or content-hash naming

### 7. Not Testing Model Loading

❌ **Wrong:** Save model, assume it loads correctly
✅ **Right:** Always test: save → load → predict → compare results

```python
# Always verify model serialization round-trip!
import numpy as np
import joblib

# Save
joblib.dump(model, "model.pkl")

# Load
loaded_model = joblib.load("model.pkl")

# Verify predictions match
original_preds = model.predict(X_test)
loaded_preds = loaded_model.predict(X_test)
assert np.array_equal(original_preds, loaded_preds), \
    "Model serialization changed predictions!"
print("✅ Model serialization verified")
```

---

## Interview Questions

### Technical Questions

**Q1: What is model versioning and why is it important?**
> **A:** Model versioning is systematically tracking different versions of ML artifacts (models, data, configs, code) to enable reproducibility, comparison, rollback, and auditing. It's critical because ML systems have three axes of change (code, data, model) — any of which can cause performance changes. Without versioning, you can't reproduce results, roll back safely, or satisfy regulatory requirements.

**Q2: What is DVC and how does it work?**
> **A:** DVC (Data Version Control) extends Git to handle large files and ML pipelines. It works by storing small pointer files (.dvc) in Git that reference large files stored in remote storage (S3, GCS, etc.). The pointer contains a content hash (MD5) of the actual data. When you `dvc push`, data goes to remote storage. When you `dvc pull`, data is downloaded based on the pointer. DVC also supports pipeline definitions (dvc.yaml) for reproducible ML workflows.

**Q3: Explain the model registry concept and its stages.**
> **A:** A model registry is a centralized store for managing model versions throughout their lifecycle. Key stages: (1) **None** — just registered, not validated. (2) **Staging** — being tested via A/B testing or shadow deployment. (3) **Production** — actively serving predictions, the "champion" model. (4) **Archived** — retired, kept for audit trail. Modern approaches (MLflow 2.x+) use flexible aliases like `@champion` and `@challenger` instead of fixed stages.

**Q4: How would you ensure reproducibility of an ML experiment?**
> **A:** Version all five components: (1) **Code** — Git with commit hash, (2) **Data** — DVC or data versioning with content hash, (3) **Config** — YAML files in Git with all hyperparameters, (4) **Environment** — Docker image or pinned requirements.txt, (5) **Random seeds** — set and log all seeds. Use an experiment tracker (MLflow) to link all five together for each run. Additionally, validate round-trip serialization of the model.

**Q5: Compare DVC with Git LFS. When would you use each?**
> **A:** **Git LFS**: Stores large files on a separate server but still tracks them in Git. Better for large code assets (game assets, design files). Limited to simple file tracking. **DVC**: Designed specifically for ML — supports pipelines (dvc.yaml), metrics comparison, experiment management, multiple storage backends, and integrates with ML workflow. Use Git LFS for non-ML large files; use DVC for ML data and model versioning.

### Scenario Questions

**Q6: A model in production is performing poorly. Walk me through the investigation using versioning tools.**
> **A:** (1) Check model registry — which version is deployed? When was it deployed? (2) Pull the lineage — what data, code, config produced this model? (3) `dvc diff` — has the input data changed since training? (4) Compare metrics — `dvc metrics diff` or MLflow comparison between current and previous version. (5) Check data drift — compare training data distribution vs current production data. (6) If data drifted, retrain on recent data. (7) If code bug, `git diff` between versions. (8) Roll back to previous version via registry while investigating.

**Q7: Your team of 5 data scientists needs a versioning strategy. What do you recommend?**
> **A:** Stack: Git (code + config) + DVC (data + pipeline) + MLflow Registry (models). Workflow: (1) Each experiment on a Git branch with DVC-tracked data, (2) DVC pipelines define reproducible training workflows, (3) Best models registered in MLflow, (4) Quality gates (accuracy, latency, fairness) before promotion to Staging, (5) A/B test in Staging, (6) Promote to Production with `archive_existing=True`, (7) Lineage tracked automatically via MLflow run IDs and DVC hashes.

**Q8: How do you handle model rollback in production?**
> **A:** With a proper registry: (1) Identify the last known good version in the registry, (2) Update the production alias/stage to point to that version — `client.set_registered_model_alias(name, "champion", old_version)`, (3) The serving infrastructure loads the new (old) model, (4) Verify rollback with monitoring, (5) Investigate the bad model using lineage. Key: rollback should be a single command/click, not a re-training. This requires that all model versions are stored with their dependencies (feature pipeline version, preprocessing).

---

## Quick Reference

### DVC Cheat Sheet

```bash
# === SETUP ===
dvc init                                    # Initialize in Git repo
dvc remote add -d store s3://bucket/path    # Configure remote storage

# === TRACK DATA ===
dvc add data/train.csv                      # Track file
dvc add data/images/                        # Track directory
git add data/train.csv.dvc data/.gitignore  # Commit pointer
dvc push                                    # Upload to remote

# === VERSION CONTROL ===
git tag data-v1                             # Tag version
dvc checkout                                # Sync data to current Git state
dvc pull                                    # Download data from remote

# === PIPELINES ===
dvc repro                                   # Run pipeline (smart caching)
dvc dag                                     # Visualize pipeline DAG
dvc metrics show                            # Show metrics
dvc metrics diff                            # Compare metrics between versions
dvc params diff                             # Compare params between versions
dvc plots show                              # Generate plots

# === EXPERIMENTS ===
dvc exp run --set-param train.lr=0.01       # Run experiment
dvc exp show                                # Show all experiments
dvc exp diff                                # Compare experiments
dvc exp apply exp-abc123                    # Apply best experiment
```

### MLflow Registry Cheat Sheet

```python
from mlflow.tracking import MlflowClient
client = MlflowClient()

# === REGISTER ===
# Auto-register during logging:
mlflow.sklearn.log_model(model, "model", registered_model_name="my-model")

# Manual register:
mlflow.register_model(f"runs:/{run_id}/model", "my-model")

# === MANAGE VERSIONS ===
client.search_model_versions("name='my-model'")        # List versions
client.update_model_version("my-model", "1", description="...") 
client.set_model_version_tag("my-model", "1", "key", "val")

# === STAGE TRANSITIONS (legacy) ===
client.transition_model_version_stage("my-model", "1", "Staging")
client.transition_model_version_stage("my-model", "1", "Production")

# === ALIASES (modern, MLflow 2.x+) ===
client.set_registered_model_alias("my-model", "champion", version=1)
client.set_registered_model_alias("my-model", "challenger", version=2)

# === LOAD FROM REGISTRY ===
model = mlflow.sklearn.load_model("models:/my-model/1")           # By version
model = mlflow.sklearn.load_model("models:/my-model/Production")  # By stage
model = mlflow.sklearn.load_model("models:/my-model@champion")    # By alias
```

### Versioning Strategy Summary

| What to Version | Tool | Where Stored | When to Update |
|-----------------|------|-------------|----------------|
| Source code | Git | GitHub/GitLab | Every commit |
| Configuration | Git (YAML) | GitHub/GitLab | When params change |
| Training data | DVC | S3/GCS/Azure | When data changes |
| Features | DVC / Feature Store | Remote storage | When pipeline changes |
| Trained model | MLflow Registry | Artifact store | After training |
| Environment | Docker / pip freeze | Container registry | When deps change |
| Pipeline definition | Git (dvc.yaml) | GitHub/GitLab | When pipeline changes |
| Metrics/Results | MLflow / DVC | Tracking server | After evaluation |

### Golden Rules

1. **Version everything together** — Code, data, model, config, and environment must be linked
2. **Never store large files in Git** — Use DVC, LakeFS, or artifact stores
3. **Pin exact dependency versions** — `==` not `>=`
4. **Automate promotion gates** — No manual "LGTM" for production models
5. **Test serialization round-trips** — Save → Load → Predict → Compare
6. **Keep lineage records** — Every prediction should trace back to its origin
7. **Make rollback a one-command operation** — If it takes more than 5 minutes, it's not safe enough

---

*Previous: [02-Experiment-Tracking.md](02-Experiment-Tracking.md)*
*Next: [04-Model-Serving.md](04-Model-Serving.md) — REST APIs, TF Serving, TorchServe, and serving at scale*
