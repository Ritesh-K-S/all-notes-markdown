# Chapter 06: Model Monitoring

## Table of Contents
- [What is Model Monitoring?](#what-is-model-monitoring)
- [Why Model Monitoring Matters](#why-model-monitoring-matters)
- [How Model Monitoring Works](#how-model-monitoring-works)
- [Data Drift Detection](#data-drift-detection)
- [Model Drift (Concept Drift)](#model-drift-concept-drift)
- [Performance Monitoring](#performance-monitoring)
- [Monitoring Infrastructure](#monitoring-infrastructure)
- [Alerting and Incident Response](#alerting-and-incident-response)
- [Monitoring Tools Ecosystem](#monitoring-tools-ecosystem)
- [Building a Monitoring Pipeline](#building-a-monitoring-pipeline)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Model Monitoring?

**Simple Explanation:** Imagine you open a restaurant and the chef makes amazing food on day one. But over time, the supplier starts sending lower-quality ingredients, customer tastes change, and the chef starts cutting corners. Without someone tasting the food regularly (monitoring), you wouldn't notice until customers stop coming.

Model monitoring is the same idea — your ML model might be perfect at deployment, but the real world changes. The data shifts, relationships between features evolve, and your model quietly starts making bad predictions. Monitoring catches this before it becomes a disaster.

**Formal Definition:** Model monitoring is the continuous process of tracking the statistical properties of input data, model predictions, and real-world outcomes in production to detect degradation, anomalies, and failures — enabling timely intervention and retraining.

### The Silent Failure Problem

```
Traditional Software:                    ML Models:
┌─────────────────────┐                  ┌─────────────────────┐
│  Bug → Crash → Alert│                  │  Drift → Silent     │
│  Obvious failure     │                  │  degradation         │
│  Easy to detect      │                  │  Hard to detect      │
│  Deterministic       │                  │  Probabilistic       │
└─────────────────────┘                  └─────────────────────┘

Traditional code fails LOUDLY.           ML models fail SILENTLY.
You get an error message.                You get wrong predictions.
No data scientist needed to debug.       Need statistical tests to detect.
```

> **Key Insight:** An ML model can return HTTP 200 (success) with every request while giving completely wrong predictions. Traditional application monitoring won't catch this — you need ML-specific monitoring.

---

## Why Model Monitoring Matters

### Real-World Horror Stories

| Company | What Happened | Impact | Root Cause |
|---------|--------------|--------|------------|
| Zillow | Home price prediction model went wrong | $500M+ loss, laid off 25% of staff | Concept drift — housing market changed |
| Amazon | Hiring model became biased | PR disaster, scrapped system | Training data bias amplified over time |
| Healthcare AI | Diagnostic model accuracy dropped | Misdiagnoses for months | New equipment changed image characteristics (data drift) |
| Finance | Fraud model stopped catching fraud | Millions in losses | Fraudsters adapted their patterns |

### What Can Go Wrong (The 5 Failure Modes)

```
                    Model Deployed
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │  Data    │   │  Model   │   │  System  │
   │  Issues  │   │  Issues  │   │  Issues  │
   └────┬─────┘   └────┬─────┘   └────┬─────┘
        │              │              │
   ┌────▼────┐   ┌────▼─────┐  ┌────▼─────┐
   │Data Drift│  │Concept   │  │Latency   │
   │Feature  │  │Drift     │  │Spike     │
   │Quality  │  │Model Stale│  │Memory    │
   │Schema   │  │Bias      │  │Leak      │
   │Changes  │  │Amplify   │  │GPU Fail  │
   └─────────┘  └──────────┘  └──────────┘
```

### Why You Can't Just Retrain Periodically

Periodic retraining (e.g., every month) is like going to the doctor only on your birthday regardless of symptoms:
- What if drift happens 2 days after retraining? You wait 28 days with a bad model.
- What if the model is still fine after 6 months? You waste compute and risk regression.
- **Monitoring tells you WHEN to retrain, not the calendar.**

---

## How Model Monitoring Works

### The Monitoring Stack

```
┌──────────────────────────────────────────────────────────────┐
│                    Production Environment                      │
│                                                                │
│  ┌──────────┐    ┌──────────┐    ┌───────────┐               │
│  │  Input   │───▶│  Model   │───▶│  Output   │               │
│  │  Data    │    │  Server  │    │(Predictions│               │
│  └────┬─────┘    └────┬─────┘    └─────┬─────┘               │
│       │              │              │                         │
│       ▼              ▼              ▼                         │
│  ┌─────────────────────────────────────────────┐             │
│  │           Logging / Telemetry Layer          │             │
│  │  • Input features    • Prediction values    │             │
│  │  • Latency          • Confidence scores     │             │
│  │  • Error counts     • Feature distributions │             │
│  └──────────────────────┬──────────────────────┘             │
└──────────────────────────┼──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                  Monitoring Platform                          │
│                                                               │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐      │
│  │  Data Drift │  │  Model Perf  │  │  System       │      │
│  │  Detection  │  │  Tracking    │  │  Metrics      │      │
│  └──────┬──────┘  └──────┬───────┘  └───────┬───────┘      │
│         │               │                │                   │
│         ▼               ▼                ▼                   │
│  ┌─────────────────────────────────────────────────┐        │
│  │              Alerting Engine                     │        │
│  │  • Slack/PagerDuty/Email notifications          │        │
│  │  • Auto-rollback triggers                       │        │
│  │  • Retraining pipeline triggers                 │        │
│  └─────────────────────────────────────────────────┘        │
│                                                               │
│  ┌─────────────────────────────────────────────────┐        │
│  │              Dashboards (Grafana)                │        │
│  └─────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────┘
```

### The Three Pillars of ML Monitoring

| Pillar | What You Monitor | When You Have Ground Truth | When You Don't |
|--------|-----------------|--------------------------|----------------|
| **Data Quality** | Input feature distributions, missing values, schema | Always possible | Always possible |
| **Model Performance** | Accuracy, F1, AUC, latency | After labels arrive (hours/days/weeks) | Use proxy metrics |
| **System Health** | CPU, GPU, memory, latency, errors | Always possible | Always possible |

---

## Data Drift Detection

### What is Data Drift?

**Analogy:** You trained a model to recognize dogs using photos taken in daylight. Now users are sending photos taken at night. The model hasn't changed, but the input data has shifted — that's data drift.

**Types of Data Drift:**

```
┌─────────────────────────────────────────────────────────┐
│                    Types of Drift                         │
├─────────────────┬──────────────────┬────────────────────┤
│   Covariate     │    Prior         │    Concept         │
│   Shift         │    Probability   │    Drift           │
│                 │    Shift         │                    │
│ P(X) changes    │ P(Y) changes    │ P(Y|X) changes    │
│ Input dist.     │ Label dist.     │ Relationship       │
│ shifts          │ shifts          │ changes            │
│                 │                 │                    │
│ Example:        │ Example:        │ Example:           │
│ Age distribution│ More fraud cases│ What "spam" looks  │
│ of users changes│ during holidays │ like changes       │
└─────────────────┴──────────────────┴────────────────────┘
```

### Statistical Tests for Drift Detection

#### 1. Kolmogorov-Smirnov Test (Numerical Features)

```python
# drift_detection.py - Statistical drift detection
import numpy as np
from scipy import stats
import pandas as pd
from typing import Dict, Tuple

class DriftDetector:
    """
    Detects data drift between reference (training) and production data.
    
    Uses multiple statistical tests because no single test catches all types of drift.
    """
    
    def __init__(self, reference_data: pd.DataFrame, significance_level: float = 0.05):
        """
        Args:
            reference_data: Training data (or a representative sample)
            significance_level: p-value threshold for drift detection
                              0.05 = 5% chance of false positive
        """
        self.reference = reference_data
        self.alpha = significance_level
        self.reference_stats = self._compute_stats(reference_data)
    
    def _compute_stats(self, df: pd.DataFrame) -> Dict:
        """Pre-compute reference statistics for efficiency"""
        stats_dict = {}
        for col in df.select_dtypes(include=[np.number]).columns:
            stats_dict[col] = {
                'mean': df[col].mean(),
                'std': df[col].std(),
                'min': df[col].min(),
                'max': df[col].max(),
                'quantiles': df[col].quantile([0.25, 0.5, 0.75]).to_dict()
            }
        return stats_dict
    
    def ks_test(self, production_data: pd.DataFrame) -> Dict[str, Dict]:
        """
        Kolmogorov-Smirnov test for numerical features.
        
        Tests if two samples come from the same distribution.
        - H0: Distributions are the same
        - H1: Distributions are different
        
        Good for: Detecting shifts in any part of the distribution
        Limitation: Less sensitive for large samples (everything becomes "significant")
        """
        results = {}
        
        for col in self.reference.select_dtypes(include=[np.number]).columns:
            if col not in production_data.columns:
                results[col] = {'error': 'Column missing in production data'}
                continue
            
            # Run KS test
            statistic, p_value = stats.ks_2samp(
                self.reference[col].dropna(),
                production_data[col].dropna()
            )
            
            drift_detected = p_value < self.alpha
            
            results[col] = {
                'test': 'Kolmogorov-Smirnov',
                'statistic': round(statistic, 4),
                'p_value': round(p_value, 4),
                'drift_detected': drift_detected,
                'severity': self._classify_severity(statistic, 'ks')
            }
        
        return results
    
    def psi_test(self, production_data: pd.DataFrame, n_bins: int = 10) -> Dict[str, Dict]:
        """
        Population Stability Index (PSI) - Industry standard for drift detection.
        
        Formula: PSI = Σ (P_i - Q_i) × ln(P_i / Q_i)
        
        Where:
        - P_i = proportion of production data in bin i
        - Q_i = proportion of reference data in bin i
        
        Interpretation:
        - PSI < 0.1:  No significant drift
        - 0.1 ≤ PSI < 0.2:  Moderate drift (investigate)
        - PSI ≥ 0.2:  Significant drift (action needed)
        """
        results = {}
        
        for col in self.reference.select_dtypes(include=[np.number]).columns:
            if col not in production_data.columns:
                continue
            
            ref_col = self.reference[col].dropna()
            prod_col = production_data[col].dropna()
            
            # Create bins from reference data
            bins = np.linspace(ref_col.min(), ref_col.max(), n_bins + 1)
            bins[0] = -np.inf
            bins[-1] = np.inf
            
            # Calculate proportions in each bin
            ref_counts = np.histogram(ref_col, bins=bins)[0]
            prod_counts = np.histogram(prod_col, bins=bins)[0]
            
            # Add small epsilon to avoid division by zero
            eps = 1e-4
            ref_proportions = (ref_counts + eps) / (len(ref_col) + eps * n_bins)
            prod_proportions = (prod_counts + eps) / (len(prod_col) + eps * n_bins)
            
            # Calculate PSI
            psi = np.sum(
                (prod_proportions - ref_proportions) * 
                np.log(prod_proportions / ref_proportions)
            )
            
            results[col] = {
                'test': 'PSI',
                'psi_value': round(psi, 4),
                'drift_detected': psi >= 0.1,
                'severity': 'none' if psi < 0.1 else 'moderate' if psi < 0.2 else 'severe'
            }
        
        return results
    
    def chi_squared_test(self, production_data: pd.DataFrame) -> Dict[str, Dict]:
        """
        Chi-Squared test for categorical features.
        
        Tests if the distribution of categories has changed.
        """
        results = {}
        
        for col in self.reference.select_dtypes(include=['object', 'category']).columns:
            if col not in production_data.columns:
                continue
            
            # Get category counts
            ref_counts = self.reference[col].value_counts()
            prod_counts = production_data[col].value_counts()
            
            # Align categories
            all_categories = set(ref_counts.index) | set(prod_counts.index)
            ref_aligned = np.array([ref_counts.get(c, 0) for c in all_categories])
            prod_aligned = np.array([prod_counts.get(c, 0) for c in all_categories])
            
            # Normalize to same total
            ref_expected = ref_aligned / ref_aligned.sum() * prod_aligned.sum()
            
            # Chi-squared test
            # Only include categories with expected count > 5 (test assumption)
            mask = ref_expected > 5
            if mask.sum() < 2:
                results[col] = {'error': 'Not enough categories for test'}
                continue
            
            statistic, p_value = stats.chisquare(
                prod_aligned[mask], 
                ref_expected[mask]
            )
            
            # Check for new categories (not in reference)
            new_categories = set(prod_counts.index) - set(ref_counts.index)
            
            results[col] = {
                'test': 'Chi-Squared',
                'statistic': round(statistic, 4),
                'p_value': round(p_value, 4),
                'drift_detected': p_value < self.alpha,
                'new_categories': list(new_categories) if new_categories else None
            }
        
        return results
    
    def _classify_severity(self, statistic: float, test_type: str) -> str:
        """Classify drift severity based on test statistic"""
        if test_type == 'ks':
            if statistic < 0.05:
                return 'none'
            elif statistic < 0.1:
                return 'low'
            elif statistic < 0.2:
                return 'moderate'
            else:
                return 'severe'
        return 'unknown'
    
    def full_report(self, production_data: pd.DataFrame) -> Dict:
        """Run all drift tests and generate comprehensive report"""
        report = {
            'ks_tests': self.ks_test(production_data),
            'psi_tests': self.psi_test(production_data),
            'chi_squared_tests': self.chi_squared_test(production_data),
            'summary': {}
        }
        
        # Count drifted features
        drifted = []
        for test_name, test_results in report.items():
            if test_name == 'summary':
                continue
            for feature, result in test_results.items():
                if isinstance(result, dict) and result.get('drift_detected'):
                    drifted.append(feature)
        
        drifted_unique = list(set(drifted))
        total_features = len(production_data.columns)
        
        report['summary'] = {
            'total_features': total_features,
            'drifted_features': len(drifted_unique),
            'drift_percentage': round(len(drifted_unique) / total_features * 100, 1),
            'drifted_feature_names': drifted_unique,
            'action_needed': len(drifted_unique) > total_features * 0.3  # >30% drifted
        }
        
        return report


# ===== USAGE EXAMPLE =====

# Simulate reference (training) data
np.random.seed(42)
reference_df = pd.DataFrame({
    'age': np.random.normal(35, 10, 10000),
    'income': np.random.lognormal(10, 1, 10000),
    'score': np.random.uniform(300, 850, 10000),
    'category': np.random.choice(['A', 'B', 'C'], 10000, p=[0.5, 0.3, 0.2])
})

# Simulate production data WITH drift
production_df = pd.DataFrame({
    'age': np.random.normal(40, 12, 5000),       # Mean shifted 35→40, std 10→12
    'income': np.random.lognormal(10.5, 1, 5000), # Slight shift
    'score': np.random.uniform(300, 850, 5000),    # No drift
    'category': np.random.choice(['A', 'B', 'C', 'D'], 5000, p=[0.3, 0.3, 0.2, 0.2])  # New category!
})

# Detect drift
detector = DriftDetector(reference_df)
report = detector.full_report(production_df)

# Print results
print("=" * 60)
print("DRIFT DETECTION REPORT")
print("=" * 60)
for feature, result in report['ks_tests'].items():
    status = "🚨 DRIFT" if result.get('drift_detected') else "✅ OK"
    print(f"  {feature}: {status} (KS stat={result.get('statistic', 'N/A')}, p={result.get('p_value', 'N/A')})")

print(f"\nSummary: {report['summary']['drifted_features']}/{report['summary']['total_features']} features drifted")
print(f"Action needed: {report['summary']['action_needed']}")
```

#### 2. Jensen-Shannon Divergence (JSD)

```python
# jsd_drift.py - Jensen-Shannon Divergence for drift detection
import numpy as np
from scipy.spatial.distance import jensenshannon
from scipy.stats import entropy

def compute_jsd(reference: np.ndarray, production: np.ndarray, n_bins: int = 50) -> float:
    """
    Jensen-Shannon Divergence: symmetric measure of distribution difference.
    
    Why JSD over KL Divergence?
    - KL divergence is asymmetric: KL(P||Q) ≠ KL(Q||P)
    - KL divergence is undefined when Q(x) = 0 but P(x) > 0
    - JSD fixes both issues: it's symmetric and always defined
    
    JSD(P || Q) = 0.5 * KL(P || M) + 0.5 * KL(Q || M)
    where M = 0.5 * (P + Q)
    
    Range: [0, 1] where 0 = identical, 1 = completely different
    """
    # Create common bins
    combined = np.concatenate([reference, production])
    bins = np.linspace(combined.min(), combined.max(), n_bins + 1)
    
    # Compute histograms (probability distributions)
    ref_hist, _ = np.histogram(reference, bins=bins, density=True)
    prod_hist, _ = np.histogram(production, bins=bins, density=True)
    
    # Normalize to probability distributions
    ref_hist = ref_hist / ref_hist.sum()
    prod_hist = prod_hist / prod_hist.sum()
    
    # Compute JSD using scipy (returns sqrt(JSD) so we square it)
    jsd = jensenshannon(ref_hist, prod_hist) ** 2
    
    return jsd

# Example
ref_data = np.random.normal(0, 1, 10000)
prod_data_no_drift = np.random.normal(0, 1, 5000)
prod_data_drift = np.random.normal(0.5, 1.5, 5000)

print(f"No drift JSD:   {compute_jsd(ref_data, prod_data_no_drift):.4f}")  # ~0.001
print(f"With drift JSD: {compute_jsd(ref_data, prod_data_drift):.4f}")    # ~0.05+
```

### Feature-Level Monitoring Dashboard

```python
# feature_monitor.py - Monitor individual feature distributions
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import json
import logging

logger = logging.getLogger(__name__)

class FeatureMonitor:
    """
    Monitors individual feature statistics over time.
    
    Tracks: mean, std, min, max, null rate, unique count, distribution
    Alerts when values move outside expected ranges.
    """
    
    def __init__(self, reference_data: pd.DataFrame, window_size: int = 1000):
        self.reference = reference_data
        self.window_size = window_size
        self.baselines = self._compute_baselines()
        self.history = []
    
    def _compute_baselines(self) -> dict:
        """Compute baseline statistics from reference data"""
        baselines = {}
        for col in self.reference.columns:
            if self.reference[col].dtype in ['float64', 'int64', 'float32', 'int32']:
                baselines[col] = {
                    'mean': self.reference[col].mean(),
                    'std': self.reference[col].std(),
                    'min': self.reference[col].min(),
                    'max': self.reference[col].max(),
                    'null_rate': self.reference[col].isnull().mean(),
                    'range_low': self.reference[col].quantile(0.01),   # 1st percentile
                    'range_high': self.reference[col].quantile(0.99),  # 99th percentile
                }
            else:
                value_counts = self.reference[col].value_counts(normalize=True)
                baselines[col] = {
                    'categories': value_counts.to_dict(),
                    'null_rate': self.reference[col].isnull().mean(),
                    'n_unique': self.reference[col].nunique()
                }
        return baselines
    
    def check_window(self, window_data: pd.DataFrame) -> dict:
        """
        Check a window of production data against baselines.
        Returns alerts for any anomalies found.
        """
        alerts = []
        stats = {}
        
        for col in window_data.columns:
            if col not in self.baselines:
                alerts.append({
                    'feature': col,
                    'type': 'new_feature',
                    'message': f'New feature "{col}" not in reference data'
                })
                continue
            
            baseline = self.baselines[col]
            
            if window_data[col].dtype in ['float64', 'int64', 'float32', 'int32']:
                current_stats = {
                    'mean': window_data[col].mean(),
                    'std': window_data[col].std(),
                    'null_rate': window_data[col].isnull().mean(),
                    'out_of_range_pct': (
                        (window_data[col] < baseline['range_low']) | 
                        (window_data[col] > baseline['range_high'])
                    ).mean() * 100
                }
                stats[col] = current_stats
                
                # Alert: Mean shifted by more than 2 standard deviations
                mean_shift = abs(current_stats['mean'] - baseline['mean']) / (baseline['std'] + 1e-10)
                if mean_shift > 2:
                    alerts.append({
                        'feature': col,
                        'type': 'mean_shift',
                        'severity': 'high' if mean_shift > 3 else 'medium',
                        'message': f'Mean shifted by {mean_shift:.1f} std devs '
                                   f'({baseline["mean"]:.2f} → {current_stats["mean"]:.2f})'
                    })
                
                # Alert: Null rate increased significantly
                if current_stats['null_rate'] > baseline['null_rate'] + 0.05:
                    alerts.append({
                        'feature': col,
                        'type': 'null_rate_increase',
                        'severity': 'high',
                        'message': f'Null rate increased: '
                                   f'{baseline["null_rate"]:.1%} → {current_stats["null_rate"]:.1%}'
                    })
                
                # Alert: Out-of-range values
                if current_stats['out_of_range_pct'] > 5:
                    alerts.append({
                        'feature': col,
                        'type': 'out_of_range',
                        'severity': 'medium',
                        'message': f'{current_stats["out_of_range_pct"]:.1f}% of values outside '
                                   f'expected range [{baseline["range_low"]:.2f}, {baseline["range_high"]:.2f}]'
                    })
        
        # Check for missing features
        for col in self.baselines:
            if col not in window_data.columns:
                alerts.append({
                    'feature': col,
                    'type': 'missing_feature',
                    'severity': 'critical',
                    'message': f'Feature "{col}" missing from production data'
                })
        
        result = {
            'timestamp': datetime.now().isoformat(),
            'window_size': len(window_data),
            'stats': stats,
            'alerts': alerts,
            'n_alerts': len(alerts),
            'has_critical': any(a.get('severity') == 'critical' for a in alerts)
        }
        
        self.history.append(result)
        return result
```

---

## Model Drift (Concept Drift)

### What is Concept Drift?

**Analogy:** Imagine a model that predicts which emails are spam. In 2020, spam was mostly about "Nigerian princes." By 2024, spam looks like fake delivery notifications and AI-generated phishing. The model's inputs (emails) look different, and what "spam" MEANS has changed — that's concept drift.

**Mathematical Definition:**

The joint probability distribution $P(X, Y)$ changes over time:

$$P_t(X, Y) \neq P_{t+1}(X, Y)$$

Since $P(X, Y) = P(Y|X) \cdot P(X)$, drift can happen in:
- $P(X)$ alone → **Covariate shift** (data drift)
- $P(Y|X)$ changes → **Concept drift** (the real danger)
- $P(Y)$ changes → **Prior probability shift**

### Types of Concept Drift

```
Sudden Drift:              Gradual Drift:           Incremental Drift:
Performance                Performance               Performance
│ ─────┐                  │ ────╲                   │ ──╲
│      │                  │      ╲╲                 │    ╲──╲
│      └─────             │        ╲╲───            │       ╲──╲──
│                         │          ╲              │            ╲──
└──────────── Time        └──────────── Time        └──────────── Time

Example: Law change        Example: Fashion          Example: Climate
(credit rules change       trends gradually           slowly changes
overnight)                 evolving)                  crop yields)

Recurring Drift:           
Performance               
│ ──╲  ╱──╲  ╱──         
│    ╲╱    ╲╱             
│                         
└──────────── Time        
                          
Example: Holiday           
shopping patterns          
```

### Detecting Concept Drift

```python
# concept_drift.py - Detecting when model's learned relationship breaks
import numpy as np
from collections import deque

class ADWIN:
    """
    ADaptive WINdowing (ADWIN) - Online drift detection algorithm.
    
    How it works:
    1. Maintains a window of recent observations
    2. Checks if any split of the window shows statistically different means
    3. If yes → drift detected, drops old data
    
    Advantage: No need to define window size, adapts automatically.
    """
    
    def __init__(self, delta: float = 0.002):
        """
        Args:
            delta: Confidence parameter. Lower = fewer false positives but slower detection.
                   0.002 is a good default for production.
        """
        self.delta = delta
        self.window = []
        self.total = 0.0
        self.variance = 0.0
        self.n = 0
    
    def add_element(self, value: float) -> bool:
        """
        Add new observation and check for drift.
        
        Returns True if drift is detected.
        """
        self.window.append(value)
        self.n += 1
        self.total += value
        
        if self.n < 30:  # Need minimum samples
            return False
        
        return self._check_drift()
    
    def _check_drift(self) -> bool:
        """Check if there's a statistically significant change in the window"""
        n = len(self.window)
        
        for i in range(10, n - 10):  # Check different split points
            # Split window into two sub-windows
            w1 = self.window[:i]
            w2 = self.window[i:]
            
            n1, n2 = len(w1), len(w2)
            mean1 = np.mean(w1)
            mean2 = np.mean(w2)
            
            # Hoeffding bound
            m = 1.0 / (1.0/n1 + 1.0/n2)
            epsilon = np.sqrt((1.0 / (2.0 * m)) * np.log(4.0 / self.delta))
            
            if abs(mean1 - mean2) >= epsilon:
                # Drift detected! Drop old data
                self.window = self.window[i:]
                self.n = len(self.window)
                self.total = sum(self.window)
                return True
        
        return False


class DDM:
    """
    Drift Detection Method (DDM) - Monitors error rate for drift.
    
    Idea: Track the running error rate. If it increases beyond a threshold
    (based on its own standard deviation), we have drift.
    
    p_i + s_i > p_min + 3 * s_min  →  DRIFT (3-sigma rule)
    p_i + s_i > p_min + 2 * s_min  →  WARNING
    
    Where:
    - p_i = error rate at time i
    - s_i = standard deviation of error rate
    """
    
    def __init__(self, min_samples: int = 30, warning_level: float = 2.0, drift_level: float = 3.0):
        self.min_samples = min_samples
        self.warning_level = warning_level
        self.drift_level = drift_level
        self.reset()
    
    def reset(self):
        """Reset the detector"""
        self.n = 0
        self.p = 0.0      # Error rate
        self.s = 0.0      # Standard deviation of error rate
        self.p_min = float('inf')
        self.s_min = float('inf')
        self.in_warning = False
    
    def add_prediction(self, is_error: bool) -> str:
        """
        Add a prediction result and check for drift.
        
        Args:
            is_error: True if the prediction was wrong
            
        Returns:
            'normal', 'warning', or 'drift'
        """
        self.n += 1
        
        # Update error rate (online running mean)
        self.p = self.p + (float(is_error) - self.p) / self.n
        
        # Standard deviation of binomial proportion
        self.s = np.sqrt(self.p * (1 - self.p) / self.n)
        
        if self.n < self.min_samples:
            return 'normal'
        
        # Track minimum p + s (best performance seen)
        if self.p + self.s < self.p_min + self.s_min:
            self.p_min = self.p
            self.s_min = self.s
        
        # Check for drift
        if self.p + self.s > self.p_min + self.drift_level * self.s_min:
            status = 'drift'
            self.reset()  # Reset after drift detection
        elif self.p + self.s > self.p_min + self.warning_level * self.s_min:
            status = 'warning'
            self.in_warning = True
        else:
            status = 'normal'
            self.in_warning = False
        
        return status


# ===== USAGE EXAMPLE =====

# Simulate a model that starts accurate then degrades
np.random.seed(42)

# Phase 1: Normal operation (5% error rate)
normal_errors = np.random.binomial(1, 0.05, 500)

# Phase 2: Drift occurs (error rate jumps to 15%)
drift_errors = np.random.binomial(1, 0.15, 500)

all_errors = np.concatenate([normal_errors, drift_errors])

# Test DDM
ddm = DDM()
for i, error in enumerate(all_errors):
    status = ddm.add_prediction(bool(error))
    if status != 'normal':
        print(f"Step {i}: {status.upper()} detected! (error rate: {ddm.p:.3f})")
        if status == 'drift':
            print(f"  → Drift confirmed at step {i}. Trigger retraining!")
            break

# Expected output: Drift detected around step 550-600
```

### Performance Monitoring Without Ground Truth

```python
# proxy_monitoring.py - When you can't wait for labels
import numpy as np
from typing import List, Dict

class ProxyMonitor:
    """
    Monitor model health without ground truth labels.
    
    Problem: Labels often arrive days/weeks after prediction.
    Solution: Use proxy metrics that indicate something is wrong.
    """
    
    def __init__(self, reference_predictions: np.ndarray):
        """
        Args:
            reference_predictions: Predictions on validation set (known-good behavior)
        """
        self.ref_preds = reference_predictions
        self.ref_confidence_mean = reference_predictions.max(axis=1).mean()
        self.ref_confidence_std = reference_predictions.max(axis=1).std()
        self.ref_class_distribution = np.bincount(
            reference_predictions.argmax(axis=1)
        ) / len(reference_predictions)
    
    def check_prediction_confidence(self, predictions: np.ndarray) -> Dict:
        """
        Monitor confidence scores.
        
        If confidence drops, model is less sure → something changed.
        """
        confidences = predictions.max(axis=1)
        current_mean = confidences.mean()
        
        # How many standard deviations from reference?
        z_score = (current_mean - self.ref_confidence_mean) / (self.ref_confidence_std + 1e-10)
        
        return {
            'metric': 'confidence',
            'reference_mean': round(self.ref_confidence_mean, 4),
            'current_mean': round(current_mean, 4),
            'z_score': round(z_score, 2),
            'alert': abs(z_score) > 2,
            'message': f'Confidence {"dropped" if z_score < -2 else "spiked" if z_score > 2 else "normal"}'
        }
    
    def check_prediction_distribution(self, predictions: np.ndarray) -> Dict:
        """
        Monitor predicted class distribution.
        
        If model suddenly predicts 90% class A (was 50%), something is wrong.
        """
        pred_classes = predictions.argmax(axis=1)
        current_distribution = np.bincount(
            pred_classes, minlength=len(self.ref_class_distribution)
        ) / len(predictions)
        
        # KL divergence between distributions
        from scipy.stats import entropy
        eps = 1e-10
        kl_div = entropy(
            current_distribution + eps,
            self.ref_class_distribution + eps
        )
        
        return {
            'metric': 'prediction_distribution',
            'reference': self.ref_class_distribution.tolist(),
            'current': current_distribution.tolist(),
            'kl_divergence': round(kl_div, 4),
            'alert': kl_div > 0.1,
            'message': f'Prediction distribution {"shifted" if kl_div > 0.1 else "stable"}'
        }
    
    def check_uncertainty(self, predictions: np.ndarray, threshold: float = 0.3) -> Dict:
        """
        Monitor high-uncertainty predictions.
        
        If more predictions fall below the confidence threshold,
        the model is struggling with the new data.
        """
        confidences = predictions.max(axis=1)
        uncertain_pct = (confidences < threshold).mean()
        
        ref_confidences = self.ref_preds.max(axis=1)
        ref_uncertain_pct = (ref_confidences < threshold).mean()
        
        return {
            'metric': 'uncertainty',
            'reference_uncertain_pct': round(ref_uncertain_pct, 4),
            'current_uncertain_pct': round(uncertain_pct, 4),
            'increase_factor': round(uncertain_pct / (ref_uncertain_pct + 1e-10), 2),
            'alert': uncertain_pct > ref_uncertain_pct * 2,
            'message': f'Uncertain predictions: {uncertain_pct:.1%} (was {ref_uncertain_pct:.1%})'
        }
```

---

## Performance Monitoring

### Tracking Real Model Performance

```python
# performance_tracker.py - Track model metrics over time
import pandas as pd
import numpy as np
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, 
    f1_score, roc_auc_score, mean_absolute_error,
    mean_squared_error
)
from datetime import datetime
from typing import Dict, List, Optional
import json
import logging

logger = logging.getLogger(__name__)

class ModelPerformanceTracker:
    """
    Tracks model performance metrics over time.
    
    Key challenge: Labels arrive with a delay.
    - Fraud: Minutes to hours (customer reports)
    - Ad clicks: Seconds (user clicks or doesn't)
    - Medical diagnosis: Days to weeks (follow-up tests)
    - Loan default: Months to years!
    
    Strategy: Track metrics at the cadence labels arrive.
    """
    
    def __init__(
        self, 
        model_name: str,
        task_type: str = 'classification',  # 'classification' or 'regression'
        baseline_metrics: Optional[Dict] = None
    ):
        self.model_name = model_name
        self.task_type = task_type
        self.baseline = baseline_metrics or {}
        self.metrics_history = []
    
    def compute_metrics(
        self, 
        y_true: np.ndarray, 
        y_pred: np.ndarray, 
        y_prob: Optional[np.ndarray] = None,
        segment: str = 'all'
    ) -> Dict:
        """
        Compute comprehensive metrics for a batch of labeled predictions.
        
        Args:
            y_true: Actual labels (ground truth)
            y_pred: Predicted labels
            y_prob: Predicted probabilities (for AUC)
            segment: Data segment (e.g., 'all', 'us_users', 'mobile')
        """
        metrics = {
            'model_name': self.model_name,
            'timestamp': datetime.now().isoformat(),
            'segment': segment,
            'n_samples': len(y_true),
        }
        
        if self.task_type == 'classification':
            metrics.update({
                'accuracy': round(accuracy_score(y_true, y_pred), 4),
                'precision': round(precision_score(y_true, y_pred, average='weighted', zero_division=0), 4),
                'recall': round(recall_score(y_true, y_pred, average='weighted', zero_division=0), 4),
                'f1': round(f1_score(y_true, y_pred, average='weighted', zero_division=0), 4),
            })
            
            if y_prob is not None:
                try:
                    metrics['auc_roc'] = round(roc_auc_score(
                        y_true, y_prob, multi_class='ovr', average='weighted'
                    ), 4)
                except ValueError:
                    metrics['auc_roc'] = None
            
            # Class-level metrics (detect bias toward specific classes)
            unique_classes = np.unique(y_true)
            class_metrics = {}
            for cls in unique_classes:
                mask = y_true == cls
                class_metrics[str(cls)] = {
                    'count': int(mask.sum()),
                    'precision': round(precision_score(y_true == cls, y_pred == cls, zero_division=0), 4),
                    'recall': round(recall_score(y_true == cls, y_pred == cls, zero_division=0), 4)
                }
            metrics['per_class'] = class_metrics
            
        elif self.task_type == 'regression':
            metrics.update({
                'mae': round(mean_absolute_error(y_true, y_pred), 4),
                'rmse': round(np.sqrt(mean_squared_error(y_true, y_pred)), 4),
                'mape': round(np.mean(np.abs((y_true - y_pred) / (y_true + 1e-10))) * 100, 2),
                'r2': round(1 - np.sum((y_true - y_pred)**2) / np.sum((y_true - np.mean(y_true))**2), 4),
            })
        
        # Compare to baseline
        if self.baseline:
            degradation = {}
            for metric_name, baseline_value in self.baseline.items():
                if metric_name in metrics and isinstance(metrics[metric_name], (int, float)):
                    current = metrics[metric_name]
                    if current is not None:
                        # For error metrics (MAE, RMSE), increase is bad
                        # For quality metrics (accuracy, F1), decrease is bad
                        error_metrics = {'mae', 'rmse', 'mape'}
                        if metric_name in error_metrics:
                            change = (current - baseline_value) / (baseline_value + 1e-10) * 100
                            is_degraded = change > 10  # >10% increase in error
                        else:
                            change = (baseline_value - current) / (baseline_value + 1e-10) * 100
                            is_degraded = change > 5   # >5% decrease in quality
                        
                        degradation[metric_name] = {
                            'baseline': baseline_value,
                            'current': current,
                            'change_pct': round(change, 2),
                            'degraded': is_degraded
                        }
            
            metrics['vs_baseline'] = degradation
            metrics['any_degradation'] = any(d['degraded'] for d in degradation.values())
        
        self.metrics_history.append(metrics)
        return metrics
    
    def get_trend(self, metric_name: str, n_windows: int = 10) -> Dict:
        """
        Analyze trend of a metric over recent windows.
        
        Returns: trend direction, slope, and alert if degrading.
        """
        if len(self.metrics_history) < 3:
            return {'trend': 'insufficient_data'}
        
        recent = self.metrics_history[-n_windows:]
        values = [m.get(metric_name) for m in recent if m.get(metric_name) is not None]
        
        if len(values) < 3:
            return {'trend': 'insufficient_data'}
        
        # Simple linear regression for trend
        x = np.arange(len(values))
        slope, intercept = np.polyfit(x, values, 1)
        
        return {
            'metric': metric_name,
            'values': values,
            'slope': round(slope, 6),
            'trend': 'improving' if slope > 0.001 else 'degrading' if slope < -0.001 else 'stable',
            'alert': slope < -0.005  # Significant downward trend
        }


# ===== USAGE =====

# Initialize tracker with baseline metrics (from validation set)
tracker = ModelPerformanceTracker(
    model_name="fraud_detector_v2",
    task_type="classification",
    baseline_metrics={
        'accuracy': 0.95,
        'precision': 0.92,
        'recall': 0.88,
        'f1': 0.90,
        'auc_roc': 0.97
    }
)

# Simulate weekly metric computation (as labels arrive)
np.random.seed(42)
for week in range(10):
    # Simulate gradually degrading predictions
    error_rate = 0.05 + week * 0.01  # Error rate increases each week
    n_samples = 1000
    
    y_true = np.random.binomial(1, 0.1, n_samples)  # 10% positive rate
    y_pred = y_true.copy()
    
    # Introduce errors
    error_mask = np.random.random(n_samples) < error_rate
    y_pred[error_mask] = 1 - y_pred[error_mask]
    
    y_prob = np.column_stack([1 - y_pred * 0.9, y_pred * 0.9])
    
    metrics = tracker.compute_metrics(y_true, y_pred, y_prob, segment=f'week_{week}')
    
    if metrics.get('any_degradation'):
        logger.warning(f"Week {week}: Performance degradation detected!")
        for name, info in metrics['vs_baseline'].items():
            if info['degraded']:
                logger.warning(f"  {name}: {info['baseline']} → {info['current']} ({info['change_pct']:+.1f}%)")

# Check trend
trend = tracker.get_trend('f1')
print(f"\nF1 Trend: {trend['trend']} (slope: {trend['slope']})")
```

---

## Monitoring Infrastructure

### Prometheus + Grafana Setup

```python
# metrics_exporter.py - Export ML metrics to Prometheus
from prometheus_client import (
    Counter, Histogram, Gauge, Summary, 
    CollectorRegistry, generate_latest
)
from fastapi import FastAPI, Response
import time
import numpy as np

# Create custom registry (avoids conflicts with default metrics)
registry = CollectorRegistry()

# ===== Define Metrics =====

# Counter: Monotonically increasing value (total predictions, errors)
PREDICTION_COUNT = Counter(
    'model_predictions_total',
    'Total number of predictions made',
    ['model_name', 'model_version', 'prediction_class'],
    registry=registry
)

PREDICTION_ERRORS = Counter(
    'model_prediction_errors_total',
    'Total number of prediction errors',
    ['model_name', 'error_type'],
    registry=registry
)

# Histogram: Distribution of values (latency, confidence)
PREDICTION_LATENCY = Histogram(
    'model_prediction_latency_seconds',
    'Time spent processing prediction request',
    ['model_name'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
    registry=registry
)

PREDICTION_CONFIDENCE = Histogram(
    'model_prediction_confidence',
    'Confidence score of predictions',
    ['model_name'],
    buckets=[0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 0.95, 0.99],
    registry=registry
)

# Gauge: Value that can go up and down (queue size, model version)
MODEL_INFO = Gauge(
    'model_info',
    'Model metadata',
    ['model_name', 'model_version', 'framework'],
    registry=registry
)

FEATURE_DRIFT_SCORE = Gauge(
    'feature_drift_score',
    'Current drift score for each feature',
    ['model_name', 'feature_name'],
    registry=registry
)

INPUT_QUEUE_SIZE = Gauge(
    'model_input_queue_size',
    'Number of requests waiting in queue',
    ['model_name'],
    registry=registry
)

# ===== Instrumented Prediction Endpoint =====

app = FastAPI()

@app.post("/predict")
async def predict(request: dict):
    model_name = "fraud_detector"
    
    start_time = time.time()
    
    try:
        # Make prediction
        features = np.array(request['features']).reshape(1, -1)
        prediction = model.predict(features)[0]
        confidence = model.predict_proba(features)[0].max()
        
        # Record metrics
        latency = time.time() - start_time
        PREDICTION_LATENCY.labels(model_name=model_name).observe(latency)
        PREDICTION_CONFIDENCE.labels(model_name=model_name).observe(confidence)
        PREDICTION_COUNT.labels(
            model_name=model_name,
            model_version="2.1",
            prediction_class=str(prediction)
        ).inc()
        
        return {
            "prediction": int(prediction),
            "confidence": float(confidence),
            "latency_ms": round(latency * 1000, 2)
        }
        
    except Exception as e:
        PREDICTION_ERRORS.labels(
            model_name=model_name,
            error_type=type(e).__name__
        ).inc()
        raise

# Prometheus scrape endpoint
@app.get("/metrics")
async def metrics():
    """Endpoint that Prometheus scrapes for metrics"""
    return Response(
        content=generate_latest(registry),
        media_type="text/plain"
    )
```

### Prometheus Configuration

```yaml
# prometheus.yml - Scrape configuration
global:
  scrape_interval: 15s      # How often to collect metrics
  evaluation_interval: 15s   # How often to evaluate alert rules

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'model-server'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['model-server:8000']
    scrape_interval: 10s

  - job_name: 'model-server-replicas'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: ml-model
        action: keep
```

### Alert Rules

```yaml
# alert_rules.yml - Prometheus alerting rules for ML models
groups:
  - name: ml_model_alerts
    rules:
      # High latency alert
      - alert: ModelHighLatency
        expr: histogram_quantile(0.95, rate(model_prediction_latency_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Model {{ $labels.model_name }} p95 latency > 500ms"
          description: "p95 latency is {{ $value }}s for the last 5 minutes"

      # High error rate
      - alert: ModelHighErrorRate
        expr: |
          rate(model_prediction_errors_total[5m]) 
          / rate(model_predictions_total[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Model {{ $labels.model_name }} error rate > 1%"

      # Low confidence alert (model is uncertain)
      - alert: ModelLowConfidence
        expr: |
          histogram_quantile(0.5, rate(model_prediction_confidence_bucket[1h])) < 0.5
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Model {{ $labels.model_name }} median confidence dropped below 50%"

      # Feature drift detected
      - alert: FeatureDriftDetected
        expr: feature_drift_score > 0.2
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Feature drift detected in {{ $labels.feature_name }}"
          description: "PSI score: {{ $value }}"

      # Prediction distribution shift
      - alert: PredictionDistributionShift
        expr: |
          abs(
            rate(model_predictions_total{prediction_class="1"}[1h]) 
            / rate(model_predictions_total[1h])
            - 0.1
          ) > 0.05
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Prediction distribution shifted for {{ $labels.model_name }}"

      # Model server down
      - alert: ModelServerDown
        expr: up{job="model-server"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Model server is down!"
```

### Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "ML Model Monitoring",
    "panels": [
      {
        "title": "Prediction Latency (p50, p95, p99)",
        "type": "timeseries",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(model_prediction_latency_seconds_bucket[5m]))",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.95, rate(model_prediction_latency_seconds_bucket[5m]))",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, rate(model_prediction_latency_seconds_bucket[5m]))",
            "legendFormat": "p99"
          }
        ]
      },
      {
        "title": "Predictions Per Second",
        "type": "stat",
        "targets": [
          {"expr": "rate(model_predictions_total[5m])"}
        ]
      },
      {
        "title": "Error Rate (%)",
        "type": "gauge",
        "targets": [
          {"expr": "rate(model_prediction_errors_total[5m]) / rate(model_predictions_total[5m]) * 100"}
        ],
        "thresholds": [
          {"value": 0, "color": "green"},
          {"value": 0.5, "color": "yellow"},
          {"value": 1, "color": "red"}
        ]
      },
      {
        "title": "Confidence Score Distribution",
        "type": "histogram",
        "targets": [
          {"expr": "model_prediction_confidence_bucket"}
        ]
      },
      {
        "title": "Feature Drift Scores",
        "type": "heatmap",
        "targets": [
          {"expr": "feature_drift_score"}
        ]
      }
    ]
  }
}
```

---

## Alerting and Incident Response

### Alert Severity Levels

```
Level 1: INFO (Log only)
├── Slight drift in non-critical features
├── Latency increase < 20%
└── Action: Log, review in weekly standup

Level 2: WARNING (Notify team)
├── Moderate drift detected (PSI 0.1-0.2)
├── Confidence scores dropping
├── Performance degradation < 10%
└── Action: Investigate within 24 hours

Level 3: CRITICAL (Immediate action)
├── Severe drift (PSI > 0.2)
├── Performance drop > 10%
├── Error rate spike
└── Action: Investigate immediately, consider rollback

Level 4: EMERGENCY (Auto-rollback)
├── Model server crash
├── Prediction latency > 5x baseline
├── Error rate > 5%
└── Action: Auto-rollback to last known good model
```

### Automated Response Pipeline

```python
# incident_response.py - Automated incident handling
import logging
from enum import Enum
from dataclasses import dataclass
from typing import Optional, Callable
from datetime import datetime

logger = logging.getLogger(__name__)

class Severity(Enum):
    INFO = 1
    WARNING = 2
    CRITICAL = 3
    EMERGENCY = 4

@dataclass
class Alert:
    name: str
    severity: Severity
    message: str
    metric_value: float
    threshold: float
    model_name: str
    timestamp: datetime = None
    
    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()

class IncidentResponder:
    """
    Automated incident response system for ML models.
    
    Escalation path:
    1. Detect anomaly → Generate alert
    2. Route alert by severity → Notify appropriate team
    3. If critical → Auto-rollback + notify
    4. Log everything for post-mortem
    """
    
    def __init__(self):
        self.alert_history = []
        self.handlers = {
            Severity.INFO: self._handle_info,
            Severity.WARNING: self._handle_warning,
            Severity.CRITICAL: self._handle_critical,
            Severity.EMERGENCY: self._handle_emergency,
        }
    
    def process_alert(self, alert: Alert):
        """Route alert to appropriate handler"""
        self.alert_history.append(alert)
        handler = self.handlers.get(alert.severity, self._handle_info)
        handler(alert)
    
    def _handle_info(self, alert: Alert):
        """Log only, no notification"""
        logger.info(f"[INFO] {alert.model_name}: {alert.message} "
                    f"(value={alert.metric_value}, threshold={alert.threshold})")
    
    def _handle_warning(self, alert: Alert):
        """Log + send Slack notification"""
        logger.warning(f"[WARNING] {alert.model_name}: {alert.message}")
        self._send_slack(
            channel="#ml-alerts",
            message=f"⚠️ *{alert.name}*\n"
                    f"Model: {alert.model_name}\n"
                    f"Message: {alert.message}\n"
                    f"Value: {alert.metric_value} (threshold: {alert.threshold})\n"
                    f"Action: Investigate within 24 hours"
        )
    
    def _handle_critical(self, alert: Alert):
        """Log + Slack + PagerDuty + prepare rollback"""
        logger.critical(f"[CRITICAL] {alert.model_name}: {alert.message}")
        self._send_slack(
            channel="#ml-alerts-critical",
            message=f"🚨 *CRITICAL: {alert.name}*\n"
                    f"Model: {alert.model_name}\n"
                    f"Message: {alert.message}\n"
                    f"Action: Immediate investigation required"
        )
        self._send_pagerduty(alert)
    
    def _handle_emergency(self, alert: Alert):
        """Auto-rollback + all notifications"""
        logger.critical(f"[EMERGENCY] {alert.model_name}: {alert.message}")
        logger.critical(f"Initiating auto-rollback for {alert.model_name}")
        
        # Auto-rollback to last known good model
        self._rollback_model(alert.model_name)
        
        self._send_slack(
            channel="#ml-alerts-critical",
            message=f"🔴 *EMERGENCY: {alert.name}*\n"
                    f"Model: {alert.model_name}\n"
                    f"Message: {alert.message}\n"
                    f"Action: AUTO-ROLLBACK initiated"
        )
        self._send_pagerduty(alert)
    
    def _rollback_model(self, model_name: str):
        """Rollback to previous model version"""
        logger.info(f"Rolling back {model_name} to previous version")
        # In practice: kubectl rollout undo deployment/<model_name>
        # or update model registry to point to previous version
    
    def _send_slack(self, channel: str, message: str):
        """Send Slack notification (implementation depends on your setup)"""
        # import slack_sdk
        # client = slack_sdk.WebClient(token=os.environ['SLACK_TOKEN'])
        # client.chat_postMessage(channel=channel, text=message)
        logger.info(f"Slack → {channel}: {message[:100]}...")
    
    def _send_pagerduty(self, alert: Alert):
        """Trigger PagerDuty incident"""
        logger.info(f"PagerDuty incident created for {alert.name}")


# ===== Integration with Monitoring =====

def evaluate_alerts(metrics: dict, responder: IncidentResponder):
    """Evaluate metrics and generate alerts"""
    model_name = metrics.get('model_name', 'unknown')
    
    # Check accuracy degradation
    if metrics.get('vs_baseline', {}).get('accuracy', {}).get('degraded'):
        change = metrics['vs_baseline']['accuracy']['change_pct']
        severity = Severity.CRITICAL if change > 15 else Severity.WARNING
        
        responder.process_alert(Alert(
            name="Model Accuracy Degradation",
            severity=severity,
            message=f"Accuracy dropped by {change:.1f}% from baseline",
            metric_value=metrics['accuracy'],
            threshold=metrics['vs_baseline']['accuracy']['baseline'],
            model_name=model_name
        ))
    
    # Check error rate
    error_rate = metrics.get('error_rate', 0)
    if error_rate > 0.05:
        responder.process_alert(Alert(
            name="High Error Rate",
            severity=Severity.EMERGENCY if error_rate > 0.1 else Severity.CRITICAL,
            message=f"Error rate at {error_rate:.1%}",
            metric_value=error_rate,
            threshold=0.05,
            model_name=model_name
        ))
```

---

## Monitoring Tools Ecosystem

### Tool Comparison

| Tool | Type | Open Source | Best For | Complexity |
|------|------|-----------|----------|------------|
| **Evidently AI** | ML Monitoring | ✅ | Data drift, model perf | Low |
| **WhyLabs** | ML Monitoring | Partial | Real-time profiling | Low |
| **NannyML** | ML Monitoring | ✅ | Performance estimation without labels | Low |
| **Prometheus + Grafana** | Infrastructure | ✅ | System metrics, alerting | Medium |
| **Arize AI** | ML Observability | ❌ | Full ML observability | Medium |
| **Fiddler AI** | ML Monitoring | ❌ | Explainability + monitoring | Medium |
| **MLflow** | Experiment Tracking | ✅ | Metric logging, comparison | Low |
| **Great Expectations** | Data Quality | ✅ | Data validation | Medium |

### Evidently AI (Most Popular Open-Source)

```python
# evidently_monitoring.py - Using Evidently for drift detection
import pandas as pd
import numpy as np
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset, TargetDriftPreset
from evidently.metrics import (
    DataDriftTable,
    DatasetDriftMetric,
    ColumnDriftMetric,
    ClassificationQualityMetric,
)
from evidently.test_suite import TestSuite
from evidently.tests import (
    TestShareOfDriftedColumns,
    TestColumnDrift,
    TestMeanInNSigmas,
    TestShareOfMissingValues,
)

# Prepare reference and production data
np.random.seed(42)
reference = pd.DataFrame({
    'feature_1': np.random.normal(0, 1, 5000),
    'feature_2': np.random.exponential(2, 5000),
    'feature_3': np.random.choice(['A', 'B', 'C'], 5000, p=[0.5, 0.3, 0.2]),
    'prediction': np.random.binomial(1, 0.1, 5000),
    'target': np.random.binomial(1, 0.1, 5000)
})

production = pd.DataFrame({
    'feature_1': np.random.normal(0.5, 1.2, 3000),   # Drift!
    'feature_2': np.random.exponential(2, 3000),       # No drift
    'feature_3': np.random.choice(['A', 'B', 'C', 'D'], 3000, p=[0.3, 0.3, 0.2, 0.2]),  # Drift!
    'prediction': np.random.binomial(1, 0.15, 3000),   # Drift!
    'target': np.random.binomial(1, 0.15, 3000)
})

# ===== 1. Data Drift Report =====
drift_report = Report(metrics=[
    DatasetDriftMetric(),       # Overall dataset drift
    DataDriftTable(),           # Per-feature drift
])

drift_report.run(
    reference_data=reference,
    current_data=production
)

# Save as HTML dashboard
drift_report.save_html("drift_report.html")

# Get programmatic results
result = drift_report.as_dict()
print(f"Dataset drift detected: {result['metrics'][0]['result']['dataset_drift']}")
print(f"Drifted features: {result['metrics'][0]['result']['number_of_drifted_columns']}")

# ===== 2. Automated Test Suite =====
test_suite = TestSuite(tests=[
    TestShareOfDriftedColumns(lt=0.3),          # Less than 30% features drifted
    TestColumnDrift(column_name='feature_1'),    # Specific feature drift
    TestMeanInNSigmas(column_name='feature_1', n=2),  # Mean within 2 std devs
    TestShareOfMissingValues(column_name='feature_2', lt=0.05),  # <5% missing
])

test_suite.run(
    reference_data=reference,
    current_data=production
)

# Check if all tests passed
test_results = test_suite.as_dict()
all_passed = all(
    test['status'] == 'SUCCESS' 
    for test in test_results['tests']
)
print(f"\nAll monitoring tests passed: {all_passed}")

for test in test_results['tests']:
    status = "✅" if test['status'] == 'SUCCESS' else "❌"
    print(f"  {status} {test['name']}: {test['status']}")

# Save test results
test_suite.save_html("test_results.html")
```

### NannyML (Performance Estimation Without Labels)

```python
# nannyml_monitoring.py - Estimate performance without ground truth
import nannyml as nml
import pandas as pd
import numpy as np

# NannyML uses Confidence-Based Performance Estimation (CBPE)
# It estimates what accuracy/F1/AUC WOULD be based on prediction patterns
# No labels needed!

# Prepare data
np.random.seed(42)

# Reference data (has labels)
reference = pd.DataFrame({
    'feature_1': np.random.normal(0, 1, 5000),
    'feature_2': np.random.exponential(2, 5000),
    'y_pred': np.random.binomial(1, 0.1, 5000),
    'y_pred_proba': np.random.beta(2, 20, 5000),  # Skewed toward 0
    'y_true': np.random.binomial(1, 0.1, 5000),
    'timestamp': pd.date_range('2024-01-01', periods=5000, freq='h')
})

# Production data (NO labels!)
production = pd.DataFrame({
    'feature_1': np.random.normal(0.3, 1.1, 3000),  # Slight drift
    'feature_2': np.random.exponential(2, 3000),
    'y_pred': np.random.binomial(1, 0.12, 3000),
    'y_pred_proba': np.random.beta(2, 18, 3000),
    'timestamp': pd.date_range('2024-07-01', periods=3000, freq='h')
})

# CBPE Performance Estimator
estimator = nml.CBPE(
    y_pred='y_pred',
    y_pred_proba='y_pred_proba',
    y_true='y_true',
    problem_type='classification_binary',
    timestamp_column_name='timestamp',
    chunk_size=500,
    metrics=['roc_auc', 'f1']
)

# Fit on reference data (learns the relationship between confidence and performance)
estimator.fit(reference)

# Estimate performance on production data (no labels needed!)
estimated_performance = estimator.estimate(production)

# Plot results
figure = estimated_performance.plot()
# figure.show()  # Opens interactive plot

# Get numerical results
for metric in estimated_performance.metrics:
    print(f"\n{metric.display_name}:")
    for chunk in estimated_performance.data:
        print(f"  Estimated: {chunk[metric.column_name]:.4f}, "
              f"Alert: {chunk[f'{metric.column_name}_alert']}")
```

---

## Building a Monitoring Pipeline

### End-to-End Monitoring Architecture

```python
# monitoring_pipeline.py - Complete production monitoring pipeline
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import json
import logging
from typing import Dict, List
from dataclasses import dataclass, asdict

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class MonitoringConfig:
    """Configuration for the monitoring pipeline"""
    model_name: str
    drift_check_interval_minutes: int = 60
    performance_check_interval_minutes: int = 360  # 6 hours
    min_samples_for_drift: int = 100
    psi_threshold: float = 0.1
    accuracy_drop_threshold: float = 0.05
    latency_p99_threshold_ms: float = 500
    confidence_drop_threshold: float = 0.1

class MonitoringPipeline:
    """
    Production monitoring pipeline that ties together:
    1. Data quality checks
    2. Drift detection
    3. Performance tracking
    4. Alerting
    
    Designed to run as a scheduled job (e.g., every hour via Airflow/cron).
    """
    
    def __init__(
        self, 
        config: MonitoringConfig,
        reference_data: pd.DataFrame,
        reference_predictions: np.ndarray,
    ):
        self.config = config
        self.reference_data = reference_data
        self.reference_predictions = reference_predictions
        
        # Initialize detectors
        self.drift_detector = DriftDetector(reference_data)
        self.feature_monitor = FeatureMonitor(reference_data)
        self.proxy_monitor = ProxyMonitor(reference_predictions)
    
    def run_checks(
        self, 
        production_data: pd.DataFrame,
        production_predictions: np.ndarray,
        ground_truth: np.ndarray = None,
        latency_ms: List[float] = None
    ) -> Dict:
        """
        Run all monitoring checks and return consolidated report.
        
        Args:
            production_data: Recent production input features
            production_predictions: Model predictions (probabilities)
            ground_truth: Actual labels (if available)
            latency_ms: Request latencies in milliseconds
        """
        report = {
            'timestamp': datetime.now().isoformat(),
            'model_name': self.config.model_name,
            'n_samples': len(production_data),
            'checks': {},
            'alerts': [],
            'overall_status': 'healthy'  # Will be updated
        }
        
        # 1. Data Quality
        logger.info("Running data quality checks...")
        quality = self.feature_monitor.check_window(production_data)
        report['checks']['data_quality'] = quality
        report['alerts'].extend(quality.get('alerts', []))
        
        # 2. Data Drift
        if len(production_data) >= self.config.min_samples_for_drift:
            logger.info("Running drift detection...")
            drift = self.drift_detector.full_report(production_data)
            report['checks']['drift'] = drift['summary']
            
            if drift['summary']['action_needed']:
                report['alerts'].append({
                    'type': 'data_drift',
                    'severity': 'warning',
                    'message': f"{drift['summary']['drift_percentage']:.0f}% of features show drift"
                })
        
        # 3. Proxy Performance (no labels needed)
        logger.info("Running proxy performance checks...")
        confidence_check = self.proxy_monitor.check_prediction_confidence(production_predictions)
        distribution_check = self.proxy_monitor.check_prediction_distribution(production_predictions)
        uncertainty_check = self.proxy_monitor.check_uncertainty(production_predictions)
        
        report['checks']['proxy_performance'] = {
            'confidence': confidence_check,
            'distribution': distribution_check,
            'uncertainty': uncertainty_check
        }
        
        for check in [confidence_check, distribution_check, uncertainty_check]:
            if check.get('alert'):
                report['alerts'].append({
                    'type': check['metric'],
                    'severity': 'warning',
                    'message': check['message']
                })
        
        # 4. Real Performance (if labels available)
        if ground_truth is not None:
            logger.info("Running performance evaluation with ground truth...")
            from sklearn.metrics import accuracy_score, f1_score
            
            y_pred = production_predictions.argmax(axis=1)
            accuracy = accuracy_score(ground_truth, y_pred)
            f1 = f1_score(ground_truth, y_pred, average='weighted')
            
            report['checks']['real_performance'] = {
                'accuracy': round(accuracy, 4),
                'f1': round(f1, 4)
            }
            
            # Check against threshold
            if accuracy < (1 - self.config.accuracy_drop_threshold):
                report['alerts'].append({
                    'type': 'performance_degradation',
                    'severity': 'critical',
                    'message': f"Accuracy dropped to {accuracy:.2%}"
                })
        
        # 5. System Health
        if latency_ms:
            p50 = np.percentile(latency_ms, 50)
            p95 = np.percentile(latency_ms, 95)
            p99 = np.percentile(latency_ms, 99)
            
            report['checks']['system_health'] = {
                'latency_p50_ms': round(p50, 1),
                'latency_p95_ms': round(p95, 1),
                'latency_p99_ms': round(p99, 1),
                'error_rate': 0  # Would come from actual error tracking
            }
            
            if p99 > self.config.latency_p99_threshold_ms:
                report['alerts'].append({
                    'type': 'high_latency',
                    'severity': 'warning',
                    'message': f"p99 latency {p99:.0f}ms exceeds threshold {self.config.latency_p99_threshold_ms}ms"
                })
        
        # Determine overall status
        severities = [a.get('severity', 'info') for a in report['alerts']]
        if 'critical' in severities or 'emergency' in severities:
            report['overall_status'] = 'critical'
        elif 'warning' in severities:
            report['overall_status'] = 'warning'
        else:
            report['overall_status'] = 'healthy'
        
        logger.info(f"Monitoring complete. Status: {report['overall_status']}, "
                    f"Alerts: {len(report['alerts'])}")
        
        return report


# ===== RUN THE PIPELINE =====

if __name__ == "__main__":
    # Setup
    np.random.seed(42)
    
    # Reference data (from training/validation)
    ref_data = pd.DataFrame({
        'age': np.random.normal(35, 10, 5000),
        'income': np.random.lognormal(10, 1, 5000),
        'score': np.random.uniform(300, 850, 5000),
    })
    ref_preds = np.column_stack([
        np.random.beta(2, 8, 5000),
        np.random.beta(8, 2, 5000)
    ])
    ref_preds = ref_preds / ref_preds.sum(axis=1, keepdims=True)
    
    # Production data (with some drift)
    prod_data = pd.DataFrame({
        'age': np.random.normal(40, 12, 1000),   # Drifted!
        'income': np.random.lognormal(10, 1, 1000),
        'score': np.random.uniform(300, 850, 1000),
    })
    prod_preds = np.column_stack([
        np.random.beta(3, 7, 1000),
        np.random.beta(7, 3, 1000)
    ])
    prod_preds = prod_preds / prod_preds.sum(axis=1, keepdims=True)
    
    latencies = np.random.exponential(50, 1000)  # Avg 50ms
    
    # Create and run pipeline
    config = MonitoringConfig(
        model_name="customer_churn_v2",
        psi_threshold=0.1,
        accuracy_drop_threshold=0.05
    )
    
    pipeline = MonitoringPipeline(config, ref_data, ref_preds)
    report = pipeline.run_checks(prod_data, prod_preds, latency_ms=latencies.tolist())
    
    # Print report
    print("\n" + "=" * 60)
    print(f"MONITORING REPORT: {report['model_name']}")
    print(f"Status: {report['overall_status'].upper()}")
    print(f"Samples: {report['n_samples']}")
    print(f"Alerts: {report['alerts'].__len__()}")
    print("=" * 60)
    
    for alert in report['alerts']:
        icon = {"critical": "🔴", "warning": "🟡", "info": "🔵"}.get(alert['severity'], "⚪")
        print(f"  {icon} [{alert['severity'].upper()}] {alert['message']}")
```

---

## Common Mistakes

### 1. Only Monitoring System Metrics

```
❌ WRONG: Only watching CPU, memory, latency
   → Model can be 100% wrong while system looks perfect
   
✅ CORRECT: Monitor at all three levels
   1. System: CPU, memory, latency, error rate
   2. Data: Feature distributions, null rates, schema
   3. Model: Predictions, confidence, accuracy (when labels arrive)
```

### 2. Using Training Data as Only Reference

```python
# ❌ WRONG: Compare production to training data
# Training data might not represent real production distribution

# ✅ CORRECT: Use validation data OR first week of production data
# as reference (after confirming model performs well)
reference = validation_data  # Better represents real-world distribution
```

### 3. Alerting on Every Statistical Blip

```python
# ❌ WRONG: Alert on every p-value < 0.05
# With 100 features, you'll get ~5 false positives every check!

# ✅ CORRECT: Use Bonferroni correction or practical significance
corrected_alpha = 0.05 / n_features  # Bonferroni correction

# Or better: combine statistical AND practical significance
if p_value < corrected_alpha and psi > 0.1:
    alert("Drift is both statistically AND practically significant")
```

### 4. Not Monitoring Prediction Distribution

```python
# ❌ Missing: Not tracking what the model is predicting
# If a fraud model suddenly says everything is fraudulent, 
# that's a problem even if you can't verify predictions yet

# ✅ Track prediction class distribution over time
from collections import Counter

prediction_counts = Counter(recent_predictions)
expected_positive_rate = 0.05  # 5% fraud expected
actual_positive_rate = prediction_counts[1] / len(recent_predictions)

if abs(actual_positive_rate - expected_positive_rate) > 0.03:
    alert(f"Positive prediction rate shifted: {expected_positive_rate:.0%} → {actual_positive_rate:.0%}")
```

### 5. Ignoring Segment-Level Monitoring

```python
# ❌ WRONG: Only monitor overall metrics
# Overall accuracy: 95% ✅
# But accuracy for "new users": 60% ❌ (hidden by the majority)

# ✅ CORRECT: Monitor per segment
for segment in ['new_users', 'returning_users', 'mobile', 'desktop', 'us', 'eu']:
    segment_data = production_data[production_data['segment'] == segment]
    if len(segment_data) > 100:
        metrics = compute_metrics(segment_data)
        if metrics['accuracy'] < threshold:
            alert(f"Performance degraded for segment: {segment}")
```

---

## Interview Questions

### Conceptual

**Q1: What's the difference between data drift and concept drift?**
> **A:** Data drift = $P(X)$ changes. The input distribution shifts but the relationship between inputs and outputs stays the same. Example: User demographics change seasonally.
> 
> Concept drift = $P(Y|X)$ changes. The relationship between inputs and outputs changes. Example: What constitutes "spam" evolves. Data drift is detectable without labels; concept drift requires labels to confirm.

**Q2: How do you monitor model performance when ground truth labels are delayed?**
> **A:** Use proxy metrics: (1) Prediction confidence distribution — if confidence drops, model is struggling. (2) Prediction distribution — if class ratios shift dramatically, something changed. (3) Feature drift as a leading indicator. (4) Use algorithms like NannyML's CBPE to estimate performance. (5) Set up a labeling pipeline for a sample of predictions.

**Q3: What is PSI and how do you interpret it?**
> **A:** Population Stability Index measures how much a distribution has shifted. PSI = $\sum (P_i - Q_i) \times \ln(P_i / Q_i)$. Interpretation: PSI < 0.1 = no significant change; 0.1–0.2 = moderate shift, investigate; > 0.2 = significant shift, action needed. It's the industry standard in finance and insurance.

**Q4: How would you design a monitoring system for 50 ML models in production?**
> **A:**
> - Centralized monitoring platform (Evidently/Arize/custom)
> - Standardized metrics: every model exports the same metrics format
> - Prometheus for collection, Grafana for dashboards
> - Tiered alerting: per-model thresholds based on criticality
> - Automated weekly drift reports
> - Shared reference data store (updated periodically)
> - Automated retraining triggers for high-drift models
> - On-call rotation for critical model alerts

**Q5: What triggers should cause an automatic model rollback?**
> **A:**
> - Error rate exceeds 5% (model server issues)
> - Latency p99 exceeds 5x baseline for 5+ minutes
> - 100% of predictions going to a single class (model collapse)
> - Memory/CPU exceeding safe limits (risk of crash)
> - Model returns NaN/null predictions
> 
> Always require human approval for: performance metric degradation (may need retraining, not rollback), data drift (new model might be better than old one for new data).

### System Design

**Q6: Design a real-time monitoring system for a fraud detection model.**
> **A:**
> 1. **Log everything**: Every prediction request/response with timestamp, features, prediction, confidence, latency
> 2. **Real-time pipeline**: Kafka → Flink/Spark Streaming → Prometheus
> 3. **Drift detection**: Hourly PSI checks on top features, ADWIN on rolling error rate
> 4. **Performance tracking**: Fast labels (fraud reports within hours), compute metrics on 6-hour windows
> 5. **Dashboards**: Real-time Grafana dashboard with latency, throughput, confidence distribution, fraud rate
> 6. **Alerts**: PagerDuty for critical (error rate spike), Slack for warning (drift detected)
> 7. **Segment monitoring**: By transaction type, geography, device type
> 8. **A/B testing framework**: Compare champion vs challenger model in production

---

## Quick Reference

### Drift Detection Methods Comparison

| Method | Best For | Pros | Cons |
|--------|----------|------|------|
| **KS Test** | Numerical features | Non-parametric, no assumptions | Sensitive to sample size |
| **PSI** | Industry standard | Easy to interpret, intuitive thresholds | Requires binning |
| **Chi-Squared** | Categorical features | Well understood | Needs sufficient category counts |
| **JSD** | Any distribution | Symmetric, bounded [0,1] | Requires binning |
| **ADWIN** | Streaming data | Adaptive window size | Only detects mean changes |
| **DDM** | Online classification | Simple, low memory | Needs error feedback |
| **MMD** | High-dimensional | Detects any distribution change | Computationally expensive |

### PSI Interpretation Guide

| PSI Value | Interpretation | Action |
|-----------|----------------|--------|
| < 0.1 | No significant drift | Continue monitoring |
| 0.1 – 0.2 | Moderate drift | Investigate, consider retraining |
| 0.2 – 0.5 | Significant drift | Retrain model |
| > 0.5 | Severe drift | Urgent retrain, possible model architecture change |

### Monitoring Checklist

| Category | Metric | Frequency | Tool |
|----------|--------|-----------|------|
| **System** | Latency (p50, p95, p99) | Real-time | Prometheus |
| **System** | Error rate | Real-time | Prometheus |
| **System** | Throughput (req/s) | Real-time | Prometheus |
| **System** | CPU/GPU/Memory | Real-time | Prometheus |
| **Data** | Feature distributions | Hourly | Evidently |
| **Data** | Null/missing rates | Hourly | Custom |
| **Data** | Schema changes | Per request | Great Expectations |
| **Data** | PSI per feature | Hourly/Daily | Custom |
| **Model** | Prediction distribution | Hourly | Custom |
| **Model** | Confidence scores | Real-time | Prometheus |
| **Model** | Accuracy/F1/AUC | When labels arrive | Custom |
| **Model** | Per-segment metrics | Daily | Custom |
| **Business** | Conversion rate | Daily | Analytics |
| **Business** | Revenue impact | Weekly | Analytics |

### Key Formulas

| Metric | Formula | Range |
|--------|---------|-------|
| PSI | $\sum (P_i - Q_i) \ln\frac{P_i}{Q_i}$ | [0, ∞) |
| KL Divergence | $\sum P(x) \ln\frac{P(x)}{Q(x)}$ | [0, ∞) |
| JS Divergence | $\frac{1}{2}KL(P \| M) + \frac{1}{2}KL(Q \| M)$ where $M = \frac{P+Q}{2}$ | [0, 1] |
| KS Statistic | $\sup_x |F_1(x) - F_2(x)|$ | [0, 1] |
| Wasserstein Distance | $\inf_{\gamma} \mathbb{E}[\|x - y\|]$ | [0, ∞) |

### Essential Prometheus Queries

```promql
# Request rate (per second)
rate(model_predictions_total[5m])

# Error rate (percentage)
rate(model_prediction_errors_total[5m]) / rate(model_predictions_total[5m]) * 100

# Latency percentiles
histogram_quantile(0.95, rate(model_prediction_latency_seconds_bucket[5m]))

# Prediction class distribution
rate(model_predictions_total{prediction_class="1"}[1h]) / rate(model_predictions_total[1h])

# Confidence below threshold
rate(model_prediction_confidence_bucket{le="0.5"}[1h]) / rate(model_prediction_confidence_count[1h])
```

### Decision Matrix: When to Retrain

```
                    Data Drift?
                   /          \
                 Yes           No
                /               \
        Concept Drift?     Performance OK?
         /        \          /        \
       Yes        No       Yes        No
        |          |        |          |
   RETRAIN    MONITOR   DO NOTHING   DEBUG
   (urgent)  (closely)              (system issue?)
```

---

> **Pro Tip:** Start monitoring from Day 1 of deployment, not when things break. The cost of setting up monitoring is tiny compared to the cost of a silent model failure. Begin with Evidently (free, open-source) + Prometheus/Grafana, then graduate to paid tools (Arize, WhyLabs) as your model count grows.
