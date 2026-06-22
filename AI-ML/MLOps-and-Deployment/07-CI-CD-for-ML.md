# CI/CD for Machine Learning

## Table of Contents
- [What is CI/CD for ML?](#what-is-cicd-for-ml)
- [Why It Matters](#why-it-matters)
- [How It Works](#how-it-works)
- [Testing ML Code](#testing-ml-code)
- [GitHub Actions for ML](#github-actions-for-ml)
- [Automated Retraining Pipelines](#automated-retraining-pipelines)
- [ML Pipeline Orchestration](#ml-pipeline-orchestration)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is CI/CD for ML?

### Simple Explanation

Imagine you're running a restaurant. Every time your chef changes the recipe (model), you want to:
1. **Test** it automatically (does it taste good? is it safe?)
2. **Package** it (put it in proper containers)
3. **Deploy** it to customers (push to production)

CI/CD for ML is like having a robot assistant that automatically tests your ML code, validates your models, and deploys them — every time you make a change.

### Formal Definition

**CI (Continuous Integration):** Automatically testing and validating code changes, data quality, and model performance whenever new code is pushed to a repository.

**CD (Continuous Delivery/Deployment):** Automatically packaging and deploying validated models to production environments with minimal human intervention.

**CT (Continuous Training):** Automatically retraining models when new data arrives or performance degrades — this is UNIQUE to ML and doesn't exist in traditional software.

---

## Why It Matters

### Real-World Relevance

| Without CI/CD for ML | With CI/CD for ML |
|---------------------|-------------------|
| "It works on my notebook" syndrome | Reproducible across environments |
| Manual model deployment (hours/days) | Automated deployment (minutes) |
| Silent model degradation | Automatic retraining triggers |
| No version control for data/models | Full lineage tracking |
| One data scientist = bottleneck | Team collaboration at scale |

### When You'd Use It

- **Any production ML system** — If your model serves real users, you need CI/CD
- **Team environments** — Multiple people working on the same model
- **Regulated industries** — Finance, healthcare need audit trails
- **High-frequency retraining** — Models that need daily/weekly updates
- **A/B testing** — Rolling out model versions to subsets of users

---

## How It Works

### The ML CI/CD Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ML CI/CD PIPELINE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  CODE    │───▶│  BUILD   │───▶│  TEST    │───▶│  DEPLOY  │      │
│  │  PUSH    │    │  & TRAIN │    │  & VALID │    │  & SERVE │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│       │                │                │               │            │
│       ▼                ▼                ▼               ▼            │
│  • Git commit     • Install deps   • Unit tests    • Stage deploy   │
│  • PR opened      • Train model    • Integration   • Canary test    │
│  • Schedule       • Build image    • Model perf    • Full rollout   │
│  • Data trigger   • Push artifact  • Data valid    • Monitor        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Traditional CI/CD vs ML CI/CD

```
Traditional Software:
  Code Change → Build → Test → Deploy → Done ✓

ML System:
  Code Change ─┐
  Data Change ──┼──▶ Build → Train → Validate → Test → Deploy → Monitor ─┐
  Config Change┘                                                          │
       ▲                                                                   │
       └──────────── Retrain Trigger (drift/schedule/manual) ◀────────────┘
```

### Key Differences from Traditional CI/CD

| Aspect | Traditional Software | ML Systems |
|--------|---------------------|------------|
| **Trigger** | Code change only | Code, data, config, or performance changes |
| **Testing** | Unit/integration tests | + Model validation, data tests |
| **Artifacts** | Binary/container | Model files + metadata + data version |
| **Rollback** | Previous version | Previous model version + its data context |
| **Monitoring** | Uptime, latency | + Prediction drift, data drift |

---

## Testing ML Code

### The ML Testing Pyramid

```
            ▲
           / \        Infrastructure Tests
          /   \       (Cloud resources, endpoints)
         /─────\
        /       \     Integration Tests  
       /         \    (Pipeline end-to-end)
      /───────────\
     /             \   Model Validation Tests
    /               \  (Performance, fairness, robustness)
   /─────────────────\
  /                   \ Data Tests
 /                     \(Schema, distribution, quality)
/───────────────────────\
         Unit Tests
   (Functions, transforms, utils)
```

### 1. Unit Tests for ML

```python
# tests/test_preprocessing.py
import pytest
import numpy as np
import pandas as pd
from sklearn.preprocessing import StandardScaler

# --- Test data transformations ---

def test_feature_engineering_age_bins():
    """Test that age binning produces correct categories."""
    # Arrange: Create sample data
    df = pd.DataFrame({'age': [5, 15, 25, 45, 70]})
    
    # Act: Apply feature engineering
    bins = [0, 12, 18, 35, 55, 100]
    labels = ['child', 'teen', 'young_adult', 'middle_aged', 'senior']
    df['age_group'] = pd.cut(df['age'], bins=bins, labels=labels)
    
    # Assert: Check correct binning
    expected = ['child', 'teen', 'young_adult', 'middle_aged', 'senior']
    assert list(df['age_group']) == expected


def test_handle_missing_values():
    """Test that missing values are handled correctly."""
    df = pd.DataFrame({
        'numeric_col': [1.0, np.nan, 3.0, np.nan, 5.0],
        'categorical_col': ['a', None, 'b', 'c', None]
    })
    
    # Fill numeric with median
    df['numeric_col'].fillna(df['numeric_col'].median(), inplace=True)
    # Fill categorical with mode
    df['categorical_col'].fillna(df['categorical_col'].mode()[0], inplace=True)
    
    # No missing values should remain
    assert df['numeric_col'].isna().sum() == 0
    assert df['categorical_col'].isna().sum() == 0
    # Median of [1, 3, 5] = 3.0
    assert df['numeric_col'].iloc[1] == 3.0


def test_text_preprocessing():
    """Test NLP preprocessing pipeline."""
    import re
    
    def clean_text(text):
        """Lowercase, remove special chars, strip whitespace."""
        text = text.lower()
        text = re.sub(r'[^a-z0-9\s]', '', text)
        text = re.sub(r'\s+', ' ', text).strip()
        return text
    
    assert clean_text("Hello, World! 123") == "hello world 123"
    assert clean_text("  Multiple   Spaces  ") == "multiple spaces"
    assert clean_text("Special@#$Chars") == "specialchars"


def test_scaling_preserves_shape():
    """Test that scaling doesn't change data dimensions."""
    X = np.random.randn(100, 5)  # 100 samples, 5 features
    
    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)
    
    assert X_scaled.shape == X.shape
    # Mean should be ~0, std should be ~1 after scaling
    np.testing.assert_array_almost_equal(X_scaled.mean(axis=0), 0, decimal=10)
    np.testing.assert_array_almost_equal(X_scaled.std(axis=0), 1, decimal=10)


# --- Test model input/output contracts ---

def test_model_input_shape():
    """Test that model accepts expected input dimensions."""
    from sklearn.ensemble import RandomForestClassifier
    
    # Train a simple model
    X_train = np.random.randn(50, 4)  # 4 features
    y_train = np.random.randint(0, 2, 50)
    
    model = RandomForestClassifier(n_estimators=10, random_state=42)
    model.fit(X_train, y_train)
    
    # Test with correct shape
    X_test = np.random.randn(10, 4)
    predictions = model.predict(X_test)
    assert predictions.shape == (10,)
    
    # Test with wrong shape should raise error
    X_wrong = np.random.randn(10, 5)  # 5 features instead of 4
    with pytest.raises(ValueError):
        model.predict(X_wrong)


def test_prediction_output_range():
    """Test that predictions are within expected bounds."""
    from sklearn.linear_model import LogisticRegression
    
    X = np.random.randn(100, 3)
    y = np.random.randint(0, 2, 100)
    
    model = LogisticRegression(random_state=42)
    model.fit(X, y)
    
    # Probabilities should be between 0 and 1
    probas = model.predict_proba(X)
    assert np.all(probas >= 0)
    assert np.all(probas <= 1)
    # Probabilities should sum to 1
    np.testing.assert_array_almost_equal(probas.sum(axis=1), 1.0)
```

### 2. Data Validation Tests

```python
# tests/test_data_quality.py
import pytest
import pandas as pd
import numpy as np
from datetime import datetime

# Using Great Expectations-style assertions

class TestDataQuality:
    """Test suite for incoming data quality."""
    
    @pytest.fixture
    def sample_data(self):
        """Load or create sample training data."""
        return pd.DataFrame({
            'user_id': range(1000),
            'age': np.random.randint(18, 80, 1000),
            'income': np.random.uniform(20000, 200000, 1000),
            'credit_score': np.random.randint(300, 850, 1000),
            'loan_approved': np.random.randint(0, 2, 1000)
        })
    
    def test_no_duplicate_ids(self, sample_data):
        """Each user should appear only once."""
        assert sample_data['user_id'].is_unique, \
            f"Found {sample_data['user_id'].duplicated().sum()} duplicate IDs"
    
    def test_age_in_valid_range(self, sample_data):
        """Age should be between 18 and 120."""
        assert sample_data['age'].between(18, 120).all(), \
            f"Found ages outside [18, 120]: {sample_data[~sample_data['age'].between(18, 120)]['age'].tolist()}"
    
    def test_no_negative_income(self, sample_data):
        """Income should never be negative."""
        assert (sample_data['income'] >= 0).all(), \
            "Found negative income values"
    
    def test_credit_score_range(self, sample_data):
        """Credit scores should be between 300 and 850."""
        assert sample_data['credit_score'].between(300, 850).all()
    
    def test_target_class_balance(self, sample_data):
        """Target shouldn't be extremely imbalanced (> 95/5 split)."""
        class_ratio = sample_data['loan_approved'].value_counts(normalize=True)
        assert class_ratio.min() > 0.05, \
            f"Extreme class imbalance detected: {class_ratio.to_dict()}"
    
    def test_missing_values_threshold(self, sample_data):
        """No column should have > 20% missing values."""
        missing_pct = sample_data.isnull().mean()
        problematic = missing_pct[missing_pct > 0.20]
        assert len(problematic) == 0, \
            f"Columns with >20% missing: {problematic.to_dict()}"
    
    def test_schema_consistency(self, sample_data):
        """Data schema should match expected columns and types."""
        expected_schema = {
            'user_id': 'int64',
            'age': 'int64',
            'income': 'float64',
            'credit_score': 'int64',
            'loan_approved': 'int64'
        }
        for col, dtype in expected_schema.items():
            assert col in sample_data.columns, f"Missing column: {col}"
            assert str(sample_data[col].dtype) == dtype, \
                f"Column {col}: expected {dtype}, got {sample_data[col].dtype}"


# --- Data Drift Detection ---

def test_feature_distribution_drift(reference_data, current_data):
    """
    Test that feature distributions haven't drifted significantly
    using Kolmogorov-Smirnov test.
    """
    from scipy import stats
    
    DRIFT_THRESHOLD = 0.05  # p-value threshold
    
    drifted_features = []
    for col in reference_data.select_dtypes(include=[np.number]).columns:
        statistic, p_value = stats.ks_2samp(
            reference_data[col].dropna(),
            current_data[col].dropna()
        )
        if p_value < DRIFT_THRESHOLD:
            drifted_features.append((col, p_value))
    
    assert len(drifted_features) == 0, \
        f"Distribution drift detected in: {drifted_features}"
```

### 3. Model Validation Tests

```python
# tests/test_model_performance.py
import pytest
import numpy as np
import joblib
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score
from sklearn.model_selection import cross_val_score

class TestModelPerformance:
    """Validate model meets minimum performance thresholds."""
    
    # Performance gates — model MUST meet these to deploy
    MIN_ACCURACY = 0.85
    MIN_F1_SCORE = 0.80
    MIN_AUC = 0.82
    MAX_LATENCY_MS = 100  # prediction latency
    
    @pytest.fixture
    def model_and_data(self):
        """Load trained model and test data."""
        model = joblib.load('models/latest/model.pkl')
        X_test = np.load('data/processed/X_test.npy')
        y_test = np.load('data/processed/y_test.npy')
        return model, X_test, y_test
    
    def test_accuracy_threshold(self, model_and_data):
        """Model accuracy must exceed minimum threshold."""
        model, X_test, y_test = model_and_data
        y_pred = model.predict(X_test)
        accuracy = accuracy_score(y_test, y_pred)
        
        assert accuracy >= self.MIN_ACCURACY, \
            f"Accuracy {accuracy:.4f} below threshold {self.MIN_ACCURACY}"
    
    def test_f1_score_threshold(self, model_and_data):
        """F1 score must exceed minimum threshold."""
        model, X_test, y_test = model_and_data
        y_pred = model.predict(X_test)
        f1 = f1_score(y_test, y_pred, average='weighted')
        
        assert f1 >= self.MIN_F1_SCORE, \
            f"F1 score {f1:.4f} below threshold {self.MIN_F1_SCORE}"
    
    def test_auc_threshold(self, model_and_data):
        """AUC-ROC must exceed minimum threshold."""
        model, X_test, y_test = model_and_data
        y_proba = model.predict_proba(X_test)[:, 1]
        auc = roc_auc_score(y_test, y_proba)
        
        assert auc >= self.MIN_AUC, \
            f"AUC {auc:.4f} below threshold {self.MIN_AUC}"
    
    def test_no_performance_regression(self, model_and_data):
        """New model should not be worse than the previous deployed model."""
        model, X_test, y_test = model_and_data
        
        # Load previous model's metrics
        import json
        with open('models/production/metrics.json', 'r') as f:
            prev_metrics = json.load(f)
        
        # Current model metrics
        y_pred = model.predict(X_test)
        current_accuracy = accuracy_score(y_test, y_pred)
        
        # Allow max 2% degradation (sometimes new data is harder)
        REGRESSION_TOLERANCE = 0.02
        assert current_accuracy >= prev_metrics['accuracy'] - REGRESSION_TOLERANCE, \
            f"Performance regression: {current_accuracy:.4f} vs previous {prev_metrics['accuracy']:.4f}"
    
    def test_prediction_latency(self, model_and_data):
        """Single prediction should be fast enough for production."""
        import time
        
        model, X_test, _ = model_and_data
        single_sample = X_test[:1]
        
        # Warm up
        model.predict(single_sample)
        
        # Measure latency over 100 predictions
        latencies = []
        for _ in range(100):
            start = time.perf_counter()
            model.predict(single_sample)
            latencies.append((time.perf_counter() - start) * 1000)
        
        p95_latency = np.percentile(latencies, 95)
        assert p95_latency < self.MAX_LATENCY_MS, \
            f"P95 latency {p95_latency:.1f}ms exceeds {self.MAX_LATENCY_MS}ms"
    
    def test_model_fairness(self, model_and_data):
        """Model should not show extreme bias across demographic groups."""
        model, X_test, y_test = model_and_data
        
        # Assuming column index 2 is a protected attribute (e.g., gender)
        # In production, use proper fairness libraries like Fairlearn
        group_0 = X_test[:, 2] == 0
        group_1 = X_test[:, 2] == 1
        
        if group_0.sum() > 0 and group_1.sum() > 0:
            acc_0 = accuracy_score(y_test[group_0], model.predict(X_test[group_0]))
            acc_1 = accuracy_score(y_test[group_1], model.predict(X_test[group_1]))
            
            # Disparate impact ratio should be > 0.8 (80% rule)
            ratio = min(acc_0, acc_1) / max(acc_0, acc_1)
            assert ratio > 0.8, \
                f"Fairness violation: accuracy ratio = {ratio:.3f} (group_0={acc_0:.3f}, group_1={acc_1:.3f})"
    
    def test_model_robustness(self, model_and_data):
        """Model should be stable with slight input perturbations."""
        model, X_test, _ = model_and_data
        
        # Add small noise to inputs
        noise = np.random.normal(0, 0.01, X_test.shape)
        X_perturbed = X_test + noise
        
        original_preds = model.predict(X_perturbed)
        clean_preds = model.predict(X_test)
        
        # At least 95% of predictions should remain the same
        stability = (original_preds == clean_preds).mean()
        assert stability > 0.95, \
            f"Model unstable: only {stability:.1%} predictions consistent after small perturbation"
```

### 4. Integration Tests

```python
# tests/test_pipeline_integration.py
import pytest
import requests
import json
import time

class TestPipelineIntegration:
    """End-to-end pipeline integration tests."""
    
    def test_full_training_pipeline(self, tmp_path):
        """Test entire training pipeline runs without errors."""
        import subprocess
        
        result = subprocess.run(
            ['python', 'src/train.py', '--config', 'configs/test_config.yaml'],
            capture_output=True,
            text=True,
            timeout=300  # 5 min timeout
        )
        
        assert result.returncode == 0, \
            f"Training pipeline failed: {result.stderr}"
    
    def test_prediction_endpoint(self):
        """Test that the model serving endpoint works correctly."""
        # Sample payload
        payload = {
            "features": [25, 50000, 720, 3, 1]  # age, income, credit, etc.
        }
        
        response = requests.post(
            "http://localhost:8000/predict",
            json=payload,
            timeout=5
        )
        
        assert response.status_code == 200
        result = response.json()
        assert 'prediction' in result
        assert 'probability' in result
        assert 0 <= result['probability'] <= 1
    
    def test_batch_prediction_pipeline(self, tmp_path):
        """Test batch prediction pipeline end-to-end."""
        import pandas as pd
        
        # Create test batch input
        test_batch = pd.DataFrame({
            'age': [25, 35, 45],
            'income': [50000, 75000, 100000],
            'credit_score': [700, 750, 800]
        })
        input_path = tmp_path / "test_batch.csv"
        test_batch.to_csv(input_path, index=False)
        
        # Run batch prediction
        import subprocess
        result = subprocess.run(
            ['python', 'src/predict_batch.py', '--input', str(input_path)],
            capture_output=True,
            text=True,
            timeout=60
        )
        
        assert result.returncode == 0
        
        # Check output exists and has correct shape
        output_path = tmp_path / "test_batch_predictions.csv"
        if output_path.exists():
            predictions = pd.read_csv(output_path)
            assert len(predictions) == 3
            assert 'prediction' in predictions.columns
```

---

## GitHub Actions for ML

### Basic ML CI/CD Workflow

```yaml
# .github/workflows/ml-pipeline.yml
name: ML CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  # Schedule for periodic retraining
  schedule:
    - cron: '0 2 * * 1'  # Every Monday at 2 AM

env:
  PYTHON_VERSION: '3.10'
  MODEL_REGISTRY: 'models/'
  
jobs:
  # ============================================
  # JOB 1: Code Quality & Unit Tests
  # ============================================
  code-quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Lint with flake8
        run: |
          flake8 src/ tests/ --max-line-length=120 --statistics
      
      - name: Type check with mypy
        run: |
          mypy src/ --ignore-missing-imports
      
      - name: Run unit tests
        run: |
          pytest tests/unit/ -v --cov=src --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml

  # ============================================
  # JOB 2: Data Validation
  # ============================================
  data-validation:
    runs-on: ubuntu-latest
    needs: code-quality
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Pull latest data (DVC)
        run: |
          pip install dvc[s3]
          dvc pull data/
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Validate data quality
        run: |
          pytest tests/data/ -v --tb=short
      
      - name: Check for data drift
        run: |
          python src/monitoring/check_drift.py \
            --reference data/reference/train.csv \
            --current data/latest/train.csv \
            --threshold 0.05

  # ============================================
  # JOB 3: Model Training & Validation
  # ============================================
  train-and-validate:
    runs-on: ubuntu-latest
    needs: data-validation
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Pull data
        run: |
          pip install dvc[s3]
          dvc pull
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Train model
        run: |
          python src/train.py \
            --config configs/production.yaml \
            --output models/candidate/
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
          WANDB_API_KEY: ${{ secrets.WANDB_API_KEY }}
      
      - name: Run model validation tests
        run: |
          pytest tests/model/ -v --tb=short
      
      - name: Compare with production model
        id: compare
        run: |
          python src/evaluate.py \
            --candidate models/candidate/model.pkl \
            --production models/production/model.pkl \
            --output comparison_report.json
          
          # Set output for conditional deployment
          IMPROVED=$(python -c "
          import json
          with open('comparison_report.json') as f:
              report = json.load(f)
          print('true' if report['candidate_better'] else 'false')
          ")
          echo "model_improved=$IMPROVED" >> $GITHUB_OUTPUT
      
      - name: Upload model artifact
        if: steps.compare.outputs.model_improved == 'true'
        uses: actions/upload-artifact@v4
        with:
          name: trained-model
          path: models/candidate/
      
      - name: Upload comparison report
        uses: actions/upload-artifact@v4
        with:
          name: comparison-report
          path: comparison_report.json
    
    outputs:
      model_improved: ${{ steps.compare.outputs.model_improved }}

  # ============================================
  # JOB 4: Build & Push Container
  # ============================================
  build-container:
    runs-on: ubuntu-latest
    needs: train-and-validate
    if: needs.train-and-validate.outputs.model_improved == 'true'
    steps:
      - uses: actions/checkout@v4
      
      - name: Download model artifact
        uses: actions/download-artifact@v4
        with:
          name: trained-model
          path: models/candidate/
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/model-server:${{ github.sha }}
            ghcr.io/${{ github.repository }}/model-server:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ============================================
  # JOB 5: Deploy to Staging
  # ============================================
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-container
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        run: |
          # Using kubectl for K8s deployment
          kubectl set image deployment/model-server \
            model-server=ghcr.io/${{ github.repository }}/model-server:${{ github.sha }} \
            --namespace=staging
      
      - name: Run smoke tests
        run: |
          # Wait for rollout
          kubectl rollout status deployment/model-server --namespace=staging --timeout=300s
          
          # Run smoke tests against staging
          pytest tests/smoke/ -v \
            --base-url=https://staging-api.example.com
      
      - name: Run integration tests
        run: |
          pytest tests/integration/ -v \
            --base-url=https://staging-api.example.com

  # ============================================
  # JOB 6: Deploy to Production (Manual Gate)
  # ============================================
  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production  # Requires manual approval
    steps:
      - uses: actions/checkout@v4
      
      - name: Canary deployment (10% traffic)
        run: |
          kubectl apply -f k8s/canary-deployment.yaml --namespace=production
      
      - name: Monitor canary (5 minutes)
        run: |
          python src/monitoring/canary_check.py \
            --duration 300 \
            --error-threshold 0.01 \
            --latency-threshold 200
      
      - name: Full rollout
        run: |
          kubectl set image deployment/model-server \
            model-server=ghcr.io/${{ github.repository }}/model-server:${{ github.sha }} \
            --namespace=production
          kubectl rollout status deployment/model-server --namespace=production
      
      - name: Update model registry
        run: |
          python src/registry/promote_model.py \
            --model-path models/candidate/ \
            --stage production \
            --version ${{ github.sha }}
```

### GitHub Actions for ML with DVC

```yaml
# .github/workflows/ml-experiment.yml
name: ML Experiment Tracking

on:
  pull_request:
    branches: [main]

jobs:
  experiment:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for DVC
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install dvc[s3]
      
      - name: Pull data with DVC
        run: dvc pull
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Reproduce pipeline
        run: dvc repro
      
      - name: Generate metrics diff
        id: metrics
        run: |
          # Compare metrics with main branch
          dvc metrics diff --md >> metrics_diff.md
          echo "## Model Metrics Comparison" > comment.md
          echo "" >> comment.md
          cat metrics_diff.md >> comment.md
          
          # Add plots
          dvc plots diff --open=false
      
      - name: Comment PR with metrics
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const comment = fs.readFileSync('comment.md', 'utf8');
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
```

### Automated Testing Configuration

```python
# conftest.py - Shared pytest fixtures for ML testing
import pytest
import numpy as np
import pandas as pd
import joblib
import os

@pytest.fixture(scope="session")
def model():
    """Load the trained model once for all tests."""
    model_path = os.environ.get('MODEL_PATH', 'models/latest/model.pkl')
    return joblib.load(model_path)

@pytest.fixture(scope="session")
def test_data():
    """Load test dataset once for all tests."""
    X_test = np.load('data/processed/X_test.npy')
    y_test = np.load('data/processed/y_test.npy')
    return X_test, y_test

@pytest.fixture(scope="session")
def reference_data():
    """Load reference data for drift detection."""
    return pd.read_csv('data/reference/train_stats.csv')

@pytest.fixture
def sample_prediction_input():
    """Generate a valid sample input for prediction."""
    return {
        "age": 30,
        "income": 65000,
        "credit_score": 720,
        "employment_years": 5,
        "num_dependents": 2
    }

# Custom markers
def pytest_configure(config):
    config.addinivalue_line("markers", "slow: marks tests as slow (deselect with '-m \"not slow\"')")
    config.addinivalue_line("markers", "gpu: marks tests requiring GPU")
    config.addinivalue_line("markers", "integration: marks integration tests")
```

---

## Automated Retraining Pipelines

### Architecture of Automated Retraining

```
┌─────────────────────────────────────────────────────────────────┐
│                   AUTOMATED RETRAINING SYSTEM                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  TRIGGERS                    PIPELINE                            │
│  ┌──────────────┐           ┌─────────────────────────┐         │
│  │ 📅 Schedule  │──┐        │  1. Fetch latest data    │         │
│  │ (daily/weekly)│  │        │  2. Validate data        │         │
│  └──────────────┘  │        │  3. Feature engineering   │         │
│  ┌──────────────┐  ├───────▶│  4. Train model           │         │
│  │ 📊 Drift     │  │        │  5. Evaluate performance  │         │
│  │ Detected     │──┘        │  6. Compare vs production │         │
│  └──────────────┘  │        │  7. Deploy if better      │         │
│  ┌──────────────┐  │        └─────────────────────────┘         │
│  │ 📈 New Data  │──┘                    │                        │
│  │ Threshold    │                        ▼                        │
│  └──────────────┘           ┌─────────────────────────┐         │
│                              │  MODEL REGISTRY           │         │
│                              │  • Version tracking       │         │
│                              │  • A/B test assignment    │         │
│                              │  • Rollback capability    │         │
│                              └─────────────────────────┘         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation with Python

```python
# src/retraining/automated_retrain.py
"""
Automated model retraining pipeline.
Triggered by: schedule, drift detection, or manual invocation.
"""

import os
import json
import logging
from datetime import datetime
from pathlib import Path
from typing import Dict, Tuple, Optional

import numpy as np
import pandas as pd
import joblib
import mlflow
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score
from scipy import stats

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


class AutomatedRetrainingPipeline:
    """
    End-to-end automated retraining pipeline.
    
    Handles: data fetching, validation, training, evaluation,
    comparison with production model, and conditional deployment.
    """
    
    def __init__(self, config_path: str):
        """
        Initialize pipeline with configuration.
        
        Args:
            config_path: Path to YAML/JSON config file
        """
        with open(config_path) as f:
            self.config = json.load(f)
        
        self.model_dir = Path(self.config['model_dir'])
        self.data_dir = Path(self.config['data_dir'])
        self.min_improvement = self.config.get('min_improvement', 0.01)
        self.drift_threshold = self.config.get('drift_threshold', 0.05)
        
        # MLflow setup
        mlflow.set_tracking_uri(self.config.get('mlflow_uri', 'mlruns'))
        mlflow.set_experiment(self.config.get('experiment_name', 'auto-retrain'))
    
    def check_retraining_trigger(self) -> Dict[str, bool]:
        """
        Determine if retraining is needed.
        
        Returns:
            Dictionary with trigger reasons and their status
        """
        triggers = {
            'scheduled': self._check_schedule_trigger(),
            'data_drift': self._check_data_drift(),
            'performance_drop': self._check_performance_drop(),
            'new_data_volume': self._check_new_data_volume()
        }
        
        logger.info(f"Retraining triggers: {triggers}")
        return triggers
    
    def _check_data_drift(self) -> bool:
        """Check if input data distribution has shifted significantly."""
        reference = pd.read_csv(self.data_dir / 'reference' / 'features.csv')
        current = pd.read_csv(self.data_dir / 'latest' / 'features.csv')
        
        drifted_features = 0
        total_features = len(reference.select_dtypes(include=[np.number]).columns)
        
        for col in reference.select_dtypes(include=[np.number]).columns:
            _, p_value = stats.ks_2samp(
                reference[col].dropna(),
                current[col].dropna()
            )
            if p_value < self.drift_threshold:
                drifted_features += 1
                logger.warning(f"Drift detected in feature '{col}' (p={p_value:.4f})")
        
        # Trigger if >20% of features have drifted
        drift_ratio = drifted_features / total_features
        return drift_ratio > 0.20
    
    def _check_performance_drop(self) -> bool:
        """Check if production model performance has degraded."""
        metrics_path = self.model_dir / 'production' / 'live_metrics.json'
        if not metrics_path.exists():
            return False
        
        with open(metrics_path) as f:
            live_metrics = json.load(f)
        
        with open(self.model_dir / 'production' / 'baseline_metrics.json') as f:
            baseline = json.load(f)
        
        # Check if accuracy dropped more than 5% from baseline
        accuracy_drop = baseline['accuracy'] - live_metrics['accuracy']
        return accuracy_drop > 0.05
    
    def _check_new_data_volume(self) -> bool:
        """Check if enough new data has accumulated for retraining."""
        new_data_path = self.data_dir / 'new'
        if not new_data_path.exists():
            return False
        
        new_data = pd.read_csv(new_data_path / 'data.csv')
        min_samples = self.config.get('min_new_samples', 1000)
        return len(new_data) >= min_samples
    
    def _check_schedule_trigger(self) -> bool:
        """Check if enough time has passed since last retraining."""
        last_train_file = self.model_dir / 'last_training.json'
        if not last_train_file.exists():
            return True
        
        with open(last_train_file) as f:
            last_train = json.load(f)
        
        last_date = datetime.fromisoformat(last_train['timestamp'])
        days_since = (datetime.now() - last_date).days
        schedule_days = self.config.get('retrain_schedule_days', 7)
        
        return days_since >= schedule_days
    
    def fetch_and_prepare_data(self) -> Tuple[np.ndarray, np.ndarray, np.ndarray, np.ndarray]:
        """
        Fetch latest data and prepare train/test splits.
        
        Returns:
            Tuple of (X_train, X_test, y_train, y_test)
        """
        logger.info("Fetching and preparing data...")
        
        # Combine existing training data with new data
        train_data = pd.read_csv(self.data_dir / 'processed' / 'train.csv')
        
        new_data_path = self.data_dir / 'new' / 'data.csv'
        if new_data_path.exists():
            new_data = pd.read_csv(new_data_path)
            train_data = pd.concat([train_data, new_data], ignore_index=True)
            logger.info(f"Added {len(new_data)} new samples. Total: {len(train_data)}")
        
        # Feature/target split
        target_col = self.config['target_column']
        feature_cols = [c for c in train_data.columns if c != target_col]
        
        X = train_data[feature_cols].values
        y = train_data[target_col].values
        
        # Train/test split with stratification
        X_train, X_test, y_train, y_test = train_test_split(
            X, y, test_size=0.2, random_state=42, stratify=y
        )
        
        logger.info(f"Data prepared: train={len(X_train)}, test={len(X_test)}")
        return X_train, X_test, y_train, y_test
    
    def train_model(self, X_train, y_train) -> object:
        """
        Train model with hyperparameter optimization.
        
        Args:
            X_train: Training features
            y_train: Training labels
            
        Returns:
            Trained model object
        """
        from sklearn.ensemble import GradientBoostingClassifier
        from sklearn.model_selection import RandomizedSearchCV
        
        logger.info("Training model with hyperparameter search...")
        
        # Define search space
        param_distributions = {
            'n_estimators': [100, 200, 300, 500],
            'max_depth': [3, 5, 7, 10],
            'learning_rate': [0.01, 0.05, 0.1, 0.2],
            'subsample': [0.8, 0.9, 1.0],
            'min_samples_split': [2, 5, 10]
        }
        
        # Randomized search with cross-validation
        base_model = GradientBoostingClassifier(random_state=42)
        search = RandomizedSearchCV(
            base_model,
            param_distributions,
            n_iter=20,
            cv=5,
            scoring='f1_weighted',
            n_jobs=-1,
            random_state=42
        )
        
        search.fit(X_train, y_train)
        
        logger.info(f"Best params: {search.best_params_}")
        logger.info(f"Best CV score: {search.best_score_:.4f}")
        
        return search.best_estimator_
    
    def evaluate_model(self, model, X_test, y_test) -> Dict[str, float]:
        """
        Comprehensive model evaluation.
        
        Returns:
            Dictionary of metric names and values
        """
        y_pred = model.predict(X_test)
        y_proba = model.predict_proba(X_test)[:, 1]
        
        metrics = {
            'accuracy': accuracy_score(y_test, y_pred),
            'f1_score': f1_score(y_test, y_pred, average='weighted'),
            'auc_roc': roc_auc_score(y_test, y_proba),
            'timestamp': datetime.now().isoformat(),
            'test_samples': len(y_test)
        }
        
        logger.info(f"Model metrics: {metrics}")
        return metrics
    
    def compare_with_production(self, new_metrics: Dict) -> bool:
        """
        Compare new model with current production model.
        
        Returns:
            True if new model is better and should be deployed
        """
        prod_metrics_path = self.model_dir / 'production' / 'metrics.json'
        
        if not prod_metrics_path.exists():
            logger.info("No production model found. New model will be deployed.")
            return True
        
        with open(prod_metrics_path) as f:
            prod_metrics = json.load(f)
        
        # Primary metric for comparison
        primary_metric = self.config.get('primary_metric', 'f1_score')
        
        improvement = new_metrics[primary_metric] - prod_metrics[primary_metric]
        
        logger.info(
            f"Comparison - Production {primary_metric}: {prod_metrics[primary_metric]:.4f}, "
            f"Candidate: {new_metrics[primary_metric]:.4f}, "
            f"Improvement: {improvement:.4f}"
        )
        
        return improvement >= self.min_improvement
    
    def deploy_model(self, model, metrics: Dict):
        """Deploy new model to production."""
        logger.info("Deploying new model to production...")
        
        # Save model
        prod_dir = self.model_dir / 'production'
        prod_dir.mkdir(parents=True, exist_ok=True)
        
        # Archive current production model
        archive_dir = self.model_dir / 'archive' / datetime.now().strftime('%Y%m%d_%H%M%S')
        if (prod_dir / 'model.pkl').exists():
            archive_dir.mkdir(parents=True, exist_ok=True)
            import shutil
            shutil.copytree(prod_dir, archive_dir, dirs_exist_ok=True)
        
        # Save new model
        joblib.dump(model, prod_dir / 'model.pkl')
        
        # Save metrics
        with open(prod_dir / 'metrics.json', 'w') as f:
            json.dump(metrics, f, indent=2)
        
        # Save baseline metrics for monitoring
        with open(prod_dir / 'baseline_metrics.json', 'w') as f:
            json.dump(metrics, f, indent=2)
        
        # Update last training timestamp
        with open(self.model_dir / 'last_training.json', 'w') as f:
            json.dump({'timestamp': datetime.now().isoformat()}, f)
        
        logger.info("Model deployed successfully!")
    
    def run(self):
        """Execute the full retraining pipeline."""
        logger.info("=" * 60)
        logger.info("STARTING AUTOMATED RETRAINING PIPELINE")
        logger.info("=" * 60)
        
        # Step 1: Check if retraining is needed
        triggers = self.check_retraining_trigger()
        if not any(triggers.values()):
            logger.info("No retraining trigger active. Exiting.")
            return
        
        # Log to MLflow
        with mlflow.start_run(run_name=f"auto-retrain-{datetime.now().strftime('%Y%m%d')}"):
            mlflow.log_params({"triggers": str(triggers)})
            
            # Step 2: Prepare data
            X_train, X_test, y_train, y_test = self.fetch_and_prepare_data()
            
            # Step 3: Train model
            model = self.train_model(X_train, y_train)
            
            # Step 4: Evaluate
            metrics = self.evaluate_model(model, X_test, y_test)
            mlflow.log_metrics(metrics)
            
            # Step 5: Compare with production
            should_deploy = self.compare_with_production(metrics)
            
            # Step 6: Deploy if better
            if should_deploy:
                self.deploy_model(model, metrics)
                mlflow.log_param("deployed", True)
                mlflow.log_artifact(str(self.model_dir / 'production' / 'model.pkl'))
                logger.info("✅ New model deployed to production!")
            else:
                mlflow.log_param("deployed", False)
                logger.info("❌ New model did not improve. Keeping current production model.")
        
        logger.info("=" * 60)
        logger.info("RETRAINING PIPELINE COMPLETE")
        logger.info("=" * 60)


# --- Entry point ---
if __name__ == '__main__':
    import argparse
    
    parser = argparse.ArgumentParser(description='Automated Model Retraining')
    parser.add_argument('--config', type=str, default='configs/retrain_config.json',
                       help='Path to retraining configuration')
    parser.add_argument('--force', action='store_true',
                       help='Force retraining regardless of triggers')
    
    args = parser.parse_args()
    
    pipeline = AutomatedRetrainingPipeline(args.config)
    pipeline.run()
```

### Retraining Configuration

```json
{
    "model_dir": "models/",
    "data_dir": "data/",
    "target_column": "loan_approved",
    "primary_metric": "f1_score",
    "min_improvement": 0.005,
    "drift_threshold": 0.05,
    "retrain_schedule_days": 7,
    "min_new_samples": 1000,
    "mlflow_uri": "http://mlflow-server:5000",
    "experiment_name": "loan-approval-model",
    "notification": {
        "slack_webhook": "https://hooks.slack.com/...",
        "email": "ml-team@company.com"
    }
}
```

---

## ML Pipeline Orchestration

### Comparison of Orchestration Tools

| Tool | Best For | Complexity | Cloud Native | ML-Specific |
|------|----------|-----------|--------------|-------------|
| **GitHub Actions** | Simple CI/CD | Low | Yes | No |
| **Apache Airflow** | Complex DAGs, scheduling | Medium | Partial | No |
| **Kubeflow Pipelines** | K8s-native ML | High | Yes | Yes |
| **Prefect** | Modern Python pipelines | Low-Medium | Yes | No |
| **ZenML** | MLOps abstraction | Low | Yes | Yes |
| **Metaflow** | Data science workflows | Low | Yes | Yes |

### Prefect Example (Modern Alternative to Airflow)

```python
# src/pipelines/training_pipeline.py
from prefect import flow, task
from prefect.tasks import task_input_hash
from datetime import timedelta
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
import joblib

@task(
    retries=3,                    # Retry on failure
    retry_delay_seconds=60,       # Wait 1 min between retries
    cache_key_fn=task_input_hash, # Cache results
    cache_expiration=timedelta(hours=1)
)
def load_data(data_path: str) -> pd.DataFrame:
    """Load and return raw data."""
    df = pd.read_csv(data_path)
    print(f"Loaded {len(df)} rows from {data_path}")
    return df

@task
def validate_data(df: pd.DataFrame) -> pd.DataFrame:
    """Validate data quality before training."""
    # Check for required columns
    required_cols = ['age', 'income', 'credit_score', 'target']
    missing = [c for c in required_cols if c not in df.columns]
    if missing:
        raise ValueError(f"Missing columns: {missing}")
    
    # Check for excessive nulls
    null_pct = df.isnull().mean()
    bad_cols = null_pct[null_pct > 0.3].index.tolist()
    if bad_cols:
        raise ValueError(f"Columns with >30% nulls: {bad_cols}")
    
    # Drop rows with any nulls (simple strategy)
    df_clean = df.dropna()
    print(f"Data validated. {len(df_clean)} clean rows (dropped {len(df) - len(df_clean)})")
    return df_clean

@task
def feature_engineering(df: pd.DataFrame) -> tuple:
    """Create features and split data."""
    from sklearn.model_selection import train_test_split
    
    # Create features
    df['income_per_year_age'] = df['income'] / (df['age'] + 1)
    df['credit_income_ratio'] = df['credit_score'] / (df['income'] / 10000)
    
    # Split
    X = df.drop('target', axis=1)
    y = df['target']
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42, stratify=y
    )
    
    return X_train, X_test, y_train, y_test

@task(log_prints=True)
def train_model(X_train, y_train, n_estimators: int = 100):
    """Train a Random Forest model."""
    model = RandomForestClassifier(
        n_estimators=n_estimators,
        max_depth=10,
        random_state=42,
        n_jobs=-1
    )
    model.fit(X_train, y_train)
    print(f"Model trained with {n_estimators} estimators")
    return model

@task
def evaluate_model(model, X_test, y_test) -> dict:
    """Evaluate model performance."""
    from sklearn.metrics import classification_report
    
    y_pred = model.predict(X_test)
    accuracy = accuracy_score(y_test, y_pred)
    
    report = classification_report(y_test, y_pred, output_dict=True)
    
    metrics = {
        'accuracy': accuracy,
        'f1_weighted': report['weighted avg']['f1-score'],
        'precision': report['weighted avg']['precision'],
        'recall': report['weighted avg']['recall']
    }
    
    print(f"Model Accuracy: {accuracy:.4f}")
    return metrics

@task
def save_model(model, metrics: dict, output_path: str):
    """Save model and metrics."""
    import json
    from pathlib import Path
    
    output = Path(output_path)
    output.mkdir(parents=True, exist_ok=True)
    
    joblib.dump(model, output / 'model.pkl')
    
    with open(output / 'metrics.json', 'w') as f:
        json.dump(metrics, f, indent=2)
    
    print(f"Model saved to {output_path}")

# --- Main Pipeline Flow ---

@flow(name="ML Training Pipeline", log_prints=True)
def training_pipeline(
    data_path: str = "data/train.csv",
    output_path: str = "models/latest",
    n_estimators: int = 200
):
    """
    Complete ML training pipeline.
    
    Orchestrates: data loading → validation → feature engineering
    → training → evaluation → saving.
    """
    # Step 1: Load data
    raw_data = load_data(data_path)
    
    # Step 2: Validate
    clean_data = validate_data(raw_data)
    
    # Step 3: Feature engineering
    X_train, X_test, y_train, y_test = feature_engineering(clean_data)
    
    # Step 4: Train
    model = train_model(X_train, y_train, n_estimators=n_estimators)
    
    # Step 5: Evaluate
    metrics = evaluate_model(model, X_test, y_test)
    
    # Step 6: Save if performance is good
    if metrics['accuracy'] >= 0.80:
        save_model(model, metrics, output_path)
        print("✅ Pipeline complete - model deployed!")
    else:
        print(f"❌ Model accuracy {metrics['accuracy']:.4f} below threshold. Not saving.")
    
    return metrics


# Run the pipeline
if __name__ == "__main__":
    training_pipeline()
```

---

## Common Mistakes

### 1. Not Testing Data, Only Code
```python
# ❌ WRONG: Only testing code logic
def test_model_trains():
    model.fit(X, y)  # This will pass even with garbage data!

# ✅ RIGHT: Test data quality BEFORE training
def test_data_quality():
    assert df['age'].between(0, 120).all()
    assert df.duplicated().sum() == 0
    assert df['target'].nunique() == 2
```

### 2. No Performance Gates
```python
# ❌ WRONG: Deploy any model that trains successfully
if model_trained:
    deploy(model)

# ✅ RIGHT: Only deploy if model meets minimum thresholds
if (metrics['accuracy'] >= 0.85 and 
    metrics['f1'] >= 0.80 and
    metrics['latency_p95'] < 100):
    deploy(model)
```

### 3. Ignoring Model Reproducibility
```python
# ❌ WRONG: No seed, no version tracking
model = RandomForestClassifier()
model.fit(X_train, y_train)

# ✅ RIGHT: Pin everything for reproducibility
model = RandomForestClassifier(
    n_estimators=100,
    random_state=42  # Fixed seed
)
# Also: pin library versions in requirements.txt
# Also: version your training data with DVC
```

### 4. Testing Only on Batch Data, Not Live Traffic Patterns
```python
# ❌ WRONG: Only test with clean batch data
def test_model():
    X_test = load_clean_test_set()
    assert model.predict(X_test) is not None

# ✅ RIGHT: Test with realistic edge cases
def test_model_edge_cases():
    # Single sample (not batch)
    assert model.predict([[25, 50000, 700]]) is not None
    
    # Extreme values
    assert model.predict([[120, 0, 300]]) is not None
    
    # Missing features (if your API allows it)
    with pytest.raises(ValueError):
        model.predict([[None, 50000, 700]])
```

### 5. No Rollback Strategy
```python
# ❌ WRONG: Overwrite production model
joblib.dump(new_model, 'production/model.pkl')

# ✅ RIGHT: Version and archive
import shutil
from datetime import datetime

# Archive current production
timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
shutil.copy('production/model.pkl', f'archive/model_{timestamp}.pkl')

# Deploy new model
joblib.dump(new_model, 'production/model.pkl')

# Save rollback metadata
with open('production/rollback_info.json', 'w') as f:
    json.dump({'previous_version': timestamp, 'deployed_at': datetime.now().isoformat()}, f)
```

---

## Interview Questions

### Conceptual Questions

**Q1: What is the difference between CI, CD, and CT in the ML context?**
> **A:** CI (Continuous Integration) automatically tests code changes — unit tests, data validation, linting. CD (Continuous Delivery/Deployment) automatically packages and deploys validated models to production. CT (Continuous Training) is unique to ML — it automatically retrains models when triggered by new data, performance degradation, or schedule. Together they form the ML automation pyramid.

**Q2: How would you design a retraining trigger system?**
> **A:** Multiple triggers working together:
> - **Schedule-based:** Weekly/monthly retraining regardless of performance
> - **Performance-based:** Monitor live metrics, retrain if accuracy drops >X%
> - **Data-drift-based:** Statistical tests (KS test, PSI) detect distribution shifts
> - **Volume-based:** Retrain after N new labeled samples accumulate
> - **Manual:** Data scientists can force retraining for experiments

**Q3: What tests should pass before deploying a new ML model?**
> **A:** The testing pyramid: (1) Unit tests for preprocessing/feature code, (2) Data quality tests — schema, ranges, missing values, (3) Model performance tests — accuracy/F1 above thresholds, (4) No regression vs current production model, (5) Latency tests — prediction speed meets SLA, (6) Fairness tests — no demographic bias, (7) Integration tests — end-to-end prediction works, (8) Canary test in staging environment.

**Q4: How do you handle model rollback in production?**
> **A:** Keep versioned model artifacts (with their data context). Use blue-green deployment: new model serves alongside old one. If monitoring detects issues (error rate spike, latency increase, drift), automatically route traffic back to previous version. Store rollback metadata linking each model to its training data, config, and git commit.

**Q5: What's the difference between testing ML code vs testing ML models?**
> **A:** Code testing verifies deterministic behavior (given input X, function returns Y). Model testing verifies statistical properties (accuracy > 85% on test set, latency < 100ms, predictions within valid range). Code tests are pass/fail; model tests use thresholds and can be flaky if not designed carefully.

### Scenario-Based Questions

**Q6: Your production model's accuracy dropped 3% overnight. Walk me through your debugging process.**
> **A:** (1) Check if it's a data issue — did input distribution change? Run drift detection. (2) Check if it's an infrastructure issue — is the correct model version deployed? Check model registry. (3) Check data pipeline — are features being computed correctly? Any upstream schema changes? (4) If data drift confirmed, trigger retraining with recent data. (5) Meanwhile, consider rollback to previous model if drop is severe. (6) Add monitoring alert for this scenario to catch it faster next time.

**Q7: How would you set up CI/CD for a team of 5 ML engineers working on the same model?**
> **A:** (1) Git branching strategy: feature branches → PR to develop → PR to main. (2) Every PR triggers: lint, unit tests, data tests, small-scale training. (3) Merge to develop triggers: full training, model comparison, staging deployment. (4) Merge to main (after review): production deployment with canary. (5) Experiment tracking: MLflow/W&B so everyone sees each other's results. (6) Model registry for version control. (7) Automated PR comments showing metric diffs.

---

## Quick Reference

| Component | Tool/Practice | Purpose |
|-----------|--------------|---------|
| **Code Tests** | pytest, unittest | Validate preprocessing, features, utils |
| **Data Tests** | Great Expectations, custom pytest | Schema, ranges, drift detection |
| **Model Tests** | Custom pytest + thresholds | Performance gates, fairness, latency |
| **CI Platform** | GitHub Actions, GitLab CI | Orchestrate test/build/deploy |
| **Pipeline** | Prefect, Airflow, Kubeflow | Orchestrate ML workflows |
| **Retraining** | Scheduled + drift-triggered | Keep model fresh |
| **Deployment** | Canary, Blue-Green, Shadow | Safe model rollout |
| **Rollback** | Versioned artifacts + routing | Quick recovery from bad deploys |

### CI/CD Pipeline Checklist

```
□ Code linting and formatting (flake8, black)
□ Type checking (mypy)
□ Unit tests for preprocessing/features
□ Data validation tests
□ Model training (on CI)
□ Performance threshold tests
□ Regression test vs production model
□ Latency/throughput tests
□ Fairness/bias tests
□ Container build
□ Staging deployment + smoke tests
□ Canary deployment to production
□ Monitoring setup for new model
□ Rollback plan documented
```

### Key Formulas

**Deployment Decision Rule:**

$$\text{Deploy} = \begin{cases} \text{True} & \text{if } M_{new} - M_{prod} \geq \epsilon \text{ AND all tests pass} \\ \text{False} & \text{otherwise} \end{cases}$$

Where $M$ is the primary metric and $\epsilon$ is the minimum improvement threshold.

**Canary Success Criteria:**

$$\text{Canary Pass} = (E_{canary} \leq E_{baseline} + \delta) \text{ AND } (L_{p95} \leq L_{threshold})$$

Where $E$ is error rate, $\delta$ is tolerance, and $L_{p95}$ is 95th percentile latency.

---

> **Pro Tip:** Start with simple CI (lint + unit tests + model threshold check) and gradually add complexity. A basic pipeline running consistently is better than a perfect pipeline that nobody maintains.
