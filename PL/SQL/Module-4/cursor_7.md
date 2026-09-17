# Topic 7: Final Combined Case Studies — Cursors, Parameters, FOR Loops, FOR UPDATE / WHERE CURRENT OF, and REF CURSOR

> **Syllabus position:** Final item (7 of 7) — the combined practice set
> **Covers:** Every concept from Topics 1–6, mixed freely across realistic, company-style requirements
> **Important:** Unlike Topics 1–6, none of the six case studies below tell you up front which concept(s) they need. That's deliberate — figuring out *which* combination of tools a requirement calls for is the actual skill this whole course has been building toward. Each case study's answer reveals the relevant concepts only inside its own "Concepts to Consider" section, exactly as they would only become clear to you *after* you'd worked through the requirement yourself.

Some of these are also written the way real requirements actually arrive — a little underspecified in places. Where that happens, the answer states an explicit assumption rather than silently picking one, and flags it as something you'd confirm with the business before shipping. That's intentional, and it's a skill in its own right.

---

## Case Study 1: Multi-Worker Warehouse Stock Adjustment Reconciliation

**Business Requirement:**
*"Multiple regional reconciliation workers run the same nightly script concurrently — one instance per warehouse cluster — to process pending stock adjustment requests in the `stock_adjustments` table. For every warehouse, the script must go through all adjustment requests still in `'PENDING'` status, verify the item exists in `inventory` with enough on-hand quantity to support the adjustment, and mark the adjustment `'APPLIED'` while updating the inventory quantity accordingly — or mark it `'REJECTED'` if there isn't enough stock. Since more than one worker process might occasionally target overlapping warehouses due to a configuration error, no two workers should ever end up processing the exact same adjustment request at the same time. If a request is already being handled by another worker, it should simply be skipped and picked up on the next run — it must not cause the whole batch to fail or wait."*

**Assumed schema:** `warehouses(warehouse_id, warehouse_name)`, `stock_adjustments(adjustment_id, warehouse_id, item_id, quantity_change, status)`, `inventory(item_id, warehouse_id, quantity_on_hand)`.

<details>
<summary><b>Click to expand setup script (DDL & Sample Data)</b></summary>

```sql
-- Tables
CREATE TABLE warehouses (
    warehouse_id NUMBER PRIMARY KEY,
    warehouse_name VARCHAR2(100)
);

CREATE TABLE inventory (
    item_id NUMBER,
    warehouse_id NUMBER,
    quantity_on_hand NUMBER,
    PRIMARY KEY (item_id, warehouse_id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id)
);

CREATE TABLE stock_adjustments (
    adjustment_id NUMBER PRIMARY KEY,
    warehouse_id NUMBER,
    item_id NUMBER,
    quantity_change NUMBER,
    status VARCHAR2(20),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id),
    FOREIGN KEY (item_id, warehouse_id) REFERENCES inventory(item_id, warehouse_id)
);

-- Sample Data
INSERT INTO warehouses VALUES (1, 'North Region');
INSERT INTO warehouses VALUES (2, 'South Region');

INSERT INTO inventory VALUES (101, 1, 50);
INSERT INTO inventory VALUES (102, 1, 10);
INSERT INTO inventory VALUES (101, 2, 100);

INSERT INTO stock_adjustments VALUES (1001, 1, 101, -5, 'PENDING');
INSERT INTO stock_adjustments VALUES (1002, 1, 102, -15, 'PENDING'); -- Should reject (-15 + 10 < 0)
INSERT INTO stock_adjustments VALUES (1003, 2, 101, 20, 'PENDING');
COMMIT;
```
</details>

### Requirement Analysis
For every warehouse, iterate its pending adjustments, validate against current inventory, and update both the adjustment's status and the inventory quantity — while guaranteeing safety against overlapping concurrent workers.

### Important Clues
- "for every warehouse... go through all adjustment requests" → a **nested, master-detail** structure (outer: warehouses, inner: that warehouse's pending adjustments).
- "no two workers should ever process the exact same request... skip... must not wait" → a specific, named concurrency behavior: don't block, don't fail, just exclude what's already claimed.
- "mark the adjustment... while updating inventory" → two different tables are written per adjustment; only one of them is the table the cursor is actually iterating.

### Concepts to Consider
Nested cursor FOR loops with a **parameterized inner cursor** (Topics 2 & 4) driving the per-warehouse iteration; `FOR UPDATE SKIP LOCKED` on the inner cursor (Topic 5) for the "don't wait, don't fail, just skip" concurrency requirement; `WHERE CURRENT OF` (Topic 5) to update the adjustment row itself, alongside a normal keyed `UPDATE` for the separate `inventory` table (since `WHERE CURRENT OF` only ever applies to the table the *cursor itself* locked).

### Step-by-Step Approach
1. Outer cursor over all warehouses.
2. Inner cursor, parameterized by warehouse ID, over that warehouse's pending adjustments, declared `FOR UPDATE SKIP LOCKED` — any adjustment already claimed by another worker is simply excluded from this run's active set.
3. For each adjustment, check inventory sufficiency, then `UPDATE ... WHERE CURRENT OF` the adjustment row, and a normal `UPDATE ... WHERE item_id = ... AND warehouse_id = ...` for inventory.
4. Commit once each warehouse's inner loop has finished (safe, since that cursor is already closed by then — see Detailed Explanation).

### Solution
```sql
DECLARE
   CURSOR c_warehouses IS
      SELECT warehouse_id, warehouse_name FROM warehouses;

   CURSOR c_pending_adjustments (p_warehouse_id warehouses.warehouse_id%TYPE) IS
      SELECT adjustment_id, item_id, quantity_change
      FROM stock_adjustments
      WHERE warehouse_id = p_warehouse_id
        AND status = 'PENDING'
      FOR UPDATE SKIP LOCKED;

   v_qty_on_hand inventory.quantity_on_hand%TYPE;
BEGIN
   FOR wh_rec IN c_warehouses LOOP
      DBMS_OUTPUT.PUT_LINE('Warehouse: ' || wh_rec.warehouse_name);

      FOR adj_rec IN c_pending_adjustments(wh_rec.warehouse_id) LOOP

         SELECT quantity_on_hand
         INTO v_qty_on_hand
         FROM inventory
         WHERE item_id = adj_rec.item_id
           AND warehouse_id = wh_rec.warehouse_id;

         IF v_qty_on_hand + adj_rec.quantity_change >= 0 THEN
            UPDATE stock_adjustments
            SET status = 'APPLIED'
            WHERE CURRENT OF c_pending_adjustments;

            UPDATE inventory
            SET quantity_on_hand = quantity_on_hand + adj_rec.quantity_change
            WHERE item_id = adj_rec.item_id
              AND warehouse_id = wh_rec.warehouse_id;
         ELSE
            UPDATE stock_adjustments
            SET status = 'REJECTED'
            WHERE CURRENT OF c_pending_adjustments;
         END IF;

      END LOOP;

      COMMIT;   -- safe here: the inner cursor for this warehouse has already closed
   END LOOP;
END;
/
```

### Detailed Explanation
The `FOR UPDATE SKIP LOCKED` on the inner cursor is what actually implements "no two workers process the same request, and neither one waits or fails" — this is precisely the job-queue pattern from Topic 5. `WHERE CURRENT OF` updates the `stock_adjustments` row without needing to re-select or re-match its key; the `inventory` update, however, is a **different table entirely**, one the cursor never locked, so it needs an ordinary keyed `WHERE` clause — a direct, practical illustration of the Topic 5 rule that `WHERE CURRENT OF` only ever applies to the table the cursor itself is based on.

The `COMMIT` placed right after the inner `FOR` loop, inside the outer loop, is worth pausing on: Topic 5's warning against committing mid-loop was specifically about committing **while a cursor you still intend to fetch from remains open**. Here, by the time `COMMIT` runs, the inner cursor for that warehouse has already been automatically closed (Topic 4's guarantee), and the *next* iteration will open a brand-new inner cursor instance for the next warehouse. So this commit placement is safe — it's a good concrete case of recognizing *why* the earlier warning applied, rather than treating "never commit inside a loop" as an absolute rule.

