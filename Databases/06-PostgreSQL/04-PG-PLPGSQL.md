# 2E.4 — PL/pgSQL & Extensions 🟡

> **"SQL tells the database WHAT to do. PL/pgSQL tells it HOW to do it — with loops, conditions, and error handling."**
> This is where PostgreSQL transforms from a data store into a full application platform.

> **Level:** 🟡 Intermediate
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 2E.1-2E.3 (Architecture, Installation, Advanced Features)

---

## 🎯 What You'll Master

```
✅ PL/pgSQL syntax — variables, control flow, loops
✅ Functions — RETURNS scalar, row, table, SETOF
✅ Stored Procedures (PostgreSQL 11+) — with transaction control
✅ Triggers — BEFORE/AFTER/INSTEAD OF, row/statement level
✅ Error handling — EXCEPTION blocks, RAISE, custom error codes
✅ Cursors — processing large result sets row by row
✅ DO blocks — anonymous code blocks (ad-hoc scripts)
✅ Custom Types, Operators, and Aggregates
✅ Must-know extensions deep dive — pg_cron, pgvector, TimescaleDB, Citus
✅ Writing your own extension — the power move
```

---

## 🏗️ 1. PL/pgSQL Fundamentals — The Language

**PL/pgSQL** = **P**rocedural **L**anguage / **P**ost**g**re**SQL**

It extends SQL with:
- Variables and constants
- IF/ELSE, CASE, LOOP, WHILE, FOR
- Exception handling (TRY/CATCH equivalent)
- Cursors
- Dynamic SQL

```
┌───────────────────────────────────────────────────────────────────┐
│              PL/pgSQL vs Other Procedural Languages               │
├──────────────┬──────────────┬──────────────┬─────────────────────┤
│  PL/pgSQL    │  PL/Python   │  PL/Perl     │  PL/V8 (JavaScript)│
│  (Built-in)  │  (Untrusted) │  (Trusted)   │  (Extension)       │
│              │              │              │                     │
│  Best for:   │  Best for:   │  Best for:   │  Best for:          │
│  DB logic,   │  Data science│  Text        │  JSON manipulation  │
│  triggers,   │  ML, complex │  processing, │  Web devs familiar  │
│  validation  │  algorithms  │  regex       │  with JavaScript    │
│              │              │              │                     │
│  ⭐ Default  │  Needs setup │  Needs setup │  Needs extension    │
│  choice      │              │              │                     │
└──────────────┴──────────────┴──────────────┴─────────────────────┘
```

### The Basic Block Structure

```sql
-- Every PL/pgSQL block follows this structure:
DO $$
DECLARE
    -- Variable declarations (optional)
    my_name TEXT := 'Ritesh';
    my_count INT;
BEGIN
    -- Executable statements
    SELECT count(*) INTO my_count FROM users;
    RAISE NOTICE 'Hello %, there are % users', my_name, my_count;
    
EXCEPTION
    -- Error handling (optional)
    WHEN OTHERS THEN
        RAISE NOTICE 'Something went wrong: %', SQLERRM;
END;
$$;
```

> 💡 **The `$$` is a "dollar quoting" delimiter.** It avoids escaping single quotes inside the block. You can also use `$function$`, `$body$`, etc.

### Variables & Data Types

```sql
DO $$
DECLARE
    -- Basic types
    user_name     TEXT := 'Ritesh';
    user_age      INT := 30;
    salary        NUMERIC(10,2) := 75000.50;
    is_active     BOOLEAN := true;
    created_at    TIMESTAMPTZ := NOW();
    
    -- Constants
    TAX_RATE      CONSTANT NUMERIC := 0.18;
    
    -- Type from table column (auto-matches!)
    v_email       users.email%TYPE;         -- Same type as users.email column
    
    -- Type from entire row
    v_user        users%ROWTYPE;            -- Entire row of users table
    
    -- Record (flexible row, determined at runtime)
    v_record      RECORD;
    
    -- Array
    v_tags        TEXT[] := ARRAY['sql', 'postgresql'];
    
    -- JSONB
    v_config      JSONB := '{"debug": true}';
    
BEGIN
    -- Assign using SELECT INTO
    SELECT email INTO v_email FROM users WHERE id = 1;
    
    -- Assign entire row
    SELECT * INTO v_user FROM users WHERE id = 1;
    RAISE NOTICE 'User: %, Email: %', v_user.name, v_user.email;
    
    -- Assign using :=
    user_age := user_age + 1;
    
    -- NULL check
    IF v_email IS NULL THEN
        RAISE NOTICE 'User not found!';
    END IF;
END;
$$;
```

### Control Flow — IF, CASE, Loops

