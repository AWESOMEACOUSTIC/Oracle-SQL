# Oracle PL/SQL — Built-in Packages
### Exam Study Guide

Covers: `DBMS_SQL` · `UTL_FILE` (file I/O) · `DBMS_METADATA` · `DBMS_SCHEDULER` · `DBMS_UTILITY`

> `DBMS_SCHEDULER` also appeared in an earlier guide as a comparison against `DBMS_JOB`. Here it gets a full standalone treatment (job types, schedules, chains, windows, monitoring) so this file can be studied on its own.

---

## 1. `DBMS_SQL`

### What it is
`DBMS_SQL` is the **original dynamic SQL API** (pre-dating `EXECUTE IMMEDIATE`). Instead of handing a finished string to one statement, you work with an explicit **cursor** through a sequence of calls: open, parse, bind, define, execute, fetch, read values, close. That verbosity buys flexibility that native dynamic SQL (NDS) can't match.

### The call sequence
| Step | Subprogram | Purpose |
|---|---|---|
| 1 | `OPEN_CURSOR` (function) | Returns an integer **cursor id** |
| 2 | `PARSE(c, statement, language_flag)` | Parses the text; use `DBMS_SQL.NATIVE` as the flag |
| 3 | `BIND_VARIABLE` / `BIND_ARRAY` | Supplies values for placeholders (`:name`) — by **name** |
| 4 | `DEFINE_COLUMN` / `DEFINE_ARRAY` | *(queries only)* declares a variable for each select-list column — by **position** (starting at 1) |
| 5 | `EXECUTE` (function) | Runs the statement; for DML returns the **number of rows processed** |
| 6 | `FETCH_ROWS` (function) | *(queries)* fetches the next row; returns 0 when no more rows |
| 7 | `COLUMN_VALUE` | Copies a fetched column into a PL/SQL variable |
| 8 | `VARIABLE_VALUE` | Reads back an `OUT` / `RETURNING` bind |
| 9 | `CLOSE_CURSOR` | Releases the cursor |

Helpers: `IS_OPEN`, `DESCRIBE_COLUMNS` (discover the select list at run time), `EXECUTE_AND_FETCH`, `LAST_ROW_COUNT`, `LAST_ERROR_POSITION`.

### Example 1 — DML with binds
```sql
DECLARE
  c INTEGER;
  n INTEGER;
BEGIN
  c := DBMS_SQL.open_cursor;
  DBMS_SQL.parse(c,
    'UPDATE employees SET salary = salary * :pct WHERE department_id = :dept',
    DBMS_SQL.native);
  DBMS_SQL.bind_variable(c, ':pct',  1.05);
  DBMS_SQL.bind_variable(c, ':dept', 50);
  n := DBMS_SQL.execute(c);                       -- rows processed
  DBMS_OUTPUT.put_line(n || ' rows updated');
  DBMS_SQL.close_cursor(c);
EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_SQL.is_open(c) THEN DBMS_SQL.close_cursor(c); END IF;   -- avoid a cursor leak
    RAISE;
END;
/
```

### Example 2 — a query whose columns are unknown until run time
This is the classic reason to use `DBMS_SQL`: a generic "print any query" utility.
```sql
CREATE OR REPLACE PROCEDURE print_query (p_sql IN VARCHAR2) IS
  c        INTEGER := DBMS_SQL.open_cursor;
  v_cols   INTEGER;
  v_desc   DBMS_SQL.desc_tab;
  v_val    VARCHAR2(4000);
  v_status INTEGER;
BEGIN
  DBMS_SQL.parse(c, p_sql, DBMS_SQL.native);
  DBMS_SQL.describe_columns(c, v_cols, v_desc);        -- how many columns? what names?
  FOR i IN 1 .. v_cols LOOP
    DBMS_SQL.define_column(c, i, v_val, 4000);         -- fetch everything as text
  END LOOP;
  v_status := DBMS_SQL.execute(c);
  WHILE DBMS_SQL.fetch_rows(c) > 0 LOOP
    FOR i IN 1 .. v_cols LOOP
      DBMS_SQL.column_value(c, i, v_val);
      DBMS_OUTPUT.put(v_desc(i).col_name || '=' || v_val || '  ');
    END LOOP;
    DBMS_OUTPUT.new_line;
  END LOOP;
  DBMS_SQL.close_cursor(c);
EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_SQL.is_open(c) THEN DBMS_SQL.close_cursor(c); END IF;
    RAISE;
END;
/
```
*(If `p_sql` can come from untrusted input, the same injection defenses apply as with any dynamic SQL — validate or whitelist it.)*

### `DBMS_SQL` vs. native dynamic SQL (`EXECUTE IMMEDIATE`)
| | `EXECUTE IMMEDIATE` (NDS) | `DBMS_SQL` |
|---|---|---|
| Syntax | Compact, one statement | Verbose, many calls |
| Speed | Generally **faster** — built into the PL/SQL engine | Generally slower for simple cases |
| **Unknown number/types of select-list columns** | Not possible | **Yes** — `DESCRIBE_COLUMNS` + `DEFINE_COLUMN` |
| **Unknown number of bind variables** at compile time | Not practical (`USING` list is fixed in the code) | **Yes** — bind in a loop |
| **Parse once, execute many** with different binds | Re-issues the statement each time | **Yes** — re-bind and `EXECUTE` again on the same parsed cursor |
| Statement text > 32 KB | No | `PARSE` has overloads accepting a table of text pieces |
| Array (bulk) binds | `FORALL` / `BULK COLLECT` | `BIND_ARRAY` / `DEFINE_ARRAY` |

