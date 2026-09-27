# Oracle PL/SQL — External Programs, Scheduling & LOB Management
### Exam Study Guide

Covers: Calling C and Java from PL/SQL · Creating External Libraries & Secure External Procedures · `DBMS_SCHEDULER` vs `DBMS_JOB` · LOB Management · Internal vs External LOBs (BLOB/CLOB/BFILE) · `DBMS_LOB` for Performance · SecureFile LOBs (Compression, Encryption, Deduplication)

---

## 1. Calling C and Java Programs from PL/SQL

### What it is
PL/SQL can invoke logic written in **other languages** when native PL/SQL/SQL isn't suitable — typically for CPU-intensive algorithms, existing legacy code, or OS-level operations. Oracle supports two distinct mechanisms:

| Mechanism | Language | Runs where |
|---|---|---|
| **External Procedure (`extproc`)** | C (or any language producing a shared library) | In a **separate OS process** (the `extproc` agent), outside the database server process |
| **Java Stored Procedure** | Java | **Inside** the database, in Oracle's embedded JVM (Oracle JVM / OJVM) |

### Calling a C function
1. Compile your C code into a **shared library** (`.so` on Linux/Unix, `.dll` on Windows).
2. Register the library with Oracle:
```sql
CREATE OR REPLACE LIBRARY c_utils_lib AS '/home/oracle/lib/c_utils.so';
```
3. Create a PL/SQL wrapper that maps to the C function:
```sql
CREATE OR REPLACE FUNCTION celsius_to_fahrenheit (p_celsius IN NUMBER)
RETURN NUMBER
IS
  LANGUAGE C
  LIBRARY c_utils_lib
  NAME "c2f"
  PARAMETERS (p_celsius DOUBLE, RETURN DOUBLE);
```
4. Call it exactly like any other PL/SQL function: `SELECT celsius_to_fahrenheit(100) FROM dual;`

Behind the scenes, when this function is called, the database contacts a small out-of-process **agent (`extproc`)**, which dynamically loads the shared library and executes the requested function, then returns the result back across the connection.

### Calling Java
1. Write and compile a Java class.
2. Load it into the database schema with **`loadjava`** (command-line tool) or `CREATE JAVA SOURCE/CLASS`:
```
loadjava -user hr/hr@orcl MyUtil.class
```
3. Publish it with a PL/SQL **call specification**:
```sql
CREATE OR REPLACE FUNCTION java_reverse_string (p_input IN VARCHAR2)
RETURN VARCHAR2
AS LANGUAGE JAVA
NAME 'MyUtil.reverse(java.lang.String) return java.lang.String';
```
4. Call it like any PL/SQL function.

### Important points to remember
- **C/external procedures run OUTSIDE the database process** (separate OS process) — a crash in the C code cannot directly crash the Oracle instance, but it *is* a security exposure (arbitrary native code execution) if not locked down (see Section 2).
- **Java stored procedures run INSIDE the database's embedded JVM** — no separate OS process, generally simpler to deploy and more portable, but Java code shares fate more closely with the session (can still be resource-managed/interrupted, but doesn't have the "separate process" isolation of C).
- The `PARAMETERS` clause in a C call spec explicitly maps PL/SQL datatypes to C datatypes — this mapping is **not automatic** and is one of the trickiest parts of writing external procedure specs.
- Java call specs require exact matching of the Java method signature (fully qualified class/method/parameter/return types).

### Exercise Questions

**Q1.** Why might a C-based external procedure be preferred over a Java stored procedure for a CPU-bound legacy algorithm?
> **A:** If the algorithm already exists as compiled/tested C code, wrapping it via `extproc` avoids a rewrite in Java. C external procedures also run in a **separate OS process**, so a misbehaving or crashing routine cannot crash the Oracle instance itself (though it can still crash the agent process and fail the call) — useful when integrating with less-trusted or unfamiliar native code, since it isolates faults away from the database process.

**Q2.** What two registration steps are required before a PL/SQL wrapper can call a C shared-library function, that are *not* required for calling a PL/SQL-native function?
> **A:** (1) `CREATE LIBRARY` to register the shared library's file path with the database as a named schema object, and (2) writing the function's call specification with `LANGUAGE C LIBRARY ... NAME ... PARAMETERS (...)`, explicitly declaring how each PL/SQL parameter maps to its C-side datatype — there is no automatic type mapping the way there is between PL/SQL and SQL.

**Q3.** True or False: A Java stored procedure runs in a separate operating-system process from the Oracle database instance, the same way a C external procedure does.
> **A:** **False.** Java stored procedures execute inside the Oracle-embedded JVM, which runs **within** the database process space — not as an independent OS process. Only C (and other native-library-based) external procedures use the separate `extproc` agent process model.

