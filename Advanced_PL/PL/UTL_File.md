# UTL_FILE — Concept, Practical Scenario & Directory Security

---

## 1. The Simple Idea

**Analogy:** The database is a locked office building. **UTL_FILE is a clerk inside the building** who can read and write paper files for you, with three rules:

1. **The clerk only works with the building's own filing cabinets**, meaning the **database server's disk**. They can never reach your laptop, even if you're the one giving the order from SQL*Plus.
2. **The clerk only opens cabinets you hold a key for.** The cabinet is a **DIRECTORY object**, and the key is a **GRANT READ/WRITE** on it.
3. **Every file must be opened, used, then closed.** Leave it open and things get lost or locked.

### Where you use it
Whenever PL/SQL needs to exchange data with files on the server:
- Exporting reports or CSV files for another system (a bank, payroll, auditors)
- Reading inbound feed files from partners
- Writing custom log or error files from batch jobs
- Archiving, renaming, copying or cleaning up those files

### Why it matters
Batch integrations in ERP systems (Oracle Fusion / EBS included) often follow a **"drop a file → process it → archive it"** cycle. UTL_FILE is how PL/SQL takes part in that cycle without any external program.

---

## 2. Function Map

| Action | Function |
|---|---|
| Does the file exist? How big is it? | `FGETATTR` |
| Open the file for read, write or append | `FOPEN` (`'r'`, `'w'`, `'a'`; byte modes `'rb'`, `'wb'`, `'ab'`) |
| Read one line | `GET_LINE` (raises `NO_DATA_FOUND` at end of file) |
| Write text | `PUT`, `PUT_LINE`, `NEW_LINE`, `PUTF` |
| Force buffered output to disk | `FFLUSH` |
| Is the handle open? / Close it | `IS_OPEN`, `FCLOSE`, `FCLOSE_ALL` |
| Copy, move or delete files | `FCOPY`, `FRENAME`, `FREMOVE` |
| Byte I/O / file pointer | `GET_RAW`, `PUT_RAW`, `FSEEK`, `FGETPOS` |

`max_linesize` in `FOPEN` defaults to 1024 bytes and can be set from 1 to 32767.

---

## 3. Why Create a Directory and Grant It?

### Why a directory object at all?
The database runs on the server under one OS account (usually `oracle`), and that account can reach many folders, **including the database's own datafiles, control files and passwords**.

If any user could write `UTL_FILE.fopen('/u01/oradata', 'system01.dbf', 'w')`, a junior developer could wipe the whole database or read sensitive files. So Oracle doesn't let you use raw paths. It uses a two-step control:

1. **CREATE DIRECTORY** puts a labelled, approved cabinet in the building: *"DATA_DIR means `/u01/app/hr/inbound`, and only that folder."*
2. **GRANT READ/WRITE** hands out keys: *"HR may read and write DATA_DIR. Nobody else may."*

> The **directory object** decides *which folder* can be used. The **grant** decides *who* can use it.

**Bonus:** your code only says `'DATA_DIR'`. If the server folder moves to `/u02/...`, the DBA re-points the directory object and **your code doesn't change**.

### Is this only the DBA's job?
**Usually yes.** Creating a directory needs the **`CREATE ANY DIRECTORY`** system privilege. It's powerful (it can point at any server folder), so in real projects only DBAs have it.

| Who | Does what |
|---|---|
| **Developer** | Requests it: "Need `DATA_DIR` → `/u01/app/hr/inbound`, READ/WRITE for schema `HR`" |
| **OS / Unix admin** | Creates the actual folder and gives the `oracle` OS user permission on it |
| **DBA** | Runs `CREATE DIRECTORY` and `GRANT READ, WRITE ... TO hr` |
| **Developer** | Writes the UTL_FILE code using `'DATA_DIR'` |

Notes:
- Whoever creates a directory automatically gets READ/WRITE on it and can grant it to others.
- In your own practice database (for example local Oracle XE), you are the DBA, so you run all of it yourself.
- Grant **directly to the schema that owns the procedure**, not through a role. Role privileges don't work inside definer's-rights stored code. That's why things can work in SQL*Plus but fail inside a procedure.
- `UTL_FILE_DIR` (the old init parameter) is deprecated since 12.2. Use directory objects.

---

## 4. Real Scenario — Daily Salary-Update File

### The question

