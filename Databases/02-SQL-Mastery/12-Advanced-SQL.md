# 2.12 — Advanced SQL — Pivoting, JSON, XML, Regex & Dynamic SQL 🔴

> **"Standard SQL handles 80% of your needs. This chapter covers the other 20% — the part that separates working developers from wizards."**
> These features turn SQL into a complete data transformation language.

---

## 🎯 What You'll Master

```
✅ PIVOT / UNPIVOT — rows to columns and back
✅ JSON functions — query, create, and manipulate JSON in SQL
✅ XML handling — parsing and generating XML
✅ Regular Expressions in SQL — pattern matching on steroids
✅ Dynamic SQL — building and executing queries at runtime
✅ LATERAL joins and CROSS APPLY / OUTER APPLY
✅ GROUPING SETS, ROLLUP, CUBE — multi-level aggregations
✅ Pattern matching with MATCH_RECOGNIZE (SQL:2016)
```

---

## 🔄 Part 1: PIVOT & UNPIVOT — Reshaping Data

### The Problem: Data in Rows, Need it in Columns

You have sales data stored **vertically** (one row per quarter):

| employee | quarter | amount |
|----------|---------|--------|
| Alice | Q1 | 50000 |
| Alice | Q2 | 60000 |
| Alice | Q3 | 55000 |
| Bob | Q1 | 45000 |
| Bob | Q2 | 70000 |
| Bob | Q3 | 65000 |

But your report needs it **horizontally**:

| employee | Q1 | Q2 | Q3 |
|----------|-------|-------|-------|
| Alice | 50000 | 60000 | 55000 |
| Bob | 45000 | 70000 | 65000 |

That transformation is called **PIVOT**.

---

### SQL Server — Native PIVOT

```sql
SELECT employee, [Q1], [Q2], [Q3], [Q4]
FROM (
    SELECT employee, quarter, amount
    FROM quarterly_sales
) AS source_data
PIVOT (
    SUM(amount)
    FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS pivoted;
```

### Oracle — Native PIVOT (11g+)

```sql
SELECT *
FROM quarterly_sales
PIVOT (
    SUM(amount)
    FOR quarter IN ('Q1' AS Q1, 'Q2' AS Q2, 'Q3' AS Q3, 'Q4' AS Q4)
);
```

### PostgreSQL — CASE Expression (Manual PIVOT)

PostgreSQL doesn't have native PIVOT, but CASE + aggregation does the same:

```sql
SELECT 
    employee,
    SUM(CASE WHEN quarter = 'Q1' THEN amount END) AS Q1,
    SUM(CASE WHEN quarter = 'Q2' THEN amount END) AS Q2,
    SUM(CASE WHEN quarter = 'Q3' THEN amount END) AS Q3,
    SUM(CASE WHEN quarter = 'Q4' THEN amount END) AS Q4
FROM quarterly_sales
GROUP BY employee;
```

### PostgreSQL — crosstab (tablefunc extension)

```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM crosstab(
    'SELECT employee, quarter, amount FROM quarterly_sales ORDER BY 1, 2',
    'SELECT DISTINCT quarter FROM quarterly_sales ORDER BY 1'
) AS ct(employee TEXT, "Q1" NUMERIC, "Q2" NUMERIC, "Q3" NUMERIC, "Q4" NUMERIC);
```

### MySQL — CASE Expression (Same as PostgreSQL)

```sql
SELECT 
    employee,
    SUM(IF(quarter = 'Q1', amount, 0)) AS Q1,
    SUM(IF(quarter = 'Q2', amount, 0)) AS Q2,
    SUM(IF(quarter = 'Q3', amount, 0)) AS Q3,
    SUM(IF(quarter = 'Q4', amount, 0)) AS Q4
FROM quarterly_sales
GROUP BY employee;
```

---

### UNPIVOT — Columns to Rows (The Reverse)

Starting from:

| employee | Q1 | Q2 | Q3 |
|----------|-------|-------|-------|
| Alice | 50000 | 60000 | 55000 |

Back to:

| employee | quarter | amount |
|----------|---------|--------|
| Alice | Q1 | 50000 |
| Alice | Q2 | 60000 |
| Alice | Q3 | 55000 |

```sql
-- SQL Server / Oracle
SELECT employee, quarter, amount
FROM pivoted_sales
UNPIVOT (
    amount FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS unpivoted;

-- PostgreSQL: LATERAL + VALUES
SELECT s.employee, v.quarter, v.amount
FROM pivoted_sales s
CROSS JOIN LATERAL (VALUES 
    ('Q1', s.q1), 
    ('Q2', s.q2), 
    ('Q3', s.q3), 
    ('Q4', s.q4)
) AS v(quarter, amount)
WHERE v.amount IS NOT NULL;

-- PostgreSQL: UNION ALL approach
SELECT employee, 'Q1' AS quarter, q1 AS amount FROM pivoted_sales
UNION ALL
SELECT employee, 'Q2', q2 FROM pivoted_sales
UNION ALL
SELECT employee, 'Q3', q3 FROM pivoted_sales
UNION ALL
SELECT employee, 'Q4', q4 FROM pivoted_sales;
```