---

## 2. Creating External Libraries & Secure External Procedures

### `CREATE LIBRARY` — registering the shared object
```sql
CREATE OR REPLACE LIBRARY c_utils_lib AS '/home/oracle/lib/c_utils.so';
```
This is a schema object (needs `CREATE LIBRARY` system privilege), and other users need `EXECUTE` privilege on it to build call specs referencing it.

### The security problem
An external procedure, by definition, executes **arbitrary native code** in a process outside the database's normal privilege boundary. Historically, the `extproc` agent was spawned directly by the **TNS Listener**, meaning a network-level attacker who could reach the listener could potentially cause it to execute unintended shared libraries/commands with the privileges of the listener's OS account — a serious, well-documented attack vector.

### Securing external procedures — key hardening steps
| Step | Purpose |
|---|---|
| **Use the default (non-listener) configuration** where the `extproc` agent is spawned **directly by the database** rather than the listener | Eliminates the network-facing attack surface entirely — the recommended, most secure setup |
| If listener-based `extproc` is unavoidable, run a **dedicated listener** solely for `extproc`, with **only an `IPC` protocol address** (no `TCP`) | Prevents remote/network access to the `extproc` service; IPC only allows local, same-machine communication |
| Set `EXTPROC_DLLS=ONLY:<allowed_lib1>:<allowed_lib2>:...` in `extproc.ora` (whitelist) rather than `EXTPROC_DLLS=ANY` | Restricts which shared libraries the agent is permitted to load — an explicit **whitelist**, closing off arbitrary code execution |
| Run the `extproc` listener/agent as an **unprivileged OS account** (e.g. `nobody`), never as the database owner or a privileged account | Limits the "blast radius" if the agent is compromised — principle of least privilege at the OS level |
| Never leave `EXTPROC_DLLS` unset in production | An unset value defaults to allowing execution of anything found in the standard `bin`/`lib` directory — effectively unrestricted |

### Example of a *secure* `tnsnames.ora`/`listener.ora` pairing (IPC-only, whitelisted)
```
# listener.ora (dedicated EXTPROC listener)
EXTPLSNR =
  (DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=ipc)(KEY=extproc))))
SID_LIST_EXTPLSNR =
  (SID_LIST=(SID_DESC=(SID_NAME=PLSExtProc)(PROGRAM=extproc)
             (ENVS="EXTPROC_DLLS=ONLY:/home/oracle/lib/c_utils.so")))

# tnsnames.ora
EXTPROC_CONNECTION_DATA =
  (DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=ipc)(KEY=extproc)))
   (CONNECT_DATA=(SID=PLSExtProc)))
```

### Important points to remember
- The **default configuration** (no listener involvement at all — the database spawns `extproc` directly) is Oracle's officially recommended, most secure option when no special network routing is needed.
- If you must use listener-spawned `extproc`, the golden rules are: **dedicated listener, IPC protocol only, whitelist the allowed libraries explicitly, unprivileged OS user**.
- `PROTOCOL=tcp` on an `extproc`-dedicated listener is a red flag in security audits/exam scenarios — it exposes the agent to network reachability it normally shouldn't have.
- `EXTPROC_DLLS=ANY` (or an unset value) is a classic "finding" in security audits — it means *any* library on the filesystem accessible to that OS account can be loaded and its functions called.

### Exercise Questions

**Q1.** Why is `EXTPROC_DLLS=ANY` considered a security risk, and what is the recommended alternative?
> **A:** With `ANY`, the `extproc` agent will load and execute functions from **any** shared library reachable by the OS account running it — meaning any code path that can invoke an external procedure call (potentially including an injected/malicious call) can execute arbitrary native code. The recommended alternative is `EXTPROC_DLLS=ONLY:<lib1>:<lib2>:...`, an explicit whitelist restricting execution to only the specific, vetted libraries the application actually needs.

**Q2.** Why should a dedicated `extproc` listener use only the `IPC` protocol rather than `TCP`?
> **A:** `IPC` (Inter-Process Communication) only allows connections from processes on the **same machine**, whereas `TCP` opens the listener to network-based connections, potentially from remote hosts. Since `extproc` execution is inherently high-risk (arbitrary native code execution), restricting it to local-machine-only access via IPC removes the remote attack surface entirely; if `TCP` truly must be used, it should at minimum use a distinct port and strict node-checking/firewalling.

**Q3.** A DBA finds that the `extproc` agent is configured to run under the database owner's OS account rather than an unprivileged account like `nobody`. Why is this a finding in a security review?
> **A:** Because the `extproc` agent executes native, potentially untrusted code, running it under a privileged account (like the database owner) means a compromised or malicious library call inherits **that account's full OS privileges** — including access to database files, configuration, and other sensitive resources. Running the agent as an unprivileged account limits what a compromised external procedure call could actually do, following the principle of least privilege.