> Every morning the HR vendor drops `salary_updates.csv` into the server folder mapped to `DATA_DIR`:
> ```
> EMP_ID,NEW_SALARY
> 100,25000
> 101,abc
> 999,8000
> 102,-500
> 103,9500
> ```
> Write a procedure `process_salary_file` that:
> 1. Checks the file exists. If not, appends a message to `error_log.txt` and stops.
> 2. Takes a **backup copy** into `ARCHIVE_DIR` before touching anything.
> 3. Reads the file line by line, skipping the header.
> 4. Updates `employees.salary` for valid lines and writes each success into a **new report file**.
> 5. **Appends** every bad line (non-numeric, negative salary, unknown employee) to `error_log.txt`.
> 6. Flushes output every 100 lines.
> 7. Writes a summary at the end of the report.
> 8. On success, commits, **moves** the input file to `ARCHIVE_DIR` as `processed_<timestamp>_...`, and **deletes** the backup.
> 9. On any failure, rolls back, closes all files, and raises a clear error.

### One-time setup (run as DBA)

```sql
CREATE OR REPLACE DIRECTORY data_dir    AS '/u01/app/hr/inbound';
CREATE OR REPLACE DIRECTORY archive_dir AS '/u01/app/hr/archive';

GRANT READ, WRITE ON DIRECTORY data_dir    TO hr;   -- grant directly, not through a role
GRANT READ, WRITE ON DIRECTORY archive_dir TO hr;
GRANT EXECUTE ON UTL_FILE TO hr;
```

### Solution

```sql
CREATE OR REPLACE PROCEDURE process_salary_file (
  p_file IN VARCHAR2 DEFAULT 'salary_updates.csv'
) IS
  c_data  CONSTANT VARCHAR2(30) := 'DATA_DIR';      -- directory names in UPPER CASE
  c_arch  CONSTANT VARCHAR2(30) := 'ARCHIVE_DIR';

  f_in    UTL_FILE.file_type;                       -- input file handle
  f_rep   UTL_FILE.file_type;                       -- report file handle
  f_err   UTL_FILE.file_type;                       -- error log handle

  v_line    VARCHAR2(32767);
  v_exists  BOOLEAN;
  v_len     NUMBER;
  v_blk     BINARY_INTEGER;

  v_emp_id  NUMBER;
  v_sal     NUMBER;
  v_line_no PLS_INTEGER := 0;
  v_ok      PLS_INTEGER := 0;
  v_bad     PLS_INTEGER := 0;
  v_stamp   VARCHAR2(20) := TO_CHAR(SYSDATE, 'YYYYMMDD_HH24MISS');
  v_backup  VARCHAR2(200);
BEGIN
  ----------------------------------------------------------------
  -- 1. Does the file exist?                              FGETATTR
  ----------------------------------------------------------------
  UTL_FILE.fgetattr(c_data, p_file, v_exists, v_len, v_blk);

  IF NOT v_exists THEN
    f_err := UTL_FILE.fopen(c_data, 'error_log.txt', 'a');     -- 'a' = append
    UTL_FILE.putf(f_err, '%s  File %s not found - nothing processed\n', v_stamp, p_file);
    UTL_FILE.fclose(f_err);
    RETURN;
  END IF;

  ----------------------------------------------------------------
  -- 2. Back up the original before processing               FCOPY
  ----------------------------------------------------------------
  v_backup := 'backup_' || v_stamp || '_' || p_file;
  UTL_FILE.fcopy(c_data, p_file, c_arch, v_backup);

  ----------------------------------------------------------------
  -- 3. Open all three files                                 FOPEN
  ----------------------------------------------------------------
  f_in  := UTL_FILE.fopen(c_data, p_file,                                'r', 32767);
  f_rep := UTL_FILE.fopen(c_data, 'salary_report_' || v_stamp || '.txt', 'w', 32767);
  f_err := UTL_FILE.fopen(c_data, 'error_log.txt',                      'a', 32767);

  -- Report header                                PUT_LINE / PUTF / NEW_LINE
  UTL_FILE.put_line(f_rep, 'SALARY UPDATE REPORT');
  UTL_FILE.putf(f_rep, 'Source: %s (%s bytes)\n', p_file, TO_CHAR(v_len));
  UTL_FILE.new_line(f_rep);

  ----------------------------------------------------------------
  -- 4. Read line by line                                 GET_LINE
  ----------------------------------------------------------------
  LOOP
    BEGIN
      UTL_FILE.get_line(f_in, v_line);
    EXCEPTION
      WHEN NO_DATA_FOUND THEN EXIT;              -- end of file
    END;

    v_line_no := v_line_no + 1;
    CONTINUE WHEN v_line_no = 1;                 -- skip the header row

    BEGIN
      v_emp_id := TO_NUMBER(TRIM(REGEXP_SUBSTR(v_line, '[^,]+', 1, 1)));
      v_sal    := TO_NUMBER(TRIM(REGEXP_SUBSTR(v_line, '[^,]+', 1, 2)));

      IF v_sal IS NULL OR v_sal <= 0 THEN
        RAISE VALUE_ERROR;                       -- treat as bad data
      END IF;

      UPDATE employees SET salary = v_sal WHERE employee_id = v_emp_id;

      IF SQL%ROWCOUNT = 0 THEN                   -- unknown employee
        v_bad := v_bad + 1;
        UTL_FILE.putf(f_err, '%s  Line %s: employee %s not found\n',
                      v_stamp, TO_CHAR(v_line_no), TO_CHAR(v_emp_id));
      ELSE
        v_ok := v_ok + 1;
        UTL_FILE.put(f_rep, RPAD(v_emp_id, 10));              -- PUT: no line break
        UTL_FILE.put_line(f_rep, 'salary set to ' || v_sal);  -- finishes the line
      END IF;

    EXCEPTION
      WHEN VALUE_ERROR OR INVALID_NUMBER THEN    -- 'abc' or a negative salary
        v_bad := v_bad + 1;
        UTL_FILE.putf(f_err, '%s  Line %s: bad data -> %s\n',
                      v_stamp, TO_CHAR(v_line_no), v_line);
    END;

    -- 6. Push buffered output to disk periodically       FFLUSH
    IF MOD(v_line_no, 100) = 0 THEN
      UTL_FILE.fflush(f_rep);
      UTL_FILE.fflush(f_err);
    END IF;
  END LOOP;

  ----------------------------------------------------------------
  -- 7. Summary, then close everything                      FCLOSE
  ----------------------------------------------------------------
  UTL_FILE.new_line(f_rep);
  UTL_FILE.putf(f_rep, 'Updated: %s    Errors: %s\n', TO_CHAR(v_ok), TO_CHAR(v_bad));

  UTL_FILE.fclose(f_in);
  UTL_FILE.fclose(f_rep);
  UTL_FILE.fclose(f_err);

  COMMIT;

  ----------------------------------------------------------------
  -- 8. Archive the input and drop the backup       FRENAME / FREMOVE
  ----------------------------------------------------------------
  UTL_FILE.frename(c_data, p_file,
                   c_arch, 'processed_' || v_stamp || '_' || p_file,
                   TRUE);                                    -- overwrite if it exists
  UTL_FILE.fremove(c_arch, v_backup);

EXCEPTION
  ----------------------------------------------------------------
  -- 9. Failure: undo DB changes, close any open files   FCLOSE_ALL
  --    (the backup stays in ARCHIVE_DIR for investigation)
  ----------------------------------------------------------------
  WHEN UTL_FILE.invalid_path THEN
    ROLLBACK;  UTL_FILE.fclose_all;
    RAISE_APPLICATION_ERROR(-20001, 'Invalid directory or no privilege on it');
  WHEN UTL_FILE.invalid_operation THEN
    ROLLBACK;  UTL_FILE.fclose_all;
    RAISE_APPLICATION_ERROR(-20002, 'File could not be opened or operated on');
  WHEN UTL_FILE.write_error THEN
    ROLLBACK;  UTL_FILE.fclose_all;
    RAISE_APPLICATION_ERROR(-20003, 'OS error while writing a file');
  WHEN OTHERS THEN
    ROLLBACK;
    IF UTL_FILE.is_open(f_in) THEN UTL_FILE.fclose(f_in); END IF;   -- IS_OPEN demo
    UTL_FILE.fclose_all;
    RAISE;
END process_salary_file;
/
```