```sql
DO $$
DECLARE
    v_score INT := 85;
    v_grade TEXT;
    v_counter INT;
    rec RECORD;
BEGIN
    -- ═══════════════════════════════════════
    --  IF / ELSIF / ELSE
    -- ═══════════════════════════════════════
    IF v_score >= 90 THEN
        v_grade := 'A';
    ELSIF v_score >= 80 THEN
        v_grade := 'B';
    ELSIF v_score >= 70 THEN
        v_grade := 'C';
    ELSE
        v_grade := 'F';
    END IF;
    RAISE NOTICE 'Grade: %', v_grade;

    -- ═══════════════════════════════════════
    --  CASE (Simple)
    -- ═══════════════════════════════════════
    v_grade := CASE v_score / 10
        WHEN 10, 9 THEN 'A'
        WHEN 8      THEN 'B'
        WHEN 7      THEN 'C'
        ELSE              'F'
    END;

    -- ═══════════════════════════════════════
    --  CASE (Searched)
    -- ═══════════════════════════════════════
    v_grade := CASE
        WHEN v_score >= 90 THEN 'A'
        WHEN v_score >= 80 THEN 'B'
        WHEN v_score >= 70 THEN 'C'
        ELSE 'F'
    END;

    -- ═══════════════════════════════════════
    --  LOOP (basic — infinite until EXIT)
    -- ═══════════════════════════════════════
    v_counter := 0;
    LOOP
        v_counter := v_counter + 1;
        EXIT WHEN v_counter >= 5;     -- Break out
        CONTINUE WHEN v_counter = 3;  -- Skip iteration
        RAISE NOTICE 'Counter: %', v_counter;
    END LOOP;

    -- ═══════════════════════════════════════
    --  WHILE LOOP
    -- ═══════════════════════════════════════
    v_counter := 1;
    WHILE v_counter <= 5 LOOP
        RAISE NOTICE 'While: %', v_counter;
        v_counter := v_counter + 1;
    END LOOP;

    -- ═══════════════════════════════════════
    --  FOR LOOP (integer range)
    -- ═══════════════════════════════════════
    FOR i IN 1..5 LOOP
        RAISE NOTICE 'For: %', i;
    END LOOP;

    -- Reverse
    FOR i IN REVERSE 5..1 LOOP
        RAISE NOTICE 'Reverse: %', i;
    END LOOP;

    -- Step by 2
    FOR i IN 1..10 BY 2 LOOP
        RAISE NOTICE 'Step: %', i;   -- 1, 3, 5, 7, 9
    END LOOP;

    -- ═══════════════════════════════════════
    --  FOR LOOP (query result)  ← MOST USEFUL!
    -- ═══════════════════════════════════════
    FOR rec IN SELECT id, name, email FROM users LIMIT 5 LOOP
        RAISE NOTICE 'User #%: % (%)', rec.id, rec.name, rec.email;
    END LOOP;

    -- ═══════════════════════════════════════
    --  FOREACH (iterate over array)
    -- ═══════════════════════════════════════
    DECLARE
        v_tags TEXT[] := ARRAY['sql', 'postgresql', 'database'];
        v_tag TEXT;
    BEGIN
        FOREACH v_tag IN ARRAY v_tags LOOP
            RAISE NOTICE 'Tag: %', v_tag;
        END LOOP;
    END;
    
END;
$$;
```

---

## ⚡ 2. Functions — The Workhorses

### Function Basics

```sql
-- ═══════════════════════════════════════════════════
--  RETURNING A SCALAR VALUE
-- ═══════════════════════════════════════════════════

CREATE OR REPLACE FUNCTION calculate_tax(
    p_amount NUMERIC, 
    p_rate NUMERIC DEFAULT 0.18
)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN ROUND(p_amount * p_rate, 2);
END;
$$;

-- Usage:
SELECT calculate_tax(1000);           -- 180.00
SELECT calculate_tax(1000, 0.28);     -- 280.00
SELECT name, price, calculate_tax(price) AS tax FROM products;
```

```sql
-- ═══════════════════════════════════════════════════
--  RETURNING A TABLE ROW (single row)
-- ═══════════════════════════════════════════════════

CREATE OR REPLACE FUNCTION get_user(p_id INT)
RETURNS users              -- Returns a row matching users table structure
LANGUAGE plpgsql
AS $$
DECLARE
    v_user users%ROWTYPE;
BEGIN
    SELECT * INTO v_user FROM users WHERE id = p_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'User % not found', p_id
            USING ERRCODE = 'P0002';     -- Custom error code
    END IF;
    
    RETURN v_user;
END;
$$;

-- Usage:
SELECT * FROM get_user(1);
SELECT (get_user(1)).name;    -- Access specific column
```

```sql
-- ═══════════════════════════════════════════════════
--  RETURNING MULTIPLE ROWS (SETOF / TABLE)
-- ═══════════════════════════════════════════════════

-- Method 1: RETURNS SETOF
CREATE OR REPLACE FUNCTION get_users_by_country(p_country TEXT)
RETURNS SETOF users
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY 
        SELECT * FROM users WHERE country = p_country;
    
    IF NOT FOUND THEN
        RAISE NOTICE 'No users found for country: %', p_country;
    END IF;
END;
$$;

-- Usage:
SELECT * FROM get_users_by_country('India');

-- Method 2: RETURNS TABLE (define custom output structure)
CREATE OR REPLACE FUNCTION search_products(
    p_search TEXT,
    p_min_price NUMERIC DEFAULT 0,
    p_max_price NUMERIC DEFAULT 999999
)
RETURNS TABLE (
    product_id   INT,
    product_name TEXT,
    product_price NUMERIC,
    match_rank   REAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
        SELECT 
            p.id,
            p.name,
            p.price,
            ts_rank(
                to_tsvector('english', p.name), 
                plainto_tsquery('english', p_search)
            )
        FROM products p
        WHERE to_tsvector('english', p.name) @@ plainto_tsquery('english', p_search)
          AND p.price BETWEEN p_min_price AND p_max_price
        ORDER BY 4 DESC;
END;
$$;

-- Usage:
SELECT * FROM search_products('laptop', 500, 2000);
```