---

## 3. `DBMS_SCHEDULER` vs. `DBMS_JOB`

### What they are
Both packages schedule PL/SQL (and other) code to run automatically at specified times/intervals — but `DBMS_SCHEDULER` (introduced 10g) is the modern, far more capable replacement for the legacy `DBMS_JOB` (which Oracle still supports for backward compatibility only).

### Core comparison
| Feature | `DBMS_JOB` (legacy) | `DBMS_SCHEDULER` (modern) |
|---|---|---|
| What can be scheduled | PL/SQL blocks only | PL/SQL blocks, stored procedures, **native OS executables/shell scripts**, chains of steps |
| Scheduling expressiveness | Simple `sysdate + interval` numeric expression | Rich **calendaring syntax** (`FREQ=DAILY; BYHOUR=6`, etc.), reusable named **schedules** |
| Reusable components | No — job defines everything inline | Yes — separate, reusable **`PROGRAM`**, **`SCHEDULE`**, and **`JOB`** objects that can be mixed and matched |
| Job chains / dependencies | Not supported | **Job chains** — multi-step workflows with dependency/condition logic between steps |
| Resource management integration | No | Can be tied to **Resource Manager consumer groups**, job classes, windows |
| Prioritization / windows | No | **Scheduler windows** (e.g., "maintenance window") and **job classes** with priorities |
| Run on a remote host | No | Yes — **remote external jobs** via a lightweight scheduler agent on another machine |
| File-based triggering | No | **File watchers** — a job can fire when a file arrives in a directory |
| Privilege model | Simple; runs as job owner | More granular — `CREATE JOB`, `MANAGE SCHEDULER`, per-object privileges |
| Logging | Minimal (`USER_JOBS`) | Rich logging/history views (`*_SCHEDULER_JOB_RUN_DETAILS`, `*_SCHEDULER_JOB_LOG`) |

### Practical `DBMS_SCHEDULER` example
```sql
BEGIN
  DBMS_SCHEDULER.create_job (
    job_name        => 'nightly_cleanup_job',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'BEGIN purge_old_logs; END;',
    start_date      => SYSTIMESTAMP,
    repeat_interval => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=0',
    enabled         => TRUE
  );
END;
/
```

### Reusable components — the "separate the parts" design
```sql
-- 1. Define WHAT to run (a reusable program)
DBMS_SCHEDULER.create_program(
  program_name => 'cleanup_prog', program_type => 'STORED_PROCEDURE',
  program_action => 'purge_old_logs', enabled => TRUE);

-- 2. Define WHEN (a reusable schedule)
DBMS_SCHEDULER.create_schedule(
  schedule_name => 'nightly_2am', repeat_interval => 'FREQ=DAILY; BYHOUR=2');

-- 3. Combine them into a job
DBMS_SCHEDULER.create_job(
  job_name => 'nightly_cleanup_job', program_name => 'cleanup_prog',
  schedule_name => 'nightly_2am', enabled => TRUE);
```
This separation lets you reuse the same `SCHEDULE` across many jobs, or swap a job's `PROGRAM` without touching its timing.

### Legacy `DBMS_JOB` (for comparison / migration awareness)
```sql
DECLARE
  v_job NUMBER;
BEGIN
  DBMS_JOB.submit(v_job, 'purge_old_logs;', SYSDATE, 'SYSDATE + 1');  -- runs daily
  COMMIT;
END;
/
```
Note the interval is a raw date-arithmetic **string expression** re-evaluated each run — much less readable/expressive than `DBMS_SCHEDULER`'s calendaring syntax, and no concept of a reusable named schedule.

### Important points to remember
- `DBMS_JOB` jobs and `DBMS_SCHEDULER` jobs are **tracked in different dictionary views** (`USER_JOBS` vs. `*_SCHEDULER_JOBS`), and are managed with different privilege sets.
- `DBMS_SCHEDULER` is the only one of the two that can run **native OS-level programs/shell scripts** (`job_type => 'EXECUTABLE'`), not just PL/SQL — a common exam differentiator.
- **Job chains** (a `DBMS_SCHEDULER`-only concept) let you build a step-by-step workflow (e.g., "run Step A, and only if it succeeds run Step B, otherwise run Step C") — something `DBMS_JOB` has no equivalent for.
- Oracle explicitly recommends `DBMS_SCHEDULER` over `DBMS_JOB` for **all new development**; `DBMS_JOB` exists solely for legacy compatibility.
- Both ultimately execute using an Oracle background process (job queue / scheduler process), not by keeping a client session open.

