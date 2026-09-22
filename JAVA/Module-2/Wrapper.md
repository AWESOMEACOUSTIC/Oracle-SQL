# Java Competitive Exam Preparation
## Chunk 6: Wrapper Classes

Every primitive type covered back in Chunk 1 has a corresponding object based counterpart, called a wrapper class. This chunk covers what wrapper classes are, why Java needs them at all given that primitives already work perfectly well for ordinary calculations, and the specific API each one offers. As always, examples continue to draw on the banking scenario, this time using wrapper classes to solve two genuine gaps primitives cannot fill on their own: representing "no value yet" and parsing raw text input into usable numbers.

---

## Table of Contents

1. What is a Wrapper Class and Why Do We Use Wrapper Classes
2. Primitive to Wrapper Mapping and the Wrapper Class Hierarchy
3. Converting Primitives to Wrappers and Vice Versa
4. Convert int to String and String to int
5. Integer Wrapper Class with Lend A Hand
6. Long and Float Wrapper Class with Lend A Hand
7. Double Wrapper Class with Lend A Hand
8. Byte, Short, Character and Boolean Wrapper Class with Lend A Hand
9. Where Wrappers Can Be Used
10. Consolidated Quiz: All Six Chunks Together
11. Programming Practice and Solutions
12. Revision Summary

---

## 1. What is a Wrapper Class and Why Do We Use Wrapper Classes

### Concept Explanation

A wrapper class is an ordinary Java class whose entire purpose is to hold a single primitive value inside an object. Every one of the eight primitive types from Chunk 1 has a corresponding wrapper class in `java.lang`: `int` has `Integer`, `double` has `Double`, `boolean` has `Boolean`, and so on. Once a primitive value is placed inside its wrapper, it can be treated exactly like any other object: it can be `null`, it can have methods called on it, and it can be used anywhere Java specifically requires an object rather than a primitive.

**Why this matters: a gap primitives genuinely cannot fill.** Consider giving `BankAccount` a field to track a pending, not yet assigned account number, before the bank's back office finishes processing a new application. An `int` field can never represent "no number yet" as a distinct state; it always holds some actual numeric value, defaulting to `0`, which is itself a perfectly valid account number and therefore ambiguous. An `Integer` field, being a reference type, can genuinely be `null`, cleanly representing "not yet assigned" as a state entirely distinct from any real number, including `0`.

```java
public class PendingApplication {
    private String applicantName;
    private Integer assignedAccountNumber; // null until the bank assigns one

    public PendingApplication(String applicantName) {
        this.applicantName = applicantName;
        this.assignedAccountNumber = null;
    }

    public boolean isProcessed() {
        return assignedAccountNumber != null;
    }
}
```

`isProcessed()` reads naturally and correctly precisely because `assignedAccountNumber` can genuinely be `null`, something no `int` field could ever express.

**Other reasons wrapper classes exist.** Wrapper classes provide useful `static` utility methods that have nowhere else to live, since a primitive type itself cannot have methods, such as `Integer.parseInt(String)` for converting text into a number. They also provide named constants such as `Integer.MAX_VALUE`, already used back in Chunk 1. And, looking ahead beyond this course, many parts of the Java standard library, particularly collections, are specifically designed to work only with objects, never with primitives directly, which makes wrapper classes essential for using primitive values in those contexts at all.

### Important Notes

- Every primitive type has exactly one corresponding wrapper class.
- Wrapper objects can be `null`; primitives never can.
- Wrapper classes provide static utility methods and named constants that primitives, having no methods of their own, cannot offer directly.
- A wrapper object holds one single value, set at construction, and, like `String`, is immutable: there is no way to change the value a wrapper object holds after it is created.

### Quick Check

Why can `Integer.MAX_VALUE`, first introduced back in Chunk 1, be accessed even though `int` itself is a primitive with no methods or fields of its own?

Because `MAX_VALUE` is not a member of `int` at all; it is a `public static final` constant declared on the `Integer` class, the wrapper class associated with `int`. Referencing it through `Integer.MAX_VALUE` works exactly like referencing any other static member through a class name, exactly as covered in Chunk 2's static keyword section.

---

## 2. Primitive to Wrapper Mapping and the Wrapper Class Hierarchy

### Concept Explanation

The eight primitive types map to eight wrapper classes with mostly predictable names, and two irregular exceptions worth memorizing specifically because exams frequently test them.

| Primitive | Wrapper Class |
|---|---|
| `byte` | `Byte` |
| `short` | `Short` |
| `int` | `Integer` |
| `long` | `Long` |
| `float` | `Float` |
| `double` | `Double` |
| `char` | `Character` |
| `boolean` | `Boolean` |

`int` maps to `Integer`, not the more predictable sounding "Intager" or "Int," and `char` maps to `Character`, not "Char." Every other pairing simply capitalizes the primitive's name directly.

**The wrapper class hierarchy.** Every wrapper class, like every class in Java, ultimately extends `Object`, exactly as covered in Chunk 3. The six numeric wrapper classes, `Byte`, `Short`, `Integer`, `Long`, `Float`, and `Double`, additionally all extend a shared abstract class called `Number`, which is a genuine, real world example of the abstract class concept from Chunk 4: `Number` declares abstract methods like `intValue()`, `doubleValue()`, `longValue()`, and more, and each concrete wrapper class provides its own implementation converting its own stored value to each of those other numeric types.

```
Object
 └── Number (abstract)
      ├── Byte
      ├── Short
      ├── Integer
      ├── Long
      ├── Float
      └── Double
 └── Character
 └── Boolean
```

`Character` and `Boolean` extend `Object` directly, not `Number`, since neither represents a numeric quantity in the way the other six do.