```sql
-- ═══════════════════════════════════════════════════
--  RETURNING VOID (side-effect only functions)
-- ═══════════════════════════════════════════════════

CREATE OR REPLACE FUNCTION log_event(
    p_event_type TEXT,
    p_payload JSONB DEFAULT '{}'
)
RETURNS VOID
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO events (event_type, payload)
    VALUES (p_event_type, p_payload);
END;
$$;

-- Usage:
SELECT log_event('user_login', '{"user_id": 42, "ip": "192.168.1.1"}');
```

### Function Volatility — Critical for Performance

```sql
-- ═══════════════════════════════════════════════════
--  VOLATILITY CATEGORIES (tell the planner what to expect)
-- ═══════════════════════════════════════════════════

-- VOLATILE (default) — can return different results each call
-- → Called every time, cannot be optimized
CREATE FUNCTION get_random() RETURNS INT 
LANGUAGE sql VOLATILE AS $$ SELECT (random() * 100)::int; $$;

-- STABLE — same result within a single query/transaction
-- → Safe to call multiple times in one query, planner can optimize
CREATE FUNCTION get_setting(key TEXT) RETURNS TEXT 
LANGUAGE sql STABLE AS $$ SELECT current_setting(key); $$;

-- IMMUTABLE — ALWAYS same result for same inputs (pure function)
-- → Can be used in indexes! Maximum optimization.
CREATE FUNCTION add_tax(amount NUMERIC) RETURNS NUMERIC 
LANGUAGE sql IMMUTABLE AS $$ SELECT amount * 1.18; $$;

-- ⭐ IMMUTABLE functions can be used in expression indexes!
CREATE INDEX idx_products_with_tax ON products (add_tax(price));
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                VOLATILITY QUICK REFERENCE                            │
├──────────────┬──────────────────┬────────────────────────────────────┤
│  Category    │  Can optimize?   │  Example                          │
├──────────────┼──────────────────┼────────────────────────────────────┤
│  IMMUTABLE   │  Maximum ✅      │  Math functions, string concat    │
│  STABLE      │  Within query ✅ │  current_timestamp, settings      │
│  VOLATILE    │  None ❌         │  random(), nextval(), INSERT      │
├──────────────┼──────────────────┼────────────────────────────────────┤
│  ⚠️ Marking a VOLATILE function as IMMUTABLE = WRONG RESULTS!     │
│  The planner trusts your declaration. Don't lie to it.              │
└──────────────┴──────────────────┴────────────────────────────────────┘
```

### SQL Language Functions (Simpler Alternative)

```sql
-- For simple functions, you can use pure SQL instead of PL/pgSQL
-- → Faster (inlineable), simpler, preferred when possible

CREATE OR REPLACE FUNCTION full_name(first TEXT, last TEXT)
RETURNS TEXT
LANGUAGE sql                    -- ← SQL, not plpgsql!
IMMUTABLE
AS $$
    SELECT first || ' ' || last;
$$;

-- Equivalent but written as SQL function — can be INLINED into queries
-- (The planner substitutes the function body directly into the query)
SELECT full_name('Ritesh', 'Singh');   -- 'Ritesh Singh'
```

---

## 🔧 3. Stored Procedures (PostgreSQL 11+) — With Transaction Control

**Functions** cannot control transactions (no COMMIT/ROLLBACK inside them).
**Procedures** can! This is the big difference.

```sql
-- ═══════════════════════════════════════════════════
--  PROCEDURE — with transaction control
-- ═══════════════════════════════════════════════════

CREATE OR REPLACE PROCEDURE transfer_funds(
    p_from_account INT,
    p_to_account INT,
    p_amount NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_from_balance NUMERIC;
BEGIN
    -- Check balance
    SELECT balance INTO v_from_balance 
    FROM accounts WHERE id = p_from_account FOR UPDATE;
    
    IF v_from_balance IS NULL THEN
        RAISE EXCEPTION 'Source account % not found', p_from_account;
    END IF;
    
    IF v_from_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds. Balance: %, Required: %', 
            v_from_balance, p_amount;
    END IF;
    
    -- Debit
    UPDATE accounts SET balance = balance - p_amount 
    WHERE id = p_from_account;
    
    -- Credit
    UPDATE accounts SET balance = balance + p_amount 
    WHERE id = p_to_account;
    
    -- Log the transfer
    INSERT INTO transactions (from_acc, to_acc, amount, txn_type)
    VALUES (p_from_account, p_to_account, p_amount, 'transfer');
    
    -- COMMIT inside procedure! (functions can't do this)
    COMMIT;
    
    RAISE NOTICE 'Transfer of % from account % to % completed', 
        p_amount, p_from_account, p_to_account;
    
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
$$;

-- Usage: (CALL, not SELECT!)
CALL transfer_funds(1001, 1002, 500.00);
```

### Functions vs Procedures — The Comparison

```
┌────────────────────────┬───────────────────────┬───────────────────────┐
│  Feature               │  FUNCTION             │  PROCEDURE            │
├────────────────────────┼───────────────────────┼───────────────────────┤
│  Called with            │  SELECT func()        │  CALL proc()          │
│  Returns value          │  ✅ Yes               │  ❌ No (OUT params)   │
│  Used in queries        │  ✅ Yes (in SELECT)   │  ❌ No                │
│  Transaction control    │  ❌ No COMMIT/ROLLBACK│  ✅ COMMIT/ROLLBACK   │
│  Can be used in index   │  ✅ (if IMMUTABLE)    │  ❌ No                │
│  Available since        │  Always               │  PostgreSQL 11+       │
│                         │                       │                       │
│  Use for:               │  Computations, data   │  Multi-step business  │
│                         │  retrieval, transforms│  logic with txn ctrl  │
└────────────────────────┴───────────────────────┴───────────────────────┘
```

