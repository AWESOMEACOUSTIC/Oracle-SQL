# Module 3, Topic 6: Function Result Cache

---

## 1. What Is the Function Result Cache?

The **Function Result Cache** is a feature that lets Oracle **store (cache) the return value of a function call**, keyed by its input parameters, so that if the **same function is called again with the same arguments**, Oracle can **skip re-executing the function body entirely** and just hand back the previously computed result instantly from memory (the System Global Area — SGA — a shared memory region on the database server).

You enable this by adding the `RESULT_CACHE` clause to a function definition.

---

## 2. Why Does This Exist? What Problem Does It Solve?

Consider a function like:

```sql
get_exchange_rate(p_currency_code)
```

Suppose this function does something moderately expensive — queries a large `exchange_rates` table, or does some computation — and it's called **thousands of times per minute** across the company:
- Every report that shows a monetary value calls it to convert currency.
- Every order calculation calls it.
- Every dashboard refresh calls it.

But here's the key business insight: **exchange rates only change a few times a day at most.** So the vast majority of these thousands of calls are asking the exact same question ("what's the USD rate right now?") and getting the exact same answer, over and over, re-running the same query against the same table every single time — pure wasted work.

This is precisely the class of problem the **Function Result Cache** solves: **expensive, pure (side-effect-free), frequently-repeated function calls where the underlying data doesn't change often relative to how often the function is called.** Cache the answer once, and every subsequent identical call becomes essentially free (a fast in-memory lookup) — until the underlying data changes and the cache is automatically invalidated.

---

## 3. Why Is It Used? (The Business Case)

- **Reference/lookup data functions** — currency rates, tax rates, configuration values, country/region lookups — data that changes rarely but is queried extremely often.
- **Expensive computations with a small set of possible inputs** — e.g., a function computing a complex eligibility score that only depends on a handful of distinct category codes.
- **High-traffic systems** — anywhere the same function with the same parameters gets called an enormous number of times (dashboards, high-volume APIs, reporting engines) — caching dramatically reduces database load and improves response time.
- **Reducing repeated I/O** — every avoided re-execution is avoided disk/buffer reads, which matters a lot at scale.

---

## 4. Syntax

```sql
CREATE [OR REPLACE] FUNCTION function_name (parameter_list)
    RETURN return_datatype
    RESULT_CACHE [RELIES_ON (table_name [, table_name2 ...])]
IS
BEGIN
    ...
    RETURN expression;
END function_name;
/
```

### Syntax Breakdown

| Clause | Meaning |
|---|---|
| `RESULT_CACHE` | The key clause — tells Oracle to cache this function's results, keyed by its input parameter values. |
| `RELIES_ON (table_name, ...)` | **Deprecated but still seen in older code/exam material.** Originally used to explicitly tell Oracle which tables the function's result depends on, so Oracle knows to invalidate the cache when those tables change. In modern Oracle versions, this is **automatically detected** — Oracle tracks which tables a `RESULT_CACHE` function reads and invalidates cached entries automatically when those tables are modified. You should know this clause exists (for reading legacy code/exam questions) but generally won't need to write it yourself in current Oracle versions. |

### Example
```sql
CREATE OR REPLACE FUNCTION get_exchange_rate (p_currency_code VARCHAR2)
    RETURN NUMBER
    RESULT_CACHE
IS
    v_rate NUMBER;
BEGIN
    SELECT rate INTO v_rate
    FROM exchange_rates
    WHERE currency_code = p_currency_code;

    RETURN v_rate;
END get_exchange_rate;
/
```

Now, calling `get_exchange_rate('USD')` the first time actually runs the query. Every subsequent call with `'USD'` (from anywhere, any session) returns instantly from cache — **until** the `exchange_rates` table is modified, at which point Oracle automatically invalidates the relevant cached entries.

---

## 5. Detailed Explanation — How It Actually Works