**Using `Number`'s conversion methods.**

```java
Integer accountNumber = 1024;
double asDouble = accountNumber.doubleValue();
long asLong = accountNumber.longValue();
System.out.println(asDouble + " " + asLong);
```

Output: `1024.0 1024`

`doubleValue()` and `longValue()` are inherited from `Number`, exactly the same inherited method mechanism from Chunk 3, since `Integer` is a concrete subclass providing real implementations for every abstract method `Number` declares.

### Important Notes

- The two irregular wrapper names are `int` to `Integer` and `char` to `Character`; every other primitive simply capitalizes to its wrapper name.
- `Byte`, `Short`, `Integer`, `Long`, `Float`, and `Double` all extend the abstract class `Number`, gaining conversion methods like `intValue()` and `doubleValue()` for free through inheritance.
- `Character` and `Boolean` extend `Object` directly, since neither is a numeric type.

### Quick Check

Could a method be written to accept any of `Byte`, `Short`, `Integer`, `Long`, `Float`, or `Double` using a single parameter type?

Yes, by declaring the parameter as `Number`, exactly the same polymorphism principle from Chunk 3 and Chunk 4: since every one of those six classes is, through inheritance, genuinely a `Number`, a `Number` typed parameter can accept any of them, and calling `.doubleValue()` on that parameter would correctly resolve, through dynamic dispatch, to whichever concrete wrapper class's own implementation the actual argument is.

---

## 3. Converting Primitives to Wrappers and Vice Versa

### Concept Explanation

Converting a primitive value into its wrapper object is called **boxing**; converting a wrapper object back into its primitive value is called **unboxing**. Java supports both manually and automatically.

**Manual boxing and unboxing.**

```java
int amount = 500;
Integer boxed = Integer.valueOf(amount);
int unboxed = boxed.intValue();
```

`Integer.valueOf(int)` is the standard, recommended way to box a primitive manually. `new Integer(int)` also exists but is deprecated since Java 9, specifically in favor of `valueOf`, for reasons covered below.

**Autoboxing and auto-unboxing.** Since Java 5, the compiler performs this conversion automatically wherever it is needed, inserting the equivalent `valueOf` or `xxxValue` call behind the scenes without you writing it explicitly.

```java
Integer boxed = 500; // autoboxing: compiler inserts Integer.valueOf(500)
int unboxed = boxed; // auto-unboxing: compiler inserts boxed.intValue()
```

**The `Integer` cache and a classic `==` trap.** For performance, `Integer.valueOf(int)`, and therefore ordinary autoboxing too, reuses a small pool of pre-created `Integer` objects for values from `-128` to `127` inclusive, rather than always allocating a new object. Values outside that range always allocate a genuinely new object.

```java
Integer a = 100;
Integer b = 100;
Integer c = 200;
Integer d = 200;

System.out.println(a == b);
System.out.println(c == d);
```

Output:
```
true
false
```

`100` falls inside the cached range, so both `a` and `b` are autoboxed to the exact same pooled object, making `a == b` true, exactly the same reference sharing behavior the string pool demonstrated in Chunk 5. `200` falls outside the cached range, so `c` and `d` are two separately allocated objects, making `c == d` false, even though their actual values are identical. `.equals()`, not `==`, should always be used to compare wrapper objects by value, exactly the same lesson Chunk 5 taught for `String`.

**The `NullPointerException` risk of unboxing `null`.** Auto-unboxing a `null` wrapper reference throws a `NullPointerException` at runtime, since there is no primitive value to extract from a reference that points to nothing at all.

```java
Integer accountNumber = null;
int x = accountNumber; // throws NullPointerException at runtime
```

This is exactly why `PendingApplication.assignedAccountNumber` from Section 1 must always be read through a `null` check, such as `isProcessed()`'s comparison, before ever being unboxed into a primitive.

### Important Notes

- Boxing converts primitive to wrapper; unboxing converts wrapper to primitive.
- Autoboxing and auto-unboxing happen automatically wherever the compiler can determine one is needed, such as assigning an `int` literal to an `Integer` variable.
- `Integer` caches values from `-128` to `127`; comparing cached wrapper values with `==` can misleadingly appear to work, while the same comparison silently breaks just outside that range.
- Always use `.equals()`, never `==`, to compare wrapper objects by value, exactly the `String` lesson from Chunk 5 applied here too.
- Unboxing a `null` wrapper reference throws `NullPointerException` at runtime; always confirm a wrapper is non `null` before an operation that would trigger unboxing.

### Quick Check

Would `Byte a = 100; Byte b = 100; System.out.println(a == b);` print `true` or `false`?

`true`. `Byte`'s entire value range, `-128` to `127`, fits completely inside the cached range that applies to every integer wrapper class, `Byte`, `Short`, `Integer`, and `Long`, so every possible `Byte` value is always drawn from the shared cache, making `==` comparisons between `Byte` objects always reliable, unlike `Integer`, whose much larger range extends far beyond what is cached.

---

## 4. Convert int to String and String to int

### Concept Explanation

Converting between numeric wrapper types and `String` is one of the most common tasks in ordinary Java code, especially whenever numeric input arrives as raw text, such as from a parsed `StringTokenizer` token, as seen throughout Chunk 5.

**Converting a number to a `String`.** There are three equivalent, common approaches.

```java
int amount = 500;
String s1 = String.valueOf(amount);
String s2 = Integer.toString(amount);
String s3 = "" + amount;
```

All three produce `"500"`. `String.valueOf` and `Integer.toString` are generally preferred in real code, since `"" + amount` relies on `String` concatenation's implicit conversion behavior from Chunk 1 rather than stating the intent directly.