---

## 🎯 4. Triggers — Automatic Event-Driven Logic

Triggers execute **automatically** when specific events occur on a table.

### Trigger Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                    TRIGGER TIMING × EVENT MATRIX                    │
│                                                                     │
│              │  INSERT  │  UPDATE  │  DELETE  │  TRUNCATE  │       │
│  ────────────┼──────────┼──────────┼──────────┼────────────┤       │
│  BEFORE      │    ✅    │    ✅    │    ✅    │     ✅     │       │
│  AFTER       │    ✅    │    ✅    │    ✅    │     ✅     │       │
│  INSTEAD OF  │  (views) │  (views) │  (views) │     ❌     │       │
│                                                                     │
│  Row-level (FOR EACH ROW) — fires once per affected row            │
│  Statement-level (FOR EACH STATEMENT) — fires once per statement   │
│                                                                     │
│  Special variables in trigger functions:                            │
│  • NEW  → The new row (INSERT/UPDATE)                              │
│  • OLD  → The old row (UPDATE/DELETE)                              │
│  • TG_OP → Operation: 'INSERT', 'UPDATE', 'DELETE'                │
│  • TG_TABLE_NAME → Table name                                      │
│  • TG_WHEN → 'BEFORE' or 'AFTER'                                  │
└────────────────────────────────────────────────────────────────────┘
```

### Practical Trigger Examples

```sql
-- ═══════════════════════════════════════════════════
--  EXAMPLE 1: Auto-Update "updated_at" Timestamp
-- ═══════════════════════════════════════════════════

-- Reusable trigger function (works for ANY table!)
CREATE OR REPLACE FUNCTION update_modified_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to any table that has an updated_at column
CREATE TRIGGER trg_users_updated
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_timestamp();

CREATE TRIGGER trg_orders_updated
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_timestamp();

-- Now updated_at is ALWAYS current after any UPDATE! ✅
```

```sql
-- ═══════════════════════════════════════════════════
--  EXAMPLE 2: Audit Trail — Track All Changes
-- ═══════════════════════════════════════════════════

CREATE TABLE audit_log (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT NOT NULL,
    action      TEXT NOT NULL,
    old_data    JSONB,
    new_data    JSONB,
    changed_by  TEXT DEFAULT current_user,
    changed_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, action, old_data)
        VALUES (TG_TABLE_NAME, 'DELETE', to_jsonb(OLD));
        RETURN OLD;
        
    ELSIF TG_OP = 'UPDATE' THEN
        -- Only log if data actually changed
        IF OLD IS DISTINCT FROM NEW THEN
            INSERT INTO audit_log (table_name, action, old_data, new_data)
            VALUES (TG_TABLE_NAME, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW));
        END IF;
        RETURN NEW;
        
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, action, new_data)
        VALUES (TG_TABLE_NAME, 'INSERT', to_jsonb(NEW));
        RETURN NEW;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Apply to tables you want to audit
CREATE TRIGGER trg_audit_users
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();

CREATE TRIGGER trg_audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

```sql
-- ═══════════════════════════════════════════════════
--  EXAMPLE 3: Data Validation (BEFORE trigger)
-- ═══════════════════════════════════════════════════

CREATE OR REPLACE FUNCTION validate_order()
RETURNS TRIGGER AS $$
BEGIN
    -- Ensure amount is positive
    IF NEW.amount <= 0 THEN
        RAISE EXCEPTION 'Order amount must be positive. Got: %', NEW.amount;
    END IF;
    
    -- Auto-normalize email to lowercase
    NEW.email = LOWER(TRIM(NEW.email));
    
    -- Set default status for new orders
    IF TG_OP = 'INSERT' AND NEW.status IS NULL THEN
        NEW.status = 'pending';
    END IF;
    
    -- Prevent changing status back to 'pending' from 'shipped'
    IF TG_OP = 'UPDATE' 
       AND OLD.status = 'shipped' 
       AND NEW.status = 'pending' THEN
        RAISE EXCEPTION 'Cannot change status from shipped back to pending';
    END IF;
    
    RETURN NEW;  -- MUST return NEW for BEFORE triggers (or NULL to cancel)
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_validate_order
    BEFORE INSERT OR UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION validate_order();
```

```sql
-- ═══════════════════════════════════════════════════
--  EXAMPLE 4: Conditional Trigger (WHEN clause)
-- ═══════════════════════════════════════════════════

-- Only fire when status changes to 'shipped'
CREATE TRIGGER trg_order_shipped
    AFTER UPDATE OF status ON orders
    FOR EACH ROW
    WHEN (OLD.status IS DISTINCT FROM NEW.status AND NEW.status = 'shipped')
    EXECUTE FUNCTION notify_shipping();

-- Only fire on large orders
CREATE TRIGGER trg_large_order_alert
    AFTER INSERT ON orders
    FOR EACH ROW
    WHEN (NEW.amount > 100000)
    EXECUTE FUNCTION alert_large_order();
```

### Trigger Management