---

## 📊 Part 2: Advanced Aggregations — GROUPING SETS, ROLLUP, CUBE

### GROUPING SETS — Multiple GROUP BYs in One Query

```sql
-- Without GROUPING SETS (3 separate queries):
SELECT department, NULL AS title, SUM(salary) FROM emp GROUP BY department
UNION ALL
SELECT NULL, title, SUM(salary) FROM emp GROUP BY title
UNION ALL
SELECT NULL, NULL, SUM(salary) FROM emp;

-- With GROUPING SETS (one pass!):
SELECT department, title, SUM(salary) AS total_salary
FROM employees
GROUP BY GROUPING SETS (
    (department),       -- Total per department
    (title),            -- Total per title
    ()                  -- Grand total
);
```

### ROLLUP — Hierarchical Subtotals

```sql
SELECT 
    region, country, city, 
    SUM(revenue) AS total_revenue
FROM sales
GROUP BY ROLLUP (region, country, city);
```

| region | country | city | total_revenue |
|--------|---------|------|---------------|
| APAC | India | Mumbai | 100000 |
| APAC | India | Delhi | 80000 |
| APAC | India | **NULL** | **180000** ← India subtotal |
| APAC | Japan | Tokyo | 200000 |
| APAC | Japan | **NULL** | **200000** ← Japan subtotal |
| APAC | **NULL** | **NULL** | **380000** ← APAC subtotal |
| **NULL** | **NULL** | **NULL** | **380000** ← Grand total |

```
ROLLUP produces hierarchy:
(region, country, city)  → Individual rows
(region, country)        → Country subtotals
(region)                 → Region subtotals
()                       → Grand total
```

### CUBE — Every Possible Combination

```sql
SELECT department, gender, SUM(salary)
FROM employees
GROUP BY CUBE (department, gender);
```

Produces:
```
(department, gender)  → Per dept + gender
(department)          → Per department
(gender)              → Per gender
()                    → Grand total
```

### GROUPING() — Identify Subtotal Rows

```sql
SELECT 
    CASE WHEN GROUPING(department) = 1 THEN 'ALL DEPTS' ELSE department END AS department,
    CASE WHEN GROUPING(title) = 1 THEN 'ALL TITLES' ELSE title END AS title,
    SUM(salary) AS total_salary,
    GROUPING(department) AS is_dept_total,
    GROUPING(title) AS is_title_total
FROM employees
GROUP BY ROLLUP (department, title);
```

> `GROUPING(col)` returns 1 when that column is a subtotal aggregation row (NULL due to ROLLUP/CUBE, not actual NULL data).

---

## 🔤 Part 3: JSON in SQL — The Modern Must-Know

### Why JSON in SQL?

Modern applications often need to:
- Store semi-structured data alongside relational data
- Return API-ready JSON directly from the database
- Query nested JSON documents without separate NoSQL database
- Handle flexible schemas for user preferences, metadata, etc.

---

### PostgreSQL — JSON Champion 🏆

PostgreSQL has the most powerful JSON support among SQL databases.

```sql
-- Two JSON types:
-- json:  Stores raw text, re-parsed on every access
-- jsonb: Binary format, indexed, faster queries ← USE THIS!

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    attributes JSONB  -- Flexible attributes
);

INSERT INTO products (name, attributes) VALUES
('Laptop', '{"brand": "Dell", "specs": {"ram": 16, "ssd": 512}, "tags": ["business", "portable"]}'),
('Phone',  '{"brand": "Apple", "specs": {"ram": 8, "storage": 256}, "tags": ["mobile", "premium"]}');
```

#### Querying JSON

```sql
-- Arrow operators: -> (returns JSON), ->> (returns TEXT)
SELECT 
    name,
    attributes->>'brand' AS brand,                    -- "Dell" (text)
    attributes->'specs'->>'ram' AS ram_gb,            -- "16" (text)
    (attributes->'specs'->>'ram')::INT AS ram_int,    -- 16 (integer)
    attributes->'tags'->>0 AS first_tag               -- "business"
FROM products;

-- #> / #>> for path-based access
SELECT attributes #>> '{specs, ram}' AS ram FROM products;  -- "16"

-- Filter by JSON value
SELECT * FROM products WHERE attributes->>'brand' = 'Dell';
SELECT * FROM products WHERE (attributes->'specs'->>'ram')::INT >= 16;

-- Check if key exists
SELECT * FROM products WHERE attributes ? 'brand';         -- Has key 'brand'?
SELECT * FROM products WHERE attributes ?| ARRAY['tags'];  -- Has ANY of these keys?
SELECT * FROM products WHERE attributes ?& ARRAY['brand', 'specs']; -- Has ALL keys?

-- Contains operator (@>)
SELECT * FROM products WHERE attributes @> '{"brand": "Dell"}';
SELECT * FROM products WHERE attributes->'tags' @> '"business"';

-- Array contains
SELECT * FROM products 
WHERE attributes->'tags' ? 'premium';  -- tags array contains "premium"
```

