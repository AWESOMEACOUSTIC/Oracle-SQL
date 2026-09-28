# CI/CD for Oracle PL/SQL
### Exam Study Guide

Covers: Version Control (Git) · Build & Validation (CI Pipeline) · Release Packaging · Automated Deployments (CD)

> **A note on scope.** Unlike the earlier guides, CI/CD is a *practice* built from tools (Git, a CI server, a migration tool, a test framework), not a single Oracle feature. Exams therefore test **concepts, ordering, and pitfalls** far more than exact command flags. Tool names, image tags and options below are **illustrative** — always check your own tool's documentation for exact syntax. Cross-references to earlier guides (utPLSQL/coverage, PL/Scope, `DBMS_UTILITY`, `DBMS_METADATA`, package state, Edition-Based Redefinition) are called out where they matter.

---

## 0. Core Vocabulary

| Term | Meaning |
|---|---|
| **Continuous Integration (CI)** | Developers merge small changes into a shared branch frequently; **every change is automatically built and tested** in a clean environment |
| **Continuous Delivery (CD)** | Every change that passes the pipeline is **kept in a releasable state** and can be deployed to production at any time — the final push is a **manual approval** |
| **Continuous Deployment (CD)** | Goes one step further: a passing change is **deployed to production automatically**, with no manual gate |
| **Pipeline** | The automated sequence of stages (build → test → package → deploy) |
| **Artifact** | The versioned, packaged output of a build that gets deployed |
| **Immutable artifact / "build once, deploy many"** | The *same* artifact, unchanged, is promoted through DEV → TEST → PROD |
| **Environment promotion** | Moving an artifact to the next environment after it passes that stage's checks |
| **Ephemeral environment** | A disposable database created for one pipeline run and thrown away afterwards |
| **Migration** | A versioned script that moves a database from one state to the next |
| **Idempotent (re-runnable)** | Running the script again leaves the same end result and doesn't fail |
| **Quality gate** | An automated pass/fail rule (e.g., "no invalid objects", "coverage ≥ 80%") that can stop a pipeline |
| **Roll-forward vs. rollback** | Fixing a bad release by deploying a *new* fix vs. reverting to the previous state |
| **Schema drift** | The database differing from what version control says it should be (e.g., a manual hotfix in production) |

---

## 1. Version Control (Git)

### What it is and why it matters
With PL/SQL, the code physically lives **inside the database** — which tempts teams to treat the database as the "master copy". That is the root of most database deployment problems. The database holds only the **current state** (no history, no author, no review, no branches). In a CI/CD approach the **Git repository is the single source of truth**, and every database — dev, test, production — is just a **deployment target** built from it.

Benefits: full change history and blame, code review through pull requests, branching for parallel work, ability to **rebuild any environment from scratch**, reproducible releases (tags), and an audit trail.

### What goes into the repository
| Version it | Do **not** commit |
|---|---|
| Package specs/bodies, procedures, functions, triggers, types, views | **Passwords, wallets, keys, tokens** |
| Table/index/sequence DDL **as migrations** | Generated artifacts (`dist/`, zips, logs, test reports) |
| Grants, synonyms, scheduler jobs | Production data |
| Reference/seed data scripts | Environment-specific settings baked into scripts |
| **Test code** (utPLSQL packages) | Personal IDE settings |
| Pipeline definitions, helper scripts, README | |

### Typical repository layout
```
repo/
├── src/
│   ├── packages/       pkg_order_api.pks   pkg_order_api.pkb
│   ├── functions/  procedures/  triggers/  views/  types/
│   └── jobs/
├── migrations/         V2_4_0_001__add_status_to_orders.sql   (tables, indexes, seed data)
├── test/               ut_pkg_order_api.pks   ut_pkg_order_api.pkb
├── ci/                 install_all.sql   check_invalid.sql
├── .gitattributes   .gitignore   README.md
└── (pipeline file: Jenkinsfile / .gitlab-ci.yml / .github/workflows/ci.yml)
```
Conventions: **one object per file**; consistent extensions (`.pks` spec, `.pkb` body, `.fnc`, `.prc`, `.trg`, `.vw`, `.tps`/`.tpb`); file name = object name; deterministic formatting so diffs show real changes only.

### State-based vs. migration-based — the key design idea
| | **State-based (declarative)** | **Migration-based (incremental)** |
|---|---|---|
| Stores | The **desired end state** of each object | An **ordered series of change scripts** |
| Fits | **Code objects**: packages, procedures, functions, views, triggers, types — `CREATE OR REPLACE` is safe to rerun | **Tables and data**: `ALTER TABLE ADD ...`, index changes, data fixes — a table holding data **cannot** be "replaced" |
| Re-run behavior | Rerun freely (or only when the file changed) | Each script runs **exactly once**, in order |
| Tool terms | Flyway *repeatable* migrations (`R__...`); Liquibase `runOnChange` | Flyway *versioned* migrations (`V2_4_0__...`); Liquibase change sets |

**In practice you use a hybrid:** migrations for tables/data, repeatable/state-based files for PL/SQL code.

