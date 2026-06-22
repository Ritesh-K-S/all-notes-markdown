# ⚡ Chapter 2B.5 — Oracle Performance Tuning

> **Level:** 🔴 Advanced | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~6-8 hours
> **Prerequisites:** Chapter 2B.1 (Oracle Architecture), Chapter 2B.4 (PL/SQL)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- **Diagnose** slow queries like a seasoned Oracle DBA
- Use **AWR, ASH, ADDM** — the holy trinity of Oracle diagnostics
- Read and **interpret Execution Plans** like reading a newspaper
- Apply **Optimizer Hints** to force Oracle's hand (and know when NOT to)
- Design **Partitioning** strategies for billion-row tables
- Harness **Parallel Execution** to chew through data at lightning speed
- Create **SQL Profiles & Baselines** to lock in good performance forever

---

## 🧠 The Big Question

> *"My query ran in 2 seconds yesterday. Today it takes 45 minutes. What happened?"*

Welcome to the **#1 reason** Oracle DBAs exist. Performance doesn't just degrade — it **collapses** when something shifts: statistics go stale, data volumes spike, an index gets dropped, or the optimizer picks a bad plan.

Oracle gives you an **entire arsenal** of diagnostic and tuning tools. Most DBAs use only 10% of them. After this chapter, you'll use **100%**.

```
╔══════════════════════════════════════════════════════════════════════╗
║              ORACLE PERFORMANCE TUNING STACK                        ║
║                                                                      ║
║   ┌──────────────┐                                                   ║
║   │   SQL LEVEL  │  ← Execution Plans, Hints, SQL Profiles          ║
║   ├──────────────┤                                                   ║
║   │  INSTANCE    │  ← AWR, ASH, ADDM, Wait Events                   ║
║   ├──────────────┤                                                   ║
║   │   STORAGE    │  ← Partitioning, Compression, IOT                 ║
║   ├──────────────┤                                                   ║
║   │  PARALLEL    │  ← Parallel Query, Parallel DML, PX              ║
║   ├──────────────┤                                                   ║
║   │   OS/HW      │  ← CPU, Memory, I/O, Network                     ║
║   └──────────────┘                                                   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 📊 Section 1: Execution Plans — Your X-Ray Machine

### What IS an Execution Plan?

Every SQL statement goes through Oracle's **Cost-Based Optimizer (CBO)**. The CBO evaluates **thousands** of possible ways to execute your query and picks the one with the **lowest estimated cost**.

The execution plan is the **blueprint** the optimizer chose.

```
Your SQL Query
     │
     ▼