### Exercise Questions

**Q1.** A requirement states: the nightly job must run a shell script on the OS, not a PL/SQL procedure. Which package must be used, and why?
> **A:** `DBMS_SCHEDULER`, because it supports `job_type => 'EXECUTABLE'`, allowing it to run native OS programs or shell scripts directly. `DBMS_JOB` can only execute PL/SQL code (a PL/SQL block string) — it has no facility to invoke an OS-level executable.

**Q2.** What is the practical benefit of `DBMS_SCHEDULER`'s separation into `PROGRAM`, `SCHEDULE`, and `JOB` objects, compared to `DBMS_JOB`'s single, monolithic job definition?
> **A:** It enables **reuse and independent maintenance** — the same reusable `SCHEDULE` (e.g., "every night at 2 AM") can be attached to many different jobs, and the same reusable `PROGRAM` (what to run) can be scheduled differently in different environments, without duplicating either definition. `DBMS_JOB`, by contrast, bundles the "what" and "when" into one inline job submission, so any change to either requires touching the single job definition directly, and nothing is shareable across jobs.

**Q3.** True or False: `DBMS_SCHEDULER` supports building a multi-step workflow where Step 2 only runs if Step 1 succeeds, and a different Step 3 runs if Step 1 fails.
> **A:** **True.** This describes a **job chain**, a `DBMS_SCHEDULER`-specific feature (`DBMS_SCHEDULER.create_chain`, `define_chain_step`, `define_chain_rule`) that lets you define conditional dependency logic between steps. `DBMS_JOB` has no equivalent concept — each job is an independent, standalone unit with no built-in dependency/condition handling between jobs.

---

## 4. Large Object (LOB) Management

### What it is
LOBs (**Large OBjects**) store unstructured data too large for typical scalar column types — free text, XML, images, audio/video, arbitrary binary blobs — well beyond the size limits of `VARCHAR2`/`RAW`.

### The four LOB datatypes
| Type | Stores | Character set aware? | Stored in the database? |
|---|---|---|---|
| **`CLOB`** | Large single-byte or fixed-width character text | Yes (database character set) | Yes (internal) |
| **`NCLOB`** | Large character text in the **national character set** | Yes (national charset) | Yes (internal) |
| **`BLOB`** | Large raw/binary data (no character set) | No | Yes (internal) |
| **`BFILE`** | A **reference/pointer** to a file stored on the OS filesystem, outside the database | No | **No** — only the pointer is stored internally; the actual file lives on disk |

### Basic column declaration and manipulation
```sql
CREATE TABLE documents (
  doc_id   NUMBER PRIMARY KEY,
  doc_text CLOB,
  doc_bin  BLOB,
  doc_file BFILE
);

INSERT INTO documents (doc_id, doc_text) VALUES (1, 'Initial short text');

-- Working with a LOB locator inside PL/SQL
DECLARE
  v_clob CLOB;
BEGIN
  SELECT doc_text INTO v_clob FROM documents WHERE doc_id = 1 FOR UPDATE;
  DBMS_LOB.append(v_clob, ' -- appended content');
END;
/
```

### Important points to remember
- Small LOB values may physically be stored **in-row** (alongside the rest of the row) for performance, up to a size threshold — controlled by `ENABLE STORAGE IN ROW` vs `DISABLE STORAGE IN ROW` (see Section 7).
- A LOB column actually stores a **locator** (a kind of pointer/handle), not necessarily the literal bytes inline — PL/SQL code manipulates the LOB *through* this locator using `DBMS_LOB` calls.
- LOB values above **32 KB inline limit** for `VARCHAR2`/character literals must be built/manipulated with `DBMS_LOB` procedures (`APPEND`, `WRITE`, `READ`) rather than simple string concatenation, which is size-limited.
- `SELECT ... FOR UPDATE` is typically required before you can modify a LOB value in place via its locator (to lock the row and get a valid, non-read-only locator for writing).

### Exercise Questions

**Q1.** What is stored in a `BFILE` column, and what is *not* stored there?
> **A:** A `BFILE` column stores only a **locator** — essentially a directory-object reference plus a filename — pointing to a file that physically resides on the **operating system filesystem**, outside the database. The actual file contents are **not** stored in the database at all; Oracle only manages the reference to it.

**Q2.** Why can't you simply build a very large `CLOB` value using string concatenation (`clob_var := clob_var || 'more text';`) the way you might with a `VARCHAR2`?
> **A:** Implicit conversions and literal handling in PL/SQL are subject to the same underlying size constraints as `VARCHAR2` (effectively capped well below typical LOB sizes), and repeated concatenation can be highly inefficient for large objects since each concatenation may create a whole new temporary LOB value. `DBMS_LOB.APPEND`/`WRITE` operate directly and efficiently on the LOB's locator/underlying storage, which is the correct, scalable way to build up large LOB content.