### Git workflow essentials
```bash
git switch -c feature/ORD-142-cancel-order          # short-lived feature branch
git add src/packages/pkg_order_api.pks src/packages/pkg_order_api.pkb test/
git commit -m "ORD-142: allow cancelling open orders"
git push -u origin feature/ORD-142-cancel-order
# open a pull/merge request -> CI runs automatically -> peer review -> merge to main

git tag -a v2.4.0 -m "Release 2.4.0"                 # annotated tag marks the release
git push origin v2.4.0
```
- **Trunk-based / short-lived branches** integrate often and keep merges small (the CI ideal); long-lived branches (heavy GitFlow) delay integration and increase conflicts.
- **Protect `main`:** require a passing pipeline and at least one review before merge.
- **Tags** (e.g., semantic versions `MAJOR.MINOR.PATCH`) identify exactly what was released.
- `merge` preserves branch history; `rebase` rewrites the branch on top of the latest `main` for a linear history. Never rebase shared/published branches.

### Database-specific version-control pitfalls
- **Line endings:** CRLF vs LF changes can alter a file's **checksum**, making migration tools think an applied script was edited. Pin them:
  ```
  # .gitattributes
  *.sql text eol=lf
  *.pks text eol=lf
  *.pkb text eol=lf
  ```
- **Monolithic package bodies** (thousands of lines) cause constant merge conflicts — split large packages into smaller cohesive ones.
- **Baseline the existing database once:** extract current DDL with `DBMS_METADATA` (see the built-in packages guide) or SQLcl (`ddl`, `project export`); strip environment noise such as storage/tablespace clauses (`SEGMENT_ATTRIBUTES`/`STORAGE` transforms set to FALSE) and hard-coded schema names.
- **One isolated database per developer** (local container, or a private schema/PDB) — a shared dev schema means developers overwrite each other's work and the repo stops being the source of truth.
- **No manual changes in shared environments** — every change goes through Git so that the repo and the database don't drift.

### Important points to remember
- **Git = source of truth; the database = deployment target.**
- **Tables → migrations; code → repeatable `CREATE OR REPLACE` files.**
- Never commit **secrets**; keep credentials in the CI secret store or an Oracle Wallet.
- **Never edit a migration that has already been applied** — add a new one (checksum protection, Section 3).
- Test code lives in the repo too, but is **not deployed to production**.

### Exercise Questions

**Q1.** A team keeps its "real" package code in the shared development database and copies it into Git occasionally. What goes wrong?
> **A:** The database becomes the master, so Git is a stale copy: there is no reliable history or review of what changed, developers overwrite one another's edits in the shared schema, environments can't be rebuilt reproducibly from the repository, and releases can't be traced to a tag. The fix is to make Git the source of truth: developers work in isolated databases, commit every change, and databases are only ever built or updated from the repository by the pipeline.

**Q2.** Why can a package body be handled with `CREATE OR REPLACE` in a repeatable file, but a table change must be a versioned migration?
> **A:** `CREATE OR REPLACE` swaps the definition of a code object with no data at stake, so re-running it is harmless and idempotent. A table holds **data**; you can't replace it without losing rows, so changes must be expressed as ordered, run-once steps (`ALTER TABLE ...`, data fixes) that transform the existing structure — which is exactly what versioned migrations do.

**Q3.** After a developer switches to Windows, the migration tool reports a checksum mismatch on an unchanged SQL file. Why, and how is it prevented?
> **A:** Git or the editor converted line endings (LF ↔ CRLF), changing the file's bytes and therefore its checksum, so the tool believes an already-applied migration was modified. Prevent it with `.gitattributes` rules that force consistent line endings (e.g., `*.sql text eol=lf`) and by not letting editors reformat applied scripts.

**Q4.** Why do very large package bodies hurt a Git-based workflow, and what is the remedy?
> **A:** Many developers edit the same file, so pull requests collide and produce frequent, painful merge conflicts and noisy diffs. The remedy is to split the package into smaller, cohesive units (one responsibility each), keep formatting consistent, and integrate small changes frequently through short-lived branches.

---

## 2. Build & Validation (the CI Pipeline)

### What the CI pipeline does
On **every push and pull request**, the CI server automatically: gets the code, creates a **clean, disposable database**, installs everything **from scratch**, checks that it compiles, runs the tests and static checks, and reports pass/fail. The goal is fast feedback — a broken change is caught minutes after it is committed, not weeks later in production.

### Typical stages
| # | Stage | Purpose |
|---|---|---|
| 1 | **Checkout** | Get the commit under test |
| 2 | **Provision an ephemeral database** | Usually a container running Oracle Database Free/XE (or a cloned PDB) — clean and identical every run |
| 3 | **Install prerequisites** | The test framework (**utPLSQL**), required roles/grants |
| 4 | **Build** | Apply migrations and code **from scratch** (and, on release branches, apply the upgrade over the previous version) |
| 5 | **Compile check** | No compile errors, **zero invalid objects** |
| 6 | **Static analysis** | Lint/code-quality rules, warnings, formatting, security patterns |
| 7 | **Unit tests** | utPLSQL suites |
| 8 | **Coverage** | Measure what the tests executed |
| 9 | **Publish reports** | JUnit XML, coverage, warnings — visible in the CI server |
| 10 | **Quality gate** | Fail the build if any rule is broken |

### Compile validation — the trap everyone hits
In SQL*Plus and SQLcl, a `CREATE OR REPLACE PACKAGE BODY` with a compile error **does not raise a SQL error**: the object is created **INVALID** and the client prints only a warning (e.g., *"Warning: Package Body created with compilation errors"*, SP2-0810). Consequently:

