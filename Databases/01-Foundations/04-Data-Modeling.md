# 1.4 — Data Modeling — The Art of Designing Data 🟢⭐

> **"Give me six hours to chop down a tree and I'll spend the first four sharpening the axe." — Abraham Lincoln**
> The same applies to databases: spend time designing your schema right, or spend forever fixing it later.

---

## 📌 What You'll Learn

- What data modeling is and why it's the **most important** database skill
- The **three stages** of data modeling (Conceptual → Logical → Physical)
- **ER Diagrams** — the universal language of database design
- **Normalization** deep dive (1NF → 2NF → 3NF → BCNF → 4NF → 5NF)
- **Denormalization** — when to break the rules
- **Star Schema & Snowflake Schema** for data warehousing
- Real-world examples throughout

---

## 1. What is Data Modeling?

### Definition

> **Data Modeling** is the process of creating a visual representation of how data is structured, stored, and accessed in a database. It defines entities, their attributes, and the relationships between them.

### Why It's the Most Important Skill

```
┌────────────────────────────────────────────────────────────────┐
│  BAD data model + GOOD code     = Constant pain, slow queries  │
│  GOOD data model + AVERAGE code = Everything works smoothly    │
│                                                                │
│  → The data model OUTLIVES your application code               │
│  → Refactoring code is easy. Migrating data is HARD.           │
│  → A bad schema decision on day 1 haunts you for YEARS.        │
└────────────────────────────────────────────────────────────────┘
```

### Real Consequences of Bad Data Modeling

| Bad Decision | Consequence |
|-------------|-------------|
| No normalization | Same customer address stored 50 times → one update misses 3 copies |
| Over-normalization | Simple query needs 12 JOINs → page takes 8 seconds to load |
| Wrong data types | Storing phone numbers as INT → lose leading zeros, can't store "+91" |
| Missing indexes | Full table scan on 100M rows → query takes 45 seconds |
| No foreign keys | Orphaned records → order exists but customer doesn't |

---

## 2. Three Stages of Data Modeling

Data modeling is done in three progressive stages, each adding more detail:

```
STAGE 1              STAGE 2              STAGE 3
CONCEPTUAL           LOGICAL              PHYSICAL
(What?)              (How?)               (Where?)
                     
  ┌──────────┐      ┌──────────────┐     ┌──────────────────┐
  │ Customer │      │ Customer      │     │ CREATE TABLE      │
  │ places   │      │ - id: INT     │     │   customers (     │
  │ Order    │      │ - name: STR   │     │   id INT PK,      │
  │          │      │ - email: STR  │     │   name VARCHAR(50),│
  └──────────┘      │               │     │   email VARCHAR(100│
                     │ Order         │     │     UNIQUE,        │
  "Business          │ - id: INT     │     │   INDEX idx_email  │
   concepts"         │ - date: DATE  │     │ ) TABLESPACE ts1; │
                     │ - customer_id │     │                    │
                     └──────────────┘     └──────────────────┘
                     
  "Detailed           "Exact DB-specific
   structure"          implementation"
```

### Stage Details

| Stage | Who Creates It | Contains | Tools |
|-------|---------------|----------|-------|
| **Conceptual** | Business analyst + Architect | Entities, Relationships (high-level) | Whiteboard, Lucidchart |
| **Logical** | Database designer | Tables, Columns, Data types, Keys, Constraints | ER diagramming tools |
| **Physical** | DBA / Developer | SQL DDL, Indexes, Partitions, Tablespaces, Storage | SQL scripts, DB-specific tools |

---

## 3. Entity-Relationship (ER) Diagrams

### The Universal Language of Database Design

ER diagrams visually represent entities, their attributes, and relationships. Every database professional must read and draw these.

### Core Components

