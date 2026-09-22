# Java Competitive Exam Preparation
## Chunk 5: String, StringBuffer, StringBuilder, StringTokenizer, and equals/hashCode

This chunk shifts from designing classes to mastering one of the most heavily used, and most heavily exam tested, families of classes in all of Java: `String` and its mutable relatives. Every example continues to draw on the banking scenario built across Chunks 2 through 4, using it to generate account names, build reports, and parse input, so these APIs are seen doing real work rather than being memorized in isolation. The final section, `equals` and `hashCode`, connects directly back to the `Object` class discussion from Chunk 3.

---

## Table of Contents

1. Introduction to the String Class
2. Lend a Hand on String Constructors
3. String Class APIs with Lend a Hand
4. StringBuffer Class APIs with Lend a Hand
5. StringBuilder APIs with Lend a Hand
6. StringTokenizer APIs with Lend a Hand
7. equals and hashCode with Lend a Hand
8. Consolidated Quiz: All Five Chunks Together
9. Programming Practice and Solutions
10. Revision Summary

---

## 1. Introduction to the String Class

### Concept Explanation

`String` is a class in `java.lang`, not a primitive type, even though Java gives it unusually convenient, primitive like syntax: you can write `String name = "Tara";` directly with a literal, and use `+` to concatenate, neither of which ordinary classes get. Underneath that convenience, `accountHolder` in `BankAccount` has always genuinely been a reference to a `String` object, exactly like `BankAccount` itself is an object, following every rule about objects covered since Chunk 2.

**Strings are immutable.** Once a `String` object is created, its content can never be changed. Every method that appears to modify a `String`, such as `toUpperCase()` or `concat()`, actually creates and returns a brand new `String` object, leaving the original completely untouched.

```java
String holder = "priya";
holder.toUpperCase();
System.out.println(holder);
```

Output: `priya`

This is a famous and heavily tested trap: calling `toUpperCase()` did genuinely create a new `String` containing `"PRIYA"`, but since that new object was never assigned to anything, it is simply discarded, and `holder` still refers to the original, lowercase `String`. The correct usage always reassigns the result: `holder = holder.toUpperCase();`.

**The String pool.** Java maintains a special memory region called the string constant pool, or string pool, specifically for `String` literals. When you write a `String` literal, such as `"Tara"`, Java checks the pool first; if an identical literal already exists there, the existing object is reused rather than a new one being created. Writing `new String("Tara")`, in contrast, always creates a brand new object on the ordinary heap, bypassing the pool entirely, even if an identical literal already exists there.

```java
String a = "Tara";
String b = "Tara";
String c = new String("Tara");

System.out.println(a == b);
System.out.println(a == c);
System.out.println(a.equals(c));
```

Output:
```
true
false
true
```

`a` and `b` are both literals with identical content, so both point to the exact same pooled object, making `a == b` true, since `==` on reference types compares references, not content, exactly as noted in Chunk 1. `c` was built with `new`, forcing a separate object on the heap, so `a == c` is `false`, even though the content is identical. `.equals()`, which `String` overrides to compare actual content rather than references, correctly reports `true` for `a.equals(c)` regardless of which object each reference points to.

**`String` is `final`.** The `String` class itself is declared `final`, meaning, exactly as covered for methods in Chunk 3, it cannot be extended at all; there is no way to create a subclass of `String`. This is a deliberate design choice that protects the guarantees the string pool and immutability rely on.

### Important Notes

- `String` is a class, and every `String` reference follows the same pass by value reference rules from Chunk 3's object passing discussion.
- Every apparent modification to a `String` actually produces a new object; the original is never altered.
- `==` compares references; `.equals()` compares content. For `String`, always use `.equals()` unless you specifically intend to check reference identity.
- Literals are pooled and reused automatically; `new String(...)` deliberately bypasses the pool.
- `String` is `final` and cannot be subclassed.

### Quick Check

If `BankAccount`'s constructor stored `this.accountHolder = accountHolder;` directly from a passed in `String` parameter, could some other code later "sneak in" and change the account holder's name by modifying that same `String` object from outside?

No. Since `String` is immutable, there is no method on a `String` object that can alter its own content at all; the only way `accountHolder` could ever appear to change is if `BankAccount`'s own code reassigned the field to point to an entirely new `String` object, such as through a setter like Chunk 2's `setAccountHolder`. This immutability is actually a quiet but genuine safety benefit of using `String` for sensitive fields.

---

## 2. Lend a Hand on String Constructors

### Concept Explanation

Beyond writing a literal, `String` offers several constructors for building a `String` object from other sources of character data.

| Constructor | What it does |
|---|---|
| `new String()` | Creates an empty `String`, equivalent to `""` |
| `new String(String original)` | Creates a new, separate object with the same content as `original`, bypassing the pool |
| `new String(char[] value)` | Builds a `String` from every character in the given array |
| `new String(char[] value, int offset, int count)` | Builds a `String` from `count` characters starting at index `offset` in the array |
| `new String(byte[] bytes)` | Builds a `String` by decoding the given bytes using the platform's default character encoding |

**Building an account holder's name from a `char[]`.**

```java
char[] letters = {'K', 'a', 'v', 'y', 'a'};
String holder = new String(letters);
System.out.println(holder);
```

Output: `Kavya`

**Using the offset and count constructor.**

```java
char[] fullText = {'M', 'r', '.', ' ', 'A', 'r', 'j', 'u', 'n'};
String firstName = new String(fullText, 4, 5);
System.out.println(firstName);
```

Output: `Arjun`

The constructor starts reading at index `4`, which is `'A'`, and reads exactly `5` characters, producing `"Arjun"`, skipping the `"Mr. "` prefix entirely.

### Lend a Hand: Quiz