```sql
WHENEVER SQLERROR EXIT FAILURE    -- does NOT trigger on compilation errors!
```
A build script must therefore **explicitly check** the dictionary after installing:
```sql
-- ci/check_invalid.sql
WHENEVER SQLERROR EXIT FAILURE
SET SERVEROUTPUT ON
DECLARE
  v_cnt NUMBER;
BEGIN
  DBMS_UTILITY.compile_schema(schema => USER, compile_all => FALSE);   -- recompile only invalid objects

  SELECT COUNT(*) INTO v_cnt FROM user_objects WHERE status = 'INVALID';

  IF v_cnt > 0 THEN
    FOR r IN (SELECT name, type, line, position, text
              FROM   user_errors
              WHERE  attribute = 'ERROR'
              ORDER  BY name, type, sequence)
    LOOP
      DBMS_OUTPUT.put_line(r.type || ' ' || r.name || ' (' || r.line || ':' || r.position || ') ' || r.text);
    END LOOP;
    RAISE_APPLICATION_ERROR(-20999, v_cnt || ' invalid object(s) after compilation');
  END IF;
END;
/
EXIT
```
Why recompile first? Objects can be temporarily invalid only because a dependency was installed after them; `COMPILE_SCHEMA` (or `UTL_RECOMP` for large schemas) resolves ordering effects so the check reports **genuine** failures only. The failing statement makes SQL*Plus/SQLcl exit non-zero, which fails the pipeline step.

### Treating warnings as build failures
```sql
ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:ALL', 'ERROR:05005', 'DISABLE:06005';
```
- `ENABLE`/`DISABLE` turn warning categories or individual warnings on/off; the **`ERROR` qualifier promotes chosen warnings to compile errors**, making the object fail to compile — a strict way to enforce rules (e.g., a specific severe warning) in CI.
- Compile with **`PLSCOPE_SETTINGS = 'IDENTIFIERS:ALL'`** in the CI build so PL/Scope data is available for analysis queries (unused variables, references to deprecated units, etc.) — see the PL/Scope guide.

### Unit testing with utPLSQL
utPLSQL is the de-facto open-source unit-testing framework for PL/SQL (inspired by JUnit/RSpec). Tests are **PL/SQL packages configured by annotations in the package specification**.
```sql
CREATE OR REPLACE PACKAGE ut_order_api AS
  --%suite(Order API)

  --%test(Cancelling an open order sets status CANCELLED)
  PROCEDURE cancel_open_order;

  --%test(Cancelling a shipped order is rejected)
  --%throws(-20010)
  PROCEDURE cancel_shipped_order;
END ut_order_api;
/
CREATE OR REPLACE PACKAGE BODY ut_order_api AS
  PROCEDURE cancel_open_order IS
    v_id     orders.order_id%TYPE;
    v_status orders.status%TYPE;
  BEGIN
    INSERT INTO orders (order_id, status) VALUES (orders_seq.NEXTVAL, 'OPEN')
    RETURNING order_id INTO v_id;

    pkg_order_api.cancel_order(v_id);

    SELECT status INTO v_status FROM orders WHERE order_id = v_id;
    ut.expect(v_status).to_equal('CANCELLED');
  END;

  PROCEDURE cancel_shipped_order IS
    v_id orders.order_id%TYPE;
  BEGIN
    INSERT INTO orders (order_id, status) VALUES (orders_seq.NEXTVAL, 'SHIPPED')
    RETURNING order_id INTO v_id;
    pkg_order_api.cancel_order(v_id);      -- expected to raise ORA-20010
  END;
END ut_order_api;
/
```
Key annotations: `--%suite`, `--%test`, `--%beforeall`, `--%beforeeach`, `--%afterall`, `--%aftereach`, `--%throws`, `--%disabled`, `--%context`. A package is a suite **only if its specification contains `--%suite`** at package level.

**Running and reporting:** `EXEC ut.run;` from SQL, or the **utPLSQL command-line client** in the pipeline. Reporters produce formats CI servers consume: the default **documentation** reporter (human-readable), **JUnit XML** (test results), and **coverage** reporters (HTML, Cobertura, SonarQube, Coveralls-style). Tests run in a transaction that utPLSQL **rolls back automatically**; if the code under test issues a `COMMIT`, that automatic rollback is invalidated and the framework reports a warning.

### Static analysis and quality checks
| Check | Examples |
|---|---|
| **Compiler warnings** | `PLSQL_WARNINGS` (unreachable code, unused variables, implicit conversions) |
| **Lint / code-smell tools** | PL/SQL Cop-style rule sets, SonarQube's PL/SQL analysis |
| **PL/Scope queries** | Unused identifiers, calls to banned/deprecated units |
| **Security patterns** | Dynamic SQL built by concatenation without bind variables or `DBMS_ASSERT`; use of `EXECUTE IMMEDIATE` on unchecked input; hard-coded credentials |
| **Formatting** | Consistent style enforced automatically so reviews focus on logic |

### Quality gates (typical)
✅ Everything installs from an empty database · ✅ **zero invalid objects** · ✅ all unit tests pass · ✅ coverage ≥ agreed threshold (and not falling) · ✅ no new severe warnings · ✅ no secrets/security-rule violations. **Any failure stops the pipeline.**

