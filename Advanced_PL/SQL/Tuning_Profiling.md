# Oracle PL/SQL — Tuning & Profiling
### Exam Study Guide

Covers: `DBMS_PROFILER` · `DBMS_HPROF` · Code Coverage Techniques · Reducing Context Switches · Function Result Cache · Compiler Optimization (`PLSQL_OPTIMIZE_LEVEL`) · SQL Tuning Inside PL/SQL

> **The golden rule of tuning:** *measure first, then change, then measure again.* Every tool in Sections 1–3 exists to answer "where is the time actually going?" before you touch any code. Sections 4–7 are the techniques you apply once you know.

---

## 1. `DBMS_PROFILER` (Flat, Line-Level Profiler)

### What it is
`DBMS_PROFILER` is the **original PL/SQL profiler**. It records, **for every source line executed**, how many times the line ran and how much time it took. The output is a **flat** (non-hierarchical) picture: you learn *which lines* are hot, but not *who called whom*.

### Setup (one-time)
Run the Oracle-supplied script `proftab.sql` (in `$ORACLE_HOME/rdbms/admin`) in the schema that will own the results. It creates three tables:

| Table | Holds |
|---|---|
| `PLSQL_PROFILER_RUNS` | One row per profiling run (run id, comment, total time) |
| `PLSQL_PROFILER_UNITS` | One row per PL/SQL unit touched in a run (owner, name, type, total time) |
| `PLSQL_PROFILER_DATA` | One row **per line** per unit per run: `LINE#`, `TOTAL_OCCUR` (execution count), `TOTAL_TIME`, `MIN_TIME`, `MAX_TIME` (times are in **nanoseconds**) |

### Usage
```sql
DECLARE
  v_run NUMBER;
BEGIN
  DBMS_PROFILER.start_profiler(run_comment => 'baseline run', run_number => v_run);

  my_batch_pkg.process_orders;          -- the code being profiled

  DBMS_PROFILER.stop_profiler;          -- also flushes collected data to the tables
END;
/
```
Other subprograms: `FLUSH_DATA` (write collected data without stopping), `PAUSE_PROFILER` / `RESUME_PROFILER`.

### Reading the results — find the hottest lines
```sql
SELECT u.unit_name, d.line#, d.total_occur,
       ROUND(d.total_time/1e9, 3) AS seconds,
       s.text
FROM   plsql_profiler_data  d
JOIN   plsql_profiler_units u ON u.runid = d.runid AND u.unit_number = d.unit_number
JOIN   user_source          s ON s.name = u.unit_name AND s.type = u.unit_type AND s.line = d.line#
WHERE  d.runid = :run_id
ORDER  BY d.total_time DESC
FETCH FIRST 10 ROWS ONLY;
```
This lists the ten most expensive lines with their source text — the classic "top-N hot lines" query.

### Important points to remember
- **Line-level and flat:** it answers "which line is slow?", not "which call path led here?" — that's what `DBMS_HPROF` adds (Section 2).
- Results live in **tables you must create** (`proftab.sql`); nothing is collected until `START_PROFILER` and nothing is guaranteed to be in the tables until `STOP_PROFILER` / `FLUSH_DATA`.
- A line that calls SQL shows the **whole SQL round trip** as that line's time — you can't see inside the statement; use SQL tuning tools (Section 7) for that.
- Heavy compiler optimization/inlining (Section 6) can **rearrange or merge code**, so line attribution may look odd; profile with the same settings you run in production, and remember that a debug-oriented `PLSQL_OPTIMIZE_LEVEL=1` build can behave differently.
- `TOTAL_OCCUR` is as valuable as `TOTAL_TIME`: a cheap line executed millions of times is often the real problem (a loop doing row-by-row SQL — Section 4).

### Exercise Questions

**Q1.** A profiling run shows one `UPDATE` line with `TOTAL_OCCUR = 250000` and a modest per-execution time, and it dominates total runtime. What does this suggest, and what is the fix?
> **A:** The line is executing once per row inside a loop — the classic row-by-row ("slow-by-slow") pattern, where the cost is dominated by **250,000 PL/SQL→SQL context switches** rather than by any single slow statement. The fix is to replace it with a single set-based SQL statement, or with `BULK COLLECT` + `FORALL` (Section 4). The high `TOTAL_OCCUR` is the clue; looking only at time-per-execution would hide it.

**Q2.** Why might data be missing from `PLSQL_PROFILER_DATA` right after your test finished?
> **A:** Profiler data is buffered and written to the tables only when `STOP_PROFILER` or `FLUSH_DATA` is called (and only if the profiler was actually started in that session). If the session ended without stopping/flushing, or the code ran in a different session from the one that started the profiler, the tables won't contain the expected rows.

**Q3.** Name a question `DBMS_PROFILER` cannot answer well that `DBMS_HPROF` can.
> **A:** "Which caller is responsible for this expensive subprogram call, and how much of a parent's time is spent inside its children versus in its own code?" `DBMS_PROFILER` is flat and line-based, with no call hierarchy; `DBMS_HPROF` records the caller/callee tree and separates *self* time from *subtree* time.

