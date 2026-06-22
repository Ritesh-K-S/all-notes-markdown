# 6.1 — Data Warehouse Concepts (OLTP vs OLAP) 🟡⭐🔥

> **"If your transactional database is the heart pumping data in real-time, the data warehouse is the brain — analyzing everything to make you smarter."**

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand  
> **Time to Master:** ~4-5 hours  
> **Prerequisites:** Chapter 1.4 (Data Modeling), Chapter 2.5 (Aggregations)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why** data warehouses exist and what problem they solve
- Clearly differentiate **OLTP vs OLAP** — and never confuse them again
- Design **Star Schema** and **Snowflake Schema** like a data architect
- Master **Fact Tables** and **Dimension Tables** — the building blocks
- Handle **Slowly Changing Dimensions (SCD)** — Types 0 through 6
- Know **ETL vs ELT** in the warehouse context
- Understand the **modern data stack** evolution: Warehouse → Lake → Lakehouse

---

## 🧠 The Origin Story — Why Do We Need a Data Warehouse?

### The Problem: Your Production DB is Drowning

Imagine you're the CTO of an e-commerce company. Your PostgreSQL production database handles:

```
🛒 10,000 orders/minute      (INSERT)
👤 50,000 user logins/hour    (SELECT + UPDATE)
📦 1,000 inventory updates    (UPDATE)
💳 Payment processing         (INSERT + UPDATE)
```

Now the CEO walks in:

```
CEO:   "What were our top 10 products by revenue across all regions, 
        compared quarter-over-quarter, for the last 3 years, segmented 
        by customer age group?"

DBA:   😱 "That query would JOIN 12 tables, scan 500 million rows, 
        and take 45 minutes... while KILLING our production database."

CEO:   "I need it in my meeting in 10 minutes."

DBA:   💀
```

### Why You Can't Just Query Production

```
❌ Problem 1: ANALYTICAL QUERIES ARE HEAVY
   - They scan millions/billions of rows
   - They JOIN many tables
   - They use GROUP BY, aggregations, window functions
   
❌ Problem 2: THEY KILL PERFORMANCE
   - Production needs sub-millisecond response for transactions
   - One big analytical query locks tables → customers can't checkout

❌ Problem 3: DATA ISN'T ANALYSIS-FRIENDLY
   - Normalized to 3NF (great for writes, horrible for reads)
   - No historical snapshots (current state only)
   - No cross-system data (CRM + ERP + Marketing are separate)

❌ Problem 4: BUSINESS NEEDS HISTORICAL DATA
   - Production only keeps "current state" 
   - "What was this customer's address last year?" → Gone
   - "How did pricing change over time?" → Not tracked
```

### The Solution: Build a Separate Analytical System

```
┌────────────────────┐    ETL/ELT    ┌──────────────────────┐
│  OLTP Systems      │ ────────────► │  DATA WAREHOUSE      │
│  (Production DBs)  │               │  (Analytical System)  │
├────────────────────┤               ├──────────────────────┤
│ • PostgreSQL       │               │ • Optimized for READS │
│ • MySQL            │   Transform   │ • Denormalized schemas│
│ • Oracle           │   + Load      │ • Historical data     │
│ • MongoDB          │               │ • Cross-system data   │
│ • SAP              │               │ • Fast aggregations   │
└────────────────────┘               └──────────────────────┘
       ↑                                        ↑
  Customers use this                   Analysts use this
  (fast transactions)                 (complex queries)
```

> 💡 **Key Insight**: A data warehouse is NOT a backup. It's a **purpose-built analytical system** that transforms raw operational data into business intelligence gold.

---

## ⚔️ OLTP vs OLAP — The Two Worlds

This is one of the **most asked interview questions** in data engineering. Nail it.

### OLTP — Online Transaction Processing

```
OLTP = The "Doer"
→ Handles day-to-day business operations
→ INSERT a new order, UPDATE inventory, DELETE a cart item
→ Fast, small, frequent transactions
→ Your production database
```

### OLAP — Online Analytical Processing

```
OLAP = The "Thinker"  
→ Analyzes historical business data for insights
→ "Total revenue by region by quarter for last 5 years"
→ Complex, heavy, infrequent queries
→ Your data warehouse
```

### Head-to-Head Comparison

| Feature | OLTP 🏃 | OLAP 🧠 |
|---------|---------|---------|
| **Purpose** | Day-to-day operations | Analysis & reporting |
| **Users** | Clerks, customers, apps | Analysts, data scientists, executives |
| **Queries** | Simple (`SELECT * WHERE id = 5`) | Complex (JOINs, GROUP BY, window functions) |
| **Query Volume** | Thousands per second | Few per hour (but each is MASSIVE) |
| **Data Size per Query** | Few rows (1-100) | Millions to billions of rows |
| **Data Freshness** | Real-time (current state) | Periodic (hourly/daily refresh) |
| **Schema Design** | Normalized (3NF) | Denormalized (Star/Snowflake) |
| **Optimization** | Write-optimized | Read-optimized |
| **Storage** | Row-oriented (read whole rows) | Column-oriented (read specific columns) |
| **History** | Current snapshot only | Full history maintained |
| **Concurrency** | Very high (10,000+ users) | Low (50-500 analysts) |
| **Response Time** | Milliseconds | Seconds to minutes |
| **Backup Focus** | Point-in-time recovery | Reload from source |
| **Examples** | MySQL, PostgreSQL, Oracle | Redshift, BigQuery, Snowflake |

### Visual: How They Work Differently

