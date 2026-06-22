# ⚡ Chapter 2B.4 — PL/SQL — Oracle's Procedural Language

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~6-8 hours
> **Prerequisites:** Chapter 2B.3 (Oracle SQL Specifics)

> **"SQL tells Oracle WHAT data you want. PL/SQL tells Oracle HOW to process it. Together, they're unstoppable."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Write PL/SQL **anonymous blocks**, **procedures**, and **functions** confidently
- Use **cursors** (implicit & explicit) to process rows like a pro
- Handle **exceptions** gracefully — no more crashing code
- Build reusable **packages** — the backbone of Oracle applications
- Work with **collections** (arrays, nested tables, associative arrays)
- Use **bulk operations** (`FORALL`, `BULK COLLECT`) for 10-100x performance gains
- Understand **triggers** and when to use (and avoid) them

---

## 1. What is PL/SQL & Why Does It Exist?

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  SQL alone can:                                               │
│    ✅ Query data (SELECT)                                    │
│    ✅ Modify data (INSERT, UPDATE, DELETE)                   │
│    ✅ Define schema (CREATE, ALTER, DROP)                    │
│                                                               │
│  SQL alone CANNOT:                                            │
│    ❌ Use IF-ELSE logic                                      │
│    ❌ Loop through records                                    │
│    ❌ Declare variables                                       │
│    ❌ Handle errors gracefully                                │
│    ❌ Build reusable modules                                  │
│    ❌ Perform complex business logic                          │
│                                                               │
│  PL/SQL = SQL + Procedural Logic                              │
│  (Procedural Language extensions to SQL)                      │
│                                                               │
│  Think of it as:                                              │
│    SQL  = Individual LEGO bricks                              │
│    PL/SQL = The instruction manual to build something amazing │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### PL/SQL vs Other Procedural SQL

| Feature | PL/SQL (Oracle) | T-SQL (SQL Server) | PL/pgSQL (PostgreSQL) |
|---------|----------------|--------------------|-----------------------|
| **Block structure** | BEGIN...END; | BEGIN...END | BEGIN...END; |
| **Variable declaration** | DECLARE section | DECLARE @var | DECLARE section |
| **Error handling** | EXCEPTION block | TRY...CATCH | EXCEPTION block |
| **Packages** | ✅ Yes (unique!) | ❌ No | ❌ No |
| **Collections** | ✅ Rich support | ❌ Limited | ✅ Arrays |
| **Bulk operations** | FORALL, BULK COLLECT | ❌ (uses sets) | ❌ (uses sets) |
| **Autonomous transactions** | ✅ Yes | ❌ No | ❌ No |
| **Object types** | ✅ Full OOP support | ❌ Limited | ✅ Some |

---

## 2. PL/SQL Block Structure — The Foundation

Every PL/SQL program is made of **blocks**. There are three sections:

```
┌──────────────────────────────────────────────────┐
│                                                   │
│  DECLARE        ← Optional: Variables, cursors,  │
│    ...             constants, types               │
│                                                   │
│  BEGIN           ← Mandatory: Executable code     │
│    ...             (SQL + procedural logic)        │
│                                                   │
│  EXCEPTION       ← Optional: Error handlers       │
│    ...                                            │
│                                                   │
│  END;            ← Mandatory: Block terminator    │
│  /               ← Executes the block in SQL*Plus │
│                                                   │
└──────────────────────────────────────────────────┘
```

### 2.1 Anonymous Block — Your First PL/SQL Program

```sql
-- Anonymous blocks are NOT stored in the database
-- They run once and are gone

-- Enable output display first:
SET SERVEROUTPUT ON;

-- Hello World!
BEGIN
    DBMS_OUTPUT.PUT_LINE('Hello, PL/SQL World!');
END;
/

-- With variables
DECLARE
    v_name     VARCHAR2(50) := 'Ritesh';
    v_salary   NUMBER(10,2) := 75000;
    v_bonus    NUMBER(10,2);
    c_tax_rate CONSTANT NUMBER := 0.30;   -- Constants can't be changed
BEGIN
    v_bonus := v_salary * 0.15;
    
    DBMS_OUTPUT.PUT_LINE('Employee: ' || v_name);
    DBMS_OUTPUT.PUT_LINE('Salary: ₹' || TO_CHAR(v_salary, '99,99,999.99'));
    DBMS_OUTPUT.PUT_LINE('Bonus: ₹' || TO_CHAR(v_bonus, '99,999.99'));
    DBMS_OUTPUT.PUT_LINE('Tax: ₹' || TO_CHAR(v_salary * c_tax_rate, '99,999.99'));
    DBMS_OUTPUT.PUT_LINE('Net: ₹' || TO_CHAR(v_salary + v_bonus - v_salary * c_tax_rate, '99,999.99'));
END;
/
```

### 2.2 Variable Declarations & Data Types

```sql
DECLARE
    -- Basic types
    v_id          NUMBER;
    v_name        VARCHAR2(100);
    v_is_active   BOOLEAN := TRUE;           -- Boolean exists in PL/SQL (not SQL!)
    v_hire_date   DATE := SYSDATE;
    v_salary      NUMBER(10,2) DEFAULT 50000; -- DEFAULT works like :=
    
    -- %TYPE — Anchor to a column's data type (RECOMMENDED!)
    v_emp_name    employees.first_name%TYPE;  -- Same type as employees.first_name
    v_emp_salary  employees.salary%TYPE;      -- Same type as employees.salary
    -- If column type changes, your variable changes automatically!
    
    -- %ROWTYPE — Anchor to an entire row
    v_emp_record  employees%ROWTYPE;          -- Has ALL columns of employees
    
    -- NOT NULL constraint
    v_count       NUMBER NOT NULL := 0;       -- Must be initialized!
    
    -- Subtypes
    v_positive    POSITIVE := 1;              -- Must be > 0
    v_natural     NATURAL := 0;               -- Must be >= 0
    v_small       PLS_INTEGER;                -- Faster integer (-2^31 to 2^31)
    v_big_text    CLOB;                       -- Large text
BEGIN
    -- Using %TYPE
    SELECT first_name, salary INTO v_emp_name, v_emp_salary
    FROM employees WHERE employee_id = 100;
    
    DBMS_OUTPUT.PUT_LINE(v_emp_name || ' earns ' || v_emp_salary);
    
    -- Using %ROWTYPE
    SELECT * INTO v_emp_record
    FROM employees WHERE employee_id = 100;
    
    DBMS_OUTPUT.PUT_LINE(v_emp_record.first_name || ' ' || 
                         v_emp_record.last_name || 
                         ' in dept ' || v_emp_record.department_id);
END;
/
```

> 💡 **Pro Tip**: ALWAYS use `%TYPE` and `%ROWTYPE` instead of hardcoding types. If the column type changes from `NUMBER(8,2)` to `NUMBER(10,2)`, your code adapts automatically. This is defensive programming.

---

## 3. Control Structures — Logic & Flow

### 3.1 IF-THEN-ELSIF-ELSE

```sql
DECLARE
    v_salary employees.salary%TYPE;
    v_grade  VARCHAR2(1);
BEGIN
    SELECT salary INTO v_salary 
    FROM employees WHERE employee_id = 100;
    
    IF v_salary >= 15000 THEN
        v_grade := 'A';
    ELSIF v_salary >= 10000 THEN
        v_grade := 'B';
    ELSIF v_salary >= 5000 THEN
        v_grade := 'C';
    ELSE
        v_grade := 'D';
    END IF;
    
    DBMS_OUTPUT.PUT_LINE('Salary: ' || v_salary || ' → Grade: ' || v_grade);
END;
/
```

### 3.2 CASE Statement & Expression

```sql
DECLARE
    v_dept_id   NUMBER := 20;
    v_dept_name VARCHAR2(50);
    v_bonus     NUMBER;
    v_salary    NUMBER := 8000;
BEGIN
    -- CASE Statement (like switch)
    CASE v_dept_id
        WHEN 10 THEN v_dept_name := 'Administration';
        WHEN 20 THEN v_dept_name := 'Marketing';
        WHEN 30 THEN v_dept_name := 'Purchasing';
        WHEN 40 THEN v_dept_name := 'HR';
        ELSE v_dept_name := 'Other';
    END CASE;
    
    -- Searched CASE (with conditions)
    v_bonus := CASE
        WHEN v_salary > 15000 THEN v_salary * 0.20
        WHEN v_salary > 10000 THEN v_salary * 0.15
        WHEN v_salary > 5000  THEN v_salary * 0.10
        ELSE v_salary * 0.05
    END;
    
    DBMS_OUTPUT.PUT_LINE(v_dept_name || ' — Bonus: ' || v_bonus);
END;
/
```