### Alternative Approach
One might consider also locking the `inventory` row with its own `FOR UPDATE` cursor, to protect against two adjustments for the *same item* being processed concurrently by different workers. This is a reasonable instinct, but it introduces a real risk beyond this syllabus's scope: if different workers could lock `stock_adjustments` and `inventory` rows in different orders, you create a genuine **deadlock risk** (Worker A holds a lock B wants, and vice versa). A full production solution would need a consistent global locking order across all workers, or a single dedicated inventory-update service — flagged here as an important awareness point, not something to implement as part of this exercise.

### Performance / Practical Considerations
`SKIP LOCKED` scales well here specifically because config-error overlaps are expected to be occasional, not the norm — most adjustments will be processed by exactly one worker, with skipped rows being the rare exception, picked up cleanly on the next run.

### Edge Cases
- If an `item_id`/`warehouse_id` combination genuinely doesn't exist in `inventory`, the `SELECT ... INTO v_qty_on_hand` would raise `NO_DATA_FOUND` — a real possibility worth flagging (full exception handling is beyond this syllabus, but a production version would need to catch this and likely reject the adjustment rather than let the whole block fail).
- A warehouse with zero pending adjustments simply produces no inner-loop output — no error, per the standard zero-row cursor behavior from Topic 1.

### Common Mistakes to Watch For
- Using `NOWAIT` instead of `SKIP LOCKED` — as established in Topic 5, `NOWAIT` would fail the **entire** inner cursor's `OPEN` for a warehouse the moment even one of its adjustments is locked by another worker, rather than just excluding that one adjustment.
- Trying to use `WHERE CURRENT OF` against the `inventory` table — invalid, since the cursor only locked `stock_adjustments`.
- Committing *inside* the inner loop, per adjustment, while that same inner cursor is still open and being fetched from — this is the actual dangerous pattern Topic 5, Exercise 6 warned about, and it's easy to mistakenly reach for here if "commit often" is applied without thinking about which cursor is still open at that point.

---

## Case Study 2: Nightly Loyalty Tier Batch with a Durable Dashboard Summary

**Business Requirement:**
*"Run a nightly batch across all active customers to evaluate their trailing-12-month spend and update their loyalty tier (BRONZE / SILVER / GOLD) according to spend thresholds (GOLD: 150,000+, SILVER: 50,000+, else BRONZE). Because customer service reps might have a customer record open for editing at the exact moment the batch runs, the batch must not wait on or interfere with them — any customer currently locked by another session should simply be left for the next run. After the batch completes, log how many customers were upgraded, downgraded, or unchanged, and provide a way for the operations dashboard to pull the most recent run's summary at any time afterward."*

**Assumed schema:** `customers(customer_id, tier, status, last_tier_check)`, `orders(order_id, customer_id, order_date, amount)`, `tier_batch_log(run_id, run_date, upgraded_count, downgraded_count, unchanged_count)` with a `tier_batch_log_seq` sequence.

<details>
<summary><b>Click to expand setup script (DDL & Sample Data)</b></summary>

```sql
-- Tables & Sequence
CREATE TABLE customers (
    customer_id NUMBER PRIMARY KEY,
    tier VARCHAR2(20),
    status VARCHAR2(20),
    last_tier_check DATE,
    -- Columns added for Case Study 6 compatibility
    email VARCHAR2(100),
    created_date DATE
);

CREATE TABLE orders (
    order_id NUMBER PRIMARY KEY,
    customer_id NUMBER,
    order_date DATE,
    amount NUMBER,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE tier_batch_log (
    run_id NUMBER PRIMARY KEY,
    run_date DATE,
    upgraded_count NUMBER,
    downgraded_count NUMBER,
    unchanged_count NUMBER
);

CREATE SEQUENCE tier_batch_log_seq START WITH 1 INCREMENT BY 1;

-- Sample Data
INSERT INTO customers (customer_id, tier, status, last_tier_check) VALUES (1, 'BRONZE', 'ACTIVE', SYSDATE-365);
INSERT INTO customers (customer_id, tier, status, last_tier_check) VALUES (2, 'GOLD', 'ACTIVE', SYSDATE-365);
INSERT INTO customers (customer_id, tier, status, last_tier_check) VALUES (3, 'SILVER', 'ACTIVE', SYSDATE-365);
INSERT INTO customers (customer_id, tier, status, last_tier_check) VALUES (4, 'BRONZE', 'INACTIVE', SYSDATE-365);

-- Customer 1: 60k spend -> upgrades to SILVER
INSERT INTO orders VALUES (101, 1, SYSDATE-10, 60000);

-- Customer 2: 10k spend -> downgrades to BRONZE
INSERT INTO orders VALUES (102, 2, SYSDATE-10, 10000);

-- Customer 3: 160k spend -> upgrades to GOLD
INSERT INTO orders VALUES (103, 3, SYSDATE-10, 160000);
COMMIT;
```
</details>

### Requirement Analysis
Evaluate and update every active customer's tier, tracking outcome counts, then make that summary retrievable **later**, on demand, by a separate dashboard request.