╔═══════════════════╗
║  Parser           ║  ← Syntax check + semantic check
╠═══════════════════╣
║  Optimizer (CBO)  ║  ← Generate plans → Estimate costs → Pick cheapest
╠═══════════════════╣
║  Row Source        ║  ← Build the actual execution tree
║  Generator         ║
╠═══════════════════╣
║  Execution Engine  ║  ← Run it and return results
╚═══════════════════╝
```

### How to Get an Execution Plan

```sql
-- Method 1: EXPLAIN PLAN (Estimated — doesn't actually run the query)
EXPLAIN PLAN FOR
SELECT e.employee_name, d.department_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 50000;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Method 2: AUTOTRACE (Runs the query + shows plan + statistics)
SET AUTOTRACE ON
SELECT e.employee_name, d.department_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 50000;

-- Method 3: DBMS_XPLAN with ACTUAL statistics (GOLD STANDARD ⭐)
SELECT /*+ GATHER_PLAN_STATISTICS */ e.employee_name, d.department_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 50000;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

### Reading an Execution Plan — Step by Step

```
Plan hash value: 1234567890

------------------------------------------------------------------------------------------
| Id  | Operation                    | Name          | Rows  | Bytes | Cost  | Time     |
------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |               |   500 | 25000 |    45 | 00:00:01 |
|   1 |  HASH JOIN                   |               |   500 | 25000 |    45 | 00:00:01 |
|   2 |   TABLE ACCESS FULL          | DEPARTMENTS   |    50 |  1200 |     3 | 00:00:01 |
|*  3 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEES     |   500 | 13000 |    42 | 00:00:01 |
|*  4 |    INDEX RANGE SCAN          | EMP_SAL_IDX   |   500 |       |     2 | 00:00:01 |
------------------------------------------------------------------------------------------

Predicate Information:
   3 - filter("E"."SALARY">50000)
   4 - access("E"."SALARY">50000)
```

**How to read it:**

```
RULE 1: Read INNERMOST → OUTERMOST (deepest indentation first)
RULE 2: At the same level, read TOP → BOTTOM

Execution Order for the plan above:
   Step 4 → INDEX RANGE SCAN on EMP_SAL_IDX (find ROWIDs where salary > 50000)
   Step 3 → TABLE ACCESS BY INDEX ROWID (fetch full rows using those ROWIDs)
   Step 2 → TABLE ACCESS FULL on DEPARTMENTS (read ALL department rows)
   Step 1 → HASH JOIN (combine employees + departments)
   Step 0 → Return results
```

### Key Operations You Must Know

```
╔══════════════════════════════════════════════════════════════════════════╗
║ Operation                │ What it Does              │ Good or Bad?     ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ TABLE ACCESS FULL        │ Reads EVERY row in table  │ 🔴 Bad for small ║
║                          │                           │    result sets   ║
║                          │                           │ 🟢 Good for big  ║
║                          │                           │    scans (>10%)  ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ INDEX UNIQUE SCAN        │ Finds exactly 1 row       │ 🟢 BEST — O(1)  ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ INDEX RANGE SCAN         │ Finds a range of rows     │ 🟢 Great         ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ INDEX FULL SCAN          │ Reads entire index         │ 🟡 Depends       ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ INDEX FAST FULL SCAN     │ Multi-block read of index  │ 🟡 Better than   ║
║                          │ (no order guaranteed)      │   full table scan║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ NESTED LOOPS             │ For each row in A,         │ 🟢 Small datasets║
║                          │ look up matching row in B  │ 🔴 Huge datasets ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ HASH JOIN                │ Build hash table from      │ 🟢 Large equi-   ║
║                          │ smaller set, probe with    │    joins         ║
║                          │ larger set                 │                  ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ SORT MERGE JOIN          │ Sort both sets, merge      │ 🟡 Pre-sorted    ║
║                          │                           │    data          ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ SORT ORDER BY            │ Sorting results            │ 🟡 Expensive for ║
║                          │                           │    large sets    ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ HASH GROUP BY            │ Grouping via hash          │ 🟢 Typical       ║
╠══════════════════════════╪═══════════════════════════╪══════════════════╣
║ FILTER                   │ Post-filter rows           │ ⚠️ Check if it   ║
║                          │                           │   can be pushed  ║
╚══════════════════════════╧═══════════════════════════╧══════════════════╝
```

### The Critical Columns

```
Rows   → Estimated rows returned (compare with ACTUAL using ALLSTATS)
Bytes  → Estimated data volume
Cost   → Optimizer's relative cost estimate (lower = cheaper)
Time   → Estimated wall-clock time

⚠️ If Estimated Rows ≠ Actual Rows → STALE STATISTICS!
   This is THE #1 cause of bad execution plans.
```

### Fixing Stale Statistics

```sql
-- Check when statistics were last gathered
SELECT table_name, last_analyzed, num_rows, blocks, avg_row_len
FROM dba_tables
WHERE owner = 'HR' AND table_name = 'EMPLOYEES';

-- Gather fresh statistics (do this regularly!)
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname  => 'HR',
        tabname  => 'EMPLOYEES',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,  -- Let Oracle decide
        method_opt       => 'FOR ALL COLUMNS SIZE AUTO',  -- Histograms too
        cascade          => TRUE                           -- Include indexes
    );
END;
/

-- Gather stats for entire schema
BEGIN
    DBMS_STATS.GATHER_SCHEMA_STATS(
        ownname          => 'HR',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        options          => 'GATHER STALE'   -- Only regather stale ones
    );
END;
/

-- Check which tables have stale statistics
SELECT owner, table_name, object_type, stale_stats, last_analyzed
FROM dba_tab_statistics
WHERE stale_stats = 'YES' AND owner = 'HR';
```

---

## 🏥 Section 2: AWR — The Automatic Workload Repository

### What is AWR?

**AWR** is Oracle's **flight recorder** — it automatically captures database performance snapshots every **60 minutes** (configurable) and stores them for **8 days** (configurable).

Think of it as a **dashcam** for your database — when something goes wrong, you rewind the tape.

```
╔══════════════════════════════════════════════════════════════════╗
║                    AWR SNAPSHOT LIFECYCLE                        ║
║                                                                  ║
║   12:00 PM     1:00 PM     2:00 PM     3:00 PM                 ║
║      │            │            │            │                    ║
║      ▼            ▼            ▼            ▼                    ║
║   ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐                 ║
║   │ SNAP │    │ SNAP │    │ SNAP │    │ SNAP │                 ║
║   │ #100 │    │ #101 │    │ #102 │    │ #103 │                 ║
║   └──────┘    └──────┘    └──────┘    └──────┘                 ║
║                                                                  ║
║   AWR Report = Difference between any TWO snapshots             ║
║                                                                  ║
║   "Compare Snap #101 vs #103 → What changed in those 2 hours?" ║
╚══════════════════════════════════════════════════════════════════╝
```

### Creating and Reading AWR Reports

```sql
-- List available snapshots
SELECT snap_id, begin_interval_time, end_interval_time
FROM dba_hist_snapshot
ORDER BY snap_id DESC
FETCH FIRST 20 ROWS ONLY;

-- Generate AWR Report (HTML — best format)
-- Run this and follow the prompts:
@$ORACLE_HOME/rdbms/admin/awrrpt.sql

-- Or generate programmatically:
SELECT * FROM TABLE(
    DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_HTML(
        l_dbid     => (SELECT dbid FROM v$database),
        l_inst_num => 1,
        l_bid      => 100,    -- Begin snapshot ID
        l_eid      => 103     -- End snapshot ID
    )
);

-- Create a manual snapshot (before/after a change)
BEGIN
    DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
END;
/
```

### AWR Report — Key Sections to Focus On

```
╔══════════════════════════════════════════════════════════════════════╗
║  AWR REPORT READING CHEAT SHEET                                     ║
║                                                                      ║
║  1️⃣  Report Summary                                                 ║
║     → DB Time vs Elapsed Time                                        ║
║     → If DB Time >> Elapsed Time → heavy concurrency                 ║
║     → If DB Time << Elapsed Time → DB is mostly idle                 ║
║                                                                      ║
║  2️⃣  Top 10 Foreground Events (MOST IMPORTANT ⭐)                   ║
║     → Shows WHERE time is spent                                      ║
║     → "db file sequential read" = single-block I/O (index lookups)   ║
║     → "db file scattered read"  = multi-block I/O (full table scans) ║
║     → "log file sync"           = commit bottleneck                  ║
║     → "enq: TX - row lock"      = row-level locking contention       ║
║     → "latch: shared pool"      = hard parsing (use bind variables!) ║
║                                                                      ║
║  3️⃣  SQL Statistics                                                  ║
║     → SQL ordered by Elapsed Time  → longest-running SQLs            ║
║     → SQL ordered by CPU Time      → CPU-hungry SQLs                 ║
║     → SQL ordered by Gets          → I/O-hungry SQLs (buffer gets)   ║
║     → SQL ordered by Executions    → most frequently executed        ║
║                                                                      ║
║  4️⃣  Instance Activity Stats                                        ║
║     → Physical reads/writes per second                               ║
║     → Logical reads per second                                       ║
║     → Parse counts (hard vs soft)                                    ║
║                                                                      ║
║  5️⃣  Wait Event Histogram                                           ║
║     → Detailed breakdown of wait times                               ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Top Wait Events — The Rosetta Stone

```
╔═══════════════════════════════════════════════════════════════════════════╗
║  Wait Event                    │ What It Means          │ Fix            ║
╠════════════════════════════════╪════════════════════════╪════════════════╣
║ db file sequential read        │ Index block reads       │ Tune indexes,  ║
║                                │ (random I/O)           │ faster storage ║
╠════════════════════════════════╪════════════════════════╪════════════════╣
║ db file scattered read         │ Full table scan         │ Add indexes,   ║
║                                │ (multi-block I/O)      │ partition table║
╠════════════════════════════════╪════════════════════════╪════════════════╣
║ log file sync                  │ Commit taking too long  │ Faster redo    ║
║                                │                        │ disks, batch   ║
║                                │                        │ commits        ║
╠════════════════════════════════╪════════════════════════╪════════════════╣
║ log file parallel write        │ LGWR writing redo logs  │ Faster I/O     ║
╠════════════════════════════════╪════════════════════════╪════════════════╣
║ enq: TX - row lock contention  │ Sessions blocking each  │ Fix app logic, ║
║                                │ other on same rows     │ reduce txn time║
╠════════════════════════════════╪════════════════════════╪════════════════╣
║ latch: shared pool             │ Too much hard parsing   │ Use bind vars! ║
╠════════════════════════════════╪════════════════════════╪════════════════╣
║ cursor: pin S wait on X        │ High-concurrency SQL    │ Use literals   ║
║                                │ mutex contention       │ for hot SQL    ║
╠════════════════════════════════╪════════════════════════╪════════════════╣
║ direct path read               │ PQ slaves doing direct  │ Normal for     ║
║                                │ path reads             │ parallel scans ║
╠════════════════════════════════╪════════════════════════╪════════════════╣
║ free buffer waits              │ Buffer cache too small  │ Increase       ║
║                                │ or I/O too slow        │ DB_CACHE_SIZE  ║
╚════════════════════════════════╧════════════════════════╧════════════════╝
```

---

## 🔥 Section 3: ASH — Active Session History

### What is ASH?

If AWR is the **dashcam** (periodic snapshots), ASH is the **live security camera** — it samples **every active session, every second**.

```
╔══════════════════════════════════════════════════════════════════╗
║                         ASH SAMPLING                            ║
║                                                                  ║
║   Every 1 second, Oracle captures:                               ║
║   ┌──────────────────────────────────────────────────────┐      ║
║   │  Session ID  │  SQL ID  │  Wait Event  │  Module    │      ║
║   │  Blocking    │  Plan    │  Wait Class  │  Action    │      ║
║   │  Session     │  Hash    │  Wait Time   │  Client    │      ║
║   └──────────────────────────────────────────────────────┘      ║
║                                                                  ║
║   Stored in: V$ACTIVE_SESSION_HISTORY (in-memory, ~1 hour)      ║
║              DBA_HIST_ACTIVE_SESS_HISTORY (on-disk, AWR)        ║
╚══════════════════════════════════════════════════════════════════╝
```

### ASH Queries — Your Go-To Toolkit

```sql
-- 🔍 What's happening RIGHT NOW?
SELECT
    s.sid,
    s.serial#,
    s.username,
    s.sql_id,
    s.event,
    s.wait_class,
    s.seconds_in_wait,
    s.blocking_session,
    s.module
FROM v$session s
WHERE s.status = 'ACTIVE'
  AND s.type = 'USER'
ORDER BY s.seconds_in_wait DESC;

-- 🔍 Top SQL in the last 30 minutes (by time spent)
SELECT
    sql_id,
    COUNT(*) AS ash_samples,         -- Each sample ≈ 1 second of DB time
    ROUND(COUNT(*) / 60, 1) AS minutes_of_db_time,
    session_state,
    event
FROM v$active_session_history
WHERE sample_time > SYSDATE - INTERVAL '30' MINUTE
GROUP BY sql_id, session_state, event
ORDER BY ash_samples DESC
FETCH FIRST 10 ROWS ONLY;

-- 🔍 Who was blocking whom in the last hour?
SELECT
    blocking_session AS blocker_sid,
    session_id AS waiter_sid,
    sql_id,
    event,
    COUNT(*) AS seconds_blocked
FROM v$active_session_history
WHERE blocking_session IS NOT NULL
  AND sample_time > SYSDATE - INTERVAL '1' HOUR
GROUP BY blocking_session, session_id, sql_id, event
ORDER BY seconds_blocked DESC;

-- 🔍 Drill into a specific slow SQL
SELECT
    sql_id,
    sql_plan_hash_value,
    session_state,
    event,
    COUNT(*) AS samples
FROM v$active_session_history
WHERE sql_id = 'abc123xyz'
  AND sample_time > SYSDATE - INTERVAL '1' HOUR
GROUP BY sql_id, sql_plan_hash_value, session_state, event
ORDER BY samples DESC;

-- 🔍 Generate an ASH Report (like AWR but focused on active sessions)
@$ORACLE_HOME/rdbms/admin/ashrpt.sql
```

### ASH vs AWR — When to Use Which?

```
╔════════════════════════════════════════════════════════════════╗
║                    │  AWR                │  ASH               ║
╠════════════════════╪═════════════════════╪═════════════════════╣
║  Granularity       │  Hourly snapshots   │  1-second samples  ║
║  Storage           │  On disk (8 days)   │  In memory (~1hr)  ║
║                    │                     │  + disk (via AWR)  ║
║  Best for          │  "What happened     │  "What's happening ║
║                    │   last night?"      │   RIGHT NOW?"      ║
║  Scope             │  Entire instance    │  Active sessions   ║
║  Overhead          │  Very low           │  Very low          ║
║  Access            │  DBA_HIST_* views   │  V$ACTIVE_SESSION_ ║
║                    │                     │  HISTORY           ║
╚════════════════════╧═════════════════════╧═════════════════════╝

💡 Rule of Thumb:
   → Problem happening NOW → Use ASH (V$ACTIVE_SESSION_HISTORY)
   → Problem happened YESTERDAY → Use AWR (DBA_HIST_* views)
   → Need a big-picture report → AWR Report
   → Need second-by-second detail → ASH Report
```

---

## 🤖 Section 4: ADDM — Automatic Database Diagnostic Monitor

### What is ADDM?

ADDM is Oracle's **AI doctor** — it automatically analyzes AWR data and gives you **specific recommendations** in plain English.

Instead of YOU staring at 200 pages of AWR data, ADDM says:

> *"Your SQL ID abc123 is consuming 65% of DB time. It's doing a full table scan on ORDERS (500M rows). Creating an index on ORDER_DATE would reduce the cost by 95%."*

```
╔══════════════════════════════════════════════════════════════════╗
║                     ADDM WORKFLOW                                ║
║                                                                  ║
║   AWR Snapshots → ADDM Analysis → Findings + Recommendations   ║
║                                                                  ║
║   ┌──────────┐     ┌──────────┐     ┌──────────────────┐       ║
║   │  SNAP    │     │  ADDM    │     │  FINDING:        │       ║
║   │  #101    │────►│  Engine  │────►│  "SQL abc123 is  │       ║
║   │  #102    │     │          │     │   causing 65% of │       ║
║   └──────────┘     └──────────┘     │   DB time"       │       ║
║                                      │                  │       ║
║                                      │  RECOMMENDATION: │       ║
║                                      │  "Create index   │       ║
║                                      │   on ORDER_DATE" │       ║
║                                      └──────────────────┘       ║
╚══════════════════════════════════════════════════════════════════╝
```

### Running ADDM

```sql
-- ADDM runs automatically after every AWR snapshot
-- But you can also run it manually:

-- Method 1: Use the advisor framework
VARIABLE task_name VARCHAR2(100);
BEGIN
    :task_name := 'MY_ADDM_TASK';
    DBMS_ADVISOR.CREATE_TASK(
        advisor_name => 'ADDM',
        task_name    => :task_name
    );
    DBMS_ADVISOR.SET_TASK_PARAMETER(
        task_name => :task_name,
        parameter => 'START_SNAPSHOT',
        value     => 100
    );
    DBMS_ADVISOR.SET_TASK_PARAMETER(
        task_name => :task_name,
        parameter => 'END_SNAPSHOT',
        value     => 103
    );
    DBMS_ADVISOR.EXECUTE_TASK(:task_name);
END;
/

-- View the results
SELECT * FROM TABLE(DBMS_ADVISOR.GET_TASK_REPORT(:task_name));

-- Method 2: Quick ADDM report from command line
@$ORACLE_HOME/rdbms/admin/addmrpt.sql
```

### ADDM Finding Categories

```
╔══════════════════════════════════════════════════════════════════╗
║  Category             │ Example Finding                         ║
╠═══════════════════════╪═════════════════════════════════════════╣
║  Top SQL              │ "SQL_ID xyz uses 45% of DB time"        ║
║  I/O Issues           │ "Excessive full table scans detected"   ║
║  Memory (SGA/PGA)     │ "Buffer cache hit ratio below 90%"     ║
║  Parsing              │ "High hard parse rate: use bind vars"   ║
║  Concurrency          │ "Lock contention on ORDERS table"       ║
║  Undo/Redo            │ "Redo log switching too frequently"     ║
║  RAC Specific         │ "Global cache block transfers high"     ║
║  Configuration        │ "Undersized PGA_AGGREGATE_TARGET"       ║
╚═══════════════════════╧═════════════════════════════════════════╝
```

---

## 🎛️ Section 5: Optimizer Hints — Talking to the Optimizer

### What Are Hints?

Hints are **suggestions** you give to the Oracle Optimizer to influence its plan. When the optimizer makes a bad decision (wrong join method, wrong index, wrong access path), hints let you **override** it.

> ⚠️ **WARNING:** Hints are a **last resort**, not a first tool. Always fix the root cause (statistics, indexes, data model) before hinting.

### Hint Syntax

```sql
-- Hints go inside a special comment: /*+ HINT_NAME */
SELECT /*+ FULL(e) */ employee_name
FROM employees e
WHERE salary > 50000;

-- Multiple hints
SELECT /*+ LEADING(d e) USE_HASH(e) PARALLEL(e, 4) */
    e.employee_name, d.department_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

### Essential Hints — The Cheat Sheet

```
╔═══════════════════════════════════════════════════════════════════════════╗
║  Category    │ Hint                │ What It Does                       ║
╠══════════════╪═════════════════════╪════════════════════════════════════╣
║              │ FULL(table)         │ Force full table scan              ║
║  ACCESS      │ INDEX(table idx)    │ Force specific index               ║
║  PATH        │ NO_INDEX(table idx) │ Prevent specific index             ║
║              │ INDEX_FFS(table)    │ Force index fast full scan         ║
╠══════════════╪═════════════════════╪════════════════════════════════════╣
║              │ USE_NL(table)       │ Force nested loops join            ║
║  JOIN        │ USE_HASH(table)     │ Force hash join                    ║
║  METHOD      │ USE_MERGE(table)    │ Force sort-merge join              ║
║              │ NO_USE_HASH(table)  │ Prevent hash join                  ║
╠══════════════╪═════════════════════╪════════════════════════════════════╣
║  JOIN        │ LEADING(t1 t2 t3)  │ Force join order (t1 first,        ║
║  ORDER       │                     │ then t2, then t3)                  ║
║              │ ORDERED             │ Join tables in FROM clause order   ║
╠══════════════╪═════════════════════╪════════════════════════════════════╣
║              │ PARALLEL(table, N)  │ Use N parallel query slaves       ║
║  PARALLEL    │ NO_PARALLEL(table)  │ Disable parallelism               ║
║              │ PQ_DISTRIBUTE(...)  │ Control parallel data distribution ║
╠══════════════╪═════════════════════╪════════════════════════════════════╣
║              │ FIRST_ROWS(n)       │ Optimize to return first N rows   ║
║  OPTIMIZER   │ ALL_ROWS            │ Optimize for total throughput     ║
║  GOAL        │ RESULT_CACHE        │ Cache the result set              ║
║              │ NO_RESULT_CACHE     │ Don't cache the result set        ║
╠══════════════╪═════════════════════╪════════════════════════════════════╣
║              │ PUSH_PRED(view)     │ Push predicate into view          ║
║  QUERY       │ NO_MERGE(view)      │ Don't merge view into main query  ║
║  TRANSFORM   │ UNNEST              │ Unnest subquery into join         ║
║              │ NO_UNNEST           │ Keep subquery as-is               ║
╚══════════════╧═════════════════════╧════════════════════════════════════╝
```

### Real-World Hinting Scenarios

```sql
-- Scenario 1: Optimizer chose NESTED LOOPS but data is huge → Force HASH JOIN
SELECT /*+ USE_HASH(o c) */
    o.order_id, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > DATE '2026-01-01';

-- Scenario 2: Wrong join order → Force the right order
SELECT /*+ LEADING(r p o) USE_HASH(p) USE_NL(o) */
    r.region_name, p.product_name, o.quantity
FROM regions r
JOIN products p ON r.region_id = p.region_id
JOIN order_items o ON p.product_id = o.product_id;

-- Scenario 3: Full table scan on 1B rows, but only need first 10
SELECT /*+ FIRST_ROWS(10) */
    employee_name, hire_date
FROM employees
WHERE department_id = 50
ORDER BY hire_date DESC;

-- Scenario 4: Optimizer ignores a good index
SELECT /*+ INDEX(e EMP_DEPT_SAL_IDX) */
    employee_name, salary
FROM employees e
WHERE department_id = 10 AND salary > 50000;
```

### When Hints DON'T Work

```
⚠️ Common Reasons Hints Are Silently Ignored:
   1. Table alias mismatch  → FULL(employees) when alias is "e"
                               Fix: FULL(e)
   2. Wrong hint syntax     → SELECT /* FULL(e) */  (missing +)
                               Fix: SELECT /*+ FULL(e) */
   3. Index doesn't exist   → INDEX(e NONEXISTENT_IDX)
   4. Conflicting hints     → FULL(e) INDEX(e IDX1)
   5. Hint is impossible    → USE_HASH on a non-equijoin

💡 Oracle NEVER errors on a bad hint — it silently ignores it!
   Always verify with EXPLAIN PLAN after adding hints.
```

---

## 🧱 Section 6: Partitioning — Divide and Conquer

### Why Partition?

When a table grows beyond **10 million+ rows**, queries slow down even with good indexes. Partitioning **physically divides** the table into smaller, manageable pieces while keeping it logically as one table.

```
╔══════════════════════════════════════════════════════════════════╗
║          WITHOUT PARTITIONING          WITH PARTITIONING         ║
║                                                                  ║
║   ┌────────────────────┐      ┌──────────┐ ┌──────────┐        ║
║   │                    │      │ Q1_2026  │ │ Q2_2026  │        ║
║   │    ORDERS TABLE    │      │  5M rows │ │  5M rows │        ║
║   │    500M rows       │      └──────────┘ └──────────┘        ║
║   │                    │      ┌──────────┐ ┌──────────┐        ║
║   │  Query scans ALL   │      │ Q3_2026  │ │ Q4_2026  │        ║
║   │  500M rows 😱      │      │  5M rows │ │  5M rows │        ║
║   └────────────────────┘      └──────────┘ └──────────┘        ║
║                                                                  ║
║                               Query for Q1 scans ONLY 5M rows  ║
║                               → 100x faster! 🚀                ║
║                               This is called PARTITION PRUNING  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Types of Partitioning

#### 1. Range Partitioning (Most Common ⭐)

```sql
-- Partition by date range — PERFECT for time-series data
CREATE TABLE orders (
    order_id    NUMBER,
    order_date  DATE,
    customer_id NUMBER,
    amount      NUMBER(10,2)
)
PARTITION BY RANGE (order_date) (
    PARTITION p_2024_q1 VALUES LESS THAN (DATE '2024-04-01'),
    PARTITION p_2024_q2 VALUES LESS THAN (DATE '2024-07-01'),
    PARTITION p_2024_q3 VALUES LESS THAN (DATE '2024-10-01'),
    PARTITION p_2024_q4 VALUES LESS THAN (DATE '2025-01-01'),
    PARTITION p_2025_q1 VALUES LESS THAN (DATE '2025-04-01'),
    PARTITION p_2025_q2 VALUES LESS THAN (DATE '2025-07-01'),
    PARTITION p_future  VALUES LESS THAN (MAXVALUE)       -- Catch-all
);

-- Interval Partitioning — Auto-creates partitions! (Oracle 11g+)
CREATE TABLE sales (
    sale_id    NUMBER,
    sale_date  DATE,
    amount     NUMBER(10,2)
)
PARTITION BY RANGE (sale_date)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))   -- Auto-create monthly partitions
(
    PARTITION p_initial VALUES LESS THAN (DATE '2024-01-01')
);
-- Oracle automatically creates p_SYS_xxxx for each new month!
```

#### 2. List Partitioning

```sql
-- Partition by discrete values — regions, statuses, categories
CREATE TABLE customers (
    customer_id   NUMBER,
    customer_name VARCHAR2(100),
    region        VARCHAR2(20),
    status        VARCHAR2(10)
)
PARTITION BY LIST (region) (
    PARTITION p_north  VALUES ('Delhi', 'Punjab', 'UP', 'Haryana'),
    PARTITION p_south  VALUES ('Karnataka', 'TN', 'Kerala', 'AP'),
    PARTITION p_east   VALUES ('WB', 'Odisha', 'Bihar', 'Jharkhand'),
    PARTITION p_west   VALUES ('Maharashtra', 'Gujarat', 'Rajasthan', 'Goa'),
    PARTITION p_other  VALUES (DEFAULT)
);
```

#### 3. Hash Partitioning

```sql
-- Even data distribution — when no natural range/list key exists
CREATE TABLE transactions (
    txn_id      NUMBER,
    account_id  NUMBER,
    amount      NUMBER(10,2),
    txn_date    DATE
)
PARTITION BY HASH (account_id)
PARTITIONS 16;    -- Always use power of 2 for even distribution
```

#### 4. Composite Partitioning (Range-Hash, Range-List, etc.)

```sql
-- Range-Hash: Partition by date (range), sub-partition by customer (hash)
CREATE TABLE order_details (
    order_id    NUMBER,
    order_date  DATE,
    customer_id NUMBER,
    product_id  NUMBER,
    quantity    NUMBER
)
PARTITION BY RANGE (order_date)
SUBPARTITION BY HASH (customer_id) SUBPARTITIONS 8
(
    PARTITION p_2025_q1 VALUES LESS THAN (DATE '2025-04-01'),
    PARTITION p_2025_q2 VALUES LESS THAN (DATE '2025-07-01'),
    PARTITION p_2025_q3 VALUES LESS THAN (DATE '2025-10-01'),
    PARTITION p_2025_q4 VALUES LESS THAN (DATE '2026-01-01'),
    PARTITION p_future  VALUES LESS THAN (MAXVALUE)
);
```

### Partition Pruning — The Whole Point

```sql
-- This query ONLY scans partition p_2025_q1 (millions of rows skipped!)
SELECT * FROM orders
WHERE order_date BETWEEN DATE '2025-01-01' AND DATE '2025-03-31';