- The cache is **keyed by the combination of input parameter values**. `get_exchange_rate('USD')` and `get_exchange_rate('EUR')` are cached as **separate entries** — caching one doesn't help the other.
- The cache lives in the **SGA** (shared server memory), meaning it's **shared across sessions and users** — one user's call can populate the cache, and a completely different user's identical call benefits from it. This is different from, say, a PL/SQL package-level variable, which is typically session-specific.
- **Automatic invalidation**: Oracle tracks the tables a `RESULT_CACHE` function depends on (via its `SELECT` statements) and automatically flushes the relevant cached entries when those underlying tables are modified (INSERT/UPDATE/DELETE/TRUNCATE). This is what makes the cache *safe* to use — you don't get stale data indefinitely; you get stale data only until the next write to the dependency, which the cache handles for you.
- There are memory limits — the result cache is a finite portion of the SGA, and if it's full, Oracle can evict older/less-used entries.

---

## 6. When to Use / When Not to Use

**Use `RESULT_CACHE` when:**
- The function is **pure** — deterministic, given the same inputs it always produces the same output, with no dependency on session-specific state (like `SYSDATE`, `USER`, session variables) that would make caching semantically wrong.
- The underlying data **changes infrequently** relative to how often the function is called.
- The function is **called very frequently** with a **relatively small number of distinct input combinations** (high "cache hit" potential).
- The computation/query behind it is **non-trivial** — caching a trivial one-line calculation isn't worth the caching overhead.

**Don't use `RESULT_CACHE` when:**
- The function's result **depends on something other than its parameters** — e.g., `SYSDATE`, session context, or frequently-changing data — because you'd either get incorrect stale-feeling behavior, or the cache would be constantly invalidated and provide no benefit (worse than useless, since there's caching overhead for no gain).
- The function has **side effects** (performs DML) — result-cached functions are expected to be pure; mixing caching with side effects is both a design smell and can produce confusing behavior (you might skip execution — and thus skip a side effect — on a cache hit).
- Input parameters have **huge cardinality** (e.g., a unique customer ID with millions of distinct customers, each called once) — you'd rarely get a cache hit, and you'd just be filling memory with one-time-use entries for no benefit.
- The function's underlying data changes **almost as often** as it's called — the cache would thrash (invalidate and repopulate constantly), adding overhead without meaningful benefit.

---

## 7. Common Mistakes & Misconceptions

1. **Mistake**: Applying `RESULT_CACHE` to a function that reads `SYSDATE` or session-specific values, expecting fresh results every call — but getting a stale cached value from an earlier call instead, because the cache doesn't know those "invisible" inputs changed.
2. **Misconception**: "The cache is per-session, like a package variable." → It is not — it's a shared, instance-wide cache, which is actually part of its power (one user's call helps everyone), but also means you must be careful about what "the same call" means across different users' contexts.
3. **Mistake**: Using `RESULT_CACHE` on a function with millions of distinct parameter combinations, expecting a performance win, when in reality there's rarely a repeated identical call to actually hit the cache.
4. **Misconception**: "I need `RELIES_ON` to make caching work correctly." → In modern Oracle, dependency tracking is automatic; `RELIES_ON` is legacy syntax mostly seen in older training material or exams.
5. **Mistake**: Caching a function that performs an `INSERT`/`UPDATE`/`DELETE` — a cache hit means the function body (and its side effect) **doesn't run at all** the second time, silently skipping intended actions. This is a serious, hard-to-debug bug class.

---

## 8. Edge Cases to Be Aware Of

- If the function is called from **different sessions with different NLS settings** (date formats, language, etc.) and its output depends on those settings, caching can produce results that look "correct" for one session's format but wrong for another's — because those settings aren't part of the cache key by default. This is a genuinely subtle issue worth being aware of in multi-locale systems.
- The result cache has a **memory footprint and management overhead** — it is not free, and using it indiscriminately on every function "just in case" can actually degrade performance rather than help it.
- Cache invalidation is triggered by **any** DML to the dependent table, even if the specific rows relevant to a cached parameter value weren't touched — e.g., updating one unrelated row in `exchange_rates` can invalidate the entire function's cache, not just the affected currency's entry. This is a coarser-grained invalidation than some people expect.

---

## 9. Interview-Level / Practical Notes