### 3.3 Loops — Three Types

```sql
-- ═══════════════════════════════════════════════════
-- 1. BASIC LOOP (loop...exit when...end loop)
-- ═══════════════════════════════════════════════════
DECLARE
    v_counter NUMBER := 1;
BEGIN
    LOOP
        DBMS_OUTPUT.PUT_LINE('Counter: ' || v_counter);
        v_counter := v_counter + 1;
        EXIT WHEN v_counter > 5;    -- Exit condition
    END LOOP;
END;
/

-- ═══════════════════════════════════════════════════
-- 2. WHILE LOOP (check condition BEFORE each iteration)
-- ═══════════════════════════════════════════════════
DECLARE
    v_counter NUMBER := 1;
BEGIN
    WHILE v_counter <= 5 LOOP
        DBMS_OUTPUT.PUT_LINE('Counter: ' || v_counter);
        v_counter := v_counter + 1;
    END LOOP;
END;
/

-- ═══════════════════════════════════════════════════
-- 3. FOR LOOP (definite iteration — most common!)
-- ═══════════════════════════════════════════════════
BEGIN
    -- Forward loop (1 to 10)
    FOR i IN 1..10 LOOP
        DBMS_OUTPUT.PUT_LINE('i = ' || i);
    END LOOP;
    
    -- Reverse loop (10 down to 1)
    FOR i IN REVERSE 1..10 LOOP
        DBMS_OUTPUT.PUT_LINE('i = ' || i);
    END LOOP;
    
    -- Note: Loop variable (i) is auto-declared, read-only,
    -- and only visible inside the loop
END;
/

-- ═══════════════════════════════════════════════════
-- CONTINUE & CONTINUE WHEN (11g+)
-- ═══════════════════════════════════════════════════
BEGIN
    FOR i IN 1..20 LOOP
        CONTINUE WHEN MOD(i, 2) = 0;  -- Skip even numbers
        DBMS_OUTPUT.PUT_LINE('Odd: ' || i);
    END LOOP;
END;
/

-- ═══════════════════════════════════════════════════
-- NESTED LOOPS with LABELS
-- ═══════════════════════════════════════════════════
BEGIN
    <<outer_loop>>
    FOR i IN 1..3 LOOP
        <<inner_loop>>
        FOR j IN 1..3 LOOP
            EXIT outer_loop WHEN i = 2 AND j = 2;  -- Exit BOTH loops
            DBMS_OUTPUT.PUT_LINE(i || ',' || j);
        END LOOP inner_loop;
    END LOOP outer_loop;
END;
/
```

---

## 4. Cursors — Processing Rows One-by-One

A cursor is a **pointer to the result set** of a query. It lets you process rows one at a time.

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  Think of a cursor like a BOOKMARK in a book of results:     │
│                                                               │
│  Result Set:                                                  │
│  ┌───────────────────────────────────┐                       │
│  │ Row 1: Steven, 24000             │                       │
│  │ Row 2: Neena, 17000        ← Cursor is HERE               │
│  │ Row 3: Lex, 17000               │                       │
│  │ Row 4: Alexander, 9000          │                       │
│  │ Row 5: Bruce, 6000              │                       │
│  └───────────────────────────────────┘                       │
│                                                               │
│  OPEN  → Execute query, create result set                    │
│  FETCH → Read current row, move cursor to next               │
│  CLOSE → Release resources                                    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 4.1 Implicit Cursors (Automatic)

Oracle creates an implicit cursor for every SQL statement inside PL/SQL.

```sql
DECLARE
    v_rows_updated NUMBER;
BEGIN
    -- Every SQL DML creates an implicit cursor
    UPDATE employees SET salary = salary * 1.10
    WHERE department_id = 50;
    
    -- Implicit cursor attributes (use SQL% prefix)
    IF SQL%FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Rows updated: ' || SQL%ROWCOUNT);
    ELSE
        DBMS_OUTPUT.PUT_LINE('No rows found!');
    END IF;
    
    -- SQL%NOTFOUND — TRUE if no rows affected
    -- SQL%FOUND    — TRUE if at least one row affected
    -- SQL%ROWCOUNT — Number of rows affected
    -- SQL%ISOPEN   — Always FALSE for implicit cursors
    
    ROLLBACK;  -- Undo for demo
END;
/

-- SELECT INTO is also an implicit cursor
DECLARE
    v_name employees.first_name%TYPE;
BEGIN
    SELECT first_name INTO v_name
    FROM employees WHERE employee_id = 100;
    
    DBMS_OUTPUT.PUT_LINE('Found: ' || v_name);
    
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Employee not found!');
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('Multiple employees found! Use a cursor.');
END;
/
```

### 4.2 Explicit Cursors (Manual Control)

```sql
-- Full lifecycle: DECLARE → OPEN → FETCH → CLOSE

DECLARE
    -- Step 1: DECLARE the cursor
    CURSOR c_employees IS
        SELECT employee_id, first_name, salary
        FROM employees
        WHERE department_id = 50
        ORDER BY salary DESC;
    
    v_id     employees.employee_id%TYPE;
    v_name   employees.first_name%TYPE;
    v_salary employees.salary%TYPE;
BEGIN
    -- Step 2: OPEN the cursor (executes the query)
    OPEN c_employees;
    
    -- Step 3: FETCH rows one by one
    LOOP
        FETCH c_employees INTO v_id, v_name, v_salary;
        EXIT WHEN c_employees%NOTFOUND;  -- No more rows
        
        DBMS_OUTPUT.PUT_LINE(v_id || ': ' || v_name || ' → ₹' || v_salary);
    END LOOP;
    
    DBMS_OUTPUT.PUT_LINE('Total rows fetched: ' || c_employees%ROWCOUNT);
    
    -- Step 4: CLOSE the cursor (release resources)
    CLOSE c_employees;
END;
/
```

### 4.3 Cursor FOR Loop — The Best Way (99% of the Time)

```sql
-- The Cursor FOR loop handles OPEN, FETCH, and CLOSE automatically!
-- This is the PREFERRED way to use cursors

DECLARE
    CURSOR c_employees IS
        SELECT employee_id, first_name, salary, department_id
        FROM employees
        WHERE salary > 10000
        ORDER BY salary DESC;
BEGIN
    -- No need to OPEN, FETCH, or CLOSE!
    FOR rec IN c_employees LOOP
        DBMS_OUTPUT.PUT_LINE(
            rec.employee_id || ': ' || 
            rec.first_name || ' → ₹' || 
            rec.salary || ' (Dept: ' || rec.department_id || ')'
        );
    END LOOP;
    -- Cursor automatically closed when loop ends
END;
/

-- Even simpler: Inline cursor (no DECLARE needed!)
BEGIN
    FOR rec IN (
        SELECT employee_id, first_name, salary
        FROM employees
        WHERE department_id = 80
        ORDER BY salary DESC
    ) LOOP
        DBMS_OUTPUT.PUT_LINE(rec.first_name || ': ₹' || rec.salary);
    END LOOP;
END;
/
```

### 4.4 Parameterized Cursors

```sql
DECLARE
    -- Cursor with parameters
    CURSOR c_emp_by_dept(p_dept_id NUMBER, p_min_salary NUMBER DEFAULT 0) IS
        SELECT employee_id, first_name, salary
        FROM employees
        WHERE department_id = p_dept_id
          AND salary >= p_min_salary
        ORDER BY salary DESC;
BEGIN
    DBMS_OUTPUT.PUT_LINE('=== Department 50, All Salaries ===');
    FOR rec IN c_emp_by_dept(50) LOOP
        DBMS_OUTPUT.PUT_LINE(rec.first_name || ': ₹' || rec.salary);
    END LOOP;
    
    DBMS_OUTPUT.PUT_LINE('');
    DBMS_OUTPUT.PUT_LINE('=== Department 80, Salary > 8000 ===');
    FOR rec IN c_emp_by_dept(80, 8000) LOOP
        DBMS_OUTPUT.PUT_LINE(rec.first_name || ': ₹' || rec.salary);
    END LOOP;
END;
/
```

