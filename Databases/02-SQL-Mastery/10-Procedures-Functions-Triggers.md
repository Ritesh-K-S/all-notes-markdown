# 2.10 — Stored Procedures, Functions & Triggers 🟡⭐

> **"SQL tells the database WHAT to do. Procedural SQL tells it HOW to do it — with logic, loops, and decisions."**
> This is where databases stop being simple data stores and become programmable engines.

---

## 🎯 What You'll Master

```
✅ Stored Procedures — reusable server-side programs
✅ User-Defined Functions (UDFs) — custom calculations
✅ Procedure vs Function — the critical differences
✅ Parameters: IN, OUT, INOUT
✅ Variables, IF/ELSE, CASE, loops (WHILE, FOR, LOOP)
✅ Error handling (TRY-CATCH, EXCEPTION blocks)
✅ Cursors — row-by-row processing (and why to avoid them)
✅ Triggers — automatic reactions to data changes
✅ Best practices and dangerous anti-patterns
```

---

## 📦 Part 1: Stored Procedures — Your Database Programs

### What is a Stored Procedure?

A stored procedure is a **precompiled block of SQL + procedural logic** stored on the database server. Think of it as a "function" that lives inside the database.

```
Without Stored Procedures:

App Server                    Database
┌──────────┐                  ┌──────────┐
│ Query 1  │ ───────────────→ │          │
│ Query 2  │ ───────────────→ │  3 round │
│ Query 3  │ ───────────────→ │  trips!  │
└──────────┘                  └──────────┘

With Stored Procedures:

App Server                    Database
┌──────────┐                  ┌──────────────────┐
│ CALL     │ ───────────────→ │ Query 1          │
│ proc()   │                  │ Query 2          │
│          │ ←─────────────── │ Query 3          │
└──────────┘  1 round trip!   │ All run locally! │
                              └──────────────────┘
```

---

### Creating Stored Procedures

#### PostgreSQL (PL/pgSQL)

```sql
CREATE OR REPLACE PROCEDURE transfer_money(
    sender_id   INT,
    receiver_id INT,
    amount      DECIMAL(10,2)
)
LANGUAGE plpgsql
AS $$
DECLARE
    sender_balance DECIMAL(10,2);
BEGIN
    -- Check sender's balance
    SELECT balance INTO sender_balance
    FROM accounts WHERE id = sender_id FOR UPDATE;
    
    IF sender_balance < amount THEN
        RAISE EXCEPTION 'Insufficient funds. Balance: %, Requested: %', 
            sender_balance, amount;
    END IF;
    
    -- Perform transfer
    UPDATE accounts SET balance = balance - amount WHERE id = sender_id;
    UPDATE accounts SET balance = balance + amount WHERE id = receiver_id;
    
    -- Log the transaction
    INSERT INTO transactions (from_id, to_id, amount, txn_date)
    VALUES (sender_id, receiver_id, amount, NOW());
    
    COMMIT;
END;
$$;

-- Call it
CALL transfer_money(1, 2, 500.00);
```

#### SQL Server (T-SQL)

```sql
CREATE OR ALTER PROCEDURE sp_transfer_money
    @sender_id   INT,
    @receiver_id INT,
    @amount      DECIMAL(10,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @sender_balance DECIMAL(10,2);
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        SELECT @sender_balance = balance
        FROM accounts WITH (UPDLOCK) WHERE id = @sender_id;
        
        IF @sender_balance < @amount
        BEGIN
            RAISERROR('Insufficient funds. Balance: %s', 16, 1, @sender_balance);
            RETURN;
        END
        
        UPDATE accounts SET balance = balance - @amount WHERE id = @sender_id;
        UPDATE accounts SET balance = balance + @amount WHERE id = @receiver_id;
        
        INSERT INTO transactions (from_id, to_id, amount, txn_date)
        VALUES (@sender_id, @receiver_id, @amount, GETDATE());
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;  -- Re-raise the error
    END CATCH
END;
GO

-- Call it
EXEC sp_transfer_money @sender_id = 1, @receiver_id = 2, @amount = 500.00;
```

#### MySQL