```sql
-- List triggers on a table
SELECT trigger_name, event_manipulation, action_timing
FROM information_schema.triggers
WHERE event_object_table = 'orders';

-- Disable a trigger (temporarily)
ALTER TABLE orders DISABLE TRIGGER trg_audit_orders;

-- Enable it back
ALTER TABLE orders ENABLE TRIGGER trg_audit_orders;

-- Disable ALL triggers on a table (useful for bulk loads)
ALTER TABLE orders DISABLE TRIGGER ALL;
ALTER TABLE orders ENABLE TRIGGER ALL;

-- Drop a trigger
DROP TRIGGER IF EXISTS trg_audit_orders ON orders;
```

---

## 🛡️ 5. Error Handling — Be Defensive

```sql
CREATE OR REPLACE FUNCTION safe_divide(
    p_numerator NUMERIC,
    p_denominator NUMERIC
)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    v_result NUMERIC;
BEGIN
    -- Main logic
    v_result := p_numerator / p_denominator;
    RETURN v_result;
    
EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Division by zero — returning NULL';
        RETURN NULL;
        
    WHEN numeric_value_out_of_range THEN
        RAISE WARNING 'Result too large for numeric type';
        RETURN NULL;
        
    WHEN OTHERS THEN
        -- Catch-all handler
        RAISE WARNING 'Unexpected error: % (SQLSTATE: %)', SQLERRM, SQLSTATE;
        RETURN NULL;
END;
$$;
```

### RAISE — Logging & Custom Errors

```sql
DO $$
BEGIN
    -- Different severity levels
    RAISE DEBUG 'Debugging info';       -- Only visible if client_min_messages = debug
    RAISE LOG 'Logged to server log';   -- Goes to server log file
    RAISE INFO 'Informational';         -- Sent to client
    RAISE NOTICE 'Gentle notice';       -- Default notification level
    RAISE WARNING 'Something fishy';    -- Warning to client
    RAISE EXCEPTION 'FATAL — stops execution here!';  -- Aborts transaction!
    
    -- This line NEVER executes after EXCEPTION ↑
    RAISE NOTICE 'You will never see this';
END;
$$;
```

```sql
-- Custom exceptions with error codes and details
CREATE OR REPLACE FUNCTION create_user(p_email TEXT, p_name TEXT)
RETURNS INT
LANGUAGE plpgsql
AS $$
DECLARE
    v_user_id INT;
BEGIN
    -- Validate email
    IF p_email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        RAISE EXCEPTION 'Invalid email format: %', p_email
            USING ERRCODE = '22023',           -- invalid_parameter_value
                  HINT = 'Email must be in format: user@domain.com',
                  DETAIL = 'The provided email "' || p_email || '" is malformed';
    END IF;
    
    -- Insert user
    INSERT INTO users (email, name) 
    VALUES (LOWER(TRIM(p_email)), TRIM(p_name))
    RETURNING id INTO v_user_id;
    
    RETURN v_user_id;

EXCEPTION
    WHEN unique_violation THEN
        RAISE EXCEPTION 'User with email % already exists', p_email
            USING ERRCODE = '23505';
END;
$$;
```

### Common PostgreSQL Error Codes

```
┌───────────────────────┬────────────────────────────────────────┐
│  Error Code (SQLSTATE)│  Condition Name                         │
├───────────────────────┼────────────────────────────────────────┤
│  23505               │  unique_violation                       │
│  23503               │  foreign_key_violation                  │
│  23502               │  not_null_violation                     │
│  23514               │  check_violation                        │
│  22012               │  division_by_zero                       │
│  22023               │  invalid_parameter_value                │
│  42P01               │  undefined_table                        │
│  42703               │  undefined_column                       │
│  P0001               │  raise_exception (user-defined)         │
│  P0002               │  no_data_found                          │
│  P0003               │  too_many_rows                          │
│  40001               │  serialization_failure                  │
│  40P01               │  deadlock_detected                      │
│  57014               │  query_canceled (timeout)               │
└───────────────────────┴────────────────────────────────────────┘
```

---

## 📜 6. Cursors — Row-by-Row Processing

Cursors let you process large result sets **one row at a time** without loading everything into memory.

```sql
CREATE OR REPLACE FUNCTION process_large_table()
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    cur CURSOR FOR SELECT id, name, email FROM users WHERE is_active = true;
    rec RECORD;
    v_count INT := 0;
BEGIN
    OPEN cur;
    
    LOOP
        FETCH cur INTO rec;
        EXIT WHEN NOT FOUND;
        
        -- Process each row
        v_count := v_count + 1;
        
        -- Do something with rec.id, rec.name, rec.email
        IF rec.email IS NULL THEN
            UPDATE users SET is_active = false WHERE id = rec.id;
        END IF;
        
        -- Commit every 1000 rows (in a procedure, not function)
        -- IF v_count % 1000 = 0 THEN COMMIT; END IF;
    END LOOP;
    
    CLOSE cur;
    RAISE NOTICE 'Processed % rows', v_count;
END;
$$;
```

```sql
-- Simpler approach: FOR loop with implicit cursor
CREATE OR REPLACE FUNCTION process_users_simple()
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    rec RECORD;
BEGIN
    -- FOR automatically opens, fetches, and closes the cursor
    FOR rec IN SELECT * FROM users WHERE is_active = true LOOP
        RAISE NOTICE 'Processing: %', rec.name;
    END LOOP;
    -- No need to OPEN/FETCH/CLOSE!
END;
$$;
```