1. What does `new String()` produce?
   A. `null`  B. An empty `String`, `""`  C. A compile error  D. A `String` containing `"null"`

2. Given `char[] c = {'H','i'}; String s = new String(c, 0, 2);`, what is `s`?
   A. `"Hi"`  B. `"H"`  C. Compile error  D. `null`

3. Does `new String("Test")` place its result in the string pool?
   A. Yes, always  B. No, `new` always creates a separate heap object, bypassing the pool  C. Only if `"Test"` was never used before  D. Only for short strings

### Answers

1. **B.** `new String()` is a valid, if rarely used, constructor that produces an empty `String` containing zero characters, distinct from `null`, which would mean no `String` object exists at all.

2. **A.** Starting at index `0` and reading `2` characters from `{'H','i'}` produces exactly `"Hi"`.

3. **B.** As established in Section 1, using `new` to construct a `String` always allocates a fresh object on the heap, regardless of whether an identical literal already exists in the pool; only literals written directly in source code are automatically pooled.

### Programming Practice

1. Write a program that builds a `BankAccount` holder's name from a `char[]` array representing initials joined with a full name, using the offset and count constructor to extract just the last name portion for a separate greeting message.

### Solution

```java
public class Solution1 {
    public static void main(String[] args) {
        char[] fullName = {'R', 'a', 'v', 'i', ' ', 'K', 'u', 'm', 'a', 'r'};
        String lastName = new String(fullName, 5, 5);
        BankAccount acc = new BankAccount(new String(fullName), 500);
        System.out.println("Welcome, Mr./Ms. " + lastName + "!");
        System.out.println(acc.getAccountHolder());
    }
}
```

Why it works: in `"Ravi Kumar"`, index `4` is the space and index `5` is `'K'`, so `new String(fullName, 5, 5)` starts right at `'K'` and reads `5` characters, correctly isolating `"Kumar"`, while `new String(fullName)` with no offset builds the complete name for the account itself.

---

## 3. String Class APIs with Lend a Hand

### Concept Explanation

`String` provides a large set of methods for inspecting and deriving new strings from existing ones. Since every one of these returns a new `String` rather than modifying the original, the immutability trap from Section 1 applies to every single method in this table: forgetting to capture the return value means nothing actually happens to any variable you can see.

| Method | What it does | Example |
|---|---|---|
| `length()` | Returns the number of characters | `"Priya".length()` → `5` |
| `charAt(int index)` | Returns the character at a position | `"Priya".charAt(0)` → `'P'` |
| `substring(int begin)` | Returns everything from `begin` to the end | `"Priya".substring(2)` → `"iya"` |
| `substring(int begin, int end)` | Returns characters from `begin` up to, but not including, `end` | `"Priya".substring(1, 3)` → `"ri"` |
| `indexOf(String s)` | Returns the first index where `s` occurs, or `-1` | `"Priya".indexOf("iy")` → `2` |
| `contains(CharSequence s)` | Returns whether `s` occurs anywhere | `"Priya".contains("ri")` → `true` |
| `equals(Object o)` | Compares content for exact equality | `"Priya".equals("priya")` → `false` |
| `equalsIgnoreCase(String s)` | Compares content ignoring case | `"Priya".equalsIgnoreCase("priya")` → `true` |
| `compareTo(String s)` | Lexicographic comparison, negative, zero, or positive | `"apple".compareTo("banana")` → negative |
| `toUpperCase()` / `toLowerCase()` | Returns a case converted copy | `"Priya".toUpperCase()` → `"PRIYA"` |
| `trim()` | Returns a copy with leading and trailing whitespace removed | `"  hi  ".trim()` → `"hi"` |
| `replace(char old, char new)` | Returns a copy with every occurrence replaced | `"Priya".replace('i','I')` → `"PrIya"` |
| `split(String regex)` | Splits into an array of substrings around matches | `"a,b,c".split(",")` → `{"a","b","c"}` |
| `concat(String s)` | Returns the concatenation, equivalent to `+` | `"Pri".concat("ya")` → `"Priya"` |
| `isEmpty()` | Returns whether length is `0` | `"".isEmpty()` → `true` |
| `startsWith(String s)` / `endsWith(String s)` | Checks a prefix or suffix | `"Priya".startsWith("Pri")` → `true` |
| `toCharArray()` | Returns the characters as a `char[]` | `"Hi".toCharArray()` → `{'H','i'}` |

**Applying these to `getAuditLog()`.** Chunk 4's `BankAccount.getAuditLog()` returned a raw concatenated `String`. Using a few of these methods, it can be made more presentable:

```java
public String getFormattedHolderName() {
    String trimmed = accountHolder.trim();
    if (trimmed.isEmpty()) {
        return "Unknown";
    }
    return trimmed.substring(0, 1).toUpperCase() + trimmed.substring(1).toLowerCase();
}
```

Given `accountHolder` is `"  priya SHARMA  "`, this first trims whitespace, checks it is not blank, guarding against the same kind of empty input issue `deposit` guarded against back in Chunk 2, then capitalizes only the first character while lowercasing the rest, producing `"Priya sharma"`.

**A common off by one trap with `substring`.** `substring(begin, end)`'s `end` index is exclusive, not inclusive. `"Priya".substring(1, 3)` returns the characters at indices `1` and `2`, `"ri"`, not including index `3`. Forgetting this exclusivity is one of the most common sources of off by one bugs with `String` on a competitive exam.

### Lend a Hand: Quiz

1. What does `"Banking".substring(3)` return?
   A. `"Ban"`  B. `"king"`  C. `"kin"`  D. `"Bank"`

2. What does `"Banking".substring(0, 4)` return?
   A. `"Bank"`  B. `"Banki"`  C. `"anki"`  D. `"Bankin"`