```sql
DELIMITER //
CREATE PROCEDURE sp_transfer_money(
    IN  p_sender_id   INT,
    IN  p_receiver_id INT,
    IN  p_amount      DECIMAL(10,2)
)
BEGIN
    DECLARE v_sender_balance DECIMAL(10,2);
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    SELECT balance INTO v_sender_balance
    FROM accounts WHERE id = p_sender_id FOR UPDATE;
    
    IF v_sender_balance < p_amount THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Insufficient funds';
    END IF;
    
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_sender_id;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_receiver_id;
    
    INSERT INTO transactions (from_id, to_id, amount, txn_date)
    VALUES (p_sender_id, p_receiver_id, p_amount, NOW());
    
    COMMIT;
END //
DELIMITER ;

-- Call it
CALL sp_transfer_money(1, 2, 500.00);
```

#### Oracle (PL/SQL)

```sql
CREATE OR REPLACE PROCEDURE transfer_money(
    p_sender_id   IN NUMBER,
    p_receiver_id IN NUMBER,
    p_amount      IN NUMBER
) AS
    v_sender_balance NUMBER;
BEGIN
    SELECT balance INTO v_sender_balance
    FROM accounts WHERE id = p_sender_id FOR UPDATE;
    
    IF v_sender_balance < p_amount THEN
        RAISE_APPLICATION_ERROR(-20001, 
            'Insufficient funds. Balance: ' || v_sender_balance);
    END IF;
    
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_sender_id;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_receiver_id;
    
    INSERT INTO transactions (from_id, to_id, amount, txn_date)
    VALUES (p_sender_id, p_receiver_id, p_amount, SYSDATE);
    
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
/

-- Call it
EXEC transfer_money(1, 2, 500.00);
```

---

### Parameter Types: IN, OUT, INOUT

```sql
-- PostgreSQL / Oracle / MySQL
CREATE PROCEDURE get_employee_stats(
    IN  p_department  VARCHAR(50),     -- Input only (read)
    OUT p_count       INT,              -- Output only (write)
    OUT p_avg_salary  DECIMAL(10,2),    -- Output only (write)
    INOUT p_min_salary DECIMAL(10,2)    -- Both input and output (read + write)
)
LANGUAGE plpgsql AS $$
BEGIN
    SELECT COUNT(*), AVG(salary), MIN(salary)
    INTO p_count, p_avg_salary, p_min_salary
    FROM employees
    WHERE department = p_department
      AND salary >= p_min_salary;  -- Using INOUT as input filter
END;
$$;

-- Call with OUT parameters (PostgreSQL)
CALL get_employee_stats('Engineering', NULL, NULL, 50000);
```

```sql
-- SQL Server uses variables for output
CREATE PROCEDURE sp_get_employee_stats
    @department  VARCHAR(50),
    @count       INT OUTPUT,
    @avg_salary  DECIMAL(10,2) OUTPUT
AS
BEGIN
    SELECT @count = COUNT(*), @avg_salary = AVG(salary)
    FROM employees
    WHERE department = @department;
END;
GO

DECLARE @c INT, @a DECIMAL(10,2);
EXEC sp_get_employee_stats 'Engineering', @c OUTPUT, @a OUTPUT;
SELECT @c AS employee_count, @a AS avg_salary;
```

---

## 🔧 Part 2: User-Defined Functions (UDFs)

### Procedure vs Function — Know the Difference

| Feature | Stored Procedure | Function |
|---------|-----------------|----------|
| **Returns** | 0 or more result sets | Exactly one value (scalar) or table |
| **Use in SELECT** | ❌ No | ✅ Yes |
| **Use in WHERE** | ❌ No | ✅ Yes |
| **Side effects** | ✅ Can modify data (INSERT/UPDATE/DELETE) | ❌ Should be pure* |
| **Transaction control** | ✅ COMMIT/ROLLBACK | ❌ Not allowed (usually) |
| **Call syntax** | CALL / EXEC | Used in expressions |

> *PostgreSQL is more relaxed — functions CAN modify data. But it's considered bad practice.

---

### Scalar Functions (Return Single Value)

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION calculate_tax(price DECIMAL, tax_rate DECIMAL DEFAULT 0.08)
RETURNS DECIMAL
LANGUAGE plpgsql
IMMUTABLE  -- Same input always returns same output (enables caching)
AS $$
BEGIN
    RETURN ROUND(price * tax_rate, 2);
