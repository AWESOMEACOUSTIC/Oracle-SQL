# Module 3, Topic 9: Overloading Procedure and Function

---

## 1. What Is Overloading?

**Overloading** means defining **multiple procedures (or functions) with the same name**, but with **different parameter lists** (different number of parameters, and/or different datatypes), **within the same package**. Oracle decides **which version to actually run** based on the arguments you pass at the call site — this is called the **matching/resolution mechanism**.

The core idea: **same name, different "shape" of input, different implementation** — and the caller doesn't have to remember multiple differently-named versions for what is conceptually "the same operation."

---

## 2. Why Does Overloading Exist? What Problem Does It Solve?

Consider this real situation: your team has a function that calculates a discount, but the discount can be looked up **either by customer ID or by customer email** — both are valid ways different callers might have the customer identified.

**Without overloading**, you'd be forced to invent artificially different names:
```sql
get_discount_by_id (p_customer_id NUMBER) RETURN NUMBER;
get_discount_by_email (p_email VARCHAR2) RETURN NUMBER;
```
This works, but it pushes the burden onto every **caller** to remember and choose the "right" name based on what they have available — even though conceptually, both are doing the exact same thing: "get this customer's discount."

**With overloading:**
```sql
get_discount (p_customer_id NUMBER) RETURN NUMBER;
get_discount (p_email VARCHAR2) RETURN NUMBER;
```
Now callers just call `get_discount(...)` with whatever they have — Oracle figures out which version applies based on the argument's datatype. This is a cleaner, more intuitive API: **one conceptual operation, one name, multiple valid ways to invoke it.**

This mirrors a very natural real-world pattern: think of a business process like "look up an employee" — sometimes you have their ID, sometimes their email, sometimes their badge number. Conceptually it's the *same* action; overloading lets your code reflect that.

---

## 3. Why Is It Used? (The Business Case)

- **Flexible input handling**: accepting different "shapes" of the same logical input (ID vs. code vs. name) without forcing callers to remember multiple function names.
- **Supporting optional/default-like behavior through structure** rather than just default parameter values — e.g., a "simple" version with fewer parameters and a "detailed" version with more, both under the same name.
- **API evolution**: adding a new overloaded version to support a new input type, without touching or breaking any existing callers of the original version.
- **Readability and discoverability**: developers exploring a package see one meaningful name (`get_discount`) rather than having to guess between several similarly-named variants (`get_discount_by_id`, `get_discount_by_email`, `get_discount_v2`...).

---

## 4. Syntax

Overloading isn't a special keyword — it's simply **declaring multiple sub programs with the same name but different parameter signatures**, inside the same package.

```sql
CREATE OR REPLACE PACKAGE pkg_customer_lookup
IS
    FUNCTION get_discount (p_customer_id IN NUMBER) RETURN NUMBER;
    FUNCTION get_discount (p_email IN VARCHAR2) RETURN NUMBER;
END pkg_customer_lookup;
/

CREATE OR REPLACE PACKAGE BODY pkg_customer_lookup
IS
    FUNCTION get_discount (p_customer_id IN NUMBER) RETURN NUMBER
    IS
        v_discount NUMBER;
    BEGIN
        SELECT discount_pct INTO v_discount FROM customers WHERE customer_id = p_customer_id;
        RETURN v_discount;
    END get_discount;

    FUNCTION get_discount (p_email IN VARCHAR2) RETURN NUMBER
    IS
        v_discount NUMBER;
    BEGIN
        SELECT discount_pct INTO v_discount FROM customers WHERE email = p_email;
        RETURN v_discount;
    END get_discount;
END pkg_customer_lookup;
/
```

Calling either version:
```sql
DECLARE
    v_d1 NUMBER;
    v_d2 NUMBER;
BEGIN
    v_d1 := pkg_customer_lookup.get_discount(501);              -- resolves to the NUMBER version
    v_d2 := pkg_customer_lookup.get_discount('a@example.com');  -- resolves to the VARCHAR2 version
END;
/
```

---

## 5. The Resolution Rules — What Actually Makes Two Signatures "Different Enough"

This is the part that trips people up most, so let's be precise.

**Valid ways to overload (Oracle can tell them apart):**
1. **Different number of parameters.**
2. **Different parameter datatypes** (from a different *datatype family* — e.g., `NUMBER` vs. `VARCHAR2` vs. `DATE`).

