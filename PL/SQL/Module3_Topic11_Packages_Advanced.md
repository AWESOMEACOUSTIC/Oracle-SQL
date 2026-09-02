# Module 3, Topic 11: Packages (Advanced Concepts & Capstone)

> This is the final, largest topic in Module 3. Topics 8–10 gave you package **structure**, **overloading**, and the **design process**. This topic fills in the **advanced mechanics** that make packages work reliably in real production systems — forward declarations, the full package state lifecycle, and dependency/invalidation behavior in depth — then closes with a comprehensive, multi-concept business case study that pulls together everything from this entire module (procedures, functions, exceptions-aware design, overloading, notation, and package structure).

---

## 1. Forward Declarations

### What It Is
Recall from Topic 10: PL/SQL resolves package body members **top-to-bottom** — a member can only call another member that's **already been defined earlier** in the same body. This becomes a real problem when **two private helpers need to call each other** (mutual/circular dependency) — no matter which order you put them in, one will always be "used before defined."

A **forward declaration** solves this: you declare just the **signature** of a sub program near the top of the package body (before its full implementation appears later), so anything in between can reference it.

### Syntax
```sql
CREATE OR REPLACE PACKAGE BODY pkg_example
IS
    -- Forward declaration: signature only, no body yet
    FUNCTION helper_b (p_val NUMBER) RETURN NUMBER;

    FUNCTION helper_a (p_val NUMBER) RETURN NUMBER
    IS
    BEGIN
        IF p_val > 100 THEN
            RETURN p_val;
        ELSE
            RETURN helper_b(p_val + 1);  -- calling helper_b before its full body appears below
        END IF;
    END helper_a;

    FUNCTION helper_b (p_val NUMBER) RETURN NUMBER
    IS
    BEGIN
        IF p_val > 100 THEN
            RETURN p_val;
        ELSE
            RETURN helper_a(p_val * 2);  -- calling helper_a, defined above
        END IF;
    END helper_b;

END pkg_example;
/
```