### Example pipeline (GitHub Actions style — illustrative)
```yaml
name: plsql-ci
on: [push, pull_request]
jobs:
  build-test:
    runs-on: ubuntu-latest
    services:
      oracle:
        image: gvenzl/oracle-free:slim-faststart      # disposable Oracle Database Free container
        env:
          ORACLE_PASSWORD: ${{ secrets.CI_DB_PASSWORD }}
          APP_USER: app
          APP_USER_PASSWORD: ${{ secrets.CI_DB_PASSWORD }}
        ports: ["1521:1521"]
        options: >-
          --health-cmd healthcheck.sh --health-interval 10s --health-timeout 5s --health-retries 30
    steps:
      - uses: actions/checkout@v4
      - name: Install tools (SQLcl, utPLSQL CLI)
        run: ./ci/install_tools.sh
      - name: Install utPLSQL framework into the fresh DB
        run: sql -S sys/${{ secrets.CI_DB_PASSWORD }}@//localhost:1521/FREEPDB1 as sysdba @ci/install_utplsql.sql
      - name: Build from scratch (migrations + code)
        run: sql -S app/${{ secrets.CI_DB_PASSWORD }}@//localhost:1521/FREEPDB1 @ci/install_all.sql
      - name: Fail on invalid objects
        run: sql -S app/${{ secrets.CI_DB_PASSWORD }}@//localhost:1521/FREEPDB1 @ci/check_invalid.sql
      - name: Unit tests + coverage
        run: >
          utplsql run app/${{ secrets.CI_DB_PASSWORD }}@//localhost:1521/FREEPDB1
          -f=ut_junit_reporter -o=reports/junit.xml
          -f=ut_coverage_cobertura_reporter -o=reports/coverage.xml
          -source_path=src -test_path=test
      - uses: actions/upload-artifact@v4
        with: { name: reports, path: reports/ }
```
*(Illustrative: image tags, CLI flags and connect strings vary by version; in real pipelines prefer a wallet or environment-based credentials over passwords on command lines.)*