**Interoperability (11g+):** `DBMS_SQL.TO_REFCURSOR(cursor_number)` converts an opened, parsed, **executed** `DBMS_SQL` cursor into a `REF CURSOR` (the `DBMS_SQL` cursor id must not be used afterwards), and `DBMS_SQL.TO_CURSOR_NUMBER(ref_cursor)` goes the other way. This lets you build the statement dynamically with `DBMS_SQL` yet return a `REF CURSOR` to a client.

### Important points to remember
- **Use NDS by default; reach for `DBMS_SQL` only for "Method 4" cases** — the number/types of columns or binds are unknown at compile time — or when you want parse-once/execute-many.
- **DDL is executed at `PARSE` time** — there is no separate `EXECUTE` needed (and DDL still commits implicitly).
- Bind by **name** (`BIND_VARIABLE`), define/read columns by **position** (`DEFINE_COLUMN`, `COLUMN_VALUE`).
- **Always close cursors**, including in the exception handler. Leaked cursors accumulate until `ORA-01000: maximum open cursors exceeded`.
- Since 11g there are **security checks on cursor use**: the cursor must be used by the same effective user/roles that parsed it, and guessed/invalid cursor numbers are rejected (e.g., `ORA-29471`), a defense against cursor-number-guessing attacks.
- Binds keep data values out of the SQL text (performance + injection safety), exactly as with NDS; identifiers still can't be bound and must be validated (`DBMS_ASSERT`).

### Exercise Questions

**Q1.** When is `DBMS_SQL` genuinely required rather than `EXECUTE IMMEDIATE`?
> **A:** When the shape of the statement isn't known at compile time in a way `EXECUTE IMMEDIATE` can express — chiefly when the **number or datatypes of the select-list columns** are only discovered at run time (a generic query browser/exporter using `DESCRIBE_COLUMNS`), or the **number of bind variables** varies at run time. `EXECUTE IMMEDIATE`'s `INTO` and `USING` lists are fixed in the source code, so it cannot handle those cases. `DBMS_SQL` is also the tool for parsing once and executing repeatedly with new bind values.

**Q2.** A procedure using `DBMS_SQL` runs fine in testing, but after several days in production sessions start failing with `ORA-01000`. What is the likely defect?
> **A:** A **cursor leak**: the code opens cursors with `OPEN_CURSOR` but doesn't `CLOSE_CURSOR` on every path — typically when an exception occurs before the close. Each leaked cursor counts against the session's `OPEN_CURSORS` limit until it's exhausted. Fix: close the cursor in the normal path **and** in the exception handler (guarded by `IS_OPEN`).

**Q3.** You call `DBMS_SQL.PARSE(c, 'CREATE TABLE t (id NUMBER)', DBMS_SQL.native);` and never call `EXECUTE`. Is the table created?
> **A:** Yes. DDL statements are executed when they are **parsed** under `DBMS_SQL`, so the `PARSE` call itself creates the table (with the usual implicit commit). `EXECUTE` is needed for DML, queries and PL/SQL blocks, not for DDL.

**Q4.** What is `DBMS_SQL.TO_REFCURSOR` used for?
> **A:** To convert a `DBMS_SQL` cursor (already opened, parsed and executed) into a strongly usable `SYS_REFCURSOR`. It lets a program build and bind a statement with `DBMS_SQL`'s flexibility and then hand the result to a caller — such as a client application — as an ordinary `REF CURSOR`. Once converted, the original `DBMS_SQL` cursor number can no longer be used.

---

## 2. `UTL_FILE` (File I/O)

### What it is
`UTL_FILE` lets PL/SQL **read and write operating-system text (and byte) files** on the **database server**. It provides a *restricted* form of stream file I/O — open, read/write, close.

> **Key exam fact:** files are opened on the **server's** file system, never on the client machine running SQL*Plus or the application (unless using client-side PL/SQL such as Oracle Forms).

### Security model — directory objects
Access is controlled by **`DIRECTORY` objects**:
```sql
CREATE OR REPLACE DIRECTORY export_dir AS '/u01/app/exports';   -- needs CREATE ANY DIRECTORY
GRANT READ, WRITE ON DIRECTORY export_dir TO hr;
```
- A user needs `READ` to read and `WRITE` to write/create/delete files through that directory object.
- The **OS account that runs the database** must also have permission on the real folder.
- **Subdirectories are not accessible** through a parent directory object — each needs its own directory object.
- The old `UTL_FILE_DIR` initialization parameter is **deprecated since 12.2** (still works for backward compatibility); directory objects replace it and can be managed **without restarting** the database.
- OS-specific parameters such as shell environment variables cannot be used in a location or file name.

### Core subprograms
| Subprogram | Purpose |
|---|---|
| `FOPEN(location, filename, open_mode, max_linesize)` | Opens a file, returns a `UTL_FILE.FILE_TYPE` handle. Modes: `R` read text, `W` write text (**overwrites**), `A` append text, and `RB`/`WB`/`AB` for byte mode. `FOPEN_NCHAR` for multibyte data. |
| `GET_LINE(file, buffer [, len])` | Reads one line; raises **`NO_DATA_FOUND` at end of file** |
| `PUT`, `PUT_LINE`, `NEW_LINE`, `PUTF` | Write text (with/without line terminator; `PUTF` is formatted) |
| `GET_RAW`, `PUT_RAW` | Byte-mode I/O (ignore line terminators) |
| `FFLUSH` | Forces buffered output to disk |
| `FCLOSE`, `FCLOSE_ALL` | Closes one file / every file open in the session |
| `IS_OPEN` | Tests a handle |
| `FREMOVE`, `FRENAME`, `FCOPY` | Delete / rename / copy files |
| `FGETATTR` | File existence, length, block size |
| `FSEEK`, `FGETPOS` | Reposition / report the file pointer |