```
┌────────────────────────────────────────────────────────────────┐
│                    ER DIAGRAM ELEMENTS                          │
│                                                                │
│  ┌───────────┐                                                 │
│  │  ENTITY   │  = A "thing" (noun): Customer, Order, Product  │
│  │ (Rectangle)│                                                │
│  └───────────┘                                                 │
│                                                                │
│  ○ Attribute    = Property of an entity: name, email, price    │
│                                                                │
│  ◆ Primary Key  = Unique identifier: customer_id               │
│    (underlined)                                                │
│                                                                │
│  ──────────── = Relationship line connecting entities          │
│                                                                │
│  1 ──── M     = Cardinality: "one to many"                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Cardinality — How Entities Relate

```
ONE-TO-ONE (1:1)
  One person has one passport.
  ┌──────────┐  1        1  ┌──────────┐
  │  Person   │──────────────│ Passport  │
  └──────────┘              └──────────┘

ONE-TO-MANY (1:M) — MOST COMMON
  One department has many employees.
  ┌──────────────┐  1        M  ┌──────────┐
  │  Department   │──────────────│ Employee  │
  └──────────────┘              └──────────┘

MANY-TO-MANY (M:N)
  Students enroll in many courses; courses have many students.
  ┌──────────┐  M        N  ┌──────────┐
  │  Student  │──────────────│  Course   │
  └──────────┘              └──────────┘
  
  → In relational DBs, M:N requires a JUNCTION TABLE:
  ┌──────────┐  1    M  ┌──────────────┐  M    1  ┌──────────┐
  │  Student  │──────────│ Enrollment    │──────────│  Course   │
  └──────────┘          │ student_id FK │          └──────────┘
                         │ course_id  FK │
                         │ enrolled_date │
                         └──────────────┘
```

### Full ER Diagram Example — E-Commerce

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   CUSTOMER    │       │    ORDER      │       │  ORDER_ITEM   │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ ◆ customer_id│──┐    │ ◆ order_id   │──┐    │ ◆ item_id    │
│   first_name │  │    │   order_date │  │    │   quantity   │
│   last_name  │  │    │   total      │  │    │   unit_price │
│   email      │  │    │   status     │  │    │   subtotal   │
│   phone      │  └──1→│   customer_id│  └──1→│   order_id   │
│   created_at │       │   address_id │       │   product_id │←─┐
└──────────────┘       └──────────────┘       └──────────────┘  │
                              │                                   │
                              │ M                                 │
                              ▼ 1                                 │
                       ┌──────────────┐       ┌──────────────┐   │
                       │   ADDRESS     │       │   PRODUCT     │   │
                       ├──────────────┤       ├──────────────┤   │
                       │ ◆ address_id │       │ ◆ product_id │───┘
                       │   street     │       │   name       │
                       │   city       │       │   price      │
                       │   state      │       │   category_id│──┐
                       │   zip_code   │       │   stock_qty  │  │
                       │   country    │       └──────────────┘  │
                       └──────────────┘                          │
                                                                 │
                                              ┌──────────────┐  │
                                              │   CATEGORY    │  │
                                              ├──────────────┤  │
                                              │ ◆ category_id│←─┘
                                              │   name       │
                                              │   parent_id  │←─(self-reference)
                                              └──────────────┘

RELATIONSHIPS:
  Customer  ─1:M─→  Order       (one customer, many orders)
  Order     ─1:M─→  OrderItem   (one order, many items)
  Product   ─1:M─→  OrderItem   (one product in many order items)
  Category  ─1:M─→  Product     (one category, many products)
  Address   ─1:M─→  Order       (one address for many orders)
  Category  ─1:M─→  Category    (self-referencing: parent-child)
```

---

## 4. Keys — The Glue of Relational Databases

### Types of Keys