### Important Clues
- "must not wait on or interfere with... reps" → the same skip-if-locked signal as Case Study 1.
- "provide a way for the operations dashboard to pull the most recent run's summary **at any time afterward**" → this phrase is the crux of the whole exercise, addressed directly below.

### Concepts to Consider
`FOR UPDATE SKIP LOCKED` + `WHERE CURRENT OF` (Topic 5) for the batch itself; and — this is the key insight — **a REF CURSOR cannot be the mechanism that makes the summary available "later, on demand."**

### Thought Process — Why a REF CURSOR Alone Doesn't Satisfy This Requirement
A very natural first instinct is: "the batch computes some counts, so just have it `OPEN` a `SYS_REFCURSOR` with those numbers and return it to the dashboard." But think carefully about **when** the dashboard actually asks for this. The batch runs once, overnight, in its own session, and that session ends when the batch finishes. A REF CURSOR is a *pointer to a result set that exists only within the scope of the session/transaction that opened it* — it is **not a persistent, storable object**. There is no way to "save" last night's REF CURSOR and hand it to a dashboard request that comes in hours later, in a completely different session. This is a genuinely common point of confusion, and recognizing it is exactly the kind of practical understanding this topic is testing.

The correct design is therefore **two separate pieces**: the batch persists its summary into a durable table (`tier_batch_log`), and a *separate*, on-demand procedure opens a **fresh** REF CURSOR against that table whenever the dashboard actually asks — which could be immediately after the batch, or days later.

### Step-by-Step Approach
1. Batch: `FOR UPDATE SKIP LOCKED` cursor over active customers; for each, compute trailing-12-month spend, determine new tier, classify the change (upgrade/downgrade/unchanged), and update via `WHERE CURRENT OF`.
2. After the loop, insert one summary row into `tier_batch_log` and commit.
3. A separate, independently-callable procedure opens a `SYS_REFCURSOR` against `tier_batch_log`, returning the most recent row, whenever it's invoked.

### Solution
```sql
-- Part 1: the nightly batch itself
DECLARE
   CURSOR c_customers IS
      SELECT customer_id, tier FROM customers WHERE status = 'ACTIVE' FOR UPDATE SKIP LOCKED;

   v_total_spend   NUMBER;
   v_new_tier      customers.tier%TYPE;
   v_upgraded_ct   NUMBER := 0;
   v_downgraded_ct NUMBER := 0;
   v_unchanged_ct  NUMBER := 0;
BEGIN
   FOR cust_rec IN c_customers LOOP

      SELECT NVL(SUM(amount), 0)
      INTO v_total_spend
      FROM orders
      WHERE customer_id = cust_rec.customer_id
        AND order_date >= ADD_MONTHS(SYSDATE, -12);

      IF v_total_spend >= 150000 THEN
         v_new_tier := 'GOLD';
      ELSIF v_total_spend >= 50000 THEN
         v_new_tier := 'SILVER';
      ELSE
         v_new_tier := 'BRONZE';
      END IF;

      IF v_new_tier = cust_rec.tier THEN
         v_unchanged_ct := v_unchanged_ct + 1;
      ELSIF (v_new_tier = 'GOLD' AND cust_rec.tier IN ('SILVER', 'BRONZE'))
         OR (v_new_tier = 'SILVER' AND cust_rec.tier = 'BRONZE') THEN
         v_upgraded_ct := v_upgraded_ct + 1;
      ELSE
         v_downgraded_ct := v_downgraded_ct + 1;
      END IF;

      UPDATE customers
      SET tier = v_new_tier, last_tier_check = SYSDATE
      WHERE CURRENT OF c_customers;

   END LOOP;

   INSERT INTO tier_batch_log (run_id, run_date, upgraded_count, downgraded_count, unchanged_count)
   VALUES (tier_batch_log_seq.NEXTVAL, SYSDATE, v_upgraded_ct, v_downgraded_ct, v_unchanged_ct);

   COMMIT;
END;
/

-- Part 2: an independently-callable dashboard procedure, invoked whenever, by anyone
CREATE OR REPLACE PROCEDURE get_latest_tier_batch_summary (p_result OUT SYS_REFCURSOR) IS
BEGIN
   OPEN p_result FOR
      SELECT run_id, run_date, upgraded_count, downgraded_count, unchanged_count
      FROM tier_batch_log
      WHERE run_date = (SELECT MAX(run_date) FROM tier_batch_log);
END;
/
```

### Detailed Explanation
Splitting this into two pieces isn't extra ceremony — it's the only design that actually satisfies "at any time afterward." The batch's job ends at `COMMIT`; the dashboard's REF CURSOR is opened **fresh, on demand**, every single time `get_latest_tier_batch_summary` is called, potentially by many different dashboard sessions across many different days, each getting its own independent cursor over the same durable log table.

### Alternative Approach
For a system with very frequent dashboard polling, `tier_batch_log` could instead be a single-row "latest summary" table that gets overwritten each run rather than accumulating history — simpler to query, but loses the ability to see trends across past runs. Keeping a full history (as designed above) costs a `WHERE run_date = (SELECT MAX...)` on every dashboard call instead of a trivial `SELECT *` — a reasonable, minor trade-off in exchange for retaining historical data.

