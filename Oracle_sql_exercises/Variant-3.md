# Oracle SQL Certification Practice — Topic 3
## Functions: Deterministic/Nondeterministic, Aggregate/Scalar, String, Math, Miscellaneous & Nesting

---

### Subtopic 1: Deterministic and Nondeterministic Functions

**Question 1**
Which two functions are classified as **nondeterministic** (can return a different result on repeated calls with the same input, or even no input, within the same statement/session)? **(Choose two)**

A. `UPPER(last_name)`
B. `SYSDATE`
C. `ROUND(salary, 2)`
D. `USER`

**Correct Answers:** B and D

**Detailed Explanation:**
A deterministic function always returns the same output for the same input, regardless of when or how many times it's called. `UPPER` and `ROUND` are pure, input-driven transformations — always deterministic. `SYSDATE` (changes with the clock) and `USER` (depends on session context, not on any argument) are classic nondeterministic functions — they depend on external, changing state rather than solely on their arguments.

**Why the Other Options Are Wrong:**
A, C. Both produce identical output every time for identical input — the definition of deterministic.

**Concept Tested:** Deterministic vs. nondeterministic function classification.

**Exam Trap:** Assuming "deterministic" is about complexity or randomness alone — it's really about whether output depends *only* on the explicit arguments.

---

**Question 2**
Why does Oracle require a function used in a **function-based index** to be declared (or provably) deterministic?

A. Nondeterministic functions run slower, and Oracle blocks them purely for performance reasons.
B. The index stores pre-computed function results; if the function could return different values for the same row's data over time, the stored index entries would silently drift out of sync with what a live evaluation would produce.
C. Oracle can only index character-returning functions, and nondeterministic functions never return characters.
D. There is no such requirement — any function can be used in a function-based index.

**Correct Answer:** B

**Detailed Explanation:**
A function-based index materializes the function's output as the indexed value. That value is only trustworthy for lookups if evaluating the function again on the same row data reliably reproduces it. A nondeterministic function (e.g., one depending on SYSDATE) would make the stored index value stale or inconsistent the moment time passes, breaking correctness — not just performance.

**Why the Other Options Are Wrong:**
A. Understates the real issue — it's a correctness guarantee, not a speed optimization.
C. Not related to data type at all.
D. Oracle explicitly enforces (or requires you to assert via `DETERMINISTIC`) this for PL/SQL functions used in function-based indexes.

**Concept Tested:** Why determinism matters for function-based indexes.

**Exam Trap:** Treating the DETERMINISTIC requirement as a stylistic/performance hint rather than a correctness safeguard.

---

**Question 3**
Which of the following built-in SQL functions is guaranteed **deterministic** and therefore safe to use directly in a function-based index?

A. `SYSDATE`
B. `DBMS_RANDOM.VALUE`
C. `UPPER(last_name)`
D. `USERENV('SESSIONID')`

**Correct Answer:** C

**Detailed Explanation:**
`UPPER` depends only on its input string — same input, same output, forever. The other three all depend on external, time- or session-varying state, making them fundamentally unsuitable for indexing.

