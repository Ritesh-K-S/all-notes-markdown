# Experiment Tracking

## Table of Contents
- [What is Experiment Tracking](#what-is-experiment-tracking)
- [Why Experiment Tracking Matters](#why-experiment-tracking-matters)
- [How Experiment Tracking Works](#how-experiment-tracking-works)
- [MLflow — Complete Guide](#mlflow--complete-guide)
- [Weights & Biases (W&B)](#weights--biases-wb)
- [TensorBoard](#tensorboard)
- [Experiment Organization Best Practices](#experiment-organization-best-practices)
- [Comparing Tools](#comparing-tools)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Experiment Tracking

### Simple Explanation

Imagine you're a chef trying to perfect a recipe. You try different amounts of salt, cooking times, and temperatures. After 50 attempts, you finally make the perfect dish — but you forgot what you did differently. Was it 350°F for 25 minutes or 375°F for 20 minutes?

**Experiment tracking** is like a detailed lab notebook for your ML experiments. Every time you train a model, you automatically record:
- What settings (hyperparameters) you used
- What data you trained on
- What results (metrics) you got
- What artifacts (model files, plots) were produced

### Formal Definition

> **Experiment tracking** is the process of systematically logging all information related to ML experiments — parameters, metrics, code versions, data versions, and artifacts — to enable comparison, reproducibility, and collaboration.

### What Gets Tracked

```
┌─────────────────────────────────────────────────────────────┐
│              ANATOMY OF A TRACKED EXPERIMENT                 │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ RUN: "random_forest_exp_042"                        │    │
│  │                                                     │    │
│  │ Parameters (Inputs):                                │    │
│  │   • n_estimators = 200                              │    │
│  │   • max_depth = 10                                  │    │
│  │   • learning_rate = 0.01                            │    │
│  │   • dataset_version = "v2.3"                        │    │
│  │   • feature_set = "all_v5"                          │    │
│  │                                                     │    │
│  │ Metrics (Outputs):                                  │    │
│  │   • accuracy = 0.943                                │    │
│  │   • f1_score = 0.938                                │    │
│  │   • training_time = 145.2s                          │    │
│  │   • inference_latency_p95 = 12ms                    │    │
│  │                                                     │    │
│  │ Artifacts (Files):                                  │    │
│  │   • model.pkl (trained model)                       │    │
│  │   • confusion_matrix.png                            │    │
│  │   • feature_importance.csv                          │    │
│  │   • requirements.txt                                │    │
│  │                                                     │    │
│  │ Metadata:                                           │    │
│  │   • git_commit = "a3f7b2c"                          │    │
│  │   • timestamp = "2024-01-15 14:30:22"               │    │
│  │   • user = "alice"                                  │    │
│  │   • environment = "gpu-large-01"                    │    │
│  │   • tags = ["baseline", "production-candidate"]     │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## Why Experiment Tracking Matters

### The Chaos Without Tracking

```
Without Experiment Tracking:
─────────────────────────────
"results_final.csv"
"results_final_v2.csv"
"results_ACTUALLY_final.csv"
"results_final_use_this_one.csv"
"model_good.pkl"
"model_better.pkl"  
"model_best_maybe.pkl"

Team chat:
Alice: "Which model is in production?"
Bob: "I think the one from Tuesday?"
Carol: "No, we reverted to Thursday's model"
Alice: "What hyperparameters was that?"
Bob: "Let me check my notebook... oh I cleared the outputs"
```

### Real-World Impact

| Problem | Without Tracking | With Tracking |
|---------|-----------------|---------------|
| "What was our best model?" | Scroll through notebooks | Sort by metric, click |
| "Can you reproduce this?" | "Let me try..." (fails) | One-click reproduce |
| "Why did performance drop?" | Days of investigation | Compare run parameters |
| "Which features helped?" | Re-run experiments | Check feature ablation logs |
| Knowledge sharing | Tribal knowledge | Searchable experiment history |
| Audit/compliance | "We think we used..." | Full lineage trail |

### When You'd Use It

✅ **Always use experiment tracking when:**
- Training any model (even "quick" experiments)
- Comparing multiple approaches
- Working in a team
- Building models for production
- Need to justify model choices to stakeholders

> **Pro Tip:** The cost of NOT tracking is always higher than tracking. A "quick experiment" often becomes the production model, and you'll wish you logged everything.

---

## How Experiment Tracking Works

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                 EXPERIMENT TRACKING ARCHITECTURE                  │
│                                                                  │
│  Training Script                    Tracking Server              │
│  ┌──────────────────┐              ┌──────────────────┐         │
│  │                  │   HTTP/API   │                  │         │
│  │  import mlflow   │─────────────►│  Tracking Server │         │
│  │                  │              │  (stores runs)   │         │
│  │  mlflow.log_*()  │              │                  │         │
│  │                  │              └────────┬─────────┘         │
│  └──────────────────┘                      │                    │
│                                            ▼                    │
│                              ┌──────────────────────┐           │
│                              │   Backend Store       │           │
│                              │   ┌───────┐ ┌──────┐ │           │
│                              │   │Metrics│ │Params│ │           │
│                              │   │  DB   │ │  DB  │ │           │
│                              │   └───────┘ └──────┘ │           │
│                              │   ┌────────────────┐ │           │
│                              │   │ Artifact Store │ │           │
│                              │   │ (S3/GCS/local) │ │           │
│                              │   └────────────────┘ │           │
│                              └──────────────────────┘           │
│                                            │                    │
│                                            ▼                    │
│                              ┌──────────────────────┐           │
│                              │       UI              │           │
│                              │  • Compare runs       │           │
│                              │  • Visualize metrics  │           │
│                              │  • Download artifacts │           │
│                              └──────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### Core Concepts

| Concept | Definition | Analogy |
|---------|------------|---------|
| **Experiment** | A group of related runs (e.g., "fraud detection v2") | A research project |
| **Run** | A single execution of training code | One lab experiment |
| **Parameter** | Input configuration (hyperparameters) | Recipe ingredients |
| **Metric** | Output measurement (accuracy, loss) | Taste test score |
| **Artifact** | File output (model, plot, data) | The cooked dish |
| **Tag** | Metadata label for organization | Post-it note |

---

## MLflow — Complete Guide

### What is MLflow

MLflow is an open-source platform for the complete ML lifecycle. It has four main components:

1. **MLflow Tracking** — Log experiments (our focus here)
2. **MLflow Projects** — Package ML code for reproducibility
3. **MLflow Models** — Package models for deployment
4. **MLflow Registry** — Manage model versions (covered in Chapter 03)

### Installation and Setup

```python
"""
MLflow Setup and Basic Usage
"""

# Installation (run in terminal):
# pip install mlflow

# Start MLflow UI (run in terminal):
# mlflow ui --port 5000
# Then visit http://localhost:5000

import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_wine
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, f1_score, precision_score, recall_score
import numpy as np

# ============================================================
# BASIC SETUP
# ============================================================

# Option 1: Local tracking (default - stores in ./mlruns)
mlflow.set_tracking_uri("file:///path/to/mlruns")

# Option 2: Remote tracking server
# mlflow.set_tracking_uri("http://mlflow-server:5000")

# Option 3: Database backend
# mlflow.set_tracking_uri("sqlite:///mlflow.db")

# Set experiment (creates if doesn't exist)
mlflow.set_experiment("wine-classification")

print(f"Tracking URI: {mlflow.get_tracking_uri()}")
print(f"Artifact URI: {mlflow.get_artifact_uri()}")
```

### Basic Experiment Tracking

```python
"""
MLflow Basic Tracking — The Foundation
Every MLOps engineer needs to know this cold.
"""

import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.datasets import load_wine
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import (
    accuracy_score, f1_score, precision_score, 
    recall_score, confusion_matrix
)
import matplotlib.pyplot as plt
import numpy as np
import json
import time

# Load data
wine = load_wine()
X, y = wine.data, wine.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# Set experiment
mlflow.set_experiment("wine-quality-classification")

# ============================================================
# METHOD 1: Manual logging (full control)
# ============================================================
with mlflow.start_run(run_name="random_forest_baseline"):
    
    # --- Log Parameters ---
    # Log individual parameters
    mlflow.log_param("model_type", "RandomForest")
    mlflow.log_param("n_estimators", 100)
    mlflow.log_param("max_depth", 10)
    mlflow.log_param("random_state", 42)
    
    # Log multiple parameters at once
    mlflow.log_params({
        "test_size": 0.2,
        "dataset": "wine",
        "n_features": X.shape[1],
        "n_samples": X.shape[0],
    })
    
    # --- Train Model ---
    start_time = time.time()
    model = RandomForestClassifier(
        n_estimators=100, max_depth=10, random_state=42
    )
    model.fit(X_train, y_train)
    training_time = time.time() - start_time
    
    # --- Compute Metrics ---
    y_pred = model.predict(X_test)
    
    # Log individual metrics
    mlflow.log_metric("accuracy", accuracy_score(y_test, y_pred))
    mlflow.log_metric("f1_weighted", f1_score(y_test, y_pred, average="weighted"))
    mlflow.log_metric("training_time_seconds", training_time)
    
    # Log multiple metrics
    mlflow.log_metrics({
        "precision_weighted": precision_score(y_test, y_pred, average="weighted"),
        "recall_weighted": recall_score(y_test, y_pred, average="weighted"),
    })
    
    # --- Log Metrics Over Time (e.g., epochs) ---
    # Useful for tracking convergence
    cv_scores = cross_val_score(model, X_train, y_train, cv=5)
    for fold, score in enumerate(cv_scores):
        mlflow.log_metric("cv_accuracy", score, step=fold)
    
    # --- Log Artifacts ---
    # Save confusion matrix as image
    fig, ax = plt.subplots(figsize=(8, 6))
    cm = confusion_matrix(y_test, y_pred)
    ax.imshow(cm, cmap='Blues')
    ax.set_title("Confusion Matrix")
    ax.set_xlabel("Predicted")
    ax.set_ylabel("Actual")
    # Add numbers to cells
    for i in range(cm.shape[0]):
        for j in range(cm.shape[1]):
            ax.text(j, i, str(cm[i, j]), ha='center', va='center')
    plt.tight_layout()
    plt.savefig("confusion_matrix.png", dpi=100)
    plt.close()
    
    # Log the image artifact
    mlflow.log_artifact("confusion_matrix.png")
    
    # Log feature importances as JSON
    feature_importance = dict(zip(
        wine.feature_names, 
        model.feature_importances_.tolist()
    ))
    with open("feature_importance.json", "w") as f:
        json.dump(feature_importance, f, indent=2)
    mlflow.log_artifact("feature_importance.json")
    
    # --- Log Model ---
    # This saves the model in MLflow's format (can be loaded later)
    mlflow.sklearn.log_model(
        model, 
        "model",
        input_example=X_test[:5],  # Saves example input for reference
    )
    
    # --- Tags ---
    mlflow.set_tag("team", "data-science")
    mlflow.set_tag("version", "v1.0")
    mlflow.set_tag("stage", "development")
    
    # Print run info
    run = mlflow.active_run()
    print(f"Run ID: {run.info.run_id}")
    print(f"Accuracy: {accuracy_score(y_test, y_pred):.4f}")
    print(f"Training time: {training_time:.2f}s")
```

**Output:**
```
Run ID: a1b2c3d4e5f6789012345678
Accuracy: 0.9722
Training time: 0.34s
```

### Autologging (Zero-Effort Tracking)

```python
"""
MLflow Autologging — Automatic tracking with zero manual effort.
Supported frameworks: sklearn, pytorch, tensorflow, xgboost, lightgbm, etc.
"""

import mlflow
import mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.datasets import load_wine
from sklearn.model_selection import train_test_split

# ============================================================
# AUTOLOGGING — The Magic Button
# ============================================================

# Enable autologging for sklearn (captures everything automatically)
mlflow.sklearn.autolog(
    log_input_examples=True,   # Save example inputs
    log_model_signatures=True, # Save input/output schema
    log_models=True,           # Save the trained model
    log_datasets=True,         # Log dataset info
)

# Now just train normally — MLflow captures everything!
wine = load_wine()
X_train, X_test, y_train, y_test = train_test_split(
    wine.data, wine.target, test_size=0.2, random_state=42
)

with mlflow.start_run(run_name="auto_logged_gbm"):
    # Just train — MLflow automatically logs:
    # - All hyperparameters (n_estimators, learning_rate, etc.)
    # - Training metrics
    # - Model artifact
    # - Feature importance plot
    # - Input example
    model = GradientBoostingClassifier(
        n_estimators=200,
        learning_rate=0.1,
        max_depth=3,
    )
    model.fit(X_train, y_train)
    
    # You can still manually log additional things
    score = model.score(X_test, y_test)
    mlflow.log_metric("test_accuracy", score)
    print(f"Test accuracy: {score:.4f}")

# ============================================================
# AUTOLOG FOR DIFFERENT FRAMEWORKS
# ============================================================

# TensorFlow/Keras
# mlflow.tensorflow.autolog()

# PyTorch (via Lightning)
# mlflow.pytorch.autolog()

# XGBoost
# mlflow.xgboost.autolog()

# LightGBM
# mlflow.lightgbm.autolog()

# All frameworks at once (convenient but less control)
# mlflow.autolog()
```

### Hyperparameter Tuning with Tracking

```python
"""
Tracking hyperparameter search experiments.
This is one of the most common real-world use cases.
"""

import mlflow
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import ParameterGrid, cross_val_score
from sklearn.datasets import load_wine
from sklearn.model_selection import train_test_split
import numpy as np

wine = load_wine()
X_train, X_test, y_train, y_test = train_test_split(
    wine.data, wine.target, test_size=0.2, random_state=42
)

# Define hyperparameter grid
param_grid = {
    "n_estimators": [50, 100, 200],
    "max_depth": [3, 5, 10, None],
    "min_samples_split": [2, 5, 10],
}

mlflow.set_experiment("wine-hyperparam-search")

# ============================================================
# NESTED RUNS — Parent run contains all child experiments
# ============================================================
with mlflow.start_run(run_name="grid_search_parent"):
    
    # Log search configuration
    mlflow.log_param("search_type", "grid_search")
    mlflow.log_param("total_combinations", len(list(ParameterGrid(param_grid))))
    mlflow.log_param("cv_folds", 5)
    
    best_score = 0
    best_params = {}
    
    for i, params in enumerate(ParameterGrid(param_grid)):
        # Each combination is a NESTED (child) run
        with mlflow.start_run(
            run_name=f"combo_{i:03d}", 
            nested=True  # This makes it a child run
        ):
            # Log parameters
            mlflow.log_params(params)
            
            # Train and evaluate
            model = RandomForestClassifier(random_state=42, **params)
            cv_scores = cross_val_score(model, X_train, y_train, cv=5)
            
            # Log metrics
            mlflow.log_metrics({
                "cv_mean_accuracy": cv_scores.mean(),
                "cv_std_accuracy": cv_scores.std(),
                "cv_min_accuracy": cv_scores.min(),
                "cv_max_accuracy": cv_scores.max(),
            })
            
            # Track best
            if cv_scores.mean() > best_score:
                best_score = cv_scores.mean()
                best_params = params
    
    # Log best results in parent run
    mlflow.log_metric("best_cv_accuracy", best_score)
    mlflow.log_params({f"best_{k}": v for k, v in best_params.items()})
    
    # Train final model with best params
    final_model = RandomForestClassifier(random_state=42, **best_params)
    final_model.fit(X_train, y_train)
    test_accuracy = final_model.score(X_test, y_test)
    mlflow.log_metric("test_accuracy", test_accuracy)
    
    # Save best model
    mlflow.sklearn.log_model(final_model, "best_model")
    
    print(f"Best params: {best_params}")
    print(f"Best CV accuracy: {best_score:.4f}")
    print(f"Test accuracy: {test_accuracy:.4f}")
```

### Loading and Comparing Runs Programmatically

```python
"""
Querying and comparing experiments programmatically.
Essential for automated pipelines and model selection.
"""

import mlflow
from mlflow.tracking import MlflowClient
import pandas as pd

# Initialize the MLflow client
client = MlflowClient()

# ============================================================
# QUERYING EXPERIMENTS
# ============================================================

# List all experiments
experiments = client.search_experiments()
for exp in experiments:
    print(f"Experiment: {exp.name} (ID: {exp.experiment_id})")

# ============================================================
# SEARCHING RUNS WITH FILTERS
# ============================================================

# Search for best runs using MLflow's filter syntax
runs = mlflow.search_runs(
    experiment_names=["wine-quality-classification"],
    filter_string="metrics.accuracy > 0.9",  # Only good models
    order_by=["metrics.accuracy DESC"],       # Best first
    max_results=10,
)

print("\nTop 10 runs with accuracy > 0.9:")
print(runs[["run_id", "params.model_type", "metrics.accuracy"]].head())

# ============================================================
# COMPARING SPECIFIC RUNS
# ============================================================

# Get specific run details
run_id = "your_run_id_here"  # Replace with actual run ID
# run = client.get_run(run_id)
# print(f"Parameters: {run.data.params}")
# print(f"Metrics: {run.data.metrics}")

# ============================================================
# LOADING A MODEL FROM A RUN
# ============================================================

# Load model from a specific run
# model = mlflow.sklearn.load_model(f"runs:/{run_id}/model")

# Load model from the model registry (better for production)
# model = mlflow.sklearn.load_model("models:/wine-classifier/Production")

# ============================================================
# AUTOMATED MODEL COMPARISON
# ============================================================

def compare_runs(experiment_name: str, metric: str = "accuracy") -> pd.DataFrame:
    """
    Compare all runs in an experiment.
    Returns a DataFrame sorted by the specified metric.
    """
    runs = mlflow.search_runs(
        experiment_names=[experiment_name],
        order_by=[f"metrics.{metric} DESC"],
    )
    
    # Select relevant columns
    param_cols = [c for c in runs.columns if c.startswith("params.")]
    metric_cols = [c for c in runs.columns if c.startswith("metrics.")]
    
    comparison = runs[["run_id", "start_time"] + param_cols + metric_cols]
    return comparison


# Example usage
# comparison_df = compare_runs("wine-quality-classification")
# print(comparison_df.head())
```

### MLflow Projects (Reproducible Runs)

```python
"""
MLflow Projects — Package experiments for reproducibility.
An MLproject file defines how to run your experiment.
"""

# ============================================================
# FILE: MLproject (YAML format, no extension)
# ============================================================
mlproject_content = """
name: wine-classification

# Environment specification
conda_env: conda.yaml
# OR: docker_env: Dockerfile
# OR: python_env: python_env.yaml

entry_points:
  # Main training entry point
  train:
    parameters:
      n_estimators: {type: int, default: 100}
      max_depth: {type: int, default: 5}
      learning_rate: {type: float, default: 0.01}
      data_path: {type: str, default: "data/wine.csv"}
    command: "python train.py --n_estimators {n_estimators} --max_depth {max_depth} --learning_rate {learning_rate} --data_path {data_path}"
  
  # Evaluation entry point
  evaluate:
    parameters:
      model_uri: {type: str}
      test_data: {type: str}
    command: "python evaluate.py --model_uri {model_uri} --test_data {test_data}"
"""

# ============================================================
# FILE: conda.yaml
# ============================================================
conda_yaml = """
name: wine-classification
channels:
  - defaults
  - conda-forge
dependencies:
  - python=3.9
  - scikit-learn=1.3.0
  - pandas=2.0.0
  - numpy=1.24.0
  - pip:
    - mlflow>=2.0
"""

# ============================================================
# Running MLflow Projects (from terminal)
# ============================================================
run_commands = """
# Run locally with default params
mlflow run . 

# Run with custom parameters
mlflow run . -P n_estimators=200 -P max_depth=10

# Run from a Git repository (reproducibility!)
mlflow run https://github.com/user/ml-project -P n_estimators=200

# Run a specific entry point
mlflow run . -e evaluate -P model_uri="runs:/abc123/model"
"""

print("MLproject file:")
print(mlproject_content)
print("\nRun commands:")
print(run_commands)
```

---

## Weights & Biases (W&B)

### What is W&B

Weights & Biases is a commercial (with free tier) experiment tracking platform known for its:
- Beautiful visualizations
- Real-time collaboration
- Deep learning focus (but works for any ML)
- Integrated hyperparameter sweeps
- Report generation

### Basic W&B Usage

```python
"""
Weights & Biases (wandb) — Complete Example
Excellent for deep learning experiments and team collaboration.
"""

# Installation: pip install wandb
# First time: wandb login (enter API key from wandb.ai/settings)

import wandb
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_wine
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, classification_report

# ============================================================
# BASIC W&B TRACKING
# ============================================================

# Initialize a run
wandb.init(
    project="wine-classification",     # Project name (groups experiments)
    name="random-forest-baseline",      # Run name (this specific experiment)
    config={                            # Hyperparameters
        "model_type": "RandomForest",
        "n_estimators": 100,
        "max_depth": 10,
        "test_size": 0.2,
        "random_state": 42,
    },
    tags=["baseline", "sklearn"],       # Tags for filtering
    notes="Initial baseline with default RF params",  # Description
)

# Access config (useful when running sweeps - params come from W&B)
config = wandb.config

# Load and split data
wine = load_wine()
X_train, X_test, y_train, y_test = train_test_split(
    wine.data, wine.target,
    test_size=config.test_size,
    random_state=config.random_state,
)

# Train model
model = RandomForestClassifier(
    n_estimators=config.n_estimators,
    max_depth=config.max_depth,
    random_state=config.random_state,
)
model.fit(X_train, y_train)

# Evaluate
y_pred = model.predict(X_test)
accuracy = accuracy_score(y_test, y_pred)

# Log metrics
wandb.log({
    "accuracy": accuracy,
    "n_train_samples": len(X_train),
    "n_test_samples": len(X_test),
})

# Log feature importances as a bar chart
feature_data = list(zip(wine.feature_names, model.feature_importances_))
feature_table = wandb.Table(
    data=feature_data, 
    columns=["Feature", "Importance"]
)
wandb.log({"feature_importance": wandb.plot.bar(
    feature_table, "Feature", "Importance", 
    title="Feature Importances"
)})

# Log confusion matrix
wandb.log({
    "confusion_matrix": wandb.plot.confusion_matrix(
        y_true=y_test,
        preds=y_pred,
        class_names=wine.target_names.tolist(),
    )
})

# Save model artifact
artifact = wandb.Artifact("trained-model", type="model")
# artifact.add_file("model.pkl")  # Would need to save model first
wandb.log_artifact(artifact)

# Summary metrics (shown in the runs table)
wandb.summary["best_accuracy"] = accuracy
wandb.summary["model_size_params"] = sum(
    tree.tree_.node_count for tree in model.estimators_
)

# Finish the run
wandb.finish()

print(f"Accuracy: {accuracy:.4f}")
print("View results at: https://wandb.ai/your-username/wine-classification")
```

### W&B Sweeps (Hyperparameter Optimization)

```python
"""
W&B Sweeps — Automated hyperparameter search with visualization.
"""

import wandb
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.datasets import load_wine
from sklearn.model_selection import train_test_split, cross_val_score

# ============================================================
# DEFINE SWEEP CONFIGURATION
# ============================================================
sweep_config = {
    "method": "bayes",  # bayesian optimization (smarter than grid/random)
    # Options: "grid", "random", "bayes"
    
    "metric": {
        "name": "cv_accuracy",  # Metric to optimize
        "goal": "maximize",     # "maximize" or "minimize"
    },
    
    "parameters": {
        "model_type": {
            "values": ["random_forest", "gradient_boosting"]
        },
        "n_estimators": {
            "min": 50,
            "max": 500,
        },
        "max_depth": {
            "values": [3, 5, 7, 10, 15, None]
        },
        "learning_rate": {
            "distribution": "log_uniform_values",  # Log scale
            "min": 0.001,
            "max": 0.3,
        },
        "min_samples_split": {
            "values": [2, 5, 10, 20]
        },
    },
    
    # Early termination (stop bad runs early)
    "early_terminate": {
        "type": "hyperband",
        "min_iter": 3,        # Minimum number of epochs/folds
        "eta": 3,             # Aggressiveness of early stopping
    },
}


def train_sweep():
    """Training function called by the sweep agent."""
    # Initialize run (config comes from sweep)
    wandb.init()
    config = wandb.config
    
    # Load data
    wine = load_wine()
    X_train, X_test, y_train, y_test = train_test_split(
        wine.data, wine.target, test_size=0.2, random_state=42
    )
    
    # Select model based on config
    if config.model_type == "random_forest":
        model = RandomForestClassifier(
            n_estimators=config.n_estimators,
            max_depth=config.max_depth,
            min_samples_split=config.min_samples_split,
            random_state=42,
        )
    else:
        model = GradientBoostingClassifier(
            n_estimators=config.n_estimators,
            max_depth=config.max_depth if config.max_depth else 3,
            learning_rate=config.learning_rate,
            min_samples_split=config.min_samples_split,
            random_state=42,
        )
    
    # Cross-validation with step logging
    cv_scores = cross_val_score(model, X_train, y_train, cv=5)
    for fold, score in enumerate(cv_scores):
        wandb.log({"fold_accuracy": score, "fold": fold})
    
    # Log summary metrics
    wandb.log({
        "cv_accuracy": cv_scores.mean(),
        "cv_std": cv_scores.std(),
    })
    
    # Final evaluation on test set
    model.fit(X_train, y_train)
    test_accuracy = model.score(X_test, y_test)
    wandb.log({"test_accuracy": test_accuracy})


# ============================================================
# RUN THE SWEEP
# ============================================================

# Create sweep (returns sweep ID)
# sweep_id = wandb.sweep(sweep_config, project="wine-sweep")

# Run sweep agent (this runs train_sweep multiple times)
# wandb.agent(sweep_id, function=train_sweep, count=50)  # 50 trials

print("Sweep config defined. To run:")
print("1. sweep_id = wandb.sweep(sweep_config, project='wine-sweep')")
print("2. wandb.agent(sweep_id, function=train_sweep, count=50)")
```

### W&B Alerts and Collaboration

```python
"""
W&B Advanced Features — Alerts, Reports, and Collaboration
"""

import wandb

# ============================================================
# ALERTS — Get notified about important events
# ============================================================

# Initialize run
wandb.init(project="production-monitoring")

# Send alert when accuracy drops below threshold
accuracy = 0.85  # Simulated
threshold = 0.90

if accuracy < threshold:
    wandb.alert(
        title="Model Performance Degradation",
        text=f"Accuracy dropped to {accuracy:.3f} (threshold: {threshold})",
        level=wandb.AlertLevel.WARN,  # INFO, WARN, ERROR
    )

# ============================================================
# TABLES — Log structured data for analysis
# ============================================================

# Log predictions table (great for error analysis)
columns = ["input", "prediction", "ground_truth", "correct"]
data = [
    ["sample_1", "class_A", "class_A", True],
    ["sample_2", "class_B", "class_A", False],  # Error!
    ["sample_3", "class_C", "class_C", True],
]

predictions_table = wandb.Table(columns=columns, data=data)
wandb.log({"predictions": predictions_table})

# ============================================================
# ARTIFACTS — Version datasets and models
# ============================================================

# Log a dataset version
# dataset_artifact = wandb.Artifact("wine-dataset", type="dataset")
# dataset_artifact.add_dir("data/processed/")
# wandb.log_artifact(dataset_artifact)

# Use a specific dataset version in training
# artifact = wandb.use_artifact("wine-dataset:v2")
# data_dir = artifact.download()

wandb.finish()
```

---

## TensorBoard

### What is TensorBoard

TensorBoard is TensorFlow's built-in visualization toolkit. While originally for TensorFlow, it now works with PyTorch and other frameworks. Best for:
- Training curve visualization
- Model graph inspection
- Embedding visualization
- Image/audio/text logging
- Profiling performance

### TensorBoard with PyTorch

```python
"""
TensorBoard with PyTorch — Real-time training visualization.
"""

# Installation: pip install tensorboard torch

import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.tensorboard import SummaryWriter
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
import numpy as np

# ============================================================
# SETUP TENSORBOARD WRITER
# ============================================================

# Writer creates log directory structure
# Run: tensorboard --logdir=runs/ --port=6006
writer = SummaryWriter("runs/experiment_001")

# ============================================================
# SIMPLE NEURAL NETWORK EXAMPLE
# ============================================================

# Create synthetic dataset
X, y = make_classification(
    n_samples=1000, n_features=20, n_classes=3,
    n_informative=15, random_state=42
)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

# Convert to tensors
X_train_t = torch.FloatTensor(X_train)
y_train_t = torch.LongTensor(y_train)
X_test_t = torch.FloatTensor(X_test)
y_test_t = torch.LongTensor(y_test)


class SimpleNet(nn.Module):
    """Simple feedforward network for classification."""
    def __init__(self, input_dim, hidden_dim, output_dim):
        super().__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(hidden_dim, hidden_dim // 2),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(hidden_dim // 2, output_dim),
        )
    
    def forward(self, x):
        return self.network(x)


# Initialize
model = SimpleNet(input_dim=20, hidden_dim=64, output_dim=3)
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001)

# ============================================================
# LOG MODEL ARCHITECTURE
# ============================================================
# Visualize model graph in TensorBoard
dummy_input = torch.randn(1, 20)
writer.add_graph(model, dummy_input)

# ============================================================
# TRAINING LOOP WITH TENSORBOARD LOGGING
# ============================================================
n_epochs = 100

for epoch in range(n_epochs):
    model.train()
    
    # Forward pass
    outputs = model(X_train_t)
    loss = criterion(outputs, y_train_t)
    
    # Backward pass
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    # --- LOG TRAINING METRICS ---
    writer.add_scalar("Loss/train", loss.item(), epoch)
    
    # Calculate training accuracy
    _, predicted = torch.max(outputs, 1)
    train_acc = (predicted == y_train_t).float().mean().item()
    writer.add_scalar("Accuracy/train", train_acc, epoch)
    
    # --- LOG VALIDATION METRICS ---
    if epoch % 5 == 0:
        model.eval()
        with torch.no_grad():
            val_outputs = model(X_test_t)
            val_loss = criterion(val_outputs, y_test_t).item()
            _, val_predicted = torch.max(val_outputs, 1)
            val_acc = (val_predicted == y_test_t).float().mean().item()
        
        writer.add_scalar("Loss/validation", val_loss, epoch)
        writer.add_scalar("Accuracy/validation", val_acc, epoch)
        
        # Log learning rate
        writer.add_scalar("LR", optimizer.param_groups[0]["lr"], epoch)
    
    # --- LOG HISTOGRAMS (weight distributions) ---
    if epoch % 10 == 0:
        for name, param in model.named_parameters():
            writer.add_histogram(f"Parameters/{name}", param, epoch)
            if param.grad is not None:
                writer.add_histogram(f"Gradients/{name}", param.grad, epoch)

# ============================================================
# LOG HYPERPARAMETERS AND FINAL METRICS
# ============================================================
writer.add_hparams(
    hparam_dict={
        "lr": 0.001,
        "hidden_dim": 64,
        "dropout": 0.3,
        "optimizer": "Adam",
    },
    metric_dict={
        "hparam/accuracy": val_acc,
        "hparam/loss": val_loss,
    },
)

# Close writer
writer.close()

print(f"Final train accuracy: {train_acc:.4f}")
print(f"Final val accuracy: {val_acc:.4f}")
print("\nRun: tensorboard --logdir=runs/ --port=6006")
```

### TensorBoard with TensorFlow/Keras

```python
"""
TensorBoard with Keras — Simplest integration.
"""

import tensorflow as tf
from tensorflow import keras
import datetime

# Create a simple model
model = keras.Sequential([
    keras.layers.Dense(128, activation='relu', input_shape=(20,)),
    keras.layers.Dropout(0.3),
    keras.layers.Dense(64, activation='relu'),
    keras.layers.Dropout(0.2),
    keras.layers.Dense(3, activation='softmax'),
])

model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy'],
)

# ============================================================
# TENSORBOARD CALLBACK — One line to add tracking!
# ============================================================
log_dir = "logs/fit/" + datetime.datetime.now().strftime("%Y%m%d-%H%M%S")

tensorboard_callback = keras.callbacks.TensorBoard(
    log_dir=log_dir,
    histogram_freq=1,       # Log weight histograms every epoch
    write_graph=True,       # Visualize model architecture
    write_images=True,      # Log weight images
    update_freq="epoch",    # Log metrics at end of each epoch
    profile_batch="500,520" # Profile compute performance (batch 500-520)
)

# Train (TensorBoard captures everything automatically)
# model.fit(
#     X_train, y_train,
#     epochs=50,
#     validation_data=(X_test, y_test),
#     callbacks=[tensorboard_callback],
# )

print(f"Logs saved to: {log_dir}")
print("Run: tensorboard --logdir=logs/fit/ --port=6006")
```

---

## Experiment Organization Best Practices

### Naming Conventions

```python
"""
Experiment Organization — Naming and Structure Best Practices
"""

# ============================================================
# NAMING CONVENTION SYSTEM
# ============================================================

# Experiment Names (group related work)
experiment_names = {
    # Pattern: {project}-{task}-{version}
    "fraud-detection-v1": "First iteration of fraud model",
    "fraud-detection-v2": "Added transaction features",
    "fraud-detection-v2-ablation": "Feature importance study",
}

# Run Names (individual experiments)
run_naming = {
    # Pattern: {model}_{key_param}_{date}
    "rf_100trees_20240115": "Random Forest, 100 trees, Jan 15",
    "xgb_lr001_depth5": "XGBoost, lr=0.01, depth=5",
    "bert_base_ep3_bs32": "BERT base, 3 epochs, batch 32",
}

# Tags (for filtering and grouping)
tag_system = {
    # Lifecycle tags
    "stage": ["experiment", "candidate", "champion", "retired"],
    
    # Quality tags
    "quality": ["baseline", "improvement", "regression"],
    
    # Feature tags
    "features": ["all_features", "top_10", "no_text", "embeddings_v2"],
    
    # Team tags
    "team": ["alice", "bob", "nlp-team", "cv-team"],
}

print("Naming Convention Examples:")
print("=" * 50)
for name, desc in experiment_names.items():
    print(f"  {name:40s} → {desc}")

print("\nTag Categories:")
for category, values in tag_system.items():
    print(f"  {category}: {values}")
```

### Experiment Documentation Template

```python
"""
Every experiment should have documentation.
Use this template as a starting point.
"""

experiment_doc_template = """
# Experiment: {experiment_name}

## Objective
What are you trying to achieve? What's the hypothesis?

## Context
- Previous best: {model} with {metric} = {value}
- This experiment tests: {hypothesis}

## Setup
- Dataset: {dataset_name} v{version} ({n_samples} samples)
- Features: {feature_set}
- Target: {target_variable}
- Split: {train_pct}% train / {val_pct}% val / {test_pct}% test

## Key Decisions
1. Why this model/approach?
2. What alternatives were considered?
3. Any assumptions or limitations?

## Results
| Run | Model | Key Param | Accuracy | F1 | Notes |
|-----|-------|-----------|----------|----|----- |
| 001 | RF    | trees=100 | 0.94     | 0.93 | Baseline |
| 002 | XGB   | lr=0.01   | 0.96     | 0.95 | Better! |

## Conclusions
- What worked?
- What didn't?
- Next steps?

## Artifacts
- Best model: runs:/{run_id}/model
- Analysis notebook: notebooks/experiment_analysis.ipynb
"""

print(experiment_doc_template)
```

### Organizing Large-Scale Experiments

```python
"""
How to organize experiments at scale (100s of runs).
"""

import mlflow
from datetime import datetime

# ============================================================
# HIERARCHICAL EXPERIMENT ORGANIZATION
# ============================================================

"""
Organization Structure:

project-name/
├── experiments/
│   ├── baseline/              # Initial models
│   │   ├── run_001 (logistic regression)
│   │   └── run_002 (random forest)
│   │
│   ├── feature-engineering/   # Feature experiments
│   │   ├── run_010 (add text features)
│   │   ├── run_011 (add time features)
│   │   └── run_012 (PCA reduction)
│   │
│   ├── model-selection/       # Architecture search
│   │   ├── run_020 (XGBoost)
│   │   ├── run_021 (LightGBM)
│   │   └── run_022 (Neural Network)
│   │
│   ├── hyperparameter-tuning/ # Fine-tuning best model
│   │   ├── sweep_001 (100 runs)
│   │   └── sweep_002 (50 runs, refined)
│   │
│   └── production-candidates/ # Final candidates
│       ├── candidate_v1
│       └── candidate_v2 (champion)
"""

# In MLflow, use experiment names to create this hierarchy
experiment_hierarchy = [
    "fraud-detection/baseline",
    "fraud-detection/feature-engineering",
    "fraud-detection/model-selection",
    "fraud-detection/hyperparameter-tuning",
    "fraud-detection/production-candidates",
]

# Pro Tip: Use a consistent tagging strategy
def start_tracked_run(
    experiment_phase: str,
    run_name: str,
    model_type: str,
    **kwargs
):
    """
    Standardized run creation with consistent metadata.
    Use this wrapper across your team for consistency.
    """
    mlflow.set_experiment(f"fraud-detection/{experiment_phase}")
    
    with mlflow.start_run(run_name=run_name) as run:
        # Standard metadata
        mlflow.set_tags({
            "phase": experiment_phase,
            "model_type": model_type,
            "author": "team-member",  # Or get from environment
            "timestamp": datetime.now().isoformat(),
        })
        
        # Log all additional params
        if kwargs:
            mlflow.log_params(kwargs)
        
        return run


# Usage example
# run = start_tracked_run(
#     experiment_phase="model-selection",
#     run_name="xgboost_v1",
#     model_type="XGBoost",
#     n_estimators=200,
#     learning_rate=0.01,
# )
```

---

## Comparing Tools

### Feature Comparison

| Feature | MLflow | W&B | TensorBoard | Neptune | Comet |
|---------|--------|-----|-------------|---------|-------|
| **Cost** | Free (OSS) | Free tier + paid | Free (OSS) | Free tier + paid | Free tier + paid |
| **Hosting** | Self-hosted or managed | Cloud (managed) | Local | Cloud (managed) | Cloud (managed) |
| **Auto-logging** | ✅ (sklearn, PyTorch, TF) | ✅ (extensive) | ✅ (TF/Keras) | ✅ | ✅ |
| **Model Registry** | ✅ (built-in) | ✅ (Registry) | ❌ | ✅ | ✅ |
| **Hyperparameter Sweeps** | ❌ (use Optuna) | ✅ (built-in) | ❌ | ✅ | ✅ |
| **Collaboration** | Basic | Excellent | Limited | Good | Good |
| **Visualization** | Good | Excellent | Good (training) | Good | Good |
| **Data Versioning** | ❌ (use DVC) | ✅ (Artifacts) | ❌ | ❌ | ❌ |
| **Reports** | ❌ | ✅ (W&B Reports) | ❌ | ❌ | ✅ |
| **Scalability** | Depends on setup | High (managed) | Medium | High | High |
| **Privacy/On-prem** | ✅ (self-hosted) | ✅ (enterprise) | ✅ | ❌ | ❌ |
| **Learning Curve** | Low | Low | Low | Low | Low |

### When to Use What

```
Decision Tree:
─────────────

Q: Do you need self-hosted / on-premise?
├── YES → MLflow (free, self-hosted, full control)
└── NO → Continue...

Q: Working mostly with deep learning?
├── YES → Weights & Biases (best DL visualization)
└── NO → Continue...

Q: Budget is zero and need model registry?
├── YES → MLflow (free, includes registry)
└── NO → Continue...

Q: Need built-in hyperparameter optimization?
├── YES → W&B Sweeps or Neptune
└── NO → Continue...

Q: Already using TensorFlow/Keras?
├── YES → TensorBoard (built-in, zero setup) + MLflow for registry
└── NO → Continue...

Default recommendation: MLflow (free, comprehensive, industry standard)
Upgrade to: W&B when collaboration/visualization matters more
```

### Integration Example (Using Multiple Tools Together)

```python
"""
Best Practice: Combine tools for their strengths.
MLflow for registry + W&B for visualization, or
MLflow for tracking + TensorBoard for training curves.
"""

import mlflow

# Option 1: MLflow + TensorBoard
# Use TensorBoard for real-time training viz
# Use MLflow for experiment comparison and model registry

# In your training loop:
# from torch.utils.tensorboard import SummaryWriter
# writer = SummaryWriter()  # Real-time viz
# mlflow.log_metric(...)    # Persistent tracking

# Option 2: MLflow + Optuna (for hyperparameter search)
import optuna

def objective(trial):
    """Optuna objective with MLflow tracking."""
    
    # Suggest hyperparameters
    n_estimators = trial.suggest_int("n_estimators", 50, 500)
    max_depth = trial.suggest_int("max_depth", 3, 15)
    
    with mlflow.start_run(nested=True):
        mlflow.log_params({
            "n_estimators": n_estimators,
            "max_depth": max_depth,
        })
        
        # Train and evaluate...
        accuracy = 0.95  # Placeholder
        
        mlflow.log_metric("accuracy", accuracy)
        return accuracy

# study = optuna.create_study(direction="maximize")
# with mlflow.start_run(run_name="optuna_search"):
#     study.optimize(objective, n_trials=100)

print("Combination strategies:")
print("1. MLflow (tracking + registry) + TensorBoard (training curves)")
print("2. MLflow (tracking) + Optuna (hyperparameter search)")
print("3. W&B (tracking + viz) + DVC (data versioning)")
```

---

## Common Mistakes

### 1. Not Tracking from Day One

❌ **Wrong:** "I'll add tracking once the model is good enough"
✅ **Right:** Track everything from the first experiment

> The first "throwaway" experiment often becomes the baseline everyone compares against. You'll need those parameters later.

### 2. Logging Too Little OR Too Much

❌ **Too little:** Only logging accuracy
❌ **Too much:** Logging every single variable in your script

✅ **Right:** Log:
- All hyperparameters (anything you might change)
- Key metrics (accuracy, loss, latency, business metrics)
- Data characteristics (size, distribution, version)
- Environment info (library versions, hardware)

### 3. Not Using Consistent Naming

❌ **Wrong:** `run1`, `test_2`, `final_model_actually_final`
✅ **Right:** `rf_100trees_allfeatures_20240115`

### 4. Forgetting to Log the Negative Results

❌ **Wrong:** Only keeping runs that worked
✅ **Right:** Keep ALL runs — negative results tell you what NOT to do

> "I already tried that — it didn't work because X" saves hours of repeated work.

### 5. Not Linking Code to Experiments

❌ **Wrong:** Logging metrics but not the exact code that produced them
✅ **Right:** Use `mlflow.log_param("git_commit", get_git_hash())` or enable git tracking

```python
# Always log the git hash!
import subprocess

def get_git_hash():
    """Get current git commit hash for reproducibility."""
    try:
        return subprocess.check_output(
            ["git", "rev-parse", "HEAD"]
        ).decode().strip()
    except Exception:
        return "unknown"

# mlflow.log_param("git_commit", get_git_hash())
```

### 6. Running Experiments Without a Hypothesis

❌ **Wrong:** "Let me try 100 random combinations and pick the best"
✅ **Right:** "I hypothesize that adding text features will improve F1 by 5% because..."

### 7. Not Comparing Against a Proper Baseline

❌ **Wrong:** "My model has 92% accuracy!"
✅ **Right:** Tracked baselines:
- Random/majority class baseline: 33%
- Simple heuristic: 75%  
- Previous model: 88%
- **My model: 92% (+4% over production model)**

---

## Interview Questions

### Technical Questions

**Q1: What is experiment tracking and why is it important?**
> **A:** Experiment tracking is the systematic recording of all ML experiment metadata (parameters, metrics, artifacts, code versions, data versions). It's important because: (1) Reproducibility — you can recreate any result, (2) Comparison — objectively compare approaches, (3) Collaboration — team members see all experiments, (4) Auditability — trace any production model back to its training, (5) Efficiency — avoid repeating failed experiments.

**Q2: Compare MLflow and Weights & Biases. When would you choose each?**
> **A:** MLflow: Choose when you need self-hosted/on-premise, want a free solution with model registry, or need minimal vendor lock-in. It's the industry standard for general MLOps. W&B: Choose when deep learning visualization matters, team collaboration is priority, you want built-in hyperparameter sweeps, or you need generated reports for stakeholders. It's better for DL-heavy teams willing to pay for managed service.

**Q3: How do you ensure reproducibility of ML experiments?**
> **A:** Track these 6 things: (1) Code version (git hash), (2) Data version (hash or DVC), (3) Environment (Docker image or conda lock), (4) Configuration (all hyperparameters externalized), (5) Random seeds (set and log them), (6) Hardware (GPU type, memory). Use MLflow Projects or Docker to package everything together.

**Q4: How would you organize experiments for a team of 5 data scientists?**
> **A:** (1) Shared tracking server (MLflow or W&B), (2) Naming convention: `{project}/{phase}/{model}_{timestamp}`, (3) Required tags: author, phase (baseline/experiment/candidate), status, (4) Experiment phases: baseline → feature engineering → model selection → tuning → production candidate, (5) Regular experiment review meetings to share findings, (6) Documentation template for each experiment's hypothesis and conclusions.

**Q5: What metrics should you track beyond accuracy?**
> **A:** Business metrics (revenue impact, user engagement), fairness metrics (demographic parity, equalized odds), operational metrics (inference latency p50/p95/p99, throughput, model size, memory usage), data metrics (input distribution stats, missing value rates), and reliability metrics (prediction confidence distribution, abstention rate).

### Scenario Questions

**Q6: You inherited a project with 500 untracked experiments. How do you organize it?**
> **A:** (1) Set up tracking server immediately for all new experiments. (2) Identify the current production model and create a "champion" baseline. (3) Don't retroactively track all 500 — focus on the 5-10 most recent/important. (4) Establish naming conventions and required tags going forward. (5) Create a "legacy" experiment group for old results that matter. (6) Train team on new process.

**Q7: Your team member's "best model" can't be reproduced. What went wrong?**
> **A:** Common causes: (1) Random seed not set/logged, (2) Different library versions (especially sklearn minor versions change defaults), (3) Data preprocessing not saved (different feature engineering), (4) Training on different data split, (5) Notebook cells run out of order, (6) Hardware differences (GPU vs CPU, floating point differences). Prevention: Use experiment tracking, Docker environments, and externalized configs.

---

## Quick Reference

### MLflow Cheat Sheet

```python
# === SETUP ===
import mlflow
mlflow.set_tracking_uri("http://server:5000")
mlflow.set_experiment("my-experiment")

# === BASIC TRACKING ===
with mlflow.start_run(run_name="my_run"):
    mlflow.log_param("key", "value")           # Single param
    mlflow.log_params({"k1": "v1", "k2": 2})   # Multiple params
    mlflow.log_metric("accuracy", 0.95)         # Single metric
    mlflow.log_metric("loss", 0.5, step=10)     # Metric at step
    mlflow.log_metrics({"f1": 0.93, "auc": 0.97})  # Multiple
    mlflow.log_artifact("file.png")             # Single file
    mlflow.log_artifacts("output_dir/")         # Directory
    mlflow.set_tag("status", "production")      # Tag
    mlflow.sklearn.log_model(model, "model")    # Log model

# === AUTOLOG ===
mlflow.sklearn.autolog()  # or mlflow.autolog() for all

# === NESTED RUNS ===
with mlflow.start_run(run_name="parent"):
    with mlflow.start_run(run_name="child", nested=True):
        pass

# === QUERY RUNS ===
runs = mlflow.search_runs(
    experiment_names=["my-exp"],
    filter_string="metrics.accuracy > 0.9",
    order_by=["metrics.accuracy DESC"],
)

# === LOAD MODEL ===
model = mlflow.sklearn.load_model(f"runs:/{run_id}/model")
model = mlflow.sklearn.load_model("models:/name/Production")
```

### W&B Cheat Sheet

```python
# === SETUP ===
import wandb
wandb.init(project="my-project", name="run-name", config={...})

# === TRACKING ===
wandb.log({"loss": 0.5, "accuracy": 0.95})      # Log metrics
wandb.log({"image": wandb.Image(img_array)})     # Log image
wandb.log({"table": wandb.Table(...)})           # Log table
wandb.summary["best_acc"] = 0.97                 # Summary metric

# === ARTIFACTS ===
artifact = wandb.Artifact("name", type="model")
artifact.add_file("model.pkl")
wandb.log_artifact(artifact)

# === SWEEPS ===
sweep_id = wandb.sweep(sweep_config, project="p")
wandb.agent(sweep_id, function=train_fn, count=50)

# === ALERTS ===
wandb.alert(title="Alert!", text="Details", level=wandb.AlertLevel.WARN)

# === FINISH ===
wandb.finish()
```

### Comparison Summary Table

| Criteria | Best Tool |
|----------|-----------|
| Free & self-hosted | MLflow |
| Best visualization | Weights & Biases |
| TensorFlow/Keras native | TensorBoard |
| Built-in HP optimization | W&B Sweeps |
| Model registry | MLflow |
| Enterprise/compliance | MLflow (on-prem) or Neptune |
| Quick prototyping | MLflow (zero setup) |
| Team collaboration | W&B |
| Deep learning research | W&B + TensorBoard |

---

*Previous: [01-MLOps-Fundamentals.md](01-MLOps-Fundamentals.md)*
*Next: [03-Model-Versioning-and-Registry.md](03-Model-Versioning-and-Registry.md) — DVC, Model Registry, and versioning everything*