**Converting a `String` to a number.** `Integer.parseInt(String)` returns a primitive `int` directly; `Integer.valueOf(String)` returns a boxed `Integer` object instead, autoboxing style, but otherwise performs the identical parsing.

```java
String input = "750";
int amount = Integer.parseInt(input);
Integer boxedAmount = Integer.valueOf(input);
```

**`NumberFormatException` for invalid input.** If the `String` does not represent a valid number of the target type, both `parseInt` and `valueOf` throw a `NumberFormatException` at runtime.

```java
String badInput = "seven hundred";
int amount = Integer.parseInt(badInput); // throws NumberFormatException
```

**Applying this to parsing a deposit amount.** Extending Chunk 5's parsing work, a raw deposit request arriving as text needs converting before it can be passed to `deposit(double amount)`.

```java
public static boolean processDepositRequest(BankAccount acc, String rawAmount) {
    double amount;
    try {
        amount = Double.parseDouble(rawAmount);
    } catch (NumberFormatException e) {
        System.out.println("Invalid deposit amount: " + rawAmount);
        return false;
    }
    acc.deposit(amount);
    return true;
}
```

This uses a `try catch` block to handle a malformed input gracefully rather than letting the program crash; full exception handling is a topic for a later chunk, but this specific, narrow pattern, catching `NumberFormatException` around a `parse` call, is common enough to be worth recognizing here already.

### Important Notes

- `String.valueOf(number)`, `Integer.toString(number)` and similar per type methods, and `"" + number` all convert a number to text; the first two are generally preferred for clarity.
- `parseXxx(String)` methods, such as `Integer.parseInt`, `Double.parseDouble`, and `Long.parseLong`, return a primitive value directly.
- `valueOf(String)` methods return a boxed wrapper object instead of a primitive, otherwise parsing identically.
- Invalid input to any `parse` or numeric `valueOf` method throws `NumberFormatException` at runtime, not a compile error, since the actual text is only known once the program runs.

### Quick Check

What is the difference in return type between `Integer.parseInt("42")` and `Integer.valueOf("42")`?

`Integer.parseInt("42")` returns a primitive `int`, the value `42`. `Integer.valueOf("42")` returns a boxed `Integer` object wrapping that same value `42`, which may or may not come from the cache described in Section 3 depending on whether `42` falls inside the cached range, which it does here.

---

## 5. Integer Wrapper Class with Lend A Hand

### Concept Explanation

`Integer` is the most commonly used wrapper class, corresponding to `int`, and offers a rich static API beyond simple boxing.

| Member | What it does |
|---|---|
| `Integer.MAX_VALUE` / `Integer.MIN_VALUE` | The boundary values of `int`, from Chunk 1 |
| `Integer.parseInt(String s)` | Parses text into a primitive `int` |
| `Integer.valueOf(...)` | Boxes a primitive, or parses text, into an `Integer` |
| `Integer.toString(int i)` | Converts an `int` into text |
| `Integer.compare(int x, int y)` | Returns negative, zero, or positive, mirroring `x - y`'s sign without risking overflow |
| `Integer.toBinaryString(int i)` | Returns the binary text representation, already used in Chunk 1 |
| `Integer.toHexString(int i)` | Returns the hexadecimal text representation |
| `intValue()` | Instance method, unboxes this `Integer` to a primitive `int` |

**Why `Integer.compare` is safer than manual subtraction.** A naive comparison such as `x - y` risks silent overflow if `x` and `y` are far apart in value, exactly the integer overflow behavior covered in Chunk 1. `Integer.compare(x, y)` avoids this entirely, always returning a correctly signed result regardless of how large `x` and `y` are.

```java
int x = Integer.MAX_VALUE;
int y = -1;
System.out.println(x - y); // overflows, unreliable
System.out.println(Integer.compare(x, y)); // reliably positive
```

### Lend A Hand: Quiz

1. What does `Integer.parseInt("100")` return?
   A. `"100"` as a `String`  B. The primitive `int` value `100`  C. An `Integer` object  D. Compile error

2. Given `Integer a = 50; Integer b = 50;`, what does `a == b` return?
   A. `false`, always  B. `true`, since `50` falls within the cached range  C. Compile error  D. `NullPointerException`

3. What does `Integer.toBinaryString(10)` return?
   A. `"10"`  B. `"1010"`  C. `10`  D. `"A"`

4. Why is `Integer.compare(x, y)` generally preferred over `x - y` for comparison purposes?
   A. It is shorter to type only  B. `x - y` can silently overflow for widely separated values, while `Integer.compare` avoids that risk entirely  C. `x - y` does not compile  D. There is no real difference

### Answers

1. **B.** `parseInt` returns a primitive `int` directly, not a `String` and not a boxed object.

2. **B.** `50` falls within the `-128` to `127` cached range from Section 3, so both `a` and `b` reference the same pooled `Integer` object.

3. **B, `"1010"`.** `10` in binary is `1010`, exactly matching Chunk 1's original coverage of `Integer.toBinaryString`.

4. **B.** Directly subtracting two `int` values risks the exact same silent overflow wraparound covered extensively in Chunk 1, which `Integer.compare` sidesteps entirely by not relying on subtraction internally.

### Programming Practice

1. Write a method `formatAccountNumber(int accountNumber)` that returns the account number's binary representation alongside its ordinary decimal form, both as one combined `String`, using `Integer.toBinaryString` and `Integer.toString`.

### Solution

```java
public class Solution1 {
    static String formatAccountNumber(int accountNumber) {
        return "Decimal: " + Integer.toString(accountNumber) +
               ", Binary: " + Integer.toBinaryString(accountNumber);
    }

    public static void main(String[] args) {
        System.out.println(formatAccountNumber(42));
    }
}
```