### Edge Cases
- If the batch is somehow run twice in the same session before commit and this needs to be reflected accurately, `tier_batch_log_seq.NEXTVAL` and a fresh `run_date` per execution ensure each run's summary is stored as its own distinct row rather than overwriting a prior one.
- `NVL(SUM(amount), 0)` guards against a customer with zero orders in the trailing 12 months, where `SUM` over no rows would otherwise be `NULL`, which would then fail the numeric threshold comparisons in an unexpected way (`NULL >= 150000` evaluates to `UNKNOWN`, not `FALSE`, but would still never take the `IF` branch — `NVL` makes the intended "zero spend" case explicit and unambiguous rather than relying on `NULL`'s three-valued-logic behavior to coincidentally do the right thing).

### Common Mistakes to Watch For
- Trying to make the *batch itself* return a REF CURSOR the dashboard can query "later" — as explained above, this doesn't work; REF CURSORs don't outlive the session/transaction that opened them.
- Forgetting `NVL` around the `SUM(amount)` and letting a zero-order customer's spend come back `NULL`.
- Using `NOWAIT` instead of `SKIP LOCKED` on the customer cursor — same trap as Case Study 1: it would fail the *entire batch's* `OPEN` the moment even one customer record is locked by a rep, rather than just excluding that one customer for the night.

---

## Case Study 3: Optional-Filter Payroll Grid for a Web Application

**Business Requirement:**
*"The payroll web application needs a single procedure that returns payroll records to display in a browsable grid. Users can optionally filter by branch and/or pay period; leaving a filter blank means 'don't filter by this.' If a branch ID is provided but doesn't exist in the system, the procedure should return an empty result rather than erroring out, since the web layer will show a 'no data' message either way."*

**Assumed schema:** `payroll_records(payroll_id, branch_id, pay_period, employee_id, amount)`.

<details>
<summary><b>Click to expand setup script (DDL & Sample Data)</b></summary>

```sql
-- Tables
CREATE TABLE payroll_records (
    payroll_id NUMBER PRIMARY KEY,
    branch_id NUMBER,
    pay_period VARCHAR2(20),
    employee_id NUMBER,
    amount NUMBER
);

-- Sample Data
INSERT INTO payroll_records VALUES (1, 10, '2023-Q1', 1001, 5000);
INSERT INTO payroll_records VALUES (2, 10, '2023-Q2', 1001, 5200);
INSERT INTO payroll_records VALUES (3, 20, '2023-Q1', 1002, 6000);
INSERT INTO payroll_records VALUES (4, 30, '2023-Q1', 1003, 4500);
COMMIT;
```
</details>

### Requirement Analysis
Return a filterable result set to an external web client, with two independent optional filters, and graceful (not erroring) behavior for an invalid branch ID.

### Important Clues
- "returns payroll records to display in a browsable grid" from a "web application" → an external client needs to consume a result set → REF CURSOR.
- "optionally filter... leaving a filter blank means don't filter" → the NULL-means-no-filter pattern.
- "If a branch ID is provided but doesn't exist... return an empty result rather than erroring out" → this sentence is doing more work than it first appears, addressed below.

### Concepts to Consider
`SYS_REFCURSOR` as an `OUT` parameter (Topic 6), combined with the optional-filter pattern (Topics 2 & 6). The interesting part of this case study isn't new syntax — it's recognizing that **the requirement's error-handling clause may already be satisfied by the filter pattern itself**, without adding anything extra.

### Thought Process — Don't Over-Engineer What's Already Handled
A natural first instinct on reading "if a branch ID is provided but doesn't exist, return an empty result rather than erroring out" is to reach for extra validation: run a `SELECT COUNT(*) INTO v_count FROM branches WHERE branch_id = p_branch_id`, and branch on whether `v_count = 0` before deciding what to do. But pause and ask: **what does the plain filter condition already do** if `p_branch_id` doesn't match any row? `WHERE (branch_id = p_branch_id OR p_branch_id IS NULL)` — when `p_branch_id` is a non-NULL value that simply doesn't exist in `payroll_records`, the `branch_id = p_branch_id` comparison matches zero rows, and since `p_branch_id IS NULL` is also false in this case, the whole condition is false for every row. The query naturally returns **an empty result set** — exactly what was asked for — with no extra logic needed at all. Adding a separate validation step here wouldn't fix a bug; it would just be unnecessary complexity solving a problem the existing logic never had. Recognizing when a requirement's edge case is *already* handled by your existing design, rather than reflexively adding more code, is as much a part of "thinking like a developer" as knowing when you *do* need to add something.

### Solution
```sql
CREATE OR REPLACE PROCEDURE get_payroll_records (
   p_branch_id  IN payroll_records.branch_id%TYPE  DEFAULT NULL,
   p_pay_period IN payroll_records.pay_period%TYPE DEFAULT NULL,
   p_result     OUT SYS_REFCURSOR
) IS
BEGIN
   OPEN p_result FOR
      SELECT payroll_id, branch_id, pay_period, employee_id, amount
      FROM payroll_records
      WHERE (branch_id  = p_branch_id  OR p_branch_id  IS NULL)
        AND (pay_period = p_pay_period OR p_pay_period IS NULL);
END;
/
```

### Detailed Explanation
Worth explicitly noting: this procedure isn't built with dynamic SQL at all — it's a single, fixed `SELECT` statement that happens to reference `p_branch_id` and `p_pay_period` directly. Oracle automatically treats PL/SQL variables referenced inside a static SQL statement as bind values — there is no string concatenation happening here, so the SQL-injection caution from Topic 6 (which specifically applies to *dynamically built SQL text*) doesn't even come into play in this version. It's easy to conflate "using bind variables" (relevant to dynamic SQL) with "referencing PL/SQL variables in an ordinary static query" (always safe, no special caution needed) — this case study is a clean example of the latter.

### Alternative Approach
If a third or fourth optional filter were added later and some of those filters needed genuinely different query shapes for performance reasons (not just an added `AND` condition), dynamic SQL with explicit `USING` bind values (as in Topic 6, Exercise 7) would become the more scalable choice. For exactly two straightforward equality filters, though, the static form above is simpler and equally correct — reaching for dynamic SQL here would be solving a problem this requirement doesn't actually have.

### Edge Cases
- Both filters `NULL` → returns every payroll record, which matches the stated behavior ("leaving a filter blank means don't filter by this") applied to both filters at once.
- A valid branch ID with no pay period specified → returns all pay periods for that branch, as expected.

### Common Mistakes to Watch For
- Adding unnecessary `COUNT(*)` validation logic for the invalid-branch case, as discussed above.
- Using plain `WHERE branch_id = p_branch_id AND pay_period = p_pay_period` without the `OR ... IS NULL` companion clauses — this is the direct NULL-comparison trap from Topic 2: leaving either filter blank would return **zero rows** instead of "don't filter by this," since `column = NULL` is never `TRUE`.

---

## Case Study 4: Subscription Renewal Engine with Failure-History Suspension

**Business Requirement:**
*"Every night, the billing system should attempt to renew all subscriptions due within the next 7 days. For each due subscription, check its last 3 billing attempts; if all 3 of the most recent attempts failed, mark the subscription 'SUSPENDED' instead of attempting renewal. Otherwise, simulate a renewal charge and mark it 'RENEWED' or 'PAYMENT_FAILED' accordingly. Because customers can also manually trigger a renewal from the mobile app at the same time this batch runs, the batch should never process a subscription that's currently being renewed by the app — skip it for tonight, it'll be retried tomorrow."*

**Assumed schema:** `subscriptions(subscription_id, customer_id, renewal_date, status)`, `billing_attempts(attempt_id, subscription_id, attempt_date, result)` where `result` is `'SUCCESS'` or `'FAILED'`.

<details>
<summary><b>Click to expand setup script (DDL & Sample Data)</b></summary>

```sql
-- Tables
CREATE TABLE subscriptions (
    subscription_id NUMBER PRIMARY KEY,
    customer_id NUMBER,
    renewal_date DATE,
    status VARCHAR2(20)
);

CREATE TABLE billing_attempts (
    attempt_id NUMBER PRIMARY KEY,
    subscription_id NUMBER,
    attempt_date DATE,
    result VARCHAR2(20),
    FOREIGN KEY (subscription_id) REFERENCES subscriptions(subscription_id)
);

-- Sample Data
-- Sub 1: Due soon, 3 recent failed attempts -> Should suspend
INSERT INTO subscriptions VALUES (1, 101, SYSDATE + 2, 'ACTIVE');
INSERT INTO billing_attempts VALUES (101, 1, SYSDATE - 3, 'FAILED');
INSERT INTO billing_attempts VALUES (102, 1, SYSDATE - 2, 'FAILED');
INSERT INTO billing_attempts VALUES (103, 1, SYSDATE - 1, 'FAILED');

-- Sub 2: Due soon, only 2 failed attempts -> Proceeds to normal renewal attempt
INSERT INTO subscriptions VALUES (2, 102, SYSDATE + 5, 'ACTIVE');
INSERT INTO billing_attempts VALUES (104, 2, SYSDATE - 2, 'FAILED');
INSERT INTO billing_attempts VALUES (105, 2, SYSDATE - 1, 'FAILED');

-- Sub 3: Not due yet -> Will be skipped
INSERT INTO subscriptions VALUES (3, 103, SYSDATE + 15, 'ACTIVE');

-- Sub 4: Due soon, mixed history -> Proceeds to normal renewal attempt
INSERT INTO subscriptions VALUES (4, 104, SYSDATE + 1, 'ACTIVE');
INSERT INTO billing_attempts VALUES (106, 4, SYSDATE - 3, 'SUCCESS');
INSERT INTO billing_attempts VALUES (107, 4, SYSDATE - 2, 'FAILED');
INSERT INTO billing_attempts VALUES (108, 4, SYSDATE - 1, 'FAILED');
COMMIT;
```
</details>

### Requirement Analysis
Iterate due subscriptions, inspect each one's recent billing history to decide between suspension and a fresh renewal attempt, while avoiding conflicts with the mobile app's own concurrent renewal attempts.

### Important Clues
- "check its last 3 billing attempts" → a **nested, parameterized inner cursor** per subscription (billing history filtered by subscription ID).
- "should never process a subscription... currently being renewed by the app — skip it" → the same skip-if-locked pattern as Case Studies 1 and 2.
- "if **all 3** of the most recent attempts failed" → a precise counting condition, and an implicit question about what happens with *fewer than 3* historical attempts (addressed as an assumption below).

### Concepts to Consider
`FOR UPDATE SKIP LOCKED` + `WHERE CURRENT OF` on the subscriptions cursor (Topic 5); a nested, parameterized cursor for billing history (Topics 2 & 4), combined with a manually tracked counter across the inner loop's iterations (the same technique used for the "detect zero qualifying rows" problem in Topic 4, generalized here to "count how many of up to 3 fetched rows are failures").

### Possible Interpretation / Assumption
The requirement says "if all 3 of the most recent attempts failed" — but what if a subscription only has 1 or 2 historical attempts total, all failed? The wording implies there must genuinely be 3 to evaluate against; a subscription with, say, only 1 prior failed attempt hasn't accumulated the pattern the rule is meant to catch. **Assumption made here:** suspension requires *exactly* 3 recorded attempts, all of which failed; anything with fewer than 3 total attempts proceeds to a normal renewal attempt regardless of how many of those few attempts failed. This is a genuine ambiguity in the original wording and should be confirmed with the billing team before shipping to production — it's entirely plausible they'd want "2 attempts, both failed" to also suspend, and the requirement as given doesn't say either way.

### Solution
```sql
DECLARE
   CURSOR c_due_subscriptions IS
      SELECT subscription_id, customer_id
      FROM subscriptions
      WHERE renewal_date <= SYSDATE + 7
        AND status = 'ACTIVE'
      FOR UPDATE SKIP LOCKED;

   CURSOR c_recent_attempts (p_subscription_id subscriptions.subscription_id%TYPE) IS
      SELECT result
      FROM billing_attempts
      WHERE subscription_id = p_subscription_id
      ORDER BY attempt_date DESC
      FETCH FIRST 3 ROWS ONLY;

   v_failed_count NUMBER;
   v_total_count  NUMBER;
BEGIN
   FOR sub_rec IN c_due_subscriptions LOOP

      v_failed_count := 0;
      v_total_count  := 0;

      FOR attempt_rec IN c_recent_attempts(sub_rec.subscription_id) LOOP
         v_total_count := v_total_count + 1;
         IF attempt_rec.result = 'FAILED' THEN
            v_failed_count := v_failed_count + 1;
         END IF;
      END LOOP;

      IF v_total_count = 3 AND v_failed_count = 3 THEN
         UPDATE subscriptions SET status = 'SUSPENDED' WHERE CURRENT OF c_due_subscriptions;
      ELSE
         IF MOD(sub_rec.subscription_id, 10) != 0 THEN   -- placeholder for "charge succeeded"
            UPDATE subscriptions
            SET status = 'ACTIVE', renewal_date = ADD_MONTHS(SYSDATE, 1)
            WHERE CURRENT OF c_due_subscriptions;
         ELSE
            UPDATE subscriptions SET status = 'PAYMENT_FAILED' WHERE CURRENT OF c_due_subscriptions;
         END IF;
      END IF;

   END LOOP;

   COMMIT;
END;
/
```

### Detailed Explanation
`v_total_count` and `v_failed_count` are reset to zero at the **start of every outer iteration**, before the inner loop runs — exactly the same discipline required for the "reset the flag per outer iteration" pattern in Topic 4, Exercise 7; forgetting this reset would let one subscription's failure history bleed into the next subscription's evaluation. The `v_total_count = 3` check directly encodes the stated assumption — a subscription with only 1 or 2 historical attempts, however many of them failed, will not match this condition and falls through to a normal renewal attempt instead.

### Alternative Approach
The failure-counting logic could instead be expressed as a single aggregate subquery — `SELECT COUNT(*) INTO v_total_count FROM (SELECT result FROM billing_attempts WHERE subscription_id = ... ORDER BY attempt_date DESC FETCH FIRST 3 ROWS ONLY) WHERE result = 'FAILED'` plus a second count for the total — collapsing the nested cursor into two scalar queries. This avoids the inner FOR loop entirely, at the cost of running two separate queries per subscription instead of one cursor with a simple counting loop; either is reasonable, and the nested-cursor version arguably reads more naturally to someone tracing the actual business rule ("look at each of the last 3 attempts and count the failures") step by step.

### Edge Cases
- A subscription due for renewal with **zero** billing history at all (a brand-new subscription) → `v_total_count` stays `0`, the suspension condition is false, and it proceeds to a normal renewal attempt, which is the sensible behavior for a first-time renewal.
- `FETCH FIRST 3 ROWS ONLY` combined with `ORDER BY attempt_date DESC` correctly returns the 3 *most recent* attempts regardless of how many total attempts exist historically — no special handling needed for subscriptions with a long attempt history.

### Common Mistakes to Watch For
- Checking `v_failed_count = 3` without also checking `v_total_count = 3` — this would incorrectly treat a subscription with, say, exactly 1 historical attempt (which failed) as having "3 failures," since a naive `IF v_failed_count = 3` would simply never trigger for fewer than 3 failures, technically avoiding the bug by accident in *this specific* direction, but the omission is still fragile and doesn't correctly encode the intended "must have 3 to evaluate" rule if the comparison were ever changed (e.g., to `>= 2`).
- Forgetting to reset `v_failed_count`/`v_total_count` inside the outer loop.
- Using `NOWAIT` instead of `SKIP LOCKED` for the subscriptions cursor — same trap as the earlier case studies.

---

## Case Study 5: Sales Reporting — A BI Dashboard Feed and a Cross-Module Notification Handoff

**Business Requirement:**
*"The sales operations team needs two things from the same underlying data: (1) A BI dashboard procedure that lets analysts pull sales transactions filtered by any combination of region, product category, and date range. (2) An automated weekly process that, for each sales region, identifies the top-spending customers (those who spent more than a set threshold in the past week), and needs that list handed off to the notification module in a form it can iterate over to build each region manager's email."*

**Assumed schema:** `sales(sale_id, region, product_category, sale_date, amount, customer_id)`.

<details>
<summary><b>Click to expand setup script (DDL & Sample Data)</b></summary>

```sql
-- Tables
CREATE TABLE sales (
    sale_id NUMBER PRIMARY KEY,
    region VARCHAR2(50),
    product_category VARCHAR2(50),
    sale_date DATE,
    amount NUMBER,
    customer_id NUMBER
);

-- Sample Data
-- East Region
INSERT INTO sales VALUES (1, 'East', 'Electronics', SYSDATE - 2, 12000, 101);
INSERT INTO sales VALUES (2, 'East', 'Furniture', SYSDATE - 5, 5000, 101); -- Cust 101 total weekly: 17000 (qualifies for > 10000 threshold)
INSERT INTO sales VALUES (3, 'East', 'Electronics', SYSDATE - 3, 2000, 104); -- Cust 104 total weekly: 2000 (doesn't qualify)

-- West Region
INSERT INTO sales VALUES (4, 'West', 'Electronics', SYSDATE - 1, 11000, 102); -- Cust 102 total weekly: 11000 (qualifies)
INSERT INTO sales VALUES (5, 'West', 'Clothing', SYSDATE - 10, 15000, 103); -- Cust 103 total weekly: 0 (outside 7 days window, doesn't qualify)
COMMIT;
```
</details>

### Requirement Analysis
Two related but distinct deliverables sharing the same underlying table: a flexible multi-filter feed for an external BI tool, and an internal per-region top-customer list that must cross into a separate module.

### Important Clues
- "(1)... BI dashboard procedure... filtered by any combination of region, product category, and date range" → optional multi-filter REF CURSOR, same shape as Case Study 3 and Topic 6, Exercise 7, but with three filters instead of two.
- "(2)... for each sales region... needs that list **handed off to the notification module**... to iterate over" → a *different* motivation for REF CURSOR than "external client consumption": here, the consumer is another **internal PL/SQL module**, and the defining need is crossing a subprogram boundary (Topic 6's purpose #1), not necessarily serving an external client (purpose #2).

### Concepts to Consider
Two independent REF CURSOR-returning procedures, for two different reasons: one because the consumer is external (BI tool), one because the consumer is a separate internal module. Also a nested cursor FOR loop (per region) driving calls to the second procedure — a genuine combination of "REF CURSOR as the standard external-facing mechanism" and "REF CURSOR as an internal module hand-off mechanism," both from the same topic but motivated differently.

### Step-by-Step Approach
1. Build `get_sales_transactions`, a `SYS_REFCURSOR`-returning procedure with three independent optional filters, following the same `(condition OR :filter IS NULL)` pattern used throughout this course.
2. Build `get_top_customers_for_region`, a second `SYS_REFCURSOR`-returning procedure taking a region and a threshold, aggregating the past week's spend per customer.
3. A driving block loops over all distinct regions, calling `get_top_customers_for_region` for each one and handing the resulting cursor onward (here, simulated as a manual fetch, standing in for what would really be a call into the actual notification module).

### Solution
```sql
-- Part 1: BI dashboard feed
CREATE OR REPLACE PROCEDURE get_sales_transactions (
   p_region    IN sales.region%TYPE           DEFAULT NULL,
   p_category  IN sales.product_category%TYPE DEFAULT NULL,
   p_date_from IN DATE                        DEFAULT NULL,
   p_date_to   IN DATE                        DEFAULT NULL,
   p_result    OUT SYS_REFCURSOR
) IS
BEGIN
   OPEN p_result FOR
      SELECT sale_id, region, product_category, sale_date, amount, customer_id
      FROM sales
      WHERE (region           = p_region     OR p_region     IS NULL)
        AND (product_category = p_category   OR p_category   IS NULL)
        AND (sale_date        >= p_date_from OR p_date_from  IS NULL)
        AND (sale_date        <= p_date_to   OR p_date_to    IS NULL);
END;
/

-- Part 2: per-region top-customer feed for the notification module
CREATE OR REPLACE PROCEDURE get_top_customers_for_region (
   p_region    IN sales.region%TYPE,
   p_threshold IN NUMBER,
   p_result    OUT SYS_REFCURSOR
) IS
BEGIN
   OPEN p_result FOR
      SELECT customer_id, SUM(amount) AS weekly_spend
      FROM sales
      WHERE region = p_region
        AND sale_date >= SYSDATE - 7
      GROUP BY customer_id
      HAVING SUM(amount) > p_threshold
      ORDER BY weekly_spend DESC;
END;
/

-- Driving block: for each region, get its top customers and hand off to the notification module
DECLARE
   CURSOR c_regions IS SELECT DISTINCT region FROM sales;
   v_top_customers SYS_REFCURSOR;
   v_customer_id   sales.customer_id%TYPE;
   v_weekly_spend  NUMBER;
BEGIN
   FOR region_rec IN c_regions LOOP
      get_top_customers_for_region(region_rec.region, 10000, v_top_customers);

      -- Stands in for handing v_top_customers to a real notify_region_manager(...) procedure;
      -- shown here as a manual fetch to demonstrate the cursor is fully usable at this point.
      LOOP
         FETCH v_top_customers INTO v_customer_id, v_weekly_spend;
         EXIT WHEN v_top_customers%NOTFOUND;
         DBMS_OUTPUT.PUT_LINE('Region ' || region_rec.region ||
            ': notify about customer ' || v_customer_id || ' (spent ' || v_weekly_spend || ')');
      END LOOP;
      CLOSE v_top_customers;
   END LOOP;
END;
/
```

### Detailed Explanation
Part 1 is a straightforward extension of the multi-filter pattern already seen twice in this course — nothing conceptually new, just a third filter added the same way. Part 2 is the more interesting design decision: it would have been entirely possible to skip a separate procedure and just write the top-customer query as a plain parameterized cursor FOR loop directly inside the driving block. The reason a REF CURSOR is the better fit here is squarely in the wording: "handed off **to the notification module**" implies a genuinely separate, independently-maintained piece of code needs to *receive* this data as an argument — and that's exactly the "passing a cursor across a subprogram boundary" use case from Topic 6, distinct from (though just as valid as) "returning to an external client."

### Alternative Approach
If the "notification module" were actually just a few lines of logic that could reasonably live right there in the driving block rather than a separately maintained module, a plain nested parameterized cursor FOR loop (no REF CURSOR at all) would be simpler and entirely sufficient — this is a direct application of Topic 3's decision framework ("does this result set genuinely need to leave this subprogram?"). The REF CURSOR is justified here specifically because the requirement frames the notification logic as its own module, not because "REF CURSOR is generally better."

### Performance / Practical Considerations
Both dashboard filters and the top-customer aggregation reference PL/SQL variables directly in static SQL (no dynamic SQL string-building), so — as in Case Study 3 — there's no SQL-injection surface here at all, and no special bind-variable ceremony beyond what's already naturally happening.

### Edge Cases
- A region with no customers over the weekly threshold → `get_top_customers_for_region` still opens successfully and simply returns zero rows; the driving block's fetch loop exits immediately for that region with no error, and no notification content is generated for it — worth confirming with the business whether "no top customers this week" should itself trigger any message to the region manager, or genuinely produce nothing (the requirement as given doesn't say).

### Common Mistakes to Watch For
- Trying to consume `v_top_customers` with a cursor FOR loop instead of a manual fetch — not supported for a REF CURSOR variable, per Topic 6.
- Forgetting to `CLOSE v_top_customers` on each outer iteration before the next region's call reuses the same variable — while reusing the *variable* across iterations is fine (each `get_top_customers_for_region` call opens it fresh against a new query), leaving a previous iteration's cursor un-closed before the variable is reassigned is still a resource-management lapse worth avoiding as a matter of discipline.

---

## Case Study 6: Duplicate Customer Record Cleanup (Ambiguous Requirement)

**Business Requirement:**
*"Marketing has found that some customers have duplicate records in the `customers` table, apparently from being entered more than once over the years. They want a cleanup script that finds these duplicates and removes the extra copies, keeping just one record per customer so email campaigns don't go out multiple times to the same person. This needs to be safe to run even if someone is actively viewing or editing a customer record in the admin tool at the same time."*

**Assumed schema:** `customers(customer_id, email, created_date, ...)`.

<details>
<summary><b>Click to expand setup script (Sample Data)</b></summary>

> **Note:** This case study uses the `customers` table created in Case Study 2. If you haven't run that script, you'll need to create the table first. We'll add some duplicate data here.

```sql
-- Sample Data (Adds duplicate records for the cleanup scenario)
-- 'jdoe@example.com' has 3 records. ID 5 is the oldest and should be kept.
INSERT INTO customers (customer_id, email, created_date, status) VALUES (5, 'jdoe@example.com', SYSDATE - 100, 'ACTIVE');
INSERT INTO customers (customer_id, email, created_date, status) VALUES (6, 'jdoe@example.com', SYSDATE - 50, 'ACTIVE');
INSERT INTO customers (customer_id, email, created_date, status) VALUES (7, 'jdoe@example.com', SYSDATE - 10, 'ACTIVE');

-- 'asmith@example.com' has 2 records. ID 8 is the oldest and should be kept.
INSERT INTO customers (customer_id, email, created_date, status) VALUES (8, 'asmith@example.com', SYSDATE - 200, 'ACTIVE');
INSERT INTO customers (customer_id, email, created_date, status) VALUES (9, 'asmith@example.com', SYSDATE - 5, 'ACTIVE');

-- 'unique@example.com' has 1 record. It should be skipped by the script.
INSERT INTO customers (customer_id, email, created_date, status) VALUES (10, 'unique@example.com', SYSDATE - 30, 'ACTIVE');
COMMIT;
```
</details>

### Requirement Analysis
This requirement is realistic precisely because it's under-specified in two important ways, and a professional response has to notice and handle that rather than guess silently.

### Important Clues — and the Ambiguity They Expose
- "duplicate records... apparently from being entered more than once" → duplicates by **what**? The requirement never states which field(s) identify "the same person." Email is the most natural proxy in this context (it's directly tied to "email campaigns don't go out multiple times"), but a real system might consider phone number, a national ID, or a combination of name and address instead.
- "keeping just one record per customer" → **which** one? Oldest by creation date? Most recently updated? The one with the most complete data? The requirement doesn't say.
- "safe to run even if someone is actively... editing a customer record" → this part is unambiguous and points directly at a concurrency-safety concept.

### Possible Interpretation / Assumptions (State Before Coding)
For this exercise, the following assumptions are made explicit, exactly as a developer should do before writing code against an under-specified requirement — and exactly as should be confirmed with marketing before running this in production:
1. **"Duplicate" = same `email` address.** Reasonable given the stated motivation (avoiding duplicate email sends), though a real investigation might reveal customers with the same person but different emails (not caught by this definition) or coincidentally shared emails between different people (a household email, over-matched by this definition). Flagged as a risk to confirm.
2. **"Keep" = the earliest-created record** (lowest `created_date`), on the reasoning that the original record is more likely to have accumulated order/loyalty history tied to it, making it the safer one to preserve. This is an assumption, not a stated fact — a real implementation might instead need to *merge* the most complete/up-to-date fields from all duplicates into the kept record rather than simply deleting the others outright, which is a meaningfully bigger task outside this exercise's cursor-focused scope.

### Concepts to Consider
`FOR UPDATE` (Topic 5) to protect against the admin tool's concurrent edits; `WHERE CURRENT OF` combined with `DELETE` (Topic 5); and a manually tracked "compare this row to the previous one" variable across loop iterations — a generalization of the running-flag technique from Topic 4, this time used to detect group boundaries in an ordered result set rather than to detect "did anything match at all."

### Step-by-Step Approach
1. Restrict the cursor to only customers whose email appears more than once (a `WHERE email IN (subquery with GROUP BY ... HAVING COUNT(*) > 1)`), ordered by `email, created_date` — so that, within each email group, the very first row encountered is the oldest one.
2. Declare the cursor `FOR UPDATE`, addressing the concurrency-safety requirement.
3. Track the last-seen email in a variable across loop iterations. The first row for a given email (where the tracked variable doesn't yet match) is the one to keep; every subsequent row sharing that same email is a duplicate to delete via `WHERE CURRENT OF`.

### Solution
```sql
DECLARE
   CURSOR c_dupe_candidates IS
      SELECT customer_id, email, created_date
      FROM customers
      WHERE email IN (
         SELECT email FROM customers GROUP BY email HAVING COUNT(*) > 1
      )
      ORDER BY email, created_date
      FOR UPDATE;

   v_last_email customers.email%TYPE := NULL;
BEGIN
   FOR cust_rec IN c_dupe_candidates LOOP
      IF cust_rec.email = v_last_email THEN
         -- not the first (oldest) record for this email -> duplicate, remove it
         DELETE FROM customers WHERE CURRENT OF c_dupe_candidates;
      ELSE
         -- first time seeing this email in this ordered pass -> this is the one we keep
         v_last_email := cust_rec.email;
      END IF;
   END LOOP;

   COMMIT;
END;
/
```

### Detailed Explanation
The `ORDER BY email, created_date` is what makes the "keep the oldest" rule work with such a simple comparison — because rows arrive grouped by email and sorted oldest-first within each group, the very first row seen for any given email is guaranteed to be the one to keep, and `v_last_email` only needs to be updated on that first sighting. It's worth explicitly confirming a reasonable question here: does `FOR UPDATE` disturb this intended fetch order? No — `FOR UPDATE` locks the entire active set at `OPEN` (per Topic 5), but it does not override or interfere with the `ORDER BY` governing the sequence in which rows are subsequently fetched; the locking and the fetch ordering are independent concerns.

### Alternative Approach
Rather than the running-variable comparison, the cursor could instead be written to select *only* the rows to delete directly, using an analytic/ranking technique to identify "everything except the earliest row per email group" in the query itself (a `ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_date)` filtered to `> 1`, for instance) — this would remove the need for the `v_last_email` tracking variable and the `IF` branch entirely, since only rows meant for deletion would ever be fetched. This is a legitimate, arguably cleaner alternative; it's deliberately not the primary solution here, since it relies on an analytic SQL feature outside this syllabus's PL/SQL cursor topics, but it's worth knowing such an option exists — sometimes the cleanest fix to a PL/SQL-loop problem is to push more of the logic into the SQL itself, a theme that has come up repeatedly throughout this course (Topic 1's warehouse example, Topic 4's stockout example) in the "filter what you don't need at the SQL level" direction.

### Performance / Practical Considerations
Since this is realistically a manually-triggered, one-off cleanup rather than a recurring high-frequency job, the default `FOR UPDATE` (wait indefinitely) is a reasonable, low-risk choice — a brief wait behind an admin's in-progress edit is tolerable for a script like this. If it were instead expected to run frequently or on a schedule, `SKIP LOCKED` would be the more defensible choice, on the reasoning that a duplicate skipped this run is still a duplicate next run — not urgently harmful to leave for a later pass, and skipping avoids blocking admin staff at all. Both are genuinely defensible; which one is "correct" depends on operational context the requirement doesn't specify — another point worth raising with whoever owns this script's schedule.

### Edge Cases
- A customer with a completely unique email (no duplicates at all) never appears in the cursor's active set at all, since the `WHERE email IN (...)` subquery excludes it up front — no risk of accidentally deleting a non-duplicate record.
- Three or more records sharing the same email are handled correctly by this logic: the oldest is kept (first sighting resets `v_last_email`), and *every* subsequent row in that same group — not just the second one — matches `cust_rec.email = v_last_email` and gets deleted, since `v_last_email` is never reset again until a genuinely different email is encountered.

### Common Mistakes to Watch For
- Silently picking a "duplicate" definition and a "which one to keep" rule without stating them anywhere — exactly the trap this case study is designed to catch. A real deliverable should state these assumptions in a comment or accompanying documentation, not bury the decision invisibly inside the `WHERE`/`ORDER BY` clauses.
- Forgetting the `ORDER BY email, created_date` — without it, "the first row fetched per email" would be arbitrary rather than reliably the oldest one, silently breaking the intended "keep the oldest" rule.
- Using `DELETE FROM customers WHERE email = cust_rec.email AND created_date > v_first_created_date` (matching by value) instead of `WHERE CURRENT OF` — besides being unnecessary extra state to track, it risks deleting the wrong rows if `created_date` isn't perfectly unique within a duplicate group, whereas `WHERE CURRENT OF` is unambiguous by construction, since it targets the cursor's exact current physical row regardless of any column's value.

---

# Course Wrap-Up

Across these six case studies, every syllabus topic has now appeared multiple times, in combination, without being labeled — which was the point. A quick map of what showed up where:

| Topic | Where it appeared |
|---|---|
| Cursor lifecycle, `%TYPE`/`%ROWTYPE` (Topic 1) | Every case study — the foundation none of the others work without. |
| Parameters (Topic 2) | Case Studies 1, 4, 5 (nested cursors), and the NULL-comparison trap surfaced again in Case Study 3. |
| Decision-making framework (Topic 3) | Explicitly invoked in Case Study 5's "does this need to leave the subprogram?" question. |
| Cursor FOR loops (Topic 4) | Every case study's outer/inner iteration; the "reset a flag per outer iteration" pattern reappeared in Case Study 4. |
| `FOR UPDATE` / `WHERE CURRENT OF` (Topic 5) | Case Studies 1, 2, 4, and 6 — including the `NOWAIT` vs. `SKIP LOCKED` distinction reinforced three separate times. |
| REF CURSOR (Topic 6) | Case Studies 2, 3, and 5 — including the important "REF CURSORs aren't persistent" lesson in Case Study 2, and the two different real motivations for reaching for one in Case Study 5. |

If you were able to work through these without needing the concept names spelled out — or even just recognized *which* concepts were candidates before reading the answer's "Concepts to Consider" section — that's the actual goal of this whole syllabus achieved: not memorizing six pieces of syntax, but recognizing, from a business requirement alone, which combination of tools the problem is actually asking for.