#### JSONB Indexing (GIN Index)

```sql
-- Index ALL keys and values (supports ?, @>, ?|, ?&)
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);

-- Index specific path
CREATE INDEX idx_products_brand ON products USING BTREE ((attributes->>'brand'));

-- Now these are fast:
SELECT * FROM products WHERE attributes @> '{"brand": "Dell"}';
SELECT * FROM products WHERE attributes->>'brand' = 'Dell';
```

#### JSON Construction

```sql
-- Build JSON objects
SELECT json_build_object(
    'id', id,
    'name', name,
    'department', department
) AS employee_json
FROM employees;

-- Aggregate rows into JSON array
SELECT json_agg(json_build_object('id', id, 'name', name))
FROM employees WHERE department = 'Sales';
-- Returns: [{"id": 4, "name": "Dave"}, {"id": 5, "name": "Eve"}]

-- Row to JSON
SELECT row_to_json(e) FROM employees e WHERE id = 1;

-- JSON aggregation with grouping
SELECT 
    department,
    json_agg(json_build_object('name', name, 'salary', salary)) AS members
FROM employees
GROUP BY department;
```

#### JSON Manipulation

```sql
-- Set/update a key
UPDATE products 
SET attributes = jsonb_set(attributes, '{specs, ram}', '32')
WHERE name = 'Laptop';

-- Add a new key
UPDATE products 
SET attributes = attributes || '{"color": "silver"}'
WHERE name = 'Laptop';

-- Remove a key
UPDATE products 
SET attributes = attributes - 'color'
WHERE name = 'Laptop';

-- Remove nested key
UPDATE products 
SET attributes = attributes #- '{specs, ssd}'
WHERE name = 'Laptop';

-- Expand JSON to rows
SELECT p.name, tag
FROM products p, jsonb_array_elements_text(p.attributes->'tags') AS tag;
-- Returns one row per tag!
```

---

### SQL Server — JSON Functions (2016+)

```sql
-- SQL Server stores JSON as NVARCHAR — no special type
DECLARE @json NVARCHAR(MAX) = N'{
    "name": "Alice",
    "age": 30,
    "skills": ["SQL", "Python", "Java"],
    "address": {"city": "Seattle", "state": "WA"}
}';

-- Query JSON
SELECT JSON_VALUE(@json, '$.name') AS name;          -- 'Alice' (scalar)
SELECT JSON_VALUE(@json, '$.address.city') AS city;   -- 'Seattle'
SELECT JSON_QUERY(@json, '$.skills') AS skills;       -- ["SQL","Python","Java"]
SELECT JSON_VALUE(@json, '$.skills[0]') AS first_skill; -- 'SQL'

-- Check if valid JSON
SELECT ISJSON(@json);  -- 1 = valid, 0 = invalid

-- Parse JSON array to rows
SELECT value AS skill
FROM OPENJSON(@json, '$.skills');

-- Parse JSON to table
SELECT *
FROM OPENJSON(@json)
WITH (
    name VARCHAR(50) '$.name',
    age INT '$.age',
    city VARCHAR(50) '$.address.city',
    skills NVARCHAR(MAX) '$.skills' AS JSON
);

-- Build JSON from query
SELECT id, name, department
FROM employees
FOR JSON PATH;
-- Returns: [{"id":1,"name":"Alice","department":"Engineering"}, ...]

-- Nested JSON output
SELECT 
    d.name AS department_name,
    (SELECT name, salary FROM employees e WHERE e.dept_id = d.id FOR JSON PATH) AS members
FROM departments d
FOR JSON PATH;
```

### MySQL — JSON Type (5.7+)

```sql
-- MySQL has a native JSON column type
CREATE TABLE events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    data JSON
);

INSERT INTO events (data) VALUES 
('{"type": "click", "page": "/home", "user": {"id": 1, "name": "Alice"}}');

-- Query JSON
SELECT 
    JSON_EXTRACT(data, '$.type') AS event_type,
    data->>'$.user.name' AS user_name,  -- ->> extracts and unquotes
    data->'$.user' AS user_object
FROM events;

-- Search in JSON
SELECT * FROM events 
WHERE JSON_EXTRACT(data, '$.type') = '"click"';
-- Or shorthand:
SELECT * FROM events WHERE data->>'$.type' = 'click';

-- JSON aggregation
SELECT JSON_ARRAYAGG(JSON_OBJECT('id', id, 'name', name))
FROM employees;

-- Modify JSON
UPDATE events SET data = JSON_SET(data, '$.processed', true) WHERE id = 1;
UPDATE events SET data = JSON_REMOVE(data, '$.user.name') WHERE id = 1;
UPDATE events SET data = JSON_INSERT(data, '$.source', 'web') WHERE id = 1;

-- Multi-valued index on JSON array (MySQL 8.0.17+)
CREATE TABLE orders (
    id INT PRIMARY KEY,
    tags JSON
);
CREATE INDEX idx_tags ON orders((CAST(tags AS UNSIGNED ARRAY)));
SELECT * FROM orders WHERE 42 MEMBER OF(tags);
```