> 💡 **When to use cursors:**
> - Processing millions of rows that don't fit in work_mem
> - Need to commit in batches (procedures only)
> - Row-by-row processing with complex logic
> 
> **When NOT to use cursors:**
> - A simple `UPDATE ... WHERE` does the job (set-based = always faster!)
> - Result set is small
> - **Rule: Set-based SQL > Cursors. Always try set-based first.**

---

## 🔀 7. Dynamic SQL — EXECUTE

Sometimes you need to build SQL strings at runtime:

```sql
CREATE OR REPLACE FUNCTION dynamic_query(
    p_table TEXT,
    p_column TEXT,
    p_value TEXT
)
RETURNS SETOF RECORD
LANGUAGE plpgsql
AS $$
BEGIN
    -- ⚠️ DANGEROUS — SQL injection risk!
    -- EXECUTE 'SELECT * FROM ' || p_table || ' WHERE ' || p_column || ' = ''' || p_value || '''';
    
    -- ✅ SAFE — use format() with %I (identifier) and %L (literal)
    RETURN QUERY EXECUTE format(
        'SELECT * FROM %I WHERE %I = %L',
        p_table,    -- %I: safely quotes as identifier
        p_column,   -- %I: safely quotes as identifier
        p_value     -- %L: safely quotes as literal
    );
END;
$$;

-- Even safer with USING clause
CREATE OR REPLACE FUNCTION search_table(
    p_table TEXT,
    p_column TEXT, 
    p_value ANYELEMENT
)
RETURNS SETOF RECORD
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY EXECUTE format(
        'SELECT * FROM %I WHERE %I = $1',  -- $1 is a parameter placeholder
        p_table,
        p_column
    ) USING p_value;    -- Pass value safely as parameter
END;
$$;
```

```
┌────────────────────────────────────────────────────────────────────┐
│  format() SPECIFIERS FOR SAFE DYNAMIC SQL                          │
├──────────────┬─────────────────────────────────────────────────────┤
│  %I          │  Identifier (table/column name) — auto-quoted      │
│              │  format('%I', 'my table')  → "my table"            │
│              │  format('%I', 'users')     → users                 │
├──────────────┼─────────────────────────────────────────────────────┤
│  %L          │  Literal (value) — auto-quoted and escaped         │
│              │  format('%L', 'O''Brien')  → 'O''Brien'            │
│              │  format('%L', NULL)        → NULL                  │
├──────────────┼─────────────────────────────────────────────────────┤
│  %s          │  Simple string (NO escaping — ⚠️ dangerous!)      │
│              │  Only use for trusted values                       │
├──────────────┼─────────────────────────────────────────────────────┤
│  USING $1,$2 │  Parameterized values (safest for data)            │
│              │  Prevents SQL injection completely                  │
└──────────────┴─────────────────────────────────────────────────────┘
```

> ⚠️ **SQL INJECTION WARNING:** Never concatenate user input directly into dynamic SQL. Always use `format(%I, %L)` or `USING` parameters.

---

## 🧩 8. DO Blocks — Anonymous Code Blocks

`DO` blocks are like unnamed functions — great for one-off scripts and migrations.

```sql
-- Quick data migration script
DO $$
DECLARE
    v_count INT;
BEGIN
    -- Add column if it doesn't exist
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns 
        WHERE table_name = 'users' AND column_name = 'phone'
    ) THEN
        ALTER TABLE users ADD COLUMN phone TEXT;
        RAISE NOTICE 'Column "phone" added';
    ELSE
        RAISE NOTICE 'Column "phone" already exists — skipping';
    END IF;
    
    -- Backfill data
    UPDATE users SET phone = '+91' || (1000000000 + id)::text 
    WHERE phone IS NULL;
    GET DIAGNOSTICS v_count = ROW_COUNT;
    RAISE NOTICE 'Updated % rows', v_count;
    
END;
$$;
```

```sql
-- Batch delete with progress reporting
DO $$
DECLARE
    v_deleted INT;
    v_total INT := 0;
BEGIN
    LOOP
        DELETE FROM large_log_table
        WHERE created_at < NOW() - INTERVAL '90 days'
        AND id IN (
            SELECT id FROM large_log_table 
            WHERE created_at < NOW() - INTERVAL '90 days'
            LIMIT 10000         -- Delete in batches of 10,000
        );
        
        GET DIAGNOSTICS v_deleted = ROW_COUNT;
        v_total := v_total + v_deleted;
        
        EXIT WHEN v_deleted = 0;
        
        RAISE NOTICE 'Deleted % rows so far...', v_total;
        
        -- Let other queries breathe
        PERFORM pg_sleep(0.1);
    END LOOP;
    
    RAISE NOTICE 'Done! Total deleted: %', v_total;
END;
$$;
```

---

## 🔌 9. Extensions Deep Dive — The Power Arsenal

### 9A. pgvector — AI/ML Vector Similarity Search