### 4.5 REF CURSOR — Dynamic Cursors

```sql
-- REF CURSOR allows you to return cursor results from procedures

-- Strong REF CURSOR (typed — knows the return type)
DECLARE
    TYPE emp_cursor_type IS REF CURSOR RETURN employees%ROWTYPE;
    v_cursor emp_cursor_type;
    v_emp    employees%ROWTYPE;
BEGIN
    OPEN v_cursor FOR 
        SELECT * FROM employees WHERE department_id = 10;
    
    LOOP
        FETCH v_cursor INTO v_emp;
        EXIT WHEN v_cursor%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_emp.first_name || ' ' || v_emp.last_name);
    END LOOP;
    CLOSE v_cursor;
END;
/

-- Weak REF CURSOR (SYS_REFCURSOR — can return any query)
DECLARE
    v_cursor SYS_REFCURSOR;
    v_name   VARCHAR2(50);
    v_salary NUMBER;
BEGIN
    OPEN v_cursor FOR
        'SELECT first_name, salary FROM employees WHERE department_id = :dept'
        USING 50;  -- Bind variable (safe from SQL injection!)
    
    LOOP
        FETCH v_cursor INTO v_name, v_salary;
        EXIT WHEN v_cursor%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_name || ': ' || v_salary);
    END LOOP;
    CLOSE v_cursor;
END;
/
```

---

## 5. Exception Handling — Graceful Error Management

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  Without exceptions:                                          │
│    Error occurs → Program CRASHES → User sees ugly error      │
│                                                               │
│  With exceptions:                                             │
│    Error occurs → EXCEPTION block catches it →                │
│    Log it, handle it, or re-raise with context                │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 5.1 Predefined Exceptions (Oracle-Named)

```sql
DECLARE
    v_name  employees.first_name%TYPE;
    v_salary employees.salary%TYPE;
BEGIN
    -- This might fail in multiple ways
    SELECT first_name, salary INTO v_name, v_salary
    FROM employees
    WHERE department_id = 10;  -- Could return 0 or multiple rows!
    
    DBMS_OUTPUT.PUT_LINE(v_name || ': ' || v_salary);
    
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('No employee found in department 10');
        
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('Multiple employees found — use a cursor!');
        
    WHEN ZERO_DIVIDE THEN
        DBMS_OUTPUT.PUT_LINE('Cannot divide by zero');
        
    WHEN VALUE_ERROR THEN
        DBMS_OUTPUT.PUT_LINE('Type conversion or size error');
        
    WHEN OTHERS THEN
        -- Catch-all (should be LAST)
        DBMS_OUTPUT.PUT_LINE('Unexpected error: ' || SQLERRM);
        DBMS_OUTPUT.PUT_LINE('Error code: ' || SQLCODE);
        -- SQLCODE = Oracle error number (negative)
        -- SQLERRM = Error message text
        RAISE;  -- Re-raise to calling code (don't swallow errors!)
END;
/
```

### Common Predefined Exceptions

| Exception | ORA Code | When It Occurs |
|-----------|----------|---------------|
| `NO_DATA_FOUND` | ORA-01403 | SELECT INTO returns 0 rows |
| `TOO_MANY_ROWS` | ORA-01422 | SELECT INTO returns > 1 row |
| `ZERO_DIVIDE` | ORA-01476 | Division by zero |
| `VALUE_ERROR` | ORA-06502 | Arithmetic, conversion, or truncation error |
| `INVALID_CURSOR` | ORA-01001 | Cursor operation on closed/invalid cursor |
| `CURSOR_ALREADY_OPEN` | ORA-06511 | Opening a cursor that's already open |
| `DUP_VAL_ON_INDEX` | ORA-00001 | Unique constraint violation |
| `INVALID_NUMBER` | ORA-01722 | String-to-number conversion fails |
| `LOGIN_DENIED` | ORA-01017 | Invalid username/password |
| `PROGRAM_ERROR` | ORA-06501 | Internal PL/SQL error |
| `STORAGE_ERROR` | ORA-06500 | Out of memory |
| `TIMEOUT_ON_RESOURCE` | ORA-00051 | Timeout waiting for resource |

### 5.2 User-Defined Exceptions

```sql
DECLARE
    -- Define custom exceptions
    e_salary_too_high EXCEPTION;
    e_invalid_dept    EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_invalid_dept, -2291);  -- Map to ORA error
    
    v_salary NUMBER;
    c_max_salary CONSTANT NUMBER := 100000;
BEGIN
    v_salary := 150000;
    
    -- Manually raise exception
    IF v_salary > c_max_salary THEN
        RAISE e_salary_too_high;
    END IF;
    
EXCEPTION
    WHEN e_salary_too_high THEN
        DBMS_OUTPUT.PUT_LINE('Error: Salary ' || v_salary || 
                             ' exceeds maximum of ' || c_max_salary);
                             
    WHEN e_invalid_dept THEN
        DBMS_OUTPUT.PUT_LINE('Error: Invalid department ID (FK violation)');
END;
/
```

### 5.3 RAISE_APPLICATION_ERROR — Custom Error Codes

```sql
-- RAISE_APPLICATION_ERROR allows you to define custom error numbers (-20000 to -20999)
-- These propagate back to the calling application like standard Oracle errors

CREATE OR REPLACE PROCEDURE give_raise(
    p_emp_id IN NUMBER,
    p_raise_pct IN NUMBER
) AS
    v_current_salary employees.salary%TYPE;
    v_new_salary     NUMBER;
    c_max_salary     CONSTANT NUMBER := 100000;
BEGIN
    -- Validate input
    IF p_raise_pct <= 0 OR p_raise_pct > 50 THEN
        RAISE_APPLICATION_ERROR(-20001, 
            'Raise percentage must be between 1 and 50. Got: ' || p_raise_pct);
    END IF;
    
    SELECT salary INTO v_current_salary
    FROM employees WHERE employee_id = p_emp_id;
    
    v_new_salary := v_current_salary * (1 + p_raise_pct/100);
    
    IF v_new_salary > c_max_salary THEN
        RAISE_APPLICATION_ERROR(-20002,
            'New salary (' || v_new_salary || ') would exceed max (' || c_max_salary || ')');
    END IF;
    
    UPDATE employees SET salary = v_new_salary
    WHERE employee_id = p_emp_id;
    
    DBMS_OUTPUT.PUT_LINE('Salary updated: ' || v_current_salary || ' → ' || v_new_salary);
    
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20003, 
            'Employee ID ' || p_emp_id || ' does not exist');
END;
/

-- Test it:
EXEC give_raise(100, 10);    -- Works
EXEC give_raise(999, 10);    -- ORA-20003: Employee ID 999 does not exist
EXEC give_raise(100, 75);    -- ORA-20001: Raise percentage must be between 1 and 50
```

---

## 6. Stored Procedures — Reusable Code Blocks

Procedures are **named PL/SQL blocks** stored in the database. They don't return a value (unlike functions).

