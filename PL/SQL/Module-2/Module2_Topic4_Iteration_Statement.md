# Module 2, Topic 4: Iteration Statement

---

## 1. What Is an Iteration Statement?

An **iteration statement** (a "loop") repeats a block of logic **multiple times** — either a known, fixed number of times, or repeatedly until/while some condition holds. This is the third and final fundamental control-flow category introduced conceptually in Topic 1.

PL/SQL provides **three** loop constructs: the **basic LOOP**, the **WHILE LOOP**, and the **FOR LOOP** — each suited to a different shape of repetition requirement.

---

## 2. Why Does Iteration Exist? What Problem Does It Solve?

Recall the very first business example from Module 1, Topic 1:

> "For every customer whose account balance is negative, deduct a penalty fee..."

The phrase **"for every"** is the unmistakable signal: this requires processing **many records, one at a time, with the same logic applied to each**. Without loops, you would need to write out the exact same block of code once per record — completely impractical (and impossible) when the number of records is only known at runtime, or numbers in the thousands/millions. Iteration statements let you write the logic **once** and have PL/SQL apply it repeatedly, automatically — the foundation of virtually all batch processing, bulk validation, and row-by-row business logic.

---

## 3. Form 1 — Basic LOOP

### Syntax
```sql
LOOP
    statement(s);
    EXIT WHEN exit_condition;
END LOOP;
```

The **basic LOOP** repeats **unconditionally and indefinitely** — it has **no built-in stopping condition at all**. You are entirely responsible for including an `EXIT` (or `EXIT WHEN condition`) statement somewhere inside it; otherwise, it runs **forever** (an infinite loop — a genuine, serious bug, not a theoretical concern).

### Example
```sql
DECLARE
    v_counter NUMBER := 1;
BEGIN
    LOOP
        DBMS_OUTPUT.PUT_LINE('Iteration: ' || v_counter);
        v_counter := v_counter + 1;
        EXIT WHEN v_counter > 5;
    END LOOP;
END;
/
-- Prints Iteration: 1 through 5, then stops.
```

### When to Use It
Use the basic LOOP when the exit condition is **most naturally checked partway through** the loop body (e.g., "do something, THEN check if you should stop") — rather than checked purely at the very start (WHILE) or driven by a simple counted range (FOR).

---

## 4. Form 2 — WHILE LOOP

### Syntax
```sql
WHILE condition LOOP
    statement(s);
END LOOP;
```

The condition is checked **before** each iteration (including the very first one) — if it's `FALSE` (or `NULL`, per the same rule you learned for `IF` in Topic 2) from the start, the loop body **never executes at all**, not even once.

### Example
```sql
DECLARE
    v_stock NUMBER := 50;
    v_orders_filled NUMBER := 0;
BEGIN
    WHILE v_stock > 0 LOOP
        v_stock := v_stock - 10;
        v_orders_filled := v_orders_filled + 1;
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('Orders filled: ' || v_orders_filled);
END;
/
```

### When to Use It
Use `WHILE` when the number of iterations **isn't known in advance** and depends entirely on a condition that might genuinely be false from the very beginning (e.g., "keep processing while there's remaining stock" — if there's no stock at all, zero iterations should happen, which `WHILE` naturally guarantees but a basic `LOOP` would not, without extra care).

---

## 5. Form 3 — FOR LOOP (Numeric Range)

### Syntax
```sql
FOR loop_counter IN [REVERSE] lower_bound..upper_bound LOOP
    statement(s);
END LOOP;
```

- `loop_counter` is **implicitly declared** by the `FOR` loop itself — you do **not** (and cannot) declare it yourself beforehand; it automatically exists only within the loop's scope and disappears once the loop ends.
- It automatically counts from `lower_bound` to `upper_bound`, incrementing by 1 each iteration (or decrementing, if `REVERSE` is specified).
- The loop runs exactly `(upper_bound - lower_bound + 1)` times — if `lower_bound > upper_bound` (without `REVERSE`), the loop body **never executes**, silently, with no error.

### Example
```sql
BEGIN
    FOR i IN 1..5 LOOP
        DBMS_OUTPUT.PUT_LINE('Value: ' || i);
    END LOOP;
END;
/
```

### Example with REVERSE
```sql
BEGIN
    FOR i IN REVERSE 1..5 LOOP
        DBMS_OUTPUT.PUT_LINE('Countdown: ' || i);  -- prints 5, 4, 3, 2, 1
    END LOOP;
END;
/
```
**Important**: even with `REVERSE`, you still write the bounds in **ascending order** (`1..5`, not `5..1`) — `REVERSE` only affects the **direction of counting**, not how you specify the range itself. Writing `REVERSE 5..1` doesn't work the way beginners often expect.