Expected output: `Decimal: 42, Binary: 101010`

Why it works: `Integer.toString` and `Integer.toBinaryString` both convert the same underlying `int` value into two different text representations, directly reusing Chunk 1's binary literal coverage in a practical formatting context.

---

## 6. Long and Float Wrapper Class with Lend A Hand

### Concept Explanation

`Long` and `Float` follow the exact same pattern as `Integer`, adapted to their own primitive's range and precision characteristics from Chunk 1.

| Member | What it does |
|---|---|
| `Long.MAX_VALUE` / `Long.MIN_VALUE` | The boundary values of `long` |
| `Long.parseLong(String s)` | Parses text into a primitive `long` |
| `Long.toString(long l)` | Converts a `long` into text |
| `Float.parseFloat(String s)` | Parses text into a primitive `float` |
| `Float.MAX_VALUE` / `Float.MIN_VALUE` | The boundary values of `float` |
| `Float.isNaN(float f)` | Checks whether a value is "not a number," connecting back to Chunk 1's floating point discussion |
| `Float.isInfinite(float f)` | Checks whether a value is infinite |

**Applying `Long` to a running total across an entire bank.** Since a single `int` total across every account in a large bank could realistically approach or exceed `Integer.MAX_VALUE`, `long` is the safer choice for such an aggregate.

```java
public static long totalHoldingsAcrossBranches(Branch[] branches) {
    long total = 0;
    for (Branch b : branches) {
        total += (long) b.getTotalHoldings();
    }
    return total;
}
```

The explicit `(long)` cast here is required because `getTotalHoldings()` returns `double`, and `double` to `long` is a narrowing conversion in Chunk 1's widening chain, truncating any fractional cents, an intentional simplification for this whole currency unit total.

**`Float.isNaN` and `Float.isInfinite`, connecting to Chunk 1.** These directly test the special floating point values Chunk 1 introduced: dividing a positive `float` by `0.0f` produces infinity rather than an exception, and certain undefined operations, like `0.0f / 0.0f`, produce `NaN`.

```java
float result = 0.0f / 0.0f;
System.out.println(Float.isNaN(result));
```

Output: `true`

### Lend A Hand: Quiz

1. What does `Long.parseLong("9000000000")` return, given the value exceeds `Integer.MAX_VALUE`?
   A. Compile error  B. The primitive `long` value `9000000000`, since `long`'s range comfortably accommodates it  C. `NumberFormatException`  D. Truncates to fit `int`

2. What does `Float.isNaN(5.0f)` return?
   A. `true`  B. `false`, since `5.0f` is a perfectly ordinary, valid number  C. Compile error  D. `NaN`

3. Why might `long` be chosen over `int` for a bank-wide total, connecting back to Chunk 1?
   A. `long` is always faster  B. `long`'s much larger range avoids the overflow risk `int` would carry for a sufficiently large aggregate total  C. `int` cannot be summed in a loop  D. There is no real reason

### Answers

1. **B.** `9000000000` comfortably fits within `long`'s far larger range from Chunk 1, so parsing succeeds and returns that exact value as a primitive `long`.

2. **B.** `5.0f` is an entirely ordinary, valid floating point value, not the special `NaN` value, so `isNaN` correctly reports `false`.

3. **B.** Exactly as covered in Chunk 1, `int` silently overflows and wraps around once a running total exceeds roughly 2.1 billion; `long`'s vastly larger range makes that overflow effectively impossible for any realistic banking total.

### Programming Practice

1. Write a method that takes a raw `String` amount, attempts `Float.parseFloat` on it, and prints whether the result `isNaN` or `isInfinite`, alongside the parsed value itself, testing it against `"3.5"`, `"Infinity"`, and `"NaN"`.

### Solution

```java
public class Solution1 {
    static void inspectFloatInput(String raw) {
        float value = Float.parseFloat(raw);
        System.out.println(raw + " -> value=" + value + ", isNaN=" + Float.isNaN(value) + ", isInfinite=" + Float.isInfinite(value));
    }

    public static void main(String[] args) {
        inspectFloatInput("3.5");
        inspectFloatInput("Infinity");
        inspectFloatInput("NaN");
    }
}
```

Expected output:
```
3.5 -> value=3.5, isNaN=false, isInfinite=false
Infinity -> value=Infinity, isNaN=false, isInfinite=true
NaN -> value=NaN, isNaN=true, isInfinite=false
```

Why it works: `Float.parseFloat` specifically recognizes the literal text `"Infinity"` and `"NaN"` as valid special values, not as malformed input, correctly parsing them into their corresponding special `float` values rather than throwing `NumberFormatException`.

---

## 7. Double Wrapper Class with Lend A Hand

### Concept Explanation

`Double`, corresponding to `double`, is the wrapper class most relevant to `BankAccount`'s own `balance` field, and shares the same shape of API as `Float`, scaled to `double`'s greater precision from Chunk 1.

| Member | What it does |
|---|---|
| `Double.parseDouble(String s)` | Parses text into a primitive `double` |
| `Double.MAX_VALUE` / `Double.MIN_VALUE` | The boundary values of `double` |
| `Double.isNaN(double d)` | Checks for "not a number" |
| `Double.isInfinite(double d)` | Checks for infinity |
| `Double.compare(double x, double y)` | Compares two `double` values, handling `NaN` and signed zero correctly, unlike a bare `<` or `>` |
| `doubleValue()` | Instance method, unboxes to a primitive `double` |