**Line-length limit:** `max_linesize` defaults to **1024** bytes and can be set from **1 to 32767** in `FOPEN`. If it is too small for the data you write, the write fails.

### Example — export to CSV
```sql
DECLARE
  f UTL_FILE.file_type;
BEGIN
  f := UTL_FILE.fopen('EXPORT_DIR', 'employees.csv', 'w', 32767);   -- directory name in UPPER CASE
  UTL_FILE.put_line(f, 'ID,NAME,SALARY');
  FOR r IN (SELECT employee_id, last_name, salary FROM employees) LOOP
    UTL_FILE.put_line(f, r.employee_id || ',' || r.last_name || ',' || r.salary);
  END LOOP;
  UTL_FILE.fclose(f);
EXCEPTION
  WHEN OTHERS THEN
    IF UTL_FILE.is_open(f) THEN UTL_FILE.fclose(f); END IF;
    RAISE;
END;
/
```

### Example — read a file line by line
```sql
DECLARE
  f      UTL_FILE.file_type;
  v_line VARCHAR2(32767);
BEGIN
  f := UTL_FILE.fopen('EXPORT_DIR', 'employees.csv', 'r', 32767);
  LOOP
    BEGIN
      UTL_FILE.get_line(f, v_line);
    EXCEPTION
      WHEN NO_DATA_FOUND THEN EXIT;         -- end of file
    END;
    DBMS_OUTPUT.put_line(v_line);
  END LOOP;
  UTL_FILE.fclose(f);
END;
/
```