### When to Use It
Use `FOR` when the **exact number of iterations is known** ahead of time — a fixed range, or (as you'll see connects naturally later, though full cursor mechanics are outside this syllabus) iterating a known collection. This is the **most common, safest** loop form in practice, precisely because you can never accidentally create an infinite loop with it — the range inherently bounds it.

---

## 6. Loop Control: EXIT, EXIT WHEN, and CONTINUE

| Statement | Effect |
|---|---|
| `EXIT;` | Immediately terminates the **current** loop entirely, unconditionally, right where it's placed. |
| `EXIT WHEN condition;` | Terminates the loop **only if** `condition` is `TRUE` — otherwise, continues normally. This is the standard way to build a controlled exit inside a basic `LOOP`. |
| `CONTINUE;` | Skips the **rest of the current iteration** and jumps straight to the next one (re-checking the loop's condition, if any). |
| `CONTINUE WHEN condition;` | Skips the rest of the current iteration **only if** `condition` is `TRUE`. |

### Example — CONTINUE
```sql
BEGIN
    FOR i IN 1..10 LOOP
        CONTINUE WHEN MOD(i, 2) = 0;  -- skip even numbers
        DBMS_OUTPUT.PUT_LINE('Odd number: ' || i);
    END LOOP;
END;
/
```

---

## 7. Nested Loops and Labels

Loops can be **nested** inside each other, exactly like blocks (Module 1, Topic 2). When nested, you can optionally **label** an outer loop, allowing an inner `EXIT`/`CONTINUE` to target the **outer** loop specifically, not just the innermost one.

```sql
BEGIN
    <<outer_loop>>
    FOR i IN 1..3 LOOP
        FOR j IN 1..3 LOOP
            IF j = 2 THEN
                EXIT outer_loop;  -- exits the OUTER loop entirely, not just the inner one
            END IF;
            DBMS_OUTPUT.PUT_LINE('i=' || i || ', j=' || j);
        END LOOP;
    END LOOP outer_loop;
END;
/
```
Without the label, a plain `EXIT;` inside the inner loop would only terminate the **inner** loop, and the outer loop would continue to its next iteration as normal — labels are what let you deliberately break out of multiple nested levels at once.

---

## 8. Detailed Explanation — Connecting Back to Earlier Modules

- Recall **`PLS_INTEGER`** from Module 1, Topic 4: loop counters (especially in `FOR` loops processing large ranges) are a textbook case where `PLS_INTEGER` outperforms `NUMBER` — though note that in a numeric `FOR` loop, the implicitly-declared loop counter is **already** effectively an integer type optimized for this purpose; the `PLS_INTEGER` consideration matters more when you declare your **own** counter/accumulator variables used heavily inside a loop body (as in the basic `LOOP`/`WHILE` examples above).
- Recall from Module 4: any statement inside a loop body that can raise an exception (a `SELECT INTO` that might fail for a particular iteration's data, for example) needs to be considered carefully — an **unhandled exception inside a loop immediately terminates the entire loop**, exactly like it would terminate any block, unless you wrap that specific risky statement in its **own nested block with local exception handling** (Module 1, Topic 2's nested-block pattern), allowing the loop to catch the problem for that one iteration and continue to the next, rather than aborting the whole batch.

```sql
BEGIN
    FOR i IN 1..10 LOOP
        BEGIN  -- nested block, local exception handling per iteration
            -- some operation that might fail for this particular i
            NULL;
        EXCEPTION
            WHEN OTHERS THEN
                DBMS_OUTPUT.PUT_LINE('Iteration ' || i || ' failed, continuing...');
        END;
    END LOOP;
END;
/
```
This pattern — **nested block inside a loop body, with its own exception handler** — is precisely the mechanism that fulfills the "one bad record shouldn't crash the whole batch" requirement first raised all the way back in Module 4, Topic 1. It's worth pausing to appreciate: this single pattern ties together Module 1 (nesting), Module 2 (iteration), and Module 4 (exception handling) into one cohesive, extremely common real-world design.

---

## 9. When to Use Which Loop

| Situation | Loop Form |
|---|---|
| Exact number of iterations known ahead of time | `FOR` |
| Number of iterations unknown, condition checked **before** each iteration (possibly zero iterations) | `WHILE` |
| Number of iterations unknown, condition more naturally checked **partway through/at the end** of each iteration | Basic `LOOP` with `EXIT WHEN` |

---

## 10. Common Mistakes & Misconceptions

1. **Mistake**: Writing a basic `LOOP` and forgetting the `EXIT`/`EXIT WHEN` entirely → a genuine infinite loop, which in a real database session will consume resources indefinitely until manually killed.
2. **Misconception**: "I can declare and reuse the FOR loop's counter variable outside the loop." → The counter is scoped **only** to the loop itself; it doesn't exist before the loop starts and ceases to exist the moment it ends — referencing it afterward is a compile error.
3. **Mistake**: Writing `FOR i IN REVERSE 5..1 LOOP` expecting it to count down from 5 to 1 → this actually produces **zero iterations**, because the bounds must still be written in ascending order (`1..5`) even with `REVERSE`.
4. **Mistake**: Using a bare `EXIT;`/`CONTINUE;` inside nested loops, expecting it to affect the **outer** loop, without realizing it only affects the **innermost** loop unless a label is used.
5. **Misconception**: "An unhandled exception inside a loop just skips that one iteration and moves to the next automatically." → False — an unhandled exception **terminates the entire loop** (and propagates outward, per Module 4 rules), exactly like it would for any other code; achieving "skip this iteration, continue the loop" requires **deliberately** wrapping the risky logic in its own nested block with local exception handling, as shown above.

---

## 11. Edge Cases to Be Aware Of

- A `FOR` loop where `lower_bound > upper_bound` (and no `REVERSE`) executes **zero times**, silently — no error, just nothing happens. This is a common source of "why didn't my loop run at all?" confusion when bounds are computed dynamically and turn out reversed.
- `WHILE` loops can also execute **zero times** if the condition is `FALSE` (or `NULL`) from the very start — unlike a basic `LOOP`, which always executes **at least once** before any `EXIT` is even reached (since the exit check typically happens after some processing, not before the first iteration).
- `CONTINUE` inside a `FOR` loop still allows the loop's automatic counter increment to happen — `CONTINUE` only skips the **remaining body statements** for that iteration, not the loop mechanism itself.

---

## 12. Interview-Level / Practical Notes

- *"What's the key structural difference between WHILE and basic LOOP?"* — `WHILE` checks its condition **before** each iteration (can run zero times); basic `LOOP` has no built-in condition at all and relies entirely on an internal `EXIT`/`EXIT WHEN` (and always runs at least once up to that exit check).
- *"Why is FOR generally considered the 'safest' loop form?"* — Because the range is fixed and bounded at the start; you cannot accidentally create an infinite loop with a numeric `FOR` loop the way you can with `WHILE` or basic `LOOP` if you mismanage the exit condition.
- *"How do you make one bad record in a batch loop not crash the entire batch?"* — Wrap the risky operation in its own nested block with local exception handling inside the loop body — a very standard, expected real-world pattern, and a great way to demonstrate integrated understanding across modules in an interview.

---

## Things You Must Remember

- Three forms: **basic LOOP** (no built-in condition, needs manual `EXIT`), **WHILE** (condition checked *before* each iteration, can run zero times), **FOR** (fixed numeric range, implicitly-scoped counter, safest against infinite loops).
- `FOR` loop counters are **implicitly declared**, scoped only to the loop, and cannot be referenced outside it.
- `REVERSE` still requires **ascending** bounds (`REVERSE 1..5`, not `REVERSE 5..1`) — it only changes counting direction.
- `EXIT`/`EXIT WHEN` stop a loop entirely; `CONTINUE`/`CONTINUE WHEN` skip just the rest of the current iteration.
- Labels (`<<label_name>>`) let `EXIT`/`CONTINUE` target a specific **outer** loop in nested loop structures.
- An unhandled exception **terminates the whole loop** — "skip this iteration, keep going" requires a **deliberate nested block with local exception handling** inside the loop body.

## How to Recognize This Concept

- "**For each** / **for every** [record/item], do X" → **iteration**, and if the count is known/bounded → **FOR loop** specifically.
- "**Keep doing X while/until** [condition]," where the condition might be false from the very start → **WHILE loop**.
- "**Repeat X, then check if** you should stop" (check happens after some work is done) → **basic LOOP with EXIT WHEN**.
- "**One bad record shouldn't stop the whole batch**" → immediately think: nested block **inside** the loop body, with its own local exception handler — this is a very strong, specific pattern-recognition signal worth internalizing deeply.

---

## Exercises

1. **(Basic LOOP)** Write a basic `LOOP` that prints numbers 1 through 10, using `EXIT WHEN` to stop.

2. **(WHILE, zero-iteration case)** Write a `WHILE` loop simulating processing a queue that starts already empty (`v_queue_size := 0;`), and explain/demonstrate that the loop body never executes at all.

3. **(FOR with REVERSE)** Write a `FOR` loop that counts down from 10 to 1, printing each value, using the syntax correctly.

4. **(CONTINUE)** Write a `FOR` loop from 1 to 20 that prints only the numbers divisible by 3, skipping all others using `CONTINUE WHEN`.

5. **(Nested loops with label)** Write a nested loop (outer 1..3, inner 1..3) that exits the **entire outer loop** the moment the combination `i=2, j=2` is reached, using a label. Show what the output looks like.

6. **(Resilient batch processing — the key pattern)** Business requirement: *"Process employee IDs 100 through 110. For each one, look up their salary. If a particular employee ID doesn't exist, log a message and continue processing the remaining IDs — the batch must not stop."* Write this using a `FOR` loop with a nested block and local exception handling, exactly matching the pattern described in Section 8. Explain, in your own words, what would happen instead if you removed the nested block's exception handler entirely.

7. **(Loop selection judgment)** For each of the following, state which loop form is most appropriate, and why:
   - a) "Print each month name for the 12 months of the year."
   - b) "Keep polling a status flag until it changes to 'COMPLETE', however long that takes."
   - c) "Process transactions from a queue, stopping only once a transaction with a specific 'END_OF_BATCH' marker is encountered — checked after each transaction is processed."

---

*Share your answers whenever you're ready. Next up: Module 2, Topic 5 — Sequential Statement and Nesting of Blocks, the final theory topic before this module's closing practice checkpoint.*