**Why `Double.compare` handles edge cases ordinary comparison operators do not.** `NaN` famously compares as neither greater than, less than, nor equal to anything, including itself, when using `<`, `>`, or `==` directly. `Double.compare` defines a total, consistent ordering that treats `NaN` as greater than any other value, including positive infinity, giving predictable results even for these edge cases.

```java
double nanValue = Double.NaN;
System.out.println(nanValue == nanValue);
System.out.println(Double.compare(nanValue, nanValue));
```

Output:
```
false
0
```

`nanValue == nanValue` is famously `false`, a direct callback to Chunk 1's floating point precision discussion, since `NaN` never equals anything under `==`, not even itself. `Double.compare(nanValue, nanValue)` instead reports `0`, treating two `NaN` values as equal to each other for ordering purposes, a deliberate, useful design choice.

**Applying `Double.parseDouble` to account balances.** This directly extends Section 4's `processDepositRequest`, and is the natural choice specifically because `balance` is declared `double` throughout the entire scenario.

### Lend A Hand: Quiz

1. What does `nanValue == nanValue` evaluate to, given `double nanValue = Double.NaN;`?
   A. `true`  B. `false`, since `NaN` never equals anything, including itself, under `==`  C. Compile error  D. `NaN`

2. What does `Double.compare(Double.NaN, Double.NaN)` return?
   A. A negative number  B. `0`, since `Double.compare` treats two `NaN` values as equal for ordering purposes  C. Throws an exception  D. `NaN`

3. Why is `Double.parseDouble` the appropriate choice for parsing a raw deposit amount string, rather than `Integer.parseInt`?
   A. `Integer.parseInt` is faster  B. `balance` is declared `double`, and deposit amounts may genuinely include fractional cents that `Integer.parseInt` could never represent  C. There is no real difference  D. `Double.parseDouble` does not exist

### Answers

1. **B.** This is the exact same `NaN` behavior covered in Chunk 1: it is specifically defined to never compare equal to anything under `==`, including another `NaN`, or even itself.

2. **B.** `Double.compare` provides a well defined total ordering that specifically treats `NaN` values as equal to each other, unlike the inconsistent behavior of `==` on `NaN`.

3. **B.** Since `balance` is `double` throughout `BankAccount`, and real deposit amounts commonly include cents, parsing with `Double.parseDouble` correctly preserves that fractional precision, while `Integer.parseInt` would either reject such input entirely or, if the input happened to have no decimal point, silently discard any intended fractional meaning.

### Programming Practice

1. Write a method that safely compares two account balances using `Double.compare`, returning a `String`, `"higher"`, `"lower"`, or `"equal"`, describing the first balance relative to the second.

### Solution

```java
public class Solution1 {
    static String compareBalances(double a, double b) {
        int result = Double.compare(a, b);
        if (result > 0) {
            return "higher";
        } else if (result < 0) {
            return "lower";
        } else {
            return "equal";
        }
    }

    public static void main(String[] args) {
        System.out.println(compareBalances(500.0, 300.0));
        System.out.println(compareBalances(300.0, 500.0));
        System.out.println(compareBalances(300.0, 300.0));
    }
}
```

Expected output:
```
higher
lower
equal
```

Why it works: `Double.compare` returns a value whose sign directly indicates the ordering between the two arguments, exactly like `Integer.compare` from Section 5, and the `if else if` ladder from Chunk 1 translates that sign into a readable description.

---

## 8. Byte, Short, Character and Boolean Wrapper Class with Lend A Hand

### Concept Explanation

The remaining four wrapper classes round out the full set, each with a smaller, more specialized API than `Integer` or `Double`.

**`Byte` and `Short`.** These mirror `Integer`'s pattern at a smaller scale: `Byte.parseByte(String s)` and `Short.parseShort(String s)` parse text into their respective primitives, and `Byte.MAX_VALUE`/`MIN_VALUE` and `Short.MAX_VALUE`/`MIN_VALUE` mirror the boundary constants from Chunk 1.

**`Character` offers a rich set of static classification methods**, most of which accept a `char` directly and are extremely commonly tested.

| Method | What it does |
|---|---|
| `Character.isDigit(char c)` | Whether `c` is a digit `0` through `9` |
| `Character.isLetter(char c)` | Whether `c` is an alphabetic letter |
| `Character.isLetterOrDigit(char c)` | Whether `c` is a letter or a digit |
| `Character.isUpperCase(char c)` / `isLowerCase(char c)` | Case checks |
| `Character.toUpperCase(char c)` / `toLowerCase(char c)` | Case conversion, returning a `char` |
| `Character.isWhitespace(char c)` | Whether `c` is a space, tab, newline, or similar |

**Validating an account holder's name character by character.**

```java
public static boolean isValidHolderName(String name) {
    if (name == null || name.isEmpty()) {
        return false;
    }
    for (int i = 0; i < name.length(); i++) {
        char c = name.charAt(i);
        if (!Character.isLetter(c) && !Character.isWhitespace(c)) {
            return false;
        }
    }
    return true;
}
```

This combines Chunk 1's traditional `for` loop, Chunk 5's `charAt`, and `Character`'s classification methods to reject any name containing digits or symbols, while still permitting spaces between a first and last name.

**`Boolean`.** `Boolean.parseBoolean(String s)` parses text into a primitive `boolean`, treating any case insensitive match of `"true"` as `true` and everything else, including malformed input, as `false`, notably never throwing an exception the way numeric parsing does. `Boolean.TRUE` and `Boolean.FALSE` are pre-built, cached constant `Boolean` objects, analogous in spirit to the `Integer` cache from Section 3, though covering the entire, tiny value space of `boolean` rather than just a range.

```java
System.out.println(Boolean.parseBoolean("TRUE"));
System.out.println(Boolean.parseBoolean("yes"));
```