### Oracle — JSON Support (12c+, enhanced in 21c)

```sql
-- Oracle 21c JSON type
CREATE TABLE events (
    id NUMBER GENERATED ALWAYS AS IDENTITY,
    data JSON
);

-- Dot notation (simple and clean)
SELECT e.data.type, e.data.user.name
FROM events e;

-- JSON_VALUE and JSON_QUERY
SELECT JSON_VALUE(data, '$.type') AS event_type
FROM events;

-- JSON_TABLE — most powerful: JSON → relational rows
SELECT jt.*
FROM events e,
JSON_TABLE(e.data, '$'
    COLUMNS (
        event_type VARCHAR2(50) PATH '$.type',
        page       VARCHAR2(100) PATH '$.page',
        user_id    NUMBER PATH '$.user.id',
        user_name  VARCHAR2(50) PATH '$.user.name'
    )
) jt;
```

---

## 📜 Part 4: XML Handling in SQL

### Why XML in SQL?

- Legacy enterprise systems still use XML extensively
- SOAP APIs return XML
- Configuration files, document storage
- Industry standards (HL7 healthcare, XBRL finance)

### SQL Server — Rich XML Support

```sql
-- XML data type
DECLARE @xml XML = N'
<employees>
    <employee id="1">
        <name>Alice</name>
        <department>Engineering</department>
        <salary>95000</salary>
    </employee>
    <employee id="2">
        <name>Bob</name>
        <department>Sales</department>
        <salary>85000</salary>
    </employee>
</employees>';

-- XQuery
SELECT 
    x.value('@id', 'INT') AS emp_id,
    x.value('(name)[1]', 'VARCHAR(50)') AS name,
    x.value('(salary)[1]', 'DECIMAL(10,2)') AS salary
FROM @xml.nodes('/employees/employee') AS t(x);

-- Generate XML from query
SELECT id, name, department, salary
FROM employees
FOR XML PATH('employee'), ROOT('employees');
```

### PostgreSQL — XML Functions

```sql
-- Generate XML
SELECT xmlelement(
    name employee,
    xmlattributes(id AS id),
    xmlelement(name name, name),
    xmlelement(name salary, salary)
)
FROM employees;

-- Parse XML with xpath
SELECT (xpath('//name/text()', xml_column))[1]::TEXT AS name
FROM xml_documents;

-- XMLTABLE — XML to rows (PostgreSQL 10+)
SELECT x.*
FROM xml_documents d,
XMLTABLE('/employees/employee' PASSING d.xml_data
    COLUMNS 
        emp_id INT PATH '@id',
        name TEXT PATH 'name',
        salary NUMERIC PATH 'salary'
) AS x;
```

### Oracle — XMLType

```sql
-- Oracle has XMLType with full XQuery support
SELECT 
    EXTRACTVALUE(xml_col, '/employee/name') AS name,
    EXTRACTVALUE(xml_col, '/employee/salary') AS salary
FROM employee_xml;

-- XMLTABLE (Oracle 10g+)
SELECT x.*
FROM employee_xml e,
XMLTABLE('/employees/employee' PASSING e.xml_data
    COLUMNS
        emp_id NUMBER PATH '@id',
        name VARCHAR2(50) PATH 'name',
        salary NUMBER PATH 'salary'
) x;
```

---

## 🔍 Part 5: Regular Expressions in SQL

### PostgreSQL — Full Regex Support

```sql
-- ~ operator (case-sensitive regex match)
SELECT * FROM customers WHERE email ~ '^[a-z]+@gmail\.com$';

-- ~* operator (case-insensitive)
SELECT * FROM customers WHERE name ~* 'mc[a-z]+';  -- McDonald, McDowell, etc.

-- !~ (NOT match)
SELECT * FROM customers WHERE phone !~ '^\+1';  -- Non-US phone numbers

-- regexp_matches — extract matching groups
SELECT (regexp_matches(email, '(.+)@(.+)\.(.+)'))[1] AS username,
       (regexp_matches(email, '(.+)@(.+)\.(.+)'))[2] AS domain
FROM customers;

-- regexp_replace — substitution
SELECT regexp_replace(phone, '[^0-9]', '', 'g') AS digits_only
FROM customers;
-- '+1 (555) 123-4567' → '15551234567'

-- regexp_split_to_table — split string into rows
SELECT regexp_split_to_table('apple,banana,,cherry', ',') AS fruit;
-- Returns: apple, banana, (empty), cherry

-- regexp_split_to_array — split into array
SELECT regexp_split_to_array('apple,banana,cherry', ',') AS fruits;
-- Returns: {apple,banana,cherry}
```