### Important points to remember
- **Build from scratch in an ephemeral database every time** — it exposes missing scripts, wrong ordering and hidden dependencies on objects that "just happen to exist" in a long-lived dev database.
- **`WHENEVER SQLERROR` does not catch PL/SQL compilation errors** — check `USER_ERRORS`/`USER_OBJECTS` (`STATUS = 'INVALID'`) yourself.
- Recompile (`DBMS_UTILITY.COMPILE_SCHEMA(..., compile_all => FALSE)` or `UTL_RECOMP`) **before** deciding an object is broken.
- **Fast feedback:** keep the pipeline short; run the heavy suites (performance, full upgrade matrix) on merge or nightly.
- Tests must be **deterministic and independent** (no reliance on other tests' data or on `SYSDATE` without control) and are **never deployed to production**.
- **Coverage measures execution, not correctness** (see the tuning guide) — use it as a trend and a floor, not proof of quality.

### Exercise Questions

**Q1.** A pipeline runs `WHENEVER SQLERROR EXIT FAILURE` and then installs a package body containing a syntax error. The pipeline stays green, but the package is unusable in test. Why, and how do you fix the pipeline?
> **A:** A PL/SQL **compilation error is not a SQL error** — `CREATE OR REPLACE` succeeds and creates the object in the `INVALID` state, with only a warning printed, so `WHENEVER SQLERROR` never fires. Add an explicit validation step that recompiles invalid objects and then fails if any remain (`USER_OBJECTS.STATUS = 'INVALID'`, details from `USER_ERRORS WHERE ATTRIBUTE = 'ERROR'`), returning a non-zero exit code.

**Q2.** Why should CI create a fresh database for each run rather than reuse the shared development database?
> **A:** A long-lived database accumulates objects and data that scripts may silently depend on, so a build can pass there yet fail on a real new environment (missing scripts, wrong install order, forgotten grants). A clean, disposable database proves the repository alone can rebuild the system, gives identical conditions every run (reproducibility), and lets tests run in isolation without interfering with anyone else.

**Q3.** A utPLSQL test inserts rows and the code under test calls `COMMIT`. What is the consequence?
> **A:** utPLSQL wraps each test in a transaction and rolls it back automatically so tests leave no data behind. A `COMMIT` inside the code under test makes that automatic rollback ineffective for the committed work — the framework reports a warning, and the data persists, which can break test independence and pollute later runs. Design tests around code that doesn't commit internally, or clean up explicitly (e.g., in an `--%afterall` procedure).

**Q4.** What does `ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:ALL','ERROR:05005'` do differently from plain `'ENABLE:ALL'`?
> **A:** `ENABLE:ALL` only **reports** warnings; the objects still compile and the build would pass. Adding `ERROR:05005` promotes that specific warning to a **compile error**, so any unit triggering it fails to compile and therefore fails the build's validity check — a way to enforce a rule automatically.

---

## 3. Release Packaging

### What it is
**Release packaging** turns a tested commit into a **versioned, self-contained, immutable artifact** — the thing that is actually deployed. The principle is **"build once, deploy many"**: the *same* bytes that passed CI and testing are promoted to UAT and production. Rebuilding for each environment would mean production runs code that was never actually tested.

### What a release package contains
| Item | Notes |
|---|---|
| **Ordered install/upgrade scripts** | Migrations for tables/data, then code files |
| **Master (driver) script** | e.g., `install.sql` calling everything in order |
| **Changelog / migration metadata** | Liquibase changelog or Flyway migration folder |
| **Pre- and post-deployment checks** | Version check, invalid-object check, smoke tests |
| **Rollback / undo scripts** (if used) | Or a documented roll-forward plan |
| **Grants, synonyms, job definitions** | Often deployed separately by a privileged account |
| **Release notes & version manifest** | What changed, version number, source commit/tag, checksums |
| **Excluded** | Unit tests, developer utilities, secrets |

Common formats: a **ZIP/tar** of scripts with a master script (this is what SQLcl Projects' `gen-artifact` produces, containing the release's `dist` files and `install.sql`), a Liquibase changelog bundle, or a Flyway migrations folder — stored in an **artifact repository** (or release attachments) with a checksum.

### Versioning
- **Semantic versioning** `MAJOR.MINOR.PATCH` — MAJOR for incompatible changes, MINOR for backward-compatible features, PATCH for fixes — mirrored by a **Git tag** (`v2.4.0`).
- Record the deployed version **inside the database**: the migration tool's history table (`DATABASECHANGELOG`, `flyway_schema_history`) and/or an application version table or package constant, so you can always query "what is deployed here?".
- Version numbers in migration names order the scripts (`V2_4_0_001__...`).

### Full install vs. upgrade packages
| | Fresh install | Upgrade (delta) |
|---|---|---|
| Used for | A new environment | Moving an existing database from version N−1 to N |
| Content | Everything, from an empty schema | Only the changes since the previous release |
| CI rule | **Test both**: build from empty *and* upgrade from the previous released version |

### Master script pattern (SQL*Plus / SQLcl)
```sql
-- install.sql
WHENEVER SQLERROR EXIT FAILURE ROLLBACK
SET DEFINE OFF               -- stop '&' in strings being treated as substitution variables
SET SQLBLANKLINES ON         -- allow blank lines inside plain SQL statements
SET SERVEROUTPUT ON

PROMPT === Migrations ===
@@migrations/V2_4_0_001__add_status_to_orders.sql

PROMPT === Types and package specifications ===
@@src/types/order_t.tps
@@src/packages/pkg_order_api.pks

PROMPT === Views, package bodies, procedures, functions ===
@@src/views/v_open_orders.vw
@@src/packages/pkg_order_api.pkb

PROMPT === Triggers, grants, jobs ===
@@src/triggers/trg_orders_biu.trg
@@src/grants/grants.sql

PROMPT === Post-checks ===
@@ci/check_invalid.sql
```
- **`@@`** runs a script located **relative to the calling script** (a single `@` is relative to the current working directory) — making the package location-independent.
- **`SET DEFINE OFF`** prevents a string such as `'AT&T'` from prompting for a substitution variable and hanging or breaking an unattended run.
- Each PL/SQL block (package, procedure, trigger) needs its terminating **`/`** on its own line.

### Ordering inside a release (because of dependencies)
1. **Pre-checks** (expected current version, privileges)
2. **Tables, sequences, indexes** (migrations)
3. **Types**
4. **Package specifications**
5. **Views**
6. **Package bodies, procedures, functions**
7. **Triggers**
8. **Grants, synonyms, scheduler jobs**
9. **Data migration / seed data**
10. **Recompile + validation** (no invalid objects)

Specs before bodies so dependents can compile against the interface; the final recompile pass resolves the remaining order effects.

### Idempotency and immutability
- **Code files:** `CREATE OR REPLACE` — safe to rerun.
- **Tables:** guard the DDL (check the dictionary first, or handle the "already exists" error); Oracle 23ai adds `IF [NOT] EXISTS` clauses for many DDL statements.
- **Checksums:** migration tools store a **checksum** of every applied script. **Never edit a migration that has been applied** — the tool will report a mismatch; write a **new** migration instead. The exception is *repeatable/`runOnChange`* files, which are **designed** to be re-applied whenever their checksum changes.
- **Flyway:** versioned (`V<version>__<description>.sql`) migrations run once in version order; repeatable (`R__<description>.sql`) migrations run **after** all pending versioned ones, again each time their checksum changes, in alphabetical order.
- **Liquibase:** change sets identified by `author:id`; tracked in `DATABASECHANGELOG`, with `DATABASECHANGELOGLOCK` preventing concurrent runs.

```sql
--liquibase formatted sql

--changeset dev1:add-status-to-orders
ALTER TABLE orders ADD (status VARCHAR2(20) DEFAULT 'OPEN' NOT NULL);
--rollback ALTER TABLE orders DROP COLUMN status;

--changeset dev1:pkg_order_api runOnChange:true endDelimiter:/ splitStatements:true
CREATE OR REPLACE PACKAGE pkg_order_api AS
  PROCEDURE cancel_order (p_order_id IN NUMBER);
END pkg_order_api;
/
```
*(Liquibase's handling of PL/SQL delimiters and quoting has version-specific quirks — validate your PL/SQL changesets in CI.)*

### Environment-specific values
The artifact must be **identical in every environment**; anything that differs (schema names, tablespaces, database links, job schedules, endpoints) is supplied **at deploy time** via placeholders/properties (Flyway placeholders, Liquibase properties, SQLcl/SQL*Plus substitution variables) or a configuration table — never hard-coded in the scripts, and **secrets never in the package**.

### Rollback planning
| Strategy | Notes |
|---|---|
| **Roll forward** | Ship a corrective release; often the **preferred** approach because data-changing DDL can't be cleanly undone |
| **Rollback scripts** | Undo scripts per change (Liquibase `--rollback`, Flyway undo migrations, custom scripts) — must be **tested** |
| **Redeploy previous version's code** | Easy for **code objects** (re-run the previous tag's PL/SQL), hard for structure/data |
| **Guaranteed restore point / Flashback** | e.g., `CREATE RESTORE POINT before_rel GUARANTEE FLASHBACK DATABASE;` as a safety net around a release window |
| **Backward-compatible ("expand/contract") changes** | Add new columns/objects first, deploy code that works with both shapes, remove old structures in a later release — makes rollback of code harmless |
| **Edition-Based Redefinition** | Code deployed into a new edition; rollback = switch sessions/default edition back (Section 4) |

### Important points to remember
- **Build once, deploy many** — promote the identical artifact; never rebuild per environment.
- Test **both** the fresh install and the upgrade path.
- `SET DEFINE OFF`, `@@`, trailing `/`, and `WHENEVER SQLERROR` are the essential script-hygiene items.
- **Applied migrations are immutable** (new change → new migration); code files are repeatable.
- Always have a **rollback/roll-forward plan** decided *before* the release.
- Keep **environment configuration and secrets out of the artifact**.

### Exercise Questions

**Q1.** Why is it a mistake to rebuild the release from Git separately for each environment?
> **A:** Each rebuild is a new build that could differ (different tool versions, a moved branch, a changed dependency), so production might run something that was never the tested artifact. "Build once, deploy many" guarantees the exact bytes validated in CI/UAT are what reach production, and lets you trace any environment back to one immutable, checksummed, tagged artifact.

**Q2.** A package installed by an unattended deployment hangs waiting for input. The package body contains the literal `'Terms & Conditions'`. What happened and what is the fix?
> **A:** SQL*Plus/SQLcl treat `&` as the start of a **substitution variable** and prompt for a value for `Conditions`; unattended, the run stalls or fails. Put **`SET DEFINE OFF`** at the top of the install script (or use a different `SET DEFINE` character/escape) so `&` is treated literally.

**Q3.** Flyway reports "checksum mismatch" for `V2__add_column.sql` after a developer fixed a typo in it, though the migration was applied in test last week. What should be done?
> **A:** Applied versioned migrations must not be edited — the stored checksum no longer matches. Revert the file to its applied content and put the correction in a **new** migration (e.g., `V3__fix_column.sql`). (Only in an environment where the migration is known never to have been applied could you repair/reset the history — a controlled exception, not the norm.) Code objects avoid this problem by living in **repeatable** files, which are expected to change.

**Q4.** Why must CI test both a fresh install and an upgrade from the previous release?
> **A:** They exercise different paths: a fresh install proves the complete script set builds an empty database, while an upgrade proves the *delta* correctly transforms a database that already holds data and objects (existing rows, constraints, dependent code). A release can pass one and break the other, and production only ever experiences the upgrade path.

**Q5.** Give two reasons roll-forward is often preferred over rollback for database releases.
> **A:** (1) Structural and data changes (dropped columns, transformed data, DML by users after go-live) are often **irreversible or lossy** to undo, whereas a forward fix preserves new data; (2) rollback scripts are seldom tested as thoroughly as deployment scripts and may fail when needed most. Safety nets such as a guaranteed restore point, backward-compatible ("expand/contract") changes and editions reduce the need to roll back at all.

---

## 4. Automated Deployments (CD)

### Continuous Delivery vs. Continuous Deployment
- **Continuous Delivery:** the pipeline automatically deploys to test/staging and makes a **release-ready** artifact; **a person approves** the production deployment.
- **Continuous Deployment:** a change that passes every automated gate is **deployed to production automatically**.
Most database teams choose *delivery with an approval gate* because of the higher cost of a bad database change.

### The deployment pipeline
```
Git tag / merged main
      │
      ▼
  CI: build + test  ──►  immutable artifact  ──►  DEV  ──►  TEST/QA  ──►  UAT/STAGING  ──►  (approval)  ──►  PROD
                                                  auto      auto           auto                              deploy + verify
```
Each stage: **pre-checks → deploy → post-deployment validation → record → notify**.

### What a deployment job does
| Step | Details |
|---|---|
| **Pre-deploy checks** | Correct starting version, connectivity, required privileges, free space; optionally record the **baseline count of invalid objects** |
| **Safety net** | Guaranteed restore point / backup confirmation for production |
| **Apply** | Run the migration tool or master script with the environment's configuration and credentials |
| **Compile & validate** | Recompile; **invalid objects must not exceed the baseline (ideally zero)**; check `USER_ERRORS` |
| **Smoke tests** | A handful of fast checks (key procedures run, critical views return rows, scheduler jobs enabled) |
| **Record** | Store the new version in the history/version table; publish release notes |
| **Notify / monitor** | Tell the team; watch error logs and job states after go-live |

### Tools you may see
Liquibase and Flyway (migration engines), **SQLcl** (`lb`/Liquibase integration and the **`project`** command — `init`, `export`, `stage`, `verify`, `release`, `gen-artifact`, `deploy` — introduced in SQLcl 24.3), plain SQL*Plus/SQLcl master scripts, and orchestrators such as Jenkins, GitLab CI, GitHub Actions, Azure DevOps or OCI DevOps.

### Credentials and privileges
- Use a **dedicated deployment identity**, not a DBA and not a person's account; grant only what deployment needs.
- **Separate the schema owner from the application user:** the owner holds the objects; application users receive only `EXECUTE` on the public API (least privilege, as in the security guides).
- **Proxy authentication** lets the pipeline connect as the deployer while acting as the schema owner (`ALTER USER app_owner GRANT CONNECT THROUGH deployer;`), avoiding sharing the owner's password.
- Keep secrets in the CI secret store or an **Oracle Wallet**; never in scripts or logs.

### The hard part: deploying PL/SQL to a **live** database
Recompiling code that sessions are using has consequences — this is where earlier guide topics come back:

| Symptom | Cause | Mitigation |
|---|---|---|
| **`ORA-04068: existing state of packages has been discarded`** (sometimes with `ORA-04061`/`ORA-04065`) | A package **with package-level state** (variables/cursors) was recompiled while other sessions held its state; their state is invalidated and their **next call fails once** (a retry then succeeds with fresh state) | Avoid package state where possible, deploy in a quiet window, have applications **retry** on 04068, or use editions |
| **Deployment waits/hangs, or `ORA-04021: timeout occurred while waiting to lock object`** | DDL/compilation needs a lock on an object that a running session is currently using | Deploy off-peak; set **`DDL_LOCK_TIMEOUT`** so DDL waits instead of failing immediately; ensure no long-running calls are executing the package |
| Dependent objects become **INVALID** | Changing a spec (or a table) invalidates its dependents | Recompile after deploy (`COMPILE_SCHEMA`/`UTL_RECOMP`); keep spec changes backward compatible |

### Zero-downtime deployments with Edition-Based Redefinition (EBR)
EBR (see the security/new-features guide) is Oracle's built-in answer: deploy the new code into a **new edition** while users keep running the old one.
```sql
CREATE EDITION rel_2_4 AS CHILD OF rel_2_3;
ALTER SESSION SET EDITION = rel_2_4;
-- deploy the release's PL/SQL, editioning views and crossedition triggers here
-- run smoke tests in this session, then:
ALTER DATABASE DEFAULT EDITION = rel_2_4;   -- new connections use the new code
-- rollback = point the default edition back to rel_2_3
```
Tables aren't editioned, so table changes use **editioning views** and **crossedition triggers** to keep both editions consistent during the transition. The trade-off: extra design effort and an "editions-enabled" schema.

### Deployment safety and governance
- **Approval gates** for production; **change windows** where required.
- **Idempotent, re-runnable scripts** so a failed run can be repeated.
- **Concurrency protection:** Liquibase's `DATABASECHANGELOGLOCK` stops two deployments running at once; if a pipeline crashes mid-run the lock can remain and must be **released** (e.g., Liquibase's release-locks command) after confirming no deployment is active.
- **Partial failure:** DDL commits implicitly, so a run that fails midway leaves the database **partly upgraded**. The tool marks which change set failed; recover by fixing forward and re-running (idempotent scripts make this safe) or restoring to the restore point — never assume "all or nothing".
- **Drift control:** forbid manual changes in shared environments; when a hotfix is unavoidable, apply it through the pipeline (hotfix branch → tag) or back-port it immediately. Detect drift by comparing the deployed schema with the repository (Liquibase diff/status, SQLcl, `DBMS_METADATA`-based comparison).
- **Post-deployment monitoring:** invalid objects, `USER_ERRORS`, scheduler job states (`*_SCHEDULER_JOB_RUN_DETAILS`), application error rates.