3. Given `String s = "test"; s.toUpperCase();`, what does `s` equal afterward?
   A. `"TEST"`  B. `"test"`, unchanged, since the returned value was never captured  C. Compile error  D. `null`

4. What does `"Hello".equals("HELLO")` return?
   A. `true`  B. `false`  C. Compile error  D. `1`

5. What does `"apple,banana,cherry".split(",")` produce?
   A. A single `String` `"apple,banana,cherry"`  B. An array `{"apple", "banana", "cherry"}`  C. `null`  D. Compile error

### Answers

1. **B, `"king"`.** `substring(3)` returns everything from index `3` to the end; index `3` in `"Banking"` is `'k'`, so the result is `"king"`.

2. **A, `"Bank"`.** The `end` parameter is exclusive, so characters at indices `0` through `3` are included, giving `"Bank"`, not extending into index `4`.

3. **B.** Exactly as covered in Section 1, `toUpperCase()` returns a new `String` without modifying `s` itself; since the result is never assigned back to `s`, `s` remains `"test"`.

4. **B, `false`.** `.equals()` performs an exact, case sensitive content comparison; `"Hello"` and `"HELLO"` differ in case, so this is `false`. `equalsIgnoreCase` would be needed for a case insensitive comparison.

5. **B.** `split(",")` divides the string at every comma, returning an array of the resulting pieces.

### Programming Practice

1. Write a method `maskAccountNumber(String accountNumber)` that returns the account number with every digit except the last four replaced by `*`, using `substring` and `length()`, and test it on a sample account number string.

### Solution

```java
public class Solution1 {
    static String maskAccountNumber(String accountNumber) {
        int len = accountNumber.length();
        if (len <= 4) {
            return accountNumber;
        }
        String visible = accountNumber.substring(len - 4);
        StringBuilder masked = new StringBuilder();
        for (int i = 0; i < len - 4; i++) {
            masked.append("*");
        }
        return masked + visible;
    }

    public static void main(String[] args) {
        System.out.println(maskAccountNumber("1029384756"));
    }
}
```

Expected output: `******4756`

Why it works: `substring(len - 4)` isolates exactly the last four characters regardless of the account number's total length, and a loop builds the masking prefix, previewing `StringBuilder`, covered fully in Section 5, as a clean way to build a `String` piece by piece rather than repeatedly concatenating with `+` in a loop.

---

## 4. StringBuffer Class APIs with Lend a Hand

### Concept Explanation

`StringBuffer` is a class specifically designed for building and modifying character sequences efficiently, in direct contrast to `String`'s immutability. Unlike `String`, a `StringBuffer` object's content genuinely changes in place when you call its modifying methods; no new object is created for each change.

| Method | What it does |
|---|---|
| `append(...)` | Adds content to the end, accepts nearly any type |
| `insert(int offset, ...)` | Inserts content at a specific position |
| `delete(int start, int end)` | Removes characters from `start` up to, exclusive, `end` |
| `deleteCharAt(int index)` | Removes a single character |
| `reverse()` | Reverses the entire sequence in place |
| `replace(int start, int end, String s)` | Replaces a range with new content |
| `length()` | Returns the current number of characters |
| `toString()` | Converts the accumulated content into an ordinary `String` |

**Why this matters: building a `Branch` report efficiently.** Chunk 4's `Branch` class could build a multi line report by repeatedly concatenating `String` with `+`, but each `+` on immutable `String`s actually discards an intermediate object and allocates a brand new one, which becomes wasteful across many iterations of a loop. `StringBuffer` avoids this entirely.

```java
public String buildFullReport() {
    StringBuffer report = new StringBuffer();
    report.append("Branch: ").append(branchName).append("\n");
    for (BankAccount acc : accounts) {
        report.append(acc.getAccountHolder()).append(": ").append(acc.getBalance()).append("\n");
    }
    return report.toString();
}
```

Each `append` call modifies the same underlying `StringBuffer` object in place, and chaining calls together, `.append(...).append(...)`, works because `append` returns a reference to the same `StringBuffer` it was called on, precisely the same `return this;` chaining pattern from Chunk 3's returning objects section.

**`StringBuffer` is thread safe.** Its methods are `synchronized`, meaning they are safe to call from multiple threads simultaneously without corrupting the internal character data, at some performance cost compared to an unsynchronized alternative, which is exactly what motivates Section 5's `StringBuilder`.

### Lend a Hand: Quiz

1. What is the defining difference between `String` and `StringBuffer`?
   A. `StringBuffer` cannot hold letters  B. `StringBuffer` is mutable; its content can genuinely change in place, unlike immutable `String`  C. They are identical  D. `StringBuffer` is a primitive

2. Given `StringBuffer sb = new StringBuffer("Hi"); sb.append("!");`, what does `sb.toString()` equal?
   A. `"Hi"`  B. `"Hi!"`  C. `"!Hi"`  D. Compile error

3. Why does `append` returning a reference to the same `StringBuffer` object matter?
   A. It does not matter  B. It enables method chaining, calling another method directly on the result, exactly like `return this;` from Chunk 3  C. It creates a new object each time  D. It only works with `String`

4. Is `StringBuffer` thread safe?
   A. No, never  B. Yes, its methods are synchronized  C. Only for `append`  D. Only in single threaded programs

### Answers

1. **B.** `String` objects never change after creation; `StringBuffer` objects are specifically designed to be modified in place, which is the entire reason it exists.

2. **B.** `append("!")` adds directly onto the existing content, in place, producing `"Hi!"`, unlike `String.concat`, which would have produced a separate new object instead.

3. **B.** Since `append` returns `this`, the same object it was called on, another method can be called directly on that returned reference, letting several operations chain together in one expression, exactly the technique Chunk 3 used for `deposit(...).deposit(...)`.

