# Chapter 01: What is Data Science

## Table of Contents
- [1.1 What is Data Science](#11-what-is-data-science)
- [1.2 The Data Science Lifecycle](#12-the-data-science-lifecycle)
- [1.3 Roles in Data Science](#13-roles-in-data-science)
- [1.4 Tools and Ecosystem](#14-tools-and-ecosystem)
- [1.5 Data Science vs Related Fields](#15-data-science-vs-related-fields)
- [1.6 Types of Data](#16-types-of-data)
- [1.7 Data Science in Practice](#17-data-science-in-practice)
- [1.8 Common Mistakes](#18-common-mistakes)
- [1.9 Interview Questions](#19-interview-questions)
- [1.10 Quick Reference](#110-quick-reference)

---

## 1.1 What is Data Science

### What It Is
Data Science is the art and science of extracting meaningful insights from data. Think of it like being a detective — but instead of solving crimes, you're solving business problems using clues hidden in data.

**Simple analogy:** Imagine you run a lemonade stand. You notice you sell more on hot days. That's a data insight. Now imagine you have 10 years of weather data, sales data, foot traffic data, and competitor pricing — and you use math + programming to predict exactly how much lemonade to make tomorrow. That's Data Science.

### Why It Matters
- **Every company generates data** — those who use it well, win
- Netflix saves **$1 billion/year** through its recommendation algorithm
- Amazon generates **35% of revenue** from its recommendation engine
- Healthcare uses data science to predict disease outbreaks before they happen
- Self-driving cars process millions of data points per second

### The Three Pillars

```
                    ┌─────────────────┐
                    │   DATA SCIENCE  │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
    ┌───────▼───────┐ ┌─────▼──────┐ ┌──────▼───────┐
    │  Mathematics  │ │  Domain    │ │  Computer    │
    │  & Statistics │ │  Expertise │ │  Science     │
    └───────────────┘ └────────────┘ └──────────────┘
    - Probability      - Business     - Programming
    - Linear Algebra     Understanding - Databases
    - Calculus         - Industry     - Big Data
    - Optimization       Knowledge    - ML/AI
```

### The Data Science Venn Diagram

```
         ┌──────────────────────┐
         │     Math/Stats       │
         │                      │
    ┌────┼──────┐               │
    │    │ ML   │  Traditional  │
    │    │Engine│  Research     │
    │    │ering │               │
    │ CS ├──────┼───────┐       │
    │    │ DATA │       │       │
    │    │SCIEN.│       │       │
    │    ├──────┼───────┤       │
    │    │      │ Domain│       │
    └────┼──────┤Expert.│───────┘
         │      │       │
         └──────┴───────┘
```

---

## 1.2 The Data Science Lifecycle

### How It Works

The Data Science process is **iterative**, not linear. You'll go back and forth between steps.

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  1.      │    │  2.      │    │  3.      │    │  4.      │    │  5.      │    │  6.      │
│  Problem │───▶│  Data    │───▶│  Data    │───▶│  EDA &   │───▶│  Model   │───▶│  Deploy  │
│  Define  │    │  Collect │    │  Clean   │    │  Feature │    │  Build & │    │  &       │
│          │◀───│          │◀───│          │◀───│  Eng.    │◀───│  Eval    │    │  Monitor │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
     5%              10%             40%              20%             15%             10%
   (time)          (time)          (time)          (time)          (time)          (time)
```

> **Important:** Data cleaning takes ~40% of a data scientist's time. If someone tells you data science is all about building fancy models, they've never done it professionally.

### Step-by-Step Breakdown

| Step | What Happens | Key Question | Output |
|------|-------------|--------------|--------|
| 1. Problem Definition | Understand what business needs | "What decision will this inform?" | Clear problem statement |
| 2. Data Collection | Gather relevant data | "What data do I need?" | Raw datasets |
| 3. Data Cleaning | Fix errors, handle missing values | "Is this data trustworthy?" | Clean dataset |
| 4. EDA & Feature Eng. | Explore patterns, create features | "What story does the data tell?" | Insights + features |
| 5. Modeling | Build & evaluate ML models | "Can we predict/classify this?" | Trained model |
| 6. Deployment | Put model into production | "How do we use this at scale?" | Live system |

### Code Example: Mini Data Science Project

```python
# A complete mini data science lifecycle in one script
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error, r2_score

# Step 1: Problem Definition
# "Can we predict house prices based on features like size, bedrooms, etc.?"

# Step 2: Data Collection (using a simple synthetic dataset)
np.random.seed(42)
n_houses = 1000

data = pd.DataFrame({
    'size_sqft': np.random.randint(500, 5000, n_houses),        # House size
    'bedrooms': np.random.randint(1, 7, n_houses),              # Number of bedrooms
    'age_years': np.random.randint(0, 50, n_houses),            # Age of house
    'distance_city_center': np.random.uniform(0.5, 30, n_houses) # Distance in km
})

# Generate price with known relationship + noise
data['price'] = (
    data['size_sqft'] * 200 +               # $200 per sqft
    data['bedrooms'] * 15000 +              # $15k per bedroom
    data['age_years'] * (-2000) +           # Older = cheaper
    data['distance_city_center'] * (-5000) + # Far from center = cheaper
    np.random.normal(0, 50000, n_houses)    # Random noise
)

# Step 3: Data Cleaning
print("=== Step 3: Data Cleaning ===")
print(f"Shape: {data.shape}")
print(f"Missing values:\n{data.isnull().sum()}")
print(f"Data types:\n{data.dtypes}")

# Step 4: Exploratory Data Analysis
print("\n=== Step 4: EDA ===")
print(f"Price Statistics:\n{data['price'].describe()}")
print(f"\nCorrelation with Price:")
print(data.corr()['price'].sort_values(ascending=False))

# Step 5: Model Building
X = data[['size_sqft', 'bedrooms', 'age_years', 'distance_city_center']]
y = data['price']

# Split data: 80% train, 20% test
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Train model
model = LinearRegression()
model.fit(X_train, y_train)

# Evaluate
y_pred = model.predict(X_test)
print(f"\n=== Step 5: Model Evaluation ===")
print(f"R² Score: {r2_score(y_test, y_pred):.4f}")
print(f"RMSE: ${mean_squared_error(y_test, y_pred, squared=False):,.0f}")
print(f"\nModel Coefficients:")
for feature, coef in zip(X.columns, model.coef_):
    print(f"  {feature}: ${coef:,.0f}")

# Step 6: Make predictions (deployment simulation)
new_house = pd.DataFrame({
    'size_sqft': [2000],
    'bedrooms': [3],
    'age_years': [10],
    'distance_city_center': [5.0]
})
predicted_price = model.predict(new_house)[0]
print(f"\n=== Step 6: Prediction ===")
print(f"Predicted price for new house: ${predicted_price:,.0f}")
```

**Expected Output:**
```
=== Step 3: Data Cleaning ===
Shape: (1000, 5)
Missing values: all zeros
Data types: int64 and float64

=== Step 4: EDA ===
Correlation with Price:
price                   1.000
size_sqft               0.932
bedrooms                0.186
age_years              -0.152
distance_city_center   -0.275

=== Step 5: Model Evaluation ===
R² Score: 0.9432
RMSE: $51,234

=== Step 6: Prediction ===
Predicted price for new house: $386,500
```

---

## 1.3 Roles in Data Science

### The Ecosystem of Roles

```
┌────────────────────────────────────────────────────────────────────────┐
│                        DATA TEAM STRUCTURE                              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Data Engineer        Data Analyst        Data Scientist    ML Engineer│
│  ┌──────────┐        ┌──────────┐        ┌──────────┐    ┌──────────┐│
│  │Build     │──data─▶│Analyze   │──ask───▶│Model &   │───▶│Deploy &  ││
│  │Pipelines │  flow  │& Report  │ deeper  │Predict   │    │Scale     ││
│  └──────────┘        └──────────┘        └──────────┘    └──────────┘│
│       │                    │                   │               │       │
│  SQL, Spark,         Excel, SQL,          Python, R,      Docker,     │
│  Airflow, Kafka      Tableau, Power BI    Scikit-learn    Kubernetes  │
│                                           TensorFlow      MLflow      │
└────────────────────────────────────────────────────────────────────────┘
```

### Detailed Role Comparison

| Aspect | Data Analyst | Data Scientist | Data Engineer | ML Engineer |
|--------|-------------|----------------|---------------|-------------|
| **Focus** | What happened? | What will happen? | How does data flow? | How to serve models? |
| **Skills** | SQL, Excel, BI Tools | Python, Stats, ML | SQL, Spark, Cloud | ML + Software Eng |
| **Output** | Reports, Dashboards | Models, Insights | Pipelines, Warehouses | Production Systems |
| **Education** | Bachelor's often enough | Master's/PhD common | CS/Engineering degree | CS + ML background |
| **Salary (US)** | $60K-$100K | $90K-$160K | $100K-$170K | $120K-$200K |
| **Day-to-day** | Queries, reports, meetings | Experiments, coding, research | Building infrastructure | MLOps, optimization |

### Other Roles

- **Research Scientist** — Publishes papers, pushes boundaries of AI/ML
- **Analytics Engineer** — Bridge between data engineer and analyst (dbt, data modeling)
- **Decision Scientist** — Focuses on causal inference and experiment design
- **AI/LLM Engineer** — Specializes in GenAI applications, prompt engineering, RAG systems

---

## 1.4 Tools and Ecosystem

### The Data Science Tech Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    DATA SCIENCE TOOLS                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  LANGUAGES        │  LIBRARIES          │  VISUALIZATION    │
│  ─────────        │  ─────────          │  ─────────────    │
│  • Python ★       │  • NumPy            │  • Matplotlib     │
│  • R              │  • Pandas ★         │  • Seaborn        │
│  • SQL ★          │  • Scikit-learn ★   │  • Plotly         │
│  • Julia          │  • TensorFlow       │  • Tableau        │
│                   │  • PyTorch          │  • Power BI       │
│                   │  • Statsmodels      │  • D3.js          │
│                   │                     │                   │
│  DATABASES        │  BIG DATA           │  CLOUD            │
│  ─────────        │  ────────           │  ─────            │
│  • PostgreSQL     │  • Apache Spark     │  • AWS (S3,       │
│  • MongoDB        │  • Hadoop           │    SageMaker)     │
│  • Redis          │  • Kafka            │  • GCP (BigQuery, │
│  • Snowflake      │  • Dask             │    Vertex AI)     │
│  • BigQuery       │  • Ray              │  • Azure ML       │
│                   │                     │                   │
│  DEV TOOLS        │  MLOPS              │  NOTEBOOKS        │
│  ─────────        │  ─────              │  ─────────        │
│  • Git            │  • MLflow           │  • Jupyter ★      │
│  • Docker         │  • DVC              │  • Google Colab   │
│  • VS Code        │  • Weights & Biases │  • Kaggle         │
│  • Linux CLI      │  • Kubeflow         │  • Databricks     │
└─────────────────────────────────────────────────────────────┘
                        ★ = Must-know
```

### Python Setup for Data Science

```python
# Essential imports for any data science project
# The "starter pack" you'll use in almost every project

# Data manipulation
import numpy as np          # Numerical computing (arrays, math)
import pandas as pd         # Data manipulation (DataFrames, like Excel on steroids)

# Visualization
import matplotlib.pyplot as plt  # Basic plotting
import seaborn as sns            # Statistical visualization (prettier matplotlib)

# Machine Learning
from sklearn.model_selection import train_test_split  # Split data
from sklearn.preprocessing import StandardScaler       # Scale features
from sklearn.linear_model import LogisticRegression   # Classification
from sklearn.ensemble import RandomForestClassifier    # Ensemble methods
from sklearn.metrics import accuracy_score, classification_report  # Evaluation

# Statistics
from scipy import stats          # Statistical tests
import statsmodels.api as sm     # Statistical modeling

# Utilities
import warnings
warnings.filterwarnings('ignore')  # Suppress warnings in notebooks

# Display settings
pd.set_option('display.max_columns', None)   # Show all columns
pd.set_option('display.max_rows', 100)       # Show up to 100 rows
pd.set_option('display.float_format', '{:.2f}'.format)  # 2 decimal places

print("All imports successful! You're ready for data science.")
```

### When to Use What Tool

| Task | Best Tool | Why |
|------|-----------|-----|
| Quick data exploration | Pandas + Jupyter | Interactive, fast iteration |
| Large datasets (>1GB) | PySpark or Dask | Distributed computing |
| Statistical modeling | Statsmodels | p-values, confidence intervals |
| ML modeling | Scikit-learn | Simple API, great docs |
| Deep Learning | PyTorch / TensorFlow | GPU support, flexibility |
| Dashboard | Streamlit / Plotly Dash | Quick deployment |
| Production pipeline | Airflow + Docker | Scheduling, reproducibility |
| Experiment tracking | MLflow / W&B | Compare runs, versioning |

---

## 1.5 Data Science vs Related Fields

### Comparison Table

| Dimension | Data Science | Machine Learning | AI | Statistics | Business Analytics |
|-----------|-------------|-----------------|-----|-----------|-------------------|
| **Goal** | Extract insights | Build predictive models | Create intelligent systems | Quantify uncertainty | Drive business decisions |
| **Approach** | End-to-end | Algorithm-focused | System-focused | Theory-focused | Report-focused |
| **Math Needed** | Medium-High | High | Very High | Very High | Medium |
| **Coding** | Medium-High | High | High | Low-Medium | Low |
| **Domain Knowledge** | Critical | Helpful | Varies | Helpful | Critical |
| **Typical Output** | Insights + Models | Trained Models | Intelligent Products | Papers + Tests | Dashboards + Reports |

### Visual Relationship

```
┌──────────────────────────────────────────────────────────┐
│                 ARTIFICIAL INTELLIGENCE                    │
│  ┌──────────────────────────────────────────────────┐    │
│  │              MACHINE LEARNING                     │    │
│  │  ┌──────────────────────────────────────────┐    │    │
│  │  │           DEEP LEARNING                   │    │    │
│  │  │  ┌──────────────────────────────────┐     │    │    │
│  │  │  │       GenAI / LLMs               │     │    │    │
│  │  │  └──────────────────────────────────┘     │    │    │
│  │  └──────────────────────────────────────────┘    │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  DATA SCIENCE intersects with all of the above +          │
│  Statistics + Domain Expertise + Communication            │
└──────────────────────────────────────────────────────────┘
```

---

## 1.6 Types of Data

### Data Classification

```
                         DATA
                          │
            ┌─────────────┼─────────────┐
            │                           │
      STRUCTURED                  UNSTRUCTURED
      (Tables, SQL)               (Text, Images, Audio)
            │                           │
    ┌───────┼───────┐          ┌────────┼────────┐
    │               │          │                  │
 NUMERICAL      CATEGORICAL   TEXT            MULTIMEDIA
    │               │          │                  │
 ┌──┴──┐       ┌──┴──┐    • Emails          • Images
 │     │       │     │    • Reviews          • Video
Cont. Disc.  Nom.  Ord.   • Documents        • Audio
                           • Social media     • Sensor data
```

| Type | Subtype | Example | Operations |
|------|---------|---------|-----------|
| **Numerical** | Continuous | Temperature (36.6°C) | Mean, std, regression |
| **Numerical** | Discrete | Number of children (0, 1, 2) | Count, mode |
| **Categorical** | Nominal | Color (Red, Blue, Green) | Mode, frequency |
| **Categorical** | Ordinal | Rating (Low, Medium, High) | Median, rank |
| **Text** | Unstructured | Customer reviews | NLP, sentiment analysis |
| **Time Series** | Sequential | Stock prices over time | Trend, seasonality |

### Code: Identifying Data Types

```python
import pandas as pd
import numpy as np

# Create a sample dataset with mixed types
df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve'],          # Categorical (Nominal)
    'age': [25, 30, 35, 28, 42],                                   # Numerical (Discrete)
    'salary': [50000.50, 75000.00, 60000.75, 82000.25, 95000.00], # Numerical (Continuous)
    'department': ['Engineering', 'Sales', 'Engineering', 'HR', 'Sales'],  # Categorical (Nominal)
    'satisfaction': ['High', 'Medium', 'Low', 'High', 'Medium'],   # Categorical (Ordinal)
    'join_date': pd.to_datetime(['2020-01-15', '2019-06-01', 
                                  '2021-03-20', '2018-11-10', '2022-07-01'])  # DateTime
})

# Check data types
print("=== Data Types ===")
print(df.dtypes)
print(f"\n=== Shape: {df.shape} ===")
print(f"Rows: {df.shape[0]}, Columns: {df.shape[1]}")

# Identify numerical vs categorical
numerical_cols = df.select_dtypes(include=[np.number]).columns.tolist()
categorical_cols = df.select_dtypes(include=['object']).columns.tolist()
datetime_cols = df.select_dtypes(include=['datetime64']).columns.tolist()

print(f"\nNumerical columns: {numerical_cols}")
print(f"Categorical columns: {categorical_cols}")
print(f"DateTime columns: {datetime_cols}")

# Quick stats for numerical
print("\n=== Numerical Summary ===")
print(df[numerical_cols].describe())

# Quick stats for categorical
print("\n=== Categorical Summary ===")
for col in categorical_cols:
    print(f"\n{col}:")
    print(f"  Unique values: {df[col].nunique()}")
    print(f"  Most common: {df[col].mode()[0]}")
    print(f"  Value counts:\n{df[col].value_counts().to_string()}")
```

---

## 1.7 Data Science in Practice

### Industry Applications

| Industry | Use Case | Data Used | Impact |
|----------|----------|-----------|--------|
| **E-commerce** | Recommendation engines | Purchase history, browsing | +35% revenue (Amazon) |
| **Healthcare** | Disease prediction | Patient records, genomics | Early detection saves lives |
| **Finance** | Fraud detection | Transaction patterns | Prevents billions in fraud |
| **Transportation** | Route optimization | GPS, traffic, weather | Saves fuel & time |
| **Marketing** | Customer segmentation | Demographics, behavior | Targeted campaigns |
| **Manufacturing** | Predictive maintenance | Sensor data, logs | Reduces downtime 30-50% |
| **Sports** | Player performance | Game stats, biometrics | Better drafting decisions |

### A Day in the Life of a Data Scientist

```
09:00 - Check model performance dashboards, review overnight alerts
09:30 - Team standup: discuss progress on customer churn project
10:00 - Data exploration: write SQL queries to understand new dataset
11:00 - Feature engineering: create new variables from raw data
12:00 - Lunch (read a paper on new embedding techniques)
13:00 - Model training: experiment with different algorithms
14:30 - Meeting with product team: translate results into business language
15:30 - Write documentation for model decisions
16:00 - Code review: review teammate's preprocessing pipeline
17:00 - Update experiment tracking (MLflow), plan tomorrow's work
```

### Pro Tips from Industry

> **Tip 1:** The best model is useless if stakeholders don't understand or trust it. Communication skills matter as much as technical skills.

> **Tip 2:** Start simple. A logistic regression that ships next week beats a transformer model that takes 3 months to deploy.

> **Tip 3:** Always ask "So what?" after finding an insight. If it doesn't lead to an action, it's not useful.

> **Tip 4:** Version everything — your data, your code, your models, your experiments.

---

## 1.8 Common Mistakes

### Mistakes Beginners Make

| # | Mistake | Why It's Wrong | What To Do Instead |
|---|---------|---------------|-------------------|
| 1 | Jumping to modeling immediately | You'll build on bad data | Spend 60%+ of time on data understanding & cleaning |
| 2 | Not defining the problem clearly | "Improve things" isn't a goal | Write a specific, measurable problem statement |
| 3 | Ignoring data leakage | Model looks great, fails in production | Ensure test data is truly unseen (no future info) |
| 4 | Using accuracy as the only metric | Misleading with imbalanced data | Use precision, recall, F1, AUC-ROC as appropriate |
| 5 | Not checking for bias | Model discriminates against groups | Audit model fairness across demographics |
| 6 | Overfitting to training data | Memorizes noise, fails on new data | Use cross-validation, regularization |
| 7 | Ignoring domain expertise | Statistically significant ≠ meaningful | Always validate findings with domain experts |
| 8 | Not considering deployment | Model only works in notebooks | Think about production from day 1 |

---

## 1.9 Interview Questions

### Conceptual Questions

**Q1: What is Data Science? How is it different from Machine Learning?**
> Data Science is a broader field that encompasses the entire process of extracting value from data — from collecting and cleaning data to analyzing it and communicating results. Machine Learning is a subset technique within data science focused specifically on building predictive models that learn from data.

**Q2: Explain the Data Science lifecycle.**
> Problem Definition → Data Collection → Data Cleaning → EDA → Feature Engineering → Modeling → Evaluation → Deployment → Monitoring. The process is iterative — you often go back to earlier steps based on findings.

**Q3: What's the difference between supervised and unsupervised learning?**
> Supervised: You have labeled data (input → known output). Example: predicting house prices.
> Unsupervised: No labels; find hidden patterns. Example: customer segmentation.

**Q4: How would you handle a situation where your model's accuracy is 99% on a fraud detection task?**
> This is likely misleading because fraud is rare (~0.1% of transactions). A model that always predicts "not fraud" would be 99.9% accurate. I'd look at precision, recall, and the confusion matrix. For fraud detection, recall (catching actual frauds) is typically more important than precision.

**Q5: What would you do if you were given a dataset and asked to "find something interesting"?**
> 1) Understand what the data represents and its business context. 2) Check data quality (missing values, duplicates, outliers). 3) Compute summary statistics. 4) Visualize distributions and relationships. 5) Look for correlations and anomalies. 6) Form hypotheses and test them. 7) Present 2-3 actionable findings with supporting evidence.

**Q6: How do you explain a complex model to a non-technical stakeholder?**
> Use analogies, visual aids, and focus on business impact rather than technical details. For example: "The model is like a very experienced loan officer who has reviewed 100,000 applications and learned which patterns predict default. It considers income, credit history, and 20 other factors to give each applicant a risk score."

---

## 1.10 Quick Reference

### Data Science Cheat Sheet

| Concept | Key Point |
|---------|-----------|
| Data Science | Extracting insights from data using math + programming + domain knowledge |
| Lifecycle | Problem → Collect → Clean → Explore → Model → Deploy → Monitor |
| Most time spent on | Data cleaning & preparation (~40%) |
| Key language | Python (most popular), R (statistics), SQL (data access) |
| Key libraries | Pandas, NumPy, Scikit-learn, Matplotlib |
| Structured data | Tables with rows & columns (CSV, SQL databases) |
| Unstructured data | Text, images, audio, video |
| Supervised learning | Has labeled target variable (prediction) |
| Unsupervised learning | No labels (clustering, pattern finding) |
| Model evaluation | Never evaluate on training data; use train/test split |
| Communication | Translate technical findings into business value |

### Essential Formulas for Data Science

| Formula | Use |
|---------|-----|
| $\bar{x} = \frac{1}{n}\sum_{i=1}^{n} x_i$ | Mean (average) |
| $\sigma = \sqrt{\frac{1}{n}\sum_{i=1}^{n}(x_i - \bar{x})^2}$ | Standard Deviation |
| $r = \frac{\sum(x_i - \bar{x})(y_i - \bar{y})}{\sqrt{\sum(x_i-\bar{x})^2 \sum(y_i-\bar{y})^2}}$ | Pearson Correlation |
| $R^2 = 1 - \frac{SS_{res}}{SS_{tot}}$ | Model fit quality |

---

**Next Chapter:** [02-Statistics-Fundamentals](./02-Statistics-Fundamentals.md) →