### Measuring delivery performance (the "DORA" metrics)
Deployment frequency · Lead time for changes · Change failure rate · Time to restore service. Fast, automated, small, well-tested releases improve all four.

### Important points to remember
- Promote the **same artifact** through environments; only **configuration** changes.
- **Continuous Delivery = manual production approval; Continuous Deployment = automatic.**
- **Package state + recompile on a live system → `ORA-04068`**; use retries, quiet windows, or **EBR**.
- `DDL_LOCK_TIMEOUT` lets DDL wait for locks rather than fail instantly.
- Always **validate after deploy** (invalid-object check, smoke tests) and record the deployed version.
- Use **least-privilege deploy identities** (proxy authentication) and keep secrets out of scripts.
- A failed deployment is usually **partially applied** because DDL auto-commits — plan for recovery, don't assume atomicity.

### Exercise Questions

**Q1.** State the difference between Continuous Delivery and Continuous Deployment, and say which usually fits a production database and why.
> **A:** In Continuous Delivery every passing change is automatically prepared and deployed up to staging, but **promotion to production needs a human approval**; in Continuous Deployment the pipeline releases to production **automatically** when all gates pass. Database changes are riskier and harder to undo than stateless application changes, so most organizations keep an approval step (delivery) for production — sometimes moving toward full deployment as tests, monitoring and backward-compatible practices mature.