```
┌──────────────────────────────────────────────────────────────┐
│                      TYPES OF KEYS                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SUPER KEY                                                   │
│  Any set of columns that uniquely identifies a row           │
│  Ex: {id}, {id, name}, {id, name, email} — all are super    │
│                                                              │
│  CANDIDATE KEY                                               │
│  Minimal super key (remove any column → no longer unique)    │
│  Ex: {id}, {email} — both uniquely identify, both minimal   │
│                                                              │
│  PRIMARY KEY (PK) ⭐                                         │
│  The candidate key chosen as THE main identifier             │
│  Ex: id (chosen as PK)                                       │
│  Rules: NOT NULL, UNIQUE, One per table, Immutable ideally  │
│                                                              │
│  ALTERNATE KEY                                               │
│  Candidate keys that were NOT chosen as PK                   │
│  Ex: email (candidate but not chosen as PK)                 │
│  Often implemented as UNIQUE constraint                     │
│                                                              │
│  FOREIGN KEY (FK) ⭐                                         │
│  Column in one table that references PK of another table     │
│  Ex: orders.customer_id → customers.id                      │
│  Enforces referential integrity                             │
│                                                              │
│  COMPOSITE KEY                                               │
│  PK made of TWO or more columns                             │
│  Ex: enrollment(student_id, course_id)                      │
│                                                              │
│  SURROGATE KEY                                               │
│  System-generated artificial key (no business meaning)       │
│  Ex: auto-increment id, UUID                                │
│                                                              │
│  NATURAL KEY                                                 │
│  Key with real-world meaning                                │
│  Ex: SSN, ISBN, email                                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Surrogate vs Natural Key — The Debate

| Factor | Surrogate Key (id INT) | Natural Key (email, SSN) |
|--------|----------------------|-------------------------|
| **Stability** | ✅ Never changes | ❌ Email can change |
| **Size** | ✅ Small (4-8 bytes) | ❌ Can be large (varchar) |
| **Performance** | ✅ Fast JOINs | ❌ Slower JOINs on strings |
| **Business Meaning** | ❌ None | ✅ Meaningful |
| **Uniqueness** | ✅ Guaranteed | ⚠️ May have edge cases |
| **Recommendation** | ⭐ **Use surrogate for PK** | Use as UNIQUE constraint |

> 💡 **Best Practice**: Use **surrogate keys** (auto-increment or UUID) as primary keys. Keep natural keys as UNIQUE constraints.

---

## 5. Normalization — Eliminating Redundancy

### What is Normalization?

> **Normalization** is the process of organizing a database to reduce **redundancy** and **dependency** by dividing data into smaller, related tables.

### Why Normalize?

```
BEFORE NORMALIZATION (One big messy table):
┌───────┬──────────┬────────────┬─────────────┬──────────┬──────────┐
│ EmpID │ EmpName  │ Dept       │ DeptHead    │ Project  │ ProjLead │
├───────┼──────────┼────────────┼─────────────┼──────────┼──────────┤
│ 101   │ Rahul    │ Engineering│ Dr. Sharma  │ Alpha    │ Rahul    │
│ 101   │ Rahul    │ Engineering│ Dr. Sharma  │ Beta     │ Priya    │
│ 102   │ Priya    │ Engineering│ Dr. Sharma  │ Beta     │ Priya    │
│ 103   │ John     │ Marketing  │ Mr. Patel   │ Gamma    │ John     │
│ 103   │ John     │ Marketing  │ Mr. Patel   │ Delta    │ Sarah    │
└───────┴──────────┴────────────┴─────────────┴──────────┴──────────┘