```
OLTP — Row-Oriented Storage (Great for transactions)
┌──────┬──────────┬─────────┬────────┬──────────┐
│ ID   │ Name     │ City    │ Amount │ Date     │  ← Row 1 (all together on disk)
├──────┼──────────┼─────────┼────────┼──────────┤
│ ID   │ Name     │ City    │ Amount │ Date     │  ← Row 2 (all together on disk)
├──────┼──────────┼─────────┼────────┼──────────┤
│ ID   │ Name     │ City    │ Amount │ Date     │  ← Row 3 (all together on disk)
└──────┴──────────┴─────────┴────────┴──────────┘

Query: SELECT * FROM orders WHERE id = 42;
→ Read 1 row = read 1 disk block ✅ FAST


OLAP — Column-Oriented Storage (Great for analytics)
┌──────┬──────┬──────┐   ┌────────┬────────┬────────┐
│ ID₁  │ ID₂  │ ID₃  │   │ Name₁  │ Name₂  │ Name₃  │   ... each column stored together
└──────┴──────┴──────┘   └────────┴────────┴────────┘

┌─────────┬─────────┬─────────┐   ┌────────┬────────┬────────┐
│ City₁   │ City₂   │ City₃   │   │Amount₁ │Amount₂ │Amount₃ │
└─────────┴─────────┴─────────┘   └────────┴────────┴────────┘

Query: SELECT SUM(Amount) FROM orders;
→ Only read the Amount column = skip 80% of data ✅ FAST
→ Compression: same data types together = 10x compression ratio
```

> 🔥 **This is why Redshift, BigQuery, and Snowflake are column-oriented** — they only read the columns you ask for.

---

## 🏗️ What IS a Data Warehouse? — Formal Definition

### Bill Inmon's Definition (The "Father of Data Warehousing")

> A data warehouse is a **subject-oriented**, **integrated**, **non-volatile**, **time-variant** collection of data in support of management's decisions.

Let's decode each word:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  📦 SUBJECT-ORIENTED                                        │
│  → Organized by business subjects (Sales, Customer, Product)│
│  → NOT by application (OrderApp, PaymentApp, ShippingApp)   │
│                                                             │
│  🔗 INTEGRATED                                              │
│  → Data from MULTIPLE sources combined into ONE view        │
│  → CRM + ERP + Marketing + Sales → Unified warehouse       │
│  → Consistent formats: "M/F" vs "Male/Female" → standardized│
│                                                             │
│  🔒 NON-VOLATILE                                            │
│  → Once data enters, it DOESN'T change                     │
│  → No UPDATE or DELETE in the traditional sense             │
│  → Historical record is preserved forever                   │
│                                                             │
│  ⏰ TIME-VARIANT                                            │
│  → Every record has a time dimension                        │
│  → Tracks how data changes over time                        │
│  → Enables trend analysis, year-over-year comparisons       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Ralph Kimball vs Bill Inmon — The Two Philosophies

```
┌──────────────────────────────┬──────────────────────────────┐
│     BILL INMON (Top-Down)    │    RALPH KIMBALL (Bottom-Up) │
├──────────────────────────────┼──────────────────────────────┤
│                              │                              │
│  Enterprise Data Warehouse   │  Dimensional Data Marts      │
│  (EDW) built FIRST           │  built FIRST, then combined  │
│                              │                              │
│  3NF Normalized warehouse    │  Star Schema (denormalized)  │
│  → Then create data marts    │  → Directly queryable        │
│                              │                              │
│  ┌───────────────┐           │  ┌──────┐ ┌──────┐ ┌──────┐│
│  │  Enterprise   │           │  │Sales │ │HR    │ │Fin.  ││
│  │  Data         │           │  │Mart  │ │Mart  │ │Mart  ││
│  │  Warehouse    │           │  └──┬───┘ └──┬───┘ └──┬───┘│
│  │  (3NF)        │           │     └────────┼────────┘    │
│  └──────┬────────┘           │         Conformed           │
│    ┌────┼────┐               │         Dimensions          │
│  ┌─▼─┐┌─▼─┐┌▼──┐            │         (shared keys)       │
│  │DM1││DM2││DM3│            │                              │
│  └───┘└───┘└───┘             │                              │
│                              │                              │
│  ✅ Single source of truth   │  ✅ Faster to build          │
│  ✅ Consistent across org    │  ✅ Business users love it   │
│  ❌ Takes years to build     │  ❌ Can become inconsistent  │
│  ❌ Complex, expensive       │  ❌ "Data mart chaos" risk   │
│                              │                              │
│  Best for: Large enterprises │  Best for: Agile teams,     │
│  (banks, government)         │  fast delivery               │
└──────────────────────────────┴──────────────────────────────┘
```

> 💡 **Modern Reality**: Most companies use a **hybrid approach** — Kimball-style dimensional modeling within an Inmon-style enterprise architecture. The "debate" is largely settled.

---

## ⭐ Star Schema — The Bread & Butter of Data Warehousing

The Star Schema is **the most widely used** data warehouse design pattern. If you learn one thing from this chapter, learn THIS.

### Anatomy of a Star Schema

```
                        ┌──────────────────┐
                        │  dim_customer    │
                        ├──────────────────┤
                        │ customer_key (PK)│
                        │ customer_id      │
                        │ name             │
                        │ email            │
                        │ age_group        │
                        │ city             │
                        │ state            │
                        │ country          │
                        └────────┬─────────┘
                                 │
┌──────────────────┐    ┌───────┴──────────┐    ┌──────────────────┐
│  dim_product     │    │   fact_sales      │    │  dim_date        │
├──────────────────┤    ├──────────────────┤    ├──────────────────┤
│ product_key (PK) │◄───│ product_key (FK) │    │ date_key    (PK) │
│ product_id       │    │ customer_key (FK)│───►│ full_date        │
│ product_name     │    │ date_key     (FK)│    │ day_of_week      │
│ category         │    │ store_key    (FK)│    │ month            │
│ subcategory      │    │ promo_key    (FK)│    │ quarter          │
│ brand            │    │──────────────────│    │ year             │
│ unit_cost        │    │ quantity_sold    │    │ is_holiday       │
│ unit_price       │    │ unit_price       │    │ fiscal_quarter   │
└──────────────────┘    │ discount_amount  │    └──────────────────┘
                        │ net_amount       │
┌──────────────────┐    │ tax_amount       │    ┌──────────────────┐
│  dim_store       │    │ total_amount     │    │  dim_promotion   │
├──────────────────┤    │ cost_amount      │    ├──────────────────┤
│ store_key   (PK) │◄───│ profit_amount    │    │ promo_key   (PK) │
│ store_id         │    └──────────────────┘───►│ promo_name       │
│ store_name       │                            │ promo_type       │
│ city             │         ↑                  │ discount_pct     │
│ state            │    The STAR shape!         │ start_date       │
│ region           │    Fact in center,         │ end_date         │
│ country          │    Dimensions around       │ channel          │
└──────────────────┘                            └──────────────────┘
```