**Q2.** During business hours a release recompiles a package that has package-level variables. Users start getting `ORA-04068` on their next call and then it disappears. Explain.
> **A:** Recompiling the package **discards its state in every session** that had already instantiated it. A session's next call detects that its package state is no longer valid and raises `ORA-04068` (the call fails once); the session is then reinitialized, so a **retry succeeds**. Remedies: deploy in a low-traffic window, have the application retry on this error, reduce reliance on package state, or use **Edition-Based Redefinition** so existing sessions keep the old edition until they reconnect.

**Q3.** A deployment step hangs, then fails with `ORA-04021`. What is happening and which setting helps?
> **A:** The DDL/compile needs a lock on an object that is currently in use (for example a package being executed by a long-running session), so it waits and eventually times out. Setting **`DDL_LOCK_TIMEOUT`** for the deployment session makes DDL wait a defined number of seconds for the lock instead of failing immediately; also deploy off-peak and check for long-running calls into the object.

**Q4.** A migration tool crashes halfway through a production release. What state is the database in, and how do you recover safely?
> **A:** Because DDL commits implicitly, the changes applied before the crash **remain in place** — the database is partly upgraded, with the failed change set marked in the history table (and possibly a stale deployment lock). Recovery: confirm no deployment is running and release the lock, diagnose the failure, then **fix forward** and re-run (idempotent, tracked change sets skip what already succeeded) — or, if forward repair isn't feasible, restore to the **guaranteed restore point** taken before the release. Never assume the release rolled back automatically.