PROBLEMS:
❌ Rahul's department "Engineering" stored TWICE
❌ "Dr. Sharma" stored THREE times
❌ If Dr. Sharma changes → update 3 rows (miss one = inconsistency!)
❌ Delete John's projects → lose the fact that John is in Marketing
❌ Can't add a new department without an employee (insertion anomaly)
```

### The Three Anomalies Normalization Prevents

| Anomaly | Description | Example |
|---------|------------|---------|
| **Update Anomaly** | Same fact stored multiple times → update misses one | Change dept head → must update all rows for that dept |
| **Insertion Anomaly** | Can't insert data without unrelated data | Can't add new department until an employee is assigned |
| **Deletion Anomaly** | Deleting data unintentionally removes other facts | Delete last employee in Marketing → lose the department itself |

---

### Normal Forms — Step by Step

#### First Normal Form (1NF) — Atomic Values

> **Rule**: Every column must contain **atomic** (indivisible) values. No repeating groups, no arrays, no comma-separated values.

```
❌ VIOLATES 1NF (multi-valued column):
┌───────┬──────────┬─────────────────────────┐
│ EmpID │ Name     │ PhoneNumbers             │
├───────┼──────────┼─────────────────────────┤
│ 101   │ Rahul    │ 9876543210, 9123456789  │  ← TWO values in one cell!
│ 102   │ Priya    │ 8765432109              │
└───────┴──────────┴─────────────────────────┘

✅ 1NF (atomic values — separate rows):
┌───────┬──────────┬──────────────┐
│ EmpID │ Name     │ PhoneNumber  │
├───────┼──────────┼──────────────┤
│ 101   │ Rahul    │ 9876543210   │  ← ONE value per cell
│ 101   │ Rahul    │ 9123456789   │  ← Separate row for 2nd number
│ 102   │ Priya    │ 8765432109   │
└───────┴──────────┴──────────────┘

OR BETTER (separate table):
employees              employee_phones
┌───────┬──────────┐   ┌───────┬──────────────┐
│ EmpID │ Name     │   │ EmpID │ PhoneNumber  │
├───────┼──────────┤   ├───────┼──────────────┤
│ 101   │ Rahul    │   │ 101   │ 9876543210   │
│ 102   │ Priya    │   │ 101   │ 9123456789   │
└───────┴──────────┘   │ 102   │ 8765432109   │
                        └───────┴──────────────┘
```

**1NF Checklist:**
- [ ] Each column has atomic values (no lists, no comma-separated)
- [ ] Each row is unique (has a primary key)
- [ ] No repeating groups of columns (phone1, phone2, phone3)

---

#### Second Normal Form (2NF) — No Partial Dependencies

> **Rule**: Must be in 1NF + every non-key column must depend on the **entire** primary key (not just part of it).

> ⚠️ 2NF only applies when you have a **composite primary key**.

```
❌ VIOLATES 2NF:
PK = (StudentID, CourseID)

┌───────────┬──────────┬─────────────┬────────────┬───────┐
│ StudentID │ CourseID │ StudentName │ CourseName │ Grade │
├───────────┼──────────┼─────────────┼────────────┼───────┤
│ S1        │ C101     │ Rahul       │ Math       │ A     │
│ S1        │ C102     │ Rahul       │ Physics    │ B     │
│ S2        │ C101     │ Priya       │ Math       │ A+    │
└───────────┴──────────┴─────────────┴────────────┴───────┘

PROBLEM:
• StudentName depends only on StudentID (partial dependency!)
• CourseName depends only on CourseID (partial dependency!)
• Only Grade depends on BOTH StudentID + CourseID

✅ 2NF (remove partial dependencies):

students               courses                enrollments
┌───────────┬─────────┐ ┌──────────┬───────────┐ ┌───────────┬──────────┬───────┐
│ StudentID │ Name    │ │ CourseID │ CourseName│ │ StudentID │ CourseID │ Grade │
├───────────┼─────────┤ ├──────────┼───────────┤ ├───────────┼──────────┼───────┤
│ S1        │ Rahul   │ │ C101     │ Math      │ │ S1        │ C101     │ A     │
│ S2        │ Priya   │ │ C102     │ Physics   │ │ S1        │ C102     │ B     │
└───────────┴─────────┘ └──────────┴───────────┘ │ S2        │ C101     │ A+    │
                                                   └───────────┴──────────┴───────┘