```sql
-- ═══════════════════════════════════════════════════
-- CREATE PROCEDURE
-- ═══════════════════════════════════════════════════
CREATE OR REPLACE PROCEDURE transfer_funds(
    p_from_account IN  NUMBER,        -- IN = input parameter (read-only)
    p_to_account   IN  NUMBER,
    p_amount       IN  NUMBER,
    p_status       OUT VARCHAR2,      -- OUT = output parameter (write-only)
    p_new_balance  OUT NUMBER         -- Caller receives this value
) AS
    v_from_balance NUMBER;
    e_insufficient_funds EXCEPTION;
BEGIN
    -- Check source account balance
    SELECT balance INTO v_from_balance
    FROM accounts WHERE account_id = p_from_account
    FOR UPDATE;  -- Lock the row!
    
    IF v_from_balance < p_amount THEN
        RAISE e_insufficient_funds;
    END IF;
    
    -- Debit source
    UPDATE accounts 
    SET balance = balance - p_amount
    WHERE account_id = p_from_account;
    
    -- Credit destination
    UPDATE accounts 
    SET balance = balance + p_amount
    WHERE account_id = p_to_account;
    
    -- Log the transaction
    INSERT INTO transaction_log (from_acct, to_acct, amount, txn_date)
    VALUES (p_from_account, p_to_account, p_amount, SYSDATE);
    
    -- Set output
    SELECT balance INTO p_new_balance
    FROM accounts WHERE account_id = p_from_account;
    
    p_status := 'SUCCESS';
    COMMIT;
    
EXCEPTION
    WHEN e_insufficient_funds THEN
        p_status := 'FAILED: Insufficient funds';
        p_new_balance := v_from_balance;
        ROLLBACK;
    WHEN NO_DATA_FOUND THEN
        p_status := 'FAILED: Account not found';
        ROLLBACK;
    WHEN OTHERS THEN
        p_status := 'FAILED: ' || SQLERRM;
        ROLLBACK;
        RAISE;  -- Re-raise for logging/debugging
END;
/

-- ═══════════════════════════════════════════════════
-- CALL PROCEDURE
-- ═══════════════════════════════════════════════════
DECLARE
    v_status  VARCHAR2(200);
    v_balance NUMBER;
BEGIN
    transfer_funds(
        p_from_account => 1001,
        p_to_account   => 1002,
        p_amount       => 5000,
        p_status       => v_status,
        p_new_balance  => v_balance
    );
    
    DBMS_OUTPUT.PUT_LINE('Status: ' || v_status);
    DBMS_OUTPUT.PUT_LINE('New Balance: ' || v_balance);
END;
/

-- Or simple call (without OUT params):
EXEC some_procedure(100, 'Hello');
```

### Parameter Modes

| Mode | Direction | Can Read? | Can Write? | Default | Use Case |
|------|-----------|-----------|------------|---------|----------|
| `IN` | Caller → Procedure | ✅ Yes | ❌ No | ✅ Yes | Passing values in |
| `OUT` | Procedure → Caller | ❌ No (NULL initially) | ✅ Yes | No | Returning values |
| `IN OUT` | Both ways | ✅ Yes | ✅ Yes | No | Modify and return |

---

## 7. Functions — Procedures That Return Values

```sql
-- Functions MUST return a value (procedures don't)
-- Functions can be used in SQL statements (procedures can't directly)

-- ═══════════════════════════════════════════════════
-- CREATE FUNCTION
-- ═══════════════════════════════════════════════════
CREATE OR REPLACE FUNCTION calculate_tax(
    p_salary IN NUMBER,
    p_country IN VARCHAR2 DEFAULT 'INDIA'
) RETURN NUMBER
AS
    v_tax NUMBER;
BEGIN
    CASE p_country
        WHEN 'INDIA' THEN
            v_tax := CASE
                WHEN p_salary <= 250000  THEN 0
                WHEN p_salary <= 500000  THEN p_salary * 0.05
                WHEN p_salary <= 1000000 THEN p_salary * 0.20
                ELSE p_salary * 0.30
            END;
        WHEN 'US' THEN
            v_tax := p_salary * 0.22;  -- Simplified
        ELSE
            v_tax := p_salary * 0.15;  -- Default
    END CASE;
    
    RETURN ROUND(v_tax, 2);
END;
/

-- ═══════════════════════════════════════════════════
-- USE FUNCTION
-- ═══════════════════════════════════════════════════

-- In PL/SQL
DECLARE
    v_tax NUMBER;
BEGIN
    v_tax := calculate_tax(750000, 'INDIA');
    DBMS_OUTPUT.PUT_LINE('Tax: ₹' || v_tax);
END;
/

-- In SQL (THIS is the power of functions!)
SELECT employee_id, first_name, salary,
       calculate_tax(salary * 12) AS annual_tax
FROM employees
WHERE department_id = 50;

-- In WHERE clause
SELECT * FROM employees
WHERE calculate_tax(salary * 12) > 100000;

-- Deterministic function (same input → always same output → Oracle can cache it)
CREATE OR REPLACE FUNCTION full_name(
    p_first VARCHAR2, p_last VARCHAR2
) RETURN VARCHAR2 DETERMINISTIC
AS
BEGIN
    RETURN INITCAP(p_first) || ' ' || INITCAP(p_last);
END;
/
```

### Procedure vs Function — When to Use What

| Aspect | Procedure | Function |
|--------|-----------|----------|
| **Returns value?** | No (uses OUT params) | Yes (RETURN statement) |
| **Use in SQL?** | ❌ Cannot | ✅ Can (SELECT, WHERE, etc.) |
| **Call syntax** | `EXEC proc(args)` or `proc(args)` in PL/SQL | `var := func(args)` |
| **Purpose** | Perform actions (DML, business logic) | Compute and return values |
| **Multiple results?** | ✅ (multiple OUT params) | One return value (but can use OUT params too) |
| **Can do DML?** | ✅ Yes | ✅ Yes (but not if called from SQL!) |
| **Best for** | Business processes, data manipulation | Calculations, transformations, queries |

---

## 8. Packages — PL/SQL's Greatest Feature

Packages are **containers** that group related procedures, functions, types, variables, and cursors into a single unit. They're Oracle's answer to **modules/namespaces**.

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  Think of a Package like a TOOLBOX:                          │
│                                                               │
│  ┌─────────────────────────────────────────────────┐         │
│  │  PACKAGE SPECIFICATION (the label on the toolbox)│         │
│  │  • Lists what's INSIDE (public interface)        │         │
│  │  • Other programs can SEE these                  │         │
│  │  • Like a .h header file in C                    │         │
│  └─────────────────────────────────────────────────┘         │
│                                                               │
│  ┌─────────────────────────────────────────────────┐         │
│  │  PACKAGE BODY (the actual tools inside)          │         │
│  │  • Contains the actual CODE                      │         │
│  │  • Can have PRIVATE items too                    │         │
│  │  • Like a .c implementation file                 │         │
│  └─────────────────────────────────────────────────┘         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 8.1 Package Specification (Header)

```sql
CREATE OR REPLACE PACKAGE emp_pkg AS
    -- ═══════════════════════════════════════════
    -- PUBLIC TYPES
    -- ═══════════════════════════════════════════
    TYPE emp_record_type IS RECORD (
        emp_id    employees.employee_id%TYPE,
        emp_name  VARCHAR2(100),
        salary    employees.salary%TYPE,
        dept_name departments.department_name%TYPE
    );
    
    TYPE emp_table_type IS TABLE OF emp_record_type;
    
    -- ═══════════════════════════════════════════
    -- PUBLIC CONSTANTS
    -- ═══════════════════════════════════════════
    c_max_salary    CONSTANT NUMBER := 100000;
    c_min_salary    CONSTANT NUMBER := 3000;
    c_default_dept  CONSTANT NUMBER := 50;
    
    -- ═══════════════════════════════════════════
    -- PUBLIC PROCEDURES & FUNCTIONS
    -- ═══════════════════════════════════════════
    PROCEDURE hire_employee(
        p_first_name IN VARCHAR2,
        p_last_name  IN VARCHAR2,
        p_email      IN VARCHAR2,
        p_salary     IN NUMBER,
        p_dept_id    IN NUMBER DEFAULT c_default_dept,
        p_emp_id     OUT NUMBER
    );
    
    PROCEDURE fire_employee(p_emp_id IN NUMBER);
    
    PROCEDURE give_raise(
        p_emp_id    IN NUMBER,
        p_raise_pct IN NUMBER
    );
    
    FUNCTION get_employee(p_emp_id IN NUMBER) RETURN emp_record_type;
    
    FUNCTION get_dept_employees(p_dept_id IN NUMBER) RETURN emp_table_type PIPELINED;
    
    FUNCTION get_employee_count RETURN NUMBER;
    
END emp_pkg;
/
```

### 8.2 Package Body (Implementation)