-- Verify pruning in the execution plan:
-- Look for: PARTITION RANGE SINGLE or PARTITION RANGE ITERATOR
-- BAD:      PARTITION RANGE ALL  (no pruning — scanning everything!)

EXPLAIN PLAN FOR
SELECT * FROM orders WHERE order_date = DATE '2025-06-15';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Plan will show:
-- PARTITION RANGE SINGLE → Oracle knows exactly which partition to scan ✅
```

### Partition Maintenance Operations

```sql
-- Add a new partition
ALTER TABLE orders ADD PARTITION p_2026_q1
    VALUES LESS THAN (DATE '2026-04-01');

-- Drop old data instantly (instead of DELETE which is slow)
ALTER TABLE orders DROP PARTITION p_2024_q1;
-- Drops MILLIONS of rows in milliseconds! 🚀

-- Split a partition
ALTER TABLE orders SPLIT PARTITION p_future AT (DATE '2026-07-01')
    INTO (PARTITION p_2026_q2, PARTITION p_future);

-- Merge two partitions
ALTER TABLE orders MERGE PARTITIONS p_2024_q3, p_2024_q4
    INTO PARTITION p_2024_h2;

-- Exchange partition (swap data with a staging table — instant!)
ALTER TABLE orders EXCHANGE PARTITION p_2025_q1
    WITH TABLE orders_staging_q1
    INCLUDING INDEXES;