- A common interview/practical question: *"Name a good candidate function for RESULT_CACHE in a typical business system."* — Reference/lookup-style functions (currency rates, tax rates, country codes, configuration lookups) are the textbook answer, precisely because they're read frequently and change rarely.
- *"Why shouldn't you result-cache a function that performs an INSERT?"* — Because a cache hit skips the function body entirely, silently skipping the insert on repeated calls — a real and serious bug source.
- Being able to articulate **both** when to use it and when *not* to (not just "it makes things faster") is what separates surface-level knowledge from genuine understanding in this topic — interviewers often probe specifically for the "when not to" side.

---

## Things You Must Remember

- Add `RESULT_CACHE` in the function header (after the `RETURN` datatype) to enable caching.
- The cache is keyed by the **exact combination of input parameter values** — different args mean different (or no) cache entries.
- The cache is **shared across sessions** (SGA-level), not per-session.
- Modern Oracle **automatically** tracks and invalidates cache entries when dependent tables change — `RELIES_ON` is largely legacy syntax now.
- Only use this on **pure, deterministic, frequently-called, infrequently-changing** functions.
- **Never** use it on functions with side effects (DML) — a cache hit means the body doesn't execute, silently skipping the side effect.

## How to Recognize This Concept

Think **Function Result Cache** when a requirement or situation describes:
- "This lookup/reference value **rarely changes** but is **checked constantly**" — classic reference-data language.
- Performance complaints about a function being called an **enormous number of times** with a **small, repeating set of inputs**.
- Config/settings/rate/tier-lookup style functions — anything that smells like "static-ish reference data accessed via a function."

If the function's answer depends on **something other than its parameters** (current time, session state) or performs **data changes**, that's your signal `RESULT_CACHE` is the *wrong* fit here.

---

## Exercises

1. **(Identify good candidates)** For each of the following functions, state whether `RESULT_CACHE` is a good fit, and justify briefly:
   - a) `get_tax_rate_for_state(p_state_code)` — tax rates change maybe twice a year, called on every single transaction.
   - b) `get_current_server_time()` — returns `SYSDATE`.
   - c) `get_customer_loyalty_points(p_customer_id)` — called once per customer per session, across millions of unique customers.
   - d) `get_country_name(p_country_code)` — a small, static list of ~195 countries, called extremely frequently across many reports.

2. **(Add caching)** Take this function and correctly add result caching to it:
   ```sql
   CREATE OR REPLACE FUNCTION get_shipping_zone (p_postal_code VARCHAR2)
       RETURN VARCHAR2
   IS
       v_zone VARCHAR2(20);
   BEGIN
       SELECT zone INTO v_zone FROM shipping_zones WHERE postal_code = p_postal_code;
       RETURN v_zone;
   END get_shipping_zone;
   /
   ```

3. **(Spot the bug)** A developer adds `RESULT_CACHE` to this function. What will go wrong in production, and why?
   ```sql
   CREATE OR REPLACE FUNCTION log_and_get_discount (p_customer_id NUMBER)
       RETURN NUMBER
       RESULT_CACHE
   IS
       v_discount NUMBER;
   BEGIN
       INSERT INTO discount_access_log (customer_id, accessed_at) VALUES (p_customer_id, SYSDATE);
       SELECT discount_pct INTO v_discount FROM customers WHERE customer_id = p_customer_id;
       RETURN v_discount;
   END log_and_get_discount;
   /
   ```

4. **(Realistic business scenario)** Business requirement: *"Our product catalog has ~500 product categories, each with a fixed commission rate that changes maybe once a quarter when finance updates pricing policy. The function that looks up commission rate by category is called on every single sales transaction — potentially tens of thousands of times per day."* Would you apply `RESULT_CACHE` here? Walk through your reasoning using the "when to use" criteria, and write the function signature (header only) with appropriate caching applied.

5. **(Judgment: borderline case)** A function `get_current_inventory_count(p_product_id)` is called very frequently on a product page, but inventory counts can change every few seconds during a big sale. Would you result-cache this function? What trade-off are you weighing, and what would you recommend instead if caching feels risky here?

---

*Share your attempts whenever you're ready. Next up: Sub Program Parameter Using Mixed and Named Notations.*