### Key Concepts

```
FACT TABLE (Center of the star)
├── Contains MEASURABLE business events (transactions, clicks, calls)
├── Has FOREIGN KEYS pointing to dimension tables
├── Contains NUMERIC MEASURES (amount, quantity, cost, profit)
├── Usually the LARGEST table (millions to billions of rows)
├── Grows FAST (new events constantly added)
└── Examples: fact_sales, fact_orders, fact_clicks, fact_calls

DIMENSION TABLE (Points of the star)
├── Contains DESCRIPTIVE ATTRIBUTES (who, what, when, where, how)
├── Has a PRIMARY KEY referenced by fact table
├── Contains TEXT/CATEGORICAL data (name, category, region)
├── Usually SMALL (thousands to millions of rows)
├── Grows SLOWLY (new products, new customers added occasionally)
└── Examples: dim_customer, dim_product, dim_date, dim_store
```

### Why Star Schema Works So Well

```sql
-- The CEO's question becomes TRIVIAL with Star Schema:
-- "Top 10 products by revenue, by region, by quarter, for 3 years"

SELECT 
    p.category,
    s.region,
    d.quarter,
    d.year,
    SUM(f.total_amount) AS revenue,
    RANK() OVER (PARTITION BY s.region, d.quarter ORDER BY SUM(f.total_amount) DESC) AS rank
FROM fact_sales f
JOIN dim_product p    ON f.product_key = p.product_key
JOIN dim_store s      ON f.store_key = s.store_key
JOIN dim_date d       ON f.date_key = d.date_key
WHERE d.year >= 2023
GROUP BY p.category, s.region, d.quarter, d.year
HAVING rank <= 10
ORDER BY d.year, d.quarter, s.region, revenue DESC;

-- ✅ Simple JOINs (1 level deep, no cascading)
-- ✅ Fast execution (star join optimization)
-- ✅ Business users can understand the query
-- ✅ BI tools (Tableau, Power BI) LOVE star schemas
```

### The dim_date Table — Your Secret Weapon

Every warehouse needs a **date dimension**. This is what separates amateurs from professionals:

```sql
CREATE TABLE dim_date (
    date_key          INT PRIMARY KEY,       -- 20260602 (YYYYMMDD format)
    full_date         DATE NOT NULL,         -- '2026-06-02'
    day_of_week       VARCHAR(10),           -- 'Tuesday'
    day_of_week_num   INT,                   -- 3 (1=Mon...7=Sun)
    day_of_month      INT,                   -- 2
    day_of_year       INT,                   -- 153
    week_of_year      INT,                   -- 22
    month_num         INT,                   -- 6
    month_name        VARCHAR(10),           -- 'June'
    month_short       VARCHAR(3),            -- 'Jun'
    quarter           INT,                   -- 2
    quarter_name      VARCHAR(2),            -- 'Q2'
    year              INT,                   -- 2026
    fiscal_quarter    INT,                   -- 1 (if fiscal year starts in April)
    fiscal_year       INT,                   -- 2027
    is_weekend        BOOLEAN,               -- FALSE
    is_holiday        BOOLEAN,               -- FALSE
    holiday_name      VARCHAR(50),           -- NULL or 'Independence Day'
    is_business_day   BOOLEAN,               -- TRUE
    season            VARCHAR(10)            -- 'Summer'
);

-- Pre-populate for 20+ years (just ~7,300 rows)
-- Now ANY date-based analysis is trivial:

-- "Sales on holidays vs non-holidays"
SELECT d.is_holiday, SUM(f.total_amount)
FROM fact_sales f JOIN dim_date d ON f.date_key = d.date_key
GROUP BY d.is_holiday;

-- "Weekend vs Weekday revenue trends by quarter"
SELECT d.quarter_name, d.is_weekend, AVG(f.total_amount)
FROM fact_sales f JOIN dim_date d ON f.date_key = d.date_key
GROUP BY d.quarter_name, d.is_weekend;
```

> 💡 **Pro Tip**: Always use an **integer surrogate key** (`20260602`) for `date_key`, not a DATE type. It's faster for JOINs and partitioning.

---

## ❄️ Snowflake Schema — When Stars Get Complex

The Snowflake Schema is a **normalized version** of the Star Schema. Dimensions are broken into sub-dimensions.

### Star vs Snowflake — Visual

```
STAR SCHEMA (Denormalized Dimensions)         SNOWFLAKE SCHEMA (Normalized Dimensions)
                                              
     ┌──────────┐                                  ┌──────────┐
     │dim_product│                                  │dim_brand │
     │──────────│                                  └────┬─────┘
     │product_key│                                      │
     │name       │                              ┌───────▼──────┐
     │category   │ ◄── All in one table         │dim_subcategory│
     │subcategory│                              └───────┬──────┘
     │brand      │                                      │
     └─────┬─────┘                              ┌───────▼──────┐
           │                                    │dim_category   │
     ┌─────▼─────┐                              └───────┬──────┘
     │fact_sales  │                                     │
     └───────────┘                              ┌───────▼──────┐
                                                │dim_product    │
                                                │(just name+keys│)
                                                └───────┬──────┘
                                                        │
                                                ┌───────▼──────┐
                                                │fact_sales     │
                                                └──────────────┘
```