### MySQL — REGEXP / RLIKE

```sql
-- Basic regex match
SELECT * FROM customers WHERE email REGEXP '^[a-z]+@gmail\\.com$';
SELECT * FROM customers WHERE name RLIKE '^mc';  -- RLIKE = synonym

-- REGEXP_LIKE (MySQL 8.0+)
SELECT * FROM customers WHERE REGEXP_LIKE(name, '^mc', 'i');  -- 'i' = case-insensitive

-- REGEXP_REPLACE
SELECT REGEXP_REPLACE(phone, '[^0-9]', '') AS digits_only FROM customers;

-- REGEXP_SUBSTR — extract first match
SELECT REGEXP_SUBSTR(email, '@[a-z]+') AS domain FROM customers;

-- REGEXP_INSTR — position of match
SELECT REGEXP_INSTR(email, '@') AS at_position FROM customers;
```

### SQL Server — Limited Native Regex (Use LIKE or CLR)

```sql
-- SQL Server doesn't have native regex, but LIKE covers basics:
SELECT * FROM customers WHERE email LIKE '%@gmail.com';
SELECT * FROM customers WHERE phone LIKE '+1%';

-- Character ranges with LIKE
SELECT * FROM products WHERE sku LIKE '[A-Z][A-Z]-[0-9][0-9][0-9]';
-- Matches: AB-123, XY-999

-- For full regex: Use PATINDEX
SELECT * FROM customers WHERE PATINDEX('%[0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]%', phone) > 0;

-- For true regex in SQL Server: Use CLR integration or STRING_SPLIT + creative LIKE
```

### Oracle — REGEXP Functions

```sql
-- REGEXP_LIKE
SELECT * FROM customers WHERE REGEXP_LIKE(email, '^[a-z]+@gmail\.com$', 'i');

-- REGEXP_SUBSTR
SELECT REGEXP_SUBSTR(email, '@(\w+)', 1, 1, NULL, 1) AS domain FROM customers;

-- REGEXP_REPLACE
SELECT REGEXP_REPLACE(phone, '[^0-9]', '') AS digits_only FROM customers;

-- REGEXP_INSTR
SELECT REGEXP_INSTR(email, '@') AS at_pos FROM customers;

-- REGEXP_COUNT
SELECT REGEXP_COUNT('hello world hello', 'hello') AS match_count FROM dual;
-- Returns: 2
```

### Common Regex Patterns for SQL

```
Email:          ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$
Phone (US):     ^\+?1?[-.\s]?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}$
IP Address:     ^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$
UUID:           ^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$
Date (YYYY-MM-DD): ^\d{4}-\d{2}-\d{2}$
Alphanumeric:   ^[a-zA-Z0-9]+$
```

---

## 💻 Part 6: Dynamic SQL — Building Queries at Runtime

### When You Need Dynamic SQL

```
✅ USE DYNAMIC SQL FOR:
   • Search queries with optional filters
   • Dynamic column names or table names
   • PIVOT with variable number of columns
   • Data-driven WHERE clauses
   • Admin scripts that operate on variable objects

❌ AVOID WHEN:
   • Static SQL can do the job
   • User input is directly concatenated (SQL injection!)
```

### ⚠️ SQL INJECTION WARNING

```sql
-- ❌ NEVER concatenate user input directly!
SET @sql = 'SELECT * FROM users WHERE name = ''' + @userInput + '''';
-- If @userInput = "'; DROP TABLE users; --"
-- Result: SELECT * FROM users WHERE name = ''; DROP TABLE users; --'
-- YOUR TABLE IS GONE! 💀

-- ✅ ALWAYS use parameterized queries
```

---

### PostgreSQL Dynamic SQL (EXECUTE)

```sql
CREATE OR REPLACE FUNCTION search_employees(
    p_department TEXT DEFAULT NULL,
    p_min_salary NUMERIC DEFAULT NULL,
    p_name_pattern TEXT DEFAULT NULL
)
RETURNS SETOF employees
LANGUAGE plpgsql
AS $$
DECLARE
    v_sql TEXT := 'SELECT * FROM employees WHERE 1=1';
BEGIN
    -- Build query dynamically based on provided parameters
    IF p_department IS NOT NULL THEN
        v_sql := v_sql || ' AND department = $1';
    END IF;
    
    IF p_min_salary IS NOT NULL THEN
        v_sql := v_sql || ' AND salary >= $2';
    END IF;
    
    IF p_name_pattern IS NOT NULL THEN
        v_sql := v_sql || ' AND name ILIKE $3';
    END IF;
    
    -- Execute with parameters (SAFE from injection!)
    RETURN QUERY EXECUTE v_sql 
    USING p_department, p_min_salary, '%' || p_name_pattern || '%';
END;
$$;

-- Use it
SELECT * FROM search_employees(p_department := 'Sales', p_min_salary := 50000);
```