-- Truncate a single partition (faster than DELETE)
ALTER TABLE orders TRUNCATE PARTITION p_2024_q1;
```

### Local vs Global Indexes on Partitioned Tables

```
╔════════════════════════════════════════════════════════════════════╗
║              LOCAL INDEX                GLOBAL INDEX              ║
║                                                                    ║
║   ┌──────────┐ ┌──────────┐     ┌──────────────────────┐        ║
║   │ Index P1 │ │ Index P2 │     │   Single Index       │        ║
║   │  for     │ │  for     │     │   spans ALL          │        ║
║   │  Part 1  │ │  Part 2  │     │   partitions         │        ║
║   └──────────┘ └──────────┘     └──────────────────────┘        ║
║   ┌──────────┐ ┌──────────┐                                      ║
║   │ Index P3 │ │ Index P4 │     Pros: Fast cross-partition       ║
║   │  for     │ │  for     │           queries                    ║
║   │  Part 3  │ │  Part 4  │     Cons: DDL on partition           ║
║   └──────────┘ └──────────┘           invalidates index!         ║
║                                                                    ║
║   Pros: Partition DDL is fast   💡 Use UPDATE INDEXES clause     ║
║         (index stays valid)        to keep global indexes valid   ║
║   Cons: Cross-partition queries                                   ║
║         must scan multiple indexes                                ║
║                                                                    ║
║   ⭐ RULE: Use LOCAL indexes by default on partitioned tables     ║
╚════════════════════════════════════════════════════════════════════╝
```

```sql
-- Local index (one index segment per partition)
CREATE INDEX idx_orders_cust ON orders(customer_id) LOCAL;