### Star vs Snowflake — Comparison

| Feature | Star Schema ⭐ | Snowflake Schema ❄️ |
|---------|---------------|---------------------|
| **Dimension structure** | Denormalized (flat) | Normalized (multiple levels) |
| **Number of tables** | Fewer | More |
| **Redundancy** | Some (by design) | Minimal |
| **Query complexity** | Simple (1-level JOINs) | Complex (multi-level JOINs) |
| **Query performance** | ⚡ Faster (fewer JOINs) | 🐌 Slower (more JOINs) |
| **Storage** | Uses more space | Uses less space |
| **ETL complexity** | Simpler | More complex |
| **BI tool compatibility** | ✅ Excellent | 🟡 Good but harder |
| **When to use** | 90% of the time | When storage is critical, or data integrity is paramount |

> 🔥 **Industry Practice**: Star Schema wins **90% of the time**. Storage is cheap. Query speed and simplicity matter more. Use Snowflake Schema only when you have very large dimensions with significant redundancy.

---

## 🕐 Slowly Changing Dimensions (SCD) — The Hard Part

This is where data warehousing gets **really interesting**. Real-world dimensions CHANGE over time:

```
Problem:
- A customer moves from Mumbai → Bangalore
- A product's price changes from ₹999 → ₹1,299
- An employee gets promoted from "Junior" → "Senior"

Question: What do you do with the old data?
- Do past sales show Mumbai or Bangalore?
- Do past revenues use old price or new price?
- Different answers = different SCD types
```

### SCD Type 0 — Retain Original (No Changes)

```
Policy: Once written, NEVER update.
Use Case: Birth date, Original sign-up date, SSN

Customer signs up:    { id: 1, name: "Ritesh", signup_city: "Mumbai" }
Customer moves:       { id: 1, name: "Ritesh", signup_city: "Mumbai" }  ← NO CHANGE

✅ Simplest approach
❌ Doesn't reflect current state
```

### SCD Type 1 — Overwrite (No History)

```
Policy: Just UPDATE the record. History is lost.

BEFORE:  { customer_key: 1, name: "Ritesh", city: "Mumbai"    }
AFTER:   { customer_key: 1, name: "Ritesh", city: "Bangalore" }

✅ Simple to implement
✅ Always shows current state  
❌ History completely LOST
❌ Past sales now show "Bangalore" — WRONG for historical analysis

Use Case: Correcting typos, non-critical attributes
```

### SCD Type 2 — Add New Row (Full History) ⭐ MOST COMMON

```
Policy: Create a NEW row for each change. Old row is "expired."

┌──────────────┬────────┬──────────┬────────────┬────────────┬──────────┐
│ customer_key │ cust_id│ city     │ eff_start  │ eff_end    │is_current│
├──────────────┼────────┼──────────┼────────────┼────────────┼──────────┤
│     1001     │  C-42  │ Mumbai   │ 2020-01-01 │ 2024-06-15 │    N     │
│     1002     │  C-42  │ Bangalore│ 2024-06-16 │ 9999-12-31 │    Y     │
└──────────────┴────────┴──────────┴────────────┴────────────┴──────────┘

Key Observations:
1. customer_key is a SURROGATE key (auto-incrementing, warehouse-assigned)
2. cust_id is the NATURAL/BUSINESS key (same person = same cust_id)
3. eff_end = '9999-12-31' means "current active record"
4. is_current flag for easy filtering

Now:
- Sales from 2023 JOIN to customer_key 1001 → city = "Mumbai"    ✅ CORRECT!
- Sales from 2025 JOIN to customer_key 1002 → city = "Bangalore" ✅ CORRECT!

✅ Full history preserved
✅ Accurate historical reporting  
❌ Table grows larger over time
❌ JOINs need care (use is_current or date range)
```

```sql
-- Query: Current customers only
SELECT * FROM dim_customer WHERE is_current = 'Y';

-- Query: Point-in-time lookup (what was customer's city on 2023-03-15?)
SELECT * FROM dim_customer 
WHERE cust_id = 'C-42' 
  AND '2023-03-15' BETWEEN eff_start AND eff_end;
```

### SCD Type 3 — Add New Column (Limited History)

```
Policy: Add columns for previous values. Only stores ONE level of history.

┌──────────────┬────────┬──────────────┬──────────────┬──────────────┐
│ customer_key │ cust_id│ current_city │ previous_city│ city_changed │
├──────────────┼────────┼──────────────┼──────────────┼──────────────┤
│     1001     │  C-42  │ Bangalore    │ Mumbai       │ 2024-06-16   │
└──────────────┴────────┴──────────────┴──────────────┴──────────────┘

✅ No extra rows (table doesn't grow)
✅ Easy to query current AND previous
❌ Only 1 level of history (what if they moved 3 times?)
❌ Schema changes needed for each tracked attribute

Use Case: When you only need current + immediately previous value
```

### SCD Type 4 — Separate History Table

```
Policy: Keep current data in main table, history in a separate table.

dim_customer (current only):
┌──────────────┬────────┬──────────┐
│ customer_key │ cust_id│ city     │
├──────────────┼────────┼──────────┤
│     1001     │  C-42  │ Bangalore│
└──────────────┴────────┴──────────┘

dim_customer_history (all changes):
┌──────────────┬────────┬──────────┬────────────┬────────────┐
│ history_key  │ cust_id│ city     │ eff_start  │ eff_end    │
├──────────────┼────────┼──────────┼────────────┼────────────┤
│     1        │  C-42  │ Mumbai   │ 2020-01-01 │ 2024-06-15 │
│     2        │  C-42  │ Bangalore│ 2024-06-16 │ 9999-12-31 │
└──────────────┴────────┴──────────┴────────────┴────────────┘

✅ Main dimension stays small and fast
✅ Full history available when needed
❌ More complex ETL (maintain 2 tables)
```