### SQL Server Dynamic SQL (sp_executesql)

```sql
CREATE PROCEDURE sp_search_employees
    @department VARCHAR(50) = NULL,
    @min_salary DECIMAL(10,2) = NULL,
    @name_pattern VARCHAR(100) = NULL
AS
BEGIN
    DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM employees WHERE 1=1';
    DECLARE @params NVARCHAR(500) = N'@p_dept VARCHAR(50), @p_salary DECIMAL(10,2), @p_name VARCHAR(100)';
    
    IF @department IS NOT NULL
        SET @sql += N' AND department = @p_dept';
    
    IF @min_salary IS NOT NULL
        SET @sql += N' AND salary >= @p_salary';
    
    IF @name_pattern IS NOT NULL
        SET @sql += N' AND name LIKE @p_name';
    
    -- SAFE: Parameterized execution
    EXEC sp_executesql @sql, @params,
        @p_dept = @department,
        @p_salary = @min_salary,
        @p_name = @name_pattern;
END;
```

### MySQL Dynamic SQL (PREPARE / EXECUTE)

```sql
DELIMITER //
CREATE PROCEDURE sp_search(IN p_department VARCHAR(50), IN p_min_salary DECIMAL(10,2))
BEGIN
    SET @sql = 'SELECT * FROM employees WHERE 1=1';
    
    IF p_department IS NOT NULL THEN
        SET @sql = CONCAT(@sql, ' AND department = ?');
    END IF;
    
    IF p_min_salary IS NOT NULL THEN
        SET @sql = CONCAT(@sql, ' AND salary >= ?');
    END IF;
    
    PREPARE stmt FROM @sql;
    
    IF p_department IS NOT NULL AND p_min_salary IS NOT NULL THEN
        SET @p1 = p_department;
        SET @p2 = p_min_salary;
        EXECUTE stmt USING @p1, @p2;
    ELSEIF p_department IS NOT NULL THEN
        SET @p1 = p_department;
        EXECUTE stmt USING @p1;
    ELSEIF p_min_salary IS NOT NULL THEN
        SET @p1 = p_min_salary;
        EXECUTE stmt USING @p1;
    ELSE
        EXECUTE stmt;
    END IF;
    
    DEALLOCATE PREPARE stmt;
END //
DELIMITER ;
```

### Dynamic PIVOT (Unknown Number of Columns)

```sql
-- SQL Server: Dynamic PIVOT
DECLARE @columns NVARCHAR(MAX), @sql NVARCHAR(MAX);

-- Get distinct quarter values dynamically
SELECT @columns = STRING_AGG(QUOTENAME(quarter), ', ')
FROM (SELECT DISTINCT quarter FROM quarterly_sales) AS q;

SET @sql = N'
SELECT employee, ' + @columns + N'
FROM quarterly_sales
PIVOT (SUM(amount) FOR quarter IN (' + @columns + N')) AS p';

EXEC sp_executesql @sql;
```

---

## 🔗 Part 7: LATERAL Joins & CROSS/OUTER APPLY

### What is LATERAL / APPLY?

A LATERAL join lets the subquery reference columns from the preceding table — like a correlated subquery, but returns multiple rows/columns.

```sql
-- PostgreSQL: LATERAL
-- "For each department, get the top 3 earners"
SELECT d.department_name, top3.name, top3.salary
FROM departments d
CROSS JOIN LATERAL (
    SELECT name, salary
    FROM employees e
    WHERE e.department_id = d.id  -- References d!
    ORDER BY salary DESC
    LIMIT 3
) AS top3;

-- SQL Server: CROSS APPLY (same concept)
SELECT d.department_name, top3.name, top3.salary
FROM departments d
CROSS APPLY (
    SELECT TOP 3 name, salary
    FROM employees e
    WHERE e.department_id = d.id
    ORDER BY salary DESC
) AS top3;

-- OUTER APPLY / LEFT JOIN LATERAL
-- Same but keeps departments with no employees (NULLs)
SELECT d.department_name, top3.name, top3.salary
FROM departments d
OUTER APPLY (
    SELECT TOP 3 name, salary
    FROM employees e
    WHERE e.department_id = d.id
    ORDER BY salary DESC
) AS top3;
```

> 💡 `CROSS APPLY` = `INNER JOIN LATERAL` = skip if subquery returns empty  
> `OUTER APPLY` = `LEFT JOIN LATERAL` = keep row with NULLs if subquery returns empty