4. **B.** `StringBuffer`'s methods are `synchronized`, making it safe for concurrent use by multiple threads, at some performance cost compared to an equivalent unsynchronized class.

### Programming Practice

1. Write a method that takes a `BankAccount[]` array and returns a single `StringBuffer` containing every account holder's name, each on its own line, built entirely through chained `append` calls inside a loop.

### Solution

```java
public class Solution1 {
    static StringBuffer buildNameList(BankAccount[] accounts) {
        StringBuffer sb = new StringBuffer();
        for (BankAccount acc : accounts) {
            sb.append(acc.getAccountHolder()).append("\n");
        }
        return sb;
    }

    public static void main(String[] args) {
        BankAccount[] accounts = {
            new BankAccount("Mateo", 100),
            new BankAccount("Sana", 200)
        };
        System.out.print(buildNameList(accounts));
    }
}
```

Why it works: each loop iteration appends directly onto the same `StringBuffer` object in place, so no intermediate objects are discarded along the way, unlike repeated `String` concatenation would have produced.

---

## 5. StringBuilder APIs with Lend a Hand

### Concept Explanation

`StringBuilder` offers exactly the same method set as `StringBuffer`, `append`, `insert`, `delete`, `deleteCharAt`, `reverse`, `replace`, and more, but its methods are not `synchronized`. This makes `StringBuilder` faster than `StringBuffer` in ordinary, single threaded code, which describes the overwhelming majority of real world usage, since there is no synchronization overhead to pay for when no other thread could ever touch the object concurrently.

```java
StringBuilder sb = new StringBuilder();
sb.append("Account #").append(1024).append(": ").append("Active");
System.out.println(sb.toString());
```

Output: `Account #1024: Active`

Notice `append` happily accepts an `int` directly, `1024`, automatically converting it to its string representation, exactly as `String` concatenation with `+` does, without requiring any manual conversion.

**Comparison: `String` versus `StringBuffer` versus `StringBuilder`.**

| | `String` | `StringBuffer` | `StringBuilder` |
|---|---|---|---|
| Mutable | No | Yes | Yes |
| Thread safe | Not applicable, immutable objects are inherently safe to share | Yes, synchronized | No |
| Performance for heavy modification | Poor, each change allocates a new object | Good, but with synchronization overhead | Best, no synchronization overhead |
| Typical use | Fixed or rarely changing text | Building text shared safely across threads | Building text within a single thread or method |

**When to choose which.** For a `String` that will not be modified after creation, such as `BankAccount`'s `accountHolder`, plain `String` remains the right choice. For assembling text through many small pieces, such as building a report inside a single method's loop, `StringBuilder` is almost always the right default, since multithreaded access to the same builder object is rare. `StringBuffer` is reserved specifically for the less common case where the same buffer genuinely will be modified from multiple threads.

### Lend a Hand: Quiz

1. What is the main practical difference between `StringBuilder` and `StringBuffer`?
   A. `StringBuilder` cannot use `append`  B. `StringBuilder`'s methods are not synchronized, making it faster in single threaded code  C. `StringBuffer` is immutable  D. There is no difference

2. Given `StringBuilder sb = new StringBuilder("Test");`, does `sb.append(5)` compile?
   A. No, `append` only accepts `String`  B. Yes, `append` is overloaded to accept many types including `int`, automatically converting them  C. Only with an explicit cast to `String`  D. Compile error

3. For building a report string inside one single threaded method's loop, which class is generally the best default choice?
   A. `String`, using `+=` in the loop  B. `StringBuilder`  C. `StringBuffer`  D. `char[]`

### Answers

1. **B.** Both classes offer an identical method set, but `StringBuffer`'s methods carry `synchronized` thread safety guarantees that `StringBuilder`'s do not, which is exactly what makes `StringBuilder` faster whenever that safety is not actually needed.

2. **B.** `append` is overloaded, connecting back to Chunk 2's overloading discussion, with versions accepting `int`, `double`, `char`, `boolean`, `Object`, and more, each converting its argument to text automatically.

3. **B.** Since a single method's local loop has no risk of concurrent access from another thread, `StringBuilder` provides the exact same functionality as `StringBuffer` with better performance, making it the standard default choice; `+=` on `String` inside a loop would repeatedly allocate and discard objects, exactly the inefficiency Section 4 introduced `StringBuffer` to avoid.

### Programming Practice

1. Rewrite Section 4's `buildFullReport()` method using `StringBuilder` instead of `StringBuffer`, and add a line at the end that reverses a separator line using `reverse()` purely to demonstrate the method exists.

### Solution

```java
public String buildFullReport() {
    StringBuilder report = new StringBuilder();
    report.append("Branch: ").append(branchName).append("\n");
    for (BankAccount acc : accounts) {
        report.append(acc.getAccountHolder()).append(": ").append(acc.getBalance()).append("\n");
    }
    StringBuilder separator = new StringBuilder("===>");
    report.append(separator.reverse());
    return report.toString();
}
```

Why it works: swapping `StringBuffer` for `StringBuilder` requires no other code changes at all, since both classes share an identical method signature set; only the underlying thread safety guarantee differs, confirming the two are interchangeable for this single threaded use case. `separator.reverse()` demonstrates `reverse()` modifying its own object in place and returning that same reference, appended directly onto the report.

---

## 6. StringTokenizer APIs with Lend a Hand

### Concept Explanation

`java.util.StringTokenizer` is an older utility class for breaking a `String` into pieces, called tokens, based on delimiter characters. It predates `String.split()` and is considered legacy, but it still appears on exams and in older codebases, so it is worth knowing its specific API.

| Constructor or Method | What it does |
|---|---|
| `new StringTokenizer(String str)` | Splits on default delimiters: space, tab, newline, carriage return, form feed |
| `new StringTokenizer(String str, String delim)` | Splits on every character in `delim` |
| `hasMoreTokens()` | Returns whether any tokens remain |
| `nextToken()` | Returns the next token, advancing past it |
| `countTokens()` | Returns how many tokens remain, without consuming any |

