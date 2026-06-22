# Chapter 04: Data Collection and Cleaning

## Table of Contents
- [4.1 Data Sources and Collection Methods](#41-data-sources-and-collection-methods)
- [4.2 Web Scraping](#42-web-scraping)
- [4.3 Working with APIs](#43-working-with-apis)
- [4.4 Handling Missing Values](#44-handling-missing-values)
- [4.5 Outlier Detection and Treatment](#45-outlier-detection-and-treatment)
- [4.6 Data Type Conversions and Consistency](#46-data-type-conversions-and-consistency)
- [4.7 Duplicate Detection and Removal](#47-duplicate-detection-and-removal)
- [4.8 Data Validation and Quality Checks](#48-data-validation-and-quality-checks)
- [4.9 Complete Data Cleaning Pipeline](#49-complete-data-cleaning-pipeline)

---

## 4.1 Data Sources and Collection Methods

### What It Is
Data collection is the process of gathering raw information from various sources — databases, files, APIs, websites, sensors, surveys — that you'll later analyze to extract insights. Think of it as grocery shopping before cooking: you need the right ingredients (data) before you can make a meal (analysis).

### Why It Matters
- **Garbage in, garbage out**: Your model/analysis is only as good as your data
- **80% of a data scientist's time** is spent on data collection and cleaning
- Choosing the right source determines the quality, freshness, and legality of your analysis
- Understanding data sources helps you assess biases and limitations

### Types of Data Sources

| Source Type | Examples | Pros | Cons |
|-------------|----------|------|------|
| **Databases** | PostgreSQL, MySQL, MongoDB | Structured, fast, reliable | Need access permissions |
| **Files** | CSV, Excel, JSON, Parquet | Easy to share, portable | Can be messy, size limits |
| **APIs** | Twitter API, Weather API | Real-time, structured | Rate limits, may cost money |
| **Web Scraping** | Any website | Huge data availability | Legal issues, fragile |
| **Surveys** | Google Forms, Typeform | Custom data, specific questions | Response bias, small sample |
| **Sensors/IoT** | Temperature, GPS, accelerometer | Real-time, continuous | Noisy, storage intensive |
| **Public Datasets** | Kaggle, UCI, government portals | Free, documented | May be outdated |

### Code Examples — Reading Data from Various Sources

```python
import pandas as pd
import sqlite3
import json

# ============================================
# 1. Reading from CSV files
# ============================================
# Basic CSV read
df_csv = pd.read_csv('data.csv')

# CSV with custom settings
df_csv = pd.read_csv(
    'data.csv',
    sep=',',                    # Delimiter (use '\t' for TSV)
    header=0,                   # Row number to use as column names
    index_col='id',             # Column to use as row index
    usecols=['name', 'age', 'salary'],  # Only read specific columns
    dtype={'age': int, 'salary': float},  # Force data types
    na_values=['NA', 'N/A', '?', ''],     # Treat these as NaN
    parse_dates=['hire_date'],             # Parse date columns
    encoding='utf-8',                      # Character encoding
    nrows=10000                            # Read only first 10k rows (for large files)
)

# ============================================
# 2. Reading from Excel files
# ============================================
df_excel = pd.read_excel(
    'report.xlsx',
    sheet_name='Sheet1',        # Specific sheet (or index like 0)
    skiprows=2,                 # Skip header rows
    engine='openpyxl'           # Engine for .xlsx files
)

# Read all sheets into a dictionary
all_sheets = pd.read_excel('report.xlsx', sheet_name=None)
# all_sheets['Sheet1'], all_sheets['Sheet2'], etc.

# ============================================
# 3. Reading from JSON
# ============================================
# Flat JSON
df_json = pd.read_json('data.json')

# Nested JSON — need to normalize
with open('nested_data.json', 'r') as f:
    raw_data = json.load(f)

# Flatten nested structure
df_nested = pd.json_normalize(
    raw_data,
    record_path='orders',       # Path to the nested list
    meta=['customer_id', 'name'],  # Fields from parent
    sep='_'                     # Separator for nested field names
)

# ============================================
# 4. Reading from SQL Database
# ============================================
# SQLite example
conn = sqlite3.connect('company.db')
df_sql = pd.read_sql(
    "SELECT * FROM employees WHERE department = 'Engineering'",
    conn
)
conn.close()

# For production databases (PostgreSQL, MySQL):
# from sqlalchemy import create_engine
# engine = create_engine('postgresql://user:password@host:5432/dbname')
# df = pd.read_sql("SELECT * FROM table", engine)

# ============================================
# 5. Reading from Parquet (columnar format — fast for big data)
# ============================================
df_parquet = pd.read_parquet('data.parquet', columns=['col1', 'col2'])

# ============================================
# 6. Reading large files in chunks
# ============================================
# When file is too large for memory
chunk_size = 50000
chunks = []

for chunk in pd.read_csv('huge_file.csv', chunksize=chunk_size):
    # Process each chunk (filter, aggregate, etc.)
    filtered = chunk[chunk['value'] > 100]
    chunks.append(filtered)

# Combine all processed chunks
df_large = pd.concat(chunks, ignore_index=True)
```

> **Pro Tip**: For files larger than 1GB, consider using **Parquet** format instead of CSV. Parquet is 2-10x smaller and 10-100x faster to read because it's columnar and compressed.

---

## 4.2 Web Scraping

### What It Is
Web scraping is automatically extracting data from websites. Imagine having a super-fast assistant who visits a webpage, reads the information, and writes it down in a spreadsheet — that's web scraping.

### Why It Matters
- Access data that's not available through APIs
- Competitive analysis (pricing, product data)
- Research (academic papers, news articles)
- Building datasets when none exist publicly

### How It Works

```
┌─────────────┐     HTTP Request      ┌──────────────┐
│  Your Code  │ ──────────────────────>│  Web Server  │
│  (Python)   │                        │  (Website)   │
│             │ <──────────────────────│              │
└─────────────┘     HTML Response      └──────────────┘
       │
       ▼
┌─────────────┐
│   Parser    │  (BeautifulSoup / lxml)
│  HTML → Data│
└─────────────┘
       │
       ▼
┌─────────────┐
│  Structured │  (DataFrame / CSV / DB)
│    Data     │
└─────────────┘
```

### Code Examples

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time

# ============================================
# Basic Web Scraping with BeautifulSoup
# ============================================

# Step 1: Send HTTP request to get the page
url = "https://quotes.toscrape.com/"
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
}
response = requests.get(url, headers=headers)

# Step 2: Check if request was successful
if response.status_code == 200:
    # Step 3: Parse HTML content
    soup = BeautifulSoup(response.text, 'html.parser')
    
    # Step 4: Find elements using CSS selectors or tag names
    quotes = soup.find_all('div', class_='quote')
    
    data = []
    for quote in quotes:
        text = quote.find('span', class_='text').get_text()
        author = quote.find('small', class_='author').get_text()
        tags = [tag.get_text() for tag in quote.find_all('a', class_='tag')]
        
        data.append({
            'quote': text,
            'author': author,
            'tags': ', '.join(tags)
        })
    
    # Step 5: Convert to DataFrame
    df = pd.DataFrame(data)
    print(df.head())

# ============================================
# Scraping Multiple Pages (Pagination)
# ============================================
all_data = []
base_url = "https://quotes.toscrape.com/page/{}/"

for page_num in range(1, 11):  # Pages 1 through 10
    url = base_url.format(page_num)
    response = requests.get(url, headers=headers)
    
    if response.status_code != 200:
        break
    
    soup = BeautifulSoup(response.text, 'html.parser')
    quotes = soup.find_all('div', class_='quote')
    
    if not quotes:  # No more quotes — stop
        break
    
    for quote in quotes:
        text = quote.find('span', class_='text').get_text()
        author = quote.find('small', class_='author').get_text()
        all_data.append({'quote': text, 'author': author})
    
    # IMPORTANT: Be respectful — add delay between requests
    time.sleep(1)  # Wait 1 second between pages

df_all = pd.DataFrame(all_data)
print(f"Scraped {len(df_all)} quotes from multiple pages")

# ============================================
# Handling Dynamic Content (JavaScript-rendered pages)
# ============================================
# For pages that load content with JavaScript, use Selenium
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

# Setup Chrome driver
driver = webdriver.Chrome()
driver.get("https://example-dynamic-site.com")

# Wait for content to load
wait = WebDriverWait(driver, 10)
element = wait.until(
    EC.presence_of_element_located((By.CLASS_NAME, "product-card"))
)

# Now scrape the rendered HTML
html = driver.page_source
soup = BeautifulSoup(html, 'html.parser')
# ... extract data as before

driver.quit()
```

### Legal and Ethical Considerations

| Rule | Description |
|------|-------------|
| Check `robots.txt` | Visit `site.com/robots.txt` to see what's allowed |
| Respect rate limits | Add delays between requests (1-5 seconds) |
| Check Terms of Service | Some sites explicitly forbid scraping |
| Don't overload servers | Limit concurrent requests |
| Use APIs when available | Always prefer official APIs over scraping |
| Cache responses | Don't re-scrape data you already have |

> **Warning**: Scraping copyrighted content, personal data (GDPR), or behind-login content can have legal consequences. Always check the website's ToS.

---

## 4.3 Working with APIs

### What It Is
An API (Application Programming Interface) is a structured way to request data from a service. Think of it as ordering food at a restaurant: you (client) give your order (request) to the waiter (API), who brings back your food (response) from the kitchen (server).

### Why It Matters
- **Reliable**: APIs are designed for data access — more stable than scraping
- **Structured**: Data comes in clean JSON/XML format
- **Real-time**: Get the latest data instantly
- **Legal**: You're using data as intended by the provider

### How It Works — REST API Anatomy

```
┌─────────── API Request ───────────┐
│                                    │
│  Method:  GET / POST / PUT / DELETE│
│  URL:     https://api.example.com/ │
│            v1/users?page=1         │
│  Headers: Authorization: Bearer... │
│           Content-Type: app/json   │
│  Body:    {"name": "John"} (POST)  │
│                                    │
└────────────────────────────────────┘
              │
              ▼
┌─────────── API Response ──────────┐
│                                    │
│  Status:  200 OK / 404 Not Found   │
│  Headers: Content-Type: app/json   │
│  Body:    {"data": [...]}          │
│                                    │
└────────────────────────────────────┘
```

### Common HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success | Process the data |
| 201 | Created | Resource was created |
| 400 | Bad Request | Fix your request parameters |
| 401 | Unauthorized | Check your API key |
| 403 | Forbidden | You don't have permission |
| 404 | Not Found | Check the URL/endpoint |
| 429 | Too Many Requests | Slow down, respect rate limits |
| 500 | Server Error | Retry later |

### Code Examples

```python
import requests
import pandas as pd
import time

# ============================================
# Basic GET Request
# ============================================
# Free public API — no key needed
response = requests.get("https://jsonplaceholder.typicode.com/posts")

if response.status_code == 200:
    data = response.json()  # Parse JSON response
    df = pd.DataFrame(data)
    print(df.head())
    # Columns: userId, id, title, body

# ============================================
# API with Parameters
# ============================================
params = {
    'lat': 28.6139,      # New Delhi latitude
    'lon': 77.2090,      # New Delhi longitude
    'appid': 'YOUR_API_KEY',
    'units': 'metric'
}
response = requests.get(
    "https://api.openweathermap.org/data/2.5/weather",
    params=params
)
weather_data = response.json()
print(f"Temperature: {weather_data['main']['temp']}°C")

# ============================================
# API with Authentication (Bearer Token)
# ============================================
headers = {
    'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
    'Content-Type': 'application/json'
}
response = requests.get(
    "https://api.example.com/v1/data",
    headers=headers
)

# ============================================
# POST Request — Sending data to an API
# ============================================
payload = {
    'title': 'New Post',
    'body': 'This is the content',
    'userId': 1
}
response = requests.post(
    "https://jsonplaceholder.typicode.com/posts",
    json=payload  # Automatically serializes to JSON
)
print(response.status_code)  # 201 = Created
print(response.json())

# ============================================
# Handling Pagination in APIs
# ============================================
all_results = []
page = 1

while True:
    response = requests.get(
        "https://api.example.com/v1/items",
        params={'page': page, 'per_page': 100},
        headers=headers
    )
    
    data = response.json()
    
    if not data['results']:  # Empty page = we're done
        break
    
    all_results.extend(data['results'])
    page += 1
    time.sleep(0.5)  # Be respectful of rate limits

df = pd.DataFrame(all_results)

# ============================================
# Robust API Client with Retries
# ============================================
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def create_robust_session():
    """Create a session with automatic retries for transient errors."""
    session = requests.Session()
    
    retries = Retry(
        total=3,               # Max 3 retries
        backoff_factor=1,      # Wait 1, 2, 4 seconds between retries
        status_forcelist=[429, 500, 502, 503, 504]  # Retry on these codes
    )
    
    adapter = HTTPAdapter(max_retries=retries)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    
    return session

# Usage
session = create_robust_session()
response = session.get("https://api.example.com/data")
```

> **Pro Tip**: Always store API keys in environment variables, never in your code. Use `os.environ.get('API_KEY')` or a `.env` file with `python-dotenv`.

---

## 4.4 Handling Missing Values

### What It Is
Missing values are gaps in your data — cells where no value was recorded. In pandas, they appear as `NaN` (Not a Number) or `None`. Think of it like a survey where some people skipped questions — you still need to analyze the data, but those blank answers need to be dealt with.

### Why It Matters
- Most ML algorithms **cannot handle** missing values (they'll crash or give wrong results)
- Missing data can introduce **bias** if not handled correctly
- How you handle missing data affects your model's accuracy significantly
- Understanding **why** data is missing is as important as filling it

### Types of Missing Data

| Type | Meaning | Example | Strategy |
|------|---------|---------|----------|
| **MCAR** (Missing Completely at Random) | Missingness has no pattern | A sensor randomly fails | Safe to drop or impute |
| **MAR** (Missing at Random) | Related to other observed variables | Older people skip income question | Use related columns to impute |
| **MNAR** (Missing Not at Random) | Related to the missing value itself | High earners hide salary | Hardest — need domain knowledge |

### How It Works — Decision Framework

```
                    ┌───────────────────────┐
                    │  Is data missing?     │
                    └──────────┬────────────┘
                               │ Yes
                    ┌──────────▼────────────┐
                    │  How much is missing? │
                    └──────────┬────────────┘
                    ┌──────────┼────────────┐
                    │          │            │
                < 5%       5-30%        > 30%
                    │          │            │
              ┌─────▼────┐ ┌──▼──────┐ ┌───▼────────┐
              │ Drop rows│ │ Impute  │ │ Drop column│
              │ or simple│ │ (smart) │ │ or create  │
              │ impute   │ │         │ │ indicator  │
              └──────────┘ └─────────┘ └────────────┘
```

### Code Examples

```python
import pandas as pd
import numpy as np
from sklearn.impute import SimpleImputer, KNNImputer

# ============================================
# Create sample data with missing values
# ============================================
np.random.seed(42)
df = pd.DataFrame({
    'age': [25, 30, np.nan, 45, 50, np.nan, 35, 28, np.nan, 40],
    'salary': [50000, np.nan, 70000, 80000, np.nan, 60000, np.nan, 55000, 75000, 90000],
    'department': ['IT', 'HR', 'IT', None, 'Finance', 'HR', 'IT', None, 'Finance', 'IT'],
    'experience': [2, 5, np.nan, 10, 15, np.nan, 7, 3, 8, 12],
    'rating': [4.5, np.nan, 3.8, 4.2, np.nan, np.nan, 4.0, 3.5, 4.1, np.nan]
})

print("Original Data:")
print(df)
print(f"\nShape: {df.shape}")

# ============================================
# Step 1: ASSESS the damage — understand missingness
# ============================================
# Count missing values per column
print("\n--- Missing Value Analysis ---")
missing_info = pd.DataFrame({
    'missing_count': df.isnull().sum(),
    'missing_percent': (df.isnull().sum() / len(df) * 100).round(2),
    'dtype': df.dtypes
})
print(missing_info)

# Total missing values
print(f"\nTotal missing: {df.isnull().sum().sum()}")
print(f"Total cells: {df.size}")
print(f"Missing %: {(df.isnull().sum().sum() / df.size * 100):.2f}%")

# Visualize missing pattern (which rows have missing values together)
print("\nMissing pattern (True = Missing):")
print(df.isnull().astype(int))

# ============================================
# Step 2: DROP — When appropriate
# ============================================
# Drop rows where ANY value is missing (aggressive — use only if few missing)
df_dropped_any = df.dropna()
print(f"\nAfter dropna(any): {len(df_dropped_any)} rows remain (from {len(df)})")

# Drop rows where ALL values are missing
df_dropped_all = df.dropna(how='all')

# Drop rows with missing values in SPECIFIC columns only
df_dropped_subset = df.dropna(subset=['age', 'salary'])
print(f"After dropping rows missing age OR salary: {len(df_dropped_subset)} rows")

# Drop COLUMNS with too many missing values (threshold approach)
threshold = 0.3  # Drop if more than 30% missing
cols_to_keep = df.columns[df.isnull().mean() < threshold]
df_filtered_cols = df[cols_to_keep]
print(f"Columns kept (< 30% missing): {list(cols_to_keep)}")

# ============================================
# Step 3: IMPUTE — Fill missing values intelligently
# ============================================

# --- Method 1: Fill with constant ---
df_const = df.copy()
df_const['department'].fillna('Unknown', inplace=True)  # Categorical
df_const['age'].fillna(0, inplace=True)                 # Risky! 0 age makes no sense

# --- Method 2: Fill with mean/median/mode ---
df_stat = df.copy()
df_stat['age'].fillna(df_stat['age'].median(), inplace=True)       # Median (robust to outliers)
df_stat['salary'].fillna(df_stat['salary'].mean(), inplace=True)   # Mean
df_stat['department'].fillna(df_stat['department'].mode()[0], inplace=True)  # Mode (most frequent)

print(f"\nAfter mean/median imputation:")
print(df_stat[['age', 'salary', 'department']].head())

# --- Method 3: Forward/Backward Fill (for time series) ---
# Forward fill: use previous value
df_ffill = df.copy()
df_ffill['salary'] = df_ffill['salary'].ffill()  # Carry forward last known value

# Backward fill: use next value  
df_bfill = df.copy()
df_bfill['salary'] = df_bfill['salary'].bfill()  # Use next known value

# --- Method 4: Group-based Imputation (SMART approach) ---
# Fill salary with the MEDIAN salary of the same department
df_group = df.copy()
df_group['salary'] = df_group.groupby('department')['salary'].transform(
    lambda x: x.fillna(x.median())
)
# Remaining NaN (where department is also missing) — fill with overall median
df_group['salary'].fillna(df_group['salary'].median(), inplace=True)

print(f"\nGroup-based imputation (salary by department):")
print(df_group[['department', 'salary']])

# --- Method 5: KNN Imputation (uses similar rows) ---
# Only works for numeric columns
numeric_cols = df.select_dtypes(include=[np.number]).columns
df_knn = df.copy()

knn_imputer = KNNImputer(n_neighbors=3)  # Use 3 nearest neighbors
df_knn[numeric_cols] = knn_imputer.fit_transform(df_knn[numeric_cols])

print(f"\nKNN Imputation result:")
print(df_knn[numeric_cols].head())

# --- Method 6: Create a "missing" indicator column ---
# Preserves the information that value WAS missing
df_indicator = df.copy()
df_indicator['salary_was_missing'] = df_indicator['salary'].isnull().astype(int)
df_indicator['salary'].fillna(df_indicator['salary'].median(), inplace=True)

print(f"\nWith missing indicator:")
print(df_indicator[['salary', 'salary_was_missing']].head())

# ============================================
# Step 4: INTERPOLATION (for ordered/time series data)
# ============================================
# Create time series example
ts = pd.Series([10, np.nan, np.nan, 40, 50, np.nan, 70])

print("\nInterpolation methods:")
print(f"Original:    {ts.values}")
print(f"Linear:      {ts.interpolate(method='linear').values}")
print(f"Polynomial:  {ts.interpolate(method='polynomial', order=2).values}")
```

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Dropping all rows with NaN | Lose too much data, introduce bias | Impute intelligently based on % missing |
| Using mean for skewed data | Mean is pulled by outliers | Use **median** for skewed distributions |
| Imputing before train/test split | **Data leakage** — test data influences imputation | Fit imputer on train, transform test |
| Ignoring why data is missing | MNAR data can't be simply imputed | Investigate the mechanism first |
| Filling categorical with mean | Categories don't have a "mean" | Use mode or 'Unknown' category |
| Not creating missing indicators | Lose the information that data was missing | Add boolean column before imputing |

> **Critical Warning — Data Leakage**: Always split your data into train/test BEFORE imputing. Calculate statistics (mean, median) only from the training set, then apply those values to the test set.

```python
from sklearn.model_selection import train_test_split
from sklearn.impute import SimpleImputer

# CORRECT way — impute after splitting
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

imputer = SimpleImputer(strategy='median')
X_train_imputed = imputer.fit_transform(X_train)  # Learn from train
X_test_imputed = imputer.transform(X_test)         # Apply to test (no fitting!)
```

---

## 4.5 Outlier Detection and Treatment

### What It Is
Outliers are data points that are significantly different from the rest of the data. Imagine measuring heights in a classroom and finding one entry of 15 feet — that's either a data entry error or a very unusual observation. Outliers can be **errors** (wrong data) or **genuine** extreme values (a billionaire in a salary survey).

### Why It Matters
- Outliers can **dramatically skew** means, standard deviations, and model parameters
- Some models (linear regression) are very sensitive to outliers
- Outliers might be the most interesting data points (fraud detection!)
- Deciding to keep or remove outliers is a **domain decision**, not just a statistical one

### Detection Methods

```
          Method Comparison
          
IQR Method:          Z-Score:           Visual:
                     
  ◄── Q1-1.5×IQR    z < -3 ──►        Boxplot
  │                  │                  │
  │   ┌───────┐     │  Normal          │  ─┬─  ← outlier
  │   │  IQR  │     │  Range           │   │
  ├───┤       │     │  │               │ ┌─┼─┐
  │   │       │     │  ▼               │ │ │ │
  │   └───────┘     │                  │ └─┼─┘
  │                  z > 3  ──►        │   │
  ◄── Q3+1.5×IQR                       │  ─┴─  ← outlier
```

### Code Examples

```python
import pandas as pd
import numpy as np
from scipy import stats

# Sample data with outliers
np.random.seed(42)
normal_data = np.random.normal(50, 10, 100)  # Mean=50, Std=10
outliers = np.array([150, 200, -50, 180])    # Obvious outliers
data = np.concatenate([normal_data, outliers])
df = pd.DataFrame({'value': data})

print(f"Data shape: {df.shape}")
print(f"Mean: {df['value'].mean():.2f}")
print(f"Median: {df['value'].median():.2f}")
print(f"Std: {df['value'].std():.2f}")

# ============================================
# Method 1: IQR (Interquartile Range) Method
# ============================================
def detect_outliers_iqr(series, multiplier=1.5):
    """
    Detect outliers using IQR method.
    Points outside [Q1 - 1.5*IQR, Q3 + 1.5*IQR] are outliers.
    Use multiplier=3.0 for extreme outliers only.
    """
    Q1 = series.quantile(0.25)
    Q3 = series.quantile(0.75)
    IQR = Q3 - Q1
    
    lower_bound = Q1 - multiplier * IQR
    upper_bound = Q3 + multiplier * IQR
    
    outlier_mask = (series < lower_bound) | (series > upper_bound)
    
    return outlier_mask, lower_bound, upper_bound

mask, lower, upper = detect_outliers_iqr(df['value'])
print(f"\n--- IQR Method ---")
print(f"Bounds: [{lower:.2f}, {upper:.2f}]")
print(f"Outliers found: {mask.sum()}")
print(f"Outlier values: {df[mask]['value'].values}")

# ============================================
# Method 2: Z-Score Method
# ============================================
def detect_outliers_zscore(series, threshold=3):
    """
    Detect outliers using Z-score.
    Points with |z| > threshold are outliers.
    Assumes roughly normal distribution.
    """
    z_scores = np.abs(stats.zscore(series))
    return z_scores > threshold

mask_z = detect_outliers_zscore(df['value'])
print(f"\n--- Z-Score Method (threshold=3) ---")
print(f"Outliers found: {mask_z.sum()}")
print(f"Outlier values: {df[mask_z]['value'].values}")

# ============================================
# Method 3: Modified Z-Score (Robust — uses median)
# ============================================
def detect_outliers_modified_zscore(series, threshold=3.5):
    """
    Uses Median Absolute Deviation (MAD) instead of mean/std.
    More robust when data already has outliers contaminating mean/std.
    """
    median = series.median()
    mad = np.median(np.abs(series - median))
    
    # 0.6745 is the z-score for the 75th percentile of standard normal
    modified_z_scores = 0.6745 * (series - median) / mad
    
    return np.abs(modified_z_scores) > threshold

mask_mod = detect_outliers_modified_zscore(df['value'])
print(f"\n--- Modified Z-Score Method ---")
print(f"Outliers found: {mask_mod.sum()}")

# ============================================
# Method 4: Percentile Method (simple but effective)
# ============================================
lower_percentile = df['value'].quantile(0.01)  # 1st percentile
upper_percentile = df['value'].quantile(0.99)  # 99th percentile
mask_pct = (df['value'] < lower_percentile) | (df['value'] > upper_percentile)
print(f"\n--- Percentile Method (1%-99%) ---")
print(f"Outliers found: {mask_pct.sum()}")

# ============================================
# TREATMENT — What to do with outliers
# ============================================
df_treated = df.copy()

# Option 1: Remove outliers
df_removed = df[~mask].copy()
print(f"\nAfter removal: {len(df_removed)} rows (was {len(df)})")

# Option 2: Cap/Clip (Winsorization) — replace outliers with bounds
df_treated['value_capped'] = df_treated['value'].clip(lower=lower, upper=upper)
print(f"Capped range: [{df_treated['value_capped'].min():.2f}, {df_treated['value_capped'].max():.2f}]")

# Option 3: Log transformation (reduces impact of extreme values)
# Only for positive values
positive_values = df[df['value'] > 0]['value']
df_log = np.log1p(positive_values)  # log(1 + x) handles zero values too
print(f"Log transform — Original std: {positive_values.std():.2f}, Log std: {df_log.std():.2f}")

# Option 4: Replace with NaN and impute
df_treated['value_nan'] = df_treated['value'].copy()
df_treated.loc[mask, 'value_nan'] = np.nan
df_treated['value_nan'].fillna(df_treated['value_nan'].median(), inplace=True)

# ============================================
# Domain-specific approach: Keep outliers as a feature
# ============================================
# In fraud detection, outliers ARE the signal!
df_treated['is_outlier'] = mask.astype(int)
print(f"\nOutlier flag added: {df_treated['is_outlier'].sum()} flagged")
```

### When to Keep vs. Remove Outliers

| Scenario | Action | Reason |
|----------|--------|--------|
| Data entry error (age = 500) | **Remove** | Clearly wrong |
| Fraud detection | **Keep** | Outliers ARE what you're looking for |
| Salary data with CEO | **Cap/Transform** | Real but skews analysis |
| Sensor malfunction | **Remove** | Not real measurements |
| Customer spending (whales) | **Keep + Flag** | Genuine high-value customers |

---

## 4.6 Data Type Conversions and Consistency

### What It Is
Ensuring every column in your dataset has the correct data type and consistent formatting. It's like making sure all the books in a library are shelved in the right section — numbers with numbers, dates with dates, categories with categories.

### Why It Matters
- Wrong types waste memory (storing "1" as string vs integer)
- Operations fail silently (you can't average strings!)
- Inconsistent formats cause joins/merges to fail
- Models require specific types (numeric for most ML algorithms)

### Code Examples

```python
import pandas as pd
import numpy as np

# Create messy data (as it comes from real sources)
df = pd.DataFrame({
    'id': ['001', '002', '003', '004', '005'],
    'price': ['$1,200.50', '$800.00', '€950.30', '$1,100.00', 'N/A'],
    'date_joined': ['2023-01-15', '01/20/2023', 'Jan 5, 2023', '2023.02.01', '15-03-2023'],
    'is_active': ['Yes', 'No', 'TRUE', '1', 'yes'],
    'age': ['25', '30', 'thirty', '45', '28'],
    'category': ['A', 'B', 'A', 'C', 'B'],
    'revenue': ['1000000', '500000', '750000', '2000000', '300000']
})

print("Original dtypes:")
print(df.dtypes)
print("\n", df)

# ============================================
# 1. Clean and convert NUMERIC columns
# ============================================
# Remove currency symbols, commas, and convert to float
def clean_currency(value):
    """Remove currency symbols and commas, return float or NaN."""
    if pd.isna(value) or value in ['N/A', 'NA', '']:
        return np.nan
    # Remove common currency symbols and commas
    cleaned = str(value).replace('$', '').replace('€', '').replace('£', '').replace(',', '')
    try:
        return float(cleaned)
    except ValueError:
        return np.nan

df['price_clean'] = df['price'].apply(clean_currency)
print(f"\nCleaned prices: {df['price_clean'].values}")

# Convert numeric strings (handling errors)
df['age_clean'] = pd.to_numeric(df['age'], errors='coerce')  # 'coerce' turns errors to NaN
print(f"Cleaned ages: {df['age_clean'].values}")  # 'thirty' becomes NaN

# Memory optimization — downcast numeric types
df['revenue'] = pd.to_numeric(df['revenue'])
print(f"\nRevenue dtype before: {df['revenue'].dtype}")  # int64 (8 bytes per value)
df['revenue'] = pd.to_numeric(df['revenue'], downcast='integer')
print(f"Revenue dtype after downcast: {df['revenue'].dtype}")  # int32 (4 bytes — 50% savings!)

# ============================================
# 2. Parse DATES consistently
# ============================================
# pandas can infer many date formats
df['date_parsed'] = pd.to_datetime(df['date_joined'], format='mixed', dayfirst=False)
print(f"\nParsed dates:\n{df[['date_joined', 'date_parsed']]}")

# Extract useful components
df['join_year'] = df['date_parsed'].dt.year
df['join_month'] = df['date_parsed'].dt.month
df['join_dayofweek'] = df['date_parsed'].dt.day_name()

# ============================================
# 3. Standardize BOOLEAN columns
# ============================================
def standardize_boolean(value):
    """Convert various representations to True/False."""
    true_values = {'yes', 'true', '1', 't', 'y'}
    false_values = {'no', 'false', '0', 'f', 'n'}
    
    str_val = str(value).lower().strip()
    if str_val in true_values:
        return True
    elif str_val in false_values:
        return False
    return None  # Unknown values

df['is_active_clean'] = df['is_active'].apply(standardize_boolean)
print(f"\nCleaned booleans: {df['is_active_clean'].values}")

# ============================================
# 4. Convert to CATEGORICAL (saves memory)
# ============================================
print(f"\nCategory column memory (object): {df['category'].memory_usage(deep=True)} bytes")
df['category'] = df['category'].astype('category')
print(f"Category column memory (category): {df['category'].memory_usage(deep=True)} bytes")

# ============================================
# 5. String CLEANING and standardization
# ============================================
messy_strings = pd.Series([' New York ', 'new york', 'NEW YORK', 'New  York', 'NewYork'])

# Standard cleaning pipeline
cleaned = (messy_strings
    .str.strip()           # Remove leading/trailing whitespace
    .str.lower()           # Lowercase everything
    .str.replace(r'\s+', ' ', regex=True)  # Multiple spaces → single space
)
print(f"\nCleaned strings: {cleaned.values}")
# All become: 'new york' (except 'newyork' — might need fuzzy matching)
```

---

## 4.7 Duplicate Detection and Removal

### What It Is
Finding and handling rows that appear more than once in your dataset. Duplicates can be **exact** (every column identical) or **partial** (same entity recorded slightly differently, like "John Smith" vs "john smith").

### Why It Matters
- Duplicates inflate your dataset, making statistics incorrect
- They can cause **data leakage** in ML (same sample in train and test)
- In business contexts, duplicates mean sending two emails, charging twice, etc.

### Code Examples

```python
import pandas as pd
import numpy as np

# Create data with various types of duplicates
df = pd.DataFrame({
    'customer_id': [1, 2, 3, 2, 4, 5, 3, 6],
    'name': ['Alice', 'Bob', 'Charlie', 'Bob', 'David', 'Eve', 'charlie', 'Frank'],
    'email': ['alice@mail.com', 'bob@mail.com', 'charlie@mail.com', 
              'bob@mail.com', 'david@mail.com', 'eve@mail.com', 
              'Charlie@Mail.com', 'frank@mail.com'],
    'purchase': [100, 200, 150, 200, 300, 250, 175, 400],
    'date': ['2023-01-01', '2023-01-02', '2023-01-03', '2023-01-02',
             '2023-01-04', '2023-01-05', '2023-01-06', '2023-01-07']
})

print("Original data:")
print(df)

# ============================================
# 1. Find EXACT duplicates (all columns match)
# ============================================
exact_dupes = df.duplicated()  # Returns boolean mask
print(f"\nExact duplicates: {exact_dupes.sum()}")
print(df[exact_dupes])

# ============================================
# 2. Find duplicates based on SPECIFIC columns
# ============================================
# Same customer_id = likely duplicate entry
id_dupes = df.duplicated(subset=['customer_id'], keep=False)  # keep=False marks ALL copies
print(f"\nDuplicate customer_ids (all copies):")
print(df[id_dupes])

# ============================================
# 3. Find FUZZY duplicates (case-insensitive)
# ============================================
# Normalize before checking
df['email_normalized'] = df['email'].str.lower().str.strip()
email_dupes = df.duplicated(subset=['email_normalized'], keep=False)
print(f"\nFuzzy email duplicates:")
print(df[email_dupes][['name', 'email', 'email_normalized']])

# ============================================
# 4. REMOVE duplicates
# ============================================
# Keep first occurrence
df_deduped = df.drop_duplicates(subset=['customer_id'], keep='first')
print(f"\nAfter dedup (keep first): {len(df_deduped)} rows")

# Keep last occurrence (useful for most recent record)
df_deduped_last = df.drop_duplicates(subset=['customer_id'], keep='last')

# Remove ALL copies (keep none)
df_no_dupes = df.drop_duplicates(subset=['customer_id'], keep=False)
print(f"After removing all duplicated IDs: {len(df_no_dupes)} rows")

# ============================================
# 5. AGGREGATE duplicates instead of dropping
# ============================================
# Instead of dropping, combine information from duplicates
df_aggregated = df.groupby('customer_id').agg({
    'name': 'first',           # Keep first name
    'email': 'first',          # Keep first email
    'purchase': 'sum',         # Sum all purchases
    'date': 'max'              # Keep most recent date
}).reset_index()

print(f"\nAggregated duplicates:")
print(df_aggregated)
```

> **Pro Tip**: In production, always log which duplicates were removed and why. Create a "rejected records" table for audit purposes.

---

## 4.8 Data Validation and Quality Checks

### What It Is
Systematically verifying that your data meets expected rules, ranges, formats, and business logic. Think of it as quality control on a factory assembly line — checking every product before it ships.

### Why It Matters
- Catches errors before they propagate to models/reports
- Ensures data pipeline reliability
- Meets compliance requirements (financial, healthcare data)
- Saves debugging time downstream

### Code Examples

```python
import pandas as pd
import numpy as np

# ============================================
# Comprehensive Data Validation Framework
# ============================================

def validate_dataframe(df, rules):
    """
    Validate a DataFrame against a set of rules.
    Returns a report of all violations.
    """
    violations = []
    
    for rule in rules:
        rule_name = rule['name']
        column = rule.get('column')
        check = rule['check']
        
        if column and column not in df.columns:
            violations.append({
                'rule': rule_name,
                'column': column,
                'issue': f"Column '{column}' not found in DataFrame",
                'rows_affected': len(df)
            })
            continue
        
        # Apply check function
        if column:
            mask = ~check(df[column])  # Invert: True = violation
        else:
            mask = ~check(df)
        
        if mask.any():
            violations.append({
                'rule': rule_name,
                'column': column,
                'issue': f"{mask.sum()} rows violate this rule",
                'rows_affected': int(mask.sum()),
                'sample_indices': df[mask].index.tolist()[:5]
            })
    
    return violations

# Define validation rules
rules = [
    {
        'name': 'age_positive',
        'column': 'age',
        'check': lambda s: s.between(0, 120) | s.isna()  # Age 0-120 or NaN is OK
    },
    {
        'name': 'salary_reasonable',
        'column': 'salary',
        'check': lambda s: (s >= 0) | s.isna()  # No negative salaries
    },
    {
        'name': 'email_format',
        'column': 'email',
        'check': lambda s: s.str.contains(r'^[\w\.\-]+@[\w\.\-]+\.\w+$', na=True)
    },
    {
        'name': 'no_future_dates',
        'column': 'hire_date',
        'check': lambda s: s <= pd.Timestamp.now()
    },
    {
        'name': 'unique_ids',
        'column': 'employee_id',
        'check': lambda s: ~s.duplicated()
    }
]

# Example usage:
# violations = validate_dataframe(df, rules)
# for v in violations:
#     print(f"❌ {v['rule']}: {v['issue']}")

# ============================================
# Quick Data Quality Summary
# ============================================
def data_quality_report(df):
    """Generate a comprehensive data quality report."""
    report = pd.DataFrame({
        'dtype': df.dtypes,
        'non_null': df.count(),
        'null_count': df.isnull().sum(),
        'null_pct': (df.isnull().sum() / len(df) * 100).round(2),
        'unique': df.nunique(),
        'duplicate_rows': df.duplicated().sum()
    })
    
    # Add statistics for numeric columns
    numeric_cols = df.select_dtypes(include=[np.number]).columns
    for col in numeric_cols:
        report.loc[col, 'min'] = df[col].min()
        report.loc[col, 'max'] = df[col].max()
        report.loc[col, 'mean'] = df[col].mean()
    
    return report

# Usage:
# print(data_quality_report(df))
```

---

## 4.9 Complete Data Cleaning Pipeline

### Production-Ready Pipeline

```python
import pandas as pd
import numpy as np
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler

class DataCleaningPipeline:
    """
    A reusable data cleaning pipeline that can be fitted on training data
    and applied consistently to new data.
    """
    
    def __init__(self, numeric_strategy='median', categorical_strategy='most_frequent'):
        self.numeric_strategy = numeric_strategy
        self.categorical_strategy = categorical_strategy
        self.numeric_imputer = None
        self.categorical_imputer = None
        self.numeric_cols = None
        self.categorical_cols = None
        self.outlier_bounds = {}
    
    def fit(self, df):
        """Learn cleaning parameters from training data."""
        self.numeric_cols = df.select_dtypes(include=[np.number]).columns.tolist()
        self.categorical_cols = df.select_dtypes(include=['object', 'category']).columns.tolist()
        
        # Fit imputers
        if self.numeric_cols:
            self.numeric_imputer = SimpleImputer(strategy=self.numeric_strategy)
            self.numeric_imputer.fit(df[self.numeric_cols])
        
        if self.categorical_cols:
            self.categorical_imputer = SimpleImputer(strategy=self.categorical_strategy)
            self.categorical_imputer.fit(df[self.categorical_cols])
        
        # Calculate outlier bounds (IQR method)
        for col in self.numeric_cols:
            Q1 = df[col].quantile(0.25)
            Q3 = df[col].quantile(0.75)
            IQR = Q3 - Q1
            self.outlier_bounds[col] = {
                'lower': Q1 - 1.5 * IQR,
                'upper': Q3 + 1.5 * IQR
            }
        
        return self
    
    def transform(self, df):
        """Apply cleaning to data."""
        df_clean = df.copy()
        
        # Step 1: Remove exact duplicates
        df_clean = df_clean.drop_duplicates()
        
        # Step 2: Impute missing values
        if self.numeric_cols:
            df_clean[self.numeric_cols] = self.numeric_imputer.transform(
                df_clean[self.numeric_cols]
            )
        if self.categorical_cols:
            df_clean[self.categorical_cols] = self.categorical_imputer.transform(
                df_clean[self.categorical_cols]
            )
        
        # Step 3: Cap outliers
        for col in self.numeric_cols:
            bounds = self.outlier_bounds[col]
            df_clean[col] = df_clean[col].clip(
                lower=bounds['lower'], 
                upper=bounds['upper']
            )
        
        return df_clean
    
    def fit_transform(self, df):
        """Fit and transform in one step."""
        return self.fit(df).transform(df)

# Usage:
# pipeline = DataCleaningPipeline(numeric_strategy='median')
# df_train_clean = pipeline.fit_transform(df_train)
# df_test_clean = pipeline.transform(df_test)  # Uses train statistics!
```

---

## Interview Questions

1. **How do you handle missing data? Walk me through your approach.**
   - Assess: how much is missing, what type (MCAR/MAR/MNAR)
   - Decide: drop vs. impute based on percentage and mechanism
   - Implement: group-based imputation > simple mean/median
   - Validate: create missing indicators, check distributions post-imputation

2. **What's data leakage and how does it relate to data cleaning?**
   - Using test data information during training (e.g., imputing with full dataset mean)
   - Solution: fit cleaning steps on train data only, then apply to test

3. **How would you handle 40% missing values in a column?**
   - Don't drop the column immediately — check if missingness is informative
   - Create a binary "is_missing" indicator
   - Consider if the column is still useful for prediction with imputation

4. **When would you keep outliers vs. remove them?**
   - Keep: fraud detection, anomaly detection, genuine extreme values
   - Remove: data entry errors, sensor malfunctions
   - Cap/Transform: when you want to reduce impact but not lose information

5. **What's the difference between web scraping and using APIs?**
   - APIs: official, structured, reliable, rate-limited, legal
   - Scraping: unofficial, fragile, potentially illegal, more data available

---

## Quick Reference

| Task | Method | Code |
|------|--------|------|
| Check missing | Count NaN | `df.isnull().sum()` |
| Drop NaN rows | Subset drop | `df.dropna(subset=['col'])` |
| Fill with median | Simple impute | `df['col'].fillna(df['col'].median())` |
| Group impute | Smart fill | `df.groupby('grp')['col'].transform(lambda x: x.fillna(x.median()))` |
| Detect outliers (IQR) | Bounds check | `Q1 - 1.5*IQR` to `Q3 + 1.5*IQR` |
| Cap outliers | Winsorize | `df['col'].clip(lower, upper)` |
| Remove duplicates | Drop dupes | `df.drop_duplicates(subset=['key'])` |
| Convert types | Coerce errors | `pd.to_numeric(df['col'], errors='coerce')` |
| Parse dates | Flexible parse | `pd.to_datetime(df['col'], format='mixed')` |
| Read large CSV | Chunked read | `pd.read_csv('file.csv', chunksize=50000)` |

---

*Next: [Chapter 05 - Exploratory Data Analysis](05-Exploratory-Data-Analysis.md)*