**NOT valid overloading (Oracle cannot tell them apart — compile error):**
1. **Parameter mode alone differs** (`IN` vs. `OUT` vs. `IN OUT`) — mode is not considered for overload resolution at all.
2. **Parameter name alone differs** — the internal parameter names (`p_customer_id` vs. `p_cust_id`) are irrelevant to resolution; only the datatype/count/order matters.
3. **Return type alone differs (for functions)** — two functions with identical parameter lists but different `RETURN` types are **not** valid overloads; Oracle has no way to pick between them based on a call alone (a call site doesn't necessarily reveal what return type it expects).
4. **Subtypes within the same family that Oracle treats as equivalent for resolution** — e.g., `NUMBER` vs. `PLS_INTEGER` in certain contexts can be ambiguous; more relevantly, `VARCHAR2` vs. `CHAR` are both "character family" and can create ambiguity depending on the call.

### Example of INVALID overloading:
```sql
FUNCTION get_discount (p_customer_id IN NUMBER) RETURN NUMBER;
FUNCTION get_discount (p_customer_id IN NUMBER) RETURN VARCHAR2;  -- ERROR: same parameter list, only return type differs
```

### Example of INVALID overloading:
```sql
PROCEDURE update_record (p_id IN NUMBER);
PROCEDURE update_record (p_id OUT NUMBER);  -- ERROR: differs only by mode, not allowed
```

---

## 6. Detailed Explanation — How Oracle Resolves an Overloaded Call

When you call an overloaded name, Oracle looks at:
1. **The number of arguments** you supplied.
2. **The datatype(s)** of those arguments.

...and matches against all available overloaded versions to find the **single best fit**. If it finds exactly one unambiguous match, that version runs. If it finds **none** that fit, or **more than one** that fit equally well (a genuine ambiguity — e.g., calling with a literal that could implicitly convert to more than one candidate type), Oracle raises a compile-time error (`PLS-00307: too many declarations of '...' match this call`).

This is resolved **at compile time**, not runtime — meaning any ambiguity is caught immediately when the calling code is compiled, not discovered later in production.

---

## 7. When to Use / When Not to Use Overloading

**Use overloading when:**
- The **same conceptual operation** genuinely has multiple valid "shapes" of meaningful input (different identifiers, different levels of detail) that callers might naturally have available.
- You want to **preserve a clean, memorable name** across those variations instead of inventing artificial name suffixes.
- You're **extending** an existing sub program's usability (e.g., adding a new way to call it) without breaking any existing callers of the original signature.

**Don't use overloading when:**
- The operations are **conceptually different**, even if related — forcing unrelated behaviors under one shared name for the sake of "cleverness" harms readability rather than helping it. (E.g., `process_order()` that sometimes creates an order and sometimes cancels one, depending on which overload — that's a misuse; those are different verbs and deserve different names.)
- **Default parameter values** would solve the same problem more simply — if the "variation" is really just "some parameters are optional," a single sub program with `DEFAULT` values (Topic 7) is usually simpler and clearer than creating multiple overloaded versions.
- The distinction between overloads would only be **the return type**, or only **parameter mode** — these aren't valid overload strategies at all, as covered above.

---

## 8. Common Mistakes & Misconceptions

1. **Mistake**: Trying to overload two functions differing only in `RETURN` type → compile error, since Oracle can't resolve based on return type from a call site.
2. **Mistake**: Trying to overload two procedures differing only in parameter **mode** (`IN` vs `OUT`) → not valid; mode isn't part of the resolution signature.
3. **Misconception**: "I can overload standalone (non-packaged) procedures/functions too." → **Overloading only works within packages** (or, more precisely, within a single PL/SQL block/subprogram scope like nested subprograms) — you **cannot** create two standalone schema-level objects with the same name; the schema itself doesn't allow duplicate object names regardless of parameter differences.
4. **Mistake**: Creating ambiguous overloads where a caller's literal value could match more than one version — e.g., overloading on `NUMBER` and a numeric subtype in a way that a plain numeric literal could satisfy either, leading to a compile-time ambiguity error on the calling code.
5. **Misconception**: "Overloading is resolved at runtime, based on what the caller intends." → False — it's a **compile-time** decision based purely on the argument's declared datatype at the call site.

---

## 9. Edge Cases to Be Aware Of

- **Important, frequently tested fact**: overloading is a **package-level (or nested-subprogram-level) feature only** — it does not work for standalone `CREATE PROCEDURE`/`CREATE FUNCTION` objects at the schema level. This is one of the strongest practical reasons (beyond organization/encapsulation) that real systems package almost everything.
- If two overloaded versions could both technically accept a given literal (e.g., a numeric literal that's compatible with two different numeric-family parameter types), Oracle raises an ambiguity error **at compile time** for the calling code — it will not "guess" or pick one silently.
- Overloading combines naturally with **default parameter values** — you can have an overloaded version with fewer parameters *and* another version with more parameters that include defaults; just be careful this doesn't itself introduce ambiguity for calls that could match either shape.

---

## 10. Interview-Level / Practical Notes

- A very common interview question: *"Can you overload procedures/functions using only different parameter modes or a different return type?"* — No to both; this is one of the most frequently tested "gotcha" facts about overloading in PL/SQL specifically (interviewers know it trips people up).
- *"Where can overloading be used — standalone objects, or only packages?"* — **Only within packages** (or nested subprogram scope) — standalone procedures/functions cannot be overloaded at all, since schema object names must be unique.
- Being able to correctly distinguish "valid overload" vs. "invalid overload" scenarios on sight is a strong, testable signal of real understanding versus surface memorization — expect this to come up in both interviews and any formal PL/SQL certification exam.

---

## Things You Must Remember

- Overloading = same name, **different parameter count and/or datatype**, within the **same package** (or nested scope) — never at the standalone schema-object level.
- Valid overload differentiators: **number of parameters**, **datatypes** of parameters.
- **Invalid** overload differentiators (compile error): parameter **mode** alone, parameter **name** alone, **return type** alone (for functions).
- Resolution happens at **compile time**, based on the arguments' datatypes at the call site — never at runtime, never "guessed."
- Ambiguous calls (matching more than one overload equally well) raise a compile-time error, not a silent/arbitrary choice.

## How to Recognize This Concept

Think **overloading** when a requirement or situation describes:
- "The **same operation**, but callers might have **different kinds of input** available" (ID vs. code vs. name, single value vs. a list, etc.).
- A desire to keep **one clean, memorable name** for a family of closely related variations, rather than several arbitrarily-suffixed names.
- Extending an existing package's function to accept a **new kind of input** while keeping full backward compatibility for existing callers.

If the "variation" you're considering is really just **some parameters becoming optional** (not a genuinely different input shape) — that's usually better solved with **default parameter values** (Topic 7) instead of overloading. And if the "variation" is really a **different action entirely** — that deserves a distinctly named sub program, not an overload.

---

## Exercises

1. **(Basic overload)** Inside a package `pkg_product_lookup`, declare and implement two overloaded functions named `get_price` — one accepting a `product_id (NUMBER)`, and one accepting a `product_code (VARCHAR2)` — both returning the product's price as `NUMBER`.

2. **(Spot the invalid overload)** Explain why each of the following pairs is **not** a valid overload, and what error you'd expect:
   - a)
     ```sql
     FUNCTION calc_total (p_amount NUMBER) RETURN NUMBER;
     FUNCTION calc_total (p_amount NUMBER) RETURN VARCHAR2;
     ```
   - b)
     ```sql
     PROCEDURE process_payment (p_amount IN NUMBER);
     PROCEDURE process_payment (p_amount OUT NUMBER);
     ```
   - c)
     ```sql
     FUNCTION get_status (p_order_id NUMBER) RETURN VARCHAR2;
     FUNCTION get_status (p_id NUMBER) RETURN VARCHAR2;
     ```