**Q3.** Name the four LOB datatypes and identify which one is fundamentally different from the other three in terms of where its data physically resides.
> **A:** `CLOB`, `NCLOB`, `BLOB`, and `BFILE`. `BFILE` is fundamentally different: `CLOB`, `NCLOB`, and `BLOB` are **internal** LOBs whose actual data is stored and managed inside the database itself (subject to transactions, backup/recovery, etc.), whereas `BFILE` is an **external** LOB — only a reference is stored in the database; the real data lives outside it, on the filesystem.

---

## 5. Internal vs. External LOBs (`BLOB`, `CLOB`, `BFILE`)

### The core distinction
| | Internal LOBs (`BLOB`, `CLOB`, `NCLOB`) | External LOB (`BFILE`) |
|---|---|---|
| Data location | Stored **inside** the database (in tablespaces, as LOB segments) | Data lives in an **OS file**; only a locator (directory object + filename) is stored in the database |
| Transactional (participates in COMMIT/ROLLBACK)? | **Yes** — fully transactional, part of normal backup/recovery | **No** — file changes made outside Oracle are not tracked by Oracle transactions at all |
| Writable via PL/SQL/SQL? | Yes — full read/write via `DBMS_LOB`, SQL `INSERT`/`UPDATE` | **Read-only** from the database's perspective — Oracle cannot write to the OS file through a `BFILE`; you can only read it |
| Requires a `DIRECTORY` object? | No | **Yes** — a `CREATE DIRECTORY` schema object must map to the OS path, and the database OS user needs actual filesystem permission to read it |
| Participates in Oracle backup/recovery (RMAN)? | Yes | **No** — you must separately back up the referenced OS files yourself |
| Typical use case | Content that should be fully managed, versioned, and protected by the database (documents, images that must be transactionally consistent with other data) | Large, mostly-static reference files already living on a filesystem (e.g., a shared repository of scanned documents, media libraries) you don't want to duplicate into the database |

### Setting up and using a `BFILE`
```sql
-- 1. Create a directory object mapping to a real OS path
CREATE OR REPLACE DIRECTORY scan_dir AS '/u01/app/scanned_docs';
GRANT READ ON DIRECTORY scan_dir TO hr;

-- 2. Populate a BFILE locator pointing to a specific file
UPDATE documents
SET    doc_file = BFILENAME('SCAN_DIR', 'invoice_1042.pdf')
WHERE  doc_id = 1;

-- 3. Read it (e.g., check existence, get length) — cannot write to it
DECLARE
  v_bfile BFILE;
  v_len   INTEGER;
BEGIN
  SELECT doc_file INTO v_bfile FROM documents WHERE doc_id = 1;
  DBMS_LOB.fileopen(v_bfile, DBMS_LOB.file_readonly);
  v_len := DBMS_LOB.getlength(v_bfile);
  DBMS_LOB.fileclose(v_bfile);
END;
/
```

### Important points to remember
- **`BFILE` is always read-only** from the database side — this is one of the most tested facts on this topic. If you need the database to be able to modify the content, it must be an internal LOB (`BLOB`/`CLOB`), not a `BFILE`.
- Because `BFILE` content lives outside the database, it is **not covered by Oracle's transactional guarantees or by RMAN backups** — if the underlying OS file is deleted or changed outside of Oracle's awareness, the database has no way to detect or prevent that, and standard database backup/recovery does not protect it.
- A `DIRECTORY` object is itself a schema object requiring `CREATE ANY DIRECTORY`/appropriate privileges to create, and `READ` (and only `READ`, never `WRITE`, for `BFILE` purposes) privilege to use.
- Internal LOBs benefit from all the standard Oracle storage features covered in Section 7 (SecureFile compression/encryption/deduplication) — **`BFILE`s cannot use any of these**, since Oracle isn't managing the actual storage.

### Exercise Questions

**Q1.** Can a PL/SQL program modify the contents of a file referenced by a `BFILE` locator?
> **A:** No. `BFILE` provides **read-only** access to the external file from within the database — `DBMS_LOB` offers functions like `FILEOPEN`, `FILEREAD`, `GETLENGTH`, and `FILEEXISTS` for `BFILE`s, but there is no `BFILE` write/append capability. To change the file's content, you must modify it outside Oracle, at the OS level.