### SCD Type 6 — Hybrid (1 + 2 + 3 Combined)

```
Policy: Add new rows (Type 2) + Add previous columns (Type 3) + Overwrite current (Type 1)

┌──────────────┬────────┬──────────────┬──────────────┬────────────┬────────────┬──────────┐
│ customer_key │ cust_id│ current_city │ historical_  │ eff_start  │ eff_end    │is_current│
│              │        │ (Type 1)     │ city (row)   │            │            │          │
├──────────────┼────────┼──────────────┼──────────────┼────────────┼────────────┼──────────┤
│     1001     │  C-42  │ Bangalore    │ Mumbai       │ 2020-01-01 │ 2024-06-15 │    N     │
│     1002     │  C-42  │ Bangalore    │ Bangalore    │ 2024-06-16 │ 9999-12-31 │    Y     │
└──────────────┴────────┴──────────────┴──────────────┴────────────┴────────────┴──────────┘

Notice: current_city = "Bangalore" in BOTH rows (Type 1 overwrite)
        historical_city = actual city at that time (Type 2 row tracking)

✅ Maximum flexibility
❌ Maximum complexity
```

### SCD Decision Matrix

```
┌────────────────────────────────────────────────┐
│        Which SCD Type Should I Use?            │
├────────────────────────────────────────────────┤
│                                                │
│  Don't need history at all?     → Type 1       │
│  Need full historical accuracy? → Type 2 ⭐    │
│  Only need "current vs last"?   → Type 3       │
│  Need speed + history on demand?→ Type 4       │
│  Need everything?               → Type 6       │
│  Value must NEVER change?       → Type 0       │
│                                                │
│  🔥 Industry Default: TYPE 2                   │
│     80% of all SCD implementations use Type 2  │
│                                                │
└────────────────────────────────────────────────┘
```

---

## 🧊 Fact Table Types — Not All Facts Are Equal

### 1. Transaction Fact Table (Most Common)

```
One row per event/transaction. The most granular type.

┌──────────┬──────────┬──────────┬────────┬──────────┬─────────┐
│ sale_key │ date_key │ prod_key │qty_sold│unit_price│ amount  │
├──────────┼──────────┼──────────┼────────┼──────────┼─────────┤
│ 1        │ 20260601 │ 501      │ 2      │ 999.00   │ 1998.00 │
│ 2        │ 20260601 │ 302      │ 1      │ 499.00   │ 499.00  │
│ 3        │ 20260602 │ 501      │ 5      │ 999.00   │ 4995.00 │
└──────────┴──────────┴──────────┴────────┴──────────┴─────────┘

✅ Most detailed, most flexible
✅ Can be aggregated to any level
❌ Largest table (millions/billions of rows)
```

### 2. Periodic Snapshot Fact Table

```
One row per entity per time period. Captures state at regular intervals.

┌──────────┬────────────┬─────────────┬────────────┬───────────┐
│ acct_key │ month_key  │ balance     │ deposits   │withdrawals│
├──────────┼────────────┼─────────────┼────────────┼───────────┤
│ A-101    │ 202601     │ 50,000.00   │ 15,000.00  │ 5,000.00  │
│ A-101    │ 202602     │ 62,000.00   │ 20,000.00  │ 8,000.00  │
│ A-101    │ 202603     │ 71,000.00   │ 12,000.00  │ 3,000.00  │
└──────────┴────────────┴─────────────┴────────────┴───────────┘

✅ Great for "balance at end of month" style reporting
✅ Predictable size (entities × periods)
❌ Loses individual transaction detail
Use Case: Bank balances, inventory levels, enrollment counts
```

### 3. Accumulating Snapshot Fact Table

```
One row per PROCESS/WORKFLOW. Updated as milestones are reached.

┌──────────┬────────────┬────────────┬───────────┬────────────┬──────────┐
│order_key │ order_date │ ship_date  │deliver_dt │ return_dt  │ amount   │
├──────────┼────────────┼────────────┼───────────┼────────────┼──────────┤
│ ORD-1    │ 2026-06-01 │ 2026-06-02 │2026-06-05│ NULL       │ 1,999.00 │
│ ORD-2    │ 2026-06-01 │ NULL       │ NULL     │ NULL       │ 499.00   │
└──────────┴────────────┴────────────┴───────────┴────────────┴──────────┘

ORD-1: Ordered → Shipped → Delivered (Return pending)
ORD-2: Ordered (Shipping pending)

Row is UPDATED as each milestone occurs (unlike transaction facts).

✅ Perfect for tracking processes with defined stages
✅ Easy to measure time between milestones
❌ Rows get updated (unlike other fact types)
Use Case: Order fulfillment, insurance claims, loan processing
```

### Fact Table Comparison

| Feature | Transaction | Periodic Snapshot | Accumulating Snapshot |
|---------|------------|-------------------|----------------------|
| **Grain** | One row per event | One row per entity/period | One row per process |
| **Updates?** | Insert only | Insert only | Insert + Update |
| **Size** | Largest | Medium | Smallest |
| **Best For** | Sales, clicks, calls | Balances, inventory | Workflows, pipelines |

---

## 🏗️ Data Warehouse Architecture — Layers