### Run it

```sql
BEGIN
  process_salary_file;
END;
/
```

### What you'll find on the server

**`DATA_DIR/salary_report_20261021_220500.txt`**
```
SALARY UPDATE REPORT
Source: salary_updates.csv (58 bytes)

100       salary set to 25000
103       salary set to 9500

Updated: 2    Errors: 3
```

**`DATA_DIR/error_log.txt`** (appended, so earlier days' entries are kept)
```
20261021_220500  Line 3: bad data -> 101,abc
20261021_220500  Line 4: employee 999 not found
20261021_220500  Line 5: bad data -> 102,-500
```

**`ARCHIVE_DIR/`** contains `processed_20261021_220500_salary_updates.csv`, and the backup copy has been removed.

---

## 5. Practical Lessons from the Scenario

- **`'w'` vs `'a'`:** the report uses `'w'` because each run produces a fresh file. The error log uses `'a'` so history accumulates.
- **End of file is an exception**, not a return value. That's why `GET_LINE` sits in its own small `BEGIN … EXCEPTION` block.
- **Two kinds of error handling:** a bad *line* is logged and the loop continues. A bad *file or directory* stops everything, with rollback and `FCLOSE_ALL`.
- **Back up first, then archive or remove.** If the run fails, the backup is still there.
- **`'DATA_DIR'` must be in upper case.** Writing `'data_dir'` is the classic cause of `INVALID_PATH`.
- **The files live on the server.** Running this from your laptop puts nothing on your laptop.