```sql
CREATE OR REPLACE PACKAGE BODY emp_pkg AS

    -- ═══════════════════════════════════════════
    -- PRIVATE VARIABLES (not in spec = hidden!)
    -- ═══════════════════════════════════════════
    g_total_hires   NUMBER := 0;  -- Package state (persists for session!)
    g_last_error    VARCHAR2(500);
    
    -- ═══════════════════════════════════════════
    -- PRIVATE PROCEDURE (helper — not in spec)
    -- ═══════════════════════════════════════════
    PROCEDURE log_action(p_action VARCHAR2, p_details VARCHAR2) AS
        PRAGMA AUTONOMOUS_TRANSACTION;  -- Independent transaction!
    BEGIN
        INSERT INTO audit_log (action, details, action_date, action_user)
        VALUES (p_action, p_details, SYSDATE, USER);
        COMMIT;  -- Commits independently of calling transaction
    END log_action;
    
    -- Validate salary (private function)
    FUNCTION is_valid_salary(p_salary NUMBER) RETURN BOOLEAN AS
    BEGIN
        RETURN p_salary BETWEEN c_min_salary AND c_max_salary;
    END is_valid_salary;
    
    -- ═══════════════════════════════════════════
    -- PUBLIC IMPLEMENTATIONS
    -- ═══════════════════════════════════════════
    
    PROCEDURE hire_employee(
        p_first_name IN VARCHAR2,
        p_last_name  IN VARCHAR2,
        p_email      IN VARCHAR2,
        p_salary     IN NUMBER,
        p_dept_id    IN NUMBER DEFAULT c_default_dept,
        p_emp_id     OUT NUMBER
    ) AS
    BEGIN
        IF NOT is_valid_salary(p_salary) THEN
            RAISE_APPLICATION_ERROR(-20010, 
                'Salary must be between ' || c_min_salary || ' and ' || c_max_salary);
        END IF;
        
        SELECT employee_seq.NEXTVAL INTO p_emp_id FROM dual;
        
        INSERT INTO employees (employee_id, first_name, last_name, email, 
                              salary, department_id, hire_date)
        VALUES (p_emp_id, p_first_name, p_last_name, p_email,
                p_salary, p_dept_id, SYSDATE);
        
        g_total_hires := g_total_hires + 1;
        
        log_action('HIRE', 'Hired ' || p_first_name || ' ' || p_last_name || 
                   ' (ID: ' || p_emp_id || ')');
        
        COMMIT;
    EXCEPTION
        WHEN DUP_VAL_ON_INDEX THEN
            RAISE_APPLICATION_ERROR(-20011, 'Email ' || p_email || ' already exists');
        WHEN OTHERS THEN
            g_last_error := SQLERRM;
            log_action('HIRE_ERROR', SQLERRM);
            ROLLBACK;
            RAISE;
    END hire_employee;
    
    PROCEDURE fire_employee(p_emp_id IN NUMBER) AS
        v_name VARCHAR2(100);
    BEGIN
        SELECT first_name || ' ' || last_name INTO v_name
        FROM employees WHERE employee_id = p_emp_id;
        
        DELETE FROM employees WHERE employee_id = p_emp_id;
        
        log_action('FIRE', 'Terminated ' || v_name || ' (ID: ' || p_emp_id || ')');
        COMMIT;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE_APPLICATION_ERROR(-20012, 'Employee ' || p_emp_id || ' not found');
    END fire_employee;
    
    PROCEDURE give_raise(
        p_emp_id    IN NUMBER,
        p_raise_pct IN NUMBER
    ) AS
        v_current_salary employees.salary%TYPE;
        v_new_salary     NUMBER;
    BEGIN
        SELECT salary INTO v_current_salary
        FROM employees WHERE employee_id = p_emp_id
        FOR UPDATE;
        
        v_new_salary := ROUND(v_current_salary * (1 + p_raise_pct/100), 2);
        
        IF NOT is_valid_salary(v_new_salary) THEN
            RAISE_APPLICATION_ERROR(-20013,
                'New salary ' || v_new_salary || ' out of range');
        END IF;
        
        UPDATE employees SET salary = v_new_salary
        WHERE employee_id = p_emp_id;
        
        log_action('RAISE', 'Employee ' || p_emp_id || ': ' || 
                   v_current_salary || ' → ' || v_new_salary);
        COMMIT;
    END give_raise;
    
    FUNCTION get_employee(p_emp_id IN NUMBER) RETURN emp_record_type AS
        v_emp emp_record_type;
    BEGIN
        SELECT e.employee_id, 
               e.first_name || ' ' || e.last_name,
               e.salary,
               d.department_name
        INTO v_emp
        FROM employees e
        LEFT JOIN departments d ON e.department_id = d.department_id
        WHERE e.employee_id = p_emp_id;
        
        RETURN v_emp;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE_APPLICATION_ERROR(-20014, 'Employee ' || p_emp_id || ' not found');
    END get_employee;
    
    FUNCTION get_dept_employees(p_dept_id IN NUMBER) RETURN emp_table_type PIPELINED AS
    BEGIN
        FOR rec IN (
            SELECT e.employee_id, 
                   e.first_name || ' ' || e.last_name AS emp_name,
                   e.salary,
                   d.department_name
            FROM employees e
            LEFT JOIN departments d ON e.department_id = d.department_id
            WHERE e.department_id = p_dept_id
            ORDER BY e.salary DESC
        ) LOOP
            PIPE ROW (emp_record_type(rec.employee_id, rec.emp_name, 
                                       rec.salary, rec.department_name));
        END LOOP;
        RETURN;
    END get_dept_employees;
    
    FUNCTION get_employee_count RETURN NUMBER AS
        v_count NUMBER;
    BEGIN
        SELECT COUNT(*) INTO v_count FROM employees;
        RETURN v_count;
    END get_employee_count;

END emp_pkg;
/
```

### 8.3 Using the Package

```sql
-- Call package procedures/functions using DOT notation
DECLARE
    v_new_id NUMBER;
    v_emp    emp_pkg.emp_record_type;
BEGIN
    -- Hire a new employee
    emp_pkg.hire_employee(
        p_first_name => 'Ritesh',
        p_last_name  => 'Singh',
        p_email      => 'RSINGH',
        p_salary     => 75000,
        p_emp_id     => v_new_id
    );
    DBMS_OUTPUT.PUT_LINE('New Employee ID: ' || v_new_id);
    
    -- Get employee details
    v_emp := emp_pkg.get_employee(v_new_id);
    DBMS_OUTPUT.PUT_LINE('Name: ' || v_emp.emp_name);
    DBMS_OUTPUT.PUT_LINE('Salary: ' || v_emp.salary);
    
    -- Give a raise
    emp_pkg.give_raise(v_new_id, 15);
    
    -- Use constant from package
    DBMS_OUTPUT.PUT_LINE('Max Salary: ' || emp_pkg.c_max_salary);
    
    -- Total employees
    DBMS_OUTPUT.PUT_LINE('Total Employees: ' || emp_pkg.get_employee_count);
END;
/

-- Use pipelined function in SQL (like a table!)
SELECT * FROM TABLE(emp_pkg.get_dept_employees(50));
```

### Why Packages Are Superior