-- Global index (spans all partitions)
CREATE INDEX idx_orders_status ON orders(status) GLOBAL;

-- Keep global indexes valid during partition operations
ALTER TABLE orders DROP PARTITION p_2024_q1 UPDATE INDEXES;
```

---

## 🚄 Section 7: Parallel Execution — Unleash All CPUs

### How Parallel Execution Works

Instead of **one process** doing all the work, Oracle divides the work among **multiple parallel slave processes** (called PX slaves).

```
╔══════════════════════════════════════════════════════════════════╗
║           SERIAL EXECUTION              PARALLEL EXECUTION      ║
║                                                                  ║
║   ┌──────────┐                  ┌──────────┐                    ║
║   │ Process  │                  │ QC (Query│  ← Query           ║
║   │  (1 CPU) │                  │ Coord.)  │    Coordinator     ║
║   │          │                  └────┬─────┘                    ║
║   │ Scans    │              ┌───────┼───────┐                   ║
║   │ 100M     │              ▼       ▼       ▼                   ║
║   │ rows     │          ┌──────┐┌──────┐┌──────┐               ║
║   │ alone    │          │ PX-1 ││ PX-2 ││ PX-3 │               ║
║   │          │          │ 33M  ││ 33M  ││ 33M  │               ║
║   │ Time:    │          │ rows ││ rows ││ rows │               ║
║   │ 60 sec   │          └──────┘└──────┘└──────┘               ║
║   └──────────┘                                                   ║
║                          Time: ~20 sec (3x faster!) 🚀          ║
╚══════════════════════════════════════════════════════════════════╝
```

### Enabling Parallel Execution

```sql
-- Method 1: Hint (per-query)
SELECT /*+ PARALLEL(o, 8) */ COUNT(*), SUM(amount)
FROM orders o
WHERE order_date > DATE '2025-01-01';