---

## 2. `DBMS_HPROF` (Hierarchical Profiler, 11g R1+)

### What it is
The **PL/SQL Hierarchical Profiler** records execution at the **subprogram level** and captures the **call hierarchy** (who called whom). It reports, per subprogram: number of calls, **function (self) elapsed time**, and **subtree elapsed time** (self + all descendants). It also reports **SQL statements** (static and dynamic) as separate entries, so you can see SQL time distinctly from PL/SQL time.

### How it differs in workflow
1. **Setup:** `EXECUTE` on `DBMS_HPROF`; a database `DIRECTORY` object with read/write access for the trace file; and (optionally) the four repository tables created with `DBMS_HPROF.CREATE_TABLES`.
2. **Collect:** `START_PROFILING(location, filename)` … run code … `STOP_PROFILING`. This writes a **raw trace file** into the directory.
3. **Analyze** — two routes:
   - `DBMS_HPROF.ANALYZE(location, filename)` loads the raw file into the tables and returns a **run id**; then query the tables.
   - The command-line utility **`plshprof`** converts raw files into an **HTML report** (and with two input files it produces a **difference report** to compare before/after tuning).

```sql
CREATE OR REPLACE DIRECTORY plshprof_dir AS '/home/oracle/prof';
GRANT READ, WRITE ON DIRECTORY plshprof_dir TO app_user;
GRANT EXECUTE ON DBMS_HPROF TO app_user;

-- as app_user
EXEC DBMS_HPROF.create_tables;                       -- once

EXEC DBMS_HPROF.start_profiling('PLSHPROF_DIR', 'run1.trc');
EXEC my_batch_pkg.process_orders;
EXEC DBMS_HPROF.stop_profiling;

DECLARE v_runid NUMBER;
BEGIN
  v_runid := DBMS_HPROF.analyze('PLSHPROF_DIR', 'run1.trc', run_comment => 'baseline');
  DBMS_OUTPUT.put_line('runid = ' || v_runid);
END;
/
```
```
$ plshprof -output run1_report run1.trc          # HTML report
$ plshprof -output diff_report run1.trc run2.trc # before/after difference report
```

### Repository tables
| Table | Holds |
|---|---|
| `DBMSHP_RUNS` | One row per analyzed run |
| `DBMSHP_FUNCTION_INFO` | One row per subprogram: calls, function elapsed time, subtree elapsed time |
| `DBMSHP_PARENT_CHILD_INFO` | Caller→callee relationships with per-edge call counts and times |
| `DBMSHP_TRACE_DATA` | Raw trace events (created with the others) |

### Key metrics (frequently tested)
| Metric | Meaning |
|---|---|
| **Calls** | How many times the subprogram was invoked |
| **Function elapsed time (self time)** | Time spent in the subprogram's *own* code, **excluding** time in subprograms it called |
| **Subtree elapsed time** | Time in the subprogram **including** everything it called |

**Reading them:** high *subtree* but low *function* time means "the time is in something I call — look down the tree." High *function* time means the hotspot is in that subprogram's own code (or the SQL it directly runs).

### Important points to remember
- Granularity is the **subprogram (function/procedure)**, not the line — trade line detail for call-tree insight. Use `DBMS_PROFILER` (or the hierarchical view + your own reading of the code) when you need line-level attribution.
- Output goes to a **file in a DIRECTORY object** first (server-side), then is analyzed — unlike `DBMS_PROFILER`, which writes straight to tables.
- SQL statements appear as pseudo-subprograms (e.g., static SQL entries with the SQL text/`SQL_ID`), so it's easy to see whether time is spent in PL/SQL logic or in SQL.
- The **difference report** from `plshprof` (two trace files) is the standard way to verify that a tuning change helped.
- HPROF works on already-compiled code — no special compile flag is required to collect data.

### Exercise Questions