**Parsing a batch of account holder names.**

```java
import java.util.StringTokenizer;

public class Demo {
    public static void main(String[] args) {
        String rawNames = "Tara,Ishaan,Devika,Kavya";
        StringTokenizer tokenizer = new StringTokenizer(rawNames, ",");

        while (tokenizer.hasMoreTokens()) {
            String name = tokenizer.nextToken();
            BankAccount acc = new BankAccount(name, 0);
            System.out.println("Created account for " + acc.getAccountHolder());
        }
    }
}
```

Output:
```
Created account for Tara
Created account for Ishaan
Created account for Devika
Created account for Kavya
```

The `while` loop, from Chunk 1, correctly stops the moment `hasMoreTokens()` reports `false`, having consumed every comma separated name in turn via `nextToken()`.

**`StringTokenizer` versus `String.split()`.** `split()`, using a regular expression, is generally the more modern and more flexible choice, supporting complex patterns `StringTokenizer` cannot express. `StringTokenizer` remains lighter weight and slightly faster for genuinely simple, fixed character delimiter splitting, and is still occasionally preferred, or simply already present, in legacy code, which is exactly why exams continue to test it directly.

### Lend a Hand: Quiz

1. What does `new StringTokenizer("a b  c").countTokens()` return, using default delimiters?
   A. `2`  B. `3`  C. `4`  D. `5`

2. Given `StringTokenizer t = new StringTokenizer("1-2-3", "-");`, what does `t.nextToken()` return on its first call?
   A. `"1-2-3"`  B. `"1"`  C. `"-"`  D. `"2"`

3. What happens if `nextToken()` is called when `hasMoreTokens()` would return `false`?
   A. Returns `null`  B. Throws a runtime exception, `NoSuchElementException`  C. Returns an empty `String`  D. Compile error

### Answers

1. **B, `3`.** Default delimiters include any run of whitespace, so `"a b  c"` splits into exactly three tokens, `"a"`, `"b"`, and `"c"`, with the double space between `"b"` and `"c"` still only separating two tokens, not producing an empty token in between.

2. **B, `"1"`.** The first token before the first `-` delimiter is `"1"`.

3. **B.** Calling `nextToken()` with no tokens remaining throws `NoSuchElementException` at runtime, which is exactly why checking `hasMoreTokens()` first, as the `while` loop example does, is essential.

### Programming Practice

1. Write a program that parses a raw input string `"Wren:500 Om:2000 Priyanka:300"` using two levels of `StringTokenizer`, an outer one splitting on spaces to get each `"name:balance"` pair, and an inner one splitting each pair on the colon, then creates and prints a `BankAccount` for each.

### Solution

```java
import java.util.StringTokenizer;

public class Solution1 {
    public static void main(String[] args) {
        String raw = "Wren:500 Om:2000 Priyanka:300";
        StringTokenizer outer = new StringTokenizer(raw, " ");

        while (outer.hasMoreTokens()) {
            String pair = outer.nextToken();
            StringTokenizer inner = new StringTokenizer(pair, ":");
            String name = inner.nextToken();
            double balance = Double.parseDouble(inner.nextToken());
            BankAccount acc = new BankAccount(name, balance);
            System.out.println(acc.getAccountHolder() + " opened with " + acc.getBalance());
        }
    }
}
```

Why it works: the outer tokenizer isolates each space separated `"name:balance"` chunk, and a fresh inner tokenizer is created for each chunk to further split it on the colon, demonstrating that `StringTokenizer` objects are independent and can be freely nested for multi level parsing.

---

## 7. equals and hashCode with Lend a Hand

### Concept Explanation

Chunk 3 established that every class implicitly extends `Object`, which provides `equals(Object other)` and, less discussed there, `hashCode()`. This section covers both fully, since together they form one of the most important and most commonly misapplied contracts in Java.

**`Object`'s default `equals()` compares references, identical to `==`.** Unless overridden, `equals()` returns `true` only if both references point to the exact same object in memory, exactly what `==` already does for objects.

```java
BankAccount a = new BankAccount("Test", 100);
BankAccount b = new BankAccount("Test", 100);
System.out.println(a.equals(b));
```

Output: `false`

Even though `a` and `b` hold identical data, `BankAccount` has never overridden `equals()`, so the inherited default from `Object` applies, comparing references, and since `a` and `b` are two genuinely separate objects, the result is `false`, exactly as `a == b` would also report.

**`Object`'s default `hashCode()` is derived from the object's identity**, typically related to its memory address, and is not required to have any relationship to an object's field values unless `hashCode()` is overridden.

**`String` overrides both.** `String`'s `equals()` compares actual character content, and its `hashCode()` is calculated from that same content, which is exactly why `a.equals(c)` returned `true` in Section 1's pooling example, despite `a` and `c` being different objects.

**Overriding `equals()` and `hashCode()` for `BankAccount`.** Suppose two `BankAccount` objects should be considered equal whenever they share the same `accountNumber`, regardless of any other field. This requires overriding both methods together, and it must be done consistently.

```java
@Override
public boolean equals(Object other) {
    if (this == other) {
        return true;
    }
    if (!(other instanceof BankAccount)) {
        return false;
    }
    BankAccount that = (BankAccount) other;
    return this.accountNumber == that.accountNumber;
}

@Override
public int hashCode() {
    return Integer.hashCode(accountNumber);
}
```

`this == other` is a quick shortcut: an object is always equal to itself. `instanceof`, from Chunk 1's relational operators, safely checks the other object's actual type before casting; without this check, casting an unrelated type to `BankAccount` would throw a `ClassCastException`. The actual comparison then reduces to a single field, `accountNumber`.