### When to Use It
- Only when you have a genuine **mutual dependency** between two private helpers (each calls the other, directly or indirectly).
- **Not needed** for the common case (Topic 10's example) where dependencies flow in one direction only (a public procedure calling a private helper defined earlier) — don't add forward declarations "just in case"; only when the top-to-bottom rule genuinely can't be satisfied otherwise.

---

## 2. Package State — The Full Lifecycle (Deep Dive)

You saw package-level variables briefly in Topic 8. Let's now be precise about their **entire lifecycle**, because this is a frequent source of subtle real-world bugs.

### Key Facts
1. **Scope**: Package-level variables are declared in the spec (if public) or body (if private), **outside** any individual procedure/function — meaning their value persists **across multiple calls**, not just within one call like a local variable would.
2. **Per-session, not global**: Each **session** that touches the package gets its **own independent copy** of package-level state. Session A's changes to a package variable are completely invisible to Session B — they are not shared across users, unlike the Function Result Cache (Topic 6), which explicitly *is* shared SGA-wide. This contrast is one of the most commonly confused points in this entire module — make sure it's crystal clear.
3. **Initialization**: The optional unnamed `BEGIN...END` block at the end of the package body runs **exactly once per session**, the very first time **any** member of the package is referenced in that session (not once per call, not once per statement).
4. **Lifetime**: Package state persists for the **life of the session** (or until the package is explicitly reset — see below), not just for the duration of one transaction or one call.
5. **Resetting mid-session**: `DBMS_SESSION.RESET_PACKAGE` (or ending/reconnecting the session) clears package state, forcing the initialization block to run again on next access. This is a real tool used in long-running application connection pools to avoid stale package state leaking between logical "users" sharing a physical database session.

### Realistic Example Showing the Full Lifecycle
```sql
CREATE OR REPLACE PACKAGE pkg_batch_counter
IS
    PROCEDURE increment;
    FUNCTION get_count RETURN NUMBER;
END pkg_batch_counter;
/

CREATE OR REPLACE PACKAGE BODY pkg_batch_counter
IS
    g_count NUMBER;  -- private package-level variable

    PROCEDURE increment
    IS
    BEGIN
        g_count := g_count + 1;
    END increment;

    FUNCTION get_count RETURN NUMBER
    IS
    BEGIN
        RETURN g_count;
    END get_count;

BEGIN
    g_count := 0;  -- initialization: runs once per session, on first reference
END pkg_batch_counter;
/
```

If Session A calls `increment` three times, `pkg_batch_counter.get_count` returns `3` **in Session A**. If Session B (a completely different connection) calls `get_count` right now, it gets `0` — its own copy, freshly initialized on its own first access, untouched by Session A's activity.

### Why This Matters in Real Systems
- **Connection pooling** (extremely common in real applications — a pool of database connections shared across many application-level users): if your application logic mistakenly assumes package state is "per logical user" but it's actually "per physical database session," and sessions get reused across different users by the pool, **stale state from a previous user can leak into the next one's logic**. This is a real, subtle, and dangerous class of production bug.
- This is exactly why some teams deliberately avoid package-level state for anything user-specific, and instead pass all needed context explicitly as parameters — trading a little convenience for a lot of safety.

---

## 3. Dependency & Invalidation — Deeper Look

Revisiting and formalizing what Topic 8 introduced:

- **Spec change** (adding/removing/changing a public member's signature) → any object that **references** the package (calls it) becomes **INVALID** and must be recompiled before it can run again (Oracle typically attempts automatic recompilation on next use, but this can fail if the change broke compatibility — e.g., a caller using a parameter that no longer exists).
- **Body-only change** (same public signatures, different internal implementation) → dependent objects are **not** invalidated. This is the core practical benefit of the spec/body split: you can fix bugs, optimize, or rewrite internals freely, and every calling application keeps working without any redeployment on their end.
- **Real-world implication**: teams design package specs to be **stable, carefully-considered contracts** — because changing them has a blast radius across every caller — while treating package bodies as **freely refactorable implementation detail**, changed far more often and cheaply.

A common real error you should recognize: `ORA-04068: existing state of packages has been discarded` — this happens when a package's state was reset/invalidated mid-session (often due to a recompilation) while a session still held references expecting the old state to persist; the session must re-initialize its package state on next access.

---

## 4. Common Mistakes & Misconceptions (Advanced-Level)

1. **Misconception**: "Package-level variables behave like the Function Result Cache — shared across all users." → No — package state is **per-session**; only the Result Cache (Topic 6) is genuinely SGA-shared across sessions. Confusing these two is a very common and consequential mistake.
2. **Mistake**: Adding forward declarations everywhere "for safety," when the natural top-to-bottom order already works fine — unnecessary and adds clutter.
3. **Mistake**: Assuming a body-only change is always 100% safe with zero risk — while dependents aren't *invalidated*, a body change could still introduce a **behavioral** bug (wrong results) even though it compiles fine and nothing is technically "broken" at the dependency level. Recompilation safety is not the same thing as correctness.
4. **Misconception**: "The initialization block runs every time I call any procedure in the package." → It runs **once per session**, not once per call — a very frequently misunderstood point.
5. **Mistake**: Relying on package-level state to pass information between **unrelated** business operations "because it's convenient," creating hidden coupling and making the code's behavior depend on call order and session history — hard to test, hard to reason about, and risky in connection-pooled environments.

---

## 5. Interview-Level / Practical Notes

- *"Is package state shared across sessions or per-session?"* — **Per-session.** This is one of the most commonly tested distinctions in PL/SQL interviews, specifically because it's easy to confuse with the Result Cache.
- *"What's the practical risk of package-level state in a connection-pooled application?"* — Stale state leaking between logically different users sharing a physical session, if the pool reuses connections without resetting package state appropriately.
- *"Does fixing a bug inside a package body require every calling application to redeploy?"* — No, as long as the public signatures (spec) are unchanged — this is a genuinely valuable, real operational benefit worth being able to articulate clearly.

---

## Things You Must Remember

- Forward declarations solve **mutual/circular dependencies** between package body members — declare the signature early, implement it later in the body.
- Package-level state is **per-session**, never shared across different sessions/users — contrast sharply with the SGA-wide Function Result Cache.
- The initialization section runs **once per session**, on first reference — not once per call.
- **Spec changes** invalidate dependents; **body-only changes** (same public signatures) do not.
- `ORA-04068` relates to package state being discarded mid-session — a real error worth recognizing, not just memorizing.
- Package-level state is powerful but risky in connection-pooled environments — use deliberately, not by default.

## How to Recognize These Concepts

- **Forward declaration** need: "these two internal helper routines call each other" — a genuine circular reference between private members.
- **Package state caution flag**: any requirement involving "remember something across multiple calls in a session," especially combined with "**many users**" or "**shared application server**" language — this should immediately raise the connection-pooling risk question in your mind.
- **Spec vs. body change reasoning**: any requirement phrased as "fix a bug without affecting other systems that already use this" → body-only change is the goal; changing the spec should be treated as a deliberate, higher-impact decision.

---

## Capstone Case Study — Combining Everything From Module 3

**Business requirement:**

> "Our company processes employee expense reimbursements. We need a package, `pkg_expense_processing`, that supports the following:
>
> 1. Submit an expense claim — given an employee ID, an amount, and a category (`'TRAVEL'`, `'MEALS'`, `'OTHER'`), insert the claim as `'PENDING'`. Before inserting, check that the amount doesn't exceed the category's maximum allowed limit (TRAVEL: 50,000; MEALS: 5,000; OTHER: 10,000) — this limit-lookup logic is purely internal and shouldn't be exposed.
> 2. Approve a claim — given a claim ID, mark it `'APPROVED'` and return the approved amount to the caller.
> 3. Allow other systems to look up a claim's current status, either by claim ID or by a claim reference code (a text identifier) — both should feel like the same conceptual lookup to callers.
> 4. Track, for reporting purposes within a single batch-processing session, how many claims were approved during that session — reset naturally each time a new session starts."

**Your task:** Using the full 7-step design process from Topic 10, plus everything from Topics 8, 9, and this topic:

1. Identify each required package member, and classify it as **public procedure**, **public function**, or **private helper**.
2. Identify where **overloading** applies (re-read requirement #3 carefully).
3. Identify where **package-level state** applies (requirement #4) — and explicitly note the **per-session risk** this introduces if this package were ever used in a connection-pooled application.
4. Write the full **package specification**.
5. Write the full **package body**, with reasonable illustrative implementations. Pay attention to member ordering (and note if any forward declaration would be needed).
6. In 4–5 sentences, explain your public/private boundary choices, tying each decision back to specific wording in the requirement.

---

## Additional Exercises

7. **(Forward declaration necessity)** Explain, in your own words, why the following body **fails to compile without a forward declaration**, and rewrite it with one added correctly:
   ```sql
   CREATE OR REPLACE PACKAGE BODY pkg_recursive_check
   IS
       FUNCTION check_even (p_val NUMBER) RETURN VARCHAR2
       IS
       BEGIN
           IF p_val = 0 THEN
               RETURN 'Y';
           ELSE
               RETURN check_odd(p_val - 1);
           END IF;
       END check_even;

       FUNCTION check_odd (p_val NUMBER) RETURN VARCHAR2
       IS
       BEGIN
           IF p_val = 0 THEN
               RETURN 'N';
           ELSE
               RETURN check_even(p_val - 1);
           END IF;
       END check_odd;
   END pkg_recursive_check;
   /
   ```

8. **(Connection pool risk reasoning)** A web application uses a connection pool of 20 physical database sessions shared across thousands of logged-in users. A package `pkg_user_context` stores the "current user's" ID in a package-level variable, set at login and read throughout the request. Explain, concretely, what could go wrong in this architecture, and what alternative design would avoid the risk entirely.

9. **(Spec vs. body judgment, applied)** For the capstone `pkg_expense_processing` package: if next month the company decides to raise the MEALS limit from 5,000 to 7,000, does this require a spec change or just a body change? What about if they decide to add an entirely new public operation, `reject_claim`? Explain both.

---

*This wraps up Module 3 in full — Procedures, Functions, and Packages. Share your capstone attempt whenever you're ready (even partial attempts are worth reviewing). Once we're done here, we'll move into Module 4: PL/SQL Exception, starting with "Intro to Exception."*