3. **(Overload vs. default parameter judgment)** A requirement says: *"The `send_notification` procedure should work whether or not a custom message is provided — if not provided, use a standard default message."* Would you solve this with overloading or a default parameter value? Justify your choice in 2–3 sentences.

4. **(Realistic business scenario)** Business requirement: *"Our `pkg_search` package needs a `find_customer` operation. Sometimes the caller has a `customer_id`, sometimes only a `phone_number`, and sometimes only a `national_id` (a text code). All three should feel like the same conceptual operation to callers."* Design the package specification (declarations only) using overloading to support all three input types, each returning a `customer_id (NUMBER)`.

5. **(Ambiguity reasoning)** Suppose a package has these two overloaded procedures:
   ```sql
   PROCEDURE log_event (p_code IN NUMBER);
   PROCEDURE log_event (p_code IN VARCHAR2);
   ```
   A caller writes: `log_event(101);` Explain why this is unambiguous and which version runs. Now suppose a caller writes `log_event(NULL);` — is this ambiguous, and why might this be a genuinely tricky edge case? (Think about what datatype an untyped `NULL` literal has on its own.)

6. **(Standalone vs. package reasoning)** A junior developer tries to create two standalone functions:
   ```sql
   CREATE FUNCTION get_total (p_order_id NUMBER) RETURN NUMBER ...
   CREATE FUNCTION get_total (p_customer_id NUMBER) RETURN NUMBER ...
   ```
   and is confused why the second `CREATE` fails. Explain what's actually going wrong here, and what change would make this design work as intended.

---

*Share your attempts whenever you're ready. Next up: Creating a Package Using Procedures and Functions.*