```

**2NF Checklist:**
- [ ] In 1NF
- [ ] No non-key column depends on only PART of a composite PK

---

#### Third Normal Form (3NF) — No Transitive Dependencies

> **Rule**: Must be in 2NF + no non-key column depends on another non-key column.
> In other words: every non-key column must depend **directly** on the primary key, not through another column.

```
❌ VIOLATES 3NF:
┌───────┬──────────┬───────────┬──────────────┐
│ EmpID │ EmpName  │ DeptID    │ DeptName     │
├───────┼──────────┼───────────┼──────────────┤
│ 101   │ Rahul    │ D1        │ Engineering  │
│ 102   │ Priya    │ D1        │ Engineering  │
│ 103   │ John     │ D2        │ Marketing    │
└───────┴──────────┴───────────┴──────────────┘

PROBLEM:
  EmpID → DeptID → DeptName (transitive dependency!)
  DeptName depends on DeptID, NOT directly on EmpID
  If Engineering changes name → update EVERY employee row

✅ 3NF (remove transitive dependencies):

employees                    departments
┌───────┬──────────┬────────┐ ┌────────┬──────────────┐
│ EmpID │ EmpName  │ DeptID │ │ DeptID │ DeptName     │
├───────┼──────────┼────────┤ ├────────┼──────────────┤
│ 101   │ Rahul    │ D1     │ │ D1     │ Engineering  │
│ 102   │ Priya    │ D1     │ │ D2     │ Marketing    │
│ 103   │ John     │ D2     │ └────────┴──────────────┘
└───────┴──────────┴────────┘
  Now "Engineering" is stored ONCE. Change it in ONE place.
```

**3NF Checklist:**
- [ ] In 2NF
- [ ] No non-key column depends on another non-key column
- [ ] Every fact is stored in exactly ONE place

> 💡 **The Mantra**: *"Every non-key column must provide a fact about the key, the whole key, and nothing but the key — so help me Codd."*

---

#### Boyce-Codd Normal Form (BCNF) — Stricter 3NF

> **Rule**: For every functional dependency X → Y, X must be a **superkey**.
> (In 3NF, Y just can't be a non-key attribute depending on another non-key. BCNF is stricter.)

```
❌ VIOLATES BCNF (but satisfies 3NF):

Scenario: Students take courses, taught by professors.
Each professor teaches only ONE course.
Each course can be taught by multiple professors.
A student picks ONE professor per course.

┌──────────┬──────────┬───────────┐
│ Student  │ Course   │ Professor │
├──────────┼──────────┼───────────┤
│ Rahul    │ Math     │ Prof. A   │
│ Rahul    │ Physics  │ Prof. B   │
│ Priya    │ Math     │ Prof. C   │
└──────────┴──────────┴───────────┘
PK = (Student, Course)

PROBLEM:
  Professor → Course  (Prof. A always teaches Math)
  But Professor is NOT a superkey!
  This violates BCNF.

✅ BCNF:

student_professors              professor_courses
┌──────────┬───────────┐       ┌───────────┬──────────┐
│ Student  │ Professor │       │ Professor │ Course   │
├──────────┼───────────┤       ├───────────┼──────────┤
│ Rahul    │ Prof. A   │       │ Prof. A   │ Math     │
│ Rahul    │ Prof. B   │       │ Prof. B   │ Physics  │
│ Priya    │ Prof. C   │       │ Prof. C   │ Math     │
└──────────┴───────────┘       └───────────┴──────────┘
```

---

#### 4NF and 5NF — Quick Overview

```
4NF — No Multi-Valued Dependencies
  If an entity has two independent multi-valued facts,
  they must be in separate tables.

  Example: An employee can have multiple skills AND multiple hobbies.
  Skills and hobbies are INDEPENDENT of each other.
  ❌ One table with (EmpID, Skill, Hobby) → cartesian explosion
  ✅ Two tables: (EmpID, Skill) and (EmpID, Hobby)

5NF — No Join Dependencies
  The table cannot be decomposed into smaller tables
  and then reconstructed by joining without loss.
  Very theoretical. Rarely encountered in practice.