**The equals-hashCode contract.** Java requires that if two objects are equal according to `equals()`, they must return the identical value from `hashCode()`. The reverse is not required: two unequal objects are permitted to share the same hash code, called a collision, though good implementations try to minimize how often that happens. Breaking this contract, by overriding `equals()` without correspondingly updating `hashCode()`, produces subtle and confusing bugs anywhere hash based behavior is relied upon, which is exactly why the two methods must always be overridden together, never just one alone.

```java
BankAccount a = new BankAccount("Priya", 500); // suppose accountNumber ends up 1001
BankAccount b = new BankAccount("Priyanka", 900); // suppose accountNumber also ends up 1001
System.out.println(a.equals(b));
System.out.println(a.hashCode() == b.hashCode());
```

If both objects happen to share `accountNumber` `1001`, `a.equals(b)` now correctly reports `true`, following the overridden logic based purely on that one field, and `a.hashCode() == b.hashCode()` also reports `true`, since both derive their hash purely from the same shared `accountNumber`, correctly satisfying the contract.

### Important Notes

- Default `equals()`, inherited from `Object`, compares references; default `hashCode()` is based on object identity.
- Overriding `equals()` without also overriding `hashCode()` to remain consistent with it violates Java's required contract and is a common, serious bug.
- Always check `instanceof` before casting inside a custom `equals()` implementation, to avoid a `ClassCastException` on an unrelated type.
- `this == other` as a first check inside `equals()` is a common, harmless optimization, since an object is always trivially equal to itself.
- `String` already overrides both methods based on content, which is exactly why `String`s are compared with `.equals()` throughout this entire course rather than `==`.

### Quick Check

If `BankAccount` overrides `equals()` to compare by `accountNumber` but does NOT override `hashCode()` at all, leaving `Object`'s default in place, has the equals-hashCode contract been violated?

Yes, potentially. Two `BankAccount` objects with the same `accountNumber` would now be `equals()` to each other, per the override, but would almost certainly still report different `hashCode()` values, since the inherited default is based on object identity, not on `accountNumber`. This directly violates the requirement that equal objects must share a hash code, and is exactly the mistake the paired override above avoids by deriving `hashCode()` from the same field `equals()` uses.

### Lend a Hand: Quiz

1. What does `Object`'s default, unoverridden `equals()` actually compare?
   A. Field values  B. References, identical to `==`  C. Class names only  D. `hashCode()` values

2. Why does the custom `BankAccount.equals()` check `instanceof` before casting?
   A. It is not necessary  B. To safely confirm `other` is actually a `BankAccount` before attempting the cast, avoiding a `ClassCastException`  C. `instanceof` is required syntax for all methods  D. To improve performance only

3. If two objects have the same `hashCode()`, must they be `equals()` to each other?
   A. Yes, always  B. No, a shared hash code is permitted between unequal objects, called a collision; only the reverse direction is guaranteed  C. Only for `String`  D. Compile error otherwise

4. What specifically goes wrong if `equals()` is overridden but `hashCode()` is left as the inherited default?
   A. Nothing, they are unrelated  B. The equals-hashCode contract is violated, since equal objects, per the new `equals()`, may no longer share a hash code  C. `equals()` stops compiling  D. `hashCode()` automatically updates itself

### Answers

1. **B.** `Object`'s own `equals()` implementation is defined purely in terms of reference identity, functionally identical to `==`, unless a subclass explicitly overrides it.

2. **B.** Without the `instanceof` check, casting `other` directly to `BankAccount` would throw a `ClassCastException` at runtime the moment `other` turned out to be some unrelated type, rather than safely returning `false` as a proper `equals()` implementation should.

3. **B.** The equals-hashCode contract only requires that equal objects share a hash code; it explicitly permits unequal objects to coincidentally share one too, a collision, which any hash code implementation with a finite range of possible values must tolerate at least occasionally.

4. **B.** This is precisely the contract violation covered above: objects the new `equals()` considers equal may report different hash codes under the unmodified default `hashCode()`, since that default has no awareness of the fields `equals()` now actually compares.

### Programming Practice

1. Override `equals()` and `hashCode()` for `Transaction`, from Chunk 4, considering two transactions equal if they have the same `amount` and the same `description`, then demonstrate with two separately constructed but matching transactions that both `.equals()` and matching `hashCode()` values hold true.

### Solution

```java
// inside Transaction
@Override
public boolean equals(Object other) {
    if (this == other) {
        return true;
    }
    if (!(other instanceof Transaction)) {
        return false;
    }
    Transaction that = (Transaction) other;
    return this.getAmount() == that.getAmount() && this.description.equals(that.description);
}

@Override
public int hashCode() {
    return Double.hashCode(getAmount()) * 31 + description.hashCode();
}
```

```java
public class Demo {
    public static void main(String[] args) {
        Transaction t1 = new DepositTransaction(200);
        Transaction t2 = new DepositTransaction(200);
        System.out.println(t1.equals(t2));
        System.out.println(t1.hashCode() == t2.hashCode());
    }
}
```

Expected output:
```
true
true
```

Why it works: both `t1` and `t2` share the same `amount`, `200`, and the same `description`, `"Deposit"`, inherited from how `DepositTransaction`'s constructor calls `super(amount, "Deposit")`, so the overridden `equals()` correctly reports `true`, and since `hashCode()` is derived from exactly those same two fields, it produces an identical value for both objects too, correctly satisfying the contract. Multiplying by a small prime, `31`, before adding the second field's hash is a common, standard technique for combining multiple fields into one well distributed hash code.

---

## 8. Consolidated Quiz: All Five Chunks Together