**Why the Other Options Are Wrong:**
A. Changes every second.
B. Deliberately randomized — the polar opposite of deterministic.
D. Session-dependent; two different sessions get different results from identical "input" (there's effectively no argument at all).

**Concept Tested:** Recognizing deterministic vs. nondeterministic built-ins.

**Exam Trap:** Assuming all built-in single-row functions are automatically safe for indexing, rather than checking each one's dependency on external state.

---

**Question 4**
True or False: `TO_CHAR(hire_date, 'YYYY-MM-DD')` is nondeterministic because its output format depends on the session's NLS settings if you omit the format model.

A. True
B. False

**Correct Answer:** B (False) — as written, with an explicit format model supplied, it IS deterministic.

**Detailed Explanation:**
The trap here is the phrase "if you omit the format model" — the question as posed *does* supply an explicit format mask (`'YYYY-MM-DD'`), which fully determines the output regardless of session NLS settings. It's `TO_CHAR(hire_date)` **without** an explicit format model that becomes session-dependent (and therefore nondeterministic in the strict sense), because it silently falls back to the session's `NLS_DATE_FORMAT`.

**Concept Tested:** Explicit format models make date-to-character conversions deterministic; omitting them reintroduces session dependency.

**Exam Trap:** Conflating "this function *can* be nondeterministic in some usage" with "this specific, fully-specified call is nondeterministic" — always check whether the actual call includes a format model.

---

**Question 5**
Which statement about `DBMS_RANDOM.VALUE` is TRUE?

A. It is deterministic because it always returns a NUMBER between 0 and 1.
B. It is nondeterministic — the whole point of the function is to return a different value on each call, regardless of any input.
C. It becomes deterministic if wrapped in `ROUND()`.
D. It is deterministic within a single transaction, but nondeterministic across transactions.

**Correct Answer:** B

**Detailed Explanation:**
Determinism is about reproducibility given the *same call*, not about the output's data type or range. `DBMS_RANDOM.VALUE` is intentionally designed to vary on every invocation — that's its entire purpose.

**Why the Other Options Are Wrong:**
A. Confuses "consistent output range" with "consistent output value."
C. Wrapping a nondeterministic function in a deterministic one doesn't retroactively fix the inner unpredictability.
D. No transaction-scoped determinism exists for this function.

**Concept Tested:** Distinguishing "constrained output range" from actual determinism.

**Exam Trap:** Assuming a function is deterministic simply because its result always falls within predictable bounds.

---

### Subtopic 2: Aggregate Functions and Scalar Functions

**Question 1**
Which two of the following are aggregate (group) functions rather than single-row (scalar) functions? **(Choose two)**

A. `LENGTH`
B. `STDDEV`
C. `ROUND`
D. `VARIANCE`

**Correct Answers:** B and D

**Detailed Explanation:**
`STDDEV` and `VARIANCE` operate across a set of rows (per group) and collapse them into one summary value per group. `LENGTH` and `ROUND` are scalar — they operate independently, row by row, producing one output per input row with no collapsing.

**Concept Tested:** Aggregate (multi-row-to-one) vs. scalar (row-by-row) function classification.

**Exam Trap:** Assuming any function with a "statistical-sounding" name is automatically scalar, or vice versa — classification depends on behavior (collapsing rows), not name familiarity.

---

**Question 2**
A table has 10 rows; `COMMISSION_PCT` is NULL in 4 of them. What do `COUNT(*)` and `COUNT(commission_pct)` each return?

A. Both return 10.
B. `COUNT(*)` returns 10; `COUNT(commission_pct)` returns 6.
C. Both return 6.
D. `COUNT(*)` returns 6; `COUNT(commission_pct)` returns 10.

**Correct Answer:** B

**Detailed Explanation:**
`COUNT(*)` counts rows unconditionally, including those with NULLs anywhere. `COUNT(column)` counts only rows where that specific column is non-null — it's the one prominent exception to "aggregate functions ignore NULLs silently while still counting the row," since here NULL rows are excluded from the count entirely, not just from a computation.

**Why the Other Options Are Wrong:**
A, C, D. Each misapplies NULL-counting logic to the wrong function.

**Concept Tested:** COUNT(*) vs. COUNT(column) with NULLs present.

**Exam Trap:** Forgetting that COUNT is the aggregate function most directly affected by NULLs in terms of *what gets counted*, not just what gets computed.

---

**Question 3**
Which of these is **valid** Oracle syntax?

A. `SELECT AVG(SUM(salary)) FROM employees;`
B. `SELECT MAX(AVG(salary)) FROM employees GROUP BY department_id;`
C. `SELECT SUM(AVG(salary)) FROM employees WHERE department_id = 10;`
D. All three are invalid.

**Correct Answer:** B

**Detailed Explanation:**
Oracle allows nesting aggregate functions **to a depth of two**, but only when the query includes a GROUP BY — the inner function computes one value per group, and the outer function then aggregates those per-group values into a single overall result. In B, `AVG(salary)` is computed per department (via GROUP BY department_id), and `MAX()` then finds the largest of those departmental averages — a classic "which department has the highest average salary" pattern. Without any GROUP BY (A and C), there is only ever a single implicit group (the whole table, or the WHERE-filtered whole table), so nesting one aggregate inside another is structurally meaningless and Oracle rejects it with **ORA-00978: nested group function without GROUP BY**.

**Why the Other Options Are Wrong:**
A. No GROUP BY present — raises ORA-00978.
C. A WHERE clause filters rows but still provides no GROUP BY — same ORA-00978 error.
D. B is legitimately valid, so this is incorrect.

**Concept Tested:** Nested aggregate functions require an accompanying GROUP BY clause.

**Exam Trap:** Assuming a WHERE clause "counts as" grouping, or that nested aggregates are either always legal or always illegal rather than conditionally legal based on GROUP BY's presence.

---

**Question 4**
Which of the following is a **legal** nesting of functions, regardless of GROUP BY?

A. `SUM(AVG(salary))` with no GROUP BY
B. `ROUND(AVG(salary), 2)`
C. `MAX(MAX(salary))` with no GROUP BY
D. `AVG(COUNT(*))` with no GROUP BY

**Correct Answer:** B

**Detailed Explanation:**
`ROUND` is a scalar function wrapping the *result* of an aggregate function — this is always legal, with or without GROUP BY, because the scalar function is simply post-processing a single already-computed aggregate value. The restriction (ORA-00978) applies specifically to nesting one **aggregate** function directly inside another aggregate function without GROUP BY — not to wrapping an aggregate's result in a scalar function.

**Why the Other Options Are Wrong:**
A, C, D. All nest one aggregate function inside another aggregate function with no GROUP BY present — each raises ORA-00978.

**Concept Tested:** Scalar(aggregate(...)) is always legal; aggregate(aggregate(...)) needs GROUP BY.

**Exam Trap:** Overextending the GROUP BY requirement to *any* nested function call, rather than specifically to aggregate-inside-aggregate nesting.

---

**Question 5**
True or False: Aggregate functions can be used directly in a WHERE clause to filter individual rows before they are grouped.

A. True
B. False

**Correct Answer:** B (False)

**Detailed Explanation:**
The WHERE clause is evaluated row-by-row, *before* any grouping or aggregation takes place — at that stage, there is no "group" for an aggregate function to summarize yet. Filtering on an aggregate's result requires the HAVING clause, which is evaluated *after* GROUP BY has produced its per-group summaries. Attempting `WHERE AVG(salary) > 5000` raises an error.

**Concept Tested:** WHERE (pre-aggregation, row-level) vs. HAVING (post-aggregation, group-level) filtering.

**Exam Trap:** Forgetting that this WHERE/HAVING distinction exists specifically *because* of when aggregation logically occurs in query processing order.

---

### Subtopic 3: String Functions

**Question 1**
What is the key functional difference between the `||` concatenation operator and the `CONCAT` function?

A. There is no difference — they are fully interchangeable in every context.
B. `CONCAT` accepts only exactly two arguments, so combining three or more strings requires nesting calls (`CONCAT(CONCAT(a,b),c)`), while `||` can chain any number of operands directly (`a || b || c`).
C. `||` only works with character literals, never with column values.
D. `CONCAT` automatically inserts a space between its arguments; `||` does not.

**Correct Answer:** B

**Detailed Explanation:**
`CONCAT(string1, string2)` is strictly binary — exactly two arguments. To join more strings with `CONCAT`, you must nest calls. `||` has no such limit and is generally the more convenient choice for joining several strings.

**Why the Other Options Are Wrong:**
A. Understates a real, commonly-tested limitation.
C. Both work identically with columns, literals, or expressions.
D. Neither function inserts a space automatically; any separator must be explicit.

**Concept Tested:** CONCAT's two-argument limit vs. `||`'s unlimited chaining.

**Exam Trap:** Assuming CONCAT scales to any number of arguments the way `||` does.

---

**Question 2**
What does `SUBSTR('Oracle Database', -8, 4)` return?

A. `atab` — a negative start position counts backward from the end of the string, so -8 lands 8 characters from the end, then reads 4 characters forward from there.
B. An error — SUBSTR does not accept negative start positions.
C. `Orac` — negative positions are simply treated as 1 (the start of the string).
D. An empty string.

**Correct Answer:** A

**Detailed Explanation:**
`'Oracle Database'` has 15 characters. A start position of -8 counts 8 characters back from the *end* of the string (position 15), landing on position 8 from the end — the 'a' in "Database" — then reads 4 characters forward from there: `atab`.

**Why the Other Options Are Wrong:**
B, C. Both wrongly claim negative positions are unsupported or normalized to 1 — SUBSTR explicitly documents backward-counting behavior for negative start positions.
D. SUBSTR does not return empty for a valid negative offset within the string's length.

**Concept Tested:** SUBSTR's negative start-position semantics (counting from the end).

**Exam Trap:** Assuming negative positions are invalid or default to the string's beginning, rather than correctly counting backward from the end.

---

**Question 3**
What does `INSTR('Mississippi', 's', 1, 3)` return?

A. 3 — the position of the first 's'.
B. 6 — the position of the third occurrence of 's', searching forward from position 1.
C. An error — INSTR only accepts two arguments.
D. 11 — the position of the last character in the string.

**Correct Answer:** B

**Detailed Explanation:**
`INSTR(string, substring, start_position, occurrence)` — the 3rd argument is where the search begins, and the 4th is *which* occurrence to find. Positions of 's' in "Mississippi" are 3, 6, 9, and 12 (1-indexed). Searching from position 1 for the 3rd occurrence returns position 9... 

Let me recompute precisely: M(1)i(2)s(3)s(4)i(5)s(6)s(7)i(8)p(9)p(10)i(11). The 's' characters are at positions 3, 4, 6, 7. The 3rd occurrence of 's' is at position 6.

**Why the Other Options Are Wrong:**
A. That's the 1st occurrence, not the 3rd.
C. INSTR fully supports all four arguments (string, substring, optional start position, optional occurrence number).
D. Misreads the function as returning string length or last-character position.

**Concept Tested:** INSTR's 4-argument form — start position and occurrence number.

**Exam Trap:** Miscounting overlapping/adjacent occurrences of the target character, or forgetting INSTR supports more than the basic 2-argument form.

---

**Question 4**
What does `RPAD('SQL', 8, '*')` return, and what does `RPAD('SQL', 2, '*')` return?

A. `'SQL*****'` and `'SQ'` — RPAD pads to reach the target length, or truncates the source string if the target length is shorter than the original.
B. `'SQL*****'` and `'SQL'` — RPAD never truncates, only pads.
C. Both raise an error if the target length is shorter than the source string.
D. `'*****SQL'` and `'SQ'` — RPAD pads on the left.

**Correct Answer:** A

**Detailed Explanation:**
RPAD right-pads a string with a repeating pad sequence until it reaches the specified total length. If the specified length is *shorter* than the original string, RPAD instead **truncates** the source string down to that length — a frequently-missed behavior, since most people only remember the padding case.

**Why the Other Options Are Wrong:**
B. Wrongly assumes RPAD is padding-only and ignores its truncation behavior for short target lengths.
C. No error occurs; truncation happens silently.
D. RPAD pads on the right (trailing); LPAD is the left-padding counterpart.

**Concept Tested:** RPAD/LPAD truncate the source string when the target length is shorter than the original.

**Exam Trap:** Only knowing the "padding" half of RPAD/LPAD's behavior and being caught off guard by the truncation case.

---

**Question 5**
What happens when you execute `SELECT TRIM('xy' FROM 'xyHelloxy') FROM dual;`?

A. Returns `'Hello'` — both leading and trailing occurrences of the substring 'xy' are stripped.
B. Raises **ORA-25710: trim set should have only one character** — TRIM's trim-character argument must be a single character, not a multi-character substring.
C. Returns `'xyHelloxy'` unchanged, since TRIM only works with a single default space character.
D. Returns `'Hello'`, but only the leading 'xy' is removed; trailing 'xy' remains.

**Correct Answer:** B

**Detailed Explanation:**
Unlike `REPLACE` or a custom strip routine, TRIM's optional trim-character argument must be exactly **one character**. Supplying a two-character string like `'xy'` is invalid syntax and Oracle raises ORA-25710. To remove a genuine multi-character substring from both ends, you'd need `REPLACE`, `REGEXP_REPLACE`, or repeated single-character TRIMs — not TRIM directly.

**Why the Other Options Are Wrong:**
A. Describes the *intuitive but incorrect* expectation — TRIM cannot strip a substring, only a single repeated character.
C. TRIM is not restricted to spaces only; a single custom character is fully supported (e.g., `TRIM('x' FROM 'xxHelloxx')` is legal and returns `'Hello'`).
D. Also assumes multi-character substring removal, which TRIM doesn't perform.

**Concept Tested:** TRIM's trim-character argument is strictly single-character.

**Exam Trap:** Confusing TRIM (single-character only) with substring-removal functions like REPLACE — one of the most commonly missed string-function distinctions.

---

### Subtopic 4: Mathematical Functions

**Question 1**
What does `MOD(-7, 3)` return in Oracle?

A. 2
B. -1
C. 1
D. -2

**Correct Answer:** B

**Detailed Explanation:**
Oracle's `MOD(n, m)` is defined as `n - m * TRUNC(n/m)` — note the use of TRUNC (toward zero), not FLOOR. `TRUNC(-7/3) = TRUNC(-2.333...) = -2`. So `MOD(-7,3) = -7 - 3*(-2) = -7 + 6 = -1`. The sign of the result follows the **dividend** (n), not the divisor — a frequent point of confusion versus mathematical modulo conventions used elsewhere.

**Why the Other Options Are Wrong:**
A. Would result from using FLOOR instead of TRUNC in the underlying formula (a different modulo convention, not Oracle's).
C, D. Both misapply the sign or the truncation direction.

**Concept Tested:** Oracle's MOD formula and its TRUNC-based (toward-zero), dividend-sign-following behavior with negative operands.

**Exam Trap:** Assuming MOD always returns a non-negative result, as some other languages/systems define modulo — Oracle's MOD can return a negative value when the dividend is negative.

---

**Question 2**
What do `ROUND(1556, -2)` and `TRUNC(1556, -2)` each return?

A. 1600 and 1500, respectively.
B. 1600 and 1600.
C. 1500 and 1500.
D. An error — negative second arguments are not supported by ROUND/TRUNC on integers.

**Correct Answer:** A

**Detailed Explanation:**
A negative second argument to ROUND/TRUNC operates on digits to the *left* of the decimal point. `ROUND(1556, -2)` rounds to the nearest hundred: 1556 → 1600 (since 56 rounds up past the halfway point of 50). `TRUNC(1556, -2)` simply chops off everything below the hundreds place without rounding: 1556 → 1500.

**Why the Other Options Are Wrong:**
B. Wrongly assumes TRUNC also rounds up.
C. Wrongly assumes ROUND doesn't round up here.
D. Negative second arguments are fully supported and are precisely how you round/truncate to tens, hundreds, thousands, etc.

**Concept Tested:** Negative-precision ROUND (rounds) vs. TRUNC (chops) on the integer part.

**Exam Trap:** Assuming ROUND and TRUNC behave identically with a negative second argument just because they behave similarly with a positive one on borderline values.

---

**Question 3**
What do `CEIL(-4.5)` and `FLOOR(-4.5)` return?

A. -4 and -5, respectively.
B. -5 and -4, respectively.
C. -4 and -4.
D. -5 and -5.

**Correct Answer:** A

**Detailed Explanation:**
CEIL always returns the smallest integer **greater than or equal to** the input — for -4.5, that's -4 (moving toward positive infinity). FLOOR always returns the largest integer **less than or equal to** the input — for -4.5, that's -5 (moving toward negative infinity). With negative numbers, this can feel reversed from intuition, since CEIL produces the "smaller-magnitude" (less negative) result and FLOOR the "larger-magnitude" (more negative) one.

**Why the Other Options Are Wrong:**
B. Reverses CEIL and FLOOR's actual directions.
C, D. Both incorrectly collapse the two functions to the same result.

**Concept Tested:** CEIL (toward +∞) vs. FLOOR (toward −∞) with negative inputs.

**Exam Trap:** Applying "CEIL rounds up in magnitude, FLOOR rounds down in magnitude" intuition from positive numbers directly to negative numbers, where the direction (toward +∞ or −∞) is what actually matters, not magnitude.

---

**Question 4**
What happens when you execute `SELECT SQRT(-4) FROM dual;`?

A. Returns 2, taking the absolute value first.
B. Returns NULL.
C. Raises ORA-01428: argument '-4' is out of range — Oracle's SQRT does not support complex/imaginary results.
D. Returns -2.

**Correct Answer:** C

**Detailed Explanation:**
Oracle's NUMBER-based `SQRT` function has no representation for imaginary numbers. Passing a negative argument is outside its valid domain, and Oracle raises ORA-01428 rather than silently returning NULL, an absolute value, or a negative root.

**Why the Other Options Are Wrong:**
A, D. Both invent behavior (absolute-valuing or negating) that Oracle does not perform.
B. NULL is reserved for propagation through a NULL *operand* — a negative number is a valid, non-null input that's simply out of SQRT's supported domain, triggering an explicit error instead.

**Concept Tested:** SQRT's domain restriction on negative inputs.

**Exam Trap:** Confusing "out-of-domain input causes an error" with "NULL input causes NULL propagation" — these are two distinct failure modes.

---

**Question 5**
For a row where `COMMISSION_PCT IS NULL`, what does `ABS(commission_pct)` return?

A. 0
B. NULL
C. An error
D. 1

**Correct Answer:** B

**Detailed Explanation:**
Consistent with the general NULL-propagation rule for single-row functions (as with arithmetic operators in Topic 1), `ABS` applied to a NULL operand returns NULL — there's no special-case substitution.

**Concept Tested:** NULL propagation through single-row mathematical functions.

**Exam Trap:** Assuming a function specifically designed to "normalize" values (like ABS, which strips sign) might therefore also "normalize" NULL to something concrete like 0 — it does not.

---

### Subtopic 5: Miscellaneous Functions (COALESCE & NULLIF)

**Question 1**
What is the key behavioral difference between `COALESCE` and a chain of nested `NVL` calls that achieves the same fallback logic?

A. There is no difference; COALESCE is purely syntactic sugar with identical runtime behavior.
B. COALESCE evaluates its arguments left to right and stops as soon as it finds the first non-null one — later arguments are never evaluated at all — whereas equivalent nested NVL calls may end up evaluating expressions that COALESCE would have skipped.
C. NVL can accept more than two arguments, exactly like COALESCE; the two functions are functionally identical in argument count.
D. COALESCE only works with NUMBER types; NVL works with any data type.

**Correct Answer:** B

**Detailed Explanation:**
COALESCE is documented to evaluate each expression **only as needed**: once a non-null value is found, remaining expressions are never evaluated. This matters most when a later, unevaluated argument would otherwise raise an error (e.g., a division by zero) or trigger an expensive subquery — COALESCE avoids that cost entirely if an earlier argument already resolved the value. Nested NVL calls don't carry this same short-circuit guarantee as cleanly, since each NVL call's second argument still gets evaluated as part of constructing that call.

**Why the Other Options Are Wrong:**
A. Understates a real, practically important distinction.
C. NVL is strictly two-argument; only COALESCE supports an arbitrary-length list.
D. Both functions work generically across compatible data types — this option is fabricated.

**Concept Tested:** COALESCE's short-circuit (lazy) evaluation of arguments.

**Exam Trap:** Treating COALESCE as "just NVL with more arguments" without recognizing its distinct evaluation-order guarantee.

---

**Question 2**
What does `NULLIF(commission_pct, 0)` return when `commission_pct = 0`? What does it return when `commission_pct = 0.1`?

A. NULL, and 0.1, respectively.
B. 0, and 0.1, respectively.
C. NULL, and NULL, respectively.
D. 0.1, and 0, respectively.

**Correct Answer:** A

**Detailed Explanation:**
`NULLIF(expr1, expr2)` returns NULL if `expr1 = expr2`; otherwise, it returns `expr1` unchanged. When commission_pct is 0, it equals the comparison value 0, so NULLIF returns NULL. When commission_pct is 0.1, it does not equal 0, so NULLIF simply returns 0.1 as-is.

**Why the Other Options Are Wrong:**
B. Reverses the "equal → NULL" rule.
C. Wrongly nulls out the non-matching case too.
D. Swaps the two results.

**Concept Tested:** NULLIF's equal-values-become-NULL logic.

**Exam Trap:** Confusing NULLIF's direction — it turns a *matching* value into NULL, which is the opposite of what many people instinctively expect from a NULL-related function (most NULL-handling functions instead replace NULL *with* something).

---

**Question 3**
Why is `NULLIF(commission_pct, 0)` a useful expression to nest inside a division, such as `salary / NULLIF(commission_pct, 0)`?

A. It ensures the divisor is always a positive number.
B. It converts a would-be division-by-zero (when commission_pct is 0) into a division by NULL instead, which cleanly yields NULL rather than raising ORA-01476.
C. It rounds the divisor to the nearest whole number before dividing.
D. It has no effect on division behavior; it's purely cosmetic.

**Correct Answer:** B

**Detailed Explanation:**
Recall from Topic 1 that division by the literal 0 raises ORA-01476, while division by NULL merely propagates to NULL silently. Wrapping a potentially-zero divisor in `NULLIF(divisor, 0)` converts the dangerous "divide by zero" case into a harmless "divide by NULL" case — a common, practical certification-tested pattern combining two distinct rules from earlier topics.

**Why the Other Options Are Wrong:**
A. NULLIF doesn't enforce sign; it only checks equality against the comparison value.
C. No rounding occurs.
D. This is a functionally significant guard, not cosmetic.

**Concept Tested:** Combining NULLIF with division to convert a zero-divisor error into a NULL result.

**Exam Trap:** Not connecting this pattern back to the earlier division-by-zero (ORA-01476) vs. NULL-propagation distinction — certification exams frequently combine concepts across what look like separate topics.

---

**Question 4**
Which two statements about COALESCE are TRUE? **(Choose two)**

A. COALESCE requires at least two arguments; a single-argument call is invalid syntax.
B. If every argument in a COALESCE call evaluates to NULL, the overall result is NULL.
C. COALESCE can only compare arguments of the exact same declared data type; no implicit conversion is ever attempted.
D. COALESCE(expr) with only one argument is legal and simply returns expr unchanged (a degenerate case).

**Correct Answers:** A and B

**Detailed Explanation:**
COALESCE requires two or more expressions — the correct minimum is 2, not 1, since a single-argument version would be trivial and is explicitly disallowed. If none of the supplied expressions are non-null, the final result correctly falls through to NULL, exactly like NVL cascades.

**Why the Other Options Are Wrong:**
C. Oracle does attempt implicit conversion among compatible data types across COALESCE's arguments, similar to other multi-argument comparison contexts.
D. Contradicts A — single-argument COALESCE is not legal syntax.

**Concept Tested:** COALESCE's minimum argument count and its NULL-if-all-NULL fallback behavior.

**Exam Trap:** Assuming COALESCE has no minimum argument requirement, when in fact at least 2 are mandatory.

---

### Subtopic 6: Nesting of Functions & SQL Expressions

**Question 1**
For nested single-row (scalar) functions like `UPPER(SUBSTR(last_name, 1, 3))`, in what order does Oracle evaluate them?

A. Outside-in: UPPER is applied first to the whole column, then SUBSTR extracts from that result.
B. Inside-out: SUBSTR is evaluated first (extracting the first 3 characters), and its result is then passed into UPPER.
C. Left-to-right across the entire expression, regardless of nesting.
D. The order is nondeterministic and can vary between executions.

**Correct Answer:** B

**Detailed Explanation:**
Nested single-row functions evaluate from the innermost function outward — exactly like nested function calls in most programming languages. SUBSTR's result becomes UPPER's actual input.

**Why the Other Options Are Wrong:**
A. Reverses the true evaluation order.
C. "Left-to-right" isn't the relevant axis here; nesting depth is.
D. Evaluation order for nested scalar functions is fully deterministic and specified, not arbitrary.

**Concept Tested:** Inside-out evaluation order for nested single-row functions.

**Exam Trap:** Misreading nested function syntax and assuming the outermost function name signals what happens "first."

---

**Question 2**
Is there a documented maximum nesting depth for single-row (scalar) functions, analogous to the 2-level limit on nested aggregate functions?

A. Yes — scalar functions are also capped at 2 levels of nesting.
B. No — single-row functions may be nested to an arbitrary depth; the 2-level restriction applies specifically to aggregate (group) functions, not scalar ones.
C. Yes — scalar functions are capped at 5 levels.
D. No single-row function can ever be nested inside another single-row function.

**Correct Answer:** B

**Detailed Explanation:**
The "nest to a depth of 2" rule from earlier in this topic applies specifically to **aggregate/group functions**. Single-row (scalar) functions carry no such documented depth limit — you can nest `UPPER(SUBSTR(TRIM(...), ...))` and continue as deep as the logic requires.

**Why the Other Options Are Wrong:**
A, C. Both invent numeric limits that don't exist for scalar functions.
D. Directly contradicts extremely common, fully legal usage.

**Concept Tested:** Aggregate-function nesting limits do not carry over to scalar-function nesting.

**Exam Trap:** Overgeneralizing the aggregate-specific 2-level nesting rule to all functions indiscriminately.

---

**Question 3**
Which of the following mixed nestings is **legal** without requiring any GROUP BY?

A. `SUM(NVL(commission_pct, 0) * salary)` — an aggregate function wrapping a scalar-function expression.
B. `SUM(AVG(salary))` with no GROUP BY.
C. `MAX(COUNT(*))` with no GROUP BY.
D. `AVG(MIN(salary))` with no GROUP BY.

**Correct Answer:** A

**Detailed Explanation:**
Wrapping a scalar expression (here, `NVL(commission_pct,0) * salary`, a per-row calculation) inside a single aggregate function is completely ordinary — the aggregate function (SUM) is only nested one level around scalar logic, not around *another aggregate function*. This is a universally legal, extremely common pattern (e.g., computing total effective pay including commission). B, C, and D all nest one aggregate function directly inside another aggregate function with no GROUP BY, triggering ORA-00978.

**Why the Other Options Are Wrong:**
B, C, D. All are aggregate-inside-aggregate nestings lacking the required GROUP BY.

**Concept Tested:** Aggregate(scalar-expression) is always legal; aggregate(aggregate) needs GROUP BY.

**Exam Trap:** Miscategorizing "a function call appears inside SUM(...)" as automatically triggering the aggregate-nesting restriction, without checking whether the inner call is itself an aggregate function.

---

**Question 4**
Evaluate: `SELECT ROUND(AVG(NVL(commission_pct, 0)), 2) FROM employees;` — which classification best describes this expression's structure, and is it legal?

A. Illegal — three levels of nesting exceeds any permitted depth.
B. Legal — reading inside-out: NVL (scalar) resolves NULLs per row first, AVG (aggregate) then summarizes across all rows, and ROUND (scalar) finally formats that single aggregate result. This is scalar(aggregate(scalar(...))) — never aggregate-inside-aggregate — so no GROUP BY is required.
C. Illegal — aggregate functions cannot have their result passed into another function.
D. Legal, but only if a GROUP BY clause is added.

**Correct Answer:** B

**Detailed Explanation:**
This expression nests three layers, but only *one* of them (AVG) is an aggregate function — the other two (NVL and ROUND) are scalar. The restriction that requires GROUP BY only kicks in for aggregate-inside-aggregate nesting; wrapping an aggregate's input or output in scalar functions is unrestricted and doesn't require GROUP BY at all, since it still produces one final aggregated value for the whole table.

**Why the Other Options Are Wrong:**
A. Nesting depth alone isn't the limiting factor — the *type* of function being nested inside another aggregate is.
C. Aggregate results are routinely passed into scalar functions (e.g., ROUND, TO_CHAR) — a completely standard pattern.
D. No GROUP BY is needed here precisely because there's no aggregate-inside-aggregate nesting.

**Concept Tested:** Correctly identifying which layers of a nested expression are scalar vs. aggregate, since only aggregate-inside-aggregate triggers the GROUP BY requirement.

**Exam Trap:** Reflexively counting nesting *depth* rather than analyzing nesting *composition* (which functions are aggregate vs. scalar at each layer) — this is the most sophisticated version of the nesting trap in this topic, deliberately combining Subtopic 2's core rule with pure scalar wrapping.

---

**Question 5**
What does `SELECT UPPER(SUBSTR('Oracle Certification', 1, 6)) || '...' FROM dual;` return?

A. `ORACLE...`
B. `Oracle...`
C. `ORACLE CERTIFICATION...`
D. An error — you cannot concatenate the result of nested functions with a literal.

**Correct Answer:** A

**Detailed Explanation:**
Working inside-out: `SUBSTR('Oracle Certification', 1, 6)` extracts the first 6 characters → `'Oracle'`. `UPPER('Oracle')` → `'ORACLE'`. Finally, `'ORACLE' || '...'` → `'ORACLE...'`.

**Why the Other Options Are Wrong:**
B. Skips the UPPER step entirely.
C. Wrongly extracts more characters than SUBSTR's length argument specifies.
D. Concatenating a function's result with a literal via `||` is completely standard and always legal.

**Concept Tested:** Full end-to-end inside-out evaluation of a realistic multi-function, multi-operator expression.

**Exam Trap:** Losing track of one step (commonly, forgetting to apply the outermost function) when manually tracing a multi-layer nested expression.