Output:
```
true
false
```

`"yes"` is not recognized as `"true"`, so it parses to `false` rather than throwing any exception, a notable and sometimes surprising difference from the numeric `parse` methods covered earlier in this chunk.

### Lend A Hand: Quiz

1. What does `Character.isDigit('7')` return?
   A. `7`  B. `true`  C. `false`  D. Compile error

2. What does `Character.toUpperCase('a')` return?
   A. `"A"` as a `String`  B. `'A'` as a `char`  C. `65`  D. `false`

3. What does `Boolean.parseBoolean("maybe")` return?
   A. Throws an exception  B. `false`, since it is not a case insensitive match for `"true"`  C. `true`  D. Compile error

4. In `isValidHolderName`, why does the loop check both `Character.isLetter(c)` and `Character.isWhitespace(c)`?
   A. Redundant, only one check is needed  B. To permit letters and spaces, such as between a first and last name, while rejecting digits and symbols  C. `isWhitespace` is required syntax  D. To reject all names entirely

### Answers

1. **B.** `'7'` is a digit character, so `isDigit` correctly returns `true`.

2. **B.** `toUpperCase(char)` returns a `char`, `'A'`, not a `String` and not the numeric code point.

3. **B.** Unlike numeric parsing, `Boolean.parseBoolean` never throws an exception for unrecognized input; it simply treats anything other than a case insensitive `"true"` match as `false`.

4. **B.** A realistic holder name may legitimately contain spaces between words, so the check must accept both letters and whitespace, rejecting the name only if some other character, such as a digit or symbol, is found.

### Programming Practice

1. Write a method `countDigits(String s)` that returns how many characters in `s` are digits, using `Character.isDigit` inside a loop, and test it on a raw account number string mixed with some formatting characters.

### Solution

```java
public class Solution1 {
    static int countDigits(String s) {
        int count = 0;
        for (int i = 0; i < s.length(); i++) {
            if (Character.isDigit(s.charAt(i))) {
                count++;
            }
        }
        return count;
    }

    public static void main(String[] args) {
        System.out.println(countDigits("AC-10293-84"));
    }
}
```

Expected output: `7`

Why it works: the loop inspects every character in turn, and `Character.isDigit` correctly identifies only the seven actual digit characters, `1,0,2,9,3,8,4`, ignoring the letters and the hyphen entirely.

---

## 9. Where Wrappers Can Be Used

### Concept Explanation

This closing section ties together every reason wrapper classes exist and adds one final, genuinely tricky point: how Java resolves overloaded methods when both primitive and wrapper versions are available.

**Wherever `null` needs to be representable.** As established in Section 1, this is wrapper classes' most fundamental advantage: `PendingApplication.assignedAccountNumber` could never be expressed as a plain `int`.

**Wherever an object is specifically required.** Many parts of the Java standard library, especially collections such as `ArrayList`, which store references rather than primitive values directly, require wrapper types rather than primitives; autoboxing makes storing an `int` into such a structure feel seamless even though a genuine `Integer` object is what actually gets stored underneath.

**Overload resolution: primitive versus wrapper parameters.** Suppose `BankAccount` had two overloaded methods, connecting directly back to Chunk 2's overloading rules.

```java
public void logAmount(int amount) {
    System.out.println("int version: " + amount);
}

public void logAmount(Integer amount) {
    System.out.println("Integer version: " + amount);
}
```

```java
BankAccount acc = new BankAccount("Test", 0);
acc.logAmount(50);
```

Output: `int version: 50`

Given a plain `int` literal argument, Java strongly prefers an exact, already matching primitive overload over one that would require autoboxing to match instead. The general preference order the compiler follows is: an exact type match first, then a widening primitive conversion, such as `int` to `long`, from Chunk 1, and only after both of those options are exhausted does the compiler resort to boxing the argument to match a wrapper typed parameter. This is precisely why `logAmount(50)` calls the `int` version here, even though `50` could technically also have been autoboxed to match the `Integer` version.

### Important Notes

- Wrapper classes are needed wherever `null`, meaning "no value," must be a genuinely representable state, which no primitive can ever express.
- Wrapper classes provide the static utility methods, such as parsing and classification, that primitives, having no methods of their own, cannot offer.
- When overloaded methods offer both a primitive and a wrapper parameter option, Java prefers an exact match, then widening, and only resorts to boxing as a last resort among these three.
- Despite autoboxing's convenience, using a primitive directly remains both simpler and more efficient whenever `null` is never actually a meaningful possibility, exactly as `BankAccount.balance` itself has remained a plain `double` throughout this entire course.

### Quick Check

If `logAmount(int)` did not exist at all, only `logAmount(Integer)`, would `acc.logAmount(50)` still compile?

Yes. With no exact or widening primitive match available, the compiler falls back to autoboxing `50` into an `Integer` to satisfy the only available overload, exactly the boxing conversion covered throughout this chunk, and the call succeeds, printing `"Integer version: 50"` instead.

---

## 10. Consolidated Quiz: All Six Chunks Together

1. Given `Integer x = 127; Integer y = 127;` and `Integer p = 128; Integer q = 128;`, which comparisons using `==` are `true`?
   A. Both  B. Only `x == y`, since `127` is within the cached range while `128` is not  C. Only `p == q`  D. Neither

2. `BankAccount`'s `deposit(double amount)` from Chunk 2 validates `amount > 0`. If a raw `String` deposit amount is first parsed with `Double.parseDouble`, which exception, from this chunk, could that parsing step itself throw before `deposit` is ever even reached?
   A. `ArithmeticException`  B. `NumberFormatException`  C. `ClassCastException`  D. None, parsing never throws