-- Method 2: Table-level default parallelism
ALTER TABLE orders PARALLEL 8;

-- Method 3: Session-level
ALTER SESSION FORCE PARALLEL QUERY PARALLEL 8;
ALTER SESSION FORCE PARALLEL DML PARALLEL 8;

-- Method 4: Statement-level (12c+)
ALTER SESSION SET PARALLEL_DEGREE_POLICY = AUTO;
-- Oracle automatically decides parallelism based on object size & system load

-- Check current parallel settings
SELECT table_name, degree
FROM user_tables
WHERE degree != '1';
```

### Parallel DML — Bulk Operations at Warp Speed

```sql
-- Enable parallel DML for the session (required!)
ALTER SESSION ENABLE PARALLEL DML;

-- Parallel INSERT (load millions of rows fast)
INSERT /*+ PARALLEL(t, 8) APPEND */ INTO target_table t
SELECT /*+ PARALLEL(s, 8) */ * FROM source_table s
WHERE created_date > DATE '2025-01-01';
COMMIT;  -- MUST commit after APPEND

-- Parallel CREATE TABLE AS SELECT (CTAS)
CREATE TABLE orders_archive
PARALLEL 8 NOLOGGING
AS SELECT * FROM orders WHERE order_date < DATE '2024-01-01';

-- Parallel UPDATE
UPDATE /*+ PARALLEL(o, 4) */ orders o
SET status = 'ARCHIVED'
WHERE order_date < DATE '2024-01-01';