1. Given `String s1 = "Bank"; String s2 = "Bank"; String s3 = new String("Bank");`, which comparisons are `true`? Consider `s1 == s2`, `s1 == s3`, and `s1.equals(s3)`.
   A. All three  B. Only `s1 == s2` and `s1.equals(s3)`  C. Only `s1.equals(s3)`  D. None

2. `BankAccount.deposit(double amount)` from Chunk 2 validates `amount > 0` before modifying `balance`. Which Chunk 1 concept does that validation directly rely on?
   A. The relational operator `>`  B. The bitwise operator `&`  C. The shift operator `<<`  D. String concatenation

3. `Transaction`, from Chunk 4, is `abstract`. Could you write `Transaction t = new Transaction(50, "Generic");` directly?
   A. Yes  B. No, abstract classes cannot be instantiated  C. Only inside `main`  D. Only with `new String(...)`

4. If `SavingsAccount` inherits `BankAccount`'s overridden `equals()`, comparing by `accountNumber`, and never overrides it further itself, what determines whether two `SavingsAccount` objects are `.equals()`?
   A. `SavingsAccount` cannot use `equals()` at all  B. The inherited `BankAccount` logic still applies, comparing `accountNumber`, exactly as covered for inherited methods in Chunk 3  C. `SavingsAccount` always uses `Object`'s default instead  D. Compile error

5. Using `StringBuilder` inside a `for` loop to build a report, as in Section 5, relies on which Chunk 1 looping construct?
   A. `do while`  B. The traditional or enhanced `for` loop  C. `switch`  D. `if else if`

6. `BankAccount implements Auditable`, from Chunk 4. If `Auditable`'s `getAuditLog()` internally called `accountHolder.trim()`, would this compile even though `accountHolder` is `private`?
   A. No, `private` blocks it  B. Yes, since `getAuditLog()` is implemented as a method inside `BankAccount` itself, it has full access to `BankAccount`'s own private fields, exactly as any instance method does  C. Only with `this.accountHolder`  D. Only if `Auditable` is also a class

7. Why is it significant that `String`'s own `equals()` compares content while `Object`'s default compares references?
   A. It is not significant  B. It demonstrates overriding in action: `String` provides its own version of a method inherited from `Object`, exactly the mechanism covered in Chunk 3  C. `String` does not actually override anything  D. `Object` has no `equals()` method

8. A `StringTokenizer` parsing `"100,200,300"` with `,` as the delimiter is used inside a `while (hasMoreTokens())` loop to create three `DepositTransaction` objects, stored in a `Transaction[]`. Which chunk's polymorphism concept applies when each is later passed to `execute(account)`?
   A. Chunk 1  B. Chunk 2  C. Chunk 3's runtime polymorphism, extended in Chunk 4 to abstract classes  D. None, this is unrelated

### Consolidated Quiz Answers

1. **B.** `s1` and `s2` are both literals sharing the same pooled object, so `s1 == s2` is `true`. `s3` was built with `new`, forcing a separate heap object, so `s1 == s3` is `false`. `.equals()` compares content, which is identical for all three, so `s1.equals(s3)` is `true`.

2. **A.** The condition `amount > 0` is a direct use of the relational operator `>` from Chunk 1, producing the `boolean` that the `if` statement then branches on.

3. **B.** `Transaction` is declared `abstract`, and as covered in Chunk 4, an abstract class can never be instantiated directly with `new`, regardless of whether it has a constructor available for subclasses to call via `super(...)`.

4. **B.** Exactly as covered for ordinary inherited methods in Chunk 3, a subclass that does not override an inherited method simply uses the superclass's version unchanged; `SavingsAccount` inherits `BankAccount`'s `equals()` logic in full, still comparing by `accountNumber`.

5. **B.** The report building loops in this chunk use `for (BankAccount acc : accounts)`, the enhanced for each loop from Chunk 1, iterating over an array to feed each element into repeated `append` calls.

6. **B.** `getAuditLog()`, once implemented inside `BankAccount`'s own class body, is an ordinary instance method of that class, and instance methods always have full access to their own class's private fields, entirely regardless of which interface originally declared the method's required signature.

7. **B.** This is precisely method overriding, from Chunk 3, applied to a method `String` inherits from `Object`; `String` provides its own implementation that better suits its purpose, content comparison, exactly the same mechanism `CheckingAccount` used to override `withdraw`.

8. **C.** Each `DepositTransaction` object's `execute` call resolves at runtime based on its actual class, the dynamic method dispatch mechanism introduced for class hierarchies in Chunk 3 and directly extended to abstract classes and their concrete subclasses in Chunk 4.

---

## 9. Programming Practice and Solutions

### Practice Problems

1. **Basic.** Write a method `initials(String fullName)` that returns the first letter of each space separated word, uppercased, concatenated together, using a combination of `String.split()` and `StringBuilder`.
2. **Intermediate.** Write a method `sanitizeHolderName(String raw)` that trims whitespace, collapses any internal run of multiple spaces down to a single space using `replace` style logic with a loop, and capitalizes the first letter, returning the cleaned result.
3. **Intermediate.** Override `equals()` and `hashCode()` for `Branch`, from Chunk 4, considering two branches equal if they share the same `branchName`, ignoring case, using `equalsIgnoreCase()` inside `equals()` and a consistent, case normalized `hashCode()`.
4. **Advanced.** Write a method that takes a raw `String` of semicolon separated `"name:balance"` pairs, uses `StringTokenizer` to parse it into individual `BankAccount` objects, stores them in a `Branch`, and returns a full report `String` built with `StringBuilder`, combining parsing, object construction, composition from Chunk 4, and efficient string building all in one method.

### Solutions

**Solution 1.**