3. `SavingsAccount extends BankAccount`, from Chunk 3. If `BankAccount` had an overloaded pair `logAmount(int)` and `logAmount(Integer)`, and `SavingsAccount` inherited both unchanged, which one would `sa.logAmount(75)` call?
   A. `logAmount(Integer)`, always  B. `logAmount(int)`, following the same exact-match-before-boxing preference covered in Section 9  C. Compile error, ambiguous  D. Both run

4. `Character.isLetter(c)` is used inside a loop validating a holder name. Which Chunk 1 looping construct is used in `isValidHolderName`?
   A. `while`  B. The traditional `for` loop  C. `do while`  D. Enhanced for each only

5. `Number` is mentioned in Section 2 as an abstract class. Which chunk introduced the general rule that an abstract class cannot be instantiated directly?
   A. Chunk 2  B. Chunk 3  C. Chunk 4  D. Chunk 5

6. If `BankAccount` overrode `equals()`, from Chunk 5, to compare by `accountNumber`, an `int` field, would boxing ever be involved in that comparison?
   A. Yes, always  B. Not necessarily; comparing two `int` primitive fields directly with `==` inside `equals()` involves no boxing at all, unlike comparing two `Integer` objects would  C. Only if `accountNumber` were `null`  D. Compile error

7. `StringTokenizer`, from Chunk 5, produces `String` tokens. Which method from this chunk would most directly convert one such token into a `double` deposit amount?
   A. `Integer.parseInt`  B. `Double.parseDouble`  C. `Boolean.parseBoolean`  D. `Character.isDigit`

8. Given `Auditable item = new BankAccount(...)`, from Chunk 4, and a hypothetical overloaded pair `process(BankAccount)` and `process(Auditable)`, would autoboxing preference rules from Section 9 apply to choosing between them?
   A. Yes, identically  B. No; that choice is resolved by ordinary reference type overload resolution from Chunk 2, not by the primitive-versus-wrapper boxing preference specific to this chunk  C. Compile error, always ambiguous  D. Only `process(Auditable)` is legal

### Consolidated Quiz Answers

1. **B.** `127` falls within the `-128` to `127` cache described in Section 3, so `x` and `y` share one pooled object; `128` falls just outside it, so `p` and `q` are two separately allocated objects, making `p == q` false despite identical values.

2. **B.** `Double.parseDouble` throws `NumberFormatException` at runtime for text that does not represent a valid number, exactly as covered in Section 4, entirely before `deposit` itself would ever be called with the resulting value.

3. **B.** Exactly as demonstrated in Section 9, and following ordinary method inheritance from Chunk 3, `SavingsAccount` inherits both overloads unchanged, and the compiler's exact-match-before-boxing preference still applies identically, selecting `logAmount(int)` for a plain `int` literal argument.

4. **B.** `isValidHolderName` in Section 8 uses `for (int i = 0; i < name.length(); i++)`, the traditional counting `for` loop from Chunk 1, to walk through each character by index.

5. **C.** Chunk 4 introduced abstract classes and established that they can never be instantiated directly with `new`, a rule that applies identically to `Number`, a real, built in example of that same concept.

6. **B.** Comparing two primitive `int` fields with `==` inside an `equals()` override, as `BankAccount.equals()` does from Chunk 5, is ordinary primitive comparison with no boxing involved whatsoever; boxing only enters the picture if the fields themselves were declared as `Integer` rather than `int`.

7. **B.** `Double.parseDouble` is specifically suited for converting text into a `double`, matching `balance`'s declared type throughout the entire scenario, exactly as Section 7 covered.

8. **B.** Choosing between `process(BankAccount)` and `process(Auditable)` is resolved through ordinary reference type and inheritance based overload resolution, the same rules from Chunk 2 and Chunk 3, since both parameter types are reference types; the primitive-versus-wrapper boxing preference from this chunk specifically concerns choosing between a primitive parameter and its own corresponding wrapper class, an unrelated kind of choice.

---

## 11. Programming Practice and Solutions

### Practice Problems

1. **Basic.** Write a method `parseAccountNumber(String raw)` that safely attempts `Integer.parseInt(raw)`, returning `null` as an `Integer` if parsing fails, rather than letting the exception propagate, using a `try catch` block around the parse call.
2. **Intermediate.** Write a method `sumValidAmounts(String[] rawAmounts)` that loops over an array of raw text amounts, safely parses each with `Double.parseDouble` inside a `try catch`, skips any that fail to parse, and returns the total of every value that succeeded.
3. **Intermediate.** Write a method `isStrongAccountPassword(String password)` that returns `true` only if the password is at least eight characters long and contains at least one digit and at least one letter, using `Character.isDigit` and `Character.isLetter` inside a loop.
4. **Advanced.** Extend Section 4's `processDepositRequest` into a full `importDeposits(BankAccount acc, String rawBatch)` method that uses `StringTokenizer`, from Chunk 5, to split a semicolon separated batch of raw amounts, attempts each with `Double.parseDouble` inside a `try catch`, deposits every valid one using `BankAccount.deposit`, and returns a `StringBuilder` report summarizing how many succeeded and how many were skipped.

### Solutions

**Solution 1.**

```java
public class Solution1 {
    static Integer parseAccountNumber(String raw) {
        try {
            return Integer.parseInt(raw);
        } catch (NumberFormatException e) {
            return null;
        }
    }

    public static void main(String[] args) {
        System.out.println(parseAccountNumber("1024"));
        System.out.println(parseAccountNumber("not-a-number"));
    }
}
```

Expected output:
```
1024
null
```