### When LATERAL/APPLY Shines

```sql
-- 1. Top-N per group (cleaner than window functions)
-- 2. Calling table-valued functions per row
-- 3. Unnesting arrays/JSON per row
-- 4. Complex calculations that reference the outer row

-- Example: Split comma-separated tags per product
SELECT p.name, t.tag
FROM products p
CROSS JOIN LATERAL unnest(string_to_array(p.tags, ',')) AS t(tag);
```

---

## 🎲 Part 8: Other Advanced SQL Features

### STRING_AGG / GROUP_CONCAT — Aggregate into String

```sql
-- PostgreSQL (9.0+) / SQL Server (2017+)
SELECT department, STRING_AGG(name, ', ' ORDER BY name) AS members
FROM employees
GROUP BY department;
-- Engineering: Alice, Bob, Carol

-- MySQL
SELECT department, GROUP_CONCAT(name ORDER BY name SEPARATOR ', ') AS members
FROM employees
GROUP BY department;

-- Oracle (11g+ using LISTAGG)
SELECT department, LISTAGG(name, ', ') WITHIN GROUP (ORDER BY name) AS members
FROM employees
GROUP BY department;
```

### FILTER Clause (PostgreSQL — Conditional Aggregation)

```sql
-- PostgreSQL: Elegant conditional aggregation
SELECT 
    department,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE salary > 100000) AS high_earners,
    AVG(salary) FILTER (WHERE hire_date > '2023-01-01') AS recent_hire_avg
FROM employees
GROUP BY department;

-- Other databases: Use CASE
SELECT 
    department,
    COUNT(*) AS total,
    COUNT(CASE WHEN salary > 100000 THEN 1 END) AS high_earners,
    AVG(CASE WHEN hire_date > '2023-01-01' THEN salary END) AS recent_hire_avg
FROM employees
GROUP BY department;
```

### GENERATE_SERIES — Create Sequences

```sql
-- PostgreSQL: Generate a series of numbers
SELECT generate_series(1, 10) AS n;

-- Generate dates for a calendar
SELECT generate_series('2024-01-01'::DATE, '2024-12-31'::DATE, '1 day'::INTERVAL) AS cal_date;

-- Fill gaps in time-series data
SELECT 
    dates.dt,
    COALESCE(o.revenue, 0) AS revenue
FROM generate_series('2024-01-01'::DATE, '2024-01-31'::DATE, '1 day') AS dates(dt)
LEFT JOIN daily_orders o ON o.order_date = dates.dt;

-- SQL Server: Recursive CTE (no generate_series)
-- MySQL 8.0.33+: Has generate_series as a table function (limited)
```

### VALUES Lists — Inline Data

```sql
-- Create inline data without a table
SELECT * FROM (VALUES 
    (1, 'Active'),
    (2, 'Inactive'),
    (3, 'Pending')
) AS statuses(id, name);

-- Useful for mappings
SELECT e.name, s.label
FROM employees e
JOIN (VALUES 
    ('Engineering', 'ENG'),
    ('Sales', 'SLS'),
    ('Marketing', 'MKT')
) AS s(dept, label) ON e.department = s.dept;
```

---

## 🧪 Practice Challenges

### Challenge 1: Dynamic Pivot
> Given a table of students and their subject scores, pivot to show each subject as a column — but the subjects should be determined dynamically.

<details>
<summary>💡 Solution (PostgreSQL)</summary>

```sql
-- Since PostgreSQL has no native PIVOT, use CASE + crosstab
SELECT 
    student_name,
    MAX(CASE WHEN subject = 'Math' THEN score END) AS Math,
    MAX(CASE WHEN subject = 'Science' THEN score END) AS Science,
    MAX(CASE WHEN subject = 'English' THEN score END) AS English
FROM student_scores
GROUP BY student_name;

-- For truly dynamic: use a function that builds SQL
CREATE OR REPLACE FUNCTION dynamic_pivot()
RETURNS TEXT
LANGUAGE plpgsql AS $$
DECLARE
    v_columns TEXT;
    v_sql TEXT;
BEGIN
    SELECT string_agg(
        format('MAX(CASE WHEN subject = %L THEN score END) AS %I', subject, subject),
        ', '
    ) INTO v_columns
    FROM (SELECT DISTINCT subject FROM student_scores ORDER BY subject) s;
    
    v_sql := format('SELECT student_name, %s FROM student_scores GROUP BY student_name', v_columns);
    RETURN v_sql;
END;
$$;
```
</details>

### Challenge 2: JSON Report
> Generate a JSON report from the employees table that groups employees by department with nested arrays.

<details>
<summary>💡 Solution (PostgreSQL)</summary>