-- Parallel DELETE
DELETE /*+ PARALLEL(o, 4) */ FROM orders o
WHERE order_date < DATE '2020-01-01';
```

### Parallel Execution Pitfalls

```
╔═══════════════════════════════════════════════════════════════════╗
║  Pitfall                        │ Solution                       ║
╠═════════════════════════════════╪════════════════════════════════╣
║ Too many parallel processes     │ Set PARALLEL_MAX_SERVERS       ║
║ starving other sessions         │ appropriately                  ║
╠═════════════════════════════════╪════════════════════════════════╣
║ Parallel REDO generation        │ Use NOLOGGING for bulk loads   ║
║ floods the redo logs            │ (but take backup after!)       ║
╠═════════════════════════════════╪════════════════════════════════╣
║ Skew — one PX slave gets        │ Use better partition/hash      ║
║ 90% of the data                │ distribution                   ║
╠═════════════════════════════════╪════════════════════════════════╣
║ Parallel INSERT requires        │ Always COMMIT after APPEND     ║
║ COMMIT before SELECT on table  │                                ║
╠═════════════════════════════════╪════════════════════════════════╣
║ OLTP systems shouldn't use      │ Reserve parallel for batch     ║
║ heavy parallelism              │ jobs, reports, and ETL          ║
╚═════════════════════════════════╧════════════════════════════════╝
```

---

## 🔒 Section 8: SQL Profiles & SQL Plan Baselines

### The Problem

You tuned a query. It's running great. Then one day — **BOOM** — the optimizer picks a different plan and performance tanks. This is called **plan regression** or **plan flip**.

### SQL Profiles — Soft Guidance

A SQL Profile stores **additional statistics** about a query that help the optimizer make better decisions. It doesn't force a plan — it **guides** the optimizer.

```sql
-- SQL Tuning Advisor automatically suggests SQL Profiles
DECLARE
    l_task_name VARCHAR2(100);
BEGIN
    l_task_name := DBMS_SQLTUNE.CREATE_TUNING_TASK(
        sql_id   => 'abc123xyz',
        scope    => DBMS_SQLTUNE.SCOPE_COMPREHENSIVE,
        time_limit => 300  -- 5 minutes
    );
    DBMS_SQLTUNE.EXECUTE_TUNING_TASK(l_task_name);
END;
/

-- View recommendations
SELECT DBMS_SQLTUNE.REPORT_TUNING_TASK('task_name') FROM dual;

-- Accept the SQL Profile (if recommended)
BEGIN
    DBMS_SQLTUNE.ACCEPT_SQL_PROFILE(
        task_name => 'task_name',
        name      => 'MY_PROFILE_FOR_SLOW_QUERY',
        replace   => TRUE
    );
END;
/

-- View active SQL Profiles
SELECT name, sql_text, status, force_matching
FROM dba_sql_profiles;

-- Drop a SQL Profile
BEGIN
    DBMS_SQLTUNE.DROP_SQL_PROFILE('MY_PROFILE_FOR_SLOW_QUERY');
END;
/
```

### SQL Plan Baselines — Hard Lock on Good Plans

SQL Plan Baselines are stronger than Profiles — they **lock in** a specific execution plan. The optimizer can only use a plan that's been **verified** and **accepted** into the baseline.

```sql
-- Enable automatic baseline capture
ALTER SYSTEM SET optimizer_capture_sql_plan_baselines = TRUE;

-- Manually load a good plan into a baseline
DECLARE
    l_plans_loaded PLS_INTEGER;
BEGIN
    l_plans_loaded := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(
        sql_id          => 'abc123xyz',
        plan_hash_value => 987654321,   -- The good plan
        enabled         => 'YES',
        fixed           => 'YES'        -- Make it the ONLY allowed plan
    );
    DBMS_OUTPUT.PUT_LINE('Plans loaded: ' || l_plans_loaded);
END;
/

-- View baselines
SELECT sql_handle, plan_name, enabled, accepted, fixed,
       optimizer_cost, executions, elapsed_time / 1000000 AS elapsed_secs
FROM dba_sql_plan_baselines
ORDER BY last_modified DESC;

-- Evolve a baseline (test new plans and accept if better)
DECLARE
    l_report CLOB;
BEGIN
    l_report := DBMS_SPM.EVOLVE_SQL_PLAN_BASELINE(
        sql_handle => 'SQL_abc123',
        verify     => 'YES',
        commit     => 'YES'
    );
    DBMS_OUTPUT.PUT_LINE(l_report);
END;
/
```

### SQL Profiles vs SQL Plan Baselines

```
╔════════════════════════════════════════════════════════════════════╗
║                   │  SQL Profile          │  SQL Plan Baseline    ║
╠═══════════════════╪═══════════════════════╪═══════════════════════╣
║  What it does     │  Adds supplementary   │  Locks in specific    ║
║                   │  statistics as hints  │  execution plan(s)   ║
╠═══════════════════╪═══════════════════════╪═══════════════════════╣
║  Strictness       │  Soft (guides CBO)    │  Hard (CBO can only  ║
║                   │                       │  use approved plans)  ║
╠═══════════════════╪═══════════════════════╪═══════════════════════╣
║  Adapts to data   │  Yes (CBO still       │  No (same plan even  ║
║  changes?         │  chooses plan)        │  if data changes)    ║
╠═══════════════════╪═══════════════════════╪═══════════════════════╣
║  Best for         │  Fixing bad estimates │  Preventing plan     ║
║                   │  without locking plan │  regression           ║
╠═══════════════════╪═══════════════════════╪═══════════════════════╣
║  Risk             │  Low (flexible)       │  Medium (plan may    ║
║                   │                       │  become suboptimal)  ║
╠═══════════════════╪═══════════════════════╪═══════════════════════╣
║  ⭐ Use when      │  Optimizer makes poor │  "This plan MUST     ║
║                   │  choices due to skew  │  never change"       ║
╚═══════════════════╧═══════════════════════╧═══════════════════════╝
```

---

## 🧪 Section 9: Real-World Performance Tuning Workflow

### The Complete Tuning Methodology

```
╔══════════════════════════════════════════════════════════════════════════╗
║                  ORACLE PERFORMANCE TUNING METHODOLOGY                  ║
║                                                                          ║
║  Step 1: IDENTIFY THE PROBLEM                                           ║
║    │  → "What's slow? A query? The entire instance? A batch job?"       ║
║    │  → Check: V$SESSION, V$ACTIVE_SESSION_HISTORY                      ║
║    │                                                                     ║
║  Step 2: GATHER EVIDENCE                                                ║
║    │  → AWR Report (for historical)                                     ║
║    │  → ASH Report (for real-time)                                      ║
║    │  → ADDM Report (for recommendations)                               ║
║    │                                                                     ║
║  Step 3: FIND THE BOTTLENECK                                            ║
║    │  → Top Wait Events (AWR Section 2)                                 ║
║    │  → Top SQL (by elapsed time, CPU, I/O)                             ║
║    │  → Check: Is it CPU? I/O? Lock? Memory?                            ║
║    │                                                                     ║
║  Step 4: ANALYZE THE SQL                                                ║
║    │  → Get execution plan (DBMS_XPLAN with ALLSTATS)                   ║
║    │  → Compare estimated vs actual rows                                ║
║    │  → Check for: full scans, bad joins, missing indexes               ║
║    │                                                                     ║
║  Step 5: FIX (in this order!)                                           ║
║    │  1. Fix statistics (DBMS_STATS)                                    ║
║    │  2. Add/modify indexes                                             ║
║    │  3. Rewrite SQL (eliminate anti-patterns)                           ║
║    │  4. Partition large tables                                          ║
║    │  5. Use parallel execution (for batch)                              ║
║    │  6. Apply SQL Profile or Baseline (last resort)                    ║
║    │  7. Optimizer hints (absolute last resort)                          ║
║    │                                                                     ║
║  Step 6: VALIDATE                                                       ║
║    │  → Re-run with GATHER_PLAN_STATISTICS                              ║
║    │  → Compare before/after execution plans                            ║
║    │  → Monitor with ASH for regression                                  ║
║    │                                                                     ║
║  Step 7: LOCK IT IN                                                     ║
║       → Create SQL Plan Baseline for critical queries                   ║
║       → Document the change                                             ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### SQL Anti-Patterns That Kill Performance