```
┌──────────────────────────────────────────────────────────────┐
│  PACKAGES vs STANDALONE PROCEDURES/FUNCTIONS                  │
│                                                               │
│  ✅ ENCAPSULATION  — Hide private logic, expose public API   │
│  ✅ INFORMATION HIDING — Private procs/vars not visible      │
│  ✅ PERSISTENCE    — Package variables persist for session    │
│  ✅ PERFORMANCE    — Entire package loaded into memory once   │
│  ✅ OVERLOADING    — Same name, different parameters          │
│  ✅ ORGANIZATION   — Group related code logically             │
│  ✅ DEPENDENCY     — Less invalidation cascading              │
│  ✅ INITIALIZATION — Package body runs init code at first use │
│                                                               │
│  Rule of thumb: If you have 3+ related procedures,           │
│  PUT THEM IN A PACKAGE.                                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 9. Collections — Oracle's Arrays & Lists

PL/SQL has three types of collections:

```
┌──────────────────────────────────────────────────────────────┐
│                    PL/SQL COLLECTIONS                          │
│                                                               │
│  1. ASSOCIATIVE ARRAY (Index-By Table)                       │
│     • Key-value pairs                                         │
│     • Index: PLS_INTEGER or VARCHAR2                         │
│     • No constructor needed                                   │
│     • PL/SQL only (can't store in DB)                        │
│                                                               │
│  2. NESTED TABLE                                              │
│     • Like a dynamic array                                    │
│     • Index: integer starting at 1                           │
│     • Can be stored in database                               │
│     • Can have gaps (after DELETE)                            │
│                                                               │
│  3. VARRAY (Variable-Size Array)                              │
│     • Fixed maximum size                                      │
│     • Dense (no gaps)                                         │
│     • Can be stored in database                               │
│     • Order is preserved                                      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 9.1 Associative Arrays (Most Common in PL/SQL)

```sql
DECLARE
    -- String-indexed associative array (like a dictionary/map)
    TYPE salary_map_type IS TABLE OF NUMBER INDEX BY VARCHAR2(50);
    salary_map salary_map_type;
    
    -- Integer-indexed associative array
    TYPE name_list_type IS TABLE OF VARCHAR2(100) INDEX BY PLS_INTEGER;
    name_list name_list_type;
    
    v_key VARCHAR2(50);
BEGIN
    -- Populate
    salary_map('Ritesh') := 75000;
    salary_map('Priya')  := 82000;
    salary_map('John')   := 65000;
    
    -- Access
    DBMS_OUTPUT.PUT_LINE('Ritesh salary: ' || salary_map('Ritesh'));
    
    -- Check existence
    IF salary_map.EXISTS('Priya') THEN
        DBMS_OUTPUT.PUT_LINE('Priya found!');
    END IF;
    
    -- Iterate (associative arrays are sparse — use FIRST/NEXT)
    v_key := salary_map.FIRST;
    WHILE v_key IS NOT NULL LOOP
        DBMS_OUTPUT.PUT_LINE(v_key || ': ' || salary_map(v_key));
        v_key := salary_map.NEXT(v_key);
    END LOOP;
    
    -- Count
    DBMS_OUTPUT.PUT_LINE('Total entries: ' || salary_map.COUNT);
    
    -- Delete
    salary_map.DELETE('John');
    
    -- Integer-indexed
    FOR i IN 1..5 LOOP
        name_list(i) := 'Employee_' || i;
    END LOOP;
    
    FOR i IN name_list.FIRST..name_list.LAST LOOP
        DBMS_OUTPUT.PUT_LINE(name_list(i));
    END LOOP;
END;
/
```

### 9.2 Nested Tables

```sql
DECLARE
    -- Define type
    TYPE number_list_type IS TABLE OF NUMBER;
    
    -- Must initialize with constructor
    numbers number_list_type := number_list_type(10, 20, 30, 40, 50);
BEGIN
    -- Access
    DBMS_OUTPUT.PUT_LINE('First: ' || numbers(1));   -- 10
    DBMS_OUTPUT.PUT_LINE('Count: ' || numbers.COUNT); -- 5
    
    -- Extend and add
    numbers.EXTEND;           -- Add 1 empty slot
    numbers(6) := 60;
    
    numbers.EXTEND(3);        -- Add 3 empty slots
    numbers(7) := 70;
    numbers(8) := 80;
    numbers(9) := 90;
    
    -- Delete (creates gap)
    numbers.DELETE(3);         -- Remove element at index 3
    
    -- Iterate (handle gaps!)
    FOR i IN 1..numbers.COUNT + 1 LOOP
        IF numbers.EXISTS(i) THEN
            DBMS_OUTPUT.PUT_LINE('Index ' || i || ': ' || numbers(i));
        END IF;
    END LOOP;
    
    -- Collection methods
    -- .COUNT, .FIRST, .LAST, .NEXT(n), .PRIOR(n)
    -- .EXISTS(n), .EXTEND, .EXTEND(n), .TRIM, .TRIM(n), .DELETE
END;
/
```

### 9.3 VARRAYs

```sql
DECLARE
    TYPE color_array IS VARRAY(5) OF VARCHAR2(20);  -- Max 5 elements
    colors color_array := color_array('Red', 'Green', 'Blue');
BEGIN
    DBMS_OUTPUT.PUT_LINE('Colors: ' || colors.COUNT);  -- 3
    DBMS_OUTPUT.PUT_LINE('Max: ' || colors.LIMIT);      -- 5
    
    -- Extend (within limit)
    colors.EXTEND;
    colors(4) := 'Yellow';
    
    -- Cannot DELETE individual elements (stays dense)
    -- Can only TRIM from end
    colors.TRIM;  -- Removes last element
    
    FOR i IN 1..colors.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(colors(i));
    END LOOP;
END;
/
```

---

## 10. BULK COLLECT & FORALL — 10-100x Performance Boost 🚀

This is where PL/SQL goes from "normal" to **blazing fast**. Context switching between SQL and PL/SQL engines is expensive. Bulk operations minimize this.

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  ROW-BY-ROW Processing (SLOW):                               │
│                                                               │
│  PL/SQL  ─→  SQL Engine  ─→  PL/SQL  ─→  SQL Engine  ─→... │
│  Engine      (1 row)         Engine      (1 row)              │
│                                                               │
│  Context Switch × 10,000 rows = 10,000 switches = SLOW!      │
│                                                               │
│  ──────────────────────────────────────────────────────       │
│                                                               │
│  BULK Processing (FAST):                                      │
│                                                               │
│  PL/SQL  ──────→  SQL Engine  ──────→  PL/SQL                │
│  Engine           (ALL rows at once)      Engine              │
│                                                               │
│  Context Switch × 1 = 1 switch = BLAZING FAST! 🚀            │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 10.1 BULK COLLECT — Fetch All Rows at Once

```sql
DECLARE
    TYPE emp_id_list    IS TABLE OF employees.employee_id%TYPE;
    TYPE emp_name_list  IS TABLE OF VARCHAR2(100);
    TYPE emp_sal_list   IS TABLE OF employees.salary%TYPE;
    
    v_ids     emp_id_list;
    v_names   emp_name_list;
    v_salaries emp_sal_list;
BEGIN
    -- ❌ SLOW: Row-by-row with cursor
    -- FOR rec IN (SELECT * FROM employees) LOOP
    --     process(rec);  -- Context switch for EACH row
    -- END LOOP;
    
    -- ✅ FAST: BULK COLLECT all at once
    SELECT employee_id, 
           first_name || ' ' || last_name,
           salary
    BULK COLLECT INTO v_ids, v_names, v_salaries
    FROM employees
    WHERE department_id = 50;
    
    DBMS_OUTPUT.PUT_LINE('Fetched ' || v_ids.COUNT || ' employees');
    
    FOR i IN 1..v_ids.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(v_ids(i) || ': ' || v_names(i) || ' → ' || v_salaries(i));
    END LOOP;
END;
/

-- With LIMIT (for very large result sets — avoid memory issues)
DECLARE
    CURSOR c_emp IS SELECT * FROM employees;
    TYPE emp_table IS TABLE OF employees%ROWTYPE;
    v_emps emp_table;
BEGIN
    OPEN c_emp;
    LOOP
        FETCH c_emp BULK COLLECT INTO v_emps LIMIT 1000;  -- 1000 rows at a time
        
        -- Process batch
        FOR i IN 1..v_emps.COUNT LOOP
            -- Process each employee
            NULL;
        END LOOP;
        
        EXIT WHEN v_emps.COUNT = 0;  -- No more rows
    END LOOP;
    CLOSE c_emp;
END;
/
```

### 10.2 FORALL — Bulk DML Operations

```sql
DECLARE
    TYPE id_list IS TABLE OF NUMBER;
    TYPE sal_list IS TABLE OF NUMBER;
    
    v_ids      id_list;
    v_salaries sal_list;
BEGIN
    -- Get employees to update
    SELECT employee_id, ROUND(salary * 1.10, 2)
    BULK COLLECT INTO v_ids, v_salaries
    FROM employees
    WHERE department_id = 50;
    
    -- ❌ SLOW: Loop with individual updates
    -- FOR i IN 1..v_ids.COUNT LOOP
    --     UPDATE employees SET salary = v_salaries(i)
    --     WHERE employee_id = v_ids(i);  -- Context switch EACH iteration
    -- END LOOP;
    
    -- ✅ FAST: FORALL (sends ALL updates in one batch)
    FORALL i IN 1..v_ids.COUNT
        UPDATE employees 
        SET salary = v_salaries(i)
        WHERE employee_id = v_ids(i);
    
    DBMS_OUTPUT.PUT_LINE('Updated ' || SQL%ROWCOUNT || ' rows');
    
    COMMIT;
END;
/

-- FORALL with SAVE EXCEPTIONS (continue despite errors)
DECLARE
    TYPE id_list IS TABLE OF NUMBER;
    v_ids id_list := id_list(100, 999, 101, 888, 102);  -- 999, 888 don't exist
    
    e_bulk_errors EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_bulk_errors, -24381);
BEGIN
    FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
        UPDATE employees SET salary = salary + 1000
        WHERE employee_id = v_ids(i);
    
EXCEPTION
    WHEN e_bulk_errors THEN
        DBMS_OUTPUT.PUT_LINE('Errors: ' || SQL%BULK_EXCEPTIONS.COUNT);
        FOR i IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
            DBMS_OUTPUT.PUT_LINE(
                'Index: ' || SQL%BULK_EXCEPTIONS(i).ERROR_INDEX ||
                ' Error: ' || SQLERRM(-SQL%BULK_EXCEPTIONS(i).ERROR_CODE)
            );
        END LOOP;
END;
/
```

### Performance Comparison

```
┌──────────────────────────────────────────────────────────────┐
│  BENCHMARK: Processing 100,000 rows                          │
│                                                               │
│  Method                        │ Time        │ Improvement   │
│  ─────────────────────────────┼────────────┼──────────────  │
│  Row-by-row (cursor FOR loop) │ 50 seconds  │ Baseline      │
│  BULK COLLECT + FORALL        │ 2 seconds   │ 25x faster!   │
│  BULK COLLECT LIMIT 1000      │ 2.5 seconds │ 20x faster    │
│  Pure SQL (single UPDATE)     │ 0.5 seconds │ 100x faster   │
│                                                               │
│  LESSON: Use pure SQL when possible.                         │
│          When you MUST use PL/SQL, use BULK operations.       │
│          NEVER do row-by-row processing on large datasets.    │
└──────────────────────────────────────────────────────────────┘
```

> 🔥 **Golden Rule**: If you can do it in a single SQL statement, do it in SQL. If you MUST use PL/SQL (complex logic), use BULK COLLECT + FORALL. NEVER use row-by-row cursor loops for large data.

---

## 11. Triggers — Automatic Code Execution

Triggers are PL/SQL blocks that **automatically execute** when a specific event occurs on a table.

```sql
-- ═══════════════════════════════════════════════════
-- ROW-LEVEL TRIGGER (fires for EACH affected row)
-- ═══════════════════════════════════════════════════
CREATE OR REPLACE TRIGGER trg_employee_audit
BEFORE UPDATE OF salary ON employees      -- When salary column is updated
FOR EACH ROW                               -- For each row being updated
WHEN (NEW.salary != OLD.salary)            -- Only when salary actually changes
DECLARE
    v_change_pct NUMBER;
BEGIN
    v_change_pct := ROUND((:NEW.salary - :OLD.salary) / :OLD.salary * 100, 2);
    
    INSERT INTO salary_audit_log (
        employee_id, old_salary, new_salary, change_pct, 
        changed_by, changed_at
    ) VALUES (
        :OLD.employee_id, :OLD.salary, :NEW.salary, v_change_pct,
        USER, SYSDATE
    );
    
    -- Prevent salary decrease > 20%
    IF :NEW.salary < :OLD.salary * 0.80 THEN
        RAISE_APPLICATION_ERROR(-20050,
            'Cannot decrease salary by more than 20%. ' ||
            'Attempted: ' || :OLD.salary || ' → ' || :NEW.salary);
    END IF;
END;
/

-- ═══════════════════════════════════════════════════
-- STATEMENT-LEVEL TRIGGER (fires ONCE per statement)
-- ═══════════════════════════════════════════════════
CREATE OR REPLACE TRIGGER trg_no_weekend_changes
BEFORE INSERT OR UPDATE OR DELETE ON employees
-- No FOR EACH ROW = statement level
BEGIN
    IF TO_CHAR(SYSDATE, 'DY') IN ('SAT', 'SUN') THEN
        RAISE_APPLICATION_ERROR(-20051,
            'Cannot modify employee data on weekends!');
    END IF;
    
    IF TO_NUMBER(TO_CHAR(SYSDATE, 'HH24')) NOT BETWEEN 8 AND 18 THEN
        RAISE_APPLICATION_ERROR(-20052,
            'Data modifications only allowed between 8 AM and 6 PM');
    END IF;
END;
/

-- ═══════════════════════════════════════════════════
-- INSTEAD OF TRIGGER (on views — makes views updatable!)
-- ═══════════════════════════════════════════════════
CREATE OR REPLACE VIEW emp_dept_view AS
SELECT e.employee_id, e.first_name, e.last_name, 
       e.salary, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- This view joins 2 tables — can't update directly
-- INSTEAD OF trigger makes it possible:

CREATE OR REPLACE TRIGGER trg_emp_dept_insert
INSTEAD OF INSERT ON emp_dept_view
FOR EACH ROW
DECLARE
    v_dept_id departments.department_id%TYPE;
BEGIN
    -- Find or create department
    SELECT department_id INTO v_dept_id
    FROM departments WHERE department_name = :NEW.department_name;
    
    -- Insert into base table
    INSERT INTO employees (employee_id, first_name, last_name, 
                          salary, department_id)
    VALUES (:NEW.employee_id, :NEW.first_name, :NEW.last_name,
            :NEW.salary, v_dept_id);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20053, 
            'Department "' || :NEW.department_name || '" not found');
END;
/

-- ═══════════════════════════════════════════════════
-- COMPOUND TRIGGER (11g+) — Combines all timing points
-- ═══════════════════════════════════════════════════
CREATE OR REPLACE TRIGGER trg_employee_compound
FOR INSERT OR UPDATE ON employees
COMPOUND TRIGGER

    -- Shared variable across all timing points
    TYPE emp_id_list IS TABLE OF NUMBER;
    v_affected_ids emp_id_list := emp_id_list();

    BEFORE STATEMENT IS
    BEGIN
        DBMS_OUTPUT.PUT_LINE('Starting batch operation...');
    END BEFORE STATEMENT;

    BEFORE EACH ROW IS
    BEGIN
        :NEW.last_name := UPPER(:NEW.last_name);  -- Normalize
    END BEFORE EACH ROW;

    AFTER EACH ROW IS
    BEGIN
        v_affected_ids.EXTEND;
        v_affected_ids(v_affected_ids.COUNT) := :NEW.employee_id;
    END AFTER EACH ROW;

    AFTER STATEMENT IS
    BEGIN
        DBMS_OUTPUT.PUT_LINE('Affected ' || v_affected_ids.COUNT || ' employees');
        -- Can do batch processing here (avoids mutating table error!)
    END AFTER STATEMENT;

END trg_employee_compound;
/
```

### Trigger Timing Summary

| Timing | Level | When It Fires | :OLD / :NEW |
|--------|-------|---------------|-------------|
| `BEFORE` + `FOR EACH ROW` | Row | Before each row changes | Both available. Can modify :NEW |
| `AFTER` + `FOR EACH ROW` | Row | After each row changes | Both available. :NEW is read-only |
| `BEFORE` (no FOR EACH ROW) | Statement | Once, before the SQL starts | Not available |
| `AFTER` (no FOR EACH ROW) | Statement | Once, after the SQL completes | Not available |
| `INSTEAD OF` | Row (views only) | Replaces the DML on the view | Both available |
| `COMPOUND` | All | Combines all timing points | Depends on section |

### ⚠️ Trigger Anti-Patterns (Avoid These!)

```
┌──────────────────────────────────────────────────────────────┐
│  TRIGGER ANTI-PATTERNS — DON'T DO THESE!                     │
│                                                               │
│  ❌ Business logic in triggers                               │
│     → Put it in packages/procedures instead                  │
│     → Triggers are hard to debug and maintain                │
│                                                               │
│  ❌ Complex queries in row-level triggers                    │
│     → Fires for EVERY row = performance disaster             │
│     → 1000 row UPDATE = trigger fires 1000 times!            │
│                                                               │
│  ❌ COMMIT/ROLLBACK inside triggers                          │
│     → Not allowed! (unless AUTONOMOUS_TRANSACTION)           │
│     → Use autonomous only for logging, not business logic    │
│                                                               │
│  ❌ Cascading triggers (trigger A fires trigger B fires C)   │
│     → Nightmare to debug                                      │
│     → Max 32 levels before ORA-00036                         │
│                                                               │
│  ❌ Reading the table being modified (mutating table!)       │
│     → ORA-04091: table is mutating                           │
│     → Use COMPOUND TRIGGER to work around this               │
│                                                               │
│  ✅ GOOD uses for triggers:                                   │
│     → Audit logging                                           │
│     → Data validation (simple rules)                         │
│     → Auto-populating columns (created_at, updated_by)       │
│     → Enforcing security rules                               │
│     → Maintaining denormalized summary tables                │
└──────────────────────────────────────────────────────────────┘
```

---

## 12. Dynamic SQL — Building SQL at Runtime

```sql
-- ═══════════════════════════════════════════════════
-- EXECUTE IMMEDIATE — Run dynamic SQL
-- ═══════════════════════════════════════════════════

DECLARE
    v_table_name VARCHAR2(30) := 'EMPLOYEES';
    v_sql        VARCHAR2(1000);
    v_count      NUMBER;
    v_name       VARCHAR2(50);
BEGIN
    -- Simple dynamic SQL
    v_sql := 'SELECT COUNT(*) FROM ' || DBMS_ASSERT.SQL_OBJECT_NAME(v_table_name);
    EXECUTE IMMEDIATE v_sql INTO v_count;
    DBMS_OUTPUT.PUT_LINE('Count: ' || v_count);
    
    -- ✅ With bind variables (ALWAYS use for user input!)
    v_sql := 'SELECT first_name FROM employees WHERE employee_id = :id';
    EXECUTE IMMEDIATE v_sql INTO v_name USING 100;
    DBMS_OUTPUT.PUT_LINE('Name: ' || v_name);
    
    -- Dynamic DDL
    EXECUTE IMMEDIATE 'CREATE TABLE temp_test (id NUMBER, name VARCHAR2(50))';
    EXECUTE IMMEDIATE 'DROP TABLE temp_test';
    
    -- ❌ NEVER DO THIS (SQL Injection vulnerable!):
    -- v_sql := 'SELECT * FROM employees WHERE name = ''' || user_input || '''';
    
    -- ✅ ALWAYS use bind variables for user input:
    -- v_sql := 'SELECT * FROM employees WHERE name = :name';
    -- EXECUTE IMMEDIATE v_sql INTO v_result USING user_input;
END;
/
```

> ⚠️ **Security**: ALWAYS use bind variables (`:param` with `USING`) for user-provided values. NEVER concatenate user input into SQL strings. This prevents **SQL Injection** attacks.

---

## 13. Autonomous Transactions — Independent Commits

```sql
-- Normal behavior: ROLLBACK undoes EVERYTHING in the transaction
-- Autonomous transaction: Has its OWN commit/rollback, independent of caller

CREATE OR REPLACE PROCEDURE log_error(
    p_error_msg IN VARCHAR2,
    p_proc_name IN VARCHAR2
) AS
    PRAGMA AUTONOMOUS_TRANSACTION;  -- This is the magic line
BEGIN
    INSERT INTO error_log (error_msg, proc_name, error_date, error_user)
    VALUES (p_error_msg, p_proc_name, SYSDATE, USER);
    COMMIT;  -- This COMMIT is independent! Doesn't affect calling transaction
END;
/

-- Usage:
BEGIN
    UPDATE employees SET salary = 999999 WHERE employee_id = 100;
    
    -- Something goes wrong...
    log_error('Test error', 'my_procedure');  -- This gets COMMITTED
    
    ROLLBACK;  -- This rolls back the UPDATE, but NOT the log entry!
    -- The error_log INSERT survives because it was autonomous
END;
/
```

---

## 🧠 Chapter Summary — PL/SQL Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│              PL/SQL — COMPLETE QUICK REFERENCE                 │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  BLOCK: DECLARE → BEGIN → EXCEPTION → END;                   │
│                                                               │
│  VARIABLES:                                                   │
│    v_name type := value;                                     │
│    v_col  table.column%TYPE;     ← Anchored (recommended!)  │
│    v_row  table%ROWTYPE;         ← Entire row               │
│                                                               │
│  CONTROL: IF/ELSIF/ELSE, CASE, LOOP, WHILE, FOR i IN 1..N   │
│                                                               │
│  CURSORS:                                                     │
│    Implicit: SELECT INTO (auto-created)                      │
│    Explicit: DECLARE → OPEN → FETCH → CLOSE                 │
│    FOR loop: FOR rec IN cursor LOOP (preferred!)             │
│    REF CURSOR: Dynamic cursor (SYS_REFCURSOR)               │
│                                                               │
│  EXCEPTIONS:                                                  │
│    Predefined: NO_DATA_FOUND, TOO_MANY_ROWS, etc.           │
│    Custom: RAISE_APPLICATION_ERROR(-20xxx, 'msg')            │
│    Always re-RAISE after logging!                             │
│                                                               │
│  STORED CODE:                                                 │
│    Procedure: Does actions, no return value                   │
│    Function: Returns value, usable in SQL                    │
│    Package: Groups related code (BEST PRACTICE!)             │
│    Trigger: Auto-executes on DML events                      │
│                                                               │
│  COLLECTIONS:                                                 │
│    Associative Array: Key-value, PL/SQL only                 │
│    Nested Table: Dynamic array, can store in DB              │
│    VARRAY: Fixed-size array, dense, can store in DB          │
│                                                               │
│  PERFORMANCE:                                                 │
│    BULK COLLECT: Fetch all rows at once (not one-by-one)     │
│    FORALL: Execute DML for all rows at once                  │
│    Use LIMIT clause for very large result sets               │
│                                                               │
│  GOLDEN RULES:                                                │
│    1. Use %TYPE and %ROWTYPE (not hardcoded types)           │
│    2. Use cursor FOR loops (auto open/fetch/close)           │
│    3. Use packages (not standalone procs/funcs)              │
│    4. Use BULK COLLECT + FORALL (not row-by-row)             │
│    5. Use bind variables in dynamic SQL (security!)          │
│    6. Always handle exceptions (don't let code crash)        │
│    7. Keep triggers simple (audit, validation only)          │
│    8. COMMIT in procedures, not in functions                  │
└──────────────────────────────────────────────────────────────┘
```

---

## ❓ Interview Questions You WILL Be Asked

| # | Question | Key Answer |
|---|----------|-----------|
| 1 | What is a PL/SQL block? | DECLARE (optional) → BEGIN (required) → EXCEPTION (optional) → END; Three types: anonymous, procedure, function. |
| 2 | Cursor FOR loop vs explicit cursor? | FOR loop auto-handles OPEN/FETCH/CLOSE. Use it 99% of the time. Explicit only when you need BULK COLLECT or specific control. |
| 3 | What is %TYPE and %ROWTYPE? | %TYPE anchors variable to column type. %ROWTYPE anchors to entire row. Both auto-adapt when column types change. |
| 4 | Procedure vs Function? | Procedure: does actions, no return. Function: returns value, can be used in SQL. |
| 5 | What is a Package? | Container for related procs, funcs, types, variables. Has spec (public API) and body (implementation). |
| 6 | BULK COLLECT vs regular cursor? | BULK COLLECT fetches all rows at once → 10-100x faster. Regular cursor = row-by-row = slow context switching. |
| 7 | What is FORALL? | Bulk DML — sends all operations to SQL engine at once instead of one-by-one. Use with collections. |
| 8 | What is an autonomous transaction? | Independent transaction (PRAGMA AUTONOMOUS_TRANSACTION). Its COMMIT/ROLLBACK doesn't affect the caller. Used for logging. |
| 9 | What is a mutating table error? | ORA-04091: Row trigger tries to read/modify the table it's triggered on. Fix: use compound trigger or package variables. |
| 10 | RAISE vs RAISE_APPLICATION_ERROR? | RAISE: re-raises current or named exception. RAISE_APPLICATION_ERROR: creates custom error with code -20000 to -20999. |
| 11 | What are the collection types? | Associative Array (key-value, PL/SQL only), Nested Table (dynamic, DB storable), VARRAY (fixed-size, dense). |
| 12 | How to prevent SQL injection in PL/SQL? | Use bind variables (:param with USING). NEVER concatenate user input. Use DBMS_ASSERT for identifiers. |

---

> **Next Chapter**: [2B.5 — Oracle Performance Tuning →](./05-Oracle-Performance.md)