**Q2.** A team stores critical scanned invoices as `BFILE`s referencing a network share, and assumes their nightly RMAN backup fully protects this data. Is this assumption correct?
> **A:** No — this is a common real-world mistake. RMAN backs up the **database's own storage** (tablespaces, datafiles), which only contains the `BFILE` **locator** (directory + filename), not the actual file bytes. The real invoice files sitting on the OS/network share must be backed up **separately**, using OS-level or network-share backup tools; RMAN has no visibility into or control over that external storage.

**Q3.** Why might an organization deliberately choose `BFILE` over `BLOB` for a large repository of media files, despite giving up transactional consistency and write access?
> **A:** Common reasons include: (a) the files already exist in a large, actively-used external repository/filesystem and duplicating them into the database would waste significant storage and migration effort; (b) the files are effectively static/reference data that doesn't need in-database write access; (c) avoiding the growth of the database's own storage/backups with very large binary content that's better managed by dedicated file-storage infrastructure; and (d) allowing external, non-database processes to continue managing that file storage directly.

---

## 6. Using `DBMS_LOB` for Performance

### What it is
`DBMS_LOB` is the built-in package providing the full API for reading, writing, comparing, and manipulating LOB values through their **locators** — the correct, efficient way to work with LOB content instead of naive whole-value manipulation.

### Key procedures/functions
| Subprogram | Purpose |
|---|---|
| `DBMS_LOB.GETLENGTH(lob)` | Returns the length of the LOB |
| `DBMS_LOB.READ(lob, amount, offset, buffer)` | Reads a **chunk** starting at a given offset — avoids loading the whole LOB into memory |
| `DBMS_LOB.WRITE(lob, amount, offset, buffer)` | Writes a chunk at a specific offset (overwrites) |
| `DBMS_LOB.WRITEAPPEND(lob, amount, buffer)` | Appends a chunk to the end |
| `DBMS_LOB.APPEND(dest_lob, src_lob)` | Appends one whole LOB's content to another |
| `DBMS_LOB.COPY(dest, src, amount, ...)` | Copies a range of bytes/characters from one LOB to another |
| `DBMS_LOB.SUBSTR(lob, amount, offset)` | Returns a portion as an in-memory value (subject to `VARCHAR2`/`RAW` size limits — for **small** extracts only) |
| `DBMS_LOB.COMPARE(lob1, lob2)` | Byte/character-wise comparison — the correct way to "compare LOBs" since `=` doesn't work directly the way it does for scalars in SQL predicates on LOB columns in all contexts |
| `DBMS_LOB.INSTR` | Locates a pattern within a LOB |
| `DBMS_LOB.TRIM` / `ERASE` | Shrinks or zeroes out part of a LOB |
| `DBMS_LOB.FILEOPEN` / `FILECLOSE` / `FILEEXISTS` | `BFILE`-specific read operations |
| `DBMS_LOB.CREATETEMPORARY` / `FREETEMPORARY` | Creates/frees a **temporary LOB** (session-duration, not tied to a table row) — very common for building up LOB content in PL/SQL before an `INSERT` |

### Practical performance-oriented example — chunked read instead of loading the whole LOB
```sql
DECLARE
  v_clob   CLOB;
  v_buffer VARCHAR2(32767);
  v_amount INTEGER := 32767;
  v_offset INTEGER := 1;
BEGIN
  SELECT doc_text INTO v_clob FROM documents WHERE doc_id = 1;

  LOOP
    BEGIN
      DBMS_LOB.read(v_clob, v_amount, v_offset, v_buffer);
    EXCEPTION
      WHEN NO_DATA_FOUND THEN EXIT;  -- reached the end
    END;
    -- process v_buffer (e.g., scan/transform this chunk)
    v_offset := v_offset + v_amount;
  END LOOP;
END;
/
```

### Building a LOB efficiently with a temporary LOB
```sql
DECLARE
  v_temp CLOB;
BEGIN
  DBMS_LOB.createtemporary(v_temp, TRUE);
  DBMS_LOB.writeappend(v_temp, LENGTH('Part 1. '), 'Part 1. ');
  DBMS_LOB.writeappend(v_temp, LENGTH('Part 2.'), 'Part 2.');

  INSERT INTO documents (doc_id, doc_text) VALUES (99, v_temp);

  DBMS_LOB.freetemporary(v_temp);
END;
/
```