Why it works: the method's return type is `Integer`, not `int`, specifically so it can return `null` for invalid input, exactly the null representability advantage covered in Section 1; `Integer.parseInt`'s own primitive `int` result is automatically autoboxed into that `Integer` return type on success.

**Solution 2.**

```java
public class Solution2 {
    static double sumValidAmounts(String[] rawAmounts) {
        double total = 0;
        for (String raw : rawAmounts) {
            try {
                total += Double.parseDouble(raw);
            } catch (NumberFormatException e) {
                System.out.println("Skipping invalid amount: " + raw);
            }
        }
        return total;
    }

    public static void main(String[] args) {
        String[] amounts = {"100.50", "oops", "250", "75.25"};
        System.out.println("Total: " + sumValidAmounts(amounts));
    }
}
```

Expected output:
```
Skipping invalid amount: oops
Total: 425.75
```

Why it works: the enhanced for each loop from Chunk 1 visits every raw entry, and the `try catch` around each individual parse attempt ensures one malformed entry does not halt processing of the rest, correctly accumulating only the three genuinely valid amounts.

**Solution 3.**

```java
public class Solution3 {
    static boolean isStrongAccountPassword(String password) {
        if (password.length() < 8) {
            return false;
        }
        boolean hasDigit = false;
        boolean hasLetter = false;
        for (int i = 0; i < password.length(); i++) {
            char c = password.charAt(i);
            if (Character.isDigit(c)) {
                hasDigit = true;
            }
            if (Character.isLetter(c)) {
                hasLetter = true;
            }
        }
        return hasDigit && hasLetter;
    }

    public static void main(String[] args) {
        System.out.println(isStrongAccountPassword("abc123xy"));
        System.out.println(isStrongAccountPassword("short1"));
        System.out.println(isStrongAccountPassword("onlyletters"));
    }
}
```

Expected output:
```
true
false
false
```

Why it works: the length check acts as an early guard clause, exactly the pattern from Chunk 3's return value section, and the loop tracks two independent `boolean` flags, only reporting success once both a digit and a letter have genuinely been found somewhere in the password.

**Solution 4.**

```java
import java.util.StringTokenizer;

public class Solution4 {
    static StringBuilder importDeposits(BankAccount acc, String rawBatch) {
        StringTokenizer tokenizer = new StringTokenizer(rawBatch, ";");
        int successCount = 0;
        int skippedCount = 0;

        while (tokenizer.hasMoreTokens()) {
            String rawAmount = tokenizer.nextToken();
            try {
                double amount = Double.parseDouble(rawAmount);
                acc.deposit(amount);
                successCount++;
            } catch (NumberFormatException e) {
                skippedCount++;
            }
        }

        StringBuilder report = new StringBuilder();
        report.append("Deposits succeeded: ").append(successCount).append("\n");
        report.append("Deposits skipped: ").append(skippedCount).append("\n");
        report.append("Final balance: ").append(acc.getBalance());
        return report;
    }

    public static void main(String[] args) {
        BankAccount acc = new BankAccount("Nadia", 0);
        System.out.println(importDeposits(acc, "100;bad-entry;250.50;75"));
    }
}
```

Expected output:
```
Deposits succeeded: 3
Deposits skipped: 1
Final balance: 425.5
```

Why it works: this single method chains together `StringTokenizer` from Chunk 5, `Double.parseDouble` and exception handling from this chunk, `BankAccount.deposit` from Chunk 2, and `StringBuilder` report assembly from Chunk 5, correctly processing every well formed entry in the batch while gracefully skipping the one malformed entry rather than letting it interrupt the entire import.

---

## 12. Revision Summary

| Concept | Key Rule | Where It Appeared |
|---|---|---|
| Why wrappers exist | Represent `null`, provide static utility methods and constants primitives cannot have | `PendingApplication.assignedAccountNumber` |
| Primitive to wrapper mapping | Mostly capitalized names; `int` to `Integer` and `char` to `Character` are the irregular pair | The mapping table |
| Wrapper hierarchy | `Byte`, `Short`, `Integer`, `Long`, `Float`, `Double` all extend abstract `Number`; `Character` and `Boolean` extend `Object` directly | `Number.doubleValue()` inherited |
| Boxing and unboxing | Manual via `valueOf`/`xxxValue`; automatic via autoboxing and auto-unboxing | `Integer boxed = 500;` |
| Integer cache | `-128` to `127` are cached and shared; `==` is unreliable outside that range | `a == b` true at 100, false at 200 |
| Unboxing null | Throws `NullPointerException` at runtime | `int x = accountNumber;` when `null` |
| Number to String | `String.valueOf`, `Integer.toString`, or `+` concatenation | Formatting an account number |
| String to number | `parseXxx` returns a primitive; `valueOf` returns a boxed object; both throw `NumberFormatException` on bad input | Parsing a raw deposit amount |
| Double.compare | Gives a consistent ordering even for `NaN`, unlike `==` | `Double.compare(NaN, NaN)` returns `0` |
| Character utilities | `isDigit`, `isLetter`, `isWhitespace`, and case conversion, all `static` | Validating a holder name |
| Overload resolution | Exact match, then widening, then boxing, in that preference order | `logAmount(int)` chosen over `logAmount(Integer)` |

**Exam tip.** Any question comparing two wrapper objects with `==` should immediately prompt one check: is the value within the small cached range, `-128` to `127` for the integer wrapper types? If yes, `==` will misleadingly appear to work; if the question uses values outside that range specifically, it is testing whether you know `.equals()` is the only reliable comparison. Separately, any question mixing overloaded primitive and wrapper parameters is testing the exact-match-then-widening-then-boxing preference order from Section 9, almost always resolved in favor of the primitive overload whenever a plain literal argument is used.