```

### Normalization Summary Table

| Normal Form | Rule | Eliminates |
|-------------|------|-----------|
| **1NF** | Atomic values, no repeating groups | Multi-valued columns |
| **2NF** | No partial dependencies (on part of composite PK) | Partial dependency |
| **3NF** | No transitive dependencies | Transitive dependency |
| **BCNF** | Every determinant is a superkey | Non-superkey determinants |
| **4NF** | No multi-valued dependencies | Independent multi-valued facts |
| **5NF** | No join dependencies | Lossy decomposition |

### How Far Should You Normalize?

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  🟢 OLTP Systems (transactional):                          │
│     → Normalize to 3NF or BCNF                             │
│     → Data integrity is critical                           │
│     → Example: Banking, E-commerce, ERP                    │
│                                                             │
│  🟡 General Web Apps:                                       │
│     → 3NF is usually perfect                               │
│     → Denormalize specific hot paths if needed             │
│                                                             │
│  🔴 OLAP Systems (analytical/reporting):                    │
│     → Denormalize aggressively (Star/Snowflake schema)     │
│     → Read performance > Write integrity                   │
│     → Example: Data Warehouses, Dashboards                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Denormalization — Breaking the Rules for Performance

### What is Denormalization?

> **Denormalization** is the intentional addition of redundancy to a normalized database to improve **read performance** at the cost of more complex writes.

### When to Denormalize

```
Normalized (3NF):
  Getting a customer's order with product names requires:
  SELECT c.name, o.order_date, p.product_name, oi.quantity
  FROM customers c
  JOIN orders o ON c.id = o.customer_id
  JOIN order_items oi ON o.id = oi.order_id
  JOIN products p ON oi.product_id = p.id
  WHERE c.id = 1001;
  → 4 tables, 3 JOINs

Denormalized:
  Store product_name directly in order_items table
  SELECT c.name, o.order_date, oi.product_name, oi.quantity
  FROM customers c
  JOIN orders o ON c.id = o.customer_id
  JOIN order_items oi ON o.id = oi.order_id
  WHERE c.id = 1001;
  → 3 tables, 2 JOINs, FASTER!

TRADE-OFF:
  ✅ Faster reads (fewer JOINs)
  ❌ If product name changes → must update everywhere
  ❌ Uses more storage (name stored many times)
```

### Common Denormalization Techniques

| Technique | How | When |
|-----------|-----|------|
| **Duplicated columns** | Copy a column from related table | Frequently joined column |
| **Pre-computed aggregates** | Store COUNT, SUM, AVG in parent table | Dashboard metrics |
| **Merged tables** | Combine 1:1 related tables | Always accessed together |
| **Cached derived columns** | Store `full_name` = first + last | Avoid runtime concatenation |
| **Materialized views** | Pre-computed query results | Complex reports |

---

## 7. Star Schema & Snowflake Schema (Data Warehousing)

### Star Schema ⭐

> The **Star Schema** is the most common data warehouse design. One central **fact table** surrounded by **dimension tables**, forming a star shape.

```
                         ┌──────────────┐
                         │  dim_date     │
                         │ date_key (PK) │
                         │ full_date     │
                         │ year          │
                         │ quarter       │
                         │ month         │
                         │ day_of_week   │
                         └──────┬───────┘
                                │
    ┌──────────────┐    ┌──────┴───────┐    ┌──────────────┐
    │ dim_product   │    │  FACT_SALES   │    │ dim_customer  │
    │ product_key   │◄───│ date_key (FK) │───►│ customer_key  │
    │ product_name  │    │ product_key   │    │ customer_name │
    │ category      │    │ customer_key  │    │ city          │
    │ brand         │    │ store_key     │    │ segment       │
    │ price         │    │               │    └──────────────┘
    └──────────────┘    │ ── MEASURES ──│
                         │ quantity      │
                         │ revenue       │    ┌──────────────┐
                         │ discount      │───►│ dim_store     │
                         │ profit        │    │ store_key     │
                         └──────────────┘    │ store_name    │
                                              │ city          │
                                              │ region        │
                                              └──────────────┘