A modern data warehouse has distinct layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                     DATA WAREHOUSE LAYERS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  📊 PRESENTATION / SERVING LAYER                          │  │
│  │  • Data Marts (Sales Mart, Finance Mart, HR Mart)         │  │
│  │  • OLAP Cubes (pre-aggregated)                            │  │
│  │  • Semantic layer / BI views                              │  │
│  │  → What analysts and BI tools see                         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                            ▲                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  ⭐ CORE WAREHOUSE LAYER                                  │  │
│  │  • Star/Snowflake schemas                                 │  │
│  │  • Fact + Dimension tables                                │  │
│  │  • Conformed dimensions (shared across marts)             │  │
│  │  • Business rules applied                                 │  │
│  │  → The "single source of truth"                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                            ▲                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  🔧 STAGING / RAW LAYER                                   │  │
│  │  • Raw data as-is from sources                            │  │
│  │  • No transformations (or minimal)                        │  │
│  │  • Temporary holding area                                 │  │
│  │  • Full loads or change data capture (CDC)                │  │
│  │  → Landing zone for ETL/ELT                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                            ▲                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  🌐 SOURCE SYSTEMS                                        │  │
│  │  • OLTP databases (PostgreSQL, Oracle, MySQL)             │  │
│  │  • APIs (Salesforce, Stripe, Google Analytics)            │  │
│  │  • Files (CSV, JSON, Parquet, Avro)                       │  │
│  │  • Streaming (Kafka, Kinesis)                             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📏 Grain — The Most Important Design Decision

> **"Declare the grain, and the rest of the design follows."** — Ralph Kimball

The **grain** defines what a single row in the fact table represents.

```
┌──────────────────────────────────────────────────────────┐
│  Getting the Grain RIGHT vs WRONG                        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ✅ CORRECT GRAIN: "One row per product per transaction" │
│     fact_sales: { sale_id, date, product, customer,      │
│                   quantity, amount }                      │
│     → Can answer ANY question about sales                │
│                                                          │
│  ❌ WRONG GRAIN: "One row per daily product total"       │
│     fact_daily_sales: { date, product, total_quantity,    │
│                         total_amount }                    │
│     → CANNOT answer: "What did customer #42 buy?"        │
│     → CANNOT answer: "What was the average order size?"  │
│     → Information is LOST forever                        │
│                                                          │
│  🔥 RULE: Always choose the LOWEST (most detailed) grain│
│     You can always aggregate UP, never disaggregate DOWN │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 🔗 Surrogate Keys vs Natural Keys

```
NATURAL KEY: The business identifier from the source system
  → customer_id = "CUST-0042" (from CRM)
  → product_sku = "SKU-ABC-123" (from inventory)

SURROGATE KEY: An auto-generated warehouse key (integer)
  → customer_key = 7042 (warehouse-assigned)
  → product_key = 1523 (warehouse-assigned)

┌─────────────────────────────────────────────────────────────┐
│  WHY USE SURROGATE KEYS?                                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Source systems can CHANGE their keys                    │
│     → CRM migrated from "CUST-42" to "C00042"              │
│     → Your warehouse is unaffected (still key = 7042)       │
│                                                             │
│  2. Multiple sources have CONFLICTING keys                  │
│     → CRM customer_id = 100                                 │
│     → ERP customer_id = 100 (DIFFERENT customer!)           │
│     → Surrogate keys keep them separate                     │
│                                                             │
│  3. SCD Type 2 REQUIRES surrogate keys                      │
│     → Same customer_id, multiple rows (Mumbai → Bangalore)  │
│     → Each version gets unique surrogate key                │
│                                                             │
│  4. Integer keys are FASTER for JOINs                       │
│     → JOIN on INT (4 bytes) vs VARCHAR(50) → 10x faster     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧱 Conformed Dimensions — The Glue

When you have multiple fact tables (fact_sales, fact_returns, fact_inventory), they should share the **same dimension tables**:

```
                    ┌──────────────┐
                    │ dim_product   │  ← CONFORMED: Shared by all facts
                    │ dim_date      │
                    │ dim_customer  │
                    │ dim_store     │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐┌────▼──────┐┌────▼──────────┐
        │fact_sales  ││fact_return││fact_inventory  │
        └───────────┘└───────────┘└────────────────┘

Benefits:
✅ "Revenue minus Returns by Product by Region" — ONE consistent answer
✅ Drill across fact tables using shared dimensions
✅ Consistent reporting across the entire organization

Without conformed dimensions:
❌ Sales team says revenue = ₹50 Cr
❌ Finance team says revenue = ₹48 Cr
❌ CEO: "Which number is right?!" 😤
```

---

## 📊 OLAP Cubes — Multi-Dimensional Analysis

OLAP Cubes let users analyze data across multiple dimensions simultaneously:

```
                         OLAP Cube: Sales Data
                         
                         Product ──────────►
                         ┌─────┬─────┬─────┐
                    ╱    │Phone│Laptop│Tab  │   ╱│
                   ╱     ├─────┼─────┼─────┤  ╱ │
           Time   ╱  Q1  │ 10K │ 25K │ 8K  │ ╱  │
              │  ╱   Q2  │ 12K │ 28K │ 9K  │╱   │  Region
              ▼ ╱    Q3  │ 15K │ 30K │ 11K │    │    │
               ┌─────┬─────┬─────┐          │    ▼
               │     │     │     │          │
               └─────┴─────┴─────┘──────────┘

OLAP Operations:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  🔍 SLICE   = Fix one dimension, see 2D cross-section    │
│     "Show me ALL products × ALL regions for Q2 only"     │
│                                                          │
│  🔲 DICE    = Fix multiple dimensions, see sub-cube      │
│     "Phone + Laptop, Q1 + Q2, North + South only"        │
│                                                          │
│  🔭 DRILL DOWN = Go from summary to detail               │
│     Year → Quarter → Month → Day                         │
│     "Why was Q3 revenue high? Let me see by month..."    │
│                                                          │
│  🔝 ROLL UP   = Go from detail to summary               │
│     Day → Month → Quarter → Year                         │
│     "Show me annual totals"                              │
│                                                          │
│  🔄 PIVOT     = Rotate the cube (swap axes)              │
│     Rows=Product, Cols=Time → Rows=Time, Cols=Product    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 🏗️ ETL vs ELT — In the Warehouse Context

```
ETL (Extract, Transform, Load) — Traditional Approach
┌──────────┐    ┌──────────────┐    ┌──────────────┐
│ Extract  │───►│  Transform   │───►│    Load      │
│ from     │    │  (outside    │    │  (into       │
│ sources  │    │   warehouse) │    │   warehouse) │
└──────────┘    └──────────────┘    └──────────────┘
                     ↑ 
        Done in ETL tool (SSIS, Informatica, Talend)
        Before data enters the warehouse