```sql
-- Install pgvector
CREATE EXTENSION vector;

-- Create table with vector column (1536 dimensions = OpenAI embedding size)
CREATE TABLE documents (
    id        INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title     TEXT NOT NULL,
    content   TEXT NOT NULL,
    embedding vector(1536)     -- 1536-dimensional vector
);

-- Insert embeddings (typically from your AI model)
INSERT INTO documents (title, content, embedding) VALUES
('PostgreSQL Guide', 'Learn about PostgreSQL...', '[0.1, 0.2, 0.3, ...]');

-- Create an index for fast similarity search
CREATE INDEX idx_docs_embedding ON documents 
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);    -- Number of clusters

-- Or use HNSW (faster queries, slower builds)
CREATE INDEX idx_docs_embedding_hnsw ON documents 
    USING hnsw (embedding vector_cosine_ops);

-- Find 5 most similar documents to a query embedding
SELECT title, content,
    1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector  -- <=> = cosine distance
LIMIT 5;

-- Distance operators:
-- <->  L2 (Euclidean) distance
-- <=>  Cosine distance
-- <#>  Inner product distance (negative)
```

> 💡 **pgvector makes PostgreSQL a vector database!** Perfect for:
> - RAG (Retrieval-Augmented Generation) with LLMs
> - Semantic search
> - Recommendation engines
> - Image similarity search

### 9B. pg_cron — Schedule Jobs Inside PostgreSQL

```sql
-- Install pg_cron (requires shared_preload_libraries config)
CREATE EXTENSION pg_cron;

-- Schedule: Delete old logs every day at 3 AM
SELECT cron.schedule(
    'cleanup-old-logs',
    '0 3 * * *',                              -- Cron expression
    $$DELETE FROM app_logs WHERE created_at < NOW() - INTERVAL '30 days'$$
);

-- Schedule: Refresh materialized view every hour
SELECT cron.schedule(
    'refresh-dashboard',
    '0 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_dashboard_stats'
);

-- Schedule: VACUUM ANALYZE every night
SELECT cron.schedule(
    'nightly-vacuum',
    '0 2 * * *',
    'VACUUM ANALYZE'
);

-- List scheduled jobs
SELECT * FROM cron.job;

-- View job execution history
SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 20;

-- Unschedule a job
SELECT cron.unschedule('cleanup-old-logs');
```

### 9C. TimescaleDB — Time-Series Superpowers

```sql
-- Install TimescaleDB
CREATE EXTENSION timescaledb;

-- Create a regular table
CREATE TABLE sensor_data (
    time        TIMESTAMPTZ NOT NULL,
    sensor_id   INT NOT NULL,
    temperature NUMERIC(5,2),
    humidity    NUMERIC(5,2)
);

-- Convert to a hypertable (TimescaleDB magic!)
SELECT create_hypertable('sensor_data', 'time');
-- Now it auto-partitions by time! ✅

-- Insert data as normal
INSERT INTO sensor_data VALUES
(NOW(), 1, 23.5, 65.2),
(NOW(), 2, 24.1, 62.8);

-- Time-bucket queries (aggregate by time intervals)
SELECT 
    time_bucket('1 hour', time) AS hour,
    sensor_id,
    AVG(temperature) AS avg_temp,
    AVG(humidity) AS avg_humidity
FROM sensor_data
WHERE time > NOW() - INTERVAL '24 hours'
GROUP BY hour, sensor_id
ORDER BY hour DESC;

-- Continuous aggregates (auto-refreshing materialized views!)
CREATE MATERIALIZED VIEW hourly_averages
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', time) AS hour,
    sensor_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp
FROM sensor_data
GROUP BY hour, sensor_id;

-- Enable compression (huge storage savings!)
ALTER TABLE sensor_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Add compression policy (compress data older than 7 days)
SELECT add_compression_policy('sensor_data', INTERVAL '7 days');
```

### 9D. Citus — Distributed PostgreSQL

```sql
-- Install Citus
CREATE EXTENSION citus;

-- Distribute a table across worker nodes
SELECT create_distributed_table('events', 'user_id');  -- Shard by user_id

-- Create a reference table (replicated to all nodes)
SELECT create_reference_table('countries');

-- Query as normal — Citus parallelizes across nodes!
SELECT user_id, count(*) 
FROM events 
WHERE event_type = 'purchase'
GROUP BY user_id 
ORDER BY count DESC 
LIMIT 10;
-- This query runs in PARALLEL across all shards! 🚀
```

### Extension Summary

```
┌───────────────────────────────────────────────────────────────────┐
│               EXTENSION ECOSYSTEM MAP                              │
├────────────────┬──────────────────────────────────────────────────┤
│  Category      │  Extensions                                      │
├────────────────┼──────────────────────────────────────────────────┤
│  Performance   │  pg_stat_statements, pg_hint_plan, pg_prewarm   │
│  Search        │  pg_trgm, unaccent, fuzzystrmatch               │
│  Data Types    │  hstore, citext, ltree, uuid-ossp, pgcrypto     │
│  Geospatial    │  PostGIS, pgrouting                             │
│  AI/ML         │  pgvector, pgml (PostgresML)                    │
│  Time Series   │  TimescaleDB                                     │
│  Distributed   │  Citus                                           │
│  Scheduling    │  pg_cron                                         │
│  Auditing      │  pgaudit                                         │
│  Replication   │  pglogical, wal2json                            │
│  Partitioning  │  pg_partman                                      │
│  Connectivity  │  postgres_fdw, mysql_fdw, oracle_fdw            │
│  Monitoring    │  pg_stat_monitor, pg_qualstats                  │
└────────────────┴──────────────────────────────────────────────────┘
```

---

## 🛠️ 10. Useful Function Patterns — Real-World Library

### Pattern: Upsert with Returning