```sql
SELECT json_build_object(
    'report_date', CURRENT_DATE,
    'departments', (
        SELECT json_agg(
            json_build_object(
                'name', department,
                'employee_count', count,
                'avg_salary', avg_salary,
                'members', members
            )
        )
        FROM (
            SELECT 
                department,
                COUNT(*) AS count,
                ROUND(AVG(salary), 2) AS avg_salary,
                json_agg(json_build_object('name', name, 'salary', salary) ORDER BY salary DESC) AS members
            FROM employees
            GROUP BY department
        ) dept_data
    )
) AS report;
```
</details>

### Challenge 3: Extract Emails with Regex
> Find all employees whose email domain is not from a list of approved domains (gmail.com, company.com, outlook.com).

<details>
<summary>💡 Solution (PostgreSQL)</summary>

```sql
SELECT name, email,
    (regexp_matches(email, '@(.+)$'))[1] AS domain
FROM employees
WHERE email !~ '@(gmail\.com|company\.com|outlook\.com)$';
```
</details>

---

## 🗄️ Feature Support Matrix

| Feature | PostgreSQL | SQL Server | MySQL | Oracle | SQLite |
|---------|-----------|------------|-------|--------|--------|
| PIVOT/UNPIVOT | ❌ (use CASE/crosstab) | ✅ Native | ❌ (use CASE) | ✅ Native | ❌ |
| ROLLUP | ✅ | ✅ | ✅ | ✅ | ❌ |
| CUBE | ✅ | ✅ | ✅ | ✅ | ❌ |
| GROUPING SETS | ✅ | ✅ | ✅ (8.0) | ✅ | ❌ |
| JSON type | ✅ (jsonb) | ✅ (NVARCHAR) | ✅ (JSON) | ✅ (21c) | ✅ (JSON1) |
| JSON indexing | ✅ GIN | ✅ Computed | ✅ Multi-valued | ✅ | ❌ |
| XML type | ✅ | ✅ (rich) | ❌ (limited) | ✅ (XMLType) | ❌ |
| REGEX | ✅ ~ operator | ❌ (PATINDEX only) | ✅ REGEXP | ✅ REGEXP_* | ❌ |
| LATERAL | ✅ | ✅ (CROSS/OUTER APPLY) | ✅ (8.0.14+) | ✅ (12c) | ❌ |
| Dynamic SQL | ✅ EXECUTE | ✅ sp_executesql | ✅ PREPARE | ✅ EXECUTE IMMEDIATE | ❌ |
| STRING_AGG | ✅ | ✅ (2017+) | ✅ GROUP_CONCAT | ✅ LISTAGG | ✅ GROUP_CONCAT |
| generate_series | ✅ | ❌ (use CTE) | ❌ | ✅ (CONNECT BY) | ❌ |
| FILTER clause | ✅ | ❌ | ❌ | ❌ | ✅ |

---

## 🎯 Key Takeaways

```
┌──────────────────────────────────────────────────────────────────┐
│  ADVANCED SQL CHEAT SHEET                                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PIVOT:     Rows → Columns (CASE + GROUP BY is universal)        │
│  UNPIVOT:   Columns → Rows (UNION ALL or LATERAL + VALUES)       │
│                                                                  │
│  ROLLUP:    Hierarchical subtotals (region → country → city)     │
│  CUBE:      All combinations of subtotals                        │
│  GROUPING SETS: Custom combination of aggregation levels         │
│                                                                  │
│  JSON:      PostgreSQL JSONB is king. All modern DBs support it. │
│             Use GIN index for JSONB. Use -> for JSON, ->> for    │
│             text. jsonb_set to update, @> to search.             │
│                                                                  │
│  REGEX:     PostgreSQL ~ operator, MySQL REGEXP, Oracle REGEXP_* │
│             SQL Server: limited (use PATINDEX or CLR)            │
│                                                                  │
│  DYNAMIC SQL:                                                    │
│             ⚠️ ALWAYS parameterize! Never concatenate user input. │
│             PostgreSQL: EXECUTE ... USING                        │
│             SQL Server: sp_executesql with @params               │
│             MySQL: PREPARE / EXECUTE USING                       │
│                                                                  │
│  LATERAL / APPLY:                                                │
│             Correlated subquery that returns multiple rows        │
│             Perfect for "top-N per group" and function calls     │
│             CROSS APPLY = INNER, OUTER APPLY = LEFT              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

> 🏁 **Congratulations!** You've completed the **SQL Language Mastery** section (2.1 → 2.12). You now have the SQL knowledge to handle anything — from simple CRUDs to complex analytical queries, from JSON APIs to dynamic reporting systems.
> 
> **Next:** Dive into database-specific chapters (Oracle 2B, SQL Server 2C, MySQL 2D, PostgreSQL 2E) to master your chosen platform.

---

*SQL is one of the oldest programming languages still in active use (since 1974). It has survived because it keeps evolving — JSON, regex, window functions, recursive queries. The developers who master "advanced SQL" don't just query data; they transform, reshape, and extract insights that others can't.* 🧙‍♂️