ELT (Extract, Load, Transform) — Modern Approach
┌──────────┐    ┌──────────────┐    ┌──────────────┐
│ Extract  │───►│    Load      │───►│  Transform   │
│ from     │    │  (raw into   │    │  (inside     │
│ sources  │    │   warehouse) │    │   warehouse) │
└──────────┘    └──────────────┘    └──────────────┘
                                         ↑
                         Done inside warehouse using SQL
                         (dbt, Snowflake SQL, BigQuery SQL)
                         Warehouse has massive compute power!

Why ELT is winning:
✅ Cloud warehouses (Snowflake, BigQuery) have MASSIVE compute
✅ Raw data preserved (reprocess anytime)
✅ dbt makes transformations version-controlled and testable
✅ No separate ETL server needed

Why ETL still matters:
✅ Complex transformations (API calls, ML models)
✅ Data quality checks before loading
✅ When warehouse compute is expensive
```

---

## 🌊 Data Lake vs Data Warehouse vs Data Lakehouse

The modern data landscape has evolved:

```
┌──────────────────┬─────────────────────┬──────────────────────┐
│   DATA WAREHOUSE │    DATA LAKE        │   DATA LAKEHOUSE     │
│   (2000s)        │    (2010s)          │   (2020s)            │
├──────────────────┼─────────────────────┼──────────────────────┤
│                  │                     │                      │
│ Structured data  │ All data types      │ All data types       │
│ only (tables)    │ (raw files, JSON,   │ WITH warehouse       │
│                  │  images, logs)      │ reliability          │
│                  │                     │                      │
│ Schema-on-Write  │ Schema-on-Read      │ Schema-on-Read +     │
│ (define before   │ (interpret when     │ Schema enforcement   │
│  loading)        │  querying)          │ (best of both)       │
│                  │                     │                      │
│ ✅ ACID          │ ❌ No ACID          │ ✅ ACID              │
│ ✅ Fast queries  │ ❌ Slow queries     │ ✅ Fast queries      │
│ ✅ Governance    │ ❌ "Data swamp"     │ ✅ Governance        │
│ ❌ Expensive     │ ✅ Cheap storage    │ ✅ Cheap storage     │
│ ❌ Structured    │ ✅ Any format       │ ✅ Any format        │
│    only          │ ❌ Hard to manage   │ ✅ Managed           │
│                  │                     │                      │
│ Tools:           │ Tools:              │ Tools:               │
│ Redshift,        │ S3/ADLS + Hive,     │ Delta Lake, Iceberg, │
│ Snowflake,       │ Hadoop, Presto      │ Hudi + Spark/Trino   │
│ BigQuery         │                     │ Databricks, Snowflake│
│                  │                     │                      │
└──────────────────┴─────────────────────┴──────────────────────┘
```

> 🔥 **The Lakehouse is the future**. It combines the reliability of warehouses with the flexibility and cost of data lakes. Delta Lake (Databricks) and Apache Iceberg are leading this revolution.

---

## 📐 Data Warehouse Measures — Additive, Semi-Additive, Non-Additive

Understanding measure types prevents **embarrassing calculation errors**:

```
┌──────────────────────────────────────────────────────────────┐
│  ADDITIVE MEASURES (Can be summed across ALL dimensions)     │
│  → revenue, quantity_sold, cost, profit, discount_amount     │
│  → SUM(revenue) across products ✅                           │
│  → SUM(revenue) across time ✅                               │
│  → SUM(revenue) across stores ✅                             │
│                                                              │
│  SEMI-ADDITIVE MEASURES (Can be summed across SOME dims)     │
│  → account_balance, inventory_count, headcount               │
│  → SUM(balance) across accounts ✅ (total bank assets)       │
│  → SUM(balance) across time ❌ (summing Jan + Feb balance    │
│    is MEANINGLESS — use AVG or END-OF-PERIOD instead)        │
│                                                              │
│  NON-ADDITIVE MEASURES (CANNOT be summed across any dim)     │
│  → unit_price, temperature, ratio, percentage                │
│  → SUM(unit_price) across products ❌ MEANINGLESS            │
│  → SUM(ratio) across time ❌ MEANINGLESS                     │
│  → Use AVG, MIN, MAX, or weighted calculations instead       │
│                                                              │
└──────────────────────────────────────────────────────────────┘

🔥 INTERVIEW TRAP:
Q: "Sum up all account balances for the year"
A: "That's a semi-additive measure. I'd use the END-OF-MONTH 
    balance, not sum across months. Summing monthly balances 
    would give a meaningless number."
```

---

## 🎯 Data Mart — Focused Subsets

```
A Data Mart is a SUBSET of the warehouse focused on one business area.

                    ┌──────────────────────────┐
                    │    DATA WAREHOUSE         │
                    │    (Enterprise-wide)      │
                    └───────────┬──────────────┘
                    ┌───────────┼───────────┐
              ┌─────▼─────┐┌───▼────┐┌─────▼─────┐
              │ Sales Mart ││HR Mart ││Finance    │
              │            ││        ││Mart       │
              │• Revenue   ││• Hiring││• P&L      │
              │• Products  ││• Salary││• Budget   │
              │• Regions   ││• Dept  ││• Expenses │
              └───────────┘└────────┘└───────────┘
                    ↑           ↑           ↑
              Sales team    HR team    Finance team