FACT TABLE: Contains measurable business events (sales, clicks, transactions)
  → Stores numbers: quantity, revenue, profit
  → Foreign keys to dimension tables
  → Can have BILLIONS of rows

DIMENSION TABLES: Descriptive attributes for filtering/grouping
  → WHO (customer), WHAT (product), WHEN (date), WHERE (store)
  → Usually denormalized (flat)
  → Typically thousands to millions of rows
```

### Snowflake Schema ❄️

> Like a Star Schema, but dimension tables are **normalized** (split into sub-dimensions).

```
                           ┌──────────┐
                           │ dim_date  │
                           └────┬─────┘
                                │
    ┌──────────┐   ┌────────────┴──────────┐   ┌────────────┐
    │ dim_brand │   │      FACT_SALES        │   │ dim_city    │
    └─────┬────┘   └──────────┬────────────┘   └──────┬─────┘
          │                    │                        │
    ┌─────┴──────┐            │                  ┌─────┴──────┐
    │dim_product │◄───────────┘                  │dim_customer│
    └────────────┘                               └────────────┘

  → Dimension tables are NORMALIZED (brand separated from product)
  → More JOINs but less storage
  → Rarely used today — Star Schema is preferred
```

### Star vs Snowflake

| Factor | Star Schema ⭐ | Snowflake Schema ❄️ |
|--------|---------------|-------------------|
| **Query Speed** | ✅ Faster (fewer JOINs) | ❌ Slower (more JOINs) |
| **Storage** | ❌ More (denormalized dims) | ✅ Less (normalized dims) |
| **Simplicity** | ✅ Simple | ❌ Complex |
| **ETL Complexity** | ✅ Simpler | ❌ More complex |
| **Industry Preference** | ⭐ **Dominant** | Niche use |

---

## 8. Functional Dependencies — The Theory Behind Normalization

### What is a Functional Dependency?

> If knowing the value of column(s) **X** uniquely determines the value of column **Y**, we say **X → Y** ("X functionally determines Y").

```
EXAMPLES:

  employee_id → employee_name     (knowing ID determines name)
  employee_id → salary            (knowing ID determines salary)
  employee_id → department_id     (knowing ID determines department)
  
  isbn → book_title               (knowing ISBN determines title)
  isbn → publisher                (knowing ISBN determines publisher)
  
  (student_id, course_id) → grade (knowing BOTH determines grade)
  
DOES NOT WORK:
  name → employee_id              ❌ (two employees can have same name)
  department_id → employee_id     ❌ (one dept has many employees)
```

### Types of Functional Dependencies

```
FULL FD:
  (A, B) → C where NEITHER A alone NOR B alone determines C
  Example: (student_id, course_id) → grade

PARTIAL FD (violates 2NF):
  (A, B) → C where A alone determines C (B is unnecessary)
  Example: (student_id, course_id) → student_name
           student_id alone determines student_name!

TRANSITIVE FD (violates 3NF):
  A → B → C  (A determines B, B determines C)
  Example: emp_id → dept_id → dept_name
           emp_id determines dept_id,
           dept_id determines dept_name
           Therefore emp_id transitively determines dept_name