```java
public class Solution1 {
    static String initials(String fullName) {
        String[] words = fullName.trim().split(" ");
        StringBuilder result = new StringBuilder();
        for (String word : words) {
            if (!word.isEmpty()) {
                result.append(Character.toUpperCase(word.charAt(0)));
            }
        }
        return result.toString();
    }

    public static void main(String[] args) {
        System.out.println(initials("priya sharma nair"));
    }
}
```

Expected output: `PSN`

Why it works: `split(" ")` divides the trimmed name into individual words, and the enhanced for loop, from Chunk 1, appends each word's first character, uppercased via `Character.toUpperCase`, onto the shared `StringBuilder`, avoiding repeated `String` concatenation.

**Solution 2.**

```java
public class Solution2 {
    static String sanitizeHolderName(String raw) {
        String trimmed = raw.trim();
        StringBuilder cleaned = new StringBuilder();
        boolean lastWasSpace = false;
        for (int i = 0; i < trimmed.length(); i++) {
            char c = trimmed.charAt(i);
            if (c == ' ') {
                if (!lastWasSpace) {
                    cleaned.append(c);
                }
                lastWasSpace = true;
            } else {
                cleaned.append(c);
                lastWasSpace = false;
            }
        }
        String result = cleaned.toString();
        if (result.isEmpty()) {
            return result;
        }
        return result.substring(0, 1).toUpperCase() + result.substring(1);
    }

    public static void main(String[] args) {
        System.out.println("[" + sanitizeHolderName("   priya   sharma  ") + "]");
    }
}
```

Expected output: `[Priya sharma]`

Why it works: the traditional `for` loop from Chunk 1 walks character by character, only appending a space when the previous character was not also a space, correctly collapsing any run of multiple spaces down to one, and the final `substring` based capitalization mirrors Section 3's `getFormattedHolderName` technique.

**Solution 3.**

```java
// inside Branch
@Override
public boolean equals(Object other) {
    if (this == other) {
        return true;
    }
    if (!(other instanceof Branch)) {
        return false;
    }
    Branch that = (Branch) other;
    return this.branchName.equalsIgnoreCase(that.branchName);
}

@Override
public int hashCode() {
    return branchName.toLowerCase().hashCode();
}
```

Why it works: `equalsIgnoreCase` performs the case insensitive comparison `equals()` needs, and `hashCode()` normalizes the same field to lowercase before hashing it, guaranteeing that two branches considered equal, regardless of casing differences in their names, also produce identical hash codes, correctly satisfying the contract from Section 7.

**Solution 4.**

```java
import java.util.StringTokenizer;

public class Solution4 {
    static String setupBranchFromRawData(String rawData, String branchName) {
        StringTokenizer outer = new StringTokenizer(rawData, ";");
        BankAccount[] accounts = new BankAccount[outer.countTokens()];
        int index = 0;

        while (outer.hasMoreTokens()) {
            String pair = outer.nextToken();
            StringTokenizer inner = new StringTokenizer(pair, ":");
            String name = inner.nextToken();
            double balance = Double.parseDouble(inner.nextToken());
            accounts[index] = new BankAccount(name, balance);
            index++;
        }

        Branch branch = new Branch(branchName, accounts);

        StringBuilder report = new StringBuilder();
        report.append("Branch: ").append(branchName).append("\n");
        for (BankAccount acc : accounts) {
            report.append(acc.getAccountHolder()).append(": ").append(acc.getBalance()).append("\n");
        }
        report.append("Total holdings: ").append(branch.getTotalHoldings());

        return report.toString();
    }

    public static void main(String[] args) {
        String raw = "Wren:500;Om:2000;Priyanka:300";
        System.out.println(setupBranchFromRawData(raw, "Main Street"));
    }
}
```

Expected output:
```
Branch: Main Street
Wren: 500.0
Om: 2000.0
Priyanka: 300.0
Total holdings: 2800.0
```

Why it works: `countTokens()` is called before any consumption to correctly size the `accounts` array up front, the outer and inner `StringTokenizer` objects parse the nested structure exactly as in Section 6's practice problem, and the final `StringBuilder` block builds the whole report in one efficient pass, tying together parsing, object construction, the `Branch` composition relationship from Chunk 4, and efficient string building from this chunk, all in a single cohesive method.

---

## 10. Revision Summary

| Concept | Key Rule | Where It Appeared |
|---|---|---|
| String immutability | Every "modifying" method returns a new `String`; the original is never changed | `holder.toUpperCase()` discarded result |
| String pool | Literals are pooled and reused; `new String(...)` always creates a separate object | `s1 == s2` true, `s1 == s3` false |
| `==` vs `.equals()` | `==` compares references; `.equals()` compares content, for `String` and any type that overrides it | Pooling example |
| String constructors | `new String(char[])`, `new String(char[], offset, count)`, and others build a `String` from raw data | Extracting a last name from a `char[]` |
| Core String API | `substring`'s end index is exclusive; most methods return, rather than mutate | `"Banking".substring(0, 4)` → `"Bank"` |
| StringBuffer | Mutable, synchronized, thread safe; `append` returns `this` for chaining | `Branch.buildFullReport()` |
| StringBuilder | Same API as `StringBuffer`, not synchronized, faster for single threaded use | Section 5's rewritten report builder |
| StringTokenizer | Legacy delimiter based splitting; `hasMoreTokens()` before `nextToken()` | Parsing `"name:balance"` pairs |
| equals/hashCode contract | Equal objects must share a hash code; unequal objects may still collide | `BankAccount.equals()` by `accountNumber` |

**Exam tip.** Whenever a question involves `String` comparison, check first whether `==` or `.equals()` is being used, and whether either operand was built with `new`; this single check resolves the large majority of `String` output questions. Whenever a question involves a custom class's `equals()` or `hashCode()`, check whether both were overridden together and whether they are genuinely based on the same fields; a mismatch between the two is the single most common bug pattern this topic tests.