Types:
• DEPENDENT Data Mart:  Created FROM the warehouse (recommended)
• INDEPENDENT Data Mart: Created directly from source systems (risky)
```

---

## 🏆 Real-World Example — Complete Warehouse Design

Let's design a warehouse for an e-commerce company:

```sql
-- ==========================================
-- DIMENSION TABLES
-- ==========================================

CREATE TABLE dim_customer (
    customer_key    INT PRIMARY KEY,          -- Surrogate key
    customer_id     VARCHAR(20),              -- Natural key
    first_name      VARCHAR(50),
    last_name       VARCHAR(50),
    email           VARCHAR(100),
    phone           VARCHAR(20),
    age_group       VARCHAR(20),              -- '18-25', '26-35', etc.
    gender          VARCHAR(10),
    city            VARCHAR(50),
    state           VARCHAR(50),
    country         VARCHAR(50),
    customer_tier   VARCHAR(20),              -- 'Bronze','Silver','Gold','Platinum'
    signup_date     DATE,
    effective_start DATE,                     -- SCD Type 2
    effective_end   DATE,                     -- SCD Type 2  
    is_current      CHAR(1) DEFAULT 'Y'      -- SCD Type 2
);

CREATE TABLE dim_product (
    product_key     INT PRIMARY KEY,
    product_id      VARCHAR(20),
    product_name    VARCHAR(200),
    category        VARCHAR(50),              -- 'Electronics'
    subcategory     VARCHAR(50),              -- 'Smartphones'
    brand           VARCHAR(50),              -- 'Samsung'
    unit_cost       DECIMAL(10,2),
    unit_price      DECIMAL(10,2),
    weight_kg       DECIMAL(6,2),
    is_active       CHAR(1),
    launch_date     DATE,
    effective_start DATE,
    effective_end   DATE,
    is_current      CHAR(1) DEFAULT 'Y'
);

CREATE TABLE dim_date (
    date_key        INT PRIMARY KEY,          -- YYYYMMDD
    full_date       DATE,
    day_name        VARCHAR(10),
    day_of_month    INT,
    day_of_year     INT,
    week_of_year    INT,
    month_num       INT,
    month_name      VARCHAR(10),
    quarter         INT,
    year            INT,
    fiscal_quarter  INT,
    fiscal_year     INT,
    is_weekend      BOOLEAN,
    is_holiday      BOOLEAN,
    holiday_name    VARCHAR(50)
);

-- ==========================================
-- FACT TABLES
-- ==========================================

CREATE TABLE fact_orders (
    order_key       BIGINT PRIMARY KEY,
    date_key        INT REFERENCES dim_date(date_key),
    customer_key    INT REFERENCES dim_customer(customer_key),
    product_key     INT REFERENCES dim_product(product_key),
    -- Degenerate dimension (no separate table needed)
    order_number    VARCHAR(20),              
    -- Measures
    quantity        INT,
    unit_price      DECIMAL(10,2),
    discount_pct    DECIMAL(5,2),
    discount_amount DECIMAL(10,2),
    tax_amount      DECIMAL(10,2),
    shipping_cost   DECIMAL(10,2),
    net_amount      DECIMAL(12,2),
    gross_amount    DECIMAL(12,2),
    cost_amount     DECIMAL(12,2),
    profit_amount   DECIMAL(12,2)
);

-- Now the CEO's query takes 2 seconds, not 45 minutes! 🚀
```

---

## 🧪 Quick Knowledge Check

```
Q1: Your production PostgreSQL is slow because analysts run heavy reports.
    What do you do?
A1: Build a data warehouse. Separate OLTP (transactions) from OLAP (analytics).

Q2: Difference between fact and dimension tables?
A2: Facts = measurable events (numbers). Dimensions = descriptive context (who/what/when/where).

Q3: A customer changes their address. How do you handle it for historical accuracy?
A3: SCD Type 2 — create a new row with new address, expire the old row.

Q4: Why column-oriented storage for analytics?
A4: Analytical queries read few columns but many rows. Column storage reads only needed 
    columns (skipping 80%+ of data) and compresses better (same data types together).

Q5: What is grain?
A5: What one row in the fact table represents. E.g., "one row per product per order line."
    Always choose the lowest grain possible.

Q6: Can you SUM(account_balance) across months?
A6: NO! It's semi-additive. Use AVG or end-of-period value instead.
```

---

## 🗺️ Chapter Summary

```
┌────────────────────────────────────────────────────────┐
│  DATA WAREHOUSING CONCEPTS — KEY TAKEAWAYS             │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ✅ OLTP = operations (fast writes, normalized)        │
│  ✅ OLAP = analytics (fast reads, denormalized)        │
│  ✅ Star Schema = fact (center) + dims (points)        │
│  ✅ Snowflake Schema = normalized star (rarely needed) │
│  ✅ SCD Type 2 = industry standard for history         │
│  ✅ Grain = what one row represents (go lowest!)       │
│  ✅ Surrogate keys = warehouse-assigned integers       │
│  ✅ Conformed dimensions = shared across fact tables    │
│  ✅ ETL → traditional, ELT → modern (dbt era)         │
│  ✅ Warehouse → Lake → Lakehouse (evolution)           │
│  ✅ Additive/Semi-additive/Non-additive measures       │
│                                                        │
│  🔥 INTERVIEW ESSENTIALS:                              │
│     Star Schema, SCD Type 2, Grain, OLTP vs OLAP,     │
│     Fact types, Measure types, ETL vs ELT              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [6.2 — Amazon Redshift →](./02-Redshift.md)

---

*"A data warehouse is not a product you buy. It's a discipline you practice."*