END;
$$;

-- Use in queries!
SELECT product_name, price, calculate_tax(price) AS tax
FROM products;

-- SQL Server
CREATE FUNCTION dbo.fn_calculate_tax(@price DECIMAL(10,2), @tax_rate DECIMAL(5,4) = 0.08)
RETURNS DECIMAL(10,2)
AS
BEGIN
    RETURN ROUND(@price * @tax_rate, 2);
END;
GO

SELECT product_name, price, dbo.fn_calculate_tax(price, DEFAULT) AS tax
FROM products;
```

---

### Table-Valued Functions (Return a Table)

```sql
-- PostgreSQL: RETURNS TABLE
CREATE OR REPLACE FUNCTION get_department_employees(dept_name VARCHAR)
RETURNS TABLE (
    emp_name   VARCHAR,
    emp_title  VARCHAR,
    emp_salary DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT name, title, salary
    FROM employees
    WHERE department = dept_name
    ORDER BY salary DESC;
END;
$$;

-- Use like a table!
SELECT * FROM get_department_employees('Engineering');

-- Join with other tables
SELECT de.emp_name, p.project_name
FROM get_department_employees('Engineering') de
JOIN project_assignments pa ON pa.emp_name = de.emp_name
JOIN projects p ON p.id = pa.project_id;
```

```sql
-- SQL Server: Inline Table-Valued Function (best performance!)
CREATE FUNCTION dbo.fn_department_employees(@dept VARCHAR(50))
RETURNS TABLE
AS
RETURN (
    SELECT name, title, salary
    FROM employees
    WHERE department = @dept
);
GO

-- Use with CROSS APPLY (SQL Server superpower!)
SELECT d.dept_name, e.name, e.salary
FROM departments d
CROSS APPLY dbo.fn_department_employees(d.dept_name) e;
```

---

### Function Volatility Categories (PostgreSQL)

```sql
CREATE FUNCTION my_func() RETURNS INT
LANGUAGE plpgsql
IMMUTABLE   -- Never changes for same input (e.g., math). Can be cached.
-- STABLE    -- Doesn't change within a single query (e.g., CURRENT_TIMESTAMP)
-- VOLATILE  -- Can change anytime (e.g., RANDOM()). Default. Always re-evaluated.
AS $$ ... $$;
```

> 💡 Mark functions `IMMUTABLE` when possible — the optimizer can pre-compute and cache results!

---

## 🔄 Part 3: Control Flow — Variables, Conditions, Loops

### Variables

```sql
-- PostgreSQL (PL/pgSQL)
DECLARE
    v_count     INT := 0;
    v_name      VARCHAR(100);
    v_total     DECIMAL(10,2) DEFAULT 0.00;
    v_row       employees%ROWTYPE;  -- Entire row as a variable!
    v_salary    employees.salary%TYPE;  -- Same type as column

-- SQL Server (T-SQL)
DECLARE @count INT = 0;
DECLARE @name VARCHAR(100);
DECLARE @total DECIMAL(10,2) = 0.00;

-- MySQL
DECLARE v_count INT DEFAULT 0;

-- Oracle (PL/SQL)
v_count     NUMBER := 0;
v_name      VARCHAR2(100);
v_row       employees%ROWTYPE;
```

### IF / ELSE

```sql
-- PostgreSQL & Oracle
IF v_salary > 100000 THEN
    v_tier := 'Senior';
ELSIF v_salary > 60000 THEN
    v_tier := 'Mid';
ELSE
    v_tier := 'Junior';
END IF;

-- SQL Server
IF @salary > 100000
    SET @tier = 'Senior';
ELSE IF @salary > 60000
    SET @tier = 'Mid';
ELSE
    SET @tier = 'Junior';

-- MySQL
IF v_salary > 100000 THEN
    SET v_tier = 'Senior';
ELSEIF v_salary > 60000 THEN
    SET v_tier = 'Mid';
ELSE
    SET v_tier = 'Junior';
END IF;
```

### CASE (works in all databases)

```sql
v_tier := CASE 
    WHEN v_salary > 100000 THEN 'Senior'
    WHEN v_salary > 60000  THEN 'Mid'
    ELSE 'Junior'
END;
```

### Loops

```sql
-- WHILE loop (all databases)
-- PostgreSQL / Oracle
WHILE v_counter < 10 LOOP
    v_counter := v_counter + 1;
    -- do work
END LOOP;

-- SQL Server
WHILE @counter < 10
BEGIN
    SET @counter = @counter + 1;
    -- do work
END;

-- FOR loop (PostgreSQL)
FOR i IN 1..10 LOOP
    RAISE NOTICE 'Iteration: %', i;
END LOOP;

-- FOR loop over query results (PostgreSQL)
FOR rec IN SELECT * FROM employees WHERE department = 'Sales' LOOP
    -- rec.name, rec.salary are accessible
    RAISE NOTICE 'Employee: %, Salary: %', rec.name, rec.salary;
END LOOP;

-- LOOP with EXIT (PostgreSQL / Oracle)
LOOP
    FETCH cur INTO v_record;
    EXIT WHEN NOT FOUND;
    -- process v_record
END LOOP;
```

---

## 🔥 Part 4: Error Handling

### PostgreSQL

```sql
CREATE OR REPLACE PROCEDURE safe_divide(a INT, b INT)
LANGUAGE plpgsql AS $$
DECLARE
    result DECIMAL;
BEGIN
    result := a::DECIMAL / b;
    RAISE NOTICE 'Result: %', result;
EXCEPTION
    WHEN division_by_zero THEN
        RAISE WARNING 'Cannot divide by zero!';
    WHEN numeric_value_out_of_range THEN
        RAISE WARNING 'Number too large!';
    WHEN OTHERS THEN
        RAISE WARNING 'Unexpected error: % — %', SQLSTATE, SQLERRM;
END;
$$;
```

### SQL Server

```sql
CREATE PROCEDURE sp_safe_operation
AS
BEGIN
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Risky operations here
        INSERT INTO orders VALUES (...);
        UPDATE inventory SET qty = qty - 1 WHERE ...;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        -- Capture error details
        DECLARE @ErrorMsg NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSev INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        DECLARE @ErrorLine INT = ERROR_LINE();
        DECLARE @ErrorProc NVARCHAR(200) = ERROR_PROCEDURE();
        
        -- Log the error
        INSERT INTO error_log (message, severity, state, line, proc_name, logged_at)
        VALUES (@ErrorMsg, @ErrorSev, @ErrorState, @ErrorLine, @ErrorProc, GETDATE());
        
        -- Re-raise
        THROW;
    END CATCH
END;
```

### MySQL

```sql
DELIMITER //
CREATE PROCEDURE sp_safe_insert(IN p_name VARCHAR(100))
BEGIN
    -- Declare handlers BEFORE variables they might use
    DECLARE EXIT HANDLER FOR 1062  -- Duplicate key
    BEGIN
        SELECT 'Duplicate entry!' AS error_message;
    END;
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SELECT 'Unknown error occurred' AS error_message;
    END;
    
    START TRANSACTION;
    INSERT INTO customers (name) VALUES (p_name);
    COMMIT;
END //
DELIMITER ;
```

### Oracle

```sql
CREATE OR REPLACE PROCEDURE safe_operation AS
BEGIN
    -- Risky code
    INSERT INTO orders VALUES (...);
EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        DBMS_OUTPUT.PUT_LINE('Duplicate key violation');
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('No data found');
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('Multiple rows returned for single-row query');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
        RAISE;  -- Re-raise after logging
END;
/
```

---

## 🖱️ Part 5: Cursors — Row-by-Row Processing

### What is a Cursor?

A cursor lets you process query results **one row at a time**. Think of it as a pointer that walks through a result set.

```
Result Set:
┌──────┬──────────┬────────┐
│ ID   │ Name     │ Salary │
├──────┼──────────┼────────┤
│ 1    │ Alice    │ 95000  │ ← Cursor points here
│ 2    │ Bob      │ 85000  │
│ 3    │ Carol    │ 110000 │
│ 4    │ Dave     │ 80000  │
└──────┴──────────┴────────┘

FETCH → moves cursor to next row
```

### PostgreSQL Cursor

```sql
CREATE OR REPLACE PROCEDURE process_employees()
LANGUAGE plpgsql AS $$
DECLARE
    emp_cursor CURSOR FOR 
        SELECT name, salary FROM employees ORDER BY salary DESC;
    emp_record RECORD;
BEGIN
    OPEN emp_cursor;
    
    LOOP
        FETCH emp_cursor INTO emp_record;
        EXIT WHEN NOT FOUND;
        
        -- Process each row
        IF emp_record.salary > 100000 THEN
            RAISE NOTICE 'High earner: % ($%)', emp_record.name, emp_record.salary;
        END IF;
    END LOOP;
    
    CLOSE emp_cursor;
END;
$$;
```

### SQL Server Cursor

```sql
DECLARE @name VARCHAR(50), @salary DECIMAL(10,2);

DECLARE emp_cursor CURSOR FAST_FORWARD FOR  -- FAST_FORWARD = read-only, forward-only
    SELECT name, salary FROM employees ORDER BY salary DESC;

OPEN emp_cursor;
FETCH NEXT FROM emp_cursor INTO @name, @salary;

WHILE @@FETCH_STATUS = 0
BEGIN
    -- Process each row
    IF @salary > 100000
        PRINT 'High earner: ' + @name;
    
    FETCH NEXT FROM emp_cursor INTO @name, @salary;
END;

CLOSE emp_cursor;
DEALLOCATE emp_cursor;
```

### ⚠️ When to Use Cursors (Almost Never!)

```
❌ AVOID CURSORS WHEN:
   • A set-based SQL statement can do the same thing (99% of cases)
   • Processing large datasets (cursors are 10-100x slower)
   • Simple UPDATE/INSERT/DELETE operations

✅ USE CURSORS WHEN:
   • Calling stored procedures for each row
   • Complex business logic that can't be expressed in SQL
   • Sending emails/notifications per row
   • External API calls per row
   • Administrative tasks (database maintenance)
```

```sql
-- ❌ CURSOR APPROACH (slow!)
DECLARE cursor FOR SELECT id, salary FROM employees;
LOOP
    FETCH cursor INTO v_id, v_salary;
    UPDATE employees SET bonus = v_salary * 0.1 WHERE id = v_id;
END LOOP;

-- ✅ SET-BASED APPROACH (100x faster!)
UPDATE employees SET bonus = salary * 0.1;
```

> 💡 **Rule:** If you're reaching for a cursor, ask yourself: "Can I do this with a single UPDATE/INSERT/DELETE?" 90% of the time, the answer is yes.

---

## ⚡ Part 6: Triggers — Automatic Reactions

### What is a Trigger?

A trigger is code that **automatically executes** when a specific event happens on a table — INSERT, UPDATE, or DELETE.

```
Without Triggers:                With Triggers:

App: INSERT INTO orders...       App: INSERT INTO orders...
App: UPDATE inventory...         ← Trigger auto-updates inventory!
App: INSERT INTO audit_log...    ← Trigger auto-logs the change!
App: Send notification...        ← Trigger auto-notifies!

(Developer must remember         (Database enforces it — impossible
 every side effect)               to forget!)
```

---

### Trigger Timing & Events

```
┌──────────────────────────────────────────────────┐
│              TRIGGER MATRIX                       │
├────────────┬──────────┬──────────┬───────────────┤
│            │  INSERT  │  UPDATE  │   DELETE      │
├────────────┼──────────┼──────────┼───────────────┤
│ BEFORE     │ Validate │ Validate │ Prevent?      │
│ (modify    │ defaults │ changes  │ soft delete?  │
│  new data) │          │          │               │
├────────────┼──────────┼──────────┼───────────────┤
│ AFTER      │ Audit    │ Audit    │ Audit         │
│ (react to  │ Notify   │ Sync     │ Cascade       │
│  change)   │ Log      │ Cache    │ Clean up      │
├────────────┼──────────┼──────────┼───────────────┤
│ INSTEAD OF │ Custom   │ Custom   │ Custom        │
│ (replace   │ logic    │ logic    │ logic         │
│  action)   │ (views)  │ (views)  │ (views)       │
└────────────┴──────────┴──────────┴───────────────┘
```

---

### PostgreSQL Triggers

```sql
-- Step 1: Create the trigger function
CREATE OR REPLACE FUNCTION trg_audit_employee_changes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO employee_audit (action, emp_id, new_data, changed_at, changed_by)
        VALUES ('INSERT', NEW.id, row_to_json(NEW), NOW(), current_user);
        RETURN NEW;
        
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO employee_audit (action, emp_id, old_data, new_data, changed_at, changed_by)
        VALUES ('UPDATE', OLD.id, row_to_json(OLD), row_to_json(NEW), NOW(), current_user);
        RETURN NEW;
        
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO employee_audit (action, emp_id, old_data, changed_at, changed_by)
        VALUES ('DELETE', OLD.id, row_to_json(OLD), NOW(), current_user);
        RETURN OLD;
    END IF;
END;
$$;

-- Step 2: Attach the trigger to a table
CREATE TRIGGER audit_employees
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW
EXECUTE FUNCTION trg_audit_employee_changes();
```

### SQL Server Triggers

```sql
CREATE OR ALTER TRIGGER trg_audit_employees
ON employees
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- INSERT: new rows are in 'inserted' pseudo-table
    INSERT INTO employee_audit (action, emp_id, new_name, new_salary, changed_at)
    SELECT 'INSERT', id, name, salary, GETDATE()
    FROM inserted
    WHERE NOT EXISTS (SELECT 1 FROM deleted WHERE deleted.id = inserted.id);
    
    -- UPDATE: old rows in 'deleted', new rows in 'inserted'
    INSERT INTO employee_audit (action, emp_id, old_salary, new_salary, changed_at)
    SELECT 'UPDATE', i.id, d.salary, i.salary, GETDATE()
    FROM inserted i
    INNER JOIN deleted d ON i.id = d.id;
    
    -- DELETE: old rows are in 'deleted' pseudo-table
    INSERT INTO employee_audit (action, emp_id, old_name, old_salary, changed_at)
    SELECT 'DELETE', id, name, salary, GETDATE()
    FROM deleted
    WHERE NOT EXISTS (SELECT 1 FROM inserted WHERE inserted.id = deleted.id);
END;
```

### MySQL Triggers

```sql
-- MySQL: Separate triggers for each event
DELIMITER //

CREATE TRIGGER trg_before_employee_update
BEFORE UPDATE ON employees
FOR EACH ROW
BEGIN
    -- Auto-set updated_at timestamp
    SET NEW.updated_at = NOW();
    
    -- Prevent salary decrease > 20%
    IF NEW.salary < OLD.salary * 0.8 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Salary decrease cannot exceed 20%';
    END IF;
    
    -- Log the change
    INSERT INTO employee_audit (emp_id, old_salary, new_salary, changed_at)
    VALUES (OLD.id, OLD.salary, NEW.salary, NOW());
END //

DELIMITER ;
```

### Oracle Triggers

```sql
CREATE OR REPLACE TRIGGER trg_employee_audit
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW
BEGIN
    IF INSERTING THEN
        INSERT INTO employee_audit VALUES ('INSERT', :NEW.id, :NEW.salary, SYSDATE);
    ELSIF UPDATING THEN
        INSERT INTO employee_audit VALUES ('UPDATE', :OLD.id, :NEW.salary, SYSDATE);
    ELSIF DELETING THEN
        INSERT INTO employee_audit VALUES ('DELETE', :OLD.id, :OLD.salary, SYSDATE);
    END IF;
END;
/
```

---

### Common Trigger Patterns

#### Pattern 1: Auto-Timestamp (updated_at)

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER auto_updated_at
BEFORE UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
-- Now updated_at is ALWAYS correct — no app code needed!
```

#### Pattern 2: Soft Delete

```sql
-- Instead of actually deleting, mark as deleted
CREATE OR REPLACE FUNCTION soft_delete()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE employees 
    SET deleted_at = NOW(), status = 'deleted'
    WHERE id = OLD.id;
    RETURN NULL;  -- Return NULL to cancel the actual DELETE
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER prevent_hard_delete
BEFORE DELETE ON employees
FOR EACH ROW
EXECUTE FUNCTION soft_delete();
```

#### Pattern 3: Denormalized Counter Cache

```sql
-- Keep a count in the parent table, updated by triggers
CREATE OR REPLACE FUNCTION update_order_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE customers SET order_count = order_count + 1 WHERE id = NEW.customer_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE customers SET order_count = order_count - 1 WHERE id = OLD.customer_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_order_count
AFTER INSERT OR DELETE ON orders
FOR EACH ROW
EXECUTE FUNCTION update_order_count();
```

#### Pattern 4: Cascade Business Rules

```sql
-- When an order is marked as 'shipped', update inventory
CREATE TRIGGER trg_order_shipped
AFTER UPDATE ON orders
FOR EACH ROW
WHEN (OLD.status != 'shipped' AND NEW.status = 'shipped')
EXECUTE FUNCTION reduce_inventory();
```

---

## ⚠️ Anti-Patterns & Best Practices

### ❌ Trigger Anti-Patterns

```
1. TRIGGER HELL (too many triggers)
   ─ Trigger A fires Trigger B which fires Trigger C
   ─ Debugging becomes a nightmare
   ─ Performance degrades silently

2. BUSINESS LOGIC IN TRIGGERS
   ─ Complex pricing calculations → move to application layer
   ─ Email notifications → use message queues instead
   ─ Data transformation → use stored procedures explicitly

3. MUTATING TABLE (Oracle)
   ─ A trigger on table X tries to query table X → Error!
   ─ Solution: Use compound triggers or application logic

4. HEAVY TRIGGERS
   ─ Triggers that call APIs, send emails, write to files
   ─ These SLOW DOWN every INSERT/UPDATE/DELETE
   ─ Solution: Write to a queue table, process async
```

### ✅ Best Practices

```
1. Keep triggers SIMPLE
   ─ Audit logging ✅
   ─ Auto-timestamps ✅
   ─ Counter caches ✅
   ─ Complex business logic ❌ (use procedures)

2. Document trigger existence
   ─ Future developers WILL forget they exist
   ─ Comment in code: "⚠️ Trigger: trg_audit exists on this table"

3. Test trigger behavior
   ─ Triggers run invisibly — bugs are hard to find
   ─ Write tests that verify trigger side effects

4. Monitor trigger performance
   ─ A slow trigger slows EVERY write to that table
   ─ Profile trigger execution time

5. Avoid recursive triggers
   ─ Trigger updates table → fires trigger → updates table → ∞
   ─ SQL Server: SET RECURSIVE_TRIGGERS OFF
   ─ PostgreSQL: pg_trigger_depth() to detect recursion
```

---

## 🗄️ Database Syntax Comparison

| Feature | PostgreSQL | SQL Server | MySQL | Oracle |
|---------|-----------|------------|-------|--------|
| **Procedure keyword** | `CREATE PROCEDURE` | `CREATE PROCEDURE` | `CREATE PROCEDURE` | `CREATE PROCEDURE` |
| **Function keyword** | `CREATE FUNCTION` | `CREATE FUNCTION` | `CREATE FUNCTION` | `CREATE FUNCTION` |
| **Language** | PL/pgSQL | T-SQL | SQL/PSM | PL/SQL |
| **Variable prefix** | none | @ | none | none |
| **Parameter prefix** | none | @ | IN/OUT keyword | IN/OUT keyword |
| **String concat** | `\|\|` | `+` | `CONCAT()` | `\|\|` |
| **Error handling** | `EXCEPTION WHEN` | `TRY-CATCH` | `HANDLER` | `EXCEPTION WHEN` |
| **Raise error** | `RAISE EXCEPTION` | `THROW/RAISERROR` | `SIGNAL` | `RAISE_APPLICATION_ERROR` |
| **BEFORE trigger** | ✅ | ❌ | ✅ | ✅ |
| **AFTER trigger** | ✅ | ✅ | ✅ | ✅ |
| **INSTEAD OF** | ✅ | ✅ | ❌ | ✅ |
| **Row-level trigger** | ✅ | ❌ (set-based) | ✅ | ✅ |
| **OLD/NEW reference** | `OLD.*` / `NEW.*` | `deleted` / `inserted` tables | `OLD.*` / `NEW.*` | `:OLD.*` / `:NEW.*` |

---

## 🧪 Practice Challenges

### Challenge 1: Audit Trail
> Create a trigger that logs all salary changes to an audit table, including who changed it and when.

<details>
<summary>💡 Solution (PostgreSQL)</summary>

```sql
CREATE TABLE salary_audit (
    id SERIAL PRIMARY KEY,
    emp_id INT,
    old_salary DECIMAL(10,2),
    new_salary DECIMAL(10,2),
    changed_by TEXT,
    changed_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION log_salary_change()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.salary != NEW.salary THEN
        INSERT INTO salary_audit (emp_id, old_salary, new_salary, changed_by)
        VALUES (OLD.id, OLD.salary, NEW.salary, current_user);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_salary_audit
BEFORE UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION log_salary_change();
```
</details>

### Challenge 2: Stored Procedure with Error Handling
> Create a procedure that enrolls a student in a course, checking prerequisites and seat availability.

<details>
<summary>💡 Solution (PostgreSQL)</summary>

```sql
CREATE OR REPLACE PROCEDURE enroll_student(
    p_student_id INT,
    p_course_id INT
)
LANGUAGE plpgsql AS $$
DECLARE
    v_seats_available INT;
    v_prereq_met BOOLEAN;
BEGIN
    -- Check seat availability
    SELECT max_seats - current_enrollment INTO v_seats_available
    FROM courses WHERE id = p_course_id;
    
    IF v_seats_available <= 0 THEN
        RAISE EXCEPTION 'Course is full. No seats available.';
    END IF;
    
    -- Check prerequisites
    SELECT NOT EXISTS (
        SELECT 1 FROM prerequisites p
        WHERE p.course_id = p_course_id
        AND p.prereq_course_id NOT IN (
            SELECT course_id FROM completed_courses WHERE student_id = p_student_id
        )
    ) INTO v_prereq_met;
    
    IF NOT v_prereq_met THEN
        RAISE EXCEPTION 'Prerequisites not met for this course.';
    END IF;
    
    -- Enroll
    INSERT INTO enrollments (student_id, course_id, enrolled_at)
    VALUES (p_student_id, p_course_id, NOW());
    
    UPDATE courses SET current_enrollment = current_enrollment + 1 
    WHERE id = p_course_id;
    
    COMMIT;
END;
$$;
```
</details>

---

## 🎯 Key Takeaways

```
┌──────────────────────────────────────────────────────────────────┐
│  PROCEDURES, FUNCTIONS & TRIGGERS CHEAT SHEET                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STORED PROCEDURE:                                               │
│  • Server-side program, reduces network round trips              │
│  • Can modify data, control transactions                         │
│  • Called with CALL/EXEC, NOT usable in SELECT                   │
│                                                                  │
│  FUNCTION:                                                       │
│  • Returns a value (scalar or table)                             │
│  • Usable in SELECT, WHERE, JOIN                                 │
│  • Should be pure (no side effects)                              │
│  • Mark IMMUTABLE/DETERMINISTIC for optimizer benefits           │
│                                                                  │
│  TRIGGER:                                                        │
│  • Auto-fires on INSERT/UPDATE/DELETE                            │
│  • BEFORE = validate/modify, AFTER = react/log                  │
│  • Keep simple: audit, timestamps, counters                      │
│  • Don't put business logic in triggers!                         │
│                                                                  │
│  CURSORS:                                                        │
│  • Row-by-row processing (avoid if possible)                     │
│  • Use set-based operations instead (99% of cases)               │
│                                                                  │
│  ERROR HANDLING:                                                 │
│  • PostgreSQL/Oracle: EXCEPTION WHEN ... THEN                    │
│  • SQL Server: BEGIN TRY ... BEGIN CATCH                         │
│  • MySQL: DECLARE HANDLER FOR ...                                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

> 🚀 **Next Up:** [2.11 — Query Optimization & Execution Plans](./11-Query-Optimization.md) — Learn to read execution plans, understand the optimizer, and make your queries fly.

---

*Stored procedures and triggers are the "automation layer" of your database. Used wisely, they enforce consistency and reduce bugs. Used poorly, they create invisible complexity that haunts teams for years. Be wise.* 🧙