### Important points to remember
- **Chunked `READ`/`WRITE` calls (with a bounded `amount`)** are the key performance technique for very large LOBs — never pull an entire multi-gigabyte LOB into a single PL/SQL variable if you only need to process it piece by piece.
- `DBMS_LOB.SUBSTR` still has the underlying scalar size limit (effectively bounded like `VARCHAR2`/`RAW`) — it is for extracting **small** portions, not a general large-scale extraction tool.
- **Temporary LOBs** (`CREATETEMPORARY`) avoid the overhead of persisting many small intermediate versions to a real LOB segment while you're still assembling content — free them explicitly with `FREETEMPORARY` when done, or they persist for the duration of the session/call and consume temp space.
- Always match your read/write **`CHUNK`** size (Section 7) to the amounts you pass to `DBMS_LOB.READ`/`WRITE` for optimal I/O efficiency — mismatched chunk sizes cause unnecessary I/O overhead.
- Comparing LOBs for equality should use `DBMS_LOB.COMPARE`, not a blind reliance on `=` in all contexts, since LOB comparison semantics/behavior can differ from scalar equality depending on version and context — `DBMS_LOB.COMPARE` is the documented, reliable API for this.

### Exercise Questions

**Q1.** A developer needs to scan a 500 MB `CLOB` for a keyword, but is worried about memory usage. What `DBMS_LOB` technique addresses this concern?
> **A:** Use a **chunked read loop** with `DBMS_LOB.READ`, pulling in a bounded amount (e.g., 32 KB) at a time into a `VARCHAR2` buffer, processing/searching each chunk, and advancing the offset — rather than attempting to load the entire 500 MB value into a single in-memory variable at once, which would be both wasteful and potentially exceed variable size limits.

**Q2.** What is the purpose of `DBMS_LOB.CREATETEMPORARY`, and why would a developer use it instead of just repeatedly updating a real table's LOB column while assembling content?
> **A:** `CREATETEMPORARY` creates a session-duration LOB that is **not tied to any table row**, letting you assemble/build up content in memory-backed temporary LOB storage efficiently before a single final `INSERT`/`UPDATE`. Repeatedly writing to a real persisted LOB column during assembly would generate unnecessary redo/undo and intermediate row versions; building in a temporary LOB and inserting once is far more efficient.

**Q3.** Why is `DBMS_LOB.SUBSTR` not an appropriate tool for extracting the "first 1 MB" of a very large LOB?
> **A:** `DBMS_LOB.SUBSTR` returns its result as a scalar `VARCHAR2`/`RAW` value, which is subject to the same underlying size ceiling as those types — it is designed for extracting **small** portions of a LOB (well under that limit), not megabyte-scale chunks. For larger extractions, `DBMS_LOB.READ` in a loop (or `DBMS_LOB.COPY` to another LOB) is the correct approach.

---

## 7. SecureFile LOBs — Compression, Encryption, and Deduplication

### What it is
**SecureFiles** (introduced 11g) is Oracle's modern LOB storage architecture — a full redesign of internal LOB storage that replaces the older **BasicFile** LOB format. Oracle recommends SecureFiles for **all new** persistent LOB storage; BasicFile remains only for backward compatibility.

### Creating a SecureFile LOB with all three features
```sql
CREATE TABLE sales_docs (
  doc_id      NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  order_id    NUMBER NOT NULL,
  doc_content BLOB
)
LOB (doc_content) STORE AS SECUREFILE sales_docs_lob (
  TABLESPACE      lob_ts
  ENABLE STORAGE IN ROW
  COMPRESS MEDIUM
  DEDUPLICATE
  ENCRYPT USING 'AES256'
  CACHE
  LOGGING
);
```

### The three headline SecureFile features
| Feature | Clause | What it does |
|---|---|---|
| **Compression** | `COMPRESS [LOW \| MEDIUM \| HIGH]` (or `NOCOMPRESS`) | Compresses LOB data on disk, transparently to readers/writers. Higher levels save more space at higher CPU cost. Best for text/office documents; gives little benefit (and wastes CPU) on already-compressed formats like JPEG/PNG/most video. |
| **Deduplication** | `DEDUPLICATE` (or `KEEP_DUPLICATES`) | If **multiple LOB values across the segment are byte-for-byte identical**, only **one physical copy** is stored, with other rows referencing it — saves significant space for repeated attachments (e.g., a standard boilerplate PDF attached to many records). |
| **Encryption** | `ENCRYPT [USING 'algorithm'] [IDENTIFIED BY password]` (or `DECRYPT`) | Encrypts LOB data at rest using Transparent Data Encryption (TDE) — requires the TDE wallet/keystore infrastructure to be configured. |

### Altering an existing SecureFile LOB
```sql
ALTER TABLE sales_docs MODIFY LOB (doc_content) (COMPRESS HIGH DEDUPLICATE);
```
Some changes apply immediately to how new data is stored (metadata-only change); others (like changing from `BASICFILE` to `SECUREFILE`, or certain storage-characteristic changes) require an actual data move (`ALTER TABLE ... MOVE`) to rewrite existing LOB data under the new settings.