**Q5.** Someone applied an emergency fix directly in production. What problems does this cause and how should the process handle it?
> **A:** Production now differs from Git and from every other environment (**drift**): the next deployment may overwrite the fix or fail because objects don't match expectations, and the change has no review, test or history. Handle by immediately capturing the change into the repository (hotfix branch → pipeline → tag, or back-porting), running drift detection, and enforcing a policy that all changes flow through the pipeline — with a controlled emergency path that still ends with the change committed and deployed properly.

**Q6.** What does Edition-Based Redefinition give a CD pipeline that ordinary `CREATE OR REPLACE` deployment doesn't?
> **A:** The ability to install and test the new PL/SQL version **alongside** the running one in a separate edition, then cut over new sessions by changing the default edition — **near-zero downtime** and a simple rollback (switch the default back). Ordinary deployment replaces code in place, invalidating dependents and (for stateful packages) disrupting sessions.

---

## 5. Putting It Together — One Change End to End

1. **Develop:** a developer works in a private database on branch `feature/ORD-142`, changes `pkg_order_api`, adds migration `V2_4_0_001__add_status_to_orders.sql` and utPLSQL tests.
2. **Commit & PR:** pushes the branch and opens a pull request → **CI starts automatically**.
3. **CI build:** ephemeral Oracle container → install utPLSQL → run migrations + code from scratch → **compile check (0 invalid)** → warnings-as-errors rules → **unit tests** → **coverage** → reports published.
4. **Review & merge:** reviewers see green checks plus code diff; merge to `main` (protected branch).
5. **Release:** maintainers tag `v2.4.0`; the pipeline builds the **release artifact** (scripts, master script, changelog, release notes, checksum) and stores it. It is tested as both a **fresh install** and an **upgrade from v2.3.x**.
6. **Deploy (CD):** the same artifact is deployed to **DEV → TEST → UAT** automatically, with post-deploy validation each time.
7. **Approval & production:** an approver signs off; the job takes a **guaranteed restore point**, deploys (ideally into a new edition, or in a quiet window), recompiles, runs **smoke tests**, records the version, and notifies the team.
8. **Operate:** monitor invalid objects, errors and jobs; if something is wrong, **roll forward** with a patch release (or roll back via the restore point/edition if required).

### Case Study Questions

**Q1.** A change passes CI but fails on the UAT deployment because a grant is missing. Which earlier practice would have caught this, and where should the fix live?
> **A:** Building the database **from scratch in an ephemeral environment** with the *same* scripts (including grants) and running an upgrade-path test would expose a missing grant if it's required by anything the tests exercise; here the CI database likely used a privileged user that already had the access. The fix belongs **in the repository** (a grants script included in the release package), not as a manual grant in UAT, so every environment is built identically; also run CI under a realistic, least-privilege user.

**Q2.** The release artifact for `v2.4.0` was tested in UAT. A last-minute code tweak is made on `main` before the production deployment. What should happen?
> **A:** The tweak is a **new change**, so it must go through CI and produce a **new version/artifact** (e.g., `v2.4.1`) that is then tested and promoted — production must never receive an artifact different from the one that was tested. Modifying the package "just for production" breaks "build once, deploy many" and traceability.

**Q3.** Coverage falls from 82% to 61% on a pull request, yet all tests pass. Should the pipeline pass?
> **A:** Not if a coverage quality gate is configured: the new code paths are being shipped **untested**, even though existing tests still pass. The gate should fail (or require justification), prompting the author to add tests for the new/changed blocks — coverage is a floor that stops untested code accumulating, even though it can't prove correctness by itself.

---

## Quick Cross-Topic Summary Table

| Topic | Key practice | Classic exam trap |
|---|---|---|
| **Version control** | Git is the source of truth; migrations for tables, repeatable `CREATE OR REPLACE` for code; isolated dev DBs; no secrets | Treating the database as the master; editing applied scripts; CRLF/LF checksum changes |
| **Build & validation (CI)** | Ephemeral DB, build from scratch, compile check, static analysis, utPLSQL, coverage, quality gates | `WHENEVER SQLERROR` **doesn't catch compile errors** — check `USER_ERRORS`/invalid objects; warnings only fail builds with `ERROR:` |
| **Release packaging** | Immutable, versioned artifact; master script; ordered contents; fresh-install *and* upgrade tested; config outside artifact | Rebuilding per environment; `&` without `SET DEFINE OFF`; editing applied migrations; no rollback/roll-forward plan |
| **Automated deployments (CD)** | Promote same artifact through environments; pre/post checks; least-privilege deploy identity; approvals; EBR for low downtime | `ORA-04068` from recompiling stateful packages; DDL lock timeouts (`DDL_LOCK_TIMEOUT`); partial failures because DDL auto-commits; drift from manual hotfixes; Delivery ≠ Deployment |