### `UTL_FILE` exceptions (frequently tested)
| Exception | Meaning |
|---|---|
| `INVALID_PATH` (ORA-29280) | Directory/location invalid or not visible/authorized |
| `INVALID_MODE` (ORA-29281) | Bad `open_mode` in `FOPEN` |
| `INVALID_FILEHANDLE` (ORA-29282) | Handle not valid (e.g., file already closed) |
| `INVALID_OPERATION` (ORA-29283) | File could not be opened/operated on as requested (e.g., reading a file that doesn't exist, or OS permission problem) |
| `READ_ERROR` (ORA-29284) / `WRITE_ERROR` (ORA-29285) | OS error during read/write (also a too-small buffer / too-long line) |
| `INVALID_MAXLINESIZE` (ORA-29287) | `max_linesize` outside 1–32767 |
| `INVALID_FILENAME` (ORA-29288) | Bad file name |
| `INVALID_OFFSET` (ORA-29290) | Bad `FSEEK` offset |
| `RENAME_FAILED` (ORA-29292), `DELETE_FAILED` | Rename/delete didn't succeed |
| `FILE_OPEN` | Requested operation failed because the file is open |
| `INTERNAL_ERROR` | Unspecified PL/SQL error |

### Important points to remember
- The **location** argument is the **directory object name** (stored in upper case unless created with quotes) — `'export_dir'` in lower case is a classic cause of `INVALID_PATH`.
- Always **close files** (`FCLOSE`) and close in the **exception handler** too; unflushed data may otherwise be lost. `FCLOSE_ALL` is a safety net.
- `GET_LINE` signals end-of-file with **`NO_DATA_FOUND`**, not a return code.
- Opening in `W` mode **replaces** an existing file; use `A` to append.
- Because it touches the server file system, **restrict access**: grant `READ`/`WRITE` on directory objects narrowly, and (as a common hardening step) revoke `EXECUTE` on `UTL_FILE` from `PUBLIC` and grant it only where needed.
- Directory-object privileges granted **through a role** aren't usable inside definer's-rights stored code — grant them **directly** to the code's owner.

### Exercise Questions

**Q1.** `UTL_FILE.FOPEN('export_dir', 'a.txt', 'w')` raises `INVALID_PATH`, although `CREATE DIRECTORY export_dir ...` succeeded. Give the most likely causes.
> **A:** (1) **Case:** the directory object was created unquoted, so its name is stored as `EXPORT_DIR`; the location string must match it exactly (`'EXPORT_DIR'`). (2) The user lacks `READ`/`WRITE` on the directory object (or the privilege was granted only via a role and the code is definer's-rights PL/SQL). (3) The directory path doesn't exist or isn't reachable by the database OS account.

**Q2.** A DBA runs a procedure that writes a report using `UTL_FILE` from SQL*Plus on their laptop. Where does the file end up?
> **A:** On the **database server's** file system, in the directory the directory object points to — not on the laptop. `UTL_FILE` runs inside the database server process, so all paths refer to server-side storage. To get the file to the client you need another mechanism (copy the file, or return the data through the client).

**Q3.** How do you correctly detect the end of a file when reading with `GET_LINE`, and what happens to the file handle afterwards?
> **A:** Catch **`NO_DATA_FOUND`** raised by `GET_LINE` (typically in a small nested block inside the read loop, then `EXIT`). The handle stays open until you call `FCLOSE` — reaching end-of-file doesn't close it, so always close it afterwards (and in the exception handler).

**Q4.** Why is `UTL_FILE_DIR` no longer recommended?
> **A:** It was deprecated as of 12.2. Directory objects are more flexible and granular (privileges per directory, per user), can be created/changed **dynamically without a restart**, and are consistent with other Oracle tools; `UTL_FILE_DIR` required editing an initialization parameter and gave broad, all-or-nothing path access.

---

## 3. `DBMS_METADATA`

### What it is
`DBMS_METADATA` **extracts the definitions (metadata) of database objects** — returned as **DDL text** (`CREATE ...` statements) or as **XML**. It is the supported way to reverse-engineer objects for version control, environment migration, documentation and comparison (and is the same engine Data Pump uses).

### Main functions
| Function | Returns |
|---|---|
| `GET_DDL(object_type, name [, schema])` | DDL of **one** object as a **CLOB** |
| `GET_DEPENDENT_DDL(object_type, base_object_name [, base_object_schema])` | DDL of all objects of a type **dependent on** a base object (e.g., the indexes, triggers, constraints of a table) |
| `GET_GRANTED_DDL(object_type, grantee)` | DDL to recreate **grants** — `'OBJECT_GRANT'`, `'SYSTEM_GRANT'`, `'ROLE_GRANT'` |
| `GET_XML(...)` | The same information as XML |

Object type names are strings such as `'TABLE'`, `'INDEX'`, `'VIEW'`, `'SEQUENCE'`, `'TRIGGER'`, `'PROCEDURE'`, `'FUNCTION'`, `'PACKAGE'` (spec + body), `'PACKAGE_SPEC'`, `'PACKAGE_BODY'`, `'TYPE'`, `'SYNONYM'`, `'MATERIALIZED_VIEW'`, `'USER'`, `'DB_LINK'`.

### Basic usage
```sql
SET LONG 1000000 PAGESIZE 0 LINESIZE 200     -- SQL*Plus: otherwise the CLOB is truncated

SELECT DBMS_METADATA.get_ddl('TABLE', 'EMPLOYEES', 'HR') FROM dual;
SELECT DBMS_METADATA.get_dependent_ddl('INDEX', 'EMPLOYEES', 'HR') FROM dual;
SELECT DBMS_METADATA.get_granted_ddl('OBJECT_GRANT', 'APP_USER') FROM dual;
```
`GET_DEPENDENT_DDL` / `GET_GRANTED_DDL` raise an error (e.g., `ORA-31608`) if no such dependent objects or grants exist.

### Customizing output — transforms
```sql
BEGIN
  DBMS_METADATA.set_transform_param(DBMS_METADATA.session_transform, 'SQLTERMINATOR',      TRUE);
  DBMS_METADATA.set_transform_param(DBMS_METADATA.session_transform, 'PRETTY',             TRUE);
  DBMS_METADATA.set_transform_param(DBMS_METADATA.session_transform, 'SEGMENT_ATTRIBUTES', FALSE);
  DBMS_METADATA.set_transform_param(DBMS_METADATA.session_transform, 'STORAGE',            FALSE);
END;
/
```
| Transform parameter | Effect | Default |
|---|---|---|
| `SQLTERMINATOR` | Append `;` (or `/`) to each statement | **FALSE** |
| `PRETTY` | Indented, line-fed output | **TRUE** |
| `SEGMENT_ATTRIBUTES` | Emit physical attributes, storage, tablespace, logging | TRUE |
| `STORAGE` | Emit the storage clause (ignored if `SEGMENT_ATTRIBUTES` is FALSE) | TRUE |
| `TABLESPACE` | Emit tablespace clauses | TRUE |
| `CONSTRAINTS` / `REF_CONSTRAINTS` | Include constraints / foreign keys in table DDL | TRUE |
| `CONSTRAINTS_AS_ALTER` | Emit constraints as separate `ALTER TABLE` statements | FALSE |
| `PARTITIONING` | Include partitioning clauses | TRUE |

`DBMS_METADATA.SESSION_TRANSFORM` applies a setting to the **whole session** (what `GET_DDL` inherits). `'DEFAULT'` resets. Use `SET_REMAP_PARAM` (with the `MODIFY` transform) to **remap** schema or tablespace names when generating DDL for another environment.

### Browsing API — many objects with filters
```sql
DECLARE
  h     NUMBER;
  th    NUMBER;
  v_ddl CLOB;
BEGIN
  h  := DBMS_METADATA.open('TABLE');                       -- what kind of object
  DBMS_METADATA.set_filter(h, 'SCHEMA', 'HR');             -- which ones
  th := DBMS_METADATA.add_transform(h, 'DDL');             -- what output
  DBMS_METADATA.set_transform_param(th, 'SQLTERMINATOR', TRUE);
  LOOP
    v_ddl := DBMS_METADATA.fetch_clob(h);                  -- NULL when finished
    EXIT WHEN v_ddl IS NULL;
    -- write v_ddl to a file (UTL_FILE), a table, etc.
  END LOOP;
  DBMS_METADATA.close(h);
END;
/
```
The sequence is **OPEN → SET_FILTER → ADD_TRANSFORM → FETCH_* loop → CLOSE**.

### Important points to remember
- Output is a **CLOB**; in SQL*Plus set `LONG` large enough or the DDL looks cut off.
- **Privileges:** you can retrieve your own objects' metadata; other schemas' objects require the **`SELECT_CATALOG_ROLE`** role. Roles aren't active inside definer's-rights stored code, so such code may fail to "see" objects — grant appropriately or use invoker's rights.
- A table's `GET_DDL` includes constraints (like its primary key) but **not** its indexes, triggers or grants separately — use `GET_DEPENDENT_DDL` and `GET_GRANTED_DDL` for those.
- `SQLTERMINATOR` defaults to **FALSE**, so scripts need it switched on to be runnable.
- Transform parameters can be set at **session level** (`SESSION_TRANSFORM`) or on a **specific transform handle** (browsing API).

### Exercise Questions

**Q1.** You spool `GET_DDL` output in SQL*Plus and the `CREATE TABLE` text stops mid-way. Why, and what is the fix?
> **A:** `GET_DDL` returns a **CLOB**, and SQL*Plus displays only as many characters as the `LONG` setting allows (default 80). Set `SET LONG 1000000` (and typically `SET PAGESIZE 0`) before running the query so the whole definition is displayed.

**Q2.** How would you script a table together with its indexes and the privileges granted on it to a user, ready to run in another schema?
> **A:** Combine calls: `GET_DDL('TABLE', ...)` for the table, `GET_DEPENDENT_DDL('INDEX', ...)` (and `'CONSTRAINT'`/`'TRIGGER'` as needed) for dependent objects, and `GET_GRANTED_DDL('OBJECT_GRANT', grantee)` for the grants. Turn on `SQLTERMINATOR` so each statement ends with `;`, and use `SET_REMAP_PARAM`/schema remapping (or edit the schema names) for the target schema; optionally disable `SEGMENT_ATTRIBUTES`/`STORAGE` to drop environment-specific physical clauses.

**Q3.** When would you use the browsing API (`OPEN`/`SET_FILTER`/`FETCH_CLOB`) instead of `GET_DDL`?
> **A:** When you need DDL for **many objects at once** selected by a filter — e.g., every table in a schema, or all objects matching a name pattern — rather than one named object. You open a handle for the object type, apply filters, attach a DDL transform, then loop over `FETCH_CLOB` until it returns `NULL`, finally `CLOSE`.

**Q4.** What is the difference between setting a transform parameter with `SESSION_TRANSFORM` and with a handle returned by `ADD_TRANSFORM`?
> **A:** `SESSION_TRANSFORM` sets the option for the **whole session**, and `GET_DDL`/`GET_DEPENDENT_DDL` inherit it. A handle from `ADD_TRANSFORM` applies **only to that particular browsing-API extraction**, letting different extractions in the same session use different options.

---

## 4. `DBMS_SCHEDULER`

### What it is
The **Oracle Scheduler** — the database's job-scheduling engine — controlled through `DBMS_SCHEDULER`. It runs PL/SQL, stored procedures, OS programs/scripts and chains of steps, on time-based or event-based schedules, with logging, priorities and resource control. (It replaces `DBMS_JOB`, which is kept for backward compatibility.)

### Scheduler objects
| Object | Role |
|---|---|
| **Job** | The unit of work to run (what + when) |
| **Program** | Reusable "what": type + action + argument definitions |
| **Schedule** | Reusable "when": start/end, calendar expression |
| **Job class** | Groups jobs; assigns a **resource consumer group**, logging level and log retention |
| **Window** | A time interval that activates a **resource plan** (e.g., a nightly batch window); jobs can be scheduled *to* a window |
| **Chain** | A multi-step workflow with conditional rules between steps |
| **Credential** | Stored OS (or database) username/password used by external jobs |
| **File watcher** | Starts a job when a file arrives |

### Job types (`job_type`)
`PLSQL_BLOCK` (anonymous block text) · `STORED_PROCEDURE` · `EXECUTABLE` (OS program, incl. shell scripts) · `CHAIN` · and from 12c the script types `SQL_SCRIPT`, `EXTERNAL_SCRIPT`, `BACKUP_SCRIPT`. There are also **job styles**: `REGULAR`, **`LIGHTWEIGHT`** (for very large numbers of short jobs — minimal metadata/overhead) and in-memory styles in newer releases.

### Creating and managing a job
```sql
BEGIN
  DBMS_SCHEDULER.create_job (
    job_name            => 'NIGHTLY_PURGE',
    job_type            => 'STORED_PROCEDURE',
    job_action          => 'APP.PURGE_OLD_ROWS',
    number_of_arguments => 1,
    start_date          => SYSTIMESTAMP,
    repeat_interval     => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=0; BYSECOND=0',
    enabled             => FALSE,                          -- the default!
    comments            => 'Purge rows older than N days');

  DBMS_SCHEDULER.set_job_argument_value('NIGHTLY_PURGE', 1, '90');
  DBMS_SCHEDULER.set_attribute('NIGHTLY_PURGE', 'logging_level', DBMS_SCHEDULER.logging_full);
  DBMS_SCHEDULER.set_attribute('NIGHTLY_PURGE', 'max_run_duration', NUMTODSINTERVAL(2, 'HOUR'));
  DBMS_SCHEDULER.enable('NIGHTLY_PURGE');
END;
/
```
Other key subprograms: `RUN_JOB`, `STOP_JOB`, `DISABLE`, `DROP_JOB`, `COPY_JOB`, `CREATE_PROGRAM`, `CREATE_SCHEDULE`, `CREATE_JOB_CLASS`, `CREATE_WINDOW`, `CREATE_CREDENTIAL`, `CREATE_CHAIN`, `DEFINE_CHAIN_STEP`, `DEFINE_CHAIN_RULE`, `RUN_CHAIN`, `EVALUATE_CALENDAR_STRING`.

**Notable job attributes:** `auto_drop` (default TRUE — a completed one-off job disappears), `max_runs`, `max_failures`, `restartable`, `job_priority`, `max_run_duration`, `logging_level` (`LOGGING_OFF`, `LOGGING_FAILED_RUNS`, `LOGGING_RUNS`, `LOGGING_FULL`), `raise_events`, `schedule_limit`.

### Schedule types
| Type | Example |
|---|---|
| **Calendar expression** | `'FREQ=WEEKLY; BYDAY=MON,WED,FRI; BYHOUR=9; BYMINUTE=30'` |
| **PL/SQL expression** (date arithmetic) | `'SYSTIMESTAMP + INTERVAL ''1'' HOUR'` |
| **Event-based** | Starts on an event (queue message, file arrival via file watcher) |
| **Named schedule** | Reuse a `CREATE_SCHEDULE` object |
| **Window / window group** | Runs when the window opens |

**Calendaring keywords:** `FREQ` (`YEARLY`, `MONTHLY`, `WEEKLY`, `DAILY`, `HOURLY`, `MINUTELY`, `SECONDLY`), `INTERVAL`, `BYMONTH`, `BYWEEKNO`, `BYYEARDAY`, `BYMONTHDAY`, `BYDAY`, `BYHOUR`, `BYMINUTE`, `BYSECOND`, `BYDATE`, `BYSETPOS`, plus `INCLUDE`/`EXCLUDE`/`INTERSECT` to combine expressions.
Examples: last day of every month → `'FREQ=MONTHLY; BYMONTHDAY=-1'`; **last weekday** of each month → `'FREQ=MONTHLY; BYDAY=MON,TUE,WED,THU,FRI; BYSETPOS=-1'`.
Test an expression without creating a job: `DBMS_SCHEDULER.EVALUATE_CALENDAR_STRING(...)` returns the next run date.

### A simple chain
```sql
BEGIN
  DBMS_SCHEDULER.create_chain('ETL_CHAIN');
  DBMS_SCHEDULER.define_chain_step('ETL_CHAIN', 'STEP_EXTRACT', 'EXTRACT_PROG');   -- programs
  DBMS_SCHEDULER.define_chain_step('ETL_CHAIN', 'STEP_LOAD',    'LOAD_PROG');
  DBMS_SCHEDULER.define_chain_step('ETL_CHAIN', 'STEP_ALERT',   'ALERT_PROG');

  DBMS_SCHEDULER.define_chain_rule('ETL_CHAIN', 'TRUE',                    'START STEP_EXTRACT');
  DBMS_SCHEDULER.define_chain_rule('ETL_CHAIN', 'STEP_EXTRACT SUCCEEDED',  'START STEP_LOAD');
  DBMS_SCHEDULER.define_chain_rule('ETL_CHAIN', 'STEP_EXTRACT FAILED',     'START STEP_ALERT');
  DBMS_SCHEDULER.define_chain_rule('ETL_CHAIN', 'STEP_LOAD COMPLETED OR STEP_ALERT COMPLETED', 'END');
  DBMS_SCHEDULER.enable('ETL_CHAIN');

  DBMS_SCHEDULER.create_job('RUN_ETL', job_type => 'CHAIN', job_action => 'ETL_CHAIN',
                            repeat_interval => 'FREQ=DAILY; BYHOUR=1', enabled => TRUE);
END;
/
```
A chain is built from **steps** (each running a program/job/another chain) and **rules** (`condition` → `action`); the conditions test step outcomes such as `SUCCEEDED`, `FAILED`, `COMPLETED`.

### Monitoring
```sql
SELECT job_name, status, error#, actual_start_date, run_duration
FROM   user_scheduler_job_run_details
ORDER  BY log_date DESC;
```
| View | Shows |
|---|---|
| `*_SCHEDULER_JOBS` | Definitions and current `STATE` (`DISABLED`, `SCHEDULED`, `RUNNING`, `COMPLETED`, `FAILED`, `BROKEN`, …) |
| `*_SCHEDULER_RUNNING_JOBS` | Jobs executing now |
| `*_SCHEDULER_JOB_LOG` | Job events (subject to `logging_level`) |
| `*_SCHEDULER_JOB_RUN_DETAILS` | One row per **run**: status, error number, start, duration |
| `*_SCHEDULER_PROGRAMS`, `_SCHEDULES`, `_CHAINS`, `_WINDOWS`, `_JOB_CLASSES`, `_CREDENTIALS` | The corresponding objects |

### Privileges and prerequisites
- `CREATE JOB` (own schema) or `CREATE ANY JOB`; `CREATE EXTERNAL JOB` for jobs that run OS programs; **`MANAGE SCHEDULER`** for windows, job classes and global settings; `EXECUTE` on `DBMS_SCHEDULER`.
- External jobs run under a **credential** (`CREATE_CREDENTIAL`) — the OS account they execute as.
- The initialization parameter **`JOB_QUEUE_PROCESSES`** must be greater than 0 for jobs to run.
- Use **`TIMESTAMP WITH TIME ZONE` with a region name** (e.g., `'Asia/Kolkata'`) as the start date so daylight-saving rules are handled correctly; a fixed offset won't adjust.

### Scheduler vs. `DBMS_JOB` — extra exam points
| | `DBMS_JOB` | `DBMS_SCHEDULER` |
|---|---|---|
| Transactional? | **Yes** — a submitted job exists only after `COMMIT`; `ROLLBACK` removes it | **No** — calls take effect immediately (they commit implicitly) |
| Runs OS programs / chains / windows / event triggers | No | Yes |
| Rich run history | Minimal | `*_JOB_RUN_DETAILS`, `*_JOB_LOG` |

### Important points to remember
- **Jobs are created disabled by default** (`enabled => FALSE`) — a job that "never runs" is very often just not enabled (or `JOB_QUEUE_PROCESSES` is 0).
- **`auto_drop` defaults to TRUE**: one-time jobs (or jobs that hit `end_date`/`max_runs`) are removed after completing; set it to FALSE if you want to inspect them.
- Separating **program / schedule / job** allows reuse; a job can reference a named program and/or schedule instead of inline values.
- `RUN_JOB(job, use_current_session => TRUE)` runs the job **synchronously in your session** (errors come straight back — handy for testing); with `FALSE` it runs **asynchronously** as a normal scheduler job.
- Chains give **conditional workflows**; job classes and windows tie jobs to **Resource Manager** plans.
- Set an appropriate **`logging_level`** — the default may not keep the run detail you want when troubleshooting.

### Exercise Questions

**Q1.** You create a job with `CREATE_JOB` and a valid `repeat_interval`, yet it never runs and shows state `DISABLED`. Why?
> **A:** `CREATE_JOB` creates the job **disabled** unless `enabled => TRUE` is specified. Enable it with `DBMS_SCHEDULER.ENABLE('job_name')`. (If it is enabled but still doesn't run, check that `JOB_QUEUE_PROCESSES` is greater than 0 and that the job's schedule/start date is what you expect.)

**Q2.** What is the difference between `RUN_JOB(..., use_current_session => TRUE)` and `FALSE`?
> **A:** With `TRUE`, the job runs **synchronously inside the calling session**, and any error is returned directly to the caller — useful for testing and debugging. With `FALSE`, the request is handed to the scheduler and the job runs **asynchronously** in a scheduler job slave, exactly as it would on its normal schedule, and the call returns without waiting.

**Q3.** Write a calendar expression for "the last weekday of every month at 18:00."
> **A:** `FREQ=MONTHLY; BYDAY=MON,TUE,WED,THU,FRI; BYSETPOS=-1; BYHOUR=18; BYMINUTE=0; BYSECOND=0`. `BYDAY` produces every weekday in the month, and `BYSETPOS=-1` then picks the **last** of that set. (`EVALUATE_CALENDAR_STRING` can be used to verify the next few dates.)

**Q4.** A nightly job must run a shell script on the database server. What job type and supporting objects/privileges are involved?
> **A:** Use an **`EXECUTABLE`** job (or `EXTERNAL_SCRIPT` in 12c+) whose action is the script/program path, run under a **credential** created with `CREATE_CREDENTIAL` (the OS account it runs as), and the owner needs the **`CREATE EXTERNAL JOB`** privilege. `DBMS_JOB` cannot do this because it can only execute PL/SQL.

**Q5.** A developer calls `DBMS_JOB.SUBMIT` in a session and then `ROLLBACK`s; another developer calls `DBMS_SCHEDULER.CREATE_JOB` and then `ROLLBACK`s. What happens in each case?
> **A:** The `DBMS_JOB` submission is **transactional** — the rollback removes the job, so it never runs. The Scheduler operation is **not** undone by the rollback: `DBMS_SCHEDULER` calls take effect immediately (they commit implicitly), so the job exists and will run.

---

## 5. `DBMS_UTILITY`

### What it is
A collection of **general-purpose utility subprograms** — error/call-stack formatting, timing, schema compilation, name handling and environment information. Exams focus on the **diagnostic** and **timing** functions and a few classic gotchas.

### Error and call-stack diagnostics
| Function | Returns |
|---|---|
| `FORMAT_ERROR_STACK` | The **error message stack** (the current error and any chained errors), e.g. `ORA-06512` lines; not truncated as early as `SQLERRM` |
| `FORMAT_ERROR_BACKTRACE` (10gR2+) | **Where the error occurred** — the program units and **line numbers** from the point of the error up to the handler that caught it |
| `FORMAT_CALL_STACK` | The **current call stack** (who called whom, with line numbers) at the point it is called — no error required |

```sql
BEGIN
  process_order(42);
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.put_line('ERROR STACK : ' || DBMS_UTILITY.format_error_stack);
    DBMS_OUTPUT.put_line('BACKTRACE   : ' || DBMS_UTILITY.format_error_backtrace);
    DBMS_OUTPUT.put_line('CALL STACK  : ' || DBMS_UTILITY.format_call_stack);
    RAISE;
END;
/
```
Each returns a `VARCHAR2` limited to roughly 2000 bytes (long stacks can be cut short — one reason the structured `UTL_CALL_STACK` package was introduced in 12c). `SQLERRM` gives only the message text; it has **no line-number information**.

**Backtrace behavior:** a bare `RAISE;` in a handler **preserves** the original backtrace, but raising a *new* exception (or calling `RAISE_APPLICATION_ERROR`) starts a fresh one. So capture `FORMAT_ERROR_BACKTRACE` in the **first/innermost handler** (and log it) before re-raising something different.

### Timing
```sql
DECLARE
  v_start PLS_INTEGER := DBMS_UTILITY.get_time;
BEGIN
  heavy_procedure;
  DBMS_OUTPUT.put_line('Elapsed: ' || (DBMS_UTILITY.get_time - v_start) / 100 || ' s');
END;
/
```
- `GET_TIME` returns elapsed time in **hundredths of a second** from an arbitrary starting point — meaningful **only as a difference** between two calls (the counter can wrap, so never treat it as a clock time).
- `GET_CPU_TIME` returns CPU time consumed, also in hundredths of a second.

### Other frequently tested subprograms
| Subprogram | Purpose / gotcha |
|---|---|
| `COMPILE_SCHEMA(schema, compile_all, reuse_settings)` | Recompiles **procedures, functions, packages and views** in a schema. `compile_all => TRUE` (default) compiles everything; **`FALSE` compiles only invalid objects**. `reuse_settings => TRUE` keeps each object's existing compiler settings. Runs with the **caller's** privileges. |
| `COMMA_TO_TABLE` / `TABLE_TO_COMMA` | Convert a comma-delimited list ⇄ a PL/SQL table (`UNCL_ARRAY`/`LNAME_ARRAY`). **Gotcha:** each token must be a **valid identifier** — a list like `'1,2,3'` or `'a b,c'` fails. For general string splitting use a `REGEXP_SUBSTR`-based approach instead. |
| `DB_VERSION(version OUT, compatibility OUT)` | Database version and `COMPATIBLE` setting |
| `PORT_STRING` | Platform/version identifier string |
| `GET_PARAMETER_VALUE` | Read an initialization parameter |
| `CURRENT_INSTANCE`, `ACTIVE_INSTANCES`, `IS_CLUSTER_DATABASE` | RAC-related information |
| `GET_HASH_VALUE(name, base, hash_size)` | Numeric hash of a string (for bucketing; **not** a cryptographic hash and not unique) |
| `NAME_RESOLVE`, `NAME_TOKENIZE`, `CANONICALIZE` | Resolve / split / normalize object names |
| `EXEC_DDL_STATEMENT(parse_string)` | Runs a DDL statement (a legacy equivalent of `EXECUTE IMMEDIATE` for DDL) |
| `INVALIDATE`, `VALIDATE` | Invalidate an object by id / attempt to make an invalid object valid |
| `ANALYZE_SCHEMA` and related | **Legacy/deprecated** statistics gathering — use `DBMS_STATS` instead |

### Important points to remember
- **Three diagnostics, three questions:** *What error?* → `FORMAT_ERROR_STACK`. *Where did it happen?* → `FORMAT_ERROR_BACKTRACE`. *How did we get here (no error needed)?* → `FORMAT_CALL_STACK`.
- `FORMAT_ERROR_BACKTRACE` is only meaningful **inside an exception handler**.
- `GET_TIME` units are **hundredths of a second**; use only differences.
- `COMPILE_SCHEMA` with `compile_all => FALSE` is the quick "fix invalid objects" call; for parallel recompilation of large schemas, `UTL_RECOMP` is the alternative.
- `COMMA_TO_TABLE` requires **identifier-shaped** tokens — a classic surprise.
- Statistics gathering belongs to **`DBMS_STATS`**, not `DBMS_UTILITY.ANALYZE_SCHEMA`.

### Exercise Questions

**Q1.** A procedure catches `WHEN OTHERS`, logs `SQLERRM`, and support still can't tell which line failed. Which `DBMS_UTILITY` function adds that information, and where must it be called?
> **A:** **`FORMAT_ERROR_BACKTRACE`**, called **inside the exception handler**, returns the chain of program units and line numbers from where the error was raised to the handler. `SQLERRM` contains only the message text. Log the backtrace (and `FORMAT_ERROR_STACK`) in the handler — preferably the first/innermost one, before any new exception is raised.

**Q2.** A handler executes `RAISE_APPLICATION_ERROR(-20001, 'Order failed');` and the outer handler's backtrace no longer points to the original failing line. Why?
> **A:** Raising a **new** exception (here via `RAISE_APPLICATION_ERROR`) starts a new error and backtrace; the original raise point is no longer reported by the outer handler. A bare `RAISE;` preserves the original backtrace. To keep the details, capture and log `FORMAT_ERROR_BACKTRACE` in the inner handler before raising the new error (or re-raise the original with `RAISE;`).

**Q3.** `DBMS_UTILITY.GET_TIME` returns 172345 at the start of a task and 172912 at the end. How long did it take, and why can't you convert `GET_TIME` to a date?
> **A:** 172912 − 172345 = 567 hundredths of a second, i.e. **5.67 seconds**. `GET_TIME` counts hundredths of a second from an arbitrary starting point (and can wrap), so its absolute value has no calendar meaning — only differences between two readings are valid.

**Q4.** `DBMS_UTILITY.COMMA_TO_TABLE('10,20,30', n, tab)` raises an error. What is wrong and what would you do instead?
> **A:** `COMMA_TO_TABLE` parses each element as a database **identifier**, and `10`, `20`, `30` aren't valid identifiers (they start with a digit), so it fails. Use a different splitting technique — e.g., a SQL/PLSQL loop with `REGEXP_SUBSTR` (or `INSTR`/`SUBSTR`), or a helper that splits arbitrary delimited strings — rather than `COMMA_TO_TABLE`.

**Q5.** After a deployment many objects in a schema are `INVALID`. What single call recompiles only those, and what does the `compile_all` parameter control?
> **A:** `DBMS_UTILITY.COMPILE_SCHEMA(schema => 'APP', compile_all => FALSE);` `compile_all => TRUE` (the default) recompiles **every** eligible object in the schema, whereas `FALSE` recompiles **only the invalid ones**, which is faster and less disruptive.

---

## Choosing the Right Package — Quick Reference

| Need | Package |
|---|---|
| Build SQL at run time where the number/type of columns or binds is unknown; parse once/execute many | `DBMS_SQL` |
| Simple dynamic SQL/DDL/DML | `EXECUTE IMMEDIATE` (not a package, but the default choice) |
| Read/write text files on the **server** | `UTL_FILE` (with `DIRECTORY` objects) |
| Extract `CREATE` scripts / grants / XML definitions of objects | `DBMS_METADATA` |
| Run code on a schedule, on events, as chains, or as OS jobs | `DBMS_SCHEDULER` |
| Error stack / backtrace / call stack, elapsed time, recompile a schema | `DBMS_UTILITY` |

## Quick Cross-Topic Summary Table

| Package | Core idea | Classic exam traps |
|---|---|---|
| `DBMS_SQL` | Explicit cursor API: open → parse → bind → define → execute → fetch → column_value → close | DDL runs at **PARSE**; cursor leaks → `ORA-01000`; use only when shape is unknown at compile time; `TO_REFCURSOR` interop |
| `UTL_FILE` | Server-side file I/O through `DIRECTORY` objects | Directory name case; files are on the **server**; `NO_DATA_FOUND` at EOF; `max_linesize` 1024 default / 32767 max; `UTL_FILE_DIR` deprecated (12.2); always `FCLOSE` |
| `DBMS_METADATA` | Extract DDL/XML of objects (CLOB) | `SET LONG`; `SQLTERMINATOR` default FALSE; dependent DDL & grants need separate calls; `SELECT_CATALOG_ROLE` for other schemas |
| `DBMS_SCHEDULER` | Jobs, programs, schedules, chains, windows, classes | Created **disabled** by default; `auto_drop` TRUE; not transactional (unlike `DBMS_JOB`); credentials for external jobs; `BYSETPOS` |
| `DBMS_UTILITY` | Diagnostics, timing, compile, misc | `FORMAT_ERROR_BACKTRACE` inside handler; `GET_TIME` = hundredths of a second (differences only); `COMMA_TO_TABLE` needs identifiers; `COMPILE_SCHEMA` `compile_all` |