```

---

## 9. Real-World Data Modeling Tips

### The 10 Golden Rules

```
┌────────────────────────────────────────────────────────────────────┐
│  GOLDEN RULES OF DATA MODELING                                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. START WITH ENTITIES, NOT TABLES                                │
│     Think "what things exist?" before "what columns do I need?"   │
│                                                                    │
│  2. USE SURROGATE KEYS                                             │
│     Auto-increment or UUID as PK. Natural keys as UNIQUE.         │
│                                                                    │
│  3. NAME THINGS CONSISTENTLY                                       │
│     snake_case, singular table names (order not orders),           │
│     _id suffix for keys                                           │
│                                                                    │
│  4. NORMALIZE TO 3NF FIRST                                         │
│     Then selectively denormalize for proven performance needs      │
│                                                                    │
│  5. ALWAYS ADD FOREIGN KEYS                                        │
│     Even if your app "handles it" — the DB is the last defense    │
│                                                                    │
│  6. USE PROPER DATA TYPES                                          │
│     Don't store dates as strings. Don't store money as FLOAT.     │
│     Use DECIMAL for money. Use DATE/TIMESTAMP for dates.          │
│                                                                    │
│  7. ADD created_at AND updated_at TO EVERY TABLE                   │
│     You'll thank yourself during debugging                        │
│                                                                    │
│  8. THINK ABOUT NULL CAREFULLY                                     │
│     NULL means "unknown." Is that valid for this column?          │
│     Use NOT NULL by default, allow NULL only when needed.         │
│                                                                    │
│  9. PLAN FOR GROWTH                                                │
│     Will this table have 1K rows or 1B rows?                      │
│     That changes everything (partitioning, indexing strategy)     │
│                                                                    │
│  10. MODEL FOR YOUR QUERIES                                        │
│     Know the access patterns BEFORE designing the schema          │
│     (Especially critical for NoSQL and data warehouses)           │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Common Data Types — Quick Reference

| Use Case | ❌ Wrong Type | ✅ Right Type | Why |
|----------|-------------|--------------|-----|
| Money | FLOAT, DOUBLE | DECIMAL(10,2) | Float has precision errors: 0.1 + 0.2 ≠ 0.3 |
| Dates | VARCHAR | DATE, TIMESTAMP | Can't sort, compare, or calculate with strings |
| Email | VARCHAR(20) | VARCHAR(254) | RFC allows up to 254 chars |
| Phone | INT, BIGINT | VARCHAR(20) | Leading zeros, +country codes, dashes |
| Yes/No | VARCHAR("yes") | BOOLEAN | Type safety, smaller storage |
| UUID | VARCHAR(36) | UUID (native type) | Efficient storage, built-in validation |
| IP Address | VARCHAR | INET (PostgreSQL) | Built-in validation, range queries |
| Large text | VARCHAR(MAX) | TEXT | Semantic clarity |

---

## 🧠 Quick Recall — Chapter Summary

| Concept | One-Line Summary |
|---------|-----------------|
| Data Modeling | Designing the structure of your database (entities, relationships) |
| ER Diagram | Visual representation of entities, attributes, and relationships |
| Cardinality | 1:1, 1:M, M:N — how entities relate in quantity |
| Primary Key | Unique identifier for each row |
| Foreign Key | Links tables together, enforces referential integrity |
| 1NF | Atomic values, no repeating groups |
| 2NF | No partial dependencies (on part of composite PK) |
| 3NF | No transitive dependencies (non-key → non-key) |
| BCNF | Every determinant must be a superkey |
| Denormalization | Add redundancy for read performance |
| Star Schema | Fact table + dimension tables (data warehousing) |
| Snowflake Schema | Star with normalized dimensions |

---

## ❓ Self-Check Questions

1. What's the difference between Conceptual, Logical, and Physical data models?
2. Draw an ER diagram for a library system (books, members, borrowings).
3. What is a transitive dependency? Give an example.
4. Convert this table to 3NF: `Order(OrderID, CustomerName, CustomerCity, ProductName, Quantity)`
5. When would you choose denormalization over normalization?
6. What is the difference between a Star Schema and a Snowflake Schema?
7. Why should you use DECIMAL instead of FLOAT for money?
8. What is the "3NF mantra"?
9. Explain the difference between a surrogate key and a natural key.
10. Name 3 consequences of bad data modeling.

---

> **Next Chapter** → [1.5 — ACID vs BASE — Two Philosophies of Data](./05-ACID-vs-BASE.md)