### `ENABLE STORAGE IN ROW` — the small-object performance lever
- If a LOB value is small enough to fit within the row's own storage (subject to an internal threshold), `ENABLE STORAGE IN ROW` (the default) stores it **inline with the row** — much faster to fetch since it avoids a separate LOB segment read.
- `DISABLE STORAGE IN ROW` forces LOB data **out-of-row** always, regardless of size — appropriate when LOBs are reliably large and you don't want to risk bloating/fragmenting the base table's blocks with even the smaller ones.

### Important points to remember
- SecureFiles-only features: `COMPRESS`, `DEDUPLICATE`, and `ENCRYPT` are **not available on legacy BasicFile LOBs** — this is a very common exam distinction ("which of the following is a SecureFile-exclusive feature?").
- `COMPRESS` and `DEDUPLICATE` are complementary but distinct: compression reduces the size of *each individual* LOB value; deduplication avoids storing multiple *copies* of the *same* value more than once. You can use both together.
- Compression provides little to no benefit — and pure CPU overhead — on data that is **already compressed** at the application level (JPEG, PNG, MP3, most video, already-zipped content, or already-encrypted content). It's most valuable for text, XML, and office document formats.
- SecureFile LOB **encryption** integrates with Oracle's standard **Transparent Data Encryption (TDE)** infrastructure — it requires a configured wallet/keystore, the same underlying mechanism used for TDE column/tablespace encryption elsewhere in the database.
- `CACHE` / `NOCACHE` / `CACHE READS` control whether LOB data is cached in the buffer cache like ordinary table data (`CACHE`), never cached (`NOCACHE` — typical for very large, rarely-reread LOBs to avoid buffer cache pollution), or cached only for reads (`CACHE READS`).
- Data dictionary views `USER_LOBS` / `ALL_LOBS` / `DBA_LOBS` show storage properties (in-row status, securefile/basicfile, etc.) for existing LOB columns.

### Exercise Questions

**Q1.** A table stores thousands of employee-submitted expense-report PDFs, many of which are the exact same standard company template with only minor variation elsewhere in the row. Which SecureFile feature specifically targets this scenario, and why?
> **A:** **`DEDUPLICATE`**. Since many of the PDF LOB values are byte-for-byte identical (the same template), deduplication detects and stores only **one physical copy** of each distinct value, with other rows simply referencing that shared copy — dramatically reducing total storage compared to storing a full separate copy of the same bytes for every row.

**Q2.** Why would enabling `COMPRESS HIGH` on a SecureFile LOB column that stores JPEG images provide little practical benefit?
> **A:** JPEG is already a compressed image format — its data has little remaining redundancy for a general-purpose compression algorithm to exploit further. Applying database-level LOB compression on top of already-compressed content typically yields minimal additional space savings while still consuming CPU cycles to attempt the compression on every write — a poor trade-off. Compression is far more effective on inherently redundant, uncompressed data like plain text or XML.

**Q3.** True or False: `COMPRESS`, `DEDUPLICATE`, and `ENCRYPT` clauses can all be specified on a `BASICFILE` LOB column.
> **A:** **False.** These three features are exclusive to the **SecureFile** LOB storage architecture. `BASICFILE` is the older LOB format retained purely for backward compatibility and does not support compression, deduplication, or this style of encryption — Oracle's official recommendation is to use SecureFile storage for all new persistent LOB columns specifically to gain access to these capabilities.

---

## Quick Cross-Topic Summary Table

| Topic | Key Package/Clause | Runs/Stored Where |
|---|---|---|
| Calling C | `CREATE LIBRARY`, `LANGUAGE C` call spec | Separate OS process (`extproc` agent) |
| Calling Java | `loadjava`, `AS LANGUAGE JAVA` call spec | Embedded JVM inside the DB process |
| Secure external procedures | `extproc.ora`, `EXTPROC_DLLS=ONLY:...`, IPC-only listener | OS-level configuration |
| Job scheduling (modern) | `DBMS_SCHEDULER` (`PROGRAM`/`SCHEDULE`/`JOB`, chains) | Scheduler background processes |
| Job scheduling (legacy) | `DBMS_JOB` | Job queue background process |
| LOB types | `CLOB`/`NCLOB`/`BLOB` (internal), `BFILE` (external) | Internal = in DB; `BFILE` = OS filesystem |
| LOB manipulation | `DBMS_LOB` (`READ`/`WRITE`/`APPEND`/`COMPARE`/temp LOBs) | Locator-based access, in PL/SQL |
| Modern LOB storage | SecureFile (`COMPRESS`/`DEDUPLICATE`/`ENCRYPT`) | Replaces legacy BasicFile storage |