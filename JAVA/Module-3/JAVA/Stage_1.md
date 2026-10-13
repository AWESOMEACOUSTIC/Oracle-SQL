# ☕ Java Stage 1: Fundamentals, Done Properly

> **Goal of this stage:** Not just "I can write Java code" but "I know *why* Java behaves the way it does."
> Every later topic (OOP, Collections, Spring, interviews) stands on what's in this file.

**How to use this guide**
1. Read one section at a time. Type out every example yourself; don't copy-paste.
2. Change the examples and break them on purpose. Predict the output *before* running.
3. At the end, do the practice section **without looking at the answers**. Answers are hidden inside expandable blocks.

---

## 📍 Table of Contents

1. [How Java Actually Runs](#1-how-java-actually-runs)
2. [Anatomy of a Java Program](#2-anatomy-of-a-java-program)
3. [Variables and Data Types](#3-variables-and-data-types)
4. [Memory: Stack vs Heap](#4-memory-stack-vs-heap)
5. [Type Conversion and Casting](#5-type-conversion-and-casting)
6. [Operators](#6-operators)
7. [Strings (The Most Asked Topic)](#7-strings-the-most-asked-topic)
8. [Input and Output](#8-input-and-output)
9. [Control Flow](#9-control-flow)
10. [Methods](#10-methods)
11. [Arrays](#11-arrays)
12. [The Math Class and Handy Utilities](#12-the-math-class-and-handy-utilities)
13. [Trap Summary: The Cheat Sheet](#13-trap-summary-the-cheat-sheet)
14. [Practice Section](#14-practice-section)

---

## 1. How Java Actually Runs

### The big idea

When you write C, the compiler turns your code into machine code for **one specific** operating system. Java does something smarter: it compiles your code into an in-between language called **bytecode**, and a program called the **JVM** runs that bytecode on any machine.

That's the famous slogan: **"Write Once, Run Anywhere."**

Think of it like this: you write a recipe in English (bytecode). Every country has a translator (JVM) who reads the English recipe and cooks it in the local kitchen. You don't rewrite the recipe for every country.

### The flow

```
  Hello.java          javac            Hello.class           java             Output
 (source code)  ───────────────►   (bytecode)   ───────────────────►   "Hello!"
  you write it      compiler        platform-         JVM loads,
                                     independent      verifies, runs
```

Inside the JVM, roughly this happens:

```
Hello.class
    │
    ▼
┌──────────────────┐
│   Class Loader   │  → loads the .class file into memory
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Bytecode Verifier│  → checks the code is safe and valid
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Execution Engine │  → Interpreter runs code line by line;
│ (Interpreter+JIT)│    JIT compiler turns "hot" (frequently used)
└──────────────────┘    code into native machine code for speed
```

### JDK vs JRE vs JVM

People mix these up all the time. Picture three boxes, one inside another:

```
┌───────────────────────────────────────────────┐
│ JDK  (Java Development Kit)                   │
│   → tools for DEVELOPERS: javac, jar,         │
│     javadoc, debugger...                      │
│  ┌─────────────────────────────────────────┐  │
│  │ JRE  (Java Runtime Environment)         │  │
│  │   → what you need to RUN Java           │  │
│  │     (JVM + core libraries)              │  │
│  │  ┌───────────────────────────────────┐  │  │
│  │  │ JVM  (Java Virtual Machine)       │  │  │
│  │  │   → actually executes bytecode    │  │  │
│  │  └───────────────────────────────────┘  │  │
│  └─────────────────────────────────────────┘  │
└───────────────────────────────────────────────┘
```

| Term | What it is | Who needs it |
|------|-----------|--------------|
| **JVM** | The engine that runs bytecode. It is platform-*specific* (there's a Windows JVM, a Linux JVM, and so on). | Everyone running Java |
| **JRE** | JVM plus standard libraries (`java.lang`, `java.util`, ...) | Users who only run Java apps |
| **JDK** | JRE plus development tools (`javac`, `jar`, `jdb`, ...) | Developers (that's you) |

> 💡 **Interview line:** "Bytecode is platform-independent; the JVM is platform-dependent. That's how Java achieves portability."

> 📝 Since Java 11, Oracle no longer ships a separate JRE download. You just install the JDK. The *concept* of the JRE still matters for understanding and for interviews.

### Try it without an IDE (do this at least once!)

```java
// File: Hello.java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
    }
}
```

```bash
javac Hello.java      # creates Hello.class (bytecode)
java Hello            # runs it — note: no ".class" extension
```

Since Java 11, you can also run a single file directly: `java Hello.java` (it compiles in memory behind the scenes).

### Is Java compiled or interpreted?

**Both.** It is *compiled* to bytecode by `javac`, then *interpreted and JIT-compiled* by the JVM at runtime.

---

## 2. Anatomy of a Java Program

```java
package com.learning;              // (optional) folder-like grouping of classes

import java.util.Scanner;          // bring in classes from other packages

public class Calculator {          // class name MUST match file name: Calculator.java

    // Entry point — the JVM looks for exactly this signature
    public static void main(String[] args) {
        int a = 5;                 // statement ends with ;
        int b = 3;
        System.out.println(a + b); // prints 8
    }
}
```

### Breaking down `public static void main(String[] args)`

| Part | Why it's there |
|------|----------------|
| `public` | The JVM is "outside" your class, so it must be able to access main. |
| `static` | The JVM calls main **without creating an object** of your class. |
| `void` | main returns nothing to the JVM. |
| `main` | The exact name the JVM searches for. |
| `String[] args` | Command-line arguments: `java Calculator 10 20` → `args[0]="10"`, `args[1]="20"` |

Valid variations: `String args[]`, `String... args`, and `static public void main(...)` (modifier order doesn't matter).

> 📝 **Modern Java note:** Java 25 (LTS) finalized *compact source files and instance main methods*, so a beginner program can be as short as `void main() { IO.println("Hi"); }`. It's nice for scripts, but you'll see the classic form everywhere in real code and in interviews, so master that one.

### Rules worth knowing

- One file can have many classes but **at most one `public` class**, and the file name must match it.
- Java is **case-sensitive**: `Main` ≠ `main`, `String` ≠ `string`.
- Comments: `// single line`, `/* multi line */`, `/** Javadoc */`.

### Identifiers (names) and conventions

**Rules (compiler enforces):**
- Can contain letters, digits, `_`, `$`
- Can't **start** with a digit
- Can't be a keyword (`int`, `class`, `static`...)

**Conventions (professionals follow):**

| Thing | Style | Example |
|-------|-------|---------|
| Class | PascalCase | `BankAccount` |
| Method / variable | camelCase | `calculateTotal`, `userName` |
| Constant | UPPER_SNAKE_CASE | `MAX_SPEED` |
| Package | lowercase | `com.company.app` |

---

## 3. Variables and Data Types

A **variable** is a labelled box that holds a value. In Java, every box has a **fixed type**: you decide upfront what kind of thing goes in it. Java is **statically typed**.

```java
int age = 25;           // declare + initialize
double price;           // declare only
price = 99.99;          // assign later
final int MAX = 100;    // final = constant, can't reassign
```

### Two families of types

```
                     Data Types
                         │
         ┌───────────────┴────────────────┐
     Primitive                        Reference
  (stores the actual value)     (stores an address/pointer to an object)
         │                                │
  byte short int long             String, arrays, classes,
  float double char boolean       interfaces, everything else
```

### The 8 primitive types (memorize this table)

| Type | Size | Range | Default* | Example |
|------|------|-------|----------|---------|
| `byte` | 8 bits | -128 to 127 | `0` | `byte b = 100;` |
| `short` | 16 bits | -32,768 to 32,767 | `0` | `short s = 30000;` |
| `int` | 32 bits | about ±2.1 billion (-2³¹ to 2³¹-1) | `0` | `int i = 2_000_000;` |
| `long` | 64 bits | about ±9.2 quintillion | `0L` | `long l = 9_000_000_000L;` |
| `float` | 32 bits | ~6–7 decimal digits precision | `0.0f` | `float f = 3.14f;` |
| `double` | 64 bits | ~15–16 decimal digits precision | `0.0` | `double d = 3.14159;` |
| `char` | 16 bits | 0 to 65,535 (unsigned Unicode) | `'\u0000'` | `char c = 'A';` |
| `boolean` | JVM-dependent | `true` / `false` | `false` | `boolean ok = true;` |

\*Defaults apply only to **fields** (class-level variables) and **array elements**, not to local variables. See below.

**Quick memory trick:** byte → short → int → long (1, 2, 4, 8 bytes, doubling each time).

### Literals: how you write values

```java
int decimal  = 42;
int hex      = 0x2A;        // 42 in hexadecimal
int binary   = 0b101010;    // 42 in binary
int octal    = 052;         // 42 in octal ⚠️ leading 0 = octal!
int readable = 1_000_000;   // underscores for readability (Java 7+)

long big     = 9_000_000_000L;   // L suffix REQUIRED beyond int range
float f      = 3.14f;            // f suffix REQUIRED (3.14 alone is a double)
double d     = 3.14;             // decimals are double by default
char c1      = 'A';
char c2      = 65;               // also 'A' (Unicode/ASCII value)
char c3      = '\u0041';         // also 'A' (Unicode escape)
```

**Defaults the compiler assumes:**
- Any whole number literal is an **`int`**.
- Any decimal literal is a **`double`**.

That's why these fail:

```java
long x = 3000000000;   // ❌ error: integer number too large (it's an int literal!)
float y = 3.14;        // ❌ error: possible lossy conversion from double to float
```

### Escape sequences in char/String

| Escape | Meaning |
|--------|---------|
| `\n` | new line |
| `\t` | tab |
| `\"` | double quote |
| `\'` | single quote |
| `\\` | backslash |

### Local variables have NO default value

```java
public class Demo {
    static int count;          // field → default 0 ✅

    public static void main(String[] args) {
        int x;                 // local variable
        System.out.println(count);  // 0
        System.out.println(x);      // ❌ compile error: variable x might not have been initialized
    }
}
```

**Why?** Fields belong to objects that the JVM zero-fills when created. Local variables live on the stack and Java forces you to initialize them, which prevents garbage values like you'd get in C.

### Reference types: the default is `null`

```java
String name;           // as a field → default null
int[] numbers;         // as a field → default null
```

`null` means "this box points to nothing." Calling a method on it gives a `NullPointerException`.

---

## 4. Memory: Stack vs Heap

This is the section that makes everything else make sense: strings, arrays, pass-by-value, all of it.

### Simple version

| | **Stack** | **Heap** |
|---|---|---|
| Stores | Method calls, local variables, **references** | **Objects** (and arrays, Strings) |
| Lifetime | Gone when the method ends | Lives until the Garbage Collector removes it (when nothing references it) |
| Speed | Very fast | Slower |
| Size | Small, per thread | Large, shared |
| Error when full | `StackOverflowError` | `OutOfMemoryError` |

### Picture it

```java
public static void main(String[] args) {
    int age = 25;
    String name = new String("Riya");
    int[] marks = {90, 85, 70};
}
```

```
        STACK (main's frame)                    HEAP
   ┌──────────────────────────┐
   │ age   │ 25               │       (primitive: value stored directly)
   ├──────────────────────────┤
   │ name  │ ref ──────────────┼──────►  String object "Riya"
   ├──────────────────────────┤
   │ marks │ ref ──────────────┼──────►  int[] {90, 85, 70}
   └──────────────────────────┘
```

**Key insight:** for a primitive, the box holds the **value**. For a reference, the box holds an **address** (like a house address written on paper), and the actual house (object) is in the heap.

### Why this matters: copying references

```java
int a = 10;
int b = a;      // copies the VALUE 10
b = 20;
System.out.println(a);   // 10  (a is unaffected)

int[] x = {1, 2, 3};
int[] y = x;    // copies the ADDRESS, not the array!
y[0] = 99;
System.out.println(x[0]);  // 99  (both point to the same array)
```

Two variables, one array. Like two people holding the same house address: if one paints the house, the other sees it painted.

### Garbage Collection (just the basics for now)

```java
int[] arr = new int[1000];
arr = null;   // the array has no references now → eligible for garbage collection
```

You never `free` or `delete` memory in Java. The GC does it automatically. Deep GC internals come in Stage 5.

---

## 5. Type Conversion and Casting

### Widening (automatic, safe)

Smaller type into bigger type: Java does it for you, no data loss.

```
byte → short → int → long → float → double
              ↑
            char
```

```java
int i = 100;
long l = i;        // ✅ automatic
double d = l;      // ✅ automatic → 100.0
char c = 'A';
int code = c;      // ✅ 65
```

> ⚠️ `long → float` is "widening" but can **lose precision** (float has only ~7 significant digits). Java still allows it automatically.

### Narrowing (manual, risky)

Bigger type into smaller type: you must cast explicitly, and data may be lost.

```java
double price = 99.99;
int rounded = (int) price;     // 99 (decimal part CHOPPED, not rounded)

int big = 130;
byte small = (byte) big;       // -126 (overflow wraps around!)
```

**Why -126?** A byte holds -128 to 127. Going past 127 wraps around to -128 like a car odometer: 128 → -128, 129 → -127, 130 → -126.

**Formula:** if the value is out of range, subtract (or add) 256 for byte until it fits. 130 - 256 = -126.

### Type promotion in expressions (huge trap)

**Rule:** in any arithmetic, `byte`, `short`, and `char` are first promoted to **`int`**. And if one operand is long/float/double, the whole expression becomes that bigger type.

```java
byte a = 10, b = 20;
byte c = a + b;          // ❌ compile error! a + b is an int
byte c = (byte)(a + b);  // ✅

byte x = 10;
x = x + 1;               // ❌ compile error (x + 1 is int)
x += 1;                  // ✅ compound operators include an implicit cast
x++;                     // ✅ same

char ch = 'A';
System.out.println(ch + 1);          // 66  (int!)
System.out.println((char)(ch + 1));  // B
```

### Integer division and mixing types

```java
System.out.println(7 / 2);      // 3     (int / int = int, decimals dropped)
System.out.println(7 / 2.0);    // 3.5   (one double → whole thing double)
System.out.println(7.0 / 2);    // 3.5
System.out.println((double) 7 / 2);   // 3.5  (cast happens BEFORE division)
System.out.println((double)(7 / 2));  // 3.0  (division happens FIRST → 3 → 3.0)
```

### Overflow: silent and dangerous

```java
int max = Integer.MAX_VALUE;   // 2147483647
System.out.println(max + 1);   // -2147483648  (no error, just wraps!)

long wrong = 1_000_000 * 1_000_000;   // -727379968 😱  (int * int overflows FIRST)
long right = 1_000_000L * 1_000_000;  // 1000000000000 ✅
```

Java does **not** throw an error on integer overflow. It wraps silently. Always think "could this exceed int?"

### Floating point is not exact

```java
System.out.println(0.1 + 0.2);          // 0.30000000000000004
System.out.println(0.1 + 0.2 == 0.3);   // false!
```

Computers store decimals in binary, and 0.1 can't be represented exactly (like 1/3 = 0.333... in decimal). **Never use `double` for money**; use `BigDecimal` (Stage 3) or store paise/cents as `long`.

### Division by zero

```java
System.out.println(10 / 0);     // ❌ ArithmeticException (integers)
System.out.println(10.0 / 0);   // Infinity (floating point)
System.out.println(-10.0 / 0);  // -Infinity
System.out.println(0.0 / 0);    // NaN (Not a Number)
```

---

## 6. Operators

### Arithmetic

| Operator | Meaning | Example | Result |
|----------|---------|---------|--------|
| `+` | add (also String concat) | `5 + 3` | `8` |
| `-` | subtract | `5 - 3` | `2` |
| `*` | multiply | `5 * 3` | `15` |
| `/` | divide | `5 / 3` | `1` |
| `%` | remainder (modulus) | `5 % 3` | `2` |

**Modulus with negatives:** the sign of the result follows the **left** operand.

```java
System.out.println(-7 % 3);   // -1
System.out.println(7 % -3);   // 1
System.out.println(-7 / 2);   // -3  (truncates toward zero, not -4)
```

**Common `%` uses:** check even/odd (`n % 2 == 0`), get the last digit (`n % 10`), wrap around (`index % length`).

### Increment and decrement: pre vs post

```java
int a = 5;
int b = a++;   // POST: use first, then increment → b = 5, a = 6
int c = ++a;   // PRE:  increment first, then use → a = 7, c = 7
```

**Memory trick:** read left to right. `a++`: you see `a` first (use it), then `++`. `++a`: you see `++` first (increment), then `a`.

```java
int x = 10;
int y = x++ + ++x;
// x++ → uses 10, x becomes 11
// ++x → x becomes 12, uses 12
// y = 10 + 12 = 22, x = 12
```

The famous trap:

```java
int i = 5;
i = i++;
System.out.println(i);   // 5 (!!)
// i++ returns 5 and sets i to 6, then the assignment overwrites i with 5
```

### Relational (always give a boolean)

`==`, `!=`, `>`, `<`, `>=`, `<=`

⚠️ For objects, `==` compares **references (addresses)**, not content. More in the Strings section.

### Logical

| Operator | Meaning | Short-circuits? |
|----------|---------|-----------------|
| `&&` | AND | ✅ Yes: if the left side is false, it skips the right |
| `\|\|` | OR | ✅ Yes: if the left side is true, it skips the right |
| `!` | NOT | n/a |
| `&` | AND (on booleans) | ❌ No: always evaluates both |
| `\|` | OR (on booleans) | ❌ No |
| `^` | XOR: true if the two differ | ❌ No |

```java
int x = 5;
if (x > 10 && ++x > 0) { }
System.out.println(x);   // 5 (++x never ran)

if (x > 10 & ++x > 0) { }
System.out.println(x);   // 6 (both sides ran)
```

**Real-world use of short-circuit:** `if (name != null && name.length() > 3)` is safe. If `name` is null, the second part never runs, so no `NullPointerException`.

### Assignment and compound assignment

`=`, `+=`, `-=`, `*=`, `/=`, `%=`. Compound operators **auto-cast** back to the left side's type:

```java
int n = 10;
n += 2.7;      // n = (int)(10 + 2.7) = 12  (compiles fine!)
```

### Ternary (a compact if-else)

```java
int age = 20;
String status = (age >= 18) ? "Adult" : "Minor";
```

Format: `condition ? valueIfTrue : valueIfFalse`. Great for simple choices; don't nest it deeply, because it gets unreadable.

### Bitwise (awareness level)

| Op | Name | Example (5 = 0101, 3 = 0011) |
|----|------|------------------------------|
| `&` | AND | `5 & 3` = 1 (0001) |
| `\|` | OR | `5 \| 3` = 7 (0111) |
| `^` | XOR | `5 ^ 3` = 6 (0110) |
| `~` | NOT | `~5` = -6 |
| `<<` | left shift | `5 << 1` = 10 (×2) |
| `>>` | right shift (keeps sign) | `-8 >> 1` = -4 |
| `>>>` | unsigned right shift | fills with 0s |

`n << k` is the same as n × 2ᵏ, and `n >> k` is the same as n ÷ 2ᵏ (for positives). You'll see these in DSA problems.

### Precedence (top = evaluated first)

```
1.  ()   [] .                    ← parentheses, array access, member access
2.  ++ -- (postfix)
3.  ++ -- (prefix), + - (unary), !, ~, (cast)
4.  *  /  %
5.  +  -
6.  << >> >>>
7.  <  >  <=  >=  instanceof
8.  ==  !=
9.  &
10. ^
11. |
12. &&
13. ||
14. ?:                           ← ternary
15. =  +=  -=  ...               ← assignment (right to left)
```

**Golden rule:** when in doubt, use parentheses. Code is read by humans.

### `+` with Strings: left to right!

```java
System.out.println(1 + 2 + "3");     // "33"  → (1+2)=3, then 3+"3"
System.out.println("1" + 2 + 3);     // "123" → "1"+2="12", then "12"+3
System.out.println("1" + (2 + 3));   // "15"
System.out.println('A' + 'B');       // 131   → chars become ints! (65+66)
System.out.println("" + 'A' + 'B');  // "AB"
```

Once a String enters the chain, everything after it becomes concatenation.

---

## 7. Strings (The Most Asked Topic)

### What is a String?

A `String` is an **object** (not a primitive) that represents a sequence of characters. It's so common that Java gives it special treatment: literals, the `+` operator, and the String Pool.

```java
String s1 = "Hello";                // literal (goes to the String Pool)
String s2 = new String("Hello");    // forces a NEW object in the heap
```

### 1️⃣ Strings are IMMUTABLE

Once created, a String object **can never change**. Any "modification" creates a **new** String.

```java
String s = "java";
s.toUpperCase();
System.out.println(s);     // "java" ← unchanged! The result was thrown away

s = s.toUpperCase();       // now s points to a NEW object "JAVA"
System.out.println(s);     // "JAVA"
```

```
Before:  s ───► "java"
After s = s.toUpperCase():
         s ───► "JAVA"       "java" (still exists, now unreferenced → GC eligible)
```

**Why did Java make Strings immutable?**
1. **String Pool**: sharing is only safe if nobody can change the shared object.
2. **Security**: file paths, URLs, and DB credentials can't be altered after validation.
3. **Thread-safety**: immutable objects can be shared across threads freely.
4. **Hashcode caching**: makes Strings fast as HashMap keys (you'll love this in Collections).

### 2️⃣ The String Pool

The String Pool is a special area in the heap where Java stores **one copy** of each string literal. When you write a literal, Java first checks the pool: if it's already there, it reuses it.

```java
String a = "Hello";
String b = "Hello";
String c = new String("Hello");

System.out.println(a == b);       // true  → same pooled object
System.out.println(a == c);       // false → c is a separate heap object
System.out.println(a.equals(c));  // true  → same content
```

```
HEAP
┌────────────────────────────────────────┐
│   String Pool                          │
│   ┌──────────┐                         │
│   │ "Hello"  │ ◄──── a                 │
│   └──────────┘ ◄──── b                 │
│                                        │
│   ┌──────────┐                         │
│   │ "Hello"  │ ◄──── c  (via new)      │
│   └──────────┘                         │
└────────────────────────────────────────┘
```

> 💡 `new String("Hello")` can create **up to 2 objects**: one in the pool (if "Hello" isn't there yet) and one in the regular heap.

### 3️⃣ `==` vs `.equals()`: never confuse these

| | `==` | `.equals()` |
|---|---|---|
| Compares | References (are they the **same object**?) | Content (do they hold the **same characters**?) |
| Use for Strings? | ❌ Almost never | ✅ Always |

**Analogy:** two printed copies of the same book. `==` asks "is this the *same physical book*?" while `.equals()` asks "do they have the *same text*?"

### 4️⃣ Compile-time vs runtime concatenation (classic interview trap)

```java
String s1 = "Java";
String s2 = "Ja" + "va";        // both literals → compiler joins them at COMPILE time → "Java" from pool
System.out.println(s1 == s2);   // true

String part = "Ja";
String s3 = part + "va";        // part is a variable → joined at RUNTIME → new heap object
System.out.println(s1 == s3);   // false

final String fpart = "Ja";      // final + literal = compile-time constant
String s4 = fpart + "va";       // compiler can join it
System.out.println(s1 == s4);   // true

System.out.println(s1 == s3.intern());  // true → intern() returns the pool copy
```

**Rule:** if every piece is a **compile-time constant** (literals, or `final` variables initialized with literals), concatenation happens at compile time and the result is pooled. Otherwise, it's a new object at runtime.

### 5️⃣ Essential String methods

```java
String s = "  Hello World  ";
String t = "Hello World";

t.length()                 // 11
t.charAt(4)                // 'o'  (index starts at 0)
t.indexOf('o')             // 4    (first occurrence)
t.lastIndexOf('o')         // 7
t.indexOf("xyz")           // -1   (not found)
t.substring(6)             // "World"  (from 6 to end)
t.substring(0, 5)          // "Hello"  (start inclusive, END EXCLUSIVE)
t.contains("World")        // true
t.startsWith("He")         // true
t.endsWith("ld")           // true
t.toUpperCase()            // "HELLO WORLD"
t.toLowerCase()            // "hello world"
t.replace('l', 'L')        // "HeLLo WorLd"
t.replace("World", "Java") // "Hello Java"
s.trim()                   // "Hello World" (removes leading/trailing spaces)
s.strip()                  // same, but Unicode-aware (Java 11+)
"".isEmpty()               // true
"   ".isBlank()            // true (Java 11+)
t.equalsIgnoreCase("hello world")  // true
"apple".compareTo("banana")// negative number (a comes before b)
t.split(" ")               // ["Hello", "World"]
String.join("-", "a", "b", "c")    // "a-b-c"
String.valueOf(42)         // "42"
t.toCharArray()            // ['H','e','l','l','o',' ','W','o','r','l','d']
"ab".repeat(3)             // "ababab" (Java 11+)
```

**`substring` trick:** `substring(start, end)` returns `end - start` characters. `"Hello".substring(1, 3)` gives 3 - 1 = 2 chars: `"el"`.

**`compareTo`:** returns 0 if equal, negative if the first comes earlier in dictionary order, positive if later. Used for sorting.

### 6️⃣ StringBuilder: the mutable alternative

Since Strings are immutable, this is wasteful:

```java
String result = "";
for (int i = 0; i < 10000; i++) {
    result = result + i;   // creates a NEW String every iteration 😬 (10,000 objects!)
}
```

Use `StringBuilder`, which modifies the same object:

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);          // same object, just grows
}
String result = sb.toString();
```

**Common StringBuilder methods:**

```java
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");       // "Hello World"
sb.insert(0, ">> ");       // ">> Hello World"
sb.reverse();              // "dlroW olleH >>"
sb.reverse();              // back to ">> Hello World"
sb.deleteCharAt(0);        // "> Hello World"
sb.delete(0, 2);           // "Hello World"
sb.setCharAt(0, 'J');      // "Jello World"
sb.replace(0, 5, "Hi");    // "Hi World"
sb.length();               // 8
sb.toString();             // convert back to String
```

> ⚠️ `StringBuilder` does **not** override `equals()`, so `sb1.equals(sb2)` compares references. Compare with `sb1.toString().equals(sb2.toString())` or `sb1.compareTo(sb2) == 0` (Java 11+).

### String vs StringBuilder vs StringBuffer

| | `String` | `StringBuilder` | `StringBuffer` |
|---|---|---|---|
| Mutable? | ❌ No | ✅ Yes | ✅ Yes |
| Thread-safe? | ✅ Yes (immutable) | ❌ No | ✅ Yes (synchronized) |
| Speed | Slow for repeated changes | **Fastest** | Slower (locking overhead) |
| Use when | Value doesn't change much | Building strings (single thread), which is most cases | Legacy multi-threaded code |

---

## 8. Input and Output

### Output

```java
System.out.println("Hello");     // prints and moves to a new line
System.out.print("Hello");       // prints, stays on the same line
System.out.printf("Name: %s, Age: %d, GPA: %.2f%n", "Riya", 21, 8.756);
// Name: Riya, Age: 21, GPA: 8.76

String msg = String.format("%05d", 42);   // "00042"
```

| Format | Meaning |
|--------|---------|
| `%d` | integer |
| `%f` / `%.2f` | decimal / 2 decimal places |
| `%s` | string |
| `%c` | char |
| `%b` | boolean |
| `%n` | new line (platform-independent) |
| `%5d` / `%-5d` | width 5, right / left aligned |

### Input with Scanner

```java
import java.util.Scanner;

public class InputDemo {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);

        System.out.print("Enter age: ");
        int age = sc.nextInt();

        System.out.print("Enter name: ");
        String name = sc.next();          // reads ONE word

        sc.nextLine();                    // ⚠️ see the trap below
        System.out.print("Enter address: ");
        String address = sc.nextLine();   // reads the WHOLE line

        System.out.println(name + " (" + age + ") lives at " + address);
        sc.close();
    }
}
```

| Method | Reads |
|--------|-------|
| `nextInt()`, `nextDouble()`, `nextLong()`, `nextBoolean()` | the next token as that type |
| `next()` | the next word (stops at whitespace) |
| `nextLine()` | the rest of the current line |
| `hasNextInt()` | checks whether the next token is an int (useful for validation) |

### ⚠️ The `nextInt()` + `nextLine()` trap

When you type `25` and press Enter, the input buffer holds `25\n`. `nextInt()` reads `25` but **leaves the `\n` behind**. The next `nextLine()` sees that `\n` immediately and returns an empty string.

**Fix:** call an extra `sc.nextLine()` to consume the leftover newline, or read everything with `nextLine()` and parse it: `int age = Integer.parseInt(sc.nextLine());`

---

## 9. Control Flow

### if / else if / else

```java
int marks = 78;

if (marks >= 90) {
    System.out.println("A");
} else if (marks >= 75) {
    System.out.println("B");      // ← this runs
} else if (marks >= 50) {
    System.out.println("C");
} else {
    System.out.println("Fail");
}
```

Java checks from top to bottom and runs **only the first** matching block.

⚠️ The condition must be a `boolean`. `if (1)` is a compile error in Java (unlike C).

⚠️ Watch for the dangling semicolon:

```java
if (marks > 90);                        // ← this semicolon IS the if body (empty)!
    System.out.println("Topper");       // always runs
```

Always use braces `{}`, even for one line.

### switch (classic)

```java
int day = 3;
switch (day) {
    case 1:
        System.out.println("Monday");
        break;
    case 2:
        System.out.println("Tuesday");
        break;
    case 3:
        System.out.println("Wednesday");   // ← runs
        break;
    default:
        System.out.println("Invalid");
}
```

**Fall-through:** without `break`, execution "falls" into the next case:

```java
int day = 2;
switch (day) {
    case 1: System.out.print("Mon ");
    case 2: System.out.print("Tue ");     // matches here
    case 3: System.out.print("Wed ");     // no break above → also runs
            break;
    default: System.out.print("Other");
}
// Output: Tue Wed
```

Fall-through can be useful on purpose:

```java
switch (month) {
    case 12: case 1: case 2:
        System.out.println("Winter");
        break;
    case 3: case 4: case 5:
        System.out.println("Summer");
        break;
}
```

**What types can switch on?** `byte`, `short`, `char`, `int` (and their wrappers), `String` (Java 7+), and `enum`.
**Not allowed:** `long`, `float`, `double`, `boolean`.

`case` labels must be **constants** and **unique**. `default` is optional and can appear anywhere.

**Modern arrow switch (Java 14+)**, a preview of Stage 4: no fall-through, and it can return a value.

```java
String name = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 6, 7 -> "Weekend";
    default -> "Midweek";
};
```

### Loops: choosing the right one

```
Do you know how many times to repeat?
   │
   ├── Yes ──► for loop
   │
   └── No ──► Must it run at least once?
                 │
                 ├── Yes ──► do-while
                 └── No  ──► while
Iterating over every element of an array/collection? ──► enhanced for
```

```java
// for: known count
for (int i = 1; i <= 5; i++) {
    System.out.print(i + " ");       // 1 2 3 4 5
}

// while: checks the condition BEFORE each run (may run 0 times)
int n = 1234, sum = 0;
while (n > 0) {
    sum += n % 10;                   // take the last digit
    n /= 10;                         // drop the last digit
}
System.out.println(sum);             // 10

// do-while: runs at least ONCE (checks AFTER)
int x = 10;
do {
    System.out.println(x);           // prints 10 even though x < 5 is false
    x++;
} while (x < 5);

// enhanced for (for-each): read every element
int[] nums = {4, 8, 15};
for (int num : nums) {
    System.out.print(num + " ");     // 4 8 15
}
```

> ⚠️ In a for-each loop, `num` is a **copy**. Changing it doesn't change the array. To modify elements, use an index loop.

**Anatomy of a for loop:**

```
for ( init ; condition ; update ) { body }
        │        │           │
        1        2 ───► body ──► 3 ──► back to 2 ...
     (once)   (checked
              every time)
```

All three parts are optional: `for (;;) { }` is an infinite loop.

### break and continue

```java
for (int i = 1; i <= 10; i++) {
    if (i == 3) continue;    // skip the rest of THIS iteration
    if (i == 6) break;       // exit the loop entirely
    System.out.print(i + " ");
}
// Output: 1 2 4 5
```

### Labeled break and continue (for nested loops)

```java
outer:
for (int i = 1; i <= 3; i++) {
    for (int j = 1; j <= 3; j++) {
        if (i * j == 4) {
            break outer;     // breaks out of BOTH loops
        }
        System.out.print(i + "" + j + " ");
    }
}
// Output: 11 12 13 21
```

A plain `break` only exits the **innermost** loop. A label lets you choose which loop.

### Variable scope

A variable lives **only inside the `{ }` block** where it's declared.

```java
for (int i = 0; i < 3; i++) { }
System.out.println(i);     // ❌ error: i doesn't exist here

int total = 0;
if (true) {
    int bonus = 10;
    total += bonus;
}
System.out.println(bonus); // ❌ bonus is out of scope
```

---

## 10. Methods

A method is a named, reusable block of code. Think of it as a machine: inputs (parameters) go in, it does work, and an output (return value) comes out.

```java
//  modifiers   return-type  name    parameters
    public static   int       add   (int a, int b) {
        return a + b;    // ← body
    }
```

**Method signature** = name + parameter types (in order). The return type is **not** part of the signature.

```java
public class MathUtils {
    static int square(int n) {
        return n * n;
    }

    static void greet(String name) {    // void = returns nothing
        System.out.println("Hi " + name);
    }

    public static void main(String[] args) {
        int result = square(5);   // 5 is the ARGUMENT; n is the PARAMETER
        System.out.println(result);   // 25
        greet("Arjun");
    }
}
```

### Static vs non-static (just enough for now)

`main` is `static`, so it can call other `static` methods directly. To call a **non-static** method, you need an object:

```java
public class Demo {
    void hello() { System.out.println("hello"); }

    public static void main(String[] args) {
        hello();                 // ❌ error: non-static method cannot be referenced from a static context
        new Demo().hello();      // ✅
    }
}
```

(Full explanation in Stage 2, OOP.)

### ⭐ Java is ALWAYS pass-by-value

This is the #1 misunderstood concept. When you call a method, Java **copies** the value of the argument into the parameter.

- For a **primitive**, it copies the value itself.
- For a **reference**, it copies the **address** (not the object!).

So both caller and method hold the same address and can modify the same object. But **reassigning** the parameter only changes the method's copy.

**Case 1: primitive**

```java
static void tryChange(int x) {
    x = 100;
}
int num = 5;
tryChange(num);
System.out.println(num);   // 5 → only the copy changed
```

**Case 2: object, modifying its contents**

```java
static void modify(int[] arr) {
    arr[0] = 99;           // follows the address → changes the real array
}
int[] data = {1, 2, 3};
modify(data);
System.out.println(data[0]);   // 99
```

**Case 3: object, reassigning the parameter**

```java
static void reassign(int[] arr) {
    arr = new int[]{7, 8, 9};  // the method's copy now points elsewhere
    arr[0] = 50;               // modifies the NEW array, not the caller's
}
int[] data = {1, 2, 3};
reassign(data);
System.out.println(data[0]);   // 1 → caller's reference untouched
```

```
Before reassign:
   data ──┐
          ├──► [1, 2, 3]
   arr  ──┘

After arr = new int[]{7,8,9}:
   data ─────► [1, 2, 3]       (unchanged)
   arr  ─────► [50, 8, 9]
```

**Analogy:** I give you a photocopy of my house address. You can go to my house and repaint it (modify the object). But if you scribble a new address on your photocopy, my house doesn't move (reassignment).

**Proof that Java isn't pass-by-reference:** a swap method doesn't work.

```java
static void swap(String a, String b) {
    String temp = a; a = b; b = temp;
}
String x = "X", y = "Y";
swap(x, y);
System.out.println(x + y);   // "XY" → not swapped
```

### Method overloading

The **same name** with **different parameter lists** (number, type, or order of types).

```java
static int add(int a, int b)          { return a + b; }
static int add(int a, int b, int c)   { return a + b + c; }
static double add(double a, double b) { return a + b; }

add(2, 3);        // calls version 1
add(2, 3, 4);     // version 2
add(2.5, 3.5);    // version 3
```

❌ **Return type alone is NOT enough:**

```java
static int calc(int a)    { ... }
static double calc(int a) { ... }   // compile error: already defined
```

**How Java chooses (priority order):**
1. **Exact match**
2. **Widening** (int → long → double...)
3. **Boxing** (int → Integer)
4. **Varargs** (int...)

```java
static void show(long x)    { System.out.println("long"); }
static void show(Integer x) { System.out.println("Integer"); }
static void show(int... x)  { System.out.println("varargs"); }

show(5);   // "long" → widening beats boxing beats varargs
```

### Varargs (variable arguments)

```java
static int sum(int... numbers) {      // numbers is actually an int[]
    int total = 0;
    for (int n : numbers) total += n;
    return total;
}

sum();              // 0
sum(5);             // 5
sum(1, 2, 3, 4);    // 10
sum(new int[]{1,2});// 3
```

**Rules:** only **one** varargs parameter per method, and it must be the **last** parameter.
`void log(String level, String... messages)` ✅ `void log(String... messages, String level)` ❌

### Recursion

A method that calls itself. Every recursion needs:
1. A **base case** (when to stop)
2. A **recursive case** that moves toward the base case

```java
static int factorial(int n) {
    if (n <= 1) return 1;           // base case
    return n * factorial(n - 1);    // recursive case
}
```

How `factorial(4)` unfolds on the stack:

```
factorial(4) = 4 * factorial(3)
                    factorial(3) = 3 * factorial(2)
                                        factorial(2) = 2 * factorial(1)
                                                            factorial(1) = 1  ← base
                                        = 2 * 1 = 2
                    = 3 * 2 = 6
= 4 * 6 = 24
```

No base case (or one that's never reached) means infinite calls, the stack fills up, and you get a **`StackOverflowError`**.

---

## 11. Arrays

An array is a **fixed-size** container holding elements of the **same type**, stored together and accessed by index (starting at 0).

```
index:    0     1     2     3     4
        ┌─────┬─────┬─────┬─────┬─────┐
marks → │ 90  │ 85  │ 70  │ 95  │ 60  │
        └─────┴─────┴─────┴─────┴─────┘
         length = 5, last index = 4
```

### Creating arrays

```java
int[] a = new int[5];             // size 5, all elements = 0
int[] b = {10, 20, 30};           // declare + initialize
int[] c = new int[]{10, 20, 30};  // same (needed when not declaring on the same line)
int d[] = new int[3];             // C-style, valid but avoid

int[] e;
e = {1, 2, 3};                    // ❌ error: shorthand only works at declaration
e = new int[]{1, 2, 3};           // ✅
```

**Arrays are objects**, so they live in the heap and the variable holds a reference.

### Default values of elements

| Type | Default |
|------|---------|
| `int`, `long`, `short`, `byte` | `0` |
| `double`, `float` | `0.0` |
| `char` | `'\u0000'` |
| `boolean` | `false` |
| Any object (String etc.) | `null` |

```java
String[] names = new String[3];
System.out.println(names[0]);           // null
System.out.println(names[0].length());  // ❌ NullPointerException
```

### Accessing and the length

```java
int[] arr = {5, 10, 15};
arr[1] = 100;                      // modify
System.out.println(arr.length);    // 3  ← a FIELD, no parentheses!
System.out.println(arr[3]);        // ❌ ArrayIndexOutOfBoundsException
```

> 🧠 Don't confuse them: `arr.length` (array field) vs `str.length()` (String method) vs `list.size()` (collections).

**Size is fixed.** Once created, an array can't grow. (That's why `ArrayList` exists, in Stage 3.)

### Iterating

```java
int[] arr = {3, 7, 1, 9};

for (int i = 0; i < arr.length; i++) {      // when you need the index
    System.out.println(i + ": " + arr[i]);
}

for (int value : arr) {                      // when you just need the values
    System.out.println(value);
}

for (int i = arr.length - 1; i >= 0; i--) {  // backward
    System.out.print(arr[i] + " ");
}
```

### Copying arrays: reference vs real copy

```java
int[] original = {1, 2, 3};

int[] ref = original;                       // ❌ NOT a copy, same array
int[] copy1 = original.clone();             // ✅ new array
int[] copy2 = Arrays.copyOf(original, 3);   // ✅ new array
int[] copy3 = new int[3];
System.arraycopy(original, 0, copy3, 0, 3); // ✅ (src, srcPos, dest, destPos, length)

copy1[0] = 99;
System.out.println(original[0]);   // 1 (unaffected)
```

### Comparing arrays

```java
int[] x = {1, 2, 3};
int[] y = {1, 2, 3};
System.out.println(x == y);                 // false (different objects)
System.out.println(x.equals(y));            // false (arrays don't override equals!)
System.out.println(Arrays.equals(x, y));    // true  ✅
```

### The `Arrays` utility class (`import java.util.Arrays;`)

```java
int[] arr = {5, 2, 8, 1};

Arrays.toString(arr);            // "[5, 2, 8, 1]"  ← printing arr directly gives "[I@1b6d3586"
Arrays.sort(arr);                // arr is now [1, 2, 5, 8]
Arrays.binarySearch(arr, 5);     // 2  (array MUST be sorted first)
Arrays.fill(arr, 0);             // [0, 0, 0, 0]
Arrays.copyOf(arr, 6);           // new array of length 6, extra slots = 0
Arrays.copyOfRange(arr, 1, 3);   // elements at index 1, 2 (end exclusive)
Arrays.equals(a, b);             // content comparison
Arrays.deepToString(matrix);     // for 2D arrays
```

**Why does `System.out.println(arr)` print `[I@1b6d3586`?** Arrays use the default `Object.toString()`: `[I` means "array of int", followed by `@` and a hash code. Use `Arrays.toString(arr)`.

### 2D arrays: arrays of arrays

```java
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6}
};
// 2 rows, 3 columns

System.out.println(matrix[1][2]);      // 6 (row 1, column 2)
System.out.println(matrix.length);     // 2 (number of rows)
System.out.println(matrix[0].length);  // 3 (columns in row 0)

for (int r = 0; r < matrix.length; r++) {
    for (int c = 0; c < matrix[r].length; c++) {
        System.out.print(matrix[r][c] + " ");
    }
    System.out.println();
}
```

```
matrix ──► [ ref0 , ref1 ]
              │       │
              ▼       ▼
           [1,2,3] [4,5,6]
```

### Jagged arrays (rows of different lengths)

```java
int[][] jagged = new int[3][];   // rows created, but each row is null!
jagged[0] = new int[2];
jagged[1] = new int[4];
jagged[2] = new int[1];

int[][] tri = { {1}, {1, 1}, {1, 2, 1} };   // Pascal's triangle style
```

Accessing `jagged[0].length` **before** assigning `jagged[0]` gives a `NullPointerException`.

### `String[] args` revisited

```java
// java Greet Riya 21
public static void main(String[] args) {
    System.out.println(args.length);   // 2
    String name = args[0];              // "Riya"
    int age = Integer.parseInt(args[1]);// 21 (convert String → int)
}
```

---

## 12. The Math Class and Handy Utilities

```java
Math.max(3, 7)         // 7
Math.min(3, 7)         // 3
Math.abs(-5)           // 5
Math.pow(2, 10)        // 1024.0  ← returns DOUBLE
Math.sqrt(16)          // 4.0
Math.cbrt(27)          // 3.0
Math.round(2.5)        // 3    (returns long for double input)
Math.round(-2.5)       // -2   (rounds toward positive infinity at .5)
Math.floor(2.9)        // 2.0
Math.ceil(2.1)         // 3.0
Math.random()          // 0.0 ≤ x < 1.0

int dice = (int)(Math.random() * 6) + 1;   // random 1–6
```

**Wrapper class constants and parsing** (wrappers come fully in Stage 3):

```java
Integer.MAX_VALUE        // 2147483647
Integer.MIN_VALUE        // -2147483648
Long.MAX_VALUE
Integer.parseInt("123")  // 123 (int)
Double.parseDouble("3.5")// 3.5
Integer.parseInt("abc")  // ❌ NumberFormatException
String.valueOf(123)      // "123"
Character.isDigit('5')   // true
Character.isLetter('a')  // true
Character.toUpperCase('a') // 'A'
```

> 🤯 Fun trap: `Math.abs(Integer.MIN_VALUE)` returns `-2147483648`, because +2147483648 doesn't fit in an int, so it overflows back.

---

## 13. Trap Summary: The Cheat Sheet

| # | Trap | What actually happens |
|---|------|------------------------|
| 1 | `long x = 3000000000;` | Compile error, needs an `L` |
| 2 | `float f = 1.5;` | Compile error, needs an `f` |
| 3 | `int x; print(x);` (local) | Compile error, not initialized |
| 4 | `byte b = b + 1;` | Compile error (promoted to int). `b += 1` works |
| 5 | `7 / 2` | `3`, integer division |
| 6 | `(double)(7/2)` vs `(double)7/2` | `3.0` vs `3.5` |
| 7 | `Integer.MAX_VALUE + 1` | Wraps to `MIN_VALUE`, no error |
| 8 | `0.1 + 0.2 == 0.3` | `false` |
| 9 | `10 / 0` vs `10.0 / 0` | Exception vs `Infinity` |
| 10 | `int i = 5; i = i++;` | `i` stays `5` |
| 11 | `'A' + 'B'` | `131` (int), not `"AB"` |
| 12 | `1 + 2 + "3"` vs `"1" + 2 + 3` | `"33"` vs `"123"` |
| 13 | `s.toUpperCase();` without assigning | `s` unchanged (immutability) |
| 14 | `str1 == str2` | Compares references, not content |
| 15 | `"Ja" + "va" == "Java"` | `true` (compile-time constant) |
| 16 | `010` | `8` (octal!) |
| 17 | `switch` without `break` | Falls through to the next cases |
| 18 | `switch` on `long`/`boolean` | Not allowed |
| 19 | `if (x > 5);` | The semicolon is the body |
| 20 | Reassigning a parameter inside a method | Caller's variable unchanged (pass-by-value) |
| 21 | Overloading by return type only | Compile error |
| 22 | `arr.length()` | Error: it's `arr.length` (field) |
| 23 | `System.out.println(arr)` | Prints `[I@hash`; use `Arrays.toString` |
| 24 | `arr1.equals(arr2)` | Compares references; use `Arrays.equals` |
| 25 | `nextInt()` then `nextLine()` | `nextLine()` returns empty |
| 26 | `substring(1, 3)` | End index is **exclusive** |
| 27 | `-7 % 3` | `-1` (sign follows the left operand) |
| 28 | for-each variable modification | Doesn't change the array |

---

## 14. Practice Section

**How to score yourself:**
- Section A (Concepts): 1 point each
- Section B (Predict the Output): 1 point each
- Section C (Will it Compile?): 1 point each
- Section D (Coding): 3 points each (working + clean + edge cases)

| Score % | Meaning |
|---------|---------|
| 90–100% | Stage 1 mastered. Move to Stage 2 |
| 70–89% | Solid. Revisit the sections where you slipped |
| 50–69% | Re-read sections 4, 5, 7, 10, then retry |
| < 50% | Go through the guide again slowly, typing every example |

> ✋ **Write your answer down before opening each answer block.** Guessing then peeking doesn't build skill.

---

### Section A: Concept Questions

**A1.** What is the difference between JDK, JRE, and JVM? Which one is platform-dependent?

<details><summary>Answer</summary>

JVM executes bytecode. JRE = JVM + core libraries (for running). JDK = JRE + development tools like `javac` (for developing). The **JVM** is platform-dependent; bytecode is platform-independent. That combination gives "write once, run anywhere."
</details>

**A2.** Is Java compiled or interpreted?

<details><summary>Answer</summary>

Both. `javac` compiles source code to bytecode. At runtime, the JVM interprets bytecode and the JIT compiler converts frequently used ("hot") code into native machine code for speed.
</details>

**A3.** Why is `main` declared `static`?

<details><summary>Answer</summary>

So the JVM can call it without creating an object of the class. At startup there are no objects yet.
</details>

**A4.** Why do local variables need initialization while fields get default values?

<details><summary>Answer</summary>

Fields are part of objects/classes, which the JVM zero-initializes when allocating them. Local variables live in stack frames, and the compiler enforces "definite assignment" to prevent bugs from using values you never set. Using an uninitialized local is a compile-time error.
</details>

**A5.** Explain stack vs heap in two lines each.

<details><summary>Answer</summary>

**Stack:** per-thread memory holding method frames, local primitives, and references. It's automatically freed when a method returns; overflowing it causes `StackOverflowError`.
**Heap:** shared memory where all objects and arrays live. Unreferenced objects are cleaned up by the garbage collector; running out causes `OutOfMemoryError`.
</details>

**A6.** Give 4 reasons why `String` is immutable.

<details><summary>Answer</summary>

1. Enables the String Pool (safe sharing). 2. Security (paths, URLs, and credentials can't be altered after checks). 3. Thread-safety without synchronization. 4. Hash code can be cached, making Strings efficient HashMap keys.
</details>

**A7.** Is Java pass-by-value or pass-by-reference? Prove it.

<details><summary>Answer</summary>

**Always pass-by-value.** For objects, the value passed is a copy of the reference. Proof: a method that swaps two String/object parameters doesn't swap them for the caller, and reassigning a parameter (`arr = new int[3]`) doesn't affect the caller's variable. You *can* mutate the object through the copied reference, which is why it's often confused with pass-by-reference.
</details>

**A8.** What is the difference between `==` and `.equals()` for Strings?

<details><summary>Answer</summary>

`==` checks whether both references point to the **same object**. `.equals()` checks whether the **character contents** are the same. Always use `.equals()` to compare String content.
</details>

**A9.** When would you use `StringBuilder` over `String`? Over `StringBuffer`?

<details><summary>Answer</summary>

Use `StringBuilder` when building or modifying a string repeatedly (loops, concatenation), since String would create a new object every time. Prefer `StringBuilder` over `StringBuffer` in single-threaded code because it's faster (no synchronization). `StringBuffer` is for when multiple threads modify the same buffer (rare today).
</details>

**A10.** Can you overload a method by changing only its return type? Why or why not?

<details><summary>Answer</summary>

No. The return type isn't part of the method signature (name + parameter types). The compiler couldn't decide which method to call in a statement like `calc(5);` where the result isn't used.
</details>

**A11.** What are the rules for varargs?

<details><summary>Answer</summary>

Only one varargs parameter per method, and it must be the last parameter. Inside the method it behaves as an array. It has the lowest priority in overload resolution (after exact match, widening, and boxing).
</details>

**A12.** Which data types are allowed in a `switch`?

<details><summary>Answer</summary>

`byte`, `short`, `char`, `int`, their wrappers (`Byte`, `Short`, `Character`, `Integer`), `String`, and `enum`. Not `long`, `float`, `double`, or `boolean`. (Newer Java versions add pattern matching for objects in switch, which is a later topic.)
</details>

**A13.** What's the difference between `&&` and `&` when used on booleans?

<details><summary>Answer</summary>

`&&` short-circuits: if the left side is false, the right side isn't evaluated. `&` always evaluates both sides. This matters when the right side has side effects (`++x`) or could throw (`obj.method()` on null).
</details>

**A14.** What happens when you don't write a base case in recursion?

<details><summary>Answer</summary>

The method calls itself forever, each call adds a stack frame, the stack fills up, and the JVM throws `StackOverflowError`.
</details>

**A15.** How many objects can `new String("Hello")` create?

<details><summary>Answer</summary>

Up to **2**: one `"Hello"` literal in the String Pool (only if it isn't already there) and one new String object in the regular heap. If the literal already exists in the pool, just 1 new object.
</details>

---

### Section B: Predict the Output

Write the exact output (or say "Exception: name") before checking.

**B1.**
```java
System.out.println(10 + 20 + "Java" + 10 + 20);
```
<details><summary>Answer</summary>

`30Java1020`. Left to right: 10+20=30 (numbers), then "30Java", and after that everything is concatenation.
</details>

**B2.**
```java
System.out.println('A' + 'B');
System.out.println("" + 'A' + 'B');
System.out.println('A' + "B");
```
<details><summary>Answer</summary>

```
131
AB
AB
```
char + char is int addition (65 + 66). Once a String is involved, it's concatenation.
</details>

**B3.**
```java
int x = 10;
int y = x++ + ++x;
System.out.println(x + " " + y);
```
<details><summary>Answer</summary>

`12 22`. x++ gives 10 (x becomes 11), ++x makes x 12 and gives 12, so y = 10 + 12 = 22.
</details>

**B4.**
```java
int i = 5;
i = i++;
System.out.println(i);
```
<details><summary>Answer</summary>

`5`. `i++` evaluates to 5 and increments i to 6, then the assignment sets i back to 5.
</details>

**B5.**
```java
byte b = (byte) 130;
int i = (int) -3.99;
System.out.println(b + " " + i);
```
<details><summary>Answer</summary>

`-126 -3`. 130 overflows the byte range (130 − 256 = −126). Casting a double to int truncates toward zero.
</details>

**B6.**
```java
System.out.println(5 / 2 * 2.0);
System.out.println(5 / 2.0 * 2);
```
<details><summary>Answer</summary>

```
4.0
5.0
```
First: 5/2 = 2 (int division), then 2 × 2.0 = 4.0. Second: 5/2.0 = 2.5, then × 2 = 5.0.
</details>

**B7.**
```java
System.out.println(10 % 3 + -10 % 3);
System.out.println(-7 / 2);
```
<details><summary>Answer</summary>

```
0
-3
```
10 % 3 = 1, -10 % 3 = -1, sum 0. Integer division truncates toward zero: -3.5 becomes -3.
</details>

**B8.**
```java
long result = 1_000_000 * 1_000_000;
System.out.println(result > 0);
```
<details><summary>Answer</summary>

`false`. The multiplication happens in `int` and overflows (to -727379968) **before** being assigned to the long. Fix: `1_000_000L * 1_000_000`.
</details>

**B9.**
```java
System.out.println(1.0 / 0);
System.out.println(0.0 / 0);
System.out.println(1 / 0);
```
<details><summary>Answer</summary>

```
Infinity
NaN
Exception in thread "main" java.lang.ArithmeticException: / by zero
```
</details>

**B10.**
```java
String s1 = "Java";
String s2 = "Ja";
String s3 = s2 + "va";
final String s4 = "Ja";
String s5 = s4 + "va";
System.out.println(s1 == s3);
System.out.println(s1 == s5);
System.out.println(s1.equals(s3));
System.out.println(s1 == s3.intern());
```
<details><summary>Answer</summary>

```
false
true
true
true
```
s3 is built at runtime (s2 isn't final), so it's a new object. s5 uses a `final` constant, so it's joined at compile time and pooled. `equals` compares content. `intern()` returns the pooled instance.
</details>

**B11.**
```java
String s = "hello";
s.toUpperCase();
s.concat(" world");
System.out.println(s);
```
<details><summary>Answer</summary>

`hello`. Strings are immutable. Both method results were discarded because they weren't assigned.
</details>

**B12.**
```java
String a = "Hello World";
System.out.println(a.substring(6));
System.out.println(a.substring(1, 3));
System.out.println(a.indexOf('o') + " " + a.lastIndexOf('o'));
System.out.println("  hi  ".trim().length());
```
<details><summary>Answer</summary>

```
World
el
4 7
2
```
</details>

**B13.**
```java
int day = 2;
switch (day) {
    case 1: System.out.print("Mon ");
    case 2: System.out.print("Tue ");
    case 3: System.out.print("Wed ");
            break;
    default: System.out.print("Other");
}
```
<details><summary>Answer</summary>

`Tue Wed `. Matches case 2, falls through to case 3 (no break), then hits the break.
</details>

**B14.**
```java
int x = 5;
switch (x) {
    default: System.out.print("D ");
    case 1: System.out.print("One ");
    case 2: System.out.print("Two ");
}
```
<details><summary>Answer</summary>

`D One Two `. No case matches 5, so execution jumps to `default` (wherever it is) and then falls through every case below it, since there are no breaks.
</details>

**B15.**
```java
int count = 0;
for (int i = 0; i < 5; i++) {
    if (i == 3) continue;
    count += i;
}
System.out.println(count);
```
<details><summary>Answer</summary>

`7`. 0 + 1 + 2 + 4 (3 is skipped).
</details>

**B16.**
```java
outer:
for (int i = 1; i <= 3; i++) {
    for (int j = 1; j <= 3; j++) {
        if (j == 2) continue outer;
        if (i == 3) break outer;
        System.out.print(i + "" + j + " ");
    }
}
```
<details><summary>Answer</summary>

`11 21 `. For i=1: j=1 prints "11", then j=2 jumps to the next i. For i=2: prints "21", then j=2 jumps again. For i=3, j=1: `i == 3`, so it breaks out of everything.
</details>

**B17.**
```java
int n = 10;
do {
    System.out.print(n + " ");
    n++;
} while (n < 5);
```
<details><summary>Answer</summary>

`10 `. A do-while runs the body once before checking the condition.
</details>

**B18.**
```java
int x = 5;
if (x > 10 && ++x > 0) { }
System.out.print(x + " ");
if (x > 10 & ++x > 0) { }
System.out.print(x);
```
<details><summary>Answer</summary>

`5 6`. `&&` short-circuits, so ++x is skipped. `&` evaluates both sides, so x is incremented.
</details>

**B19.**
```java
static void change(int[] arr) {
    arr[0] = 99;
    arr = new int[]{1, 2, 3};
    arr[0] = 50;
}
// in main:
int[] nums = {1, 2, 3};
change(nums);
System.out.println(nums[0]);
```
<details><summary>Answer</summary>

`99`. The first line modifies the caller's array via the copied reference. After reassignment, the method works on a different array.
</details>

**B20.**
```java
static void modify(StringBuilder sb, String s) {
    sb.append(" World");
    s = s + " World";
    sb = new StringBuilder("New");
}
// in main:
StringBuilder sb = new StringBuilder("Hello");
String s = "Hello";
modify(sb, s);
System.out.println(sb + " | " + s);
```
<details><summary>Answer</summary>

`Hello World | Hello`. `append` mutates the shared StringBuilder. The String concatenation creates a new object assigned only to the local copy. Reassigning `sb` afterwards doesn't affect the caller.
</details>

**B21.**
```java
static void show(int x)     { System.out.print("int "); }
static void show(long x)    { System.out.print("long "); }
static void show(double x)  { System.out.print("double "); }
// in main:
byte b = 5;
show(b);
show(5L);
show(5.0f);
show('A');
```
<details><summary>Answer</summary>

`int long double int `. byte widens to the nearest match (int). 5L is exact long. float widens to double. char widens to int.
</details>

**B22.**
```java
static void test(long x)    { System.out.print("long "); }
static void test(Integer x) { System.out.print("Integer "); }
static void test(int... x)  { System.out.print("varargs "); }
// in main:
test(10);
test();
```
<details><summary>Answer</summary>

`long varargs `. Widening (int → long) beats boxing (int → Integer). With no arguments, only varargs fits.
</details>

**B23.**
```java
int[] a = {1, 2, 3};
int[] b = a;
int[] c = a.clone();
b[0] = 100;
c[1] = 200;
System.out.println(a[0] + " " + a[1]);
System.out.println(a == b);
System.out.println(a == c);
System.out.println(java.util.Arrays.equals(a, new int[]{100, 2, 3}));
```
<details><summary>Answer</summary>

```
100 2
true
false
true
```
b is the same array as a. c is an independent copy. `Arrays.equals` compares content.
</details>

**B24.**
```java
int[] arr = {1, 2, 3};
for (int x : arr) {
    x = x * 10;
}
System.out.println(java.util.Arrays.toString(arr));
```
<details><summary>Answer</summary>

`[1, 2, 3]`. The for-each variable is a copy of each element, so changing it doesn't touch the array.
</details>

**B25.**
```java
int[][] jag = new int[3][];
System.out.println(jag.length);
System.out.println(jag[0]);
System.out.println(jag[0].length);
```
<details><summary>Answer</summary>

```
3
null
Exception in thread "main" java.lang.NullPointerException
```
Only the outer array is created; the inner rows are null until assigned.
</details>

**B26.**
```java
System.out.println(010 + 0x10 + 0b10);
```
<details><summary>Answer</summary>

`26`. Octal 010 = 8, hex 0x10 = 16, binary 0b10 = 2. 8 + 16 + 2 = 26.
</details>

**B27.**
```java
char c = 'a';
c++;
c += 2;
System.out.println(c);
System.out.println((int) c);
```
<details><summary>Answer</summary>

```
d
100
```
'a' (97) becomes 'b' (98), then 'd' (100). `++` and `+=` include an implicit cast back to char.
</details>

**B28.**
```java
int a = 5, b = 10;
String r = a > b ? "A" : a == b ? "Equal" : "B";
System.out.println(r);
System.out.println(Math.round(-2.5) + " " + Math.round(2.5));
```
<details><summary>Answer</summary>

```
B
-2 3
```
`Math.round` rounds .5 toward positive infinity.
</details>

---

### Section C: Will It Compile?

Answer **Yes** or **No**. If No, say why.

**C1.** `long big = 3000000000;`
<details><summary>Answer</summary>No. The int literal is out of range; it needs `3000000000L`.</details>

**C2.** `float f = 3.14;`
<details><summary>Answer</summary>No. 3.14 is a double; it needs `3.14f` or a `(float)` cast.</details>

**C3.** `char c = 65;`
<details><summary>Answer</summary>Yes. A constant int within char range is allowed; c = 'A'.</details>

**C4.** `boolean flag = 1;`
<details><summary>Answer</summary>No. Java booleans are only `true`/`false`; there's no int conversion.</details>

**C5.**
```java
byte b = 10;
b = b * 2;
```
<details><summary>Answer</summary>No. `b * 2` is an int. Use `b *= 2;` or `b = (byte)(b * 2);`</details>

**C6.**
```java
byte b = 10;
b *= 2;
```
<details><summary>Answer</summary>Yes. Compound assignment includes an implicit cast.</details>

**C7.**
```java
int x;
if (Math.random() > 0.5) x = 1;
System.out.println(x);
```
<details><summary>Answer</summary>No. x might not be initialized on every path.</details>

**C8.**
```java
long val = 5;
switch (val) { case 5: System.out.println("five"); }
```
<details><summary>Answer</summary>No. `long` isn't allowed in switch.</details>

**C9.**
```java
int[] arr;
arr = {1, 2, 3};
```
<details><summary>Answer</summary>No. The shorthand initializer only works at declaration. Use `arr = new int[]{1, 2, 3};`</details>

**C10.**
```java
static int calc(int a) { return a; }
static double calc(int a) { return a; }
```
<details><summary>Answer</summary>No. Same signature; the return type alone doesn't count for overloading.</details>

**C11.** `static void log(String... msgs, int level) { }`
<details><summary>Answer</summary>No. Varargs must be the last parameter.</details>

**C12.**
```java
for (int i = 0; i < 3; i++) { }
System.out.println(i);
```
<details><summary>Answer</summary>No. `i` is out of scope after the loop.</details>

**C13.**
```java
int x = 10;
int x = 20;
```
<details><summary>Answer</summary>No. A variable can't be redeclared in the same scope.</details>

**C14.**
```java
final int MAX = 10;
MAX = 20;
```
<details><summary>Answer</summary>No. You can't reassign a final variable.</details>

**C15.**
```java
int n = 10;
n += 2.7;
System.out.println(n);
```
<details><summary>Answer</summary>Yes. It prints 12 (compound assignment casts the result to int).</details>

---

### Section D: Coding Exercises

Write each as a complete method or program. Test with normal input **and** edge cases (0, negative, empty, a single element).

**D1. Swap two integers without a third variable.**
<details><summary>Solution</summary>

```java
int a = 5, b = 9;
a = a + b;   // 14
b = a - b;   // 5
a = a - b;   // 9
System.out.println(a + " " + b);   // 9 5

// Alternative with XOR (no overflow risk):
a = a ^ b;
b = a ^ b;
a = a ^ b;
```
⚠️ The addition approach can overflow for very large values; XOR doesn't.
</details>

**D2. Check whether a number is prime.**
<details><summary>Solution</summary>

```java
static boolean isPrime(int n) {
    if (n < 2) return false;           // 0, 1, negatives are not prime
    if (n == 2) return true;
    if (n % 2 == 0) return false;
    for (int i = 3; (long) i * i <= n; i += 2) {   // only up to √n, odd numbers only
        if (n % i == 0) return false;
    }
    return true;
}
```
**Why √n?** If n = a × b, one of a or b must be ≤ √n. Checking only odd numbers halves the work. The `(long)` cast avoids overflow of `i * i` near `Integer.MAX_VALUE`.
</details>

**D3. Reverse a string: (a) using StringBuilder, (b) manually with a loop.**
<details><summary>Solution</summary>

```java
// (a)
static String reverseA(String s) {
    return new StringBuilder(s).reverse().toString();
}

// (b) two-pointer swap on a char array
static String reverseB(String s) {
    char[] chars = s.toCharArray();
    int left = 0, right = chars.length - 1;
    while (left < right) {
        char temp = chars[left];
        chars[left] = chars[right];
        chars[right] = temp;
        left++;
        right--;
    }
    return new String(chars);
}
```
Avoid `result = result + s.charAt(i)` in a loop, since it creates a new String every time.
</details>

**D4. Check whether a string is a palindrome, ignoring case and non-letters.** (`"A man, a plan, a canal: Panama"` → true)
<details><summary>Solution</summary>

```java
static boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        char l = s.charAt(left), r = s.charAt(right);
        if (!Character.isLetterOrDigit(l)) { left++; continue; }
        if (!Character.isLetterOrDigit(r)) { right--; continue; }
        if (Character.toLowerCase(l) != Character.toLowerCase(r)) return false;
        left++;
        right--;
    }
    return true;
}
```
</details>

**D5. Count vowels and consonants in a string.**
<details><summary>Solution</summary>

```java
static void countVowels(String s) {
    int vowels = 0, consonants = 0;
    for (char ch : s.toLowerCase().toCharArray()) {
        if (ch >= 'a' && ch <= 'z') {
            if ("aeiou".indexOf(ch) != -1) vowels++;
            else consonants++;
        }
    }
    System.out.println("Vowels: " + vowels + ", Consonants: " + consonants);
}
```
</details>

**D6. Print the first n Fibonacci numbers** (0 1 1 2 3 5 8 ...) **iteratively, then write a recursive `fib(n)`.**
<details><summary>Solution</summary>

```java
static void printFib(int n) {
    long a = 0, b = 1;
    for (int i = 0; i < n; i++) {
        System.out.print(a + " ");
        long next = a + b;
        a = b;
        b = next;
    }
}

static long fib(int n) {          // recursive
    if (n <= 1) return n;         // base cases: fib(0)=0, fib(1)=1
    return fib(n - 1) + fib(n - 2);
}
```
⚠️ The recursive version is exponentially slow (it recomputes the same values). Try `fib(45)` and feel it. You'll fix this with memoization later.
</details>

**D7. Factorial of n using recursion. What's the largest n that fits in `int`? In `long`?**
<details><summary>Solution</summary>

```java
static long factorial(int n) {
    if (n < 0) throw new IllegalArgumentException("n must be >= 0");
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}
```
`int` holds up to **12!** (479,001,600); 13! overflows. `long` holds up to **20!**; 21! overflows. For bigger values use `BigInteger` (Stage 3).
</details>

**D8. Find the largest and second-largest element in an array in a single pass.**
<details><summary>Solution</summary>

```java
static void topTwo(int[] arr) {
    if (arr == null || arr.length < 2) {
        System.out.println("Need at least 2 elements");
        return;
    }
    int first = Integer.MIN_VALUE, second = Integer.MIN_VALUE;
    for (int num : arr) {
        if (num > first) {
            second = first;
            first = num;
        } else if (num > second && num != first) {
            second = num;
        }
    }
    if (second == Integer.MIN_VALUE) System.out.println("No distinct second largest");
    else System.out.println("Largest: " + first + ", Second: " + second);
}
```
Edge cases: duplicates like `{5, 5, 5}` and negative numbers. (Strictly, an array that genuinely contains `Integer.MIN_VALUE` as its second largest would be misreported. A `boolean found` flag fixes that, which makes a good extension.)
</details>

**D9. Reverse an array in place** (no second array).
<details><summary>Solution</summary>

```java
static void reverse(int[] arr) {
    for (int i = 0, j = arr.length - 1; i < j; i++, j--) {
        int temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
}
```
Notice that the caller sees the change, because we mutate the array object through the copied reference.
</details>

**D10. Count the frequency of each character in a lowercase string, using only arrays.** (`"banana"` → a=3 b=1 n=2)
<details><summary>Solution</summary>

```java
static void charFrequency(String s) {
    int[] freq = new int[26];
    for (char ch : s.toCharArray()) {
        freq[ch - 'a']++;            // 'a'→0, 'b'→1, ... uses char arithmetic
    }
    for (int i = 0; i < 26; i++) {
        if (freq[i] > 0) {
            System.out.print((char) ('a' + i) + "=" + freq[i] + " ");
        }
    }
}
```
The `ch - 'a'` trick maps characters to indexes and shows up all the time in interview problems.
</details>

**D11. Check whether a number is an Armstrong number** (153 = 1³ + 5³ + 3³).
<details><summary>Solution</summary>

```java
static boolean isArmstrong(int n) {
    if (n < 0) return false;
    int digits = String.valueOf(n).length();
    int temp = n, sum = 0;
    while (temp > 0) {
        int d = temp % 10;
        sum += (int) Math.pow(d, digits);
        temp /= 10;
    }
    return sum == n;
}
```
</details>

**D12. Print this pyramid for n = 4:**
```
   *
  ***
 *****
*******
```
<details><summary>Solution</summary>

```java
static void pyramid(int n) {
    for (int row = 1; row <= n; row++) {
        for (int s = 0; s < n - row; s++) System.out.print(" ");
        for (int st = 0; st < 2 * row - 1; st++) System.out.print("*");
        System.out.println();
    }
}
```
**How to think about patterns:** for each row, ask "how many spaces? how many stars?" and write the formula in terms of `row`. Row r has (n - r) spaces and (2r - 1) stars.
</details>

**D13. Transpose a matrix** (rows become columns). `{{1,2,3},{4,5,6}}` → `{{1,4},{2,5},{3,6}}`
<details><summary>Solution</summary>

```java
static int[][] transpose(int[][] m) {
    int rows = m.length, cols = m[0].length;
    int[][] t = new int[cols][rows];
    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            t[c][r] = m[r][c];
        }
    }
    return t;
}
// print: System.out.println(Arrays.deepToString(transpose(m)));
```
</details>

**D14. Remove duplicates from a sorted array in place and return the new length.** (`{1,1,2,3,3,3,4}` → 4, array starts with `1,2,3,4`)
<details><summary>Solution</summary>

```java
static int removeDuplicates(int[] arr) {
    if (arr.length == 0) return 0;
    int write = 1;                         // position for the next unique value
    for (int read = 1; read < arr.length; read++) {
        if (arr[read] != arr[read - 1]) {
            arr[write] = arr[read];
            write++;
        }
    }
    return write;
}
```
This is the classic **two-pointer** technique: one pointer reads, one writes.
</details>

**D15. Mini program: read n marks from the user with Scanner, then print the average (2 decimals), the highest, the lowest, and a grade (A ≥ 90, B ≥ 75, C ≥ 50, else F) using a switch on `avg / 10`.**
<details><summary>Solution</summary>

```java
import java.util.Scanner;

public class MarksReport {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.print("How many subjects? ");
        int n = sc.nextInt();
        if (n <= 0) {
            System.out.println("Need at least one subject.");
            return;
        }

        int[] marks = new int[n];
        int sum = 0, max = Integer.MIN_VALUE, min = Integer.MAX_VALUE;
        for (int i = 0; i < n; i++) {
            System.out.print("Marks for subject " + (i + 1) + ": ");
            marks[i] = sc.nextInt();
            sum += marks[i];
            max = Math.max(max, marks[i]);
            min = Math.min(min, marks[i]);
        }

        double avg = (double) sum / n;      // cast BEFORE dividing!
        String grade;
        switch ((int) avg / 10) {
            case 10: case 9: grade = "A"; break;
            case 8:
                grade = (avg >= 75) ? "B" : "C"; break;   // 80–89 → B
            case 7:
                grade = (avg >= 75) ? "B" : "C"; break;   // 75–79 → B, 70–74 → C
            case 6: case 5: grade = "C"; break;
            default: grade = "F";
        }

        System.out.printf("Average: %.2f%nHighest: %d%nLowest: %d%nGrade: %s%n",
                          avg, max, min, grade);
        sc.close();
    }
}
```
Things this exercises: Scanner, arrays, casting before division, Math.max/min, switch with grouped cases, the ternary, and printf. (Note how B's 75 cutoff doesn't align with tens, which is why `if-else` is often cleaner than switch for ranges. Knowing *when not* to use switch is part of the skill.)
</details>

---

### Section E: Interview-Style Quick Fire

Answer in one or two sentences, out loud, as if in an interview.

1. Why is Java called platform-independent?
2. What is bytecode?
3. What does the JIT compiler do?
4. What is the size of `char` in Java, and why isn't it 1 byte like in C?
5. Why is `0.1 + 0.2 != 0.3`, and how do you handle money in Java?
6. What happens on integer overflow in Java?
7. Where is the String Pool located?
8. What does `intern()` do?
9. Why does `System.out.println(arr)` print something like `[I@1b6d3586`?
10. Difference between `arr.length`, `str.length()`, and `list.size()`?
11. What is the difference between `break` and `continue`?
12. Can a `switch` work without `default`? Can `default` be first?
13. What is method overloading, and what is the resolution order?
14. What is a `StackOverflowError` vs an `OutOfMemoryError`?
15. Why does `nextLine()` sometimes get skipped after `nextInt()`?

<details><summary>Model answers</summary>

1. Code compiles to bytecode, which any OS can run as long as it has a JVM.
2. A platform-neutral instruction set produced by `javac`, stored in `.class` files and executed by the JVM.
3. At runtime it compiles frequently executed bytecode into native machine code, so hot code runs at near-native speed.
4. 16 bits (2 bytes), because Java chars use UTF-16 Unicode and can represent characters from many languages, not just ASCII.
5. Binary floating point can't represent 0.1 exactly, so tiny rounding errors appear. Use `BigDecimal` or store the smallest unit (paise/cents) as a `long`.
6. It wraps around silently (MAX + 1 becomes MIN) with no exception. `Math.addExact` and friends throw if you want detection.
7. In the heap (since Java 7; earlier it was in PermGen).
8. It returns the canonical String from the pool, adding it if it isn't there, so equal strings can be compared with `==`.
9. Arrays inherit `Object.toString()`, which prints the type code (`[I` = int array) plus `@` plus a hash. Use `Arrays.toString()`.
10. `length` is a field on arrays, `length()` is a method on String, and `size()` is a method on collections.
11. `break` exits the loop (or switch) entirely; `continue` skips to the next iteration.
12. Yes, `default` is optional, and it can appear anywhere. It runs only if no case matches, and fall-through still applies from wherever it sits.
13. Same method name with different parameter lists. Resolution order: exact match → widening → boxing → varargs.
14. `StackOverflowError`: too-deep call chains, usually infinite recursion. `OutOfMemoryError`: the heap is full and the GC can't free enough space.
15. `nextInt()` leaves the newline in the buffer and `nextLine()` consumes it immediately. Add an extra `nextLine()` or parse lines with `Integer.parseInt(sc.nextLine())`.
</details>

---

## ✅ Stage 1 Completion Checklist

Tick each one only if you can **explain it to a friend without looking**:

- [ ] JDK vs JRE vs JVM, and the compile → bytecode → run flow
- [ ] All 8 primitives, their sizes, and the default literal types (int, double)
- [ ] Why locals need initialization but fields don't
- [ ] Stack vs heap, and what a reference really is
- [ ] Widening vs narrowing, type promotion to int, compound-assignment casting
- [ ] Integer division, overflow, floating-point imprecision
- [ ] Pre vs post increment, including `i = i++`
- [ ] Short-circuit `&&` / `||` vs `&` / `|`
- [ ] String immutability, the String Pool, `==` vs `equals`, compile-time concatenation
- [ ] String vs StringBuilder vs StringBuffer
- [ ] The Scanner `nextInt` / `nextLine` trap
- [ ] switch fall-through and allowed types
- [ ] Choosing between for / while / do-while / for-each, plus labeled break
- [ ] **Pass-by-value**, including the "modify vs reassign" difference
- [ ] Overloading rules and resolution order; varargs rules
- [ ] Recursion with a base case
- [ ] Arrays: defaults, `length`, copying vs referencing, `Arrays` utilities, 2D and jagged arrays

**All ticked and scoring 90%+ in practice?** You're ready for **Stage 2: OOP in depth**. 🚀