```sql
-- ❌ ANTI-PATTERN 1: Function on indexed column (kills index usage)
SELECT * FROM employees WHERE UPPER(last_name) = 'SMITH';
-- ✅ FIX: Function-based index
CREATE INDEX idx_emp_upper_name ON employees(UPPER(last_name));

-- ❌ ANTI-PATTERN 2: Implicit type conversion
SELECT * FROM orders WHERE order_id = '12345';  -- order_id is NUMBER
-- Oracle wraps: TO_NUMBER('12345') — can't use index efficiently
-- ✅ FIX: Use correct data type
SELECT * FROM orders WHERE order_id = 12345;

-- ❌ ANTI-PATTERN 3: SELECT * (fetches ALL columns including BLOBs)
SELECT * FROM employees WHERE dept_id = 10;
-- ✅ FIX: Select only needed columns
SELECT employee_id, first_name, last_name FROM employees WHERE dept_id = 10;

-- ❌ ANTI-PATTERN 4: NOT IN with NULLs (returns no rows!)
SELECT * FROM orders WHERE customer_id NOT IN (SELECT customer_id FROM blacklist);
-- If blacklist has a NULL customer_id → entire query returns NOTHING
-- ✅ FIX: Use NOT EXISTS
SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM blacklist b WHERE b.customer_id = o.customer_id);

-- ❌ ANTI-PATTERN 5: Correlated subquery runs once per row
SELECT e.employee_name,
       (SELECT d.department_name FROM departments d WHERE d.dept_id = e.dept_id)
FROM employees e;
-- ✅ FIX: Use a JOIN
SELECT e.employee_name, d.department_name
FROM employees e JOIN departments d ON e.dept_id = d.dept_id;

-- ❌ ANTI-PATTERN 6: Using HAVING instead of WHERE
SELECT department_id, COUNT(*)
FROM employees
GROUP BY department_id
HAVING department_id IN (10, 20, 30);
-- ✅ FIX: Filter BEFORE grouping
SELECT department_id, COUNT(*)
FROM employees
WHERE department_id IN (10, 20, 30)
GROUP BY department_id;
```

---

## 🧪 Interview-Ready Explanations

### "How would you tune a slow Oracle query?"

> *"First, I get the execution plan using DBMS_XPLAN.DISPLAY_CURSOR with ALLSTATS LAST to see actual vs estimated rows. If they differ significantly, I gather fresh statistics with DBMS_STATS. I check for full table scans on large tables and verify indexes exist on WHERE clause columns. I look for anti-patterns like functions on indexed columns, implicit type conversions, and unnecessary SELECT *. For large tables, I consider partitioning. For batch operations, I leverage parallel execution. If needed, I use SQL Tuning Advisor and may accept a SQL Profile. For critical queries, I lock in the good plan with a SQL Plan Baseline."*

### Quick-Fire Interview Questions

| Question | Answer |
|----------|--------|
| "What is AWR?" | Automatic Workload Repository — snapshots of database performance every hour, stored for 8 days. Used to generate performance reports. |
| "AWR vs ASH?" | AWR = periodic snapshots (hourly). ASH = continuous sampling (every second) of active sessions. AWR for historical analysis, ASH for real-time. |
| "What is partition pruning?" | The optimizer skips irrelevant partitions based on the WHERE clause, scanning only the partitions that contain matching data. |
| "Parallel query vs serial?" | Parallel divides work among multiple PX slaves. Use for large scans/batch jobs. Avoid for OLTP (short transactions). |
| "SQL Profile vs Baseline?" | Profile = soft guidance (supplementary stats). Baseline = hard lock on approved execution plans. Profile adapts; Baseline doesn't. |
| "What causes plan regression?" | Stale statistics, data volume changes, new indexes, parameter changes, or Oracle upgrades changing optimizer behavior. |
| "FIRST_ROWS vs ALL_ROWS?" | FIRST_ROWS(n) optimizes to return the first N rows quickly (interactive). ALL_ROWS optimizes for total throughput (batch). |

---

## 🔑 Key Takeaways

```
✅ Execution Plans are your X-ray — learn to read them fluently
✅ AWR + ASH + ADDM = The Holy Trinity of Oracle diagnostics
✅ Top Wait Events tell you WHERE time is being spent
✅ Stale statistics are the #1 cause of bad plans → DBMS_STATS regularly
✅ Hints are a LAST RESORT — fix root causes first
✅ Partitioning: Range for dates, List for categories, Hash for even distribution
✅ Partition pruning = scanning only relevant partitions = massive speedup
✅ Parallel execution = divide work across CPUs (batch only, not OLTP)
✅ SQL Profiles = soft guidance | SQL Baselines = hard plan lock
✅ Tuning order: Statistics → Indexes → SQL rewrite → Partition → Parallel → Profile → Hints
```

---

## 🔗 What's Next?

**Chapter 2B.6 → [Oracle RAC, Data Guard & High Availability](./06-Oracle-HA.md)**
Where we learn how Oracle ensures your database **never goes down** — even when entire data centers fail.

---

> *"Performance tuning is not about making things fast — it's about removing the things that make them slow."* — Tom Kyte, Oracle Legend