```sql
CREATE OR REPLACE FUNCTION upsert_user(
    p_email TEXT,
    p_name TEXT,
    p_metadata JSONB DEFAULT '{}'
)
RETURNS TABLE (user_id INT, action TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    INSERT INTO users (email, name, metadata)
    VALUES (LOWER(TRIM(p_email)), TRIM(p_name), p_metadata)
    ON CONFLICT (email) 
    DO UPDATE SET 
        name = EXCLUDED.name,
        metadata = users.metadata || EXCLUDED.metadata,
        updated_at = NOW()
    RETURNING id, 
        CASE WHEN xmax = 0 THEN 'inserted' ELSE 'updated' END;
        -- xmax = 0 means new row (inserted), otherwise updated
END;
$$;

SELECT * FROM upsert_user('ritesh@example.com', 'Ritesh Singh');
-- (1, 'inserted') or (1, 'updated')
```

### Pattern: Pagination with Total Count

```sql
CREATE OR REPLACE FUNCTION paginate_users(
    p_page INT DEFAULT 1,
    p_per_page INT DEFAULT 20,
    p_search TEXT DEFAULT NULL
)
RETURNS TABLE (
    user_id INT,
    user_name TEXT,
    user_email TEXT,
    total_count BIGINT,
    total_pages INT
)
LANGUAGE plpgsql STABLE
AS $$
DECLARE
    v_offset INT := (p_page - 1) * p_per_page;
    v_total BIGINT;
BEGIN
    -- Get total count first
    SELECT count(*) INTO v_total 
    FROM users
    WHERE (p_search IS NULL OR name ILIKE '%' || p_search || '%');
    
    -- Return paginated results with total
    RETURN QUERY
    SELECT 
        u.id,
        u.name,
        u.email,
        v_total,
        CEIL(v_total::float / p_per_page)::int
    FROM users u
    WHERE (p_search IS NULL OR u.name ILIKE '%' || p_search || '%')
    ORDER BY u.id
    LIMIT p_per_page OFFSET v_offset;
END;
$$;

-- Usage:
SELECT * FROM paginate_users(1, 10, 'rit');
```

### Pattern: Retry Logic with Exponential Backoff

```sql
CREATE OR REPLACE PROCEDURE retry_operation(p_max_attempts INT DEFAULT 3)
LANGUAGE plpgsql
AS $$
DECLARE
    v_attempt INT := 0;
    v_delay NUMERIC;
BEGIN
    LOOP
        v_attempt := v_attempt + 1;
        
        BEGIN
            -- Your operation here
            INSERT INTO important_table (data) VALUES ('critical_data');
            
            RAISE NOTICE 'Operation succeeded on attempt %', v_attempt;
            RETURN;  -- Success — exit
            
        EXCEPTION
            WHEN serialization_failure OR deadlock_detected THEN
                IF v_attempt >= p_max_attempts THEN
                    RAISE EXCEPTION 'Operation failed after % attempts: %', 
                        v_attempt, SQLERRM;
                END IF;
                
                -- Exponential backoff: 0.1s, 0.2s, 0.4s...
                v_delay := 0.1 * power(2, v_attempt - 1);
                RAISE NOTICE 'Attempt % failed, retrying in %s...', v_attempt, v_delay;
                PERFORM pg_sleep(v_delay);
                
                -- Rollback to savepoint and retry
                ROLLBACK;
        END;
    END LOOP;
END;
$$;
```

---

## 📝 Chapter Summary — Key Takeaways

```
┌────────────────────────────────────────────────────────────────────┐
│  🧠 REMEMBER THESE:                                               │
│                                                                     │
│  1. PL/pgSQL = SQL + control flow + error handling                 │
│     → DECLARE / BEGIN / EXCEPTION / END structure                  │
│                                                                     │
│  2. Functions: RETURNS scalar | SETOF | TABLE | VOID              │
│     → Mark volatility correctly: IMMUTABLE > STABLE > VOLATILE     │
│     → SQL functions are faster (inlineable) — prefer when possible │
│                                                                     │
│  3. Procedures (PG 11+): Can COMMIT/ROLLBACK inside               │
│     → Called with CALL, not SELECT                                 │
│                                                                     │
│  4. Triggers: BEFORE (modify/validate) vs AFTER (audit/notify)    │
│     → NEW (insert/update), OLD (update/delete), TG_OP             │
│     → Universal updated_at trigger = must-have pattern             │
│                                                                     │
│  5. Error Handling: EXCEPTION WHEN ... THEN                        │
│     → Use ERRCODE for programmatic error handling                  │
│     → RAISE EXCEPTION stops execution & rolls back                 │
│                                                                     │
│  6. Dynamic SQL: Use format(%I, %L) or USING $1 — NEVER concat!   │
│                                                                     │
│  7. Extensions are PostgreSQL's secret weapon:                     │
│     → pgvector (AI/ML), pg_cron (scheduling), TimescaleDB (IoT)   │
│     → Citus (distributed), PostGIS (geo), pg_stat_statements       │
│                                                                     │
│  8. Always prefer SET-BASED SQL over cursors/loops                 │
│     → One UPDATE ... WHERE beats 1 million cursor iterations       │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [2E.5 — PostgreSQL Performance Tuning](./05-PG-Performance.md) 🔴⭐🔥
> *EXPLAIN ANALYZE, pg_stat_statements, index types, partitioning — make PostgreSQL fly.*

---

[← Back to Index](../INDEX.md)