**Q1.** A subprogram `A` shows subtree time 10 s and function time 0.1 s. `A` calls `B` (subtree 9.9 s). Where should tuning effort go?
> **A:** Into `B` (and B's children). `A`'s own code takes only 0.1 s; almost all of its 10 s is the time spent in its callee. Optimizing `A`'s own statements would gain almost nothing. This "subtree vs. function time" comparison is precisely what the hierarchical profiler is for.

**Q2.** Why is the `plshprof` difference report useful after a tuning change?
> **A:** It takes the raw trace files from two runs (e.g., before and after) and shows side-by-side changes in calls and elapsed times per subprogram, letting you confirm the change actually reduced the targeted hotspot and didn't shift cost elsewhere — objective evidence instead of a guess.

**Q3.** What prerequisites must exist before `DBMS_HPROF.START_PROFILING` can succeed?
> **A:** `EXECUTE` privilege on `DBMS_HPROF`, and a valid database `DIRECTORY` object on which the profiling user has `READ` and `WRITE` (the raw trace file is written there). Repository tables are needed only if you plan to `ANALYZE` into the database (created with `DBMS_HPROF.CREATE_TABLES`); the HTML route with `plshprof` works from the raw file alone.

---

## 3. Code Coverage Techniques

### What it is
**Code coverage** measures **how much of your code was actually executed** by a test run. It doesn't tell you whether the tests are *good* — only which code they *reached*. Untested code is where regressions hide, so coverage is the standard metric for judging a test suite's reach.

### `DBMS_PLSQL_CODE_COVERAGE` (12.2+)
Collects coverage at the **basic block** level. A **basic block** is a single-entry, single-exit sequence of code (roughly, a straight run of statements with no branching in or out). Branches (`IF`/`CASE` arms, loop bodies, exception handlers) create separate blocks, so block coverage naturally reveals untested branches.

**Workflow (same rhythm as the profilers):**
```sql
-- 1. one-time: create the coverage tables
EXEC DBMS_PLSQL_CODE_COVERAGE.create_coverage_tables;   -- FORCE_IT => TRUE drops/recreates

-- 2-4. start, run tests, stop
DECLARE
  v_run NUMBER;
BEGIN
  v_run := DBMS_PLSQL_CODE_COVERAGE.start_coverage(run_comment => 'unit tests build 42');

  test_suite_pkg.run_all;                     -- exercise your code

  DBMS_PLSQL_CODE_COVERAGE.stop_coverage;
END;
/
```
Tables created: `DBMSPCC_RUNS` (one row per run), `DBMSPCC_UNITS` (units touched), `DBMSPCC_BLOCKS` (each basic block with line/column and whether it was `COVERED`; a `NOT_FEASIBLE` flag lets you mark blocks that can't realistically be reached).

**Computing coverage %:**
```sql
SELECT u.name,
       COUNT(*)                                              AS total_blocks,
       SUM(CASE WHEN b.covered = 1 THEN 1 ELSE 0 END)        AS covered_blocks,
       ROUND(100 * SUM(CASE WHEN b.covered = 1 THEN 1 ELSE 0 END) / COUNT(*), 1) AS pct
FROM   dbmspcc_units  u
JOIN   dbmspcc_blocks b ON b.run_id = u.run_id AND b.object_id = u.object_id
WHERE  u.run_id = :run_id
GROUP  BY u.name;
```
*(Column names as commonly documented — confirm against `DESC` in your release.)* `GET_BLOCK_MAP` returns the mapping of *all* blocks to source lines, which is how you calculate total coverage including units with **zero** executed blocks.

### Other coverage techniques
- **Profiler-based line coverage:** with `DBMS_PROFILER`, a line with `TOTAL_OCCUR > 0` was executed; compare against the unit's executable lines to estimate line coverage. This was the standard approach before 12.2 and is what older test frameworks used.
- **Test frameworks** (e.g., utPLSQL) can drive the tests and report coverage using these Oracle packages.
- **Coverage levels** (know the vocabulary): *statement/line* coverage (was each statement run?), *basic block* coverage (was each straight-line segment run?), *branch/decision* coverage (was each true/false outcome taken?). Block coverage of an `IF` with an `ELSE` catches the untested `ELSE`.

### Important points to remember
- Available from **Oracle 12.2**; requires `EXECUTE` on `DBMS_PLSQL_CODE_COVERAGE` (granted to `PUBLIC` by default) and the coverage tables in your schema.
- Collection is **per session** (`START_COVERAGE` → run code in that session → `STOP_COVERAGE`), and each run gets a **`RUN_ID`** (returned by `START_COVERAGE`, a function).
- **100% coverage ≠ correct code.** It proves every block ran, not that assertions verified the results. Conversely, low coverage proves parts of the code are untested.
- **Exception handlers** are the usual uncovered blocks — they need tests that deliberately force failures; some genuinely unreachable blocks can be flagged `NOT_FEASIBLE`.
- Coverage complements profiling: the profilers ask "what's slow?", coverage asks "what's untested?" — run coverage **before** an optimization pass so you have a regression safety net.

### Exercise Questions

**Q1.** What is a "basic block" and why is block coverage more informative than a simple "was the procedure called?" check?
> **A:** A basic block is a single-entry, single-exit run of code with no internal branching. Because every branch (`IF`/`ELSE` arm, loop body, exception handler) forms its own block(s), block coverage reveals exactly which branches never ran. "Was the procedure called?" would report 100% for a procedure whose `ELSE` and `EXCEPTION` sections were never executed.

**Q2.** A unit shows 100% block coverage, yet a bug reaches production. Is that a contradiction?
> **A:** No. Coverage only shows code *executed*, not code *verified*. A test may run every block without asserting anything about the results, or may use inputs that don't trigger the faulty combination of conditions. Coverage is a necessary-but-not-sufficient indicator of test quality.

**Q3.** Why would a unit be missing entirely from a coverage report's percentages if you only compute from `DBMSPCC_BLOCKS` rows?
> **A:** Block rows are recorded for units that were touched during the run; a unit that was never executed may contribute no rows at all, making it invisible (and making the overall percentage look better than reality). `GET_BLOCK_MAP` supplies the complete block list for a unit so you can count *all* blocks — including those in units the tests never reached — when calculating true coverage.

---

## 4. Reducing Context Switches

### Recap of the problem
Every time control passes between the **PL/SQL engine** and the **SQL engine** — in *either* direction — Oracle pays a **context-switch** cost. A loop that issues one SQL statement per row multiplies that cost by the row count.

### Technique 1 — Use one SQL statement instead of a loop
```sql
-- Slow-by-slow:  one switch PER ROW
FOR r IN (SELECT id FROM emp_big WHERE dept_id = 50) LOOP
  UPDATE emp_big SET sal = sal * 1.1 WHERE id = r.id;
END LOOP;

-- Set-based:  one switch TOTAL
UPDATE emp_big SET sal = sal * 1.1 WHERE dept_id = 50;
```
If the logic can be expressed in SQL (including `MERGE`, analytic functions, `CASE`), this is always the fastest option.

### Technique 2 — `BULK COLLECT` (fetch side) and `FORALL` (DML side)
```sql
DECLARE
  CURSOR c IS SELECT id FROM emp_big WHERE dept_id = 50;
  TYPE t_ids IS TABLE OF emp_big.id%TYPE;
  v_ids t_ids;
BEGIN
  OPEN c;
  LOOP
    FETCH c BULK COLLECT INTO v_ids LIMIT 1000;   -- 1 switch per 1000 rows
    EXIT WHEN v_ids.COUNT = 0;                    -- NOT c%NOTFOUND (see below)

    FORALL i IN 1 .. v_ids.COUNT                  -- 1 switch per batch
      UPDATE emp_big SET sal = sal * 1.1 WHERE id = v_ids(i);
  END LOOP;
  CLOSE c;
END;
/
```
- **`LIMIT`** bounds PGA memory: a bare `BULK COLLECT INTO` loads **all** rows into memory and can exhaust the PGA on big result sets.
- With `LIMIT`, test **`v_ids.COUNT = 0`** to exit. Using `c%NOTFOUND` right after the fetch would exit as soon as the last, *partial* batch is fetched — **skipping that batch's processing**. This is a classic exam trap.
- `FORALL` is **not a loop** — it's a single bulk-bind statement handing the whole array to the SQL engine. It supports exactly **one DML statement** in its body.
- Error handling: `FORALL ... SAVE EXCEPTIONS` continues past failed rows and raises `ORA-24381` at the end; details are in `SQL%BULK_EXCEPTIONS` (`ERROR_INDEX`, `ERROR_CODE`). `SQL%BULK_ROWCOUNT(i)` gives per-element row counts. `INDICES OF` / `VALUES OF` let `FORALL` work with sparse collections.
- `RETURNING ... BULK COLLECT INTO` captures values generated by bulk DML without an extra query.

### Technique 3 — Let the cursor `FOR` loop optimization help
When `PLSQL_OPTIMIZE_LEVEL >= 2` (the default), the compiler automatically **bulk-fetches 100 rows at a time** in a cursor `FOR` loop, so a read-only cursor loop is already reasonably efficient — but DML *inside* the loop is still one switch per row, so `FORALL` still matters.

### Technique 4 — Cut SQL→PL/SQL switches (functions called from SQL)
Calling a PL/SQL function from a SQL statement forces a switch **per row**. Ways to reduce it:
| Technique | How it helps |
|---|---|
| **Inline the logic in SQL** | Removes the function call entirely |
| **`WITH FUNCTION` clause** (12.1+) | Defines the function inside the query, which lowers the call/switch overhead |
| **`PRAGMA UDF`** (12.1+) | Marks a stored function as intended for SQL; the compiler optimizes the calling path for that use |
| **Scalar subquery caching:** `SELECT (SELECT f(col) FROM dual) ...` | Oracle caches subquery results per distinct input within the statement, so `f` runs once per *distinct* value |
| **`DETERMINISTIC`** | Tells Oracle the function returns the same output for the same input (allows result reuse in some contexts; the promise is *yours* to keep) |
| **Function result cache** (Section 5) | Serves repeated inputs from the SGA without re-running the body |

```sql
CREATE OR REPLACE FUNCTION tax (p_amt NUMBER) RETURN NUMBER IS
  PRAGMA UDF;
BEGIN
  RETURN p_amt * 0.20;
END;
/

WITH FUNCTION tax2 (p NUMBER) RETURN NUMBER IS BEGIN RETURN p * 0.2; END;
SELECT tax2(salary) FROM employees;
/
```

### Important points to remember
- **`BULK COLLECT` = fewer switches on the query side; `FORALL` = fewer switches on the DML side.** Know which is which.
- Always pair `BULK COLLECT` with `LIMIT` for large sets; typical batch sizes are in the hundreds to low thousands.
- Bulk techniques trade **memory** (collections live in PGA) for **fewer round trips**.
- If a single SQL statement can do the job, prefer it over any PL/SQL loop, bulk or not.
- Switches happen in **both** directions — SQL calling PL/SQL functions counts.

### Exercise Questions

**Q1.** Why can `EXIT WHEN c%NOTFOUND;` immediately after `FETCH ... BULK COLLECT ... LIMIT 1000` lose data?
> **A:** `%NOTFOUND` becomes true when a fetch returns *fewer rows than the limit*, including the **last, partial batch** which still contains valid rows. Exiting right there skips processing those rows. The safe pattern is to test `collection.COUNT = 0` (or process the batch first and exit afterwards).

**Q2.** A developer replaces a row-by-row loop with a single `BULK COLLECT INTO` (no `LIMIT`) of 20 million rows, and the session fails with a memory error. What happened and what's the fix?
> **A:** Without `LIMIT`, all 20 million rows are loaded into a PL/SQL collection held in PGA, exhausting memory. Fix: fetch in batches with `LIMIT n` inside a loop and process (`FORALL`) each batch, keeping memory use bounded — or better, do the whole operation in one SQL statement if possible.

**Q3.** A query calls `SELECT calc_bonus(emp_id) FROM employees` and is slow. Give two ways to reduce the overhead without changing what the function computes.
> **A:** (1) Reduce switch cost — mark the stored function `PRAGMA UDF` or move its logic into the query via a `WITH FUNCTION` clause (12.1+); (2) avoid recomputation — wrap it as a scalar subquery (`(SELECT calc_bonus(emp_id) FROM dual)`) so repeated inputs reuse a cached result, or make it `RESULT_CACHE` if it depends on rarely-changing data. (Rewriting the logic as pure SQL removes the switches entirely.)

---

## 5. Function Result Cache

### What it is
Adding the **`RESULT_CACHE`** clause to a stored function makes Oracle **cache its return value, keyed by the input argument values**, in the **server result cache** (a component of the **SGA**, shared by all sessions). A later call with the same arguments returns the cached value **without executing the function body** — no SQL, no computation.

```sql
CREATE OR REPLACE FUNCTION get_dept_name (p_dept_id IN NUMBER)
  RETURN VARCHAR2
  RESULT_CACHE
IS
  v_name departments.department_name%TYPE;
BEGIN
  SELECT department_name INTO v_name
  FROM   departments
  WHERE  department_id = p_dept_id;
  RETURN v_name;
EXCEPTION
  WHEN NO_DATA_FOUND THEN RETURN NULL;
END;
/
```
Best candidates: functions **called very frequently** with a **limited set of inputs** whose underlying data **rarely changes** (lookup/reference data, configuration, recursive computations).

### Automatic invalidation
Oracle **automatically detects** every table/view the function reads. When a transaction that changes any of them **commits**, the cached results depending on them are **invalidated** (across all instances in RAC) and repopulated on the next call.
- The old **`RELIES_ON`** clause (11.1) listed the dependencies manually; from **11.2 it's deprecated** and **as of 12c it does nothing** — dependencies are always detected automatically.
- A session with **uncommitted** changes to a dependent table **bypasses** the cache (it must see its own changes), so the function executes normally for that session.

### Restrictions (very testable)
- **No `OUT` or `IN OUT` parameters.**
- **`IN` parameters and the return value cannot be** `BLOB`, `CLOB`, `NCLOB`, `REF CURSOR`, objects, or records (and collections/records containing those); collections as **`IN` parameters** are also disallowed.
- Not allowed on **functions in anonymous blocks**, **pipelined table functions**, or **nested functions**.
- If a function is declared before it's defined, `RESULT_CACHE` must appear in **both** declarations.
- In 11g, result-cached functions could not be in **invoker's rights** units; that restriction is not listed in later language references — verify for your release.
- Don't cache functions whose results depend on **session state** rather than arguments (`SYS_CONTEXT`, NLS settings, `SYSDATE`, random values) — the cache is keyed only on arguments and would return stale/wrong values.

### Managing and monitoring
| Item | Purpose |
|---|---|
| `RESULT_CACHE_MAX_SIZE` | SGA memory for the cache (**0 disables** the result cache) |
| `V$RESULT_CACHE_OBJECTS`, `V$RESULT_CACHE_STATISTICS`, `V$RESULT_CACHE_MEMORY` | Inspect entries, hit/miss stats, memory |
| `DBMS_RESULT_CACHE` | `MEMORY_REPORT`, `STATUS`, `FLUSH`, `BYPASS(TRUE/FALSE)` (e.g., bypass during testing) |

*Contrast:* the SQL hint `/*+ RESULT_CACHE */` caches a **query's** result set in the same cache; `RESULT_CACHE_MODE` (MANUAL/FORCE) governs SQL queries.

### Function result cache vs. a package-variable cache
| | Function result cache | Package variable (e.g., associative array) |
|---|---|---|
| Memory | **SGA**, shared by all sessions | **PGA/UGA**, private per session |
| Invalidation | **Automatic** on commit to dependencies | **Manual** — you must code refresh logic |
| Speed of a hit | Slightly slower (SGA access) | Fastest (private memory) |
| Memory cost | One shared copy | One copy **per session** |

### Important points to remember
- The cache stores **argument → return value**, not the SQL statements; it removes the whole function execution on a hit, including its **context switches**.
- Correctness depends on the function being a **pure function of its arguments and committed data**.
- Great for lookup functions called from SQL: turns thousands of per-row executions into a handful of real ones.
- Results can also be **aged out** under memory pressure — treat the cache as an optimization, never as storage.

### Exercise Questions

**Q1.** A result-cached function reads the `TAX_RATES` table. An administrator updates a rate and commits. What happens to cached results?
> **A:** Oracle automatically invalidates the cached results that depend on `TAX_RATES` when the change commits (RELIES_ON is not needed — deprecated in 11.2 and ignored in 12c). The next call recomputes and re-caches the value, so callers see the new rate without any manual flush.

**Q2.** Why is `RESULT_CACHE` a bad idea for `FUNCTION current_user_discount(p_item NUMBER)` that internally uses `SYS_CONTEXT('APP','CUSTOMER_TIER')` to pick a discount?
> **A:** The cache key is only the argument (`p_item`). The result actually depends on hidden session state (`CUSTOMER_TIER`), so a value cached for one user could be returned to another with a different tier — a correctness (and potentially security) bug. Session-dependent functions must not be result-cached, or the dependent context must be passed as an explicit argument.

**Q3.** Which of these functions can be `RESULT_CACHE`d: (a) one with an `OUT` parameter, (b) one returning a `CLOB`, (c) one taking `NUMBER` and returning `VARCHAR2`, (d) a pipelined function?
> **A:** Only **(c)**. `OUT`/`IN OUT` parameters, LOB/`REF CURSOR`/object/record types, and pipelined functions are all disallowed; scalar-in, scalar-out stored functions are the natural fit.

---

## 6. Compiler Optimization (`PLSQL_OPTIMIZE_LEVEL`)

> The engine-level overview appeared in the earlier Engine guide; this section adds the tuning-specific details and how to check/set the parameter.

### The optimizer
Since 10g, the PL/SQL compiler includes an **optimizing compiler** that transforms your code (without changing its meaning) to run faster — e.g., **constant folding**, **dead-code elimination**, **hoisting loop-invariant expressions**, **automatic bulk-fetching in cursor `FOR` loops**, and **subprogram inlining**.

| `PLSQL_OPTIMIZE_LEVEL` | Behavior |
|---|---|
| `0` | No optimization (legacy behavior) |
| `1` | Basic optimizations; **no code rearrangement** (useful for debugging) |
| `2` (**default**) | Full optimization incl. rearrangement and **cursor-loop bulk fetch**; inlining only where `PRAGMA INLINE(name,'YES')` requests it |
| `3` | Adds **aggressive automatic inlining** of subprograms the compiler judges beneficial (pragma can still force `'YES'`/`'NO'`) |

### Setting and checking it
```sql
ALTER SESSION SET PLSQL_OPTIMIZE_LEVEL = 3;              -- affects units compiled afterwards
ALTER PROCEDURE my_proc COMPILE PLSQL_OPTIMIZE_LEVEL = 3 REUSE SETTINGS;

SELECT name, type, plsql_optimize_level, plsql_code_type
FROM   user_plsql_object_settings
WHERE  name = 'MY_PROC';
```
- The setting is **stored per compiled unit** and applies at **compile time** — changing the session/system parameter has no effect on units until they're **recompiled**.
- `REUSE SETTINGS` keeps a unit's existing compile settings when you recompile instead of picking up the current session's values.

### Inlining
Inlining copies a subprogram's body into the call site to eliminate call overhead (only when caller and callee are in the same compilation unit).
```sql
PRAGMA INLINE (helper_fn, 'YES');   -- request inlining for the next call to helper_fn
PRAGMA INLINE (helper_fn, 'NO');    -- suppress it
```
Enable `PLSQL_WARNINGS` informational messages (e.g., **PLW-06005** "inlining … was done") to see what the compiler did.

### Native vs. interpreted compilation — `PLSQL_CODE_TYPE`
- `INTERPRETED` (default): compiled to **byte code** run by the PL/SQL virtual machine.
- `NATIVE`: compiled to **machine code** stored in the database; from 11g no external C compiler is needed.
- Native helps **compute-intensive** PL/SQL (number crunching, loops, string manipulation). It gives **little or no benefit for SQL-bound code**, whose time is spent in the SQL engine.

### Important points to remember
- **Default level is 2**; level 3 is for maximum inlining; level 1 is the "don't rearrange my code" setting (debugging, or to avoid rare edge-case differences from reordering).
- The optimizer **cannot fix** algorithmic problems or context-switch-heavy designs — a row-by-row loop stays slow at any level. Tune SQL and switches first (Sections 4 and 7).
- Recompile after changing the parameter; verify via `USER_PLSQL_OBJECT_SETTINGS`.
- Optimization can alter line attribution in profilers and step order in debuggers — another reason level 1 is used when debugging.

### Exercise Questions

**Q1.** You `ALTER SESSION SET PLSQL_OPTIMIZE_LEVEL = 3;` but a package's performance doesn't change. Why?
> **A:** The parameter is applied when a unit is *compiled*, and the setting is stored with each compiled unit. Existing packages keep the settings they were compiled with until recompiled (e.g., `ALTER PACKAGE p COMPILE PLSQL_OPTIMIZE_LEVEL = 3`). Check `USER_PLSQL_OBJECT_SETTINGS` to see what they actually have.

**Q2.** A batch job is slow because it does 5 million single-row `UPDATE`s in a loop. Would setting `PLSQL_OPTIMIZE_LEVEL = 3` or `PLSQL_CODE_TYPE = NATIVE` fix it?
> **A:** No. The time is in 5 million context switches and SQL executions, not in interpreting PL/SQL byte code, so neither optimization or native compilation addresses the bottleneck. The fix is structural: a single set-based statement or `BULK COLLECT`/`FORALL`.

**Q3.** Which inlining behavior differs between level 2 and level 3, and how can you override the compiler at level 3?
> **A:** At level 2 the compiler inlines only calls explicitly requested with `PRAGMA INLINE(sub,'YES')`; at level 3 it also inlines automatically wherever it judges it beneficial. At any level you can use `PRAGMA INLINE(sub,'NO')` to prevent inlining of a specific call (and `'YES'` to force priority at level 3).

---

## 7. SQL Tuning Inside PL/SQL

### Why it matters
In most real applications, PL/SQL time is dominated by the **SQL it runs**. A perfectly optimized PL/SQL loop around a bad SQL statement is still slow. The profilers point you to the statement; then you tune the statement itself.

### Step 1 — Identify the statement
- `DBMS_HPROF` lists SQL statements as separate entries (with `SQL_ID`); `DBMS_PROFILER` shows the calling line.
- Find its plan and stats: `V$SQL` / `V$SQL_PLAN` by `SQL_ID`, AWR/ASH reports, or SQL trace (`DBMS_MONITOR`, `ALTER SESSION SET SQL_TRACE`) formatted with `TKPROF`.
- Display the actual plan: `SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(:sql_id, NULL, 'ALLSTATS LAST'));`

### Step 2 — Apply standard fixes (PL/SQL-specific ones highlighted)
| Problem | Fix |
|---|---|
| Literals concatenated into dynamic SQL → constant hard parses | **Bind variables** (static SQL binds PL/SQL variables automatically — a benefit of static SQL) |
| **Function applied to an indexed column** in `WHERE` (`WHERE TRUNC(hire_date) = ...`, `WHERE UPPER(name) = ...`) — blocks index use | Rewrite as a range/sargable predicate, or create a **function-based index** |
| **Implicit data-type conversion** (e.g., comparing a `VARCHAR2` column to a `NUMBER` variable) — disables index, adds conversion cost (compiler warning **PLW-07204**) | Declare variables with `%TYPE` of the column so types match |
| Row-by-row loops | Set-based SQL, `MERGE`, or `BULK COLLECT`/`FORALL` (Section 4) |
| `SELECT COUNT(*)` just to test existence | Use `EXISTS`, or fetch with `ROWNUM = 1`/`FETCH FIRST 1 ROW ONLY` and handle `NO_DATA_FOUND` |
| Two statements to change then re-read a row | **`RETURNING ... INTO`** on the DML — one round trip |
| Selecting all columns (`SELECT *`, `%ROWTYPE`) when few are needed | Select only the needed columns into a smaller custom record |
| Unbounded `BULK COLLECT` | Add `LIMIT` |
| Repeated identical lookups | `RESULT_CACHE` function, scalar-subquery caching, or a package-level cache |
| Stale/bad statistics → poor plans | Gather optimizer statistics (`DBMS_STATS`) |
| Heavy dynamic SQL for structures that don't vary | Prefer **static SQL** (compile-time checked, cursor-cached, bind-friendly) |

### Example — before and after
```sql
-- Problem: function on indexed column + implicit conversion + existence via COUNT(*)
SELECT COUNT(*) INTO v_cnt
FROM   orders
WHERE  TRUNC(order_date) = TRUNC(SYSDATE)      -- kills an index on order_date
AND    customer_id = p_cust_char;              -- VARCHAR2 vs NUMBER column: implicit conversion
IF v_cnt > 0 THEN ...

-- Better
SELECT 1 INTO v_dummy FROM dual
WHERE  EXISTS (SELECT 1
               FROM   orders
               WHERE  order_date >= TRUNC(SYSDATE)
               AND    order_date <  TRUNC(SYSDATE) + 1     -- sargable range
               AND    customer_id = p_cust_id);            -- p_cust_id declared as orders.customer_id%TYPE
```

### Static vs. dynamic SQL for tuning
Static SQL (embedded directly in PL/SQL) is **parsed once, cached by the PL/SQL engine**, dependency-tracked, and uses bind variables automatically. Dynamic SQL must be parsed at run time (a soft parse at best) and needs explicit binds — use it only when the statement's *structure* truly varies.

### Important points to remember
- **Profile first, then tune the statement the profiler points at** — don't guess.
- Two very common PL/SQL-specific SQL killers: **implicit conversions** and **functions wrapped around indexed columns**.
- Declaring variables with **`%TYPE`/`%ROWTYPE`** keeps types aligned with columns (avoiding conversions) and adapts automatically to column changes.
- Use **bind variables**; with static SQL this is automatic, with dynamic SQL you must write `USING`.
- The best-tuned SQL still suffers if called in a slow-by-slow loop — combine SQL tuning with switch reduction.

### Exercise Questions

**Q1.** `WHERE UPPER(last_name) = 'SMITH'` is slow despite an index on `last_name`. Why, and give two fixes.
> **A:** Wrapping the indexed column in `UPPER()` means the optimizer cannot use the ordinary index on `last_name` (the indexed values aren't uppercased), so it falls back to a full scan. Fixes: create a **function-based index** on `UPPER(last_name)`, or store/compare data in a consistent case (e.g., normalize on insert) so the plain index applies.

**Q2.** Why is `SELECT COUNT(*) INTO v_cnt ... ; IF v_cnt > 0` often a poor way to test whether rows exist?
> **A:** `COUNT(*)` may have to scan and count *all* matching rows even though you only need to know whether at least one exists. `EXISTS` (or fetching a single row) can stop at the first match, doing far less work — especially on large tables.

**Q3.** A PL/SQL variable `v_id VARCHAR2(10)` is compared to a `NUMBER` column `customer_id` in a `WHERE` clause, and the query is slow. What is likely happening and how do you prevent it?
> **A:** Oracle implicitly converts one side to match the other — here typically converting the *column* to the variable's type or the reverse — which can disable an index on `customer_id` and add per-row conversion cost (the compiler may raise PLW-07204 when performance warnings are enabled). Prevent it by declaring the variable with `orders.customer_id%TYPE` (or a matching `NUMBER`) so types align exactly.

---

## Putting It Together — A Tuning Workflow

1. **Establish a safety net:** run the test suite under `DBMS_PLSQL_CODE_COVERAGE` so you know what's protected.
2. **Find the hotspot:** profile with `DBMS_HPROF` (call tree, SQL vs. PL/SQL time); drill into lines with `DBMS_PROFILER` if needed.
3. **Diagnose:** is it a slow **SQL statement** (Section 7), too many **context switches** (Section 4), or **repeated lookups** (Section 5)?
4. **Apply the fix** — set-based SQL / bulk processing / result cache / SQL rewrite — before reaching for compiler flags.
5. **Tune compilation last:** `PLSQL_OPTIMIZE_LEVEL` (2→3 for inlining), `NATIVE` for compute-heavy code; recompile and confirm via `USER_PLSQL_OBJECT_SETTINGS`.
6. **Re-measure:** compare with a `plshprof` difference report; re-run tests to confirm behavior is unchanged.

---

## Quick Cross-Topic Summary Table

| Tool / Technique | Granularity / Purpose | Key Point |
|---|---|---|
| `DBMS_PROFILER` | **Line-level**, flat | Tables from `proftab.sql`; `TOTAL_OCCUR` & `TOTAL_TIME` (ns) |
| `DBMS_HPROF` | **Subprogram-level**, hierarchical | Trace file in a `DIRECTORY`; `ANALYZE` / `plshprof`; function vs. subtree time; diff report |
| `DBMS_PLSQL_CODE_COVERAGE` (12.2+) | **Basic-block** coverage | `DBMSPCC_RUNS/UNITS/BLOCKS`; 100% coverage ≠ correct |
| Reduce context switches | Fewer engine hand-offs | Set-based SQL > `BULK COLLECT` + `FORALL` (with `LIMIT`) > row-by-row; `PRAGMA UDF`, `WITH FUNCTION` |
| Function result cache | Cache argument → result in **SGA** | Auto-invalidated on commit; no OUT params/LOB/record/etc.; avoid session-dependent functions |
| `PLSQL_OPTIMIZE_LEVEL` | Compile-time optimization | 0/1/2(default)/3; per-unit setting; recompile required; level 3 = auto-inlining |
| SQL tuning in PL/SQL | Fix the statements PL/SQL runs | Binds, sargable predicates, matching types (`%TYPE`), `EXISTS`, `DBMS_XPLAN` |