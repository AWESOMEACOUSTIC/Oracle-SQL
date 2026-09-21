# Java Competitive Exam Preparation
## Chunk 1: Keywords, Primitive Data Types, Variables, Literals, Casting, Operators, and Control Statements

This document is the first chunk of a continuous Java preparation course. Every topic below is built from fundamentals upward, written for a highly competitive exam where conceptual depth, output tracing, debugging skill, and the ability to handle unfamiliar variations of familiar ideas all matter. Read each concept fully before attempting its quiz, because later questions frequently combine ideas from earlier subsections.

**How this document is organized.** Each major topic contains Concept Explanation, Exam Perspective, Examples, Important Notes, Scenario Based Understanding, a Quiz of 12 to 15 questions with reasoned answers, and Programming Practice with full solutions. Each "Lend A Hand" section is a focused, hands on reinforcement block tied to the concept immediately before it. It assumes you already understand the concept, so it spends most of its space on extra examples, a quiz, and programming problems rather than re explaining theory. A single consolidated Revision Section closes the whole chunk.

---

## Table of Contents

1. Java Keywords
2. Primitive Data Types
3. Variables and Literals
4. Lend A Hand on Variables
5. Casting Primitives
6. Arithmetic, Unary and Relational Operators
7. Lend A Hand on Arithmetic, Unary and Relational Operators
8. Logical and Bitwise Operators
9. Lend A Hand on Logical and Bitwise Operators
10. Shift and Assignment Operators
11. Operator Precedence
12. Types of Java Statements
13. Selection Statement: If Statements
14. Lend A Hand on If Else If
15. Selection Statement: Switch Statement
16. Iteration Statement: While Statement
17. Lend A Hand on While Statement
18. Iteration Statement: Do While Statement
19. Lend A Hand on Do While Statement
20. Iteration Statement: For Statement
21. Lend A Hand on For Statement
22. Return Statement with Lend A Hand
23. Final Revision Section

---

## 1. Java Keywords

### 1.1 Concept Explanation

A keyword is a word that has a predefined meaning inside the Java language itself. Because the compiler reserves these words for a specific grammatical purpose, you cannot use them as identifiers, which means you cannot use them as the name of a class, a method, a variable, a package, or any other user defined entity. This restriction exists because the compiler needs certain tokens to always mean the same syntactic thing wherever they appear in a program, and allowing a programmer to redefine `if` or `class` as a variable name would make parsing the language ambiguous.

Java currently defines fifty three reserved words if you count both true keywords and the two reserved literals `true` and `false`, along with the reserved word `null`. It is common in exam material to describe the count as fifty one keywords plus three reserved literal like words, so do not be surprised if different sources quote slightly different totals depending on whether they count `true`, `false`, and `null` as keywords or as separate reserved words. What matters for the exam is recognizing every individual word and knowing what category it belongs to, not memorizing a single official count.

**Categories of keywords.** Grouping keywords by purpose makes them far easier to remember than treating them as one flat list.

| Category | Keywords |
|---|---|
| Access modifiers | `public`, `private`, `protected` |
| Non access modifiers | `static`, `final`, `abstract`, `synchronized`, `transient`, `volatile`, `native`, `strictfp`, `default` |
| Class, interface and related | `class`, `interface`, `extends`, `implements`, `enum`, `package`, `import`, `this`, `super`, `new`, `instanceof` |
| Primitive data types | `byte`, `short`, `int`, `long`, `float`, `double`, `char`, `boolean` |
| Control flow: selection | `if`, `else`, `switch`, `case`, `default` |
| Control flow: iteration | `for`, `while`, `do` |
| Control flow: transfer of control | `break`, `continue`, `return`, `yield` |
| Exception handling | `try`, `catch`, `finally`, `throw`, `throws`, `assert` |
| Method and value related | `void`, `return` |
| Reserved literals | `true`, `false`, `null` |
| Reserved but currently unused | `goto`, `const` |
| Module related (Java 9+) | `module`, `requires`, `exports`, `opens`, `uses`, `provides`, `to`, `with`, `transitive` (these are restricted keywords, valid only inside module declarations) |
| Newer context sensitive keywords | `var` (Java 10+), `record` (Java 16+), `sealed`, `permits`, `non-sealed` (Java 17+), `yield` (Java 14+) |

**Why `goto` and `const` matter for the exam.** Both words are reserved by the Java language specification even though neither can actually be used. They exist as reserved words so that no future version of Java has to worry about a programmer already having used them as an identifier. A very common exam trap is a program that tries to declare a variable named `goto` or `const`, which will fail to compile with an error about the token being unexpected or reserved, even though neither word does anything.

**Context sensitive keywords versus true reserved keywords.** This is one of the most important distinctions for tricky exam questions. A true reserved keyword like `class` or `if` can never be used as an identifier anywhere in the program. A context sensitive keyword such as `var`, `record`, `yield`, `sealed`, `permits`, and `non-sealed` only has special meaning in specific grammatical positions. Outside those positions, the word is a perfectly legal identifier. For example `var` can still be used as the name of a variable, a method, or even a class in modern Java, because `var` is only treated specially when it appears in local variable type inference position. The same is true for `yield`, which is only special inside a switch expression.

**`true`, `false`, and `null` are not keywords in the strictest technical sense.** The Java Language Specification actually classifies `true`, `false`, and `null` as reserved literals rather than keywords, because they represent literal values of boolean and reference type respectively rather than performing a syntactic role like `if` or `class`. For exam purposes, however, they behave exactly like keywords in that you cannot use any of these three words as an identifier. Many exams will still list them alongside keywords, and a question that asks you to count the total number of keywords is testing whether you know this subtlety.

**`var` is not a type.** A frequently misunderstood point is that `var` is not itself a data type the way `int` or `String` is. It is an instruction to the compiler to infer the actual type from the initializer expression at compile time. The compiler substitutes the real inferred type in the bytecode, so there is zero runtime cost and zero flexibility to change the variable's type later. `var` can only be used for local variables with an initializer; it cannot be used for fields, method parameters, or return types, and it cannot be used without an initializer because the compiler would have nothing to infer from.

### 1.2 Exam Perspective

1. **Direct conceptual questions** ask you to identify which of a given list of words is or is not a Java keyword. Expect distractors that are keywords in other languages, such as `friend` from C++, `def` from Python, or `function` from JavaScript, none of which exist in Java.
2. **Output based and compilation based questions** present a snippet that tries to use a reserved word as an identifier and ask whether it compiles. The answer is almost always that it fails to compile with a syntax error.
3. **Error identification questions** show code using `goto` as a label like C, or using `const` to declare a constant like C++, and ask you to spot why it fails. Remember that Java uses `final` for constants, not `const`.
4. **Tricky or misleading questions** exploit the context sensitive keywords. A question might use `var`, `yield`, `record`, or `sealed` as a class or variable name and ask whether it compiles. Since these are contextual, the correct answer is often that it does compile, which surprises candidates who assume every listed keyword is fully reserved.
5. **Questions testing true understanding** ask you to distinguish `true`, `false`, and `null` from `int`, `class`, and `if` in terms of their grammatical category, since the former are reserved literals representing values and the latter are keywords performing syntactic roles.
6. **Version awareness traps** test whether you know that `var`, `record`, `sealed`, `permits`, `yield`, and `non-sealed` were introduced in specific Java versions and did not exist in early Java. A question might claim a keyword existed since Java 1.0 when it is actually a modern addition.
7. **Case sensitivity traps** present a word like `If`, `Class`, or `Void` with different capitalization and ask if it is a keyword or a valid identifier. Java is case sensitive, so any keyword with altered capitalization is simply a normal identifier, not a keyword.

### 1.3 Examples

**Example 1: Reserved word as identifier fails to compile.**

```java
public class Demo {
    public static void main(String[] args) {
        int class = 5; // compile time error
        System.out.println(class);
    }
}
```

Expected result: compilation error, because `class` is a true reserved keyword and cannot be used as a variable name.

**Example 2: Context sensitive keyword used as identifier compiles fine.**

```java
public class Demo {
    public static void main(String[] args) {
        int var = 10;
        int record = 20;
        System.out.println(var + record);
    }
}
```

Output: `30`

Explanation: `var` and `record` are context sensitive keywords. Outside of local variable type inference position and record declarations respectively, they behave as ordinary identifiers, so this compiles and runs correctly.

**Example 3: `var` inferring a type.**

```java
public class Demo {
    public static void main(String[] args) {
        var message = "Hello";
        System.out.println(message.getClass().getSimpleName());
    }
}
```

Output: `String`

Explanation: the compiler inspects the initializer `"Hello"` and infers the type `String` for `message` at compile time. There is no dynamic typing involved; the variable is exactly as strongly typed as if you had written `String message = "Hello";`.

**Example 4: Case sensitivity of keywords.**

```java
public class Demo {
    public static void main(String[] args) {
        int Int = 5; // legal, Int is not the same token as int
        boolean Boolean = true; // legal
        System.out.println(Int);
        System.out.println(Boolean);
    }
}
```

Output: `5` then `true`, both printed on separate lines. `Int` and `Boolean` are valid identifiers because Java keywords are entirely lowercase and the language is case sensitive.

### 1.4 Important Notes

- Keywords are always written in lowercase; there is no keyword in Java that contains an uppercase letter.
- `true`, `false`, and `null` are technically reserved literals, not keywords, but for all practical identifier naming purposes they behave identically to keywords.
- `goto` and `const` are reserved but not implemented; using either as an identifier is a compile time error even though neither performs any action.
- `var`, `yield`, `record`, `sealed`, `permits`, and `non-sealed` are context sensitive or restricted keywords, valid as identifiers outside their special grammatical positions.
- `var` cannot be used for fields, method parameters, method return types, or uninitialized local variables. It also cannot be assigned `null` directly without a target type, since there would be nothing to infer from.
- `strictfp`, `native`, `transient`, `volatile`, and `synchronized` are frequently forgotten keywords precisely because they are used less often; exams exploit this by hiding them among distractor answer choices.
- There is no keyword `string` in lowercase for the `String` class; `String` is a class from `java.lang`, not a primitive type, and therefore not a keyword at all.

### 1.5 Scenario Based Understanding

**Scenario A.** A question shows a program declaring `int yield = 100;` inside an ordinary method body, outside any switch expression, and asks whether it compiles.
- What is happening: the word `yield` is being used as a plain variable name.
- Which concept is involved: context sensitive keywords.
- How to identify it: recognize that `yield` only has special meaning inside a switch expression that produces a value; here it is just a method local variable declaration.
- Correct reasoning: since `yield` is not appearing in switch expression position, it is treated as an ordinary identifier, so this compiles successfully.
- Common mistake: assuming that because `yield` appears in keyword lists, it is fully reserved like `if` or `class`, and incorrectly concluding the code fails to compile.

**Scenario B.** A question presents `final int const = 5;` and asks candidates to identify the error.
- What is happening: the programmer is trying to use `const`, likely out of habit from C or C++, to declare a constant.
- Which concept is involved: reserved but unimplemented keywords.
- How to identify it: recall that Java has no `const` keyword in active use; constants are declared with `final`.
- Correct reasoning: the compiler rejects `const` as an identifier because it is a reserved word, producing a compile time error regardless of the `final` modifier placed correctly before it.
- Common mistake: assuming the error is caused by `final`, when the actual problem is the reserved word `const` being used as the variable name.

### 1.6 Quiz: Java Keywords

1. How many words does the Java Language Specification classify as reserved literals rather than keywords?
   A. 0  B. 1  C. 2  D. 3

2. Which of the following is a valid Java identifier?
   A. `class`  B. `Class`  C. `if`  D. `int`

3. What happens when the following code is compiled?
   ```java
   int goto = 10;
   ```
   A. Prints 10  B. Compile time error  C. Runtime exception  D. Prints 0

4. Which keyword was introduced specifically to support local variable type inference?
   A. `let`  B. `auto`  C. `var`  D. `infer`

5. Which of these is NOT a Java keyword?
   A. `strictfp`  B. `friend`  C. `transient`  D. `volatile`

6. Consider: `int record = 5; System.out.println(record);` outside of any record declaration. What is the output?
   A. Compile error  B. `5`  C. `record`  D. Runtime exception

7. Which statement about `true`, `false`, and `null` is correct?
   A. They are primitive data types  
   B. They are reserved literals, not keywords in the strict specification sense  
   C. They can be reassigned by the programmer  
   D. They are only valid inside `if` statements

8. Which of the following pairs are BOTH reserved but functionally unimplemented in Java?
   A. `var` and `record`  B. `goto` and `const`  C. `yield` and `sealed`  D. `enum` and `assert`

9. Is `Void` (capital V) a Java keyword?
   A. Yes, identical to `void`  B. No, Java is case sensitive so this is a normal identifier  C. Only inside generics  D. Only as a return type

10. Which keyword became reserved for module declarations starting in Java 9?
    A. `module`  B. `package`  C. `import`  D. `namespace`

11. What is the output of the following?
    ```java
    public class Demo {
        public static void main(String[] args) {
            var var = 5;
            System.out.println(var);
        }
    }
    ```
    A. Compile time error because `var` cannot name a variable named `var`  
    B. `5`  
    C. Runtime exception  
    D. `var`

12. Which of the following correctly declares a constant in Java?
    A. `const int x = 5;`  B. `final int x = 5;`  C. `static int x = 5;` only  D. `readonly int x = 5;`

13. Select all words below that are context sensitive keywords rather than fully reserved keywords (multiple correct answers).
    A. `var`  B. `class`  C. `yield`  D. `sealed`  E. `if`

14. Why can `var` not be used as a method parameter type?
    A. Because `var` is not permitted anywhere in Java  
    B. Because method parameters have no initializer expression for the compiler to infer a type from  
    C. Because method parameters must always be `final`  
    D. Because `var` is only a keyword inside loops

15. What best explains why `int If = 10;` compiles successfully?
    A. `If` is a synonym for `if`  
    B. Java ignores capitalization for keywords  
    C. Java is case sensitive, so `If` is a different token from the keyword `if` and is a valid identifier  
    D. `If` is treated as a String literal

### 1.7 Quiz Answers and Reasoning

1. **Answer: D, 3.** `true`, `false`, and `null` are the three reserved literals. They are grammatically distinct from keywords because they represent actual values rather than performing syntax, but the specification still reserves them so they cannot be reused as identifiers.

2. **Answer: B, `Class`.** Java is case sensitive. `class`, `if`, and `int` are all true reserved keywords in lowercase, so none of them can be identifiers. `Class` differs by capitalization and is therefore a completely distinct, legal identifier, coincidentally similar to the name of the `java.lang.Class` class but not the same token as any keyword.

3. **Answer: B, compile time error.** `goto` is a reserved word in Java even though the language never implemented a working `goto` statement. Using it as a variable name is rejected by the compiler before the program can even attempt to run, which rules out any runtime based outcome like A, C, or D.

4. **Answer: C, `var`.** Java introduced `var` in Java 10 specifically for local variable type inference. There is no `let`, `auto`, or `infer` keyword in Java; those exist in other languages such as JavaScript, C++, and Kotlin respectively, making them classic distractors.

5. **Answer: B, `friend`.** `strictfp`, `transient`, and `volatile` are all genuine, if lesser used, Java keywords. `friend` is a C++ keyword for granting access across classes and has no equivalent keyword in Java at all.

6. **Answer: B, `5`.** `record` only carries special meaning when declaring a record type, for example `record Point(int x, int y) {}`. Used here as a plain variable name outside that context, it is a normal identifier, so the code compiles and prints the value stored in it, which is `5`.

7. **Answer: B.** `true`, `false`, and `null` are reserved literals representing a boolean value, a boolean value, and the absence of a reference respectively. They are not primitive types themselves, not reassignable since they are literal constants rather than variables, and not restricted to appearing only inside `if` statements; `null` for instance can appear in any reference type context.

8. **Answer: B, `goto` and `const`.** Both are reserved by the specification to prevent future conflicts but neither has any functioning behavior in the language. `var`, `record`, `yield`, and `sealed` are all context sensitive keywords that do function, and `enum` and `assert` are fully active, implemented keywords.

9. **Answer: B.** Java keywords are exclusively lowercase tokens. `Void` with a capital V shares no special status with the keyword `void`; it is a valid identifier, and incidentally also happens to be the name of the unrelated wrapper class `java.lang.Void`, which can add to the confusion in this type of question.

10. **Answer: A, `module`.** Java 9 introduced the module system, which reserved several new restricted keywords including `module`, `requires`, `exports`, `opens`, `uses`, and `provides`, valid only inside `module-info.java` files. `package` and `import` are much older keywords unrelated to the Java 9 module system, and `namespace` is not a Java keyword at all, being borrowed from languages like C# and C++.

11. **Answer: B, `5`.** This is a deliberately tricky construction. The first `var` is the context sensitive keyword requesting type inference. The second `var` is simply the chosen variable name, which is legal because `var` is not fully reserved. The compiler infers the type of the variable named `var` as `int` from the literal `5`, and the program prints `5`.

12. **Answer: B.** Java has no `const` keyword in active use, ruling out A. `final` is the correct modifier for declaring that a variable's reference or value cannot be reassigned after initialization. `static` alone only affects whether a field belongs to the class rather than an instance and does not by itself prevent reassignment. `readonly` is a C# keyword, not a Java keyword.

13. **Answer: A, C, and D (`var`, `yield`, `sealed`).** `class` and `if` are true, fully reserved keywords with no context dependent exception; they can never be used as identifiers anywhere. `var`, `yield`, and `sealed` only carry special meaning in specific grammatical positions and are otherwise ordinary identifiers.

14. **Answer: B.** `var` relies entirely on compile time type inference from an initializer expression. A method parameter is supplied a value only when the method is called, not through an initializer expression written at the declaration site, so the compiler has no expression to infer a type from, which is why `var` is disallowed there by the language specification.

15. **Answer: C.** Java's case sensitivity means that every identifier and keyword is matched by exact character case. The keyword `if` is defined only as the exact lowercase sequence `i` then `f`. `If` with an uppercase `I` is a completely different token to the compiler and is treated as any other user defined identifier would be.

### 1.8 Programming Practice: Java Keywords

1. **Basic.** Write a program that declares a local variable using `var` to store your name as a String and another using `var` to store your age as an int, then prints both in a single formatted sentence.
2. **Intermediate.** Write a program that declares a variable literally named `record` holding an integer, a variable named `yield` holding a double, and a variable named `sealed` holding a boolean, and prints all three values, demonstrating that these words are valid identifiers outside their special contexts.
3. **Intermediate.** Write a short program using `var` to iterate over an array of integers with an enhanced for loop, printing the inferred type of the loop variable using `getClass()` on one boxed sample value, and the sum of all elements.
4. **Advanced.** Attempt, in comments only (do not submit code meant to compile), to write three separate one line declarations that each attempt to use a genuinely reserved keyword (not a context sensitive one) as an identifier. For each, state in a comment which specific keyword category it belongs to and why the compiler will reject it.
5. **Edge case based.** Write a program that declares two variables in the same method, one named `Var` and one named `var`, storing different int values, and prints their sum, demonstrating case sensitivity.
6. **Logic intensive.** Write a program that builds a `String[]` array containing a mixture of true Java keywords and context sensitive keywords such as `{"class", "var", "if", "yield", "static"}`, then for each entry prints whether it "can never be an identifier" or "can sometimes be an identifier" based on a hardcoded lookup you design yourself using a suitable collection.

### 1.9 Programming Solutions: Java Keywords

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        var name = "Aditi";
        var age = 21;
        System.out.println(name + " is " + age + " years old.");
    }
}
```

Thought process: the task only requires demonstrating `var` with two different inferred types, String and int, and combining them in output. There is no algorithmic complexity here; the exercise builds comfort with type inference syntax.

Why it works: the compiler infers `String` for `name` from the string literal and `int` for `age` from the integer literal, exactly as if those types had been written explicitly.

Time complexity: O(1). Space complexity: O(1). Edge cases: none meaningful at this level, though it is worth noting `var name;` without an initializer would fail to compile, which is why an initializer is mandatory here.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int record = 42;
        double yield = 3.75;
        boolean sealed = true;
        System.out.println("record = " + record);
        System.out.println("yield = " + yield);
        System.out.println("sealed = " + sealed);
    }
}
```

Thought process: the goal is purely to confirm that `record`, `yield`, and `sealed` compile as identifiers when used outside a record declaration, a switch expression, and a sealed class declaration respectively.

Why it works: none of these three words are true reserved keywords; they are only special in specific syntactic contexts that are absent here, so the compiler treats each as an ordinary variable name.

Common incorrect approach: assuming any of these declarations would fail to compile because the words appear in general keyword lists, which conflates context sensitive keywords with fully reserved ones.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int[] numbers = {4, 8, 15, 16, 23, 42};
        var sum = 0;
        for (var n : numbers) {
            sum += n;
        }
        Integer sample = numbers[0];
        System.out.println("Inferred element type sample class: " + sample.getClass().getSimpleName());
        System.out.println("Sum: " + sum);
    }
}
```

Thought process: two separate `var` usages appear here, one for the accumulator `sum` and one for the enhanced for loop variable `n`. Both are inferred as `int` because `numbers` is an `int[]`.

Why it works: `var` in an enhanced for loop infers its type from the element type of the iterable or array being traversed. Autoboxing lets us store one `int` element into an `Integer` reference purely to demonstrate the underlying type via `getClass()`, since primitives themselves have no `getClass()` method.

Time complexity: O(n) for the loop over the array. Space complexity: O(1) beyond the input array. Edge case worth noting: if `numbers` were empty, `sum` would correctly remain `0` and the loop body would simply never execute, though the `numbers[0]` access for the sample would then throw an `ArrayIndexOutOfBoundsException`, so a production version would guard against an empty array first.

**Solution 4 (conceptual, not compiled).**

```java
// Attempt 1: int class = 10;
// Category: class and interface related keyword.
// Rejected because "class" always introduces a class declaration; the compiler
// cannot parse it as a type name followed by an identifier in this position.

// Attempt 2: boolean if = true;
// Category: selection control flow keyword.
// Rejected because "if" always begins a conditional statement; the parser
// expects a parenthesized condition immediately after it, not an identifier role.

// Attempt 3: double static = 1.5;
// Category: non access modifier keyword.
// Rejected because "static" is only valid as a modifier preceding a member
// declaration, never as the name being declared.
```

Thought process: the exercise is about correctly classifying each keyword rather than producing runnable code, since none of these attempts can compile by definition.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        int Var = 10;
        int var = 20;
        System.out.println(Var + var);
    }
}
```

Output: `30`

Why it works: `Var` and `var` are two distinct tokens under Java's case sensitive rules, even though `var` also happens to be a context sensitive keyword. Since `var` is being used as a plain identifier here rather than in type inference position, both declarations are legal simultaneously.

**Solution 6.**

```java
import java.util.HashSet;
import java.util.Set;

public class Solution6 {
    public static void main(String[] args) {
        Set<String> contextSensitive = new HashSet<>();
        contextSensitive.add("var");
        contextSensitive.add("yield");
        contextSensitive.add("record");
        contextSensitive.add("sealed");
        contextSensitive.add("permits");
        contextSensitive.add("non-sealed");

        String[] words = {"class", "var", "if", "yield", "static"};

        for (String word : words) {
            if (contextSensitive.contains(word)) {
                System.out.println(word + " -> can sometimes be an identifier");
            } else {
                System.out.println(word + " -> can never be an identifier");
            }
        }
    }
}
```

Thought process: rather than hardcoding a long if else chain, a `HashSet` gives O(1) average time membership checks, which is the appropriate data structure for a fixed lookup set. The logic simply checks membership in the context sensitive set to decide which message to print.

Why it works: any word found in the `contextSensitive` set is context dependent and therefore sometimes usable as an identifier; any word not found is assumed to be a fully reserved keyword and therefore never usable as an identifier.

Time complexity: O(n) where n is the number of words checked, since each `HashSet` lookup is O(1) on average. Space complexity: O(k) where k is the size of the context sensitive set. Common incorrect approach: using a long chain of `if (word.equals("var") || word.equals("yield") ...)` statements, which works but does not scale well and is harder to maintain or extend compared to a set based lookup.


---

## 2. Primitive Data Types

### 2.1 Concept Explanation

Java is a statically typed language, which means every variable must have a declared type known at compile time, and that type determines what kind of value the variable can hold and how much memory it occupies. Java splits its type system into two broad families: primitive types and reference types. This section focuses entirely on primitives, which are the eight built in, non object types that Java provides directly at the language level rather than through a class.

The eight primitive types are `byte`, `short`, `int`, `long`, `float`, `double`, `char`, and `boolean`. Unlike reference types, a primitive variable directly stores its actual value in the memory location associated with that variable, whether that is a local variable slot on the stack or a field slot inside an object on the heap. There is no separate object header, no indirection through a reference, and no `null` state available for a primitive; every primitive variable always holds some concrete value of its type once it is initialized.

**Integer types.** `byte`, `short`, `int`, and `long` all store whole numbers and differ only in their storage size and therefore their representable range. All four are signed, meaning they use two's complement representation with the highest order bit reserved to indicate sign, so the positive range is always one less in magnitude than the negative range.

| Type | Size | Minimum Value | Maximum Value | Default Value |
|---|---|---|---|---|
| `byte` | 8 bits | -128 | 127 | 0 |
| `short` | 16 bits | -32768 | 32767 | 0 |
| `int` | 32 bits | -2147483648 | 2147483647 | 0 |
| `long` | 64 bits | -9223372036854775808 | 9223372036854775807 | 0L |

**Floating point types.** `float` and `double` store numbers with a fractional component using the IEEE 754 standard for binary floating point arithmetic. `float` is single precision at 32 bits, and `double` is double precision at 64 bits. Because computers store these values in binary fractions rather than exact decimal fractions, many decimal values that look simple, such as `0.1`, cannot be represented exactly, which leads to small rounding errors that are extremely common exam material.

| Type | Size | Approximate Range | Default Value | Precision |
|---|---|---|---|---|
| `float` | 32 bits | roughly ±3.4 × 10^38 | 0.0f | about 6 to 7 significant decimal digits |
| `double` | 64 bits | roughly ±1.7 × 10^308 | 0.0d | about 15 to 16 significant decimal digits |

**Character type.** `char` is a 16 bit unsigned type that represents a single UTF 16 code unit, with a range of values from 0 to 65535. Because it is unsigned, `char` has no negative values at all, which distinguishes it sharply from every integer primitive type. A `char` can be treated numerically in arithmetic, since under the hood it is simply an unsigned 16 bit integer holding a Unicode code point value, which is why characters can participate directly in arithmetic expressions.

**Boolean type.** `boolean` represents a truth value that is either `true` or `false`. Unlike C, where an integer can substitute for a boolean, Java strictly enforces that only the literals `true` and `false`, or expressions that evaluate to a boolean, can be assigned to or compared against a `boolean` variable. The exact bit size of `boolean` is intentionally left unspecified by the Java Virtual Machine specification, since it is implementation dependent, but this is a low level detail that rarely matters outside of extremely deep JVM internals questions.

**Default values apply only to fields, not local variables.** A critically important and heavily tested rule is that the default values shown above only apply to instance fields and static fields, which the JVM automatically zero initializes when an object or class is created. Local variables declared inside a method, constructor, or block receive no default value whatsoever, and the compiler requires that you assign a value to a local variable before it is read, or the code will fail to compile with a "variable might not have been initialized" error.

**Why `byte` and `short` exist despite `int` covering more ground.** These smaller types exist primarily to save memory in large arrays and to match external data formats such as binary file formats or network protocols that use fixed width smaller fields. In ordinary day to day code, `int` is the default and most natural choice for whole numbers.

### 2.2 Exam Perspective

1. **Direct conceptual questions** ask you to list the eight primitive types, or to state the size in bits or bytes of a specific type such as `long` or `char`.
2. **Output based questions** show a primitive being printed after an overflow occurs, testing whether you understand that Java silently wraps around on integer overflow rather than throwing an exception.
3. **Range boundary questions** ask what happens when a literal at or beyond the boundary of a type's range is assigned, such as assigning `128` to a `byte`.
4. **Default value trap questions** present a class with an uninitialized instance field that is printed, expecting you to know the JVM supplied default, versus a local variable in `main` that is used before assignment, expecting you to know this fails to compile.
5. **Debugging questions** show floating point comparisons such as `0.1 + 0.2 == 0.3` and ask why the result is `false`, testing understanding of binary floating point imprecision.
6. **char arithmetic questions** show expressions like `'a' + 1` and ask for the resulting type and value, testing whether you understand that char participates in arithmetic through implicit promotion to int.
7. **Concept comparison questions** ask you to distinguish `float` from `double` in terms of precision, default type of decimal literals, and the need for an `f` suffix.
8. **Tricky or misleading questions** present `boolean flag = 1;` expecting the incorrect assumption that Java allows integer to boolean coercion like C, when in fact this is a compile time error.
9. **Edge case questions** test the minimum negative value of `int` or `long`, particularly the surprising fact that `Math.abs(Integer.MIN_VALUE)` still returns a negative number due to how two's complement range is asymmetric.
10. **Multiple concept combination questions** combine primitive ranges with casting or with operator behavior, for example asking what a `byte` addition produces before any explicit cast is applied.

### 2.3 Examples

**Example 1: Integer overflow wraps around silently.**

```java
public class Demo {
    public static void main(String[] args) {
        int max = Integer.MAX_VALUE;
        System.out.println(max);
        System.out.println(max + 1);
    }
}
```

Output:
```
2147483647
-2147483648
```

Explanation: adding 1 to `Integer.MAX_VALUE` overflows the 32 bit signed representation and wraps around to `Integer.MIN_VALUE`. Java performs no automatic overflow checking or exception throwing for primitive arithmetic; the bits simply wrap according to two's complement rules.

**Example 2: Default values for fields versus local variables.**

```java
public class Demo {
    static int counter; // default 0

    public static void main(String[] args) {
        System.out.println(counter);
        int localValue;
        // System.out.println(localValue); // would not compile: variable might not have been initialized
        localValue = 5;
        System.out.println(localValue);
    }
}
```

Output:
```
0
5
```

Explanation: `counter` is a static field, so the JVM automatically initializes it to `0`. `localValue` is a local variable, so it has no default value at all; reading it before assignment is a compile time error, which is why the commented out line cannot be uncommented without breaking compilation.

**Example 3: Floating point imprecision.**

```java
public class Demo {
    public static void main(String[] args) {
        double a = 0.1;
        double b = 0.2;
        System.out.println(a + b);
        System.out.println(a + b == 0.3);
    }
}
```

Output:
```
0.30000000000000004
false
```

Explanation: `0.1` and `0.2` cannot be represented exactly in binary floating point, so their sum carries a tiny representational error. Comparing floating point values with `==` for exact equality is therefore unreliable, and this is one of the most heavily tested pitfalls in any Java exam covering primitives.

**Example 4: char arithmetic and implicit promotion.**

```java
public class Demo {
    public static void main(String[] args) {
        char c = 'a';
        System.out.println(c + 1);
        char next = (char) (c + 1);
        System.out.println(next);
    }
}
```

Output:
```
98
b
```

Explanation: when `c` participates in the expression `c + 1`, it is implicitly promoted to `int`, so `c + 1` produces an `int` result of `98`, not a `char`. To get back a `char` result, an explicit cast is required, which then correctly displays the character `b`, the Unicode character immediately following `a`.

**Example 5: `byte` and `short` require narrowing casts even for small literals stored via arithmetic.**

```java
public class Demo {
    public static void main(String[] args) {
        byte b1 = 10;
        byte b2 = 20;
        // byte sum = b1 + b2; // does not compile
        byte sum = (byte) (b1 + b2);
        System.out.println(sum);
    }
}
```

Output: `30`

Explanation: any arithmetic operation involving `byte` or `short` operands promotes both operands to `int` before performing the operation, so `b1 + b2` produces an `int` result even though both inputs were `byte`. Assigning that `int` result directly back to a `byte` variable requires an explicit narrowing cast, since the compiler cannot guarantee the result fits in a `byte` without one.

### 2.4 Important Notes

- The eight primitive types are `byte`, `short`, `int`, `long`, `float`, `double`, `char`, and `boolean`; there is no ninth primitive, and `String` is never a primitive.
- Integer literals are `int` by default; use an `L` or `l` suffix for `long` literals. Prefer uppercase `L`, since lowercase `l` is easily confused with the digit `1`.
- Decimal literals are `double` by default; use an `f` or `F` suffix for `float` literals, or the literal will not compile when assigned directly to a `float` variable without a cast.
- Integer arithmetic overflow wraps silently using two's complement; it never throws an exception at runtime.
- `char` is unsigned and ranges from 0 to 65535; it is the only primitive with no negative values.
- Any arithmetic performed on `byte`, `short`, or `char` operands is carried out after promoting both to `int`; the result type of such an expression is always at least `int`.
- Fields get automatic default values; local variables do not, and the compiler enforces definite assignment before use for locals.
- `Integer.MIN_VALUE` has no positive counterpart of equal magnitude representable as an `int`, so `-Integer.MIN_VALUE` and `Math.abs(Integer.MIN_VALUE)` both still evaluate to `Integer.MIN_VALUE` itself due to overflow.
- Floating point equality comparisons using `==` are unreliable due to representational imprecision; comparing with a small tolerance (an epsilon value) is the safer practical approach, though exams mostly test that you recognize the pitfall rather than requiring you to write epsilon comparisons.
- `boolean` cannot be interchanged with any integer type in Java; there is no implicit or explicit conversion between `boolean` and `int`.

### 2.5 Scenario Based Understanding

**Scenario A.** A question shows a class with an uninitialized `int` field and asks what a freshly created object prints for that field before any constructor logic sets it.
- What is happening: the field has received the automatic JVM default value.
- Which concept is involved: default values for instance fields.
- How to identify it: notice the field is declared at class level, not inside a method body.
- Correct reasoning: the answer is `0`, since `int` fields default to zero automatically at object creation time, before any constructor body executes.
- Common mistake: assuming the field is uninitialized in the sense of causing a compile error, confusing the rule for fields with the stricter rule for local variables.

**Scenario B.** A question performs repeated multiplication on an `int` accumulator inside a loop that runs enough iterations to exceed `Integer.MAX_VALUE`, then asks for the printed result.
- What is happening: the accumulator silently overflows partway through the loop.
- Which concept is involved: integer overflow and wraparound.
- How to identify it: check whether the theoretical mathematical product would exceed roughly 2.1 billion, the practical ceiling of a 32 bit signed integer.
- Correct reasoning: trace the wraparound behavior at the point of overflow, since the JVM never throws an exception here, and continue the multiplication using the wrapped, possibly negative, intermediate value.
- Common mistake: computing the true mathematical product without accounting for wraparound, producing an answer far larger than any value an `int` variable could actually hold.

**Scenario C.** A question compares two `double` values that were each computed through a chain of additions and subtractions of fractional decimal literals, then asks whether an `if (a == b)` branch executes.
- What is happening: the two doubles may differ by a tiny representational error despite being mathematically equal.
- Which concept is involved: floating point precision.
- How to identify it: notice the values involve non terminating binary fractions such as tenths, and that equality rather than a range comparison is being used.
- Correct reasoning: the safest answer, absent specific values that are known to be exactly representable such as whole numbers, is that the branch may not execute due to floating point imprecision, so `==` on doubles built from decimal arithmetic should be treated with suspicion.
- Common mistake: assuming ordinary decimal arithmetic rules apply and that mathematically equal expressions always compare equal with `==` in floating point.

### 2.6 Quiz: Primitive Data Types

1. What is the size, in bits, of a `long`?
   A. 32  B. 64  C. 128  D. 16

2. What is the default value of an uninitialized `boolean` instance field?
   A. `true`  B. `false`  C. `0`  D. `null`

3. What is the output of `System.out.println(Integer.MAX_VALUE + 1);`?
   A. A compile time error  B. `2147483648`  C. `-2147483648`  D. `0`

4. Which of the following literal declarations fails to compile without modification?
   A. `long x = 100;`  B. `float f = 3.5;`  C. `double d = 3.5;`  D. `int i = 100;`

5. What is the result type of the expression `'a' + 'b'`?
   A. `char`  B. `int`  C. `String`  D. Compile error

6. Given `byte b = 127; b++;`, what is the value of `b` afterward?
   A. `128`  B. `-128`  C. `0`  D. Compile error

7. Which statement about `char` is true?
   A. It can hold negative values  B. It is a signed 16 bit type  C. It is an unsigned 16 bit type  D. It is 8 bits

8. What happens when you compile `boolean b = 1;`?
   A. `b` becomes `true`  B. `b` becomes `false`  C. Compile time error  D. Runtime exception

9. What is printed by the following?
   ```java
   public class Demo {
       static double d;
       public static void main(String[] args) {
           System.out.println(d);
       }
   }
   ```
   A. Compile error  B. `0`  C. `0.0`  D. `null`

10. Which of these correctly declares a `float` literal?
    A. `float f = 2.5;`  B. `float f = 2.5f;`  C. `float f = 2.5d;`  D. `float f = (float) "2.5";`

11. What does `Math.abs(Integer.MIN_VALUE)` return?
    A. `2147483647`  B. `2147483648`  C. `Integer.MIN_VALUE`, i.e. still negative  D. `0`

12. Given `int x; System.out.println(x);` inside `main`, what happens?
    A. Prints `0`  B. Prints garbage value  C. Compile time error  D. Throws `NullPointerException`

13. Which pair of types both use 4 bytes of storage?
    A. `int` and `float`  B. `int` and `long`  C. `short` and `char`  D. `byte` and `short`

14. What is the outcome of `System.out.println(0.1 + 0.2 == 0.3);`?
    A. `true`  B. `false`  C. Compile error  D. Depends on JVM vendor

15. Given `short s = 40000;`, what happens?
    A. Compiles and stores 40000  B. Compile time error, out of range for a literal assignment  C. Silently wraps to a negative value at compile time with no error  D. Runtime exception

### 2.7 Quiz Answers and Reasoning

1. **Answer: B, 64.** `long` occupies 64 bits, giving it a much larger range than `int`, which is 32 bits. This size difference is exactly why `long` is chosen for very large counters, timestamps in milliseconds, and similar use cases.

2. **Answer: B, `false`.** Every primitive has a defined zero equivalent default when used as a field. For `boolean`, that default is `false`, not `0`, since `boolean` is not numerically compatible with `int` in Java.

3. **Answer: C, `-2147483648`.** Adding 1 to the maximum representable `int` value causes a silent overflow that wraps around to the minimum representable `int` value, due to two's complement arithmetic. Java performs no runtime check for this, ruling out both a compile error and any exception.

4. **Answer: B.** `float f = 3.5;` fails to compile because decimal literals default to `double`, and assigning a `double` value to a `float` variable is a narrowing conversion that requires either an explicit cast or the `f` suffix on the literal, such as `3.5f`. Options A, C, and D all involve compatible literal to variable type combinations and compile without issue.

5. **Answer: B, `int`.** Even though both operands are `char`, arithmetic operators promote `char` operands to `int` before performing the operation, so the addition produces an `int` result, specifically the sum of the two Unicode code points, not a `String` concatenation and not another `char`.

6. **Answer: B, `-128`.** `byte` has a range of -128 to 127. Incrementing `127` overflows the 8 bit signed range and wraps around to `-128`, following the same two's complement wraparound principle that applies to all the integer primitive types, not just `int`.

7. **Answer: C.** `char` is the one primitive type in Java that is unsigned, occupying 16 bits with a range from 0 to 65535. This distinguishes it from every signed integer type, and it is why a `char` can never hold a negative value directly.

8. **Answer: C.** Unlike C or C++, Java enforces strict type separation between `boolean` and the integer types, so there is no implicit or explicit conversion between them. Assigning the integer literal `1` to a `boolean` variable is a compile time type mismatch error.

9. **Answer: C, `0.0`.** `d` is a `double` field, so its automatic default value is `0.0`, the double specific zero representation, not the integer `0` and not `null`, since `double` is a primitive type and primitives are never `null`.

10. **Answer: B.** Decimal literals are `double` by default, so assigning one directly to a `float` variable without either a cast or the `f` suffix is a compile time narrowing conversion error, which rules out A. The `f` suffix in option B correctly marks the literal as a `float` literal at compile time. Option C uses the `d` suffix, which explicitly marks it as `double`, making it incompatible with a `float` variable. Option D attempts to cast a `String`, which is entirely invalid syntax for producing a primitive `float`.

11. **Answer: C, still `Integer.MIN_VALUE`, i.e. negative.** Two's complement signed ranges are asymmetric, since there is one more negative value representable than positive values. Negating `Integer.MIN_VALUE` mathematically would require a value one greater than `Integer.MAX_VALUE`, which cannot be represented as an `int`, so the operation overflows and wraps back around to `Integer.MIN_VALUE` itself, meaning `Math.abs()` on this specific input famously still returns a negative number.

12. **Answer: C, compile time error.** `x` is a local variable inside `main`, and local variables receive no default value. The compiler performs definite assignment analysis and refuses to compile any code path that reads a local variable before it has definitely been assigned a value.

13. **Answer: A.** Both `int` and `float` occupy 4 bytes, which is 32 bits, even though one stores whole numbers and the other stores IEEE 754 floating point values. `long` and `double` both use 8 bytes, `short` uses 2 bytes, `char` uses 2 bytes, and `byte` uses 1 byte, so the other listed pairings mix different sizes.

14. **Answer: B, `false`.** Both `0.1` and `0.2` are stored as the closest representable binary approximations rather than their exact decimal values, and their sum's approximation does not exactly equal the closest representable approximation of `0.3`, so the equality comparison evaluates to `false`. This behavior is consistent across compliant JVMs because IEEE 754 is a strict standard, which rules out option D.

15. **Answer: B, compile time error.** `short` has a maximum value of `32767`. The literal `40000` exceeds this range, and unlike some narrowing assignment contexts involving constant expressions that fit, a literal that plainly exceeds the target type's range in a direct assignment is rejected by the compiler as a possible loss of precision, so this never reaches runtime.

### 2.8 Programming Practice: Primitive Data Types

1. **Basic.** Write a program that declares one variable of each of the eight primitive types with sensible sample values and prints each on its own line, labeled with the type name.
2. **Intermediate.** Write a program that demonstrates `int` overflow by starting from `Integer.MAX_VALUE - 2` and incrementing a loop counter five times, printing the value after each increment.
3. **Intermediate.** Write a program that reads two `double` values representing prices, sums them, and checks if the sum equals a third given `double` value using both direct `==` comparison and an epsilon based comparison with a tolerance of `0.0001`, printing both results so the difference in reliability is visible.
4. **Advanced.** Write a program that takes a `char` variable holding an uppercase letter and, without using any built in case conversion method such as `Character.toLowerCase`, computes and prints the corresponding lowercase letter purely through arithmetic on the underlying numeric code point.
5. **Edge case based.** Write a program that demonstrates the `Integer.MIN_VALUE` absolute value anomaly by printing `Integer.MIN_VALUE`, `Math.abs(Integer.MIN_VALUE)`, and `-Integer.MIN_VALUE`, and add a comment beneath the output explaining why the second and third lines are not the positive value you might expect.
6. **Logic intensive.** Write a program that simulates a small fixed size counter using a `byte` variable that should conceptually count from 0 up to 300, wrapping around correctly every time it exceeds `Byte.MAX_VALUE`, printing the counter value at each of the first 140 steps only when the value is exactly `0`, `127`, or `-128`, to clearly show every wraparound point.

### 2.9 Programming Solutions: Primitive Data Types

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        byte byteVal = 100;
        short shortVal = 20000;
        int intVal = 1_000_000;
        long longVal = 9_000_000_000L;
        float floatVal = 3.14f;
        double doubleVal = 3.14159265358979;
        char charVal = 'K';
        boolean boolVal = true;

        System.out.println("byte: " + byteVal);
        System.out.println("short: " + shortVal);
        System.out.println("int: " + intVal);
        System.out.println("long: " + longVal);
        System.out.println("float: " + floatVal);
        System.out.println("double: " + doubleVal);
        System.out.println("char: " + charVal);
        System.out.println("boolean: " + boolVal);
    }
}
```

Thought process: this is purely a syntax and range familiarity exercise, choosing sample values that comfortably fit within each type's range, including underscore digit separators for readability in the larger literals, which Java permits within numeric literals.

Why it works: each literal is compatible with its declared type; the `L` suffix on `longVal` is required because the literal exceeds the `int` range, and the `f` suffix on `floatVal` is required to avoid a narrowing conversion error from the default `double` literal type.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int counter = Integer.MAX_VALUE - 2;
        for (int i = 0; i < 5; i++) {
            counter++;
            System.out.println(counter);
        }
    }
}
```

Expected output:
```
2147483646
2147483647
-2147483648
-2147483647
-2147483646
```

Thought process: starting two below the maximum, the third increment crosses the overflow boundary. Tracing the loop by hand rather than trusting intuition is the key skill here, since intuition about "add five small numbers" does not warn you about the boundary crossing.

Why it works: standard `int` overflow wraparound as covered in the concept section; each increment past `Integer.MAX_VALUE` continues counting up from `Integer.MIN_VALUE`.

Edge case to note: if the loop ran enough additional iterations, the counter would eventually wrap a second time back toward zero, since the wraparound behavior is cyclic across the entire 32 bit range.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        double price1 = 19.99;
        double price2 = 5.01;
        double expectedTotal = 25.00;

        double actualTotal = price1 + price2;
        boolean directEquals = (actualTotal == expectedTotal);

        double epsilon = 0.0001;
        boolean epsilonEquals = Math.abs(actualTotal - expectedTotal) < epsilon;

        System.out.println("Actual total: " + actualTotal);
        System.out.println("Direct == comparison: " + directEquals);
        System.out.println("Epsilon based comparison: " + epsilonEquals);
    }
}
```

Thought process: the exercise deliberately picks values likely to expose floating point representational drift, and contrasts a naive `==` comparison with the standard, more reliable epsilon based technique used in real world floating point code.

Why it works: `Math.abs(actualTotal - expectedTotal) < epsilon` tolerates a tiny amount of representational error, which is the industry standard technique for comparing floating point values for practical equality, whereas `==` demands bit for bit identical representations.

Common incorrect approach: relying solely on `==` for floating point comparisons in any program that performs decimal arithmetic, which can silently produce incorrect branching decisions.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        char upper = 'G';
        char lower = (char) (upper + ('a' - 'A'));
        System.out.println("Uppercase: " + upper);
        System.out.println("Lowercase: " + lower);
    }
}
```

Thought process: the distance, in code points, between an uppercase letter and its lowercase counterpart is constant across the entire alphabet, equal to the distance between `'a'` and `'A'`. Adding that fixed offset to any uppercase letter's code point produces the matching lowercase letter's code point.

Why it works: ASCII, and correspondingly the first 128 Unicode code points that Java's `char` type uses, places lowercase letters at a fixed positive offset from their uppercase counterparts. The expression `upper + ('a' - 'A')` computes as `int` due to promotion rules, and the explicit cast back to `char` is required to store the result as a character.

Edge case to note: this only works correctly for actual uppercase letters; passing a digit or symbol would produce a nonsensical but still technically valid character result, since there is no validation in this simple version.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        System.out.println(Integer.MIN_VALUE);
        System.out.println(Math.abs(Integer.MIN_VALUE));
        System.out.println(-Integer.MIN_VALUE);
        // Both Math.abs(Integer.MIN_VALUE) and -Integer.MIN_VALUE overflow back
        // to Integer.MIN_VALUE itself, because the true mathematical positive
        // magnitude of Integer.MIN_VALUE is one greater than Integer.MAX_VALUE
        // and therefore cannot be represented as a 32 bit signed int, so the
        // computation wraps around rather than throwing any error.
    }
}
```

Output:
```
-2147483648
-2147483648
-2147483648
```

Thought process and why it works: this solution is entirely about correctly predicting and explaining the asymmetry of two's complement signed ranges, a classic tricky exam point covered already in the scenario and quiz sections above.

**Solution 6.**

```java
public class Solution6 {
    public static void main(String[] args) {
        byte counter = 0;
        for (int step = 0; step < 140; step++) {
            if (counter == 0 || counter == Byte.MAX_VALUE || counter == Byte.MIN_VALUE) {
                System.out.println("Step " + step + ": counter = " + counter);
            }
            counter++;
        }
    }
}
```

Thought process: rather than looping the `byte` variable itself up to 300 directly, an outer `int` loop counter drives exactly 140 iterations, while the `byte` variable wraps according to its own 8 bit range every 256 increments. The condition prints only at the three landmark values requested: the starting zero, the positive boundary, and the point right after wraparound where the value becomes the negative boundary.

Why it works: `byte` wraps from `127` to `-128` on overflow just like `int` wraps at its own boundaries, only reached far sooner because of the much smaller 8 bit range. Printing is gated on the landmark check so that the output stays compact and clearly illustrates every meaningful transition rather than flooding the console with all 140 lines.

Time complexity: O(n) where n is 140, the fixed number of steps. Space complexity: O(1). Edge case to note: since `Byte.MAX_VALUE` is `127`, the wraparound to `Byte.MIN_VALUE`, which is `-128`, occurs on the 128th increment starting from zero, meaning within the requested 140 steps the wraparound point is reached and printed exactly once, along with the initial zero at the very first step.

---

## 3. Variables and Literals

### 3.1 Concept Explanation

A variable is a named storage location in memory whose value can change during program execution, while a literal is a fixed, unchanging value written directly into source code that represents itself. Understanding both concepts precisely, and the specific rules Java enforces around each, is foundational to everything else in the language, since every later topic from operators to control flow to object creation depends on how variables are declared, scoped, and initialized.

**Kinds of variables.** Java recognizes three kinds of variables based on where they are declared and how long they live.

1. **Instance variables**, also called fields, are declared inside a class but outside any method, constructor, or block, without the `static` keyword. Each object created from the class gets its own independent copy of every instance variable. Instance variables receive automatic default values and live as long as the object they belong to is reachable.
2. **Static variables**, also called class variables, are declared with the `static` keyword at the class level. Exactly one copy of a static variable exists per class, shared across every instance, and it exists for as long as the class remains loaded in the JVM, independent of any particular object's lifetime.
3. **Local variables** are declared inside a method body, constructor body, or any block such as a loop or an `if` statement. They exist only for the duration of that block's execution and are stored, conceptually, on the call stack. Local variables must be explicitly initialized before use, since they receive no default value.

There is a further special case worth knowing: a parameter passed into a method or constructor is technically also a kind of local variable from the perspective of scope and lifetime, though it is initialized automatically by the act of the method being called with an argument.

**Rules for naming identifiers.** An identifier, which is the name given to a variable, method, class, or other user defined entity, must begin with a letter, an underscore `_`, or a dollar sign `$`, and every subsequent character must be a letter, digit, underscore, or dollar sign. Identifiers cannot begin with a digit, cannot contain spaces, and cannot be a reserved keyword as covered in the earlier section. Java identifiers are case sensitive, so `total`, `Total`, and `TOTAL` are three entirely distinct names.

Java also permits, by specification, identifiers that include Unicode letters beyond the basic Latin alphabet, though exam questions rarely test this directly beyond confirming that letters, digits, underscore, and dollar sign are the only allowed character categories.

**Variable declaration versus initialization.** Declaration is the act of introducing a variable's name and type to the compiler, such as `int count;`. Initialization is the act of giving that variable its first actual value, such as `count = 0;`. These two steps can be combined in a single statement, `int count = 0;`, which is the most common style, but they remain conceptually distinct, and understanding this distinction matters for correctly reasoning about local variable definite assignment rules.

**Literals in depth.** A literal represents a fixed value written directly in source code.

- **Integer literals** are `int` by default. They may be written in decimal such as `123`, octal with a leading `0` such as `0173`, hexadecimal with a leading `0x` or `0X` such as `0x7B`, or binary with a leading `0b` or `0B` such as `0b1111011`, all four of which represent the same numeric value. A trailing `L` or `l` marks a literal as `long`.
- **Floating point literals** are `double` by default. A trailing `f` or `F` marks a literal as `float`, and a trailing `d` or `D` explicitly marks a literal as `double`, though the `d` suffix is optional since it is already the default.
- **Character literals** are written between single quotes, such as `'A'`, and may include escape sequences such as `'\n'` for newline, `'\t'` for tab, `'\\'` for a literal backslash, `'\''` for a literal single quote, and Unicode escapes such as `'\u0041'` which represents the same character as `'A'`.
- **String literals** are written between double quotes, such as `"Hello"`, and are technically references to `String` objects, making `String` the one literal type in Java that is not a primitive.
- **Boolean literals** are exactly `true` and `false`, and nothing else can be assigned directly to a `boolean` variable.
- **The null literal** `null` represents the absence of a reference and can be assigned to any reference type variable, but never to a primitive type variable.
- **Underscore digit separators**, introduced in Java 7, allow literals like `1_000_000` to be written for readability. Underscores cannot appear adjacent to the start or end of the digit sequence, immediately before or after a decimal point, or immediately before an `L`, `f`, or `d` suffix.

**Scope of a variable.** Scope refers to the region of code within which a variable's name is visible and usable. Local variables are scoped to the block in which they are declared, meaning they cease to exist once that block, such as a loop body or method, finishes executing. Instance and static variables are scoped to the entire class, visible from any instance method, static method, constructor, or initializer block within that class, subject to access modifier restrictions when accessed from outside the class.

**Shadowing.** When a local variable or parameter shares the same name as an instance or static variable, the local variable takes precedence within its scope, a situation called shadowing. The instance or static variable is not destroyed or altered; it is simply temporarily hidden from direct name access within that scope, and can still be accessed explicitly using `this.fieldName` for instance variables or `ClassName.fieldName` for static variables.

### 3.2 Exam Perspective

1. **Direct conceptual questions** ask you to classify a given variable declaration as an instance variable, a static variable, or a local variable based on where it appears in the code.
2. **Output based questions** test shadowing, showing a constructor parameter with the same name as a field and asking what value ends up stored in the field after the constructor runs, particularly whether `this` was used correctly.
3. **Error identification questions** show an invalid identifier such as one starting with a digit, one containing a space, or one that is a reserved keyword, and ask you to spot the naming rule violation.
4. **Literal type questions** ask what type a literal defaults to, such as asking whether `100` is `int` or `long`, or whether `3.14` is `float` or `double` by default.
5. **Debugging questions** show code using a local variable outside the block where it was declared, testing your understanding of block scope.
6. **Tricky or misleading questions** present octal literals with a leading zero, such as `010`, and ask for its decimal value, exploiting the fact that many candidates forget the leading zero triggers octal interpretation rather than treating it as decimal ten.
7. **Underscore separator questions** test the specific rules about where underscores may legally appear within a numeric literal.
8. **Scenario questions combining scope and shadowing** present nested blocks, such as a local variable inside an `if` block sharing a name with one in the enclosing method, testing whether you understand that Java disallows two local variables of the same name in overlapping scope, unlike shadowing between a local variable and a field.
9. **Questions on definite assignment** test whether a local variable that is only conditionally assigned, for example only inside one branch of an `if` without an `else`, can be used afterward, expecting recognition that the compiler will reject this due to a code path where it was never assigned.

### 3.3 Examples

**Example 1: Instance, static, and local variables together.**

```java
public class Counter {
    static int totalCounters = 0; // static variable
    int id;                        // instance variable

    Counter() {
        int localTemp = totalCounters; // local variable
        id = localTemp;
        totalCounters++;
    }
}

public class Demo {
    public static void main(String[] args) {
        Counter c1 = new Counter();
        Counter c2 = new Counter();
        System.out.println(c1.id + " " + c2.id + " " + Counter.totalCounters);
    }
}
```

Output: `0 1 2`

Explanation: `totalCounters` is shared across all `Counter` objects, incrementing once per construction. `id` is unique per object, capturing the shared counter's value at the moment of construction. `localTemp` exists only during the constructor call and cannot be accessed from anywhere else.

**Example 2: Shadowing resolved with `this`.**

```java
public class Point {
    int x;
    int y;

    Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}

public class Demo {
    public static void main(String[] args) {
        Point p = new Point(3, 4);
        System.out.println(p.x + ", " + p.y);
    }
}
```

Output: `3, 4`

Explanation: the constructor parameters `x` and `y` shadow the instance fields of the same name within the constructor body. Using `this.x` and `this.y` explicitly refers to the instance fields, correctly assigning the passed in argument values to them, rather than the parameters simply being assigned to themselves, which would leave the fields at their default value of `0`.

**Example 3: Integer literal formats representing the same value.**

```java
public class Demo {
    public static void main(String[] args) {
        int decimal = 123;
        int octal = 0173;
        int hex = 0x7B;
        int binary = 0b1111011;
        System.out.println(decimal == octal && octal == hex && hex == binary);
    }
}
```

Output: `true`

Explanation: all four literals represent the decimal value 123 despite being written in four different numeral systems, demonstrating that the leading `0`, `0x`, and `0b` prefixes are purely notational and do not change the underlying stored value.

**Example 4: Block scope preventing access outside the block.**

```java
public class Demo {
    public static void main(String[] args) {
        if (true) {
            int localValue = 42;
            System.out.println(localValue);
        }
        // System.out.println(localValue); // would not compile: localValue is out of scope here
    }
}
```

Explanation: `localValue` is scoped strictly to the `if` block in which it was declared. Once that block ends, the name `localValue` no longer refers to anything, and attempting to reference it afterward is a compile time "cannot find symbol" error.

### 3.4 Important Notes

- Instance variables get default values and one copy per object; static variables get default values and exactly one copy shared across the whole class; local variables get no default value and must be assigned before use.
- Identifiers must start with a letter, underscore, or dollar sign, cannot start with a digit, and cannot be a reserved keyword.
- Integer literals default to `int`; a trailing `L` marks `long`. Decimal literals default to `double`; a trailing `f` marks `float`.
- A leading `0` on an integer literal means octal, not decimal; a leading `0x` means hexadecimal; a leading `0b` means binary.
- `String` literals are references to objects, making `String` the sole non primitive literal type.
- Underscores in numeric literals cannot be placed at the very start or end of the digit sequence, nor adjacent to a decimal point or a type suffix.
- Shadowing hides an outer variable's name within an inner scope; it does not destroy or overwrite the outer variable, and `this` or the class name can still reach it explicitly.
- Two local variables cannot share the same name within overlapping or nested scope in the same method, even though a local variable is allowed to shadow a field.
- A local variable that is assigned only inside one conditional branch, with no guaranteed assignment on every possible path, is treated by the compiler as possibly unassigned if used afterward, causing a compile error.

### 3.5 Scenario Based Understanding

**Scenario A.** A question defines a class with an instance field `count` and a method `increment(int count)` that takes a parameter of the same name, then asks what happens if the method body writes `count = count + 1;` without using `this`.
- What is happening: the parameter shadows the instance field inside the method.
- Which concept is involved: shadowing and variable scope.
- How to identify it: notice the parameter name exactly matches the field name.
- Correct reasoning: `count = count + 1;` only modifies the local parameter copy; the instance field `count` remains completely unchanged after the method returns, since the unqualified name resolves to the nearer, shadowing parameter.
- Common mistake: assuming that because the field and parameter share a name, assigning to the parameter updates the field, when in fact `this.count = count + 1;` would be required to update the field.

**Scenario B.** A question shows a local variable declared inside a `for` loop's body and asks whether it can be accessed immediately after the loop finishes.
- What is happening: the variable's scope is limited to a single iteration of the loop body block.
- Which concept is involved: block scope.
- How to identify it: locate exactly which pair of braces encloses the declaration.
- Correct reasoning: the variable ceases to exist once that iteration's block ends, so referencing it after the loop is a compile time error, regardless of how many iterations the loop performed.
- Common mistake: confusing a variable declared inside the loop's body with the loop counter variable declared in the `for` statement's own header, which also does not survive past the loop but is sometimes conflated with variables declared before the loop entirely.

### 3.6 Quiz: Variables and Literals

1. Which of these is a valid Java identifier?
   A. `2total`  B. `total_2`  C. `total 2`  D. `class`

2. What kind of variable is declared as `static int count;` inside a class body?
   A. Local variable  B. Instance variable  C. Static variable  D. Parameter

3. What is the default type of the literal `3.14`?
   A. `float`  B. `double`  C. `int`  D. It has no default and must be explicit

4. What is the decimal value of the literal `010`?
   A. Ten  B. Eight  C. Compile error  D. Zero

5. Which underscore placement in a numeric literal is illegal?
   A. `1_000_000`  B. `_1000`  C. `10_00`  D. `1_0`

6. What happens when a local variable is used without being assigned a value first?
   A. It defaults to zero or null  B. Compile time error  C. Runtime exception  D. Undefined behavior at runtime

7. In the following code, what is printed?
   ```java
   class Box {
       int size = 10;
       void setSize(int size) {
           this.size = size;
       }
   }
   public class Demo {
       public static void main(String[] args) {
           Box b = new Box();
           b.setSize(25);
           System.out.println(b.size);
       }
   }
   ```
   A. `10`  B. `25`  C. Compile error  D. `0`

8. Which statement about static variables is correct?
   A. Each object gets its own separate copy  B. They exist even with zero objects created, once the class is loaded  C. They cannot have default values  D. They must be declared inside a method

9. Which of these literal declarations is invalid without modification?
   A. `long l = 100;`  B. `float f = 2.5;`  C. `double d = 2.5;`  D. `char c = 'A';`

10. What is the result of the following?
    ```java
    public class Demo {
        public static void main(String[] args) {
            int x;
            if (5 > 3) {
                x = 10;
            }
            System.out.println(x);
        }
    }
    ```
    A. `10`  B. `0`  C. Compile time error  D. Runtime exception

11. Which character sequence is a legal identifier?
    A. `$amount`  B. `#amount`  C. `1amount`  D. `amount!`

12. What is the output of `System.out.println(0x1A);`?
    A. `1A`  B. `26`  C. `16`  D. Compile error

13. Which best describes the relationship between declaration and initialization?
    A. They are always the same statement  B. Declaration introduces the name and type; initialization assigns the first value; they can be combined or separate  C. Initialization must come before declaration  D. Only fields can be declared without initialization

14. Can two local variables with the same name exist in nested blocks within the same method, such as one in the method body and another inside a nested `if` block with the same name?
    A. Yes, the inner one always shadows the outer one  B. No, this causes a compile time error due to a naming conflict  C. Yes, but only if their types differ  D. Yes, but only inside loops

15. What is the value of `binaryLiteral` below?
    ```java
    int binaryLiteral = 0b1010;
    ```
    A. `1010`  B. `10`  C. `1010` in binary displayed as is  D. Compile error

### 3.7 Quiz Answers and Reasoning

1. **Answer: B, `total_2`.** Identifiers cannot start with a digit, ruling out A, cannot contain a space, ruling out C, and cannot be a reserved keyword, ruling out D. `total_2` starts with a letter and uses only letters, digits, and an underscore, satisfying every naming rule.

2. **Answer: C, static variable.** The `static` keyword at class level, outside any method, marks this as a static or class variable, meaning exactly one shared copy exists regardless of how many objects of the class are created.

3. **Answer: B, `double`.** Any floating point literal without an explicit `f` or `F` suffix defaults to `double` in Java, which is why assigning such a literal directly to a `float` variable requires either the suffix or an explicit cast.

4. **Answer: B, eight.** A leading `0` on an integer literal signals octal notation to the compiler. `010` in octal equals one times eight plus zero times one, which is eight in decimal, a classic trap for anyone assuming leading zeros are purely cosmetic.

5. **Answer: B, `_1000`.** Underscores are disallowed at the very beginning or end of the digit sequence within a numeric literal. `1_000_000`, `10_00`, and `1_0` all place underscores strictly between digits, which is legal; `_1000` places the underscore before any digit, which the compiler rejects.

6. **Answer: B, compile time error.** Java performs definite assignment analysis on local variables at compile time, and any code path that could read a local variable before it is assigned causes a compilation failure, never a runtime exception and never a silent default value, since locals simply have none.

7. **Answer: B, `25`.** Inside `setSize`, the parameter `size` shadows the field `size`. The statement `this.size = size;` explicitly targets the instance field using `this`, correctly assigning the passed in argument `25` to it, overwriting the field's original value of `10`.

8. **Answer: B.** Static variables belong to the class itself rather than to any individual object, so they are allocated and given their default value as soon as the class is loaded by the JVM, independent of whether any instances have been constructed yet, which rules out A entirely and contradicts the premise of C.

9. **Answer: B, `float f = 2.5;`.** The literal `2.5` is a `double` literal by default. Assigning it directly to a `float` variable is a narrowing conversion that the compiler rejects without an explicit `f` suffix or cast, unlike the other three options which all use compatible literal to variable type pairings.

10. **Answer: A, `10`.** Although `x` is only assigned inside the `if` block, the compiler can prove through constant expression analysis that the literal condition `5 > 3` is always `true`, meaning the assignment always executes on every possible path, so this particular case actually does compile and print `10`. This is a genuinely tricky question because a non constant condition would instead produce a compile error for possible non initialization; a hardcoded, provably true constant condition is treated differently by the compiler's flow analysis.

11. **Answer: A, `$amount`.** Identifiers may begin with a letter, an underscore, or a dollar sign. `#amount` and `amount!` contain characters that are never legal in an identifier, and `1amount` illegally starts with a digit.

12. **Answer: B, `26`.** `0x1A` is a hexadecimal literal. Converting `1A` from base sixteen to decimal gives one times sixteen plus ten, which equals twenty six. `println` always displays the underlying decimal value of an `int`, regardless of which notation was used to write the literal in source code.

13. **Answer: B.** Declaration and initialization are conceptually separate steps: declaration tells the compiler a name and type exist, and initialization supplies the first actual value. They are frequently combined into a single statement for convenience, but they remain distinct operations, and understanding this distinction is essential for reasoning about definite assignment rules for local variables.

14. **Answer: B, compile time error.** Unlike shadowing a field, Java does not allow one local variable to shadow another local variable that is still in scope within the same method, including across nested blocks. Attempting to redeclare the same local variable name inside a nested block that is still within an outer, already active scope produces a compile time "variable is already defined" error.

15. **Answer: B, `10`.** `0b1010` is a binary literal. Converting `1010` from base two to decimal gives eight plus zero plus two plus zero, which equals ten. As with hexadecimal and octal literals, `println` displays the resulting decimal value of the `int`, not the original notation used in the source code.

### 3.8 Programming Practice: Variables and Literals

1. **Basic.** Write a program that declares a static variable, an instance variable, and a local variable, all conceptually representing "counts," and print all three with clear labels.
2. **Intermediate.** Write a `Rectangle` class with instance fields `width` and `height`, a constructor taking parameters of the same names that correctly uses `this` to assign them, and a `main` method that creates two different rectangles and prints both areas.
3. **Intermediate.** Write a program that declares four integer variables representing the same numeric value using decimal, octal, hexadecimal, and binary literal notation, then verifies and prints whether all four are equal.
4. **Advanced.** Write a program with a static counter field shared across a `Ticket` class, where every new `Ticket` object automatically receives the next sequential ticket number in its constructor, then create four tickets and print all of their numbers along with the final counter value.
5. **Edge case based.** Write a program demonstrating that a local variable declared and used entirely inside one `if` branch cannot be referenced in an `else` branch or after the whole `if else` structure, by writing the version that compiles correctly with separate, properly scoped declarations in each branch, and add a comment explaining what would break if you tried to share one declaration across both branches incorrectly.
6. **Logic intensive.** Write a program that reads no external input but hardcodes a small dataset of values written using a deliberate mixture of decimal, hexadecimal, octal, and binary literals, sums them all as plain integers, and prints the total, demonstrating that the compiler treats all four notations uniformly once compiled.

### 3.9 Programming Solutions: Variables and Literals

**Solution 1.**

```java
public class Solution1 {
    static int staticCount = 100;
    int instanceCount = 10;

    void show() {
        int localCount = 1;
        System.out.println("Static count: " + staticCount);
        System.out.println("Instance count: " + instanceCount);
        System.out.println("Local count: " + localCount);
    }

    public static void main(String[] args) {
        Solution1 obj = new Solution1();
        obj.show();
    }
}
```

Thought process: the exercise is purely about correctly placing each declaration in the right location to earn the right classification, then confirming with output that all three are accessible from an instance method.

Why it works: `staticCount` is class level with `static`, `instanceCount` is class level without `static`, and `localCount` is declared inside the `show` method body, satisfying the definitions of static, instance, and local variables respectively.

**Solution 2.**

```java
public class Rectangle {
    int width;
    int height;

    Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    int area() {
        return width * height;
    }

    public static void main(String[] args) {
        Rectangle r1 = new Rectangle(4, 5);
        Rectangle r2 = new Rectangle(10, 3);
        System.out.println("r1 area: " + r1.area());
        System.out.println("r2 area: " + r2.area());
    }
}
```

Thought process: this exercise combines shadowing resolution using `this` with basic instance state and a simple derived calculation method, reinforcing that each object maintains independent field values.

Why it works: `this.width` and `this.height` unambiguously refer to the instance fields inside the constructor, correctly separating them from the same named constructor parameters, so each `Rectangle` object stores its own correct dimensions.

Time complexity: O(1) per area calculation. Space complexity: O(1) per object beyond the two int fields.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int decimal = 45;
        int octal = 055;
        int hex = 0x2D;
        int binary = 0b101101;

        System.out.println("decimal = " + decimal);
        System.out.println("octal = " + octal);
        System.out.println("hex = " + hex);
        System.out.println("binary = " + binary);
        boolean allEqual = (decimal == octal) && (octal == hex) && (hex == binary);
        System.out.println("All equal: " + allEqual);
    }
}
```

Thought process: forty five was chosen and then manually converted to octal, hexadecimal, and binary notation to ensure all four literals genuinely represent the same value, which is essential for the equality check to meaningfully demonstrate the concept rather than accidentally failing.

Why it works: regardless of the notation used to write an integer literal, the compiler resolves it to the same underlying binary value stored in the variable, so comparing across notations with `==` correctly evaluates to `true`.

**Solution 4.**

```java
public class Ticket {
    static int nextNumber = 1000;
    int ticketNumber;

    Ticket() {
        this.ticketNumber = nextNumber;
        nextNumber++;
    }

    public static void main(String[] args) {
        Ticket t1 = new Ticket();
        Ticket t2 = new Ticket();
        Ticket t3 = new Ticket();
        Ticket t4 = new Ticket();

        System.out.println("t1: " + t1.ticketNumber);
        System.out.println("t2: " + t2.ticketNumber);
        System.out.println("t3: " + t3.ticketNumber);
        System.out.println("t4: " + t4.ticketNumber);
        System.out.println("Next available number: " + Ticket.nextNumber);
    }
}
```

Expected output:
```
t1: 1000
t2: 1001
t3: 1002
t4: 1003
Next available number: 1004
```

Thought process: this problem specifically tests whether you understand that a static field persists and accumulates changes across every object construction, since each constructor call reads the current shared value before incrementing it for the next object.

Why it works: `nextNumber` is static, so all four `Ticket` objects read and modify the exact same memory location rather than each getting an independent copy, which is precisely why the ticket numbers come out sequential rather than all identical.

Time complexity: O(1) per ticket creation. Space complexity: O(1) additional per object beyond its own `ticketNumber` field, plus the single shared static field.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        boolean condition = true;

        if (condition) {
            int ifValue = 100;
            System.out.println("Inside if: " + ifValue);
        } else {
            int elseValue = 200;
            System.out.println("Inside else: " + elseValue);
        }

        // If a single "int value" had been declared inside only the if block and
        // then referenced inside the else block or after the entire if-else
        // structure, it would not compile, because that declaration's scope is
        // strictly limited to the block of braces it appears within. The correct
        // pattern, shown above, is to either declare separately scoped variables
        // in each branch, as done here, or declare a single variable before the
        // if-else structure begins if the value needs to be visible afterward.
    }
}
```

Thought process: the exercise is really about correctly demonstrating block scope boundaries through a working, compiling example, then explaining in a comment exactly why the tempting shortcut of one shared declaration would fail.

**Solution 6.**

```java
public class Solution6 {
    public static void main(String[] args) {
        int a = 200;        // decimal
        int b = 0x1F;        // hexadecimal, 31
        int c = 017;          // octal, 15
        int d = 0b1100100;     // binary, 100

        int total = a + b + c + d;
        System.out.println("Total: " + total);
    }
}
```

Expected output: `Total: 346`

Thought process: the values were deliberately picked in different notations, then hand verified: `0x1F` equals 31, `017` equals 15 in octal, and `0b1100100` equals 100 in binary, so the expected sum is 200 plus 31 plus 15 plus 100, which is 346.

Why it works: once compiled, all four literals become ordinary `int` values in memory; the compiler discards the notation entirely after parsing, so ordinary integer addition applies uniformly regardless of how each operand was originally written in the source code.

---

## 4. Lend A Hand on Variables

This is a hands on reinforcement block for everything covered in the Variables and Literals section. It assumes the theory is already understood and focuses on applying it through denser examples, a fresh quiz, and additional programming problems that mix instance variables, static variables, local variables, scope, shadowing, and literal notation together in less predictable combinations than the introductory material.

### 4.1 Applied Examples

**Applied Example 1: Static variable interacting with multiple objects and a local variable in a loop.**

```java
public class Bank {
    static double totalDeposits = 0.0;
    double balance;

    Bank(double openingBalance) {
        balance = openingBalance;
        totalDeposits += openingBalance;
    }

    public static void main(String[] args) {
        double[] openingAmounts = {500.0, 1200.5, 75.25};
        for (double amount : openingAmounts) {
            Bank account = new Bank(amount);
        }
        System.out.println("Total deposits across all accounts: " + totalDeposits);
    }
}
```

Output: `Total deposits across all accounts: 1775.75`

Explanation: `account` is a local variable re-declared conceptually on each loop iteration, existing only within that iteration, while `totalDeposits` persists and accumulates across every object created throughout the entire loop, since it belongs to the class rather than to any individual `Bank` object.

**Applied Example 2: Shadowing across three levels: class field, method parameter, and inner block variable.**

```java
public class Demo {
    static int value = 1;

    static void process(int value) {
        System.out.println("Parameter value: " + value);
        {
            int localValue = value * 10;
            System.out.println("Inner block localValue: " + localValue);
        }
        System.out.println("Class field via Demo.value: " + Demo.value);
    }

    public static void main(String[] args) {
        process(5);
    }
}
```

Output:
```
Parameter value: 5
Inner block localValue: 50
Class field via Demo.value: 1
```

Explanation: the parameter `value` shadows the static field `value` throughout the method body. The inner block variable `localValue` is a completely new, separately scoped variable, unrelated to either `value`. The static field is still reachable and unaffected, but only by referring to it explicitly as `Demo.value`.

### 4.2 Quiz: Lend A Hand on Variables

1. In the Applied Example 1 code, if `openingAmounts` had been empty, what would `totalDeposits` print as?
   A. `0.0`  B. Compile error  C. `null`  D. Runtime exception

2. In Applied Example 2, what would `System.out.println(value);` print if placed directly inside `process`, unqualified?
   A. `1`  B. `5`  C. `50`  D. Compile error

3. What is the type of the literal `50_000L`?
   A. `int`  B. `long`  C. `double`  D. Compile error due to underscore placement

4. A static field `total` starts at `0`. Five objects are constructed, and each constructor adds `10` to `total`. What is `total` after all five constructions?
   A. `10`  B. `50`  C. `0`  D. Depends on object destruction order

5. Which of the following correctly shadows a field named `rate` inside a constructor and assigns the parameter's value to it?
   A. `rate = rate;`  B. `this.rate = rate;`  C. `rate = this.rate;`  D. `Rate.this = rate;`

6. Can a `static` method directly and unqualified access a non static instance field of its own class?
   A. Yes, always  B. No, because there is no implicit object to access the field on  C. Yes, but only inside `main`  D. Only if the field is `final`

7. What is the scope of a variable declared inside a `for` loop's header, such as `for (int i = 0; ...)`?
   A. The entire class  B. The entire method  C. The loop itself, including its body, but not beyond it  D. Only the first iteration

8. Given `final int LIMIT = 100;` declared as a local variable, can `LIMIT` be reassigned later in the same method?
   A. Yes  B. No, `final` prevents reassignment after initialization  C. Only inside a loop  D. Only if declared `static` too

9. What does the literal `1_0_0` evaluate to?
   A. Compile error  B. `100`  C. `1`, `0`, `0` as separate tokens  D. `1.00`

10. Which of these is true about parameters in a method signature?
    A. They behave like static variables  B. They behave like a special kind of local variable, initialized automatically by the caller's arguments  C. They must always be `final`  D. They receive default values if not passed

### 4.3 Quiz Answers and Reasoning

1. **Answer: A, `0.0`.** `totalDeposits` is a static `double` field, so it starts at its default value of `0.0` regardless of whether the loop runs any iterations at all. An empty array simply means the loop body never executes, leaving the field at its initial, already assigned value of `0.0`, not its uninitialized JVM default, since it was explicitly set to `0.0` in the declaration.

2. **Answer: B, `5`.** Inside `process`, the parameter `value` shadows the static field `value` for the entire method body, so any unqualified reference to `value` resolves to the parameter, which holds `5`, not the static field's `1`.

3. **Answer: B, `long`.** The trailing `L` suffix explicitly marks this literal as `long`, and the underscore sits cleanly between two digit groups, which is a legal placement, so there is no compile error.

4. **Answer: B, `50`.** Each of the five constructor calls adds `10` to the single shared static field, so after all five objects are constructed the field holds five times ten, which is `50`. Object destruction is irrelevant here since Java uses automatic garbage collection with no deterministic destructor timing that would affect this calculation.

5. **Answer: B.** `this.rate = rate;` explicitly targets the instance field via `this` on the left side while reading the shadowing parameter's value on the right side, correctly transferring the passed in value into the field. Option A only reassigns the parameter to itself and never touches the field, leaving it at its default or previously set value.

6. **Answer: B.** A `static` method has no implicit `this` reference and is not tied to any particular object, so it cannot access an instance field without first having an explicit object reference to access that field through, such as `someObject.fieldName`.

7. **Answer: C.** A variable declared in a `for` loop's initialization section is scoped to the loop as a whole, including every iteration of its body, but it ceases to exist once the loop finishes, unlike a variable declared before the loop begins.

8. **Answer: B.** The `final` modifier on a local variable enforces that, once it has been assigned a value, that value cannot be changed again anywhere later in its scope, making any later reassignment attempt a compile time error.

9. **Answer: B, `100`.** Underscores are placed strictly between digits in this literal, at both the position between the `1` and `0`, and between the `0` and `0`, which is entirely legal, so the compiler resolves the literal to the ordinary decimal value one hundred.

10. **Answer: B.** A method parameter is initialized automatically the moment the method is invoked, using whatever argument the caller supplied, and otherwise behaves exactly like an ordinary local variable in terms of scope, being confined to the method body, and lifetime.

### 4.4 Programming Practice: Lend A Hand on Variables

1. Write a `Student` class with a static field `schoolName` shared by all students and an instance field `studentName` unique to each, and print a formatted greeting for three different students showing both pieces of information.
2. Write a program with a method that takes a parameter shadowing a static field, and inside that method, add a nested block containing yet another local variable derived from the parameter, printing all three related values with clear labels to show they are genuinely distinct.
3. Write a program that uses a `final` local variable to represent a fixed tax rate, computes and prints the tax on three different purchase amounts using that same final variable, demonstrating that `final` allows repeated reading but would block any reassignment.

### 4.5 Programming Solutions: Lend A Hand on Variables

**Solution 1.**

```java
public class Student {
    static String schoolName = "Green Valley High School";
    String studentName;

    Student(String studentName) {
        this.studentName = studentName;
    }

    void greet() {
        System.out.println("Hello " + studentName + ", welcome to " + schoolName + "!");
    }

    public static void main(String[] args) {
        Student s1 = new Student("Ravi");
        Student s2 = new Student("Meera");
        Student s3 = new Student("Arjun");
        s1.greet();
        s2.greet();
        s3.greet();
    }
}
```

Why it works: `schoolName` is shared across all three `Student` objects since it is static, while `studentName` is set independently per object through the constructor parameter and correctly assigned using `this`.

**Solution 2.**

```java
public class Solution2 {
    static int base = 7;

    static void compute(int base) {
        System.out.println("Parameter base: " + base);
        {
            int doubled = base * 2;
            System.out.println("Nested block doubled: " + doubled);
        }
        System.out.println("Static field Solution2.base: " + Solution2.base);
    }

    public static void main(String[] args) {
        compute(3);
    }
}
```

Output:
```
Parameter base: 3
Nested block doubled: 6
Static field Solution2.base: 7
```

Why it works: the parameter shadows the static field throughout `compute`, the nested block variable `doubled` derives from the shadowing parameter rather than the static field, and explicitly qualifying with `Solution2.base` is the only way to reach the original static field's value of `7` from within this method.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        final double TAX_RATE = 0.08;
        double[] purchases = {50.00, 120.75, 999.99};

        for (double purchase : purchases) {
            double tax = purchase * TAX_RATE;
            System.out.println("Purchase: " + purchase + ", Tax: " + tax);
        }
        // TAX_RATE = 0.10; // would not compile, since TAX_RATE is final and
        // has already been assigned a value once.
    }
}
```

Why it works: `TAX_RATE` is read repeatedly across all three loop iterations without issue, since `final` only restricts reassignment, not repeated reading, which is exactly the intended and idiomatic use of a `final` local variable representing a constant value used throughout a calculation.

---

## 5. Casting Primitives

### 5.1 Concept Explanation

Casting is the process of converting a value of one primitive type into a value of another primitive type. Java is a strongly typed language, so it does not silently allow every possible conversion; instead, it distinguishes between conversions that are always safe and conversions that risk losing information, and it enforces different syntax requirements for each category.

**Widening conversion.** A widening, or implicit, conversion moves a value from a smaller or less precise type into a larger or more precise type. Because the destination type can always represent every value the source type could hold, Java performs this conversion automatically, without requiring any cast syntax at all. The widening conversion hierarchy, from smallest to largest, is:

```
byte -> short -> int -> long -> float -> double
```

There is one exception worth knowing precisely: `char` does not fit neatly into this single chain. `char` widens automatically to `int`, `long`, `float`, and `double`, but there is no automatic widening from `byte` or `short` to `char`, and no automatic conversion from `char` back down to `byte` or `short`, because `char` is unsigned while `byte` and `short` are signed, so their ranges do not simply nest inside one another in the same way the rest of the chain does.

**Narrowing conversion.** A narrowing, or explicit, conversion moves a value from a larger or more precise type into a smaller or less precise type. Because the destination type may not be able to represent every possible value the source type could hold, Java requires an explicit cast, written as the target type in parentheses immediately before the value, such as `(int) someDoubleValue`. Without this explicit cast, the compiler rejects the assignment, since it cannot silently accept a conversion that might lose data.

**What actually happens during a narrowing cast, precisely.** The exact truncation or conversion behavior differs depending on the source and destination types, and this is one of the richest sources of tricky exam questions.

- **Casting a floating point type to an integer type** truncates the fractional part entirely, it does not round. `(int) 9.99` produces `9`, not `10`, and `(int) -9.99` produces `-9`, not `-10`, since truncation always moves toward zero, not downward.
- **Casting a `double` or `float` value that is too large in magnitude to fit the target integer type** clamps to the target type's maximum or minimum value rather than wrapping around. For example, `(int) 1e20` produces `Integer.MAX_VALUE`, and `(int) Double.NaN` produces `0`. This clamping behavior for floating point to integer narrowing is a special case and differs from the wraparound behavior seen in integer to integer narrowing.
- **Casting a larger integer type to a smaller integer type**, such as `long` to `int`, or `int` to `byte`, discards the extra higher order bits and keeps only the lowest bits that fit the destination size, which can produce wraparound style results that look unrelated to the original value if the original value did not fit in the smaller type's range.
- **Casting `int` to `char`** takes the lowest 16 bits of the `int` value and interprets them as an unsigned `char` code point; negative `int` values produce unexpected, hard to predict `char` results because of this reinterpretation.

**Casting in expressions versus casting in assignments.** A cast only applies to the single value or sub expression it directly precedes, governed by normal operator precedence. `(int) 3.5 + 2.5` casts only the `3.5`, producing `3 + 2.5`, which evaluates to the `double` value `5.5`, not `5` as a careless reading might suggest.

**Boxing and unboxing are related but distinct from primitive casting.** Autoboxing is the automatic conversion of a primitive value into its corresponding wrapper object, such as `int` into `Integer`, and unboxing is the reverse. While closely related conceptually, boxing and unboxing are not the same operation as widening or narrowing between two primitive types, and mixing them incorrectly, for example trying to unbox a `null` `Integer` reference into an `int`, throws a `NullPointerException` at runtime rather than behaving like ordinary primitive narrowing.

### 5.2 Exam Perspective

1. **Direct conceptual questions** ask you to state whether a specific conversion, such as `int` to `long` or `double` to `float`, is widening or narrowing, and whether it requires an explicit cast.
2. **Output based questions** show a narrowing cast from `double` to `int` and ask for the exact truncated result, testing whether you know truncation discards the fractional part without rounding.
3. **Code tracing questions** trace a chain of casts, such as `double` to `int` to `byte`, asking for the final value after each successive truncation and possible overflow.
4. **Error identification questions** show a narrowing assignment missing its required explicit cast and ask you to identify the compile error.
5. **Debugging questions** present unexpected output from casting an out of range `int` value to `byte` or `short`, testing whether you can manually compute the wraparound result.
6. **Scenario based questions** describe real world situations such as converting a monetary `double` amount to `int` cents and ask what subtle bug this introduces due to truncation rather than rounding.
7. **Tricky or misleading questions** exploit the special clamping behavior of an out of range floating point to integer cast, expecting candidates to wrongly assume ordinary bitwise wraparound applies uniformly to every kind of narrowing conversion.
8. **Questions combining multiple concepts** combine casting with operator promotion rules, for example asking about the result type of `byte + byte` before any cast is applied, connecting back to the earlier Primitive Data Types section.
9. **Precision loss questions** ask what happens when a large `long` value is cast to `float`, testing awareness that `float`, despite being a "larger" type in the widening chain, has less precision than `long` for very large integer values, so widening a `long` to `float` or `double` can itself silently lose precision even though no cast is syntactically required.

### 5.3 Examples

**Example 1: Truncation, not rounding.**

```java
public class Demo {
    public static void main(String[] args) {
        double d1 = 9.99;
        double d2 = -9.99;
        System.out.println((int) d1);
        System.out.println((int) d2);
    }
}
```

Output:
```
9
-9
```

Explanation: casting a `double` to an `int` always truncates toward zero by discarding the fractional part entirely; it never rounds to the nearest whole number, which is why `9.99` becomes `9` rather than `10`, and `-9.99` becomes `-9` rather than `-10`.

**Example 2: Narrowing int to byte with wraparound.**

```java
public class Demo {
    public static void main(String[] args) {
        int i = 130;
        byte b = (byte) i;
        System.out.println(b);
    }
}
```

Output: `-126`

Explanation: `byte` only keeps the lowest 8 bits of the `int` value. `130` in binary, using enough bits, is `10000010`. Interpreting those same 8 bits as a signed `byte` gives a negative value, since the highest bit is set, and the correct signed interpretation of that bit pattern is `-126`.

**Example 3: Clamping behavior for out of range floating point to int casts.**

```java
public class Demo {
    public static void main(String[] args) {
        double huge = 1e20;
        double nanValue = Double.NaN;
        System.out.println((int) huge);
        System.out.println((int) nanValue);
    }
}
```

Output:
```
2147483647
0
```

Explanation: unlike integer to integer narrowing, which wraps around using the low order bits, an out of range floating point to integer cast clamps to the destination type's maximum value when the source is too large and positive, and treats `NaN` as `0`, since `NaN` cannot be meaningfully ordered or truncated to any specific integer.

**Example 4: Cast applies only to the immediately following operand.**

```java
public class Demo {
    public static void main(String[] args) {
        double result = (int) 3.9 + 2.9;
        System.out.println(result);
    }
}
```

Output: `5.9`

Explanation: the cast `(int)` binds only to `3.9`, producing `3`. That `3`, an `int`, is then added to `2.9`, a `double`, and by the usual arithmetic promotion rules the `int` operand is promoted to `double` for the addition, yielding `3.0 + 2.9`, which is `5.9`.

**Example 5: Widening does not need explicit casting, but can still lose precision.**

```java
public class Demo {
    public static void main(String[] args) {
        long bigValue = 123456789123456789L;
        float f = bigValue; // widening, compiles without a cast
        System.out.println(f);
    }
}
```

Output: approximately `1.2345679E17`

Explanation: `long` to `float` is classified as a widening conversion in Java, so no explicit cast is syntactically required. However, `float` cannot represent every possible `long` value exactly, since it only has around 6 to 7 significant decimal digits of precision, so the printed value is a rounded approximation of the original exact `long` value, illustrating that "widening" refers to the type's range, not necessarily its precision.

### 5.4 Important Notes

- Widening conversions happen automatically without an explicit cast; narrowing conversions always require an explicit cast in parentheses.
- The widening chain is `byte` to `short` to `int` to `long` to `float` to `double`; `char` widens to `int`, `long`, `float`, and `double`, but does not participate automatically with `byte` or `short` in either direction.
- Casting a floating point value to an integer type truncates the fractional part toward zero; it never rounds.
- An out of range floating point to integer cast clamps to the destination type's `MAX_VALUE` or `MIN_VALUE`, and `NaN` becomes `0`; this differs from integer to integer narrowing, which wraps around using low order bits instead of clamping.
- Casting a larger integer type to a smaller one keeps only the lowest bits that fit, which can produce results that look unrelated to the original value if it did not fit the smaller range.
- A cast operator has high precedence and binds only to the single value or parenthesized expression immediately following it, not to an entire longer expression.
- Widening a `long` into a `float` or `double` can still silently lose precision even though no explicit cast is required, because `float` and `double` prioritize range over exact integer precision beyond a certain magnitude.
- Unboxing a `null` wrapper reference into a primitive throws a `NullPointerException` at runtime; this is a boxing related runtime error, distinct from any purely primitive to primitive casting rule.

### 5.5 Scenario Based Understanding

**Scenario A.** A program calculates a monetary total as a `double`, then casts it directly to `int` to obtain a whole number of cents for storage, and a later question asks why the stored total is sometimes one cent less than expected.
- What is happening: the cast truncates rather than rounds.
- Which concept is involved: narrowing floating point to integer truncation.
- How to identify it: notice the direct `(int)` cast applied to a `double` monetary computation without any prior rounding step such as `Math.round`.
- Correct reasoning: if the true fractional cents value is something like `249.999999` due to floating point imprecision, truncation produces `249`, one less than the intended `250`, so `Math.round` should be used before narrowing in monetary or precision sensitive contexts.
- Common mistake: assuming `(int)` behaves like standard mathematical rounding, when it always truncates toward zero regardless of how close the fractional part is to the next whole number.

**Scenario B.** A question shows a sensor reading stored as an `int` that occasionally receives values above 300 due to a hardware glitch, and the code narrows this `int` directly into a `byte` field for compact storage, then asks what values might be observed after the glitch.
- What is happening: values outside the `byte` range wrap around using low order bits, not clamp.
- Which concept is involved: integer to integer narrowing conversion.
- How to identify it: recognize the source and destination are both integer types, not a floating point source, which rules out clamping behavior.
- Correct reasoning: compute the value modulo 256 semantics using two's complement wraparound to determine the actual stored `byte` value, which can appear as an unrelated, possibly negative, number compared to the original glitchy reading.
- Common mistake: assuming the value simply gets clamped to `Byte.MAX_VALUE`, confusing integer to integer narrowing behavior with the different clamping rule that applies specifically to floating point to integer narrowing.

### 5.6 Quiz: Casting Primitives

1. Which of the following is a widening conversion that requires no explicit cast?
   A. `double` to `float`  B. `int` to `byte`  C. `int` to `long`  D. `long` to `int`

2. What is the output of `System.out.println((int) 7.8);`?
   A. `8`  B. `7`  C. `7.8`  D. Compile error

3. What is the output of `System.out.println((int) -3.2);`?
   A. `-4`  B. `-3`  C. `3`  D. Compile error

4. Given `int i = 300; byte b = (byte) i;`, what is the value of `b`?
   A. `300`  B. `44`  C. Compile error  D. `-44`

5. What is the output of `System.out.println((int) Double.POSITIVE_INFINITY);`?
   A. A very large negative number  B. `Integer.MAX_VALUE`  C. `0`  D. Compile error

6. Does `char` widen automatically to `short`?
   A. Yes, always  B. No, an explicit cast is required  C. Only for values under 128  D. Only in switch statements

7. What is printed by `System.out.println((int) 5.99 + (int) 2.99);`?
   A. `8.98`  B. `7`  C. `9`  D. `8`

8. Given `long l = 10_000_000_000L; int i = (int) l;`, is a cast required, and what category of conversion is this?
   A. No cast required; widening  B. Cast required; narrowing  C. No cast required; narrowing  D. Cast required; widening

9. What is the output of `System.out.println((byte) 128);`?
   A. `128`  B. `-128`  C. `127`  D. Compile error

10. Which conversion does NOT require an explicit cast?
    A. `float` to `int`  B. `double` to `long`  C. `int` to `double`  D. `long` to `int`

11. What does `(char) 65` evaluate to when printed?
    A. `65`  B. `A`  C. Compile error  D. `a`

12. What is the output of `System.out.println((int) Double.NaN);`?
    A. Compile error  B. `0`  C. Largest possible int  D. Smallest possible int

13. In `double result = (int) 4.5 * 2;`, what is the value of `result`?
    A. `9.0`  B. `8.0`  C. `9`  D. `8`

14. Which statement correctly describes casting `long` to `float`?
    A. It is narrowing and always requires an explicit cast  B. It is widening syntactically but can still lose precision  C. It is illegal in Java  D. It always throws a runtime exception if the value is large

15. What is the result of assigning `int x = 'A' + 1;` and printing `x`?
    A. `B`  B. `66`  C. Compile error  D. `A1`

### 5.7 Quiz Answers and Reasoning

1. **Answer: C, `int` to `long`.** This conversion moves strictly upward along the widening chain from a smaller integer type to a larger one that can represent every value the smaller type could, so the compiler performs it automatically. Option A goes from a larger type to a smaller one and is actually narrowing despite appearances; options B and D both move from a larger integer type to a smaller one and require explicit casts.

2. **Answer: B, `7`.** Casting a `double` to `int` truncates the fractional part entirely rather than rounding, so `7.8` becomes `7`, discarding the `.8` completely regardless of how close it is to `8`.

3. **Answer: B, `-3`.** Truncation always moves toward zero, not downward toward negative infinity. `-3.2` truncated toward zero drops the fractional `.2`, leaving `-3`, not `-4`, which would be the result of flooring rather than truncating.

4. **Answer: D, `-44`.** `300` exceeds the `byte` range of -128 to 127. Only the lowest 8 bits of `300`'s binary representation are kept when narrowing to `byte`, and interpreting those retained bits under two's complement signed rules produces `-44`, illustrating integer to integer narrowing wraparound rather than any form of clamping.

5. **Answer: B, `Integer.MAX_VALUE`.** Casting an excessively large or infinite floating point value to `int` clamps to the destination type's maximum representable value rather than wrapping around, which is the special behavior reserved specifically for floating point to integer narrowing conversions.

6. **Answer: B.** `char` and `short` are not automatically interchangeable in either direction, because `char` is unsigned across the full 16 bit range while `short` is signed, so their value ranges do not simply nest inside one another; an explicit cast is required to convert between them in either direction.

7. **Answer: B, `7`.** `(int) 5.99` truncates to `5`, and `(int) 2.99` truncates to `2`. Both casts happen before the addition due to the cast operator's high precedence, so the final addition is the pure integer sum `5 + 2`, which equals `7`.

8. **Answer: B, cast required, narrowing.** `long` is a larger type than `int` in the widening chain, so converting from `long` down to `int` moves in the narrowing direction and requires an explicit cast, which is correctly present in the given code.

9. **Answer: B, `-128`.** `128` falls just one beyond the maximum positive value a signed `byte` can hold, which is `127`. Narrowing `128` to `byte` retains only the lowest 8 bits, and the resulting bit pattern under two's complement signed interpretation is exactly `-128`, the most negative representable `byte` value.

10. **Answer: C, `int` to `double`.** This is a widening conversion moving from a smaller, less precise integer type up to a strictly larger floating point type capable of representing every `int` value, so no cast is required. All the other listed options move from a larger or more precise type down to a smaller one, requiring explicit casts.

11. **Answer: B, `A`.** The `int` value `65` corresponds to the Unicode code point for the uppercase letter `A`. Casting `65` to `char` reinterprets that numeric value as the character it represents, and printing a `char` displays the character itself rather than its numeric code point.

12. **Answer: B, `0`.** `NaN`, meaning not a number, cannot be meaningfully truncated to any specific finite integer value, so the Java Language Specification defines the result of casting `NaN` to an integer type as `0`, a special defined case distinct from ordinary truncation or clamping.

13. **Answer: B, `8.0`.** `(int) 4.5` truncates first to `4`, due to the cast's high precedence binding only to `4.5`. That `int` `4` is then multiplied by the `int` literal `2`, giving the `int` `8`. Finally, assigning that `int` `8` to a `double` variable widens it automatically to `8.0`.

14. **Answer: B.** Java classifies `long` to `float` as syntactically widening, meaning no explicit cast is required by the compiler, since `float` has a far larger representable magnitude range than `long`. However, `float`'s precision, around 6 to 7 significant decimal digits, is much lower than what is needed to represent every large `long` value exactly, so the conversion can silently round to the nearest representable `float` value, losing exact integer precision despite requiring no cast.

15. **Answer: B, `66`.** `'A' + 1` promotes the `char` `'A'`, whose Unicode code point is `65`, to `int` for the addition, producing `65 + 1`, which is `66`. Since `x` is declared as `int`, not `char`, the printed output is the numeric value `66`, not the character `B`.

### 5.8 Programming Practice: Casting Primitives

1. **Basic.** Write a program that declares a `double` value with a fractional part, casts it to `int`, and prints both the original and truncated values side by side with clear labels.
2. **Intermediate.** Write a program that takes an `int` value representing a temperature reading that may occasionally be corrupted to a value above 200, casts it to `byte`, and prints the resulting wrapped value, along with a manual explanation printed as text describing why the result looks the way it does.
3. **Intermediate.** Write a program that demonstrates the difference between truncation and proper rounding by printing, for the value `7.6`, both `(int) 7.6` and `Math.round(7.6)`, clearly labeling which technique produces which result.
4. **Advanced.** Write a program that reads a hardcoded array of `double` values representing scientific measurements, some of which exceed the `int` range in magnitude, casts every value to `int`, and prints each original value alongside its cast result, demonstrating clamping for the out of range entries.
5. **Edge case based.** Write a program that casts `Double.NaN`, `Double.POSITIVE_INFINITY`, and `Double.NEGATIVE_INFINITY` all to `int`, printing each result with a label, to concretely demonstrate all three special floating point to integer narrowing cases in one place.
6. **Logic intensive.** Write a program that takes a `long` value larger than `Integer.MAX_VALUE`, casts it down to `int`, and separately casts an `int` value up to `long`, printing all four values, clearly demonstrating both a narrowing conversion that loses information and a widening conversion that preserves it exactly.

### 5.9 Programming Solutions: Casting Primitives

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        double original = 42.987;
        int truncated = (int) original;
        System.out.println("Original: " + original);
        System.out.println("Truncated: " + truncated);
    }
}
```

Why it works: `(int) original` discards everything after the decimal point without rounding, directly demonstrating the fundamental truncation rule covered in the concept explanation.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int corruptedReading = 250;
        byte stored = (byte) corruptedReading;
        System.out.println("Original int reading: " + corruptedReading);
        System.out.println("Stored byte value: " + stored);
        System.out.println("This happens because only the lowest 8 bits of " +
                corruptedReading + " are kept, and those bits, read as a " +
                "signed byte, represent " + stored + " rather than " +
                corruptedReading + ".");
    }
}
```

Why it works: `250` exceeds the `byte` maximum of `127`. Narrowing keeps only the lowest 8 bits of `250`'s binary form, and interpreting that bit pattern as signed produces a negative wraparound result, which the printed explanation walks through in plain language.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        double value = 7.6;
        int truncatedResult = (int) value;
        long roundedResult = Math.round(value);
        System.out.println("Truncation (int) 7.6 = " + truncatedResult);
        System.out.println("Rounding Math.round(7.6) = " + roundedResult);
    }
}
```

Why it works: `(int) 7.6` simply discards the `.6`, giving `7`, while `Math.round` applies genuine mathematical rounding to the nearest whole number, giving `8`, clearly illustrating that these are two different operations that are easy to conflate. Note that `Math.round(double)` returns a `long`, which is why `roundedResult` is declared as `long` here.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        double[] measurements = {12.5, -999999999999.9, 3.14159, 5e15, -42.0};

        for (double measurement : measurements) {
            int castResult = (int) measurement;
            System.out.println(measurement + " -> " + castResult);
        }
    }
}
```

Thought process: the array intentionally mixes ordinary sized values with two extreme magnitude values, one very large positive and one very large negative, in order to visibly demonstrate clamping toward `Integer.MAX_VALUE` and `Integer.MIN_VALUE` respectively for the entries that exceed the `int` range.

Why it works: ordinary values simply truncate as expected, while the two extreme entries clamp to the boundary `int` values rather than wrapping around, correctly demonstrating the special floating point to integer narrowing rule.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        System.out.println("NaN cast to int: " + (int) Double.NaN);
        System.out.println("Positive infinity cast to int: " + (int) Double.POSITIVE_INFINITY);
        System.out.println("Negative infinity cast to int: " + (int) Double.NEGATIVE_INFINITY);
    }
}
```

Expected output:
```
NaN cast to int: 0
Positive infinity cast to int: 2147483647
Negative infinity cast to int: -2147483648
```

Why it works: these are exactly the three special defined cases for floating point to integer narrowing specified by the Java Language Specification: `NaN` becomes `0`, positive infinity clamps to `Integer.MAX_VALUE`, and negative infinity clamps to `Integer.MIN_VALUE`.

**Solution 6.**

```java
public class Solution6 {
    public static void main(String[] args) {
        long bigLong = 5_000_000_000L;
        int narrowedInt = (int) bigLong;

        int smallInt = 123456;
        long widenedLong = smallInt;

        System.out.println("Original long: " + bigLong);
        System.out.println("Narrowed to int: " + narrowedInt);
        System.out.println("Original int: " + smallInt);
        System.out.println("Widened to long: " + widenedLong);
    }
}
```

Thought process: `5_000_000_000L` exceeds `Integer.MAX_VALUE`, chosen specifically so the narrowing cast visibly loses information, while the second half of the program shows the opposite direction, a small `int` widened into a `long`, which preserves the value exactly with no cast required at all.

Why it works: narrowing `long` to `int` keeps only the lowest 32 bits, which for a value like `5_000_000_000L` produces a result that bears little obvious resemblance to the original number, while widening `int` to `long` always succeeds exactly, since every `int` value fits perfectly within the much larger `long` range with no possible loss of information.

---

## 6. Arithmetic, Unary and Relational Operators in Java

### 6.1 Concept Explanation

Operators are special symbols that perform operations on one or more operands to produce a result. This section covers three closely related families: arithmetic operators, which perform mathematical calculations; unary operators, which act on a single operand; and relational operators, which compare two values and produce a boolean result.

**Arithmetic operators.** Java provides five arithmetic operators: `+` for addition, `-` for subtraction, `*` for multiplication, `/` for division, and `%` for remainder, sometimes called modulus.

- **Addition and subtraction** behave as expected for both integer and floating point operands.
- **Multiplication** behaves as expected, though it is worth remembering that multiplying two large `int` values can overflow just as easily, and often more suddenly, than addition, since multiplication grows magnitude far faster.
- **Division** behaves very differently depending on operand types. Integer division between two integer types truncates toward zero and discards any remainder entirely, so `7 / 2` produces `3`, not `3.5`. Division where at least one operand is a floating point type produces a floating point result with the fractional part retained, so `7.0 / 2` produces `3.5`. Division by the integer zero throws an `ArithmeticException` at runtime for integer operands, but division by zero for floating point operands does not throw an exception; instead it produces `Infinity`, `-Infinity`, or `NaN` depending on the exact values involved, since IEEE 754 floating point arithmetic defines these special results.
- **The remainder operator `%`** returns what is left over after integer division. For positive operands this matches everyday intuition, but for negative operands, Java's `%` result takes the sign of the dividend, the left operand, not the divisor. So `-7 % 2` produces `-1`, and `7 % -2` produces `1`. The remainder operator also works on floating point operands in Java, following IEEE 754 remainder semantics, which is a lesser known but occasionally tested capability.

**String concatenation with `+`.** When either operand of the `+` operator is a `String`, the operator performs string concatenation instead of arithmetic, converting the other operand to its string representation first. This overlapping use of the same symbol for two entirely different operations, arithmetic addition and string concatenation, is one of the richest sources of tricky exam output questions, especially because evaluation proceeds strictly left to right, so the position of a `String` operand within a longer chain of `+` operators changes the entire outcome.

**Unary operators.** These operate on a single operand.

- **Unary plus `+`** simply returns the operand's value unchanged; it exists mostly for symmetry and readability and is rarely functionally significant.
- **Unary minus `-`** negates the operand's numeric value.
- **Logical complement `!`** inverts a `boolean` value, turning `true` into `false` and vice versa; this belongs conceptually with logical operators but is unary in arity.
- **Increment `++`** and **decrement `--`** each add or subtract exactly `1` from a variable. Both come in two forms: prefix, written before the variable such as `++x`, which increments the variable first and then yields the new value as the expression's result, and postfix, written after the variable such as `x++`, which yields the variable's original value as the expression's result and then increments the variable afterward. This prefix versus postfix distinction, and exactly when the increment takes effect relative to the surrounding expression being evaluated, is an extremely heavily tested area.

**Relational operators.** These compare two operands and always produce a `boolean` result: `==` for equality, `!=` for inequality, `<` for less than, `>` for greater than, `<=` for less than or equal to, and `>=` for greater than or equal to. For primitive numeric and character types, these compare actual values directly. It is critical to remember that `==` on reference types, which is outside the scope of primitives but frequently confused with primitive behavior on exams, compares object references rather than object content, though that full discussion belongs to a later object oriented topic and is only mentioned here as a boundary to be aware of.

### 6.2 Exam Perspective

1. **Direct conceptual questions** ask you to state the result type and value of a basic arithmetic expression involving mixed operand types.
2. **Output based questions** are extremely common here, showing chains of `+` mixing numeric and `String` operands, testing whether you correctly trace left to right evaluation.
3. **Code tracing questions** trace the exact sequence of increments and decrements in an expression using both prefix and postfix forms of the same variable multiple times.
4. **Error identification and debugging questions** show integer division by a literal `0` and ask whether it throws an exception or produces `Infinity`, testing whether you distinguish integer division by zero from floating point division by zero.
5. **Scenario based questions** describe a real calculation, such as splitting a bill or computing an average, where using integer division where floating point division was intended produces silently wrong results, testing your ability to spot this common real world bug pattern.
6. **Tricky or misleading questions** test the sign behavior of `%` with negative operands, since intuition from pure mathematics about modulus does not always match Java's definition, which follows the sign of the dividend.
7. **Questions combining multiple concepts** combine increment or decrement operators with array indexing, such as `arr[i++] = i;`, requiring you to determine the precise order in which the index expression and the assignment target are evaluated.
8. **Edge case questions** test what happens when `++` or `--` is applied to a variable already at its type's boundary value, such as incrementing `Integer.MAX_VALUE`, connecting back to overflow behavior from the primitive types section.

### 6.3 Examples

**Example 1: Integer division versus floating point division.**

```java
public class Demo {
    public static void main(String[] args) {
        System.out.println(7 / 2);
        System.out.println(7.0 / 2);
        System.out.println(7 / 2.0);
    }
}
```

Output:
```
3
3.5
3.5
```

Explanation: `7 / 2` uses two `int` operands, so the result is truncated integer division, discarding the remainder. As soon as either operand is a floating point type, the entire division promotes to floating point arithmetic, producing the precise fractional result.

**Example 2: Remainder with negative operands.**

```java
public class Demo {
    public static void main(String[] args) {
        System.out.println(-7 % 2);
        System.out.println(7 % -2);
        System.out.println(-7 % -2);
    }
}
```

Output:
```
-1
1
-1
```

Explanation: in each case the result's sign matches the sign of the dividend, the left hand operand, regardless of the divisor's sign. `-7 % 2` is `-1` because `-7` is negative; `7 % -2` is `1` because `7` is positive; `-7 % -2` is `-1` because `-7` is negative.

**Example 3: String concatenation order dependence.**

```java
public class Demo {
    public static void main(String[] args) {
        System.out.println(1 + 2 + "3");
        System.out.println("1" + 2 + 3);
        System.out.println(1 + "2" + 3);
    }
}
```

Output:
```
33
123
123
```

Explanation: `+` associates left to right. In the first line, `1 + 2` is pure integer addition since both are numeric, producing `3`, which is then concatenated with `"3"` to give `"33"`. In the second line, `"1" + 2` concatenates immediately since a `String` operand is present first, producing `"12"`, and appending `3` gives `"123"`. In the third line, `1 + "2"` concatenates to `"12"` first, and appending `3` gives `"123"`; note this differs from the first line purely because of where the `String` literal sits in the chain.

**Example 4: Prefix versus postfix increment.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = 5;
        int a = x++;
        int y = 5;
        int b = ++y;
        System.out.println("x=" + x + " a=" + a);
        System.out.println("y=" + y + " b=" + b);
    }
}
```

Output:
```
x=6 a=5
y=6 b=6
```

Explanation: `x++` yields `x`'s original value, `5`, as the result of the expression assigned to `a`, and only afterward increments `x` itself to `6`. `++y` increments `y` first, to `6`, and yields that already incremented value as the result of the expression assigned to `b`.

**Example 5: A single variable modified multiple times within one expression.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = 5;
        int result = x++ + ++x;
        System.out.println("x=" + x + " result=" + result);
    }
}
```

Output: `x=7 result=12`

Explanation: `x++` yields `5` and then increments `x` to `6`. Next, `++x` increments `x` from `6` to `7` and yields `7`. The addition therefore computes `5 + 7`, which is `12`, and `x` ends at `7` after both operations have taken effect.

### 6.4 Important Notes

- Integer division truncates toward zero and discards any remainder; division by zero using two integer operands throws `ArithmeticException` at runtime.
- Floating point division by zero never throws; it produces `Infinity`, `-Infinity`, or `NaN` depending on the numerator's sign and value.
- The `%` operator's result always takes the sign of the dividend, the left operand, in Java.
- `+` performs string concatenation whenever either operand is a `String`; otherwise it performs numeric addition, and evaluation strictly proceeds left to right.
- Prefix `++x` and `--x` change the variable and then yield the new value; postfix `x++` and `x--` yield the original value and then change the variable.
- A variable can be modified more than once by increment or decrement operators within a single expression, and the order of evaluation, strictly left to right for the operands of most operators, determines the final result.
- Unary `+` has virtually no practical effect on a value's magnitude or sign; do not confuse it with binary `+` used for addition or concatenation.
- Relational operators always produce a `boolean` result and can only be applied to compatible, comparable operand types.

### 6.5 Scenario Based Understanding

**Scenario A.** A billing program computes `int average = totalCost / numberOfItems;` and a later question asks why the displayed average always appears to be a whole number even when the true average should have a fractional part.
- What is happening: both operands are `int`, so the division truncates.
- Which concept is involved: integer division.
- How to identify it: check the declared types of both `totalCost` and `numberOfItems`.
- Correct reasoning: to preserve the fractional part, at least one operand must be promoted to a floating point type before the division, for example by writing `(double) totalCost / numberOfItems`.
- Common mistake: casting the entire final result after the division has already happened, such as `(double) (totalCost / numberOfItems)`, which does not fix the problem since the truncation has already occurred inside the parentheses before the cast is even applied.

**Scenario B.** A question shows `arr[i] = i++;` inside a loop and asks which array index actually receives the assignment, the original or incremented value of `i`.
- What is happening: the index expression on the left side and the value expression on the right side both involve `i`, with a postfix increment.
- Which concept is involved: order of evaluation combined with postfix increment semantics.
- How to identify it: recognize that Java evaluates the array reference and index on the left hand side before evaluating the right hand side expression, and that postfix `i++` yields the original value of `i` as its result.
- Correct reasoning: the index used for `arr[i]` is `i`'s value at the time the left side is evaluated, which is the original, pre increment value, and the value stored is also that same original value, since `i++` yields the original value; `i` itself is incremented as a side effect but that new value is not what gets used in this particular statement.
- Common mistake: assuming the incremented value of `i` is used for either the index or the assigned value, confusing postfix increment's "yield old value" rule with prefix increment's "yield new value" rule.

### 6.6 Quiz: Arithmetic, Unary and Relational Operators

1. What is the output of `System.out.println(9 / 4);`?
   A. `2.25`  B. `2`  C. `3`  D. Compile error

2. What is the output of `System.out.println(9.0 / 4);`?
   A. `2`  B. `2.25`  C. `2.5`  D. Compile error

3. What happens when `System.out.println(5 / 0);` executes?
   A. Prints `Infinity`  B. Prints `0`  C. Throws `ArithmeticException` at runtime  D. Compile error

4. What happens when `System.out.println(5.0 / 0);` executes?
   A. Throws `ArithmeticException`  B. Prints `Infinity`  C. Compile error  D. Prints `0.0`

5. What is the value of `-9 % 4`?
   A. `1`  B. `-1`  C. `3`  D. `-3`

6. What is the output of `System.out.println("Result: " + 2 + 3);`?
   A. `Result: 5`  B. `Result: 23`  C. Compile error  D. `5Result: `

7. What is the output of `System.out.println(2 + 3 + "Result");`?
   A. `Result: 5`  B. `5Result`  C. `23Result`  D. Compile error

8. Given `int x = 10; int y = x--;`, what are the final values of `x` and `y`?
   A. `x=9, y=10`  B. `x=9, y=9`  C. `x=10, y=9`  D. `x=10, y=10`

9. Given `int x = 10; int y = --x;`, what are the final values of `x` and `y`?
   A. `x=9, y=10`  B. `x=9, y=9`  C. `x=10, y=9`  D. `x=10, y=10`

10. What is the output of the following?
    ```java
    int a = 4;
    int b = a++ + a++;
    System.out.println(a + " " + b);
    ```
    A. `6 8`  B. `6 9`  C. `5 9`  D. `6 10`

11. What is the value of `10 % -3`?
    A. `-1`  B. `1`  C. `-2`  D. `2`

12. What does the unary `+` operator do to `int x = -5;` when written as `+x`?
    A. Converts it to `5`  B. Has no effect on the value, still `-5`  C. Compile error  D. Converts it to `0`

13. What is the output of `System.out.println(1 + 2 + "" + 3 + 4);`?
    A. `1234`  B. `33` followed by `4` as `334`  C. `10`  D. `1+2+3+4`

14. Given `int i = 5; int result = i++ + i;`, what is the value of `result`?
    A. `10`  B. `11`  C. `12`  D. `9`

15. Which relational expression correctly checks that `x` is strictly between `5` and `10`, exclusive on both ends?
    A. `x > 5 & x < 10`  B. `x > 5 && x < 10`  C. `5 < x < 10`  D. `x >= 5 && x <= 10`

### 6.7 Quiz Answers and Reasoning

1. **Answer: B, `2`.** Both `9` and `4` are `int` literals, so `9 / 4` performs integer division, truncating the true mathematical result of `2.25` down to `2` by discarding the fractional part entirely.

2. **Answer: B, `2.25`.** Since `9.0` is a `double` literal, the division promotes to floating point arithmetic, preserving the exact fractional result of `2.25` rather than truncating it.

3. **Answer: C, throws `ArithmeticException` at runtime.** Integer division by the literal `0` is a well defined runtime error condition in Java, distinct from floating point division by zero; the program compiles successfully but fails when this specific line actually executes.

4. **Answer: B, prints `Infinity`.** Floating point division by zero follows IEEE 754 rules rather than throwing an exception. A positive finite numerator divided by zero produces positive infinity, which Java's `Double.toString` representation prints as the literal text `Infinity`.

5. **Answer: B, `-1`.** The result of `%` always takes the sign of the dividend, which here is `-9`, a negative number. The magnitude of the remainder, ignoring sign, is `1`, since `-9` divided by `4` gives a quotient of `-2` with a remainder of `-1` when computed consistently with truncating division, and the sign matches the dividend, giving `-1`.

6. **Answer: B, `Result: 23`.** Evaluation proceeds strictly left to right. `"Result: " + 2` concatenates immediately, since a `String` operand is present, producing `"Result: 2"`. Appending `3` continues the concatenation chain, producing `"Result: 23"`, rather than performing any numeric addition at all once a `String` has entered the chain.

7. **Answer: B, `5Result`.** Here `2 + 3` is evaluated first as pure numeric addition, since neither operand is a `String` at that point in the left to right scan, producing the `int` `5`. Only after that does `5 + "Result"` concatenate, since a `String` operand now appears, producing `"5Result"`. This is the mirror image of Example 3 above: because the numeric operands both come before the `String` literal here, they combine arithmetically first, unlike a chain such as `"Result: " + 2 + 3`, where the `String` appears at the very start and every `+` after it concatenates instead.

8. **Answer: C, `x=9, y=10`.** Postfix `x--` yields the original value of `x`, `10`, which is assigned to `y`, and only afterward decrements `x` to `9`.

9. **Answer: B, `x=9, y=9`.** Prefix `--x` decrements `x` first, from `10` to `9`, and yields that new, already decremented value, which is then assigned to `y`.

10. **Answer: B, `6 9`.** The first `a++` yields `4`, the current value of `a`, and increments `a` to `5`. The second `a++` yields `5`, the now current value of `a`, and increments `a` to `6`. The sum `b` is `4 + 5`, which is `9`, and the final value of `a` after both postfix increments is `6`.

11. **Answer: B, `1`.** The dividend `10` is positive, so the result's sign is positive regardless of the divisor's sign. The magnitude of the remainder after dividing `10` by `-3` is `1`, since `-3` times `-3` is `9`, leaving a remainder of `1`, and the sign matches the positive dividend, giving `1`.

12. **Answer: B, has no effect, still `-5`.** Unary `+` in Java simply returns its operand's value unchanged; it does not negate, flip sign, or perform any transformation, unlike unary `-` which does negate.

13. **Answer: B, `334`.** Trace it one operator at a time, strictly left to right: `1 + 2` performs numeric addition first, since both are numeric, giving `3`. Then `3 + ""` concatenates, since a `String` operand now appears, producing the text `"3"`. Then `"3" + 3` concatenates again, producing `"33"`. Finally `"33" + 4` concatenates once more, producing `"334"`. The empty string literal `""` is exactly what flips the rest of the chain from arithmetic into concatenation, which is precisely the kind of small detail this style of question is designed to test; skipping any single intermediate step, rather than tracing every one individually, is exactly how this type of question causes mistakes even for well prepared candidates.

14. **Answer: B, `11`.** `i++` yields the original value of `i`, `5`, and then increments `i` to `6` as a side effect. The right hand `i` in `i++ + i` is then evaluated after that increment has already taken effect, reading `6`. The sum is therefore `5 + 6`, which is `11`.

15. **Answer: B.** Java has no chained comparison syntax like `5 < x < 10`, which is actually a compile error since `5 < x` produces a `boolean`, and a `boolean` cannot then be compared with `<` against an `int`. The correct way to express a range check requires the logical AND operator `&&` combining two full relational expressions, as shown in option B. Option A uses the single ampersand `&`, which is also technically valid for combining two `boolean` expressions but does not short circuit, making `&&` the more idiomatic and generally preferred choice, so B best represents standard practice, while D describes an inclusive rather than the requested exclusive range.

**A note on items 7 and 13 above:** these two questions were deliberately included, worked through with visible self correction in the reasoning, to model exactly the kind of careful, step by step left to right tracing discipline that competitive exams reward. If you found yourself initially trusting the first instinctive answer choice rather than tracing character by character, treat that as a signal to slow down specifically on `+` chains mixing `String` and numeric operands during your actual exam attempt.

### 6.8 Programming Practice: Arithmetic, Unary and Relational Operators

1. **Basic.** Write a program that takes two hardcoded `int` values and prints the results of all five arithmetic operators applied to them, each on its own labeled line.
2. **Intermediate.** Write a program that correctly computes and prints the true floating point average of five hardcoded `int` exam scores, being careful to avoid the integer division pitfall.
3. **Intermediate.** Write a program demonstrating and printing the difference between `x++ + x++` and `++x + ++x` starting from the same initial value of `x` each time, using two entirely separate variables so the two expressions do not interfere with each other, and clearly label both the intermediate variable states and the final results.
4. **Advanced.** Write a program that simulates a simple digital clock's minute rollover using `%`, where given a starting minute value and a number of minutes to add, it correctly prints the resulting minute value on a 60 minute clock face, handling the case where the addition would exceed 59.
5. **Edge case based.** Write a program that demonstrates all four sign combinations of the `%` operator, positive dividend with positive divisor, positive with negative, negative with positive, and negative with negative, printing all four results with labels confirming the dividend sign rule.
6. **Logic intensive.** Write a program that uses only relational and logical operators, no `if` statements, to compute and print a `boolean` indicating whether a given hardcoded `int` age qualifies as a teenager, defined as being between 13 and 19 inclusive.

### 6.9 Programming Solutions: Arithmetic, Unary and Relational Operators

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int a = 17;
        int b = 5;
        System.out.println("a + b = " + (a + b));
        System.out.println("a - b = " + (a - b));
        System.out.println("a * b = " + (a * b));
        System.out.println("a / b = " + (a / b));
        System.out.println("a % b = " + (a % b));
    }
}
```

Why it works: parentheses around each arithmetic expression ensure the arithmetic is computed first before being concatenated to the surrounding label string, avoiding the left to right `+` chain ambiguity discussed extensively above.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int score1 = 88, score2 = 91, score3 = 76, score4 = 95, score5 = 82;
        int total = score1 + score2 + score3 + score4 + score5;
        double average = total / 5.0;
        System.out.println("Total: " + total);
        System.out.println("Average: " + average);
    }
}
```

Why it works: dividing by the `double` literal `5.0` rather than the `int` literal `5` forces the division to promote to floating point arithmetic, correctly preserving the fractional part of the average rather than truncating it.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int x1 = 3;
        int postfixResult = x1++ + x1++;
        System.out.println("Postfix case: final x1 = " + x1 + ", result = " + postfixResult);

        int x2 = 3;
        int prefixResult = ++x2 + ++x2;
        System.out.println("Prefix case: final x2 = " + x2 + ", result = " + prefixResult);
    }
}
```

Expected output:
```
Postfix case: final x1 = 5, result = 7
Prefix case: final x2 = 5, result = 9
```

Thought process: in the postfix case, the first `x1++` yields `3` then becomes `4`; the second `x1++` yields `4` then becomes `5`; the sum is `3 + 4`, which is `7`. In the prefix case, the first `++x2` becomes `4` and yields `4`; the second `++x2` becomes `5` and yields `5`; the sum is `4 + 5`, which is `9`. Both final variable values end at `5` since each expression performs two increments total starting from `3`, but the summed results differ because of when each increment's effect becomes visible to the addition.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        int currentMinute = 45;
        int minutesToAdd = 30;
        int newMinute = (currentMinute + minutesToAdd) % 60;
        System.out.println("Starting minute: " + currentMinute);
        System.out.println("Minutes to add: " + minutesToAdd);
        System.out.println("Resulting minute on clock face: " + newMinute);
    }
}
```

Expected output:
```
Starting minute: 45
Minutes to add: 30
Resulting minute on clock face: 15
```

Why it works: `%` naturally implements clock style wraparound, since `45 + 30` is `75`, and `75 % 60` correctly gives `15`, the true minute value once the excess full hour is discarded, exactly mimicking how a clock face wraps back to zero after fifty nine minutes.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        System.out.println("7 % 3 = " + (7 % 3));
        System.out.println("7 % -3 = " + (7 % -3));
        System.out.println("-7 % 3 = " + (-7 % 3));
        System.out.println("-7 % -3 = " + (-7 % -3));
    }
}
```

Expected output:
```
7 % 3 = 1
7 % -3 = 1
-7 % 3 = -1
-7 % -3 = -1
```

Why it works: in every single case, the sign of the result matches the sign of the dividend, the left operand, entirely independent of the divisor's sign, which is exactly the rule stated in the concept explanation and directly confirmed here across all four possible sign combinations.

**Solution 6.**

```java
public class Solution6 {
    public static void main(String[] args) {
        int age = 16;
        boolean isTeenager = age >= 13 && age <= 19;
        System.out.println("Age " + age + " is teenager: " + isTeenager);
    }
}
```

Why it works: `&&` combines the two relational checks, `age >= 13` and `age <= 19`, into a single `boolean` expression that is `true` only when both bounds are satisfied simultaneously, correctly and concisely expressing the inclusive range check entirely without any `if` statement, as required by the exercise.

---

## 7. Lend A Hand on Arithmetic, Unary and Relational Operators

This reinforcement block builds on the previous section with denser combinations: multiple increment or decrement operators layered together, mixed String and numeric chains inside method calls, and relational operators combined with the arithmetic results computed alongside them.

### 7.1 Applied Examples

**Applied Example 1: Multiple side effects on the same variable within one statement.**

```java
public class Demo {
    public static void main(String[] args) {
        int a = 2;
        int result = a++ + a-- + ++a + --a;
        System.out.println("a = " + a + ", result = " + result);
    }
}
```

Trace step by step, strictly left to right:
- `a++` yields `2`, then `a` becomes `3`.
- `a--` yields `3`, then `a` becomes `2`.
- `++a` makes `a` become `3`, and yields `3`.
- `--a` makes `a` become `2`, and yields `2`.
- Sum: `2 + 3 + 3 + 2 = 10`.

Output: `a = 2, result = 10`

**Applied Example 2: Relational operators built directly on arithmetic sub expressions.**

```java
public class Demo {
    public static void main(String[] args) {
        int score = 78;
        int bonus = 5;
        boolean passesWithBonus = (score + bonus) >= 80;
        boolean passesWithoutBonus = score >= 80;
        System.out.println("With bonus: " + passesWithBonus);
        System.out.println("Without bonus: " + passesWithoutBonus);
    }
}
```

Output:
```
With bonus: true
Without bonus: false
```

Explanation: the arithmetic sub expression `(score + bonus)` is fully evaluated to `83` before the relational comparison `>= 80` is applied, demonstrating that arithmetic operators bind tighter than relational operators, a precedence rule that will be formalized in the upcoming Operator Precedence section.

### 7.2 Quiz: Lend A Hand on Arithmetic, Unary and Relational Operators

1. Given `int a = 10; int b = a-- - --a;`, what is the value of `b`?
   A. `2`  B. `1`  C. `0`  D. `-1`

2. What is the output of `System.out.println(5 + 5 + "" + 5 + 5);`?
   A. `1010`  B. `5555`  C. `1055`  D. `10`

3. Given `int x = 4; boolean result = (x++ == 4) && (x == 5);`, what is `result`?
   A. `true`  B. `false`  C. Compile error  D. Depends on evaluation order

4. What is the value of `15 % 4 + 15 / 4`?
   A. `6`  B. `7`  C. `3.75`  D. `4`

5. Given `int p = 3; int q = p++ * ++p;`, what is `q`?
   A. `12`  B. `15`  C. `16`  D. `9`

6. Which expression correctly checks if a number `n` is even using only arithmetic and relational operators?
   A. `n / 2 == 0`  B. `n % 2 == 0`  C. `n % 2 = 0`  D. `n / 2 = 0`

7. What is the output of `System.out.println(10 > 5 == 3 < 8);`?
   A. `true`  B. `false`  C. Compile error  D. `1`

### 7.3 Quiz Answers and Reasoning

1. **Answer: A, `2`.** `a--` yields the original value `10`, then `a` becomes `9`. `--a` then decrements `a` from `9` to `8` and yields `8`. So `b = 10 - 8 = 2`.

2. **Answer: C, `1055`.** `5 + 5` performs numeric addition first, giving `10`, since both operands are numeric at that point in the left to right scan. `10 + ""` then concatenates, since a `String` operand now appears, giving `"10"`. `"10" + 5` concatenates again, giving `"105"`. `"105" + 5` concatenates once more, giving `"1055"`. As in the previous section's examples, the position of the empty string literal is exactly what flips the rest of the chain from arithmetic into concatenation.

3. **Answer: A, `true`.** `x++ == 4` first compares the original value of `x`, `4`, against `4`, which is `true`, and only afterward increments `x` to `5` as a side effect. By the time `x == 5` is evaluated as the second operand of `&&`, `x` has already become `5`, making that comparison also `true`. Both sides being `true` makes the overall `&&` result `true`.

4. **Answer: A, `6`.** `15 % 4` is `3`, since `4 × 3 = 12` leaves a remainder of `3`. `15 / 4` is `3` using integer division, truncating the `.75` entirely. The sum `3 + 3` is `6`. Always compute each sub expression independently and write down the intermediate result before combining them, since combining two separately simple calculations too quickly is exactly where arithmetic slips happen under exam time pressure.

5. **Answer: B, `15`.** `p++` yields the original value `3` and then increments `p` to `4` as a side effect. `++p` then increments `p` from `4` to `5` and yields that new value, `5`. The multiplication is therefore `3 * 5`, which is `15`.

6. **Answer: B.** `n % 2 == 0` correctly tests whether `n` leaves no remainder when divided by `2`, which is the standard definition of an even number, and correctly uses `==` for comparison rather than the assignment operator `=` used incorrectly in options C and D.

7. **Answer: A, `true`.** `10 > 5` evaluates to `true`, and `3 < 8` evaluates to `true`, both due to relational operators having higher precedence than `==`. The comparison then becomes `true == true`, which evaluates to `true`.

**A note on this quiz section:** questions 2, 4, and 5 above all hinge on tracing a multi step expression one operator at a time rather than computing a final answer in a single mental pass. This is precisely the skill a competitive exam rewards: write out each intermediate value explicitly, in order, before committing to a final answer choice, especially whenever a String literal, an empty string, or repeated increment and decrement operators appear inside the same expression.

### 7.4 Programming Practice: Lend A Hand on Arithmetic, Unary and Relational Operators

1. Write a program that starts with `int n = 5;` and computes `n-- - --n + n++ + ++n` in a single statement, printing both the final value of `n` and the computed result, then add comments tracing every single sub step.
2. Write a program that checks, using only arithmetic and relational operators combined with `&&` and `||`, whether a hardcoded year is a leap year, using the standard rule that a year is a leap year if it is divisible by 4 and not divisible by 100, unless it is also divisible by 400.
3. Write a program that builds and prints a single concatenated summary sentence mixing at least four numeric sub expressions and plain text, deliberately ordered so that at least one pair of adjacent numeric values combines arithmetically before any concatenation begins, and at least one pair concatenates directly, then explain in a comment which parts did which.

### 7.5 Programming Solutions: Lend A Hand on Arithmetic, Unary and Relational Operators

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int n = 5;
        // n-- yields 5, n becomes 4
        // --n makes n become 3, yields 3
        // n++ yields 3, n becomes 4
        // ++n makes n become 5, yields 5
        // result = 5 - 3 + 3 + 5 = 10
        int result = n-- - --n + n++ + ++n;
        System.out.println("Final n = " + n + ", result = " + result);
    }
}
```

Why it works: each sub step is commented individually before the statement executes, matching the discipline recommended throughout this section: never trust a single mental pass over a multi operator expression involving repeated increments and decrements on the same variable.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int year = 2024;
        boolean isLeapYear = (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0);
        System.out.println(year + " is a leap year: " + isLeapYear);
    }
}
```

Why it works: the expression directly mirrors the standard leap year rule using relational operators for each divisibility check via `%`, combined with `&&` for the "divisible by 4 but not by 100" branch and `||` to allow the "divisible by 400" exception to independently qualify a year as a leap year even when the first branch would exclude it.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int apples = 3;
        int oranges = 4;
        // apples + oranges combines numerically first, since both are int, giving 7,
        // before the following string literal ever enters the chain.
        String summary = "You have " + (apples + oranges) + " pieces of fruit, made up of "
                + apples + " apples and " + oranges + " oranges.";
        System.out.println(summary);
    }
}
```

Why it works: wrapping `(apples + oranges)` in explicit parentheses guarantees the numeric addition happens before any concatenation touches those two values, removing any ambiguity about evaluation order, while the remaining `+` operators concatenate `apples` and `oranges` individually as plain numbers into the surrounding sentence text.

---

## 8. Logical and Bitwise Operators

### 8.1 Concept Explanation

Java provides operators that work at two different levels: logical operators that combine or invert `boolean` values based on truth value semantics, and bitwise operators that manipulate the individual binary bits of integer operands directly. Several symbols are shared between these two families, which is a major source of exam confusion, so this section is careful to separate boolean context usage from integer context usage throughout.

**Logical operators.** These operate on `boolean` operands and produce a `boolean` result.

- **Logical AND `&&`** produces `true` only if both operands are `true`.
- **Logical OR `||`** produces `true` if at least one operand is `true`.
- **Logical NOT `!`** is unary and inverts a single `boolean` operand.
- **Short circuit evaluation** is the defining behavioral feature of `&&` and `||`. For `&&`, if the left operand evaluates to `false`, the right operand is never evaluated at all, since the overall result must be `false` regardless. For `||`, if the left operand evaluates to `true`, the right operand is never evaluated, since the overall result must be `true` regardless. This matters enormously whenever the right operand has a side effect, such as a method call that increments a counter, or an expression that could throw an exception, such as accessing an array element or dividing by a variable that might be zero.

**Non short circuit logical operators `&` and `|` on boolean operands.** Java allows `&` and `|`, which are more commonly known as bitwise operators, to also be applied directly to `boolean` operands. When used this way, they perform the same logical AND and logical OR truth table as `&&` and `||`, but critically, they always evaluate both operands, with no short circuiting whatsoever, even when the left operand alone already determines the final result.

**Bitwise operators on integer types.** When `&`, `|`, and `^` are applied to integer typed operands rather than `boolean` operands, they perform bit by bit logical operations directly on the binary representation of the values.

- **Bitwise AND `&`** sets each result bit to `1` only if both corresponding operand bits are `1`.
- **Bitwise OR `|`** sets each result bit to `1` if at least one corresponding operand bit is `1`.
- **Bitwise XOR `^`** sets each result bit to `1` if the corresponding operand bits differ from each other, and `0` if they are the same. XOR has the well known and occasionally tested property that XORing any value with itself produces `0`, and XORing any value with `0` returns that value unchanged.
- **Bitwise complement `~`** is unary and inverts every single bit of its operand, turning every `0` into a `1` and every `1` into a `0`. For a signed integer type, this has the specific numeric effect of computing `-(x + 1)`, so `~5` equals `-6`, a formula worth memorizing directly since manually flipping bits under exam time pressure is error prone.

### 8.2 Exam Perspective

1. **Direct conceptual questions** ask you to state the truth table result of `&&`, `||`, `&`, or `|` for given boolean operand combinations.
2. **Output based questions** are extremely common around short circuit evaluation, showing a method call or side effecting expression as the right operand of `&&` or `||` where the left operand alone already determines the outcome, testing whether you know the right side is skipped entirely.
3. **Debugging questions** deliberately swap `&&` for `&`, or `||` for `|`, in code containing a side effecting right operand, such as an array bounds check followed by an array access, and ask whether this introduces a bug, since `&` and `|` always evaluate both sides.
4. **Code tracing questions** trace bitwise `&`, `|`, and `^` operations on small integer values, requiring you to convert to binary, perform the bitwise operation, and convert back to decimal.
5. **Tricky or misleading questions** exploit the visual similarity between `&&` versus `&` and `||` versus `|`, testing whether you notice the single versus double symbol difference in a snippet, and correctly infer whether short circuiting applies.
6. **Scenario based questions** present a null check combined with a method call on that same reference, such as `if (obj != null && obj.getValue() > 0)`, testing whether you understand that short circuiting here is not just a performance optimization but a correctness requirement that prevents a `NullPointerException`.
7. **Questions combining multiple concepts** combine bitwise complement with signed integer representation, testing your understanding of the `~x` equals `-(x + 1)` relationship alongside two's complement representation covered in the primitive types section.
8. **XOR based questions** test the specific properties of XOR, such as its use in simple value swapping tricks or its property that any value XORed with itself is zero.

### 8.3 Examples

**Example 1: Short circuit evaluation preventing a runtime exception.**

```java
public class Demo {
    public static void main(String[] args) {
        String text = null;
        if (text != null && text.length() > 0) {
            System.out.println("Non-empty");
        } else {
            System.out.println("Empty or null");
        }
    }
}
```

Output: `Empty or null`

Explanation: since `text != null` evaluates to `false`, `&&` short circuits and never evaluates `text.length() > 0` at all, avoiding what would otherwise be a `NullPointerException` from calling `.length()` on a `null` reference. If `&` had been used instead of `&&`, both operands would always be evaluated regardless of the left side's result, causing the program to crash here.

**Example 2: Non short circuit `&` always evaluating both sides.**

```java
public class Demo {
    static int callCount = 0;

    static boolean sideEffect() {
        callCount++;
        return true;
    }

    public static void main(String[] args) {
        boolean result = false && sideEffect();
        System.out.println("callCount after &&: " + callCount);

        result = false & sideEffect();
        System.out.println("callCount after &: " + callCount);
    }
}
```

Output:
```
callCount after &&: 0
callCount after &: 1
```

Explanation: with `&&`, since the left operand is already `false`, the right side `sideEffect()` is never called, so `callCount` remains `0`. With `&`, both operands are always evaluated regardless of the left side's value, so `sideEffect()` executes, incrementing `callCount` to `1`, even though the overall result is still `false` either way.

**Example 3: Bitwise AND, OR, and XOR on small integers.**

```java
public class Demo {
    public static void main(String[] args) {
        int a = 12; // binary 1100
        int b = 10; // binary 1010
        System.out.println(a & b); // AND
        System.out.println(a | b); // OR
        System.out.println(a ^ b); // XOR
    }
}
```

Output:
```
8
14
6
```

Explanation: `1100 & 1010` gives `1000`, which is `8`. `1100 | 1010` gives `1110`, which is `14`. `1100 ^ 1010` gives `0110`, which is `6`, since XOR produces `1` only in bit positions where the two operands differ.

**Example 4: Bitwise complement.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = 5;
        System.out.println(~x);
    }
}
```

Output: `-6`

Explanation: `~x` equals `-(x + 1)` for any signed integer, so `~5` equals `-(5 + 1)`, which is `-6`. This follows directly from how flipping every bit of a two's complement representation transforms the value.

### 8.4 Important Notes

- `&&` and `||` short circuit; `&` and `|` do not, even when applied to `boolean` operands.
- Use `&&` and `||` whenever the right operand has a side effect that should be conditionally skipped, or whenever evaluating the right operand unconditionally could cause an error such as a `NullPointerException` or `ArrayIndexOutOfBoundsException`.
- `&`, `|`, and `^` behave as logical operators when applied to `boolean` operands, and as bitwise operators when applied to integer typed operands; the same symbols serve both roles depending on operand type.
- `~x` equals `-(x + 1)` for signed integer types; memorize this formula rather than manually flipping every bit under time pressure.
- XOR of any value with itself is `0`; XOR of any value with `0` returns that value unchanged; these two identities are the basis of several classic bit manipulation tricks.
- `!` inverts a `boolean` value and has no bitwise equivalent role; it cannot be applied to integer operands at all, unlike `&`, `|`, and `^`.
- Short circuiting is not merely a performance detail; it is frequently a correctness requirement, especially for guarding against `null` references or invalid indices before a dependent operation is attempted.

### 8.5 Scenario Based Understanding

**Scenario A.** A question shows `if (index >= 0 && index < array.length && array[index] > 0)` and asks what happens if `index` is negative.
- What is happening: three conditions are chained with `&&`, and the first one, `index >= 0`, fails for a negative index.
- Which concept is involved: short circuit evaluation with `&&`.
- How to identify it: notice the ordering places the safety bound checks before the actual array access.
- Correct reasoning: since `index >= 0` is `false`, `&&` short circuits immediately, and neither `index < array.length` nor `array[index] > 0` is ever evaluated, so no `ArrayIndexOutOfBoundsException` occurs; the overall expression simply evaluates to `false`.
- Common mistake: assuming all three conditions are always evaluated regardless of order, which would incorrectly suggest a crash risk here, when in fact the deliberate ordering with `&&`'s short circuiting is precisely what makes this pattern safe.

**Scenario B.** A question presents two integer flags being combined with `|` instead of `||` inside a condition that also calls a logging method as the second operand, and asks whether the log method always executes.
- What is happening: `|` is used instead of `||`, removing short circuit protection.
- Which concept is involved: non short circuit logical evaluation using the bitwise operator on boolean operands.
- How to identify it: spot the single pipe character rather than the double pipe.
- Correct reasoning: regardless of the first flag's value, the logging method on the right side of `|` is always called, since `|` does not short circuit, which may be an unintended side effect if the programmer meant to use `||` for conditional short circuit behavior.
- Common mistake: assuming `|` and `||` are interchangeable simply because they produce the same boolean truth table result, overlooking the difference in whether the right operand is unconditionally evaluated.

### 8.6 Quiz: Logical and Bitwise Operators

1. What is the output of `System.out.println(true || (5 / 0 == 0));`?
   A. Throws `ArithmeticException`  B. `true`  C. `false`  D. Compile error

2. What is the output of `System.out.println(false & (5 / 0 == 0));`?
   A. `false`  B. Throws `ArithmeticException`  C. `true`  D. Compile error

3. What is `6 & 3`?
   A. `2`  B. `7`  C. `9`  D. `0`

4. What is `6 | 3`?
   A. `2`  B. `7`  C. `9`  D. `18`

5. What is `6 ^ 3`?
   A. `2`  B. `7`  C. `5`  D. `3`

6. What is `~0`?
   A. `0`  B. `1`  C. `-1`  D. `-2`

7. Given `int callCount = 0;` and a method `check()` that increments `callCount` and returns `true`, what is `callCount` after `boolean r = true || check();`?
   A. `0`  B. `1`  C. Compile error  D. `2`

8. Given the same setup as question 7, what is `callCount` after `boolean r = true | check();`?
   A. `0`  B. `1`  C. Compile error  D. `2`

9. What does `x ^ x` evaluate to for any integer `x`?
   A. `x`  B. `0`  C. `1`  D. `-x`

10. Which operator can be applied to both `boolean` and integer operands in Java, with different behavior depending on operand type?
    A. `&&`  B. `!`  C. `&`  D. `%`

11. What is the output of the following?
    ```java
    boolean a = false;
    boolean b = true;
    System.out.println(a && b || !a);
    ```
    A. `true`  B. `false`  C. Compile error  D. `1`

12. What is `~7`?
    A. `-7`  B. `-8`  C. `8`  D. `7`

13. Given `int x = 9;` (binary `1001`) and `int y = 5;` (binary `0101`), what is `x & y`?
    A. `1`  B. `13`  C. `12`  D. `4`

14. Which statement about `!` is correct?
    A. It can be applied to both `int` and `boolean` operands  B. It can only be applied to `boolean` operands  C. It performs bitwise complement on integers  D. It short circuits like `&&`

15. What is the output of `System.out.println(10 > 5 && 5 > 20 || 3 < 4);`?
    A. `true`  B. `false`  C. Compile error  D. `10`

### 8.7 Quiz Answers and Reasoning

1. **Answer: B, `true`.** `||` short circuits: since the left operand `true` already guarantees the overall result is `true`, the right operand `5 / 0 == 0`, which would otherwise throw an `ArithmeticException`, is never evaluated at all.

2. **Answer: B, throws `ArithmeticException`.** `&` never short circuits, even with `boolean` operands, so both sides are always evaluated. The right side `5 / 0 == 0` attempts integer division by zero, which throws an `ArithmeticException` at runtime regardless of the left operand's value.

3. **Answer: A, `2`.** `6` is `0110` in binary and `3` is `0011`. Bitwise AND keeps a `1` only where both have a `1`, which occurs only in the second position from the right, giving `0010`, which is `2`.

4. **Answer: B, `7`.** `0110` OR `0011` sets a `1` wherever either operand has a `1`, giving `0111`, which is `7`.

5. **Answer: C, `5`.** `0110` XOR `0011` sets a `1` wherever the two operands differ, giving `0101`, which is `5`.

6. **Answer: C, `-1`.** Using the formula `~x = -(x + 1)`, `~0` equals `-(0 + 1)`, which is `-1`.

7. **Answer: A, `0`.** `||` short circuits: the left operand `true` already guarantees the overall result, so `check()` is never called, leaving `callCount` at its initial value of `0`.

8. **Answer: B, `1`.** `|` does not short circuit, so `check()` is always called regardless of the left operand's value, incrementing `callCount` to `1`, even though the overall boolean result would be `true` either way.

9. **Answer: B, `0`.** XOR of any value with itself always produces `0`, since every corresponding bit pair is identical, and identical bit pairs always produce `0` under XOR's "differs" rule.

10. **Answer: C, `&`.** `&` performs logical AND when applied to `boolean` operands and bitwise AND when applied to integer typed operands, making it dual purpose depending on context, unlike `&&`, which only ever applies to `boolean` operands, and `%`, which is purely an arithmetic operator.

11. **Answer: A, `true`.** `a && b` is `false && true`, which is `false`. `!a` is `!false`, which is `true`. The full expression becomes `false || true`, which evaluates to `true`, since `&&` binds tighter than `||`, so the AND portion is computed first.

12. **Answer: B, `-8`.** Using `~x = -(x + 1)`, `~7` equals `-(7 + 1)`, which is `-8`.

13. **Answer: A, `1`.** `1001` AND `0101` keeps a `1` only where both operands have a `1`, which occurs only in the rightmost position, giving `0001`, which is `1`.

14. **Answer: B.** `!` is exclusively a logical negation operator for `boolean` operands in Java; it has no defined meaning or valid usage on integer operands, unlike `&`, `|`, and `^`, which serve dual roles, and unlike `~`, which is the separate operator reserved specifically for bitwise complement of integers.

15. **Answer: A, `true`.** `&&` has higher precedence than `||`, so `10 > 5 && 5 > 20` is evaluated as one unit first: `true && false`, which is `false`. Then `false || (3 < 4)` becomes `false || true`, which evaluates to `true`.

### 8.8 Programming Practice: Logical and Bitwise Operators

1. **Basic.** Write a program that declares two `boolean` variables and prints the results of applying `&&`, `||`, and `!` to them in every combination that makes sense.
2. **Intermediate.** Write a program with a method that simulates an expensive check by printing a message every time it is called and returning a `boolean`, then demonstrate, with two separate `if` statements, one using `&&` and one using `&`, that the expensive method is skipped in one case and always called in the other, when the left operand is already `false`.
3. **Intermediate.** Write a program that takes two hardcoded `int` values, prints their binary representations using `Integer.toBinaryString`, and then prints the results of `&`, `|`, and `^` applied to them, also shown in binary, so the bit level pattern is directly visible.
4. **Advanced.** Write a program that swaps the values of two `int` variables using only XOR operations, without using a third temporary variable, and prints the values before and after the swap to confirm correctness.
5. **Edge case based.** Write a program that safely checks whether the fifth character of a `String` variable is a vowel, using short circuit `&&` to first confirm the string's length is at least five characters before attempting to access that character, demonstrating the pattern for two different input strings, one long enough and one too short.

### 8.9 Programming Solutions: Logical and Bitwise Operators

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        boolean p = true;
        boolean q = false;
        System.out.println("p && q = " + (p && q));
        System.out.println("p || q = " + (p || q));
        System.out.println("!p = " + (!p));
        System.out.println("!q = " + (!q));
    }
}
```

Why it works: this directly exercises the fundamental truth table behavior of `&&`, `||`, and `!` on a representative mixed `true` and `false` pair, giving a complete, self checkable reference of all four basic outcomes.

**Solution 2.**

```java
public class Solution2 {
    static boolean expensiveCheck() {
        System.out.println("expensiveCheck() was called");
        return true;
    }

    public static void main(String[] args) {
        boolean flag = false;

        System.out.println("Using &&:");
        if (flag && expensiveCheck()) {
            System.out.println("Branch taken");
        } else {
            System.out.println("Branch not taken");
        }

        System.out.println("Using &:");
        if (flag & expensiveCheck()) {
            System.out.println("Branch taken");
        } else {
            System.out.println("Branch not taken");
        }
    }
}
```

Expected output:
```
Using &&:
Branch not taken
Using &:
expensiveCheck() was called
Branch not taken
```

Why it works: with `flag` already `false`, `&&` short circuits and never prints the "was called" message, while `&` always evaluates both operands regardless, so the message prints even though the overall branch outcome, `false` either way, ends up identical in both cases.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int a = 22;
        int b = 15;
        System.out.println("a in binary: " + Integer.toBinaryString(a));
        System.out.println("b in binary: " + Integer.toBinaryString(b));
        System.out.println("a & b = " + (a & b) + " (" + Integer.toBinaryString(a & b) + ")");
        System.out.println("a | b = " + (a | b) + " (" + Integer.toBinaryString(a | b) + ")");
        System.out.println("a ^ b = " + (a ^ b) + " (" + Integer.toBinaryString(a ^ b) + ")");
    }
}
```

Why it works: `Integer.toBinaryString` converts each value, including each bitwise result, into its binary text representation, making the bit level effect of `&`, `|`, and `^` directly visible and verifiable alongside their decimal results.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        int x = 17;
        int y = 42;
        System.out.println("Before: x = " + x + ", y = " + y);

        x = x ^ y;
        y = x ^ y;
        x = x ^ y;

        System.out.println("After: x = " + x + ", y = " + y);
    }
}
```

Expected output:
```
Before: x = 17, y = 42
After: x = 42, y = 17
```

Thought process: this is a classic bit manipulation trick relying entirely on the two XOR identities from the concept explanation, that a value XORed with itself is `0` and a value XORed with `0` is unchanged. After the first line, `x` holds `17 ^ 42`. After the second line, `y` becomes `(17 ^ 42) ^ 42`, which simplifies to `17`, since the `42`s cancel out via the self XOR identity. After the third line, `x` becomes `(17 ^ 42) ^ 17`, which simplifies to `42` by the same cancellation logic.

Why it works: each step algebraically cancels one of the original values back out using XOR's self inverse property, ultimately achieving a full swap without ever needing a third temporary storage variable.

Common incorrect approach: attempting this without carefully tracking that `y` must be reassigned before the final line recomputes `x`, since reversing the order of the last two lines would not correctly complete the swap.

**Solution 5.**

```java
public class Solution5 {
    static void checkFifthCharVowel(String text) {
        if (text.length() >= 5 && isVowel(text.charAt(4))) {
            System.out.println("\"" + text + "\": fifth character is a vowel.");
        } else {
            System.out.println("\"" + text + "\": fifth character is not a vowel, or string too short.");
        }
    }

    static boolean isVowel(char c) {
        c = Character.toLowerCase(c);
        return c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u';
    }

    public static void main(String[] args) {
        checkFifthCharVowel("Banana");
        checkFifthCharVowel("Sky");
    }
}
```

Expected output:
```
"Banana": fifth character is a vowel.
"Sky": fifth character is not a vowel, or string too short.
```

Why it works: `text.length() >= 5` is checked first, and because `&&` short circuits, `text.charAt(4)` is only ever attempted when the length check has already confirmed it is safe to do so; for `"Sky"`, which has only three characters, the length check fails, `&&` short circuits, and `charAt(4)` is never called, correctly avoiding a `StringIndexOutOfBoundsException` that would otherwise occur.

---

## 9. Lend A Hand on Logical and Bitwise Operators

### 9.1 Applied Examples

**Applied Example 1: Combining short circuiting with a guarded division.**

```java
public class Demo {
    public static void main(String[] args) {
        int divisor = 0;
        int numerator = 10;
        if (divisor != 0 && (numerator / divisor) > 2) {
            System.out.println("Ratio exceeds 2");
        } else {
            System.out.println("Cannot safely compute or ratio too small");
        }
    }
}
```

Output: `Cannot safely compute or ratio too small`

Explanation: `divisor != 0` is `false`, so `&&` short circuits and never attempts `numerator / divisor`, which would otherwise throw an `ArithmeticException`. This is the same guard pattern as the array bounds scenario in the previous section, generalized to a division by zero risk.

**Applied Example 2: Bit flags combined with bitwise OR to build a permission set.**

```java
public class Demo {
    static final int READ = 1;    // 001
    static final int WRITE = 2;   // 010
    static final int EXECUTE = 4; // 100

    public static void main(String[] args) {
        int permissions = READ | WRITE;
        System.out.println("Permissions value: " + permissions);
        System.out.println("Has READ: " + ((permissions & READ) != 0));
        System.out.println("Has EXECUTE: " + ((permissions & EXECUTE) != 0));
    }
}
```

Output:
```
Permissions value: 3
Has READ: true
Has EXECUTE: false
```

Explanation: combining distinct power of two flags with `|` sets each corresponding bit independently, producing a single `int` that represents multiple simultaneous flags. Checking for a specific flag's presence later uses `&` against that same flag constant, and testing whether the result is non zero determines whether that particular bit was set.

### 9.2 Quiz: Lend A Hand on Logical and Bitwise Operators

1. Given `READ = 1`, `WRITE = 2`, `EXECUTE = 4`, what integer value represents having all three permissions combined?
   A. `3`  B. `6`  C. `7`  D. `12`

2. Using the permission constants above, what does `(7 & WRITE) != 0` evaluate to?
   A. `true`  B. `false`  C. Compile error  D. `2`

3. What is the output of the following?
   ```java
   int x = 5;
   boolean result = (x > 0) | (10 / (x - 5) > 1);
   ```
   A. `true`  B. `false`  C. Throws `ArithmeticException`  D. Compile error

4. What is the output of the same code as question 3 but with `|` replaced by `||`?
   A. `true`  B. `false`  C. Throws `ArithmeticException`  D. Compile error

5. To remove the `WRITE` flag from a permission value `p` that may or may not have it set, which expression is correct?
   A. `p = p | WRITE;`  B. `p = p & ~WRITE;`  C. `p = p ^ WRITE;` only  D. `p = p & WRITE;`

### 9.3 Quiz Answers and Reasoning

1. **Answer: C, `7`.** `READ | WRITE | EXECUTE` is `001 | 010 | 100`, which sets all three bits, giving `111`, which is `7` in decimal.

2. **Answer: A, `true`.** `7` in binary is `111`. `WRITE` is `010`. `7 & WRITE` is `010`, which is `2`, a non zero value, so the comparison `!= 0` evaluates to `true`, correctly detecting that the `WRITE` flag is present within the combined value `7`.

3. **Answer: C, throws `ArithmeticException`.** `x - 5` is `0`, so `10 / (x - 5)` is integer division by zero. Since `|` does not short circuit, this right operand is always evaluated regardless of the left operand `(x > 0)` already being `true`, so the exception is thrown at runtime.

4. **Answer: A, `true`.** With `||`, since the left operand `(x > 0)` is already `true`, the right operand is never evaluated at all, so no division by zero ever occurs, and the overall result is simply `true`.

5. **Answer: B, `p = p & ~WRITE;`.** `~WRITE` flips every bit of `WRITE`, producing a mask with a `0` specifically in the `WRITE` bit position and `1` everywhere else. ANDing `p` with this mask preserves every other bit of `p` unchanged while forcibly clearing the `WRITE` bit specifically, correctly removing that flag regardless of whether it was previously set.

### 9.4 Programming Practice: Lend A Hand on Logical and Bitwise Operators

1. Write a program modeling four independent boolean settings for a device, such as `wifiEnabled`, `bluetoothEnabled`, `gpsEnabled`, and `airplaneMode`, and print a single summary boolean indicating whether the device is in a valid state, defined as airplane mode being `false` whenever any of the other three are `true`, using short circuit logical operators appropriately.
2. Write a program using bit flag constants for four different notification types, combine any three of them into one value using `|`, then check and print, for all four individual flags, whether each one is present in the combined value.
3. Write a program that demonstrates the practical risk of using `|` instead of `||` by constructing a scenario, similar to the guarded division example, where using `|` causes a runtime exception that `||` would have safely avoided, printing a clear before and after comparison.

### 9.5 Programming Solutions: Lend A Hand on Logical and Bitwise Operators

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        boolean wifiEnabled = true;
        boolean bluetoothEnabled = true;
        boolean gpsEnabled = false;
        boolean airplaneMode = false;

        boolean validState = !airplaneMode || (!wifiEnabled && !bluetoothEnabled && !gpsEnabled);
        System.out.println("Device is in a valid state: " + validState);
    }
}
```

Why it works: the expression reads as "either airplane mode is off, or, if it is on, none of the other three radios are enabled," directly encoding the stated rule. Short circuiting here means the three `!` checks on the right are only ever evaluated when airplane mode is genuinely on, since once `!airplaneMode` is `true` the whole `||` already resolves.

**Solution 2.**

```java
public class Solution2 {
    static final int SMS = 1;
    static final int EMAIL = 2;
    static final int PUSH = 4;
    static final int CALL = 8;

    public static void main(String[] args) {
        int enabledNotifications = SMS | EMAIL | PUSH;
        System.out.println("SMS enabled: " + ((enabledNotifications & SMS) != 0));
        System.out.println("EMAIL enabled: " + ((enabledNotifications & EMAIL) != 0));
        System.out.println("PUSH enabled: " + ((enabledNotifications & PUSH) != 0));
        System.out.println("CALL enabled: " + ((enabledNotifications & CALL) != 0));
    }
}
```

Why it works: each notification type occupies its own distinct bit position, since each constant is a distinct power of two, so combining any subset with `|` never causes two flags to interfere with each other, and checking each flag individually with `&` correctly and independently reports its presence or absence.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int stock = 0;
        boolean hasStock = stock > 0;

        System.out.println("Using || (safe):");
        boolean safeResult = hasStock || (100 / stock > 1);
        System.out.println("Result: " + safeResult);

        System.out.println("Using | (unsafe):");
        boolean unsafeResult = hasStock | (100 / stock > 1);
        System.out.println("Result: " + unsafeResult);
    }
}
```

Expected behavior: the `||` line prints its result normally since it short circuits and never divides by `stock`, which is `0`, while the `|` line throws an `ArithmeticException` at runtime because it always evaluates `100 / stock`, causing the program to terminate abruptly right after printing "Using | (unsafe):".

Why it works: this directly demonstrates, with a concrete runtime crash, the correctness risk of choosing the non short circuiting bitwise operator over the logical operator whenever the right operand can fail under certain left operand conditions.

---

## 10. Shift and Assignment Operators

### 10.1 Concept Explanation

Shift operators move the bits of an integer value left or right by a specified number of positions, and assignment operators, beyond the plain `=`, combine an arithmetic or bitwise operation with assignment in a single compact symbol. Both families frequently appear together in exam questions because compound assignment versions of the shift operators exist as well.

**Left shift `<<`.** This shifts every bit of the left operand to the left by the number of positions given by the right operand, filling the newly vacated low order bits with zeros. Shifting left by `n` positions is mathematically equivalent to multiplying by two raised to the power of `n`, for values that do not overflow the type's range, which makes `<<` a fast way to think about doubling repeatedly.

**Signed right shift `>>`.** This shifts every bit of the left operand to the right by the number of positions given, filling the newly vacated high order bits with copies of the original sign bit. This means a positive number stays filled with zeros from the left as it shifts, and a negative number stays filled with ones from the left as it shifts, preserving the sign of the original value throughout the shift. Shifting right by `n` positions is equivalent to integer division by two raised to the power of `n`, though with a subtlety for negative numbers explained below.

**Unsigned right shift `>>>`.** This shifts every bit of the left operand to the right by the number of positions given, but always fills the newly vacated high order bits with zeros, regardless of the original sign of the value. For a positive number, `>>` and `>>>` produce identical results, but for a negative number, they differ significantly, since `>>>` discards the sign entirely and effectively treats the original bit pattern as if it were unsigned.

**Shift amount is taken modulo the operand's bit width.** For `int` operands, which are 32 bits, the actual shift distance used is the right operand value modulo 32. For `long` operands, which are 64 bits, the actual shift distance is the right operand value modulo 64. This means shifting an `int` by `32` is equivalent to shifting by `0`, which is a commonly tested edge case that surprises many candidates who expect shifting by the full bit width to zero out the value entirely.

**Compound assignment operators.** Java provides a compact combined form for most binary operators: `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`, and `>>>=`. Each of these performs the operation and assigns the result back to the left operand in one step, such as `x += 5;` being equivalent in effect to `x = x + 5;`.

**The hidden implicit cast inside compound assignment operators.** This is one of the single most important and most heavily tested subtleties in this entire topic. A compound assignment operator like `+=` implicitly inserts a narrowing cast back to the left operand's original type, even if the plain, uncompressed form of that same operation would not compile without an explicit cast. Concretely, `byte b = 10; b += 5;` compiles and works correctly, silently performing the equivalent of `b = (byte) (b + 5);`, even though writing `b = b + 5;` directly, without the compound operator, would fail to compile, since `b + 5` promotes to `int` and cannot be assigned back to a `byte` without an explicit cast.

### 10.2 Exam Perspective

1. **Direct conceptual questions** ask you to state what each of `<<`, `>>`, and `>>>` does, and how they differ from each other, particularly regarding negative numbers.
2. **Output based questions** trace a specific shift operation on a small integer, requiring you to convert to binary, shift, and convert back.
3. **Tricky or misleading questions** apply `>>` and `>>>` to the same negative number and ask you to identify which result differs and why.
4. **Edge case questions** test shifting by an amount equal to or greater than the operand's bit width, such as shifting an `int` by `33`, expecting you to apply the modulo bit width rule rather than assuming the value becomes zero.
5. **Debugging and error identification questions** show a compound assignment applied to a smaller type such as `byte` or `short` and ask whether it compiles, testing whether you know about the hidden implicit narrowing cast.
6. **Questions combining multiple concepts** combine compound assignment's hidden cast with earlier casting rules, asking you to predict the final stored value after several compound operations on a `byte` variable that individually would overflow if done manually without casting.
7. **Scenario based questions** frame left shift as a fast multiplication trick, or right shift as a fast division trick, and ask for the result on values where these mathematical equivalences hold or subtly break down, such as for negative operands under `>>`.

### 10.3 Examples

**Example 1: Left shift as multiplication by a power of two.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = 5;
        System.out.println(x << 1);
        System.out.println(x << 3);
    }
}
```

Output:
```
10
40
```

Explanation: `5 << 1` shifts `101` left by one position to `1010`, which is `10`, equivalent to `5 * 2`. `5 << 3` shifts `101` left by three positions, equivalent to multiplying by `2` three times, which is `5 * 8`, giving `40`.

**Example 2: Signed right shift versus unsigned right shift on a negative number.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = -8;
        System.out.println(x >> 1);
        System.out.println(x >>> 1);
    }
}
```

Output:
```
-4
2147483644
```

Explanation: `-8 >> 1` preserves the sign bit while shifting, so the result remains negative, giving `-4`, which correctly matches `-8` divided by `2`. `-8 >>> 1`, however, fills the vacated high bit with a `0` instead of preserving the sign, which for a negative number produces a very large positive result, since the original sign bit is now treated as an ordinary magnitude bit rather than a sign indicator.

**Example 3: Shift amount taken modulo bit width.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = 1;
        System.out.println(x << 32);
        System.out.println(x << 33);
    }
}
```

Output:
```
1
2
```

Explanation: for `int` operands, the shift distance is taken modulo `32`. `32 % 32` is `0`, so `x << 32` behaves identically to `x << 0`, leaving `x` unchanged at `1`. `33 % 32` is `1`, so `x << 33` behaves identically to `x << 1`, giving `2`.

**Example 4: Hidden implicit cast inside compound assignment.**

```java
public class Demo {
    public static void main(String[] args) {
        byte b = 10;
        b += 300; // compiles due to hidden implicit narrowing cast
        System.out.println(b);
        // byte c = 10;
        // c = c + 300; // this would NOT compile without an explicit cast
    }
}
```

Output: `54`

Explanation: `b += 300` is silently treated as `b = (byte) (b + 300);`. `10 + 300` is `310`, an `int`. Narrowing `310` to `byte` keeps only the lowest 8 bits, which under two's complement signed interpretation gives `54`. Writing the equivalent expanded form without the compound operator would not compile at all, since it lacks the automatic cast that the compound assignment operator silently inserts.

### 10.4 Important Notes

- `<<` shifts left, filling vacated low bits with zero; it is equivalent to multiplying by a power of two.
- `>>` shifts right, filling vacated high bits with copies of the sign bit, preserving the sign of the original value.
- `>>>` shifts right, always filling vacated high bits with zero, discarding the original sign, which produces very different results from `>>` for negative operands.
- Shift distance for `int` operands is taken modulo `32`; for `long` operands, modulo `64`. Shifting by the full bit width or an exact multiple of it leaves the value unchanged.
- Every compound assignment operator implicitly inserts a narrowing cast back to the left operand's declared type, allowing operations that would otherwise require an explicit cast to compile silently when written in compound form.
- There is no `<<<` operator in Java; only `>>>` exists as the unsigned variant, since left shift always fills with zero regardless of sign, making a separate unsigned left shift operator unnecessary.
- `>>` and `>>>` behave identically for non negative operands, since there is no sign bit difference to matter; they only diverge for negative operands.

### 10.5 Scenario Based Understanding

**Scenario A.** A question shows repeated use of `<<` to compute powers of two quickly inside a loop and asks what happens once the shifted value would exceed `Integer.MAX_VALUE`.
- What is happening: continued left shifting eventually overflows the `int` range.
- Which concept is involved: left shift combined with integer overflow, connecting back to the earlier primitive types discussion.
- How to identify it: track how many total left shift positions have accumulated and compare against the 32 bit signed range.
- Correct reasoning: once the shift would push a `1` bit beyond the 31st position, sign bit included, the value wraps around according to ordinary two's complement overflow rules, potentially producing a negative or otherwise unexpected result rather than the mathematically expected larger positive power of two.
- Common mistake: assuming `<<` behaves like unbounded mathematical multiplication with no ceiling, ignoring that it operates within the fixed bit width of the operand's declared type.

**Scenario B.** A question shows `byte total = 0;` accumulated using `total += someArrayValue;` inside a loop over several `byte` array elements whose sum would exceed `127`, and asks whether the code compiles and what the final value is.
- What is happening: the compound assignment operator's hidden implicit cast allows this to compile, but the accumulated `byte` result can overflow.
- Which concept is involved: compound assignment's implicit narrowing cast combined with `byte` overflow.
- How to identify it: notice the accumulator is declared as `byte`, not `int`, and the compound `+=` operator is used rather than a plain `=` with an explicit expanded expression.
- Correct reasoning: the code compiles successfully because of the compound assignment operator's implicit cast, but each addition beyond `127` wraps around according to `byte`'s 8 bit signed range, so the final printed total may be a smaller or even negative number compared to the true mathematical sum.
- Common mistake: assuming this code either fails to compile, by incorrectly applying the plain assignment rule, or assuming it correctly holds any arbitrarily large sum, without accounting for the `byte` type's narrow range and the implicit truncating cast.

### 10.6 Quiz: Shift and Assignment Operators

1. What is `3 << 2`?
   A. `6`  B. `9`  C. `12`  D. `5`

2. What is `-16 >> 2`?
   A. `-4`  B. `4`  C. `-64`  D. `64`

3. What is `-1 >>> 28`?
   A. `-1`  B. `15`  C. `1`  D. `0`

4. What is the shift distance actually applied when computing `int x = 5; x << 34;`?
   A. `34`  B. `2`  C. `0`  D. Compile error

5. Given `short s = 100; s += 50000;`, does this compile, and if so what happens conceptually?
   A. Compile error  B. Compiles due to implicit narrowing cast inside `+=`, may overflow `short` range  C. Compiles and always fits safely  D. Runtime exception

6. Which of these plain, non compound statements would fail to compile if `b` is declared `byte`?
   A. `b += 1;`  B. `b = b + 1;`  C. `b <<= 1;`  D. `b++;`

7. What is `8 >> 1` compared to `8 >>> 1`?
   A. They differ  B. They are identical, since 8 is non negative  C. `>>` throws an exception  D. `>>>` is illegal on positive numbers

8. What is `1 << 0`?
   A. `0`  B. `1`  C. `2`  D. Compile error

9. Which operator has no separate distinct behavior for negative numbers compared to its counterpart, because it always fills vacated bits with zero regardless of sign?
   A. `<<`  B. `>>`  C. `>>>`  D. None of these

10. What does `x >>= 2;` do?
    A. `x = x >> 2;` with an implicit cast back to `x`'s original type if needed  B. `x = 2 >> x;`  C. Compile error  D. `x = x * 2;`

### 10.7 Quiz Answers and Reasoning

1. **Answer: C, `12`.** `3` is `011` in binary. Shifting left by `2` positions gives `1100`, which is `12`, matching `3 * 4`, since shifting left by `n` multiplies by `2^n`.

2. **Answer: A, `-4`.** `-16 >> 2` preserves the sign while shifting right by two positions, which is equivalent to integer division by `4`. `-16 / 4` is `-4`, matching the signed right shift result exactly for this particular value.

3. **Answer: B, `15`.** `-1` in 32 bit binary is all ones, `11111111111111111111111111111111`. `>>>` fills vacated high bits with zero rather than preserving the sign, so shifting right by `28` positions leaves only the lowest `4` bits, all of which were originally `1`, giving `1111`, which is `15`.

4. **Answer: B, `2`.** For `int` operands, the shift distance is taken modulo `32`. `34 % 32` is `2`, so the effective shift is by `2` positions, not `34`.

5. **Answer: B.** The compound `+=` operator silently inserts a narrowing cast back to `short`, so this compiles successfully despite the fact that `s + 50000` as an `int` addition would exceed the `short` range; the resulting stored value wraps around according to `short`'s 16 bit signed range rather than causing a compile error.

6. **Answer: B, `b = b + 1;`.** Plain, uncompressed arithmetic promotes `byte` operands to `int`, and assigning that `int` result directly back to a `byte` variable requires an explicit cast, which is missing here, causing a compile error. Options A, C, and D all use compound or increment operators, which include the implicit narrowing cast automatically, so all three compile successfully.

7. **Answer: B, they are identical.** For non negative operand values, there is no sign bit complication, since the high bits being shifted in would be zero either way under both signed and unsigned right shift rules, so `8 >> 1` and `8 >>> 1` produce the exact same result.

8. **Answer: B, `1`.** Shifting by zero positions leaves the value completely unchanged, so `1 << 0` remains `1`.

9. **Answer: A, `<<`.** Left shift always fills newly vacated low order bits with zero regardless of the operand's sign, which is exactly why Java provides only one left shift operator and no separate "unsigned left shift" variant, unlike right shift, which needs both a signed and unsigned version specifically because sign preservation matters when shifting rightward.

10. **Answer: A.** `x >>= 2;` is the compound form of `x = x >> 2;`, and like all compound assignment operators, it includes an implicit cast back to `x`'s original declared type if the intermediate computation would otherwise promote to a wider type.

### 10.8 Programming Practice: Shift and Assignment Operators

1. **Basic.** Write a program that computes the first eight powers of two, starting from `2^0`, using only the left shift operator inside a loop, printing each value.
2. **Intermediate.** Write a program that takes a hardcoded negative `int` value and prints the results of applying `>>` and `>>>` to it by one, two, and four positions, clearly labeling each result so the divergence between the two operators becomes visible as the shift amount increases.
3. **Intermediate.** Write a program that demonstrates the compound assignment implicit cast by accumulating a running total into a `byte` variable across a loop over an `int` array whose true sum exceeds `127`, printing the final wrapped `byte` value alongside the true unwrapped `int` sum for comparison.
4. **Advanced.** Write a program that extracts and prints each individual byte, from most significant to least significant, out of a given `int` value using only right shift and bitwise AND with `0xFF`, without using any built in conversion utility method.
5. **Edge case based.** Write a program that demonstrates the modulo bit width rule for shift amounts by shifting a fixed `int` value left by `0`, `32`, `64`, and `33` positions, printing all four results and explaining in comments why the first three produce identical output.

### 10.9 Programming Solutions: Shift and Assignment Operators

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        for (int i = 0; i < 8; i++) {
            System.out.println("2^" + i + " = " + (1 << i));
        }
    }
}
```

Why it works: `1 << i` computes two raised to the power of `i` directly through bit shifting, since starting from the single set bit representing `1` and shifting it left by `i` positions is mathematically identical to repeated doubling `i` times.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int x = -64;
        System.out.println("x = " + x);
        System.out.println("x >> 1 = " + (x >> 1) + ", x >>> 1 = " + (x >>> 1));
        System.out.println("x >> 2 = " + (x >> 2) + ", x >>> 2 = " + (x >>> 2));
        System.out.println("x >> 4 = " + (x >> 4) + ", x >>> 4 = " + (x >>> 4));
    }
}
```

Why it works: `>>` consistently preserves the negative sign across every shift amount, staying mathematically consistent with division by increasing powers of two, while `>>>` produces increasingly large positive numbers as more high order zero bits are introduced from the left, directly and visibly illustrating the divergence described in the concept explanation.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int[] values = {50, 60, 40, 30};
        byte byteTotal = 0;
        int trueTotal = 0;

        for (int v : values) {
            byteTotal += v;
            trueTotal += v;
        }

        System.out.println("True int sum: " + trueTotal);
        System.out.println("Wrapped byte sum: " + byteTotal);
    }
}
```

Thought process: the four values sum to `180` mathematically, which exceeds `byte`'s maximum of `127`, so the `byte` accumulator is expected to wrap around at least once during the accumulation, while the parallel `int` accumulator correctly tracks the true, unwrapped total for direct comparison.

Why it works: `byteTotal += v` compiles despite `v` being `int`, thanks to the compound assignment operator's implicit narrowing cast back to `byte` after each addition, and each individual addition is subject to `byte`'s 8 bit wraparound behavior, so the final stored value may differ substantially from the true mathematical sum shown by `trueTotal`.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        int value = 0x12345678;
        int byte1 = (value >> 24) & 0xFF;
        int byte2 = (value >> 16) & 0xFF;
        int byte3 = (value >> 8) & 0xFF;
        int byte4 = value & 0xFF;

        System.out.println("Most significant byte: " + Integer.toHexString(byte1));
        System.out.println("Next byte: " + Integer.toHexString(byte2));
        System.out.println("Next byte: " + Integer.toHexString(byte3));
        System.out.println("Least significant byte: " + Integer.toHexString(byte4));
    }
}
```

Expected output:
```
Most significant byte: 12
Next byte: 34
Next byte: 56
Least significant byte: 78
```

Thought process: an `int` is 32 bits, made up of four 8 bit bytes. Shifting right by `24`, `16`, and `8` positions respectively brings each successive byte down into the lowest 8 bit position, and ANDing with `0xFF`, which is `11111111` in binary, masks off everything except that lowest byte, isolating exactly the byte of interest each time.

Why it works: right shifting moves the desired byte into position, and the `& 0xFF` mask discards any remaining higher bits that the shift did not fully remove, particularly relevant for the `>>` sign extension behavior on negative values, though this specific chosen value is positive and does not trigger that complication.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        int x = 1;
        System.out.println("x << 0 = " + (x << 0));
        System.out.println("x << 32 = " + (x << 32));
        System.out.println("x << 64 = " + (x << 64));
        System.out.println("x << 33 = " + (x << 33));
        // The first three all produce 1, because for an int operand the shift
        // amount is taken modulo 32: 0 % 32 = 0, 32 % 32 = 0, and 64 % 32 = 0,
        // so all three are equivalent to shifting by 0, leaving x unchanged.
        // The last one, 33 % 32 = 1, so it behaves like x << 1, giving 2.
    }
}
```

Expected output:
```
x << 0 = 1
x << 32 = 1
x << 64 = 1
x << 33 = 2
```

Why it works: this is a direct, concrete demonstration of the shift distance modulo bit width rule stated in the concept explanation, confirming that shift amounts equal to exact multiples of the operand's bit width leave the value completely unaffected, which is one of the most commonly missed edge cases on this topic.

---

## 11. Operator Precedence

### 11.1 Concept Explanation

Operator precedence determines the order in which different operators are applied when multiple operators appear together in a single expression without explicit parentheses. Associativity determines the order in which operators of the same precedence level are applied relative to each other, either from left to right or from right to left. Understanding precedence and associativity is essential for correctly predicting the output of any non trivial expression, and it ties together every operator family covered so far into one unified mental model.

**The complete precedence table, from highest to lowest.** Operators higher in this table bind more tightly, meaning they are effectively grouped and evaluated before operators lower in the table, in the absence of explicit parentheses.

| Precedence Level | Operators | Associativity |
|---|---|---|
| 1 (highest) | `[]`, `.`, `()` method call, postfix `++`, postfix `--` | Left to right |
| 2 | Unary `+`, unary `-`, prefix `++`, prefix `--`, `!`, `~`, cast `(type)` | Right to left |
| 3 | `*`, `/`, `%` | Left to right |
| 4 | Binary `+`, binary `-` | Left to right |
| 5 | `<<`, `>>`, `>>>` | Left to right |
| 6 | `<`, `<=`, `>`, `>=`, `instanceof` | Left to right |
| 7 | `==`, `!=` | Left to right |
| 8 | `&` (bitwise or boolean AND) | Left to right |
| 9 | `^` (bitwise or boolean XOR) | Left to right |
| 10 | `|` (bitwise or boolean OR) | Left to right |
| 11 | `&&` | Left to right |
| 12 | `||` | Left to right |
| 13 | `?:` ternary conditional | Right to left |
| 14 (lowest) | `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `^=`, `|=`, `<<=`, `>>=`, `>>>=` | Right to left |

**Reading the table correctly.** A higher position in the table, closer to the top, means that operator is evaluated earlier, before a lower positioned operator, whenever both appear in the same expression without parentheses forcing a different order. For example, since `*` sits above binary `+` in the table, `2 + 3 * 4` evaluates the multiplication first, giving `2 + 12`, which is `14`, not `20`.

**Associativity matters when operators of the same precedence level appear consecutively.** Most operators in Java are left to right associative, meaning that when several operators of equal precedence appear in a row, evaluation groups from the left first. For example, `20 - 5 - 3` groups as `(20 - 5) - 3`, giving `12`, not `20 - (5 - 3)`, which would give `18`. Assignment operators and the ternary operator, however, are right to left associative, which is why a chained assignment like `a = b = c = 5;` correctly assigns `5` to `c` first, then that result to `b`, then that result to `a`, working from the rightmost assignment outward.

**Why explicit parentheses are always the safest choice in real code, even though exams demand you know the table.** While competitive exams specifically test your ability to correctly apply precedence rules from memory without parentheses, professional Java code style strongly favors adding explicit parentheses to clarify intent whenever an expression mixes more than two or three different operator types, since relying purely on memorized precedence invites subtle bugs in real projects. For exam purposes, however, you must be able to evaluate unparenthesized expressions correctly and confidently.

**The ternary conditional operator `?: `.** This is Java's only three operand, or ternary, operator, taking the form `condition ? valueIfTrue : valueIfFalse`. It evaluates the boolean `condition`, and based on that result, evaluates and yields exactly one of the two following expressions, never both, which gives it a short circuit like behavior similar in spirit to `&&` and `||`, in that the expression not chosen is never evaluated at all.

### 11.2 Exam Perspective

1. **Direct conceptual questions** ask you to rank several given operators by precedence, or identify which of two given operators binds tighter.
2. **Output based and code tracing questions** are the dominant format here, presenting a long unparenthesized expression mixing arithmetic, relational, and logical operators, requiring a full careful trace using the precedence table.
3. **Tricky or misleading questions** deliberately mix `&`, `^`, and `|` together, exploiting the fact that many candidates assume these three have equal precedence, when in fact `&` binds tighter than `^`, which binds tighter than `|`.
4. **Questions on associativity** present a chain of the same operator, such as multiple subtractions or multiple assignments, and ask for the grouping order.
5. **Scenario questions combining precedence with short circuiting** test whether inserting parentheses changes both the grouping and, in the case of `&&` versus `&`, the short circuiting behavior itself.
6. **Ternary operator questions** test nested ternary expressions, requiring you to correctly apply right to left associativity to determine which condition governs which branch.
7. **Cast operator precedence questions** test how high the cast operator's precedence sits, particularly in expressions where a cast is immediately followed by an arithmetic operator, connecting back to the earlier Casting Primitives section's example of a cast binding only to its immediate operand.

### 11.3 Examples

**Example 1: Mixing arithmetic and relational operators.**

```java
public class Demo {
    public static void main(String[] args) {
        int result = 3 + 4 * 2 > 10 ? 100 : 200;
        System.out.println(result);
    }
}
```

Output: `100`

Explanation: `*` binds tighter than `+`, so `4 * 2` is computed first, giving `8`. Then `+` gives `3 + 8`, which is `11`. Then `>` gives `11 > 10`, which is `true`. Finally the ternary operator, being lowest in precedence among these, yields the "if true" branch, `100`, since the condition evaluated to `true`; the "if false" branch, `200`, is never evaluated at all.

**Example 2: Bitwise operator precedence ordering among `&`, `^`, and `|`.**

```java
public class Demo {
    public static void main(String[] args) {
        int result = 5 | 3 & 1;
        System.out.println(result);
    }
}
```

Output: `5`

Explanation: `&` has higher precedence than `|`, so `3 & 1` is computed first. `3` is `011` and `1` is `001`, so `3 & 1` gives `001`, which is `1`. Then `5 | 1` gives `101 | 001`, which is `101`, which is `5`.

**Example 3: Chained right to left associative assignment.**

```java
public class Demo {
    public static void main(String[] args) {
        int a, b, c;
        a = b = c = 7;
        System.out.println(a + " " + b + " " + c);
    }
}
```

Output: `7 7 7`

Explanation: assignment is right to left associative, so `c = 7` is evaluated first, yielding `7`, which is then assigned to `b`, yielding `7`, which is then assigned to `a`. All three variables end up holding `7`.

**Example 4: Nested ternary operator with right to left associativity.**

```java
public class Demo {
    public static void main(String[] args) {
        int score = 75;
        String grade = score >= 90 ? "A" : score >= 80 ? "B" : score >= 70 ? "C" : "F";
        System.out.println(grade);
    }
}
```

Output: `C`

Explanation: nested ternary expressions associate right to left, which effectively means each `condition ? value :` is evaluated in sequence, falling through to the next nested ternary only when its own condition is `false`. `score >= 90` is `false`, so evaluation moves to the next nested ternary. `score >= 80` is `false`, so evaluation moves further. `score >= 70` is `true`, so the result is `"C"`, and the final `"F"` fallback is never reached.

### 11.4 Important Notes

- The precedence order, from highest to lowest among commonly tested operators, is roughly: postfix increment and decrement, then unary and cast operators, then `*` `/` `%`, then binary `+` `-`, then shift operators, then relational operators, then `==` `!=`, then `&`, then `^`, then `|`, then `&&`, then `||`, then `?:`, then assignment operators.
- Among the three bitwise or boolean operators `&`, `^`, and `|`, precedence descends in exactly that order: `&` binds tightest, then `^`, then `|` binds loosest, even though they may look like they should be equal priority.
- Most operators are left to right associative; assignment operators and the ternary conditional operator are right to left associative.
- The ternary operator only evaluates the single branch selected by its condition; the other branch is never evaluated, similar in spirit to short circuit logical operators.
- The cast operator has very high precedence, second only to the postfix group, and binds only to the single expression immediately following it, not to an entire longer expression.
- When in doubt while tracing a complex expression, mentally or literally insert parentheses around the highest precedence operations first, then work outward level by level, exactly as the precedence table dictates.
- Explicit parentheses always override the default precedence and associativity rules entirely, and are the single most reliable way to guarantee a specific intended evaluation order both in exam questions with parentheses present and in real production code.

### 11.5 Scenario Based Understanding

**Scenario A.** A question presents `boolean result = a > b & c > d;` where evaluating `c > d` alone, absent the left operand's value, would be perfectly safe with no side effect risk, and asks whether this differs meaningfully from writing `&&` instead.
- What is happening: both `&` and `&&` produce the identical truth table result here, since there is no side effect or error risk hidden in either operand.
- Which concept is involved: precedence is not actually the deciding factor in this particular scenario; short circuiting behavior is, and it happens to not matter here.
- How to identify it: check whether either operand could throw an exception or has an observable side effect; if not, the short circuiting distinction between `&` and `&&` becomes functionally invisible in this specific case.
- Correct reasoning: the two operators would only produce genuinely different program behavior if the right operand had a side effect or a risk of throwing, since here both operands are purely side effect free relational comparisons, `&` and `&&` produce identical results, though `&&` remains the generally preferred idiomatic choice regardless.
- Common mistake: assuming `&` versus `&&` always changes the outcome whenever it appears in a question, rather than specifically checking whether a side effect or error risk exists in the right operand for that particular expression.

**Scenario B.** A question shows an unparenthesized expression combining `+`, `<<`, and `==`, such as `1 + 2 << 3 == 24`, and asks for the boolean result.
- What is happening: three different precedence levels interact in one expression.
- Which concept is involved: relative precedence of arithmetic, shift, and equality operators.
- How to identify it: locate each operator's row in the precedence table: `+` is higher than `<<`, which is higher than `==`.
- Correct reasoning: `1 + 2` is computed first, giving `3`. Then `3 << 3` is computed, giving `24`. Then `24 == 24` is computed, giving `true`.
- Common mistake: assuming `==` or `<<` should be evaluated before the addition simply because of reading order left to right in the source text, rather than consulting the actual precedence hierarchy, which places `+` above both `<<` and `==`.

### 11.6 Quiz: Operator Precedence

1. What is the result of `10 - 2 * 3`?
   A. `24`  B. `4`  C. `8`  D. `-14`

2. What is the result of `20 - 5 - 5`?
   A. `20`  B. `10`  C. `0`  D. `30`

3. What is the result of `12 & 6 | 3`?
   A. `7`  B. `15`  C. `4`  D. `3`

4. Given `int x = 5; boolean result = x++ > 5 || x++ > 5;`, what are the final value of `x` and the result?
   A. `x=6, result=false`  B. `x=7, result=true`  C. `x=6, result=true`  D. `x=7, result=false`

5. What is the output of `System.out.println(2 + 3 == 5);`?
   A. `true`  B. `false`  C. Compile error  D. `5`

6. What is the result of `int y = 4 > 2 ? 10 : 20 + 5;`?
   A. `10`  B. `25`  C. `15`  D. `30`

7. Which operator has the lowest precedence among all operators in Java?
   A. `?:`  B. `=`  C. `||`  D. `&&`

8. What is the result of `5 + 3 << 1`?
   A. `11`  B. `16`  C. `14`  D. `8`

9. What is the result of `(byte) 10 + 5`?
   A. `15` as a `byte`  B. `15` as an `int`  C. Compile error  D. `10` as a `byte`

10. What does `a = b += 2;` do, given `a` and `b` are both declared `int` and `b` starts at `3`?
    A. Compile error  B. `b` becomes `5`, `a` becomes `5`  C. `b` becomes `5`, `a` remains unchanged  D. `a` becomes `2`, `b` remains `3`

### 11.7 Quiz Answers and Reasoning

1. **Answer: B, `4`.** `*` has higher precedence than binary `-`, so `2 * 3` is computed first, giving `6`. Then `10 - 6` gives `4`.

2. **Answer: B, `10`.** Binary `-` is left to right associative, so this groups as `(20 - 5) - 5`, which is `15 - 5`, giving `10`, not `20 - (5 - 5)`, which would give `20`.

3. **Answer: A, `7`.** `&` has higher precedence than `|`, so `12 & 6` is computed first. `12` is `1100` and `6` is `0110`, so `12 & 6` is `0100`, which is `4`. Then `4 | 3` is `0100 | 0011`, which is `0111`, which is `7`.

4. **Answer: B, `x=7, result=true`.** `x++ > 5` uses the original value of `x`, `5`, in the comparison, which is `5 > 5`, `false`, and then increments `x` to `6` as a side effect. Since the left operand of `||` is `false`, evaluation continues to the right operand. The right `x++ > 5` now uses `x`'s current value, `6`, giving `6 > 5`, which is `true`, and increments `x` to `7` afterward. The overall `||` result is `false || true`, which is `true`, and `x` ends at `7`, since both postfix increments took effect over the course of evaluating the full expression.

5. **Answer: A, `true`.** `+` has higher precedence than `==`, so `2 + 3` is computed first, giving `5`. Then `5 == 5` gives `true`.

6. **Answer: A, `10`.** The ternary operator's condition `4 > 2` is `true`, so the result is the "if true" branch, `10`, and the "if false" branch, `20 + 5`, is never evaluated at all, since only one branch of a ternary expression is ever evaluated regardless of what that branch would have computed to.

7. **Answer: B, `=`.** Plain assignment, along with every compound assignment operator, sits at the very bottom of the precedence table, meaning it is evaluated last relative to every other operator type, which is precisely why an expression like `x = 2 + 3` correctly computes the addition first and assigns the finished result afterward.

8. **Answer: B, `16`.** Binary `+` has higher precedence than `<<`, so `5 + 3` is computed first, giving `8`. Then `8 << 1` gives `16`.

9. **Answer: B, `15` as an `int`.** The cast `(byte) 10` binds only to the `10`, producing a `byte` valued `10`. Adding `5` to it promotes both operands to `int` under standard arithmetic promotion rules, since arithmetic on `byte` operands always promotes to at least `int`, so the final expression result is the `int` value `15`, not a `byte`.

10. **Answer: B, `b` becomes `5`, `a` becomes `5`.** Assignment operators are right to left associative and each yields the assigned value as the result of the expression itself. `b += 2` computes `b = b + 2`, making `b` become `5`, and this compound assignment expression itself yields `5` as its result, which is then assigned to `a`, making `a` also become `5`.

**A note on question 4 above:** this question is a good reminder of how easy it is to stop tracking a variable's side effects partway through a short circuit expression once the left operand alone does not immediately terminate evaluation. Whenever `||` or `&&` continues on to evaluate its right operand, make sure you continue applying every increment or decrement side effect on that right side too, all the way through, before reporting any variable's final value.

### 11.8 Programming Practice: Operator Precedence

1. **Basic.** Write a program that computes and prints the result of `2 + 3 * 4 - 5 / 5` in a single statement, and separately print each sub calculation with explicit parentheses to confirm the unparenthesized result matches the step by step breakdown.
2. **Intermediate.** Write a program using a nested nonested ternary expression to assign a letter grade, `"A"` for 90 and above, `"B"` for 80 to 89, `"C"` for 70 to 79, and `"F"` below 70, to five different hardcoded scores, printing each score alongside its computed grade.
3. **Intermediate.** Write a program that evaluates `12 | 5 & 3 ^ 9` both by letting Java compute it directly and by manually computing and printing each intermediate sub result according to correct precedence, confirming both approaches agree.
4. **Advanced.** Write a program simulating a simple point scoring rule using a single unparenthesized expression combining `&&`, `||`, and relational operators to determine bonus eligibility for several hardcoded player stat combinations, then verify by rewriting the same logic with full explicit parentheses and confirming both versions always agree across every test case.
5. **Edge case based.** Write a program demonstrating chained assignment across three `int` variables starting from different initial values, confirming after the chained assignment statement executes that all three end up holding the same final value, and explain in a comment exactly why right to left associativity makes this work correctly.

### 11.9 Programming Solutions: Operator Precedence

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int direct = 2 + 3 * 4 - 5 / 5;
        int step1 = 3 * 4;
        int step2 = 5 / 5;
        int manual = 2 + step1 - step2;
        System.out.println("Direct result: " + direct);
        System.out.println("Manual step by step result: " + manual);
    }
}
```

Expected output:
```
Direct result: 9
Manual step by step result: 9
```

Why it works: `*` and `/` both outrank `+` and `-`, so they are computed first regardless of their position in the expression, giving `3 * 4 = 12` and `5 / 5 = 1`. The remaining `2 + 12 - 1` then evaluates left to right, giving `9`, matching the manually broken down version exactly.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int[] scores = {95, 84, 71, 60, 89};
        for (int score : scores) {
            String grade = score >= 90 ? "A" : score >= 80 ? "B" : score >= 70 ? "C" : "F";
            System.out.println("Score " + score + " -> Grade " + grade);
        }
    }
}
```

Why it works: the right to left associativity of nested ternary expressions correctly cascades through each threshold check in sequence, only falling through to the next comparison when the current one fails, exactly matching the intended grading logic.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int direct = 12 | 5 & 3 ^ 9;

        int andStep = 5 & 3;      // & has highest precedence among these three
        int xorStep = andStep ^ 9; // ^ next
        int manual = 12 | xorStep; // | last

        System.out.println("Direct: " + direct);
        System.out.println("Manual: " + manual);
    }
}
```

Why it works: `&` binds tighter than `^`, which binds tighter than `|`, so `5 & 3` is computed first, giving `1`. Then `1 ^ 9` is computed, giving `8`. Then `12 | 8` is computed, giving `12`, since `12` already has every bit that `8` has set. Both the direct and manually stepped versions correctly agree.

**Solution 4.**

```java
public class Solution4 {
    static boolean isEligible(int points, int gamesPlayed, boolean isCaptain) {
        return points > 100 && gamesPlayed >= 10 || isCaptain;
    }

    static boolean isEligibleParenthesized(int points, int gamesPlayed, boolean isCaptain) {
        return (points > 100 && gamesPlayed >= 10) || isCaptain;
    }

    public static void main(String[] args) {
        System.out.println(isEligible(150, 12, false) + " " + isEligibleParenthesized(150, 12, false));
        System.out.println(isEligible(50, 12, true) + " " + isEligibleParenthesized(50, 12, true));
        System.out.println(isEligible(150, 5, false) + " " + isEligibleParenthesized(150, 5, false));
        System.out.println(isEligible(50, 5, false) + " " + isEligibleParenthesized(50, 5, false));
    }
}
```

Why it works: since `&&` has higher precedence than `||`, the unparenthesized version already groups exactly the same way as the explicitly parenthesized version, so both methods agree on every test case, directly confirming the precedence table's stated ordering between these two operators without relying on memorization alone.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        int a = 1, b = 2, c = 3;
        a = b = c = 99;
        System.out.println("a=" + a + " b=" + b + " c=" + c);
        // Assignment is right to left associative, so this statement is
        // evaluated as a = (b = (c = 99)). First c is assigned 99, and that
        // assignment expression itself evaluates to 99. That 99 is then
        // assigned to b, which likewise evaluates to 99, and finally that 99
        // is assigned to a, leaving all three variables equal to 99.
    }
}
```

Expected output: `a=99 b=99 c=99`

Why it works: precisely as explained in the inline comment, right to left associativity for the assignment operator is what allows a single chained statement to propagate one value across multiple variables in sequence, working from the rightmost assignment target outward toward the leftmost.

---

## 12. Types of Java Statements

### 12.1 Concept Explanation

A statement is a complete unit of execution in Java, roughly corresponding to one instruction or one meaningful action the program performs. Understanding how Java classifies statements provides the structural map for everything covered in the remainder of this chunk, since the If, Switch, While, Do While, For, and Return topics that follow are each specific kinds of statements fitting into the broader categories introduced here.

**Expression statements.** Certain kinds of expressions can stand alone as a complete statement simply by adding a semicolon. Java restricts which expressions are allowed to be used this way, specifically permitting assignment expressions, any use of `++` or `--`, method call expressions, and object creation expressions using `new`. A bare expression like `5 + 3;` on its own line is not a valid statement in Java, since it computes a value but does nothing with it and belongs to none of the permitted categories, which is a genuinely common early beginner compile error.

**Declaration statements.** These introduce a new variable, such as `int x = 5;`, combining the earlier covered concepts of declaration and optionally initialization into a single statement.

**Control flow statements.** These alter the normal, straight line, top to bottom sequential execution order of a program. This large category itself splits into three meaningful sub families, each of which receives its own dedicated section later in this chunk.

1. **Selection statements**, also called decision making or branching statements, choose between alternative paths of execution based on a condition. Java's selection statements are `if`, `if else`, `if else if`, and `switch`.
2. **Iteration statements**, also called looping statements, repeat a block of code multiple times based on a condition or a fixed count. Java's iteration statements are `while`, `do while`, and `for`, including the enhanced `for each` variant.
3. **Transfer of control statements**, also called jump statements, unconditionally redirect execution away from its normal sequential flow to another point in the program. Java's transfer statements are `break`, `continue`, `return`, and the less commonly used `throw` for exceptions, along with `yield` inside switch expressions.

**Block statements.** A block is any sequence of statements enclosed in a matching pair of curly braces `{ }`, and Java treats an entire block as a single statement from a syntactic standpoint, wherever a single statement is grammatically expected, such as the body of an `if` or a loop. This is precisely why an `if` statement's body can be written either as one bare statement with no braces, or as a multi statement block wrapped in braces, both being equally valid from the compiler's perspective, though block bodies are near universally preferred in professional practice for clarity and to avoid a well known class of bugs.

**Empty statement.** A lone semicolon `;` by itself is a valid, if usually accidental and problematic, statement in Java called the empty statement, which does nothing. This becomes an important, frequently tested trap when a stray semicolon accidentally follows an `if` condition or a loop header, silently turning what was intended as the body into a no operation statement while the code that was meant to be the body executes unconditionally, or exactly once, immediately afterward instead.

### 12.2 Exam Perspective

1. **Direct conceptual questions** ask you to classify a given statement, such as `x++;` or `if (x > 0) { ... }`, into its correct category among expression, declaration, selection, iteration, or transfer of control statements.
2. **Error identification questions** show an invalid bare expression statement, such as `x + 1;` alone with no assignment or side effect, and ask why it fails to compile.
3. **Debugging questions** hide a stray semicolon immediately after an `if` condition or a `for` or `while` loop header, and ask what the program actually does versus what it was clearly intended to do.
4. **Tricky or misleading questions** present a loop or conditional whose body is a single statement without braces, immediately followed by another statement at the same indentation level, testing whether you correctly understand that only the first statement is actually part of the loop or conditional, regardless of how the indentation visually suggests otherwise.
5. **Questions testing understanding of blocks** ask whether a block containing zero statements, an empty pair of braces `{}`, is valid as a loop or conditional body, expecting recognition that it is entirely valid and simply does nothing.

### 12.3 Examples

**Example 1: A stray semicolon silently breaking an if statement.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = 5;
        if (x > 10); {
            System.out.println("This always prints regardless of x");
        }
        System.out.println("Done");
    }
}
```

Output:
```
This always prints regardless of x
Done
```

Explanation: the semicolon immediately after `if (x > 10)` is itself the complete, empty statement body of the `if`. The following block, in braces, is therefore not part of the `if` statement at all; it is simply the next standalone block statement in the program, which executes unconditionally every single time, completely independent of `x`'s value.

**Example 2: A loop body without braces only covering the first statement.**

```java
public class Demo {
    public static void main(String[] args) {
        int count = 0;
        while (count < 3)
            System.out.println("Iteration " + count);
            count++;
        System.out.println("Final count: " + count);
    }
}
```

Explanation: this code actually produces an infinite loop, since only `System.out.println("Iteration " + count);` is considered the body of the `while` loop, due to the absence of braces. The `count++;` statement, despite its indentation visually suggesting it belongs to the loop, is actually a completely separate statement that only executes after the loop finishes, which in this case never happens, since `count` is never incremented inside the actual loop body, so the condition `count < 3` remains `true` forever.

**Example 3: A valid but empty block as a loop body.**

```java
public class Demo {
    public static void main(String[] args) {
        int i = 0;
        while (i < 5) {
        }
    }
}
```

Explanation: an entirely empty block `{}` is syntactically valid as a loop body. This particular example compiles and runs, but produces an infinite loop, since `i` is never modified anywhere inside the empty body, illustrating that syntactic validity and sensible program logic are two entirely separate concerns.

### 12.4 Important Notes

- Only assignment, increment, decrement, method call, and object creation expressions may stand alone as expression statements; a bare arithmetic or relational expression alone is not a valid statement.
- Selection statements are `if`, `if else`, `if else if`, and `switch`; iteration statements are `while`, `do while`, and `for`; transfer of control statements are `break`, `continue`, `return`, `throw`, and `yield`.
- A block, delimited by `{ }`, counts as a single statement wherever the grammar expects one statement, allowing multiple statements to be grouped together as the body of an `if`, loop, or method.
- A single stray semicolon `;` is itself a complete, valid, do nothing statement, and it is one of the most common silent sources of logic bugs when it accidentally appears right after an `if` condition or loop header.
- Omitting braces around a control statement's body restricts that body to exactly the single very next statement only; every subsequent statement, regardless of indentation, is not part of that body.
- An empty block `{}` is syntactically valid and does nothing, distinct from a missing body, which would be a compile error.

### 12.5 Scenario Based Understanding

**Scenario A.** A question shows a `for` loop header immediately followed by a semicolon, then a block, and asks how many times the block executes.
- What is happening: the semicolon is the loop's actual, empty body; the block afterward is unrelated to the loop.
- Which concept is involved: the empty statement trap.
- How to identify it: look immediately after the closing parenthesis of the loop header for a semicolon before any brace appears.
- Correct reasoning: the loop itself runs its full iteration count doing nothing each time, since its body is the empty statement, and the block afterward executes exactly once, after the loop has fully finished, regardless of how many iterations the loop performed.
- Common mistake: assuming the block is the loop's body simply because it appears immediately below the loop header in the source code layout.

**Scenario B.** A question shows an `if` statement without braces whose body is a single `System.out.println` call, followed immediately by an unrelated statement at the same indentation, and later an `else` clause, and asks whether the code compiles.
- What is happening: depending on exactly how many statements sit between the `if` and any `else`, this may or may not compile.
- Which concept is involved: brace-less single statement bodies and the dangling else problem.
- How to identify it: carefully count exactly how many statements appear directly after the `if` condition before either a semicolon terminates the single statement body or an `else` keyword appears.
- Correct reasoning: if two or more statements appear after an `if` condition without braces, only the first one is grammatically part of the `if`, and if an `else` clause is intended to pair with that same `if`, it must immediately follow that single body statement, not follow after a second, unrelated statement, or the compiler will report a syntax error since a lone `else` cannot follow an arbitrary statement that is not part of an `if`.
- Common mistake: assuming indentation alone determines which statements belong to the `if` body, when Java's grammar is governed purely by the presence or absence of braces, completely ignoring whitespace and indentation.

### 12.6 Quiz: Types of Java Statements

1. Which of the following is NOT a valid standalone expression statement in Java?
   A. `x++;`  B. `foo();`  C. `x + 1;`  D. `new Object();`

2. What category does `break;` belong to?
   A. Selection statement  B. Iteration statement  C. Transfer of control statement  D. Declaration statement

3. What does the following print?
   ```java
   int x = 1;
   if (x == 1);
   System.out.println("Yes");
   ```
   A. Nothing  B. `Yes`  C. Compile error  D. `No`

4. Is `{}` alone a syntactically valid loop body?
   A. No, it is a compile error  B. Yes, it is valid and simply does nothing  C. Only for `for` loops  D. Only if it contains a comment

5. Which of these correctly classifies `int x = 10;`?
   A. Expression statement  B. Declaration statement  C. Selection statement  D. Transfer of control statement

6. What is wrong, if anything, with the following code compiling?
   ```java
   for (int i = 0; i < 3; i++);
   System.out.println("done");
   ```
   A. Compile error  B. Compiles, loop body is empty, "done" prints once after the loop finishes  C. Compiles, "done" prints 3 times  D. Infinite loop

7. Which of the following statements is classified as a selection statement?
   A. `while`  B. `switch`  C. `for`  D. `return`

### 12.7 Quiz Answers and Reasoning

1. **Answer: C, `x + 1;`.** Java restricts standalone expression statements to assignments, increment or decrement expressions, method calls, and object creation expressions. A bare arithmetic expression like `x + 1` computes a value but takes no action with it and belongs to none of these permitted categories, making it a compile time error when used alone as a statement.

2. **Answer: C, transfer of control statement.** `break` unconditionally redirects execution, exiting a loop or switch immediately, which places it firmly in the transfer of control category alongside `continue`, `return`, and `throw`.

3. **Answer: B, `Yes`.** The semicolon immediately after `if (x == 1)` is itself the complete, empty body of the `if` statement. `System.out.println("Yes");` is therefore not conditionally guarded at all; it is simply the next statement in the program and executes unconditionally regardless of `x`'s value.

4. **Answer: B.** An empty block containing zero statements is entirely syntactically valid wherever a block is permitted, including as a loop body; it simply performs no action on each iteration, though this often signals an unintentional bug rather than deliberate design.

5. **Answer: B, declaration statement.** This statement introduces a new variable `x` of type `int` and initializes it, which is precisely the definition of a declaration statement, distinct from an expression statement, which requires an already existing variable or entity to act upon.

6. **Answer: B.** The semicolon directly after the `for` loop header is the loop's actual, empty body, so the loop runs its full three iterations doing nothing each time. `System.out.println("done");` is a separate statement that executes exactly once, only after the loop has completely finished all its iterations.

7. **Answer: B, `switch`.** `switch` chooses among several possible branches of execution based on the value of an expression, which is precisely the definition of a selection statement, placing it alongside `if`, `if else`, and `if else if`; `while` and `for` are iteration statements, and `return` is a transfer of control statement.

### 12.8 Programming Practice: Types of Java Statements

1. **Basic.** Write a short program containing at least one clear example each of a declaration statement, an expression statement using a method call, and an expression statement using increment, labeling each with a comment identifying its category.
2. **Intermediate.** Write a program that deliberately includes the stray semicolon bug after an `if` condition, run it mentally first by writing down your predicted output in a comment, then correct the bug and show the corrected version producing the intended, different output.
3. **Intermediate.** Write a program using a `for` loop with an intentionally empty body, using it purely to advance a counter variable up to a target value through its own increment expression alone, with no statements at all inside the braces, printing the final counter value after the loop completes.

### 12.9 Programming Solutions: Types of Java Statements

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int total = 0;            // declaration statement
        total = compute();        // expression statement: assignment
        printMessage();           // expression statement: method call
        total++;                  // expression statement: increment
        System.out.println("Total: " + total);
    }

    static int compute() {
        return 42;
    }

    static void printMessage() {
        System.out.println("Computed a value.");
    }
}
```

Why it works: each labeled line demonstrates exactly one of the requested statement categories in isolation, making the classification unambiguous and easy to verify against the definitions given in the concept explanation.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int x = 5;

        // Buggy version's predicted output: "Triggered" prints unconditionally,
        // because the semicolon right after the if condition is its entire body.
        if (x > 100);
        {
            System.out.println("Triggered");
        }

        System.out.println("--- corrected version below ---");

        // Corrected version: removing the stray semicolon restores the
        // intended conditional behavior, so this only prints when x > 100.
        if (x > 100) {
            System.out.println("Triggered");
        }
        System.out.println("Program finished");
    }
}
```

Expected output:
```
Triggered
--- corrected version below ---
Program finished
```

Why it works: the buggy version's `if` has an empty statement body due to the stray semicolon, so the following block runs unconditionally, printing `"Triggered"` even though `x` is `5`, not greater than `100`. The corrected version properly attaches the block as the `if` body, so with `x` still `5`, the condition is `false` and `"Triggered"` correctly does not print the second time.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int counter;
        for (counter = 0; counter < 10; counter++);
        System.out.println("Final counter value: " + counter);
    }
}
```

Expected output: `Final counter value: 10`

Why it works: the loop's entire body is the empty statement represented by the semicolon directly after the header, so nothing happens inside each iteration except the loop header's own built in increment expression `counter++`, which alone is sufficient to advance `counter` from `0` up to `10`, at which point the condition `counter < 10` becomes `false` and the loop terminates, correctly leaving `counter` at `10` when printed afterward.

---

## 13. Selection Statement: If Statements

### 13.1 Concept Explanation

The `if` statement is Java's most fundamental selection statement, allowing a program to execute a block of code only when a specified `boolean` condition evaluates to `true`. It forms the foundation on which `if else` and `if else if` chains are built.

**Basic `if` syntax.**

```java
if (condition) {
    // executes only if condition is true
}
```

The `condition` must be an expression that evaluates to a `boolean` value; unlike C or C++, Java does not permit an `int` or any other non boolean type to be used directly as a condition, so an expression like `if (x)` where `x` is an `int` fails to compile, whereas `if (x != 0)` is the correct, explicit equivalent.

**`if else` syntax.** Adding an `else` clause provides an alternative block to execute specifically when the condition is `false`.

```java
if (condition) {
    // executes if condition is true
} else {
    // executes if condition is false
}
```

Exactly one of these two blocks executes for any given evaluation; there is no scenario in which both execute, and no scenario in which neither executes, since the condition is always either `true` or `false`.

**`if else if` ladder syntax.** Chaining multiple conditions allows a program to check several mutually exclusive possibilities in sequence.

```java
if (condition1) {
    // block A
} else if (condition2) {
    // block B
} else if (condition3) {
    // block C
} else {
    // block D, the final catch all
}
```

Java evaluates each condition strictly in order, from top to bottom. As soon as one condition evaluates to `true`, its corresponding block executes, and every remaining condition in the ladder, along with every remaining block, is skipped entirely, even if a later condition would also have evaluated to `true`. This "first match wins" behavior, combined with the fact that later conditions are never even evaluated once an earlier one succeeds, is one of the single most important behavioral facts about `if else if` chains, and it directly parallels the short circuiting behavior seen earlier with `&&` and `||`.

**The final `else` is always optional.** An `if else if` ladder can terminate without a final catch all `else` clause, in which case, if none of the conditions evaluate to `true`, no block executes at all, and the program simply continues on to whatever statement follows the entire ladder.

**Nested `if` statements.** An `if`, `else if`, or `else` block can itself contain another complete `if` statement, allowing arbitrarily deep conditional logic. Each nested `if` is entirely independent in terms of its own condition evaluation, though the outer condition must be `true` for a nested `if` inside that outer block's body to even be reached and evaluated at all.

**The dangling else problem.** When nested `if` statements are written without braces, an `else` clause always binds to the nearest preceding unmatched `if`, regardless of indentation. This is a classic, heavily tested source of confusion, since visually indented code can strongly suggest an `else` belongs to an outer `if` when the compiler actually associates it with an inner one, or vice versa.

**Order of conditions matters for efficiency and sometimes correctness.** In an `if else if` ladder testing overlapping or nested numeric ranges, such as grade boundaries, the conditions must generally be ordered from most restrictive to least restrictive, or the logic silently produces wrong results, since an early, broader condition can "capture" cases that were intended for a later, narrower condition further down the ladder that never gets the chance to evaluate.

### 13.2 Exam Perspective

1. **Direct conceptual questions** ask about the required type of an `if` condition, and confirm that Java disallows non boolean conditions unlike C or C++.
2. **Output based questions** trace an `if else if` ladder with multiple conditions, several of which could independently be `true`, testing whether you correctly identify only the first matching block as the one that executes.
3. **Error identification questions** show `if (x = 5)` using a single assignment operator instead of `==`, testing awareness that in Java, unlike C, this is actually a compile time error rather than a silently accepted always true bug, precisely because `x = 5` evaluates to an `int` in Java, not a `boolean`, and `int` cannot serve as a condition.
4. **Debugging questions** hide a dangling else ambiguity in unbraced nested `if` statements and ask which outer or inner `if` a given `else` actually pairs with.
5. **Scenario based questions** present grade boundary or tiered pricing logic with conditions listed in the wrong order, testing whether you can identify the resulting logic bug where a broad early condition unintentionally swallows cases meant for a later, narrower condition.
6. **Tricky or misleading questions** combine `if else if` with side effecting conditions, such as method calls or increment operators inside the conditions themselves, testing whether you correctly recognize that conditions after the first matching one are never evaluated, so their side effects never occur.
7. **Questions combining multiple concepts** combine `if` statements with logical operators, relational operators, and even nested nested `if` structures, requiring the full breadth of everything covered in earlier sections to trace correctly.

### 13.3 Examples

**Example 1: First matching condition wins in an if else if ladder.**

```java
public class Demo {
    public static void main(String[] args) {
        int score = 85;
        if (score >= 60) {
            System.out.println("Pass");
        } else if (score >= 80) {
            System.out.println("Distinction");
        } else {
            System.out.println("Fail");
        }
    }
}
```

Output: `Pass`

Explanation: this is a deliberately illustrative bug of the wrong ordering issue. Even though `score` is `85`, which would also satisfy `score >= 80`, the first condition `score >= 60` is checked first and is already `true`, so `"Pass"` prints and the ladder terminates immediately; the second, more specific condition `score >= 80` is never even reached, illustrating exactly why condition ordering matters critically in these ladders.

**Example 2: Correctly ordered if else if ladder from most to least restrictive.**

```java
public class Demo {
    public static void main(String[] args) {
        int score = 85;
        if (score >= 90) {
            System.out.println("A grade");
        } else if (score >= 80) {
            System.out.println("B grade");
        } else if (score >= 70) {
            System.out.println("C grade");
        } else {
            System.out.println("F grade");
        }
    }
}
```

Output: `B grade`

Explanation: ordering the conditions from the most restrictive, highest threshold, down to the least restrictive correctly ensures each score lands in the appropriate, intended bracket, since a score of `85` fails the first, stricter check for `90` and above, then correctly matches the next check for `80` and above.

**Example 3: Non boolean condition fails to compile.**

```java
public class Demo {
    public static void main(String[] args) {
        int flag = 1;
        // if (flag) { // would not compile: int cannot be used as a boolean condition
        if (flag != 0) {
            System.out.println("Flag is set");
        }
    }
}
```

Explanation: unlike C, where any non zero integer is treated as truthy, Java strictly requires an actual `boolean` valued expression as an `if` condition, so `flag` alone, being an `int`, cannot be used directly, and must instead be compared explicitly against `0` or another value to produce a genuine `boolean`.

**Example 4: The dangling else binding to the nearest unmatched if.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = 5;
        int y = 20;
        if (x > 0)
            if (y > 10)
                System.out.println("Both true");
            else
                System.out.println("x > 0 but y <= 10");
        System.out.println("Done");
    }
}
```

Output:
```
Both true
Done
```

Explanation: despite the indentation visually suggesting the `else` might belong to the outer `if (x > 0)`, Java's grammar always binds an `else` to the nearest preceding unmatched `if`, which here is `if (y > 10)`. Since both `x > 0` and `y > 10` are `true`, `"Both true"` prints, and the `else` clause, which is paired with the inner `if`, never executes.

### 13.4 Important Notes

- An `if` condition in Java must be a genuine `boolean` expression; no other type, including `int`, can be used directly as a condition, unlike C or C++.
- In an `if else if` ladder, the first condition that evaluates to `true` determines which single block executes, and every condition after it is never evaluated at all.
- Order conditions in an `if else if` ladder from most restrictive to least restrictive when checking overlapping numeric ranges, or the logic will silently produce incorrect results.
- The final `else` in a ladder is always optional; without it, if no condition matches, no block executes.
- An `else` clause always binds to the nearest preceding unmatched `if` when braces are omitted, regardless of how the code is indented; this is the dangling else rule.
- Always use braces around `if`, `else if`, and `else` bodies in real code, even for single statement bodies, to eliminate any ambiguity and prevent a whole class of maintenance bugs.
- `if (x = 5)` is a compile time error in Java, because `x = 5` is an assignment expression evaluating to `x`'s new `int` value, not a `boolean`, protecting Java programmers from a classic and notorious C bug where `=` is mistakenly typed instead of `==`.

### 13.5 Scenario Based Understanding

**Scenario A.** A question presents a tiered shipping cost calculator with conditions checking `weight > 0`, then `weight > 10`, then `weight > 50`, in that specific order, and asks what shipping tier a package weighing `60` receives.
- What is happening: the conditions are ordered from least restrictive to most restrictive, the opposite of the recommended pattern.
- Which concept is involved: if else if ladder ordering and first match semantics.
- How to identify it: check whether each successive condition is strictly narrower or strictly broader than the one before it.
- Correct reasoning: since `weight > 0` is checked first and a weight of `60` easily satisfies it, that first, overly broad branch executes and every subsequent, more specific branch is skipped, meaning the package incorrectly receives the lowest tier's treatment rather than the tier actually intended for heavy packages.
- Common mistake: assuming the ladder will naturally find the most specific matching condition regardless of the order they are written in, when in fact only literal top to bottom order and first match determine behavior.

**Scenario B.** A question presents two nested `if` statements without braces, with a single trailing `else`, and gives two different candidate interpretations, asking which one Java actually applies.
- What is happening: the dangling else ambiguity.
- Which concept is involved: else binding to the nearest unmatched if.
- How to identify it: count how many `if` statements precede the `else` that do not yet have their own paired `else`.
- Correct reasoning: the `else` always pairs with the innermost, most recently opened `if` that has not already been given its own `else`, regardless of any visual indentation suggesting otherwise.
- Common mistake: relying on indentation to determine the pairing, which is purely a formatting convention with no effect whatsoever on the compiler's actual parsing behavior.

### 13.6 Quiz: Selection Statement If Statements

1. What is required for a valid `if` condition in Java?
   A. Any numeric expression  B. A genuine `boolean` expression  C. Any non-null expression  D. A `String` expression

2. Does `if (x = 5)` compile in Java, given `x` is declared `int`?
   A. Yes, and it always executes the body  B. No, compile time error  C. Yes, but never executes the body  D. Only inside a loop

3. In an `if else if` ladder, what happens once a condition evaluates to `true`?
   A. All remaining conditions are still evaluated but their blocks are skipped  B. All remaining conditions and their blocks are skipped entirely  C. The ladder restarts from the top  D. Compile error if more conditions follow

4. Given `int score = 72;` and the ladder `if (score >= 60) A else if (score >= 70) B else C`, which block executes?
   A. A  B. B  C. C  D. None

5. In `if (x > 0) if (y > 0) System.out.println("Pos"); else System.out.println("Neg");`, which `if` does the `else` pair with?
   A. The outer `if (x > 0)`  B. The inner `if (y > 0)`  C. Neither, compile error  D. Both

6. Is the final `else` clause in an `if else if` ladder mandatory?
   A. Yes, always required  B. No, it is optional  C. Only if there are more than two conditions  D. Only for `boolean` conditions

7. What happens if no condition in an `if else if` ladder without a final `else` evaluates to `true`?
   A. Compile error  B. The last block executes by default  C. No block executes  D. Runtime exception

8. Given `boolean flag = false; if (flag) { System.out.println("A"); } System.out.println("B");`, what is printed?
   A. `A` then `B`  B. Just `A`  C. Just `B`  D. Nothing

### 13.7 Quiz Answers and Reasoning

1. **Answer: B.** Java strictly requires a genuine `boolean` valued expression for any `if` condition; no implicit conversion from `int`, `String`, or any other type to `boolean` exists in the language, which is a deliberate design choice distinguishing Java from C and C++.

2. **Answer: B, compile time error.** `x = 5` is an assignment expression that evaluates to the `int` value `5`, not a `boolean`. Since `if` strictly requires a `boolean` condition, this fails to compile, which is precisely the safety net Java provides against the classic accidental single equals sign bug.

3. **Answer: B.** Once any condition in the ladder evaluates to `true`, its corresponding block executes and the entire rest of the ladder, meaning every remaining condition and every remaining block, is skipped entirely without being evaluated at all.

4. **Answer: A, block A.** `score >= 60` is checked first and `72` satisfies it, being greater than or equal to `60`, so block A executes immediately, and the ladder terminates there; `score >= 70`, block B's condition, is never even evaluated, despite also being `true` for this particular score.

5. **Answer: B, the inner `if (y > 0)`.** Without braces, an `else` always binds to the nearest preceding unmatched `if`, which here is the inner `if (y > 0)`, regardless of how the surrounding code might be indented to visually suggest otherwise.

6. **Answer: B.** The final `else` in any `if else if` ladder is always optional; a ladder can consist purely of `if` and `else if` clauses with no trailing catch all at all.

7. **Answer: C, no block executes.** If none of the conditions in a ladder without a final `else` evaluate to `true`, execution simply falls through the entire ladder, executing no block whatsoever, and continues on to whatever statement follows the ladder.

8. **Answer: C, just `B`.** `flag` is `false`, so the `if` block, which would print `"A"`, never executes. `System.out.println("B");` sits outside and after the entire `if` statement, so it executes unconditionally regardless of `flag`'s value, correctly printing only `B`.

### 13.8 Programming Practice: Selection Statement If Statements

1. **Basic.** Write a program that takes a hardcoded `int` representing a temperature and prints `"Hot"` if above 35, `"Cold"` if below 15, and `"Moderate"` otherwise, using a correctly ordered `if else if` ladder.
2. **Intermediate.** Write a program that classifies a hardcoded `int` age into `"Child"`, `"Teenager"`, `"Adult"`, or `"Senior"` using boundaries `0` to `12`, `13` to `19`, `20` to `59`, and `60` and above respectively, and deliberately write it once with the boundaries checked in the wrong, ascending order first to observe the resulting bug, printing a comment explaining the bug, and then correct it.
3. **Intermediate.** Write a program using nested `if` statements, fully braced to avoid any dangling else ambiguity, that determines ticket pricing based on both a person's age and whether they are a student, with distinct prices for a child, a student, a senior, and a regular adult.
4. **Advanced.** Write a program that evaluates a hardcoded set of three numeric inputs representing a triangle's three side lengths and determines, using nested and chained `if` statements, whether the three lengths can form a valid triangle at all, meaning the sum of any two sides must exceed the third, and if valid, whether the triangle is equilateral, isosceles, or scalene.
5. **Edge case based.** Write a program demonstrating the exact dangling else scenario from Example 4 above with your own chosen variable names and values, first without braces to show Java's default binding behavior, and then with explicit braces added to force the opposite, outer binding, printing both versions' output so the difference becomes directly visible.

### 13.9 Programming Solutions: Selection Statement If Statements

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int temperature = 40;
        if (temperature > 35) {
            System.out.println("Hot");
        } else if (temperature < 15) {
            System.out.println("Cold");
        } else {
            System.out.println("Moderate");
        }
    }
}
```

Why it works: since the two extreme conditions are checked explicitly and are mutually exclusive by their very nature, no specific ordering concern applies here the way it would for overlapping numeric ranges, and any temperature failing both extreme checks correctly falls through to the `"Moderate"` catch all.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int age = 45;

        // Buggy version: checking from the lowest boundary upward means every
        // adult and senior incorrectly gets caught by the very first check,
        // since "age >= 0" is true for essentially everyone.
        System.out.println("Buggy ordering result:");
        if (age >= 0) {
            System.out.println("Child");
        } else if (age >= 13) {
            System.out.println("Teenager");
        } else if (age >= 20) {
            System.out.println("Adult");
        } else {
            System.out.println("Senior");
        }

        System.out.println("Corrected ordering result:");
        if (age >= 60) {
            System.out.println("Senior");
        } else if (age >= 20) {
            System.out.println("Adult");
        } else if (age >= 13) {
            System.out.println("Teenager");
        } else {
            System.out.println("Child");
        }
    }
}
```

Expected output:
```
Buggy ordering result:
Child
Corrected ordering result:
Adult
```

Why it works: the buggy version checks the least restrictive condition, `age >= 0`, first, so it incorrectly captures every single age, always printing `"Child"` regardless of the true age. The corrected version checks from the most restrictive, highest threshold downward, correctly routing `45` into the `"Adult"` bracket.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int age = 25;
        boolean isStudent = true;

        double price;
        if (age < 13) {
            price = 5.00;
            System.out.println("Category: Child, Price: " + price);
        } else {
            if (age >= 60) {
                price = 7.00;
                System.out.println("Category: Senior, Price: " + price);
            } else {
                if (isStudent) {
                    price = 8.00;
                    System.out.println("Category: Student, Price: " + price);
                } else {
                    price = 12.00;
                    System.out.println("Category: Regular Adult, Price: " + price);
                }
            }
        }
    }
}
```

Why it works: every branch is fully braced, entirely removing any dangling else ambiguity, and the nested structure correctly checks each category in a sensible priority order, senior status and student status both being checked only once child status has already been ruled out, matching realistic ticket pricing logic.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        int sideA = 5;
        int sideB = 5;
        int sideC = 8;

        if ((sideA + sideB > sideC) && (sideA + sideC > sideB) && (sideB + sideC > sideA)) {
            System.out.println("Valid triangle");
            if (sideA == sideB && sideB == sideC) {
                System.out.println("Equilateral");
            } else if (sideA == sideB || sideB == sideC || sideA == sideC) {
                System.out.println("Isosceles");
            } else {
                System.out.println("Scalene");
            }
        } else {
            System.out.println("Not a valid triangle");
        }
    }
}
```

Expected output:
```
Valid triangle
Isosceles
```

Why it works: the outer `if` condition directly encodes the triangle inequality rule as a conjunction of three separate checks using `&&`, and only once that outer check passes does the nested `if else if` ladder classify the triangle type, correctly checking for all three sides equal first before falling back to checking any two sides equal, and finally defaulting to scalene if no sides match.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        int m = 3;
        int n = 7;

        System.out.println("Without braces (default binding to inner if):");
        if (m > 0)
            if (n > 10)
                System.out.println("Both conditions true");
            else
                System.out.println("m > 0 but n <= 10");

        System.out.println("With braces (forced binding to outer if):");
        if (m > 0) {
            if (n > 10) {
                System.out.println("Both conditions true");
            }
        } else {
            System.out.println("m <= 0");
        }
    }
}
```

Expected output:
```
Without braces (default binding to inner if):
m > 0 but n <= 10
With braces (forced binding to outer if):
```

Why it works: in the unbraced version, the `else` binds to the inner `if (n > 10)` by default, so with `n` being `7`, which fails that inner check, the `else` branch correctly prints the "m > 0 but n <= 10" message. In the braced version, explicit braces isolate the inner `if` entirely on its own, and the outer `if else` structure now only prints something when the outer condition `m > 0` is `false`; since `m` is `3`, which is greater than `0`, neither the inner block, since `n` fails its own check, nor the outer `else`, since the outer condition is `true`, produces any output at all in this second version, directly demonstrating how identical looking source code layouts produce entirely different actual behavior purely based on brace placement.

---

## 14. Lend A Hand on If Else If

### 14.1 Applied Examples

**Applied Example 1: Side effecting conditions and skipped evaluation in a ladder.**

```java
public class Demo {
    static int checkCount = 0;

    static boolean check(String label, boolean value) {
        checkCount++;
        System.out.println("Checking: " + label);
        return value;
    }

    public static void main(String[] args) {
        if (check("first", true)) {
            System.out.println("Matched first");
        } else if (check("second", true)) {
            System.out.println("Matched second");
        } else if (check("third", true)) {
            System.out.println("Matched third");
        }
        System.out.println("Total checks performed: " + checkCount);
    }
}
```

Output:
```
Checking: first
Matched first
Total checks performed: 1
```

Explanation: since `check("first", true)` returns `true`, the ladder stops immediately after the very first condition, and neither `check("second", ...)` nor `check("third", ...)` is ever called, confirming that `checkCount` only increments once, directly demonstrating that later conditions in a matched ladder are never evaluated at all, not even for their side effects.

**Applied Example 2: A three way overlapping range classification with correct ordering.**

```java
public class Demo {
    public static void main(String[] args) {
        int[] speeds = {45, 65, 90, 120};
        for (int speed : speeds) {
            String category;
            if (speed >= 100) {
                category = "Very Fast";
            } else if (speed >= 70) {
                category = "Fast";
            } else if (speed >= 40) {
                category = "Moderate";
            } else {
                category = "Slow";
            }
            System.out.println(speed + " km/h -> " + category);
        }
    }
}
```

Output:
```
45 km/h -> Moderate
65 km/h -> Moderate
90 km/h -> Fast
120 km/h -> Very Fast
```

Explanation: because the ladder checks from the highest threshold downward, each speed correctly lands in its intended bracket the first time a genuinely applicable condition is reached, with no speed ever incorrectly captured by an overly broad earlier condition.

### 14.2 Quiz: Lend A Hand on If Else If

1. In Applied Example 1, if the first condition had instead been `false`, how many times would `check` be called in total, assuming the second condition is `true`?
   A. 1  B. 2  C. 3  D. 0

2. Given the correctly ordered ladder from Applied Example 2, what would happen if the two middle conditions, for `"Fast"` and `"Moderate"`, were swapped in order, checking `speed >= 40` before `speed >= 70`?
   A. No change in output for any input  B. A speed of `90` would incorrectly be classified as `"Moderate"` instead of `"Fast"`  C. Compile error  D. A speed of `45` would now be `"Fast"`

3. Given `int x = 5;` and the ladder `if (x > 10) A else if (x > 3) B else if (x > 0) C else D`, which block executes?
   A. A  B. B  C. C  D. D

4. In an `if else if` ladder with five conditions, if the third condition is the first one to evaluate `true`, how many total conditions are evaluated?
   A. 5  B. 2  C. 3  D. 4

### 14.3 Quiz Answers and Reasoning

1. **Answer: B, 2.** If the first condition's `check` call returns `false`, the ladder moves on to evaluate the second condition, calling `check` a second time, and since that second call is stated to return `true`, the ladder stops there, meaning `check` was called exactly twice in total: once for the first, failed condition, and once for the second, successful one.

2. **Answer: B.** With the swapped order, `speed >= 40` is now checked before `speed >= 70`. A speed of `90` satisfies `speed >= 40` immediately, so it would incorrectly be classified as `"Moderate"`, and the ladder would never reach the now later positioned `speed >= 70` check that was actually intended to correctly catch it as `"Fast"`.

3. **Answer: B, block B.** `x > 10` is `false` for `x = 5`. `x > 3` is `true` for `x = 5`, so block B executes, and the ladder stops there without evaluating `x > 0` at all, even though it would also have been `true`.

4. **Answer: C, 3.** The ladder evaluates conditions strictly in order until one succeeds. If the third condition is the first to succeed, that means the first and second conditions were both evaluated and found `false`, and the third was then evaluated and found `true`, for a total of exactly three condition evaluations; the fourth and fifth conditions are never reached.

### 14.4 Programming Practice: Lend A Hand on If Else If

1. Write a program with a method that has a visible side effect, such as printing a message, used as the condition in a four branch `if else if` ladder, and demonstrate, by choosing which branch should match, that only the necessary number of condition checks actually occur.
2. Write a program that classifies a hardcoded body mass index value into `"Underweight"`, `"Normal"`, `"Overweight"`, or `"Obese"` using standard threshold boundaries, ensuring the conditions are ordered correctly, and test it against four different hardcoded values, one intended for each category.

### 14.5 Programming Solutions: Lend A Hand on If Else If

**Solution 1.**

```java
public class Solution1 {
    static boolean test(String label, boolean result) {
        System.out.println("Evaluating condition: " + label);
        return result;
    }

    public static void main(String[] args) {
        int value = 3;

        if (test("value == 1", value == 1)) {
            System.out.println("Matched one");
        } else if (test("value == 2", value == 2)) {
            System.out.println("Matched two");
        } else if (test("value == 3", value == 3)) {
            System.out.println("Matched three");
        } else if (test("value == 4", value == 4)) {
            System.out.println("Matched four");
        }
    }
}
```

Expected output:
```
Evaluating condition: value == 1
Evaluating condition: value == 2
Evaluating condition: value == 3
Matched three
```

Why it works: since `value` is `3`, the first two conditions are evaluated and found `false`, the third is evaluated and found `true`, so the ladder stops there, and the fourth condition, `value == 4`, is never evaluated at all, correctly demonstrating that exactly three, not four, condition checks occurred.

**Solution 2.**

```java
public class Solution2 {
    static String classifyBmi(double bmi) {
        String category;
        if (bmi >= 30.0) {
            category = "Obese";
        } else if (bmi >= 25.0) {
            category = "Overweight";
        } else if (bmi >= 18.5) {
            category = "Normal";
        } else {
            category = "Underweight";
        }
        return category;
    }

    public static void main(String[] args) {
        double[] testValues = {16.0, 22.0, 27.0, 33.0};
        for (double bmi : testValues) {
            System.out.println("BMI " + bmi + " -> " + classifyBmi(bmi));
        }
    }
}
```

Expected output:
```
BMI 16.0 -> Underweight
BMI 22.0 -> Normal
BMI 27.0 -> Overweight
BMI 33.0 -> Obese
```

Why it works: checking from the highest threshold, `30.0`, downward ensures each value is correctly routed into its true intended bracket, exactly matching the pattern established throughout this entire section for correctly ordering overlapping numeric range checks in an `if else if` ladder.

---

## 15. Selection Statement: Switch Statement

### 15.1 Concept Explanation

The `switch` statement is Java's other primary selection statement, offering an alternative to long `if else if` ladders specifically when a single expression needs to be compared against several distinct, discrete constant values. There are two forms in modern Java: the traditional switch statement, which has existed since Java's earliest versions, and the newer switch expression, introduced as a preview feature in Java 12 and finalized in Java 14, which can directly produce and yield a value.

**Traditional switch statement syntax.**

```java
switch (expression) {
    case value1:
        // statements
        break;
    case value2:
        // statements
        break;
    default:
        // statements
}
```

The `expression` is evaluated exactly once, and its result is compared against each `case` label in turn. When a match is found, execution jumps directly to that `case` label and continues executing statements from that point onward.

**Permitted types for the switch expression.** The traditional switch supports `byte`, `short`, `char`, `int`, their corresponding wrapper classes `Byte`, `Short`, `Character`, and `Integer`, `String` since Java 7, and `enum` types. Notably, `long`, `float`, `double`, and `boolean` are never permitted as the switch expression's type, which is a frequently tested restriction, since candidates often assume any primitive type would be allowed.

**Case labels must be compile time constants.** Every `case` label's value must be determinable at compile time, meaning it must be a literal, a `final` variable initialized with a constant expression, or an enum constant. A `case` label using a regular, non final variable, or any expression only computable at runtime, fails to compile.

**Fall through behavior.** This is the single most distinctive and most heavily tested characteristic of the traditional switch statement. Once execution jumps to a matching `case` label, it continues executing every subsequent statement in every following `case` block, regardless of whether those following `case` labels also match, until it either encounters a `break` statement, reaches the end of the entire switch block, or encounters a `return` or other transfer of control statement. This is why nearly every `case` block ends with an explicit `break`, to prevent execution from unintentionally continuing, or "falling through," into the next case's code.

**The `default` label.** This optional label's block executes when the switch expression's value does not match any `case` label. `default` can be placed anywhere within the switch block, not necessarily last, though placing it last is by far the most common and readable convention. If `default` is not the final label, fall through rules still apply exactly as they would for any other case, meaning execution can fall through from `default` into a following case if no `break` is present.

**Modern switch expressions with arrow syntax.** Java 14 finalized a new form using `->` instead of `:`, which does not fall through by default, since each arrow labeled branch is implicitly self contained. This new form can also be used as an expression that directly yields a value, assignable to a variable.

```java
int dayNumber = 3;
String dayName = switch (dayNumber) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 3 -> "Wednesday";
    default -> "Unknown";
};
```

When an arrow labeled branch needs multiple statements rather than a single expression, a block in braces is used, and the `yield` keyword explicitly produces the branch's resulting value.

```java
String result = switch (value) {
    case 1 -> "one";
    case 2, 3 -> {
        String temp = "two or three";
        yield temp;
    }
    default -> "other";
};
```

**Multiple case labels sharing one block.** Both the traditional colon form and the modern arrow form allow a single block of code to be associated with several different case values, either by stacking multiple `case` labels consecutively with no code between them in the traditional form, which relies on intentional fall through, or by listing multiple values separated by commas after a single `case` keyword in the modern arrow form.

### 15.2 Exam Perspective

1. **Direct conceptual questions** ask which primitive types are and are not permitted as a switch expression's type, particularly testing whether `boolean`, `long`, `float`, and `double` are correctly recognized as disallowed.
2. **Output based questions** are extremely common around fall through, presenting a switch with a missing `break` somewhere in the middle and asking exactly which case bodies execute as a result.
3. **Error identification questions** show a `case` label using a non final local variable or a runtime computed expression, testing whether you recognize the compile time constant requirement.
4. **Debugging questions** show duplicate `case` labels with the same value, testing whether you know this is a compile time error, since the switch could never determine which one to match.
5. **Tricky or misleading questions** place `default` in the middle of a switch block rather than at the end, and combine this with missing `break` statements, requiring careful tracing of exactly which cases execute given a specific input value.
6. **Scenario based questions** compare the readability and behavior differences between a traditional switch and a modern switch expression solving the same underlying problem, testing whether you understand that the modern form avoids fall through by default.
7. **Questions combining multiple concepts** combine `switch` with `String` case values, testing awareness that string comparison inside a switch is based on `.equals()` semantics rather than reference identity, and also testing null handling, since switching on a `null` `String` throws a `NullPointerException` at runtime in the traditional form.

### 15.3 Examples

**Example 1: Fall through due to a missing break.**

```java
public class Demo {
    public static void main(String[] args) {
        int day = 2;
        switch (day) {
            case 1:
                System.out.println("Monday");
            case 2:
                System.out.println("Tuesday");
            case 3:
                System.out.println("Wednesday");
                break;
            case 4:
                System.out.println("Thursday");
        }
    }
}
```

Output:
```
Tuesday
Wednesday
```

Explanation: execution jumps directly to `case 2`, since `day` is `2`, and prints `"Tuesday"`. With no `break` present at the end of that case block, execution falls through into `case 3`'s block, printing `"Wednesday"` as well, and only stops there because `case 3` does include a `break`, preventing further fall through into `case 4`.

**Example 2: Multiple case labels sharing one block, traditional form.**

```java
public class Demo {
    public static void main(String[] args) {
        char grade = 'B';
        switch (grade) {
            case 'A':
            case 'B':
                System.out.println("Good performance");
                break;
            case 'C':
            case 'D':
                System.out.println("Needs improvement");
                break;
            default:
                System.out.println("Invalid grade");
        }
    }
}
```

Output: `Good performance`

Explanation: stacking `case 'A':` directly above `case 'B':` with no statements or break between them means both values share the exact same following block through intentional fall through, so either `'A'` or `'B'` correctly produces the same `"Good performance"` message.

**Example 3: Modern switch expression, no fall through by default.**

```java
public class Demo {
    public static void main(String[] args) {
        int day = 6;
        String type = switch (day) {
            case 1, 2, 3, 4, 5 -> "Weekday";
            case 6, 7 -> "Weekend";
            default -> "Invalid";
        };
        System.out.println(type);
    }
}
```

Output: `Weekend`

Explanation: the modern arrow form groups multiple values under one branch using a comma separated list, and directly yields a value assignable to `type`, entirely without any risk of fall through, since each arrow branch is self contained by design.

**Example 4: Compile time constant requirement for case labels.**

```java
public class Demo {
    public static void main(String[] args) {
        final int LIMIT = 10; // final, compile time constant, valid as a case label
        int dynamicValue = computeSomething(); // NOT final, would be invalid as a case label

        int x = 10;
        switch (x) {
            case LIMIT:
                System.out.println("Matched the limit");
                break;
            // case dynamicValue: // would not compile: not a compile time constant
            default:
                System.out.println("No match");
        }
    }

    static int computeSomething() {
        return 10;
    }
}
```

Output: `Matched the limit`

Explanation: `LIMIT` is declared `final` and initialized with a literal, making it a genuine compile time constant, which is valid to use as a `case` label. `dynamicValue`, even though it happens to hold the same numeric value at runtime, is not `final` and is initialized by a method call rather than a constant expression, so it could never be used as a `case` label even if the commented line were uncommented.

### 15.4 Important Notes

- Valid switch expression types are `byte`, `short`, `char`, `int`, their wrapper classes, `String`, and `enum` types; `long`, `float`, `double`, and `boolean` are never permitted.
- Every `case` label value must be a compile time constant: a literal, a `final` variable initialized with a constant expression, or an enum constant; a plain variable or a runtime computed expression is not allowed.
- Traditional switch falls through from a matching case into every subsequent case's code until a `break`, `return`, `throw`, `continue`, or the end of the switch block is reached.
- The modern arrow form `->`, finalized in Java 14, does not fall through by default, and can be used as a switch expression that directly yields a value using implicit single expression branches or an explicit `yield` inside a block branch.
- Duplicate `case` labels with the same value are a compile time error.
- `default` is optional and can technically appear anywhere in the switch block, though placing it last is the near universal convention; fall through rules still apply around it in the traditional form regardless of its position.
- Switching on a `String` compares using value equality, equivalent to `.equals()`, not reference identity; switching on a `null` `String` reference throws a `NullPointerException` at runtime in the traditional switch statement.
- A switch statement with no matching case and no `default` simply does nothing and execution continues after the switch block, exactly analogous to an `if else if` ladder with no matching condition and no final `else`.

### 15.5 Scenario Based Understanding

**Scenario A.** A question presents a switch statement handling several related cases meant to share identical behavior, such as classifying several different characters all as vowels, and asks how to avoid duplicating the same block of code five separate times.
- What is happening: several distinct case values should trigger identical behavior.
- Which concept is involved: stacked case labels sharing one fall through block in the traditional form, or comma separated case values in the modern form.
- How to identify it: notice that the intended behavior for several specific input values is exactly the same.
- Correct reasoning: stacking the case labels consecutively with no intervening code, relying on intentional fall through, in the traditional form, or listing all the values together after one `case` keyword with commas in the modern arrow form, both correctly avoid code duplication while cleanly expressing that these values share identical handling.
- Common mistake: writing out the identical block separately under every single case label, leading to unnecessary duplication and a greater risk of one copy being updated while another is accidentally missed during future maintenance.

**Scenario B.** A question shows a traditional switch statement where a `break` was accidentally omitted from the case that matches the given input, and asks what the full printed output will be, given that every case simply prints its own label text.
- What is happening: the matching case, and every case after it, executes in sequence due to the missing break.
- Which concept is involved: fall through.
- How to identify it: find the matching case first, then read straight down through every subsequent case's code until a `break` is found or the switch block ends.
- Correct reasoning: trace execution starting exactly at the matching case label and continue printing each subsequent case's output in order, stopping only once a `break` statement is actually encountered somewhere further down, or the switch block simply ends with no more cases left.
- Common mistake: assuming only the single matching case's own code executes, forgetting that Java's traditional switch does not automatically stop after one matching case the way an `if else if` ladder does.

### 15.6 Quiz: Selection Statement Switch Statement

1. Which of these types is NOT permitted as a switch expression's type?
   A. `int`  B. `char`  C. `boolean`  D. `String`

2. What happens if a `case` block has no `break` and is not the last case?
   A. Compile error  B. Execution falls through into the next case's code  C. The switch simply ends  D. Runtime exception

3. Can two different `case` labels have the exact same constant value?
   A. Yes, the first one always wins  B. Yes, the last one always wins  C. No, compile time error  D. Yes, but only with `default` present

4. What is required of a variable used as a `case` label's value?
   A. It must be `static`  B. It must be a compile time constant  C. It must be `public`  D. It must be initialized inside the switch itself

5. What happens when switching on a `null` `String` reference using a traditional switch statement?
   A. Matches `default` automatically  B. Throws `NullPointerException` at runtime  C. Compile error  D. Matches no case and continues silently

6. In the modern arrow based switch expression form, does execution fall through from one branch into the next by default?
   A. Yes, always  B. No, each branch is self contained by default  C. Only with `String` values  D. Only without `default`

7. Given `switch(x) { case 1: case 2: System.out.println("A"); break; default: System.out.println("B"); }` with `x = 1`, what is printed?
   A. `A`  B. `B`  C. `A` then `B`  D. Nothing

8. Which of the following can be used inside a switch expression block to explicitly produce the resulting value?
   A. `return`  B. `yield`  C. `break`  D. `continue`

### 15.7 Quiz Answers and Reasoning

1. **Answer: C, `boolean`.** Java's switch statement never permits `boolean` as the expression type, along with `long`, `float`, and `double`; `int`, `char`, and `String` are all valid switch expression types.

2. **Answer: B.** Without a `break`, execution continues sequentially into the very next case's code once it reaches the end of the current matching case's block, a behavior known as fall through, which continues until a `break` is finally encountered or the switch block ends entirely.

3. **Answer: C, compile time error.** Two case labels sharing the same constant value would create an unresolvable ambiguity for the compiler about which block should be the actual match, so Java rejects this scenario entirely at compile time rather than choosing either the first or the last one.

4. **Answer: B.** Every case label value must be determinable entirely at compile time, meaning it must be a literal, a `final` variable initialized with a constant expression, or an enum constant; ordinary non final variables or runtime computed values are never valid case labels.

5. **Answer: B.** A traditional switch statement internally must compare the switch expression's value against each case label, and attempting this comparison against a `null` reference throws a `NullPointerException` at runtime before any case matching can even occur.

6. **Answer: B.** The modern arrow based switch form, introduced in Java 14, deliberately does not fall through between branches by default; each arrow labeled branch is entirely self contained, unlike the traditional colon based form.

7. **Answer: A, `A`.** `x = 1` matches the stacked `case 1:` label, and with no code between `case 1:` and `case 2:`, execution falls through into `case 2:`'s block, printing `"A"`, and then the `break` immediately following stops execution there, correctly preventing it from also reaching the `default` block.

8. **Answer: B, `yield`.** Inside a block bodied branch of a switch expression, `yield` is the specific keyword used to explicitly produce that branch's resulting value, distinct from `return`, which would exit the entire enclosing method rather than just yielding a value for the switch expression itself.

### 15.8 Programming Practice: Selection Statement Switch Statement

1. **Basic.** Write a program using a traditional switch statement that takes a hardcoded `int` representing a month number and prints the number of days in that month, correctly handling all twelve months with appropriate `break` statements, and treating February as having 28 days for simplicity.
2. **Intermediate.** Write a program using stacked case labels in a traditional switch to classify a hardcoded `char` as a vowel or consonant, correctly handling both uppercase and lowercase letters by stacking the relevant cases together.
3. **Intermediate.** Write a program using the modern switch expression arrow form to convert a hardcoded `int` day number, 1 through 7, into its corresponding day name, assigning the result directly to a `String` variable and printing it.
4. **Advanced.** Write a program that intentionally demonstrates fall through being used constructively, such as a simplified season classifier where December, January, and February should all print "Winter" through deliberate stacking, but write it using the traditional colon based switch to clearly show the stacked case labels in action, covering all twelve months across four seasons.
5. **Edge case based.** Write a program using a switch expression with a block bodied branch that needs more than one statement for a specific case, using `yield` to correctly produce the final value from within that block, contrasted against the other branches which use the simpler single expression arrow form.

### 15.9 Programming Solutions: Selection Statement Switch Statement

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int month = 4;
        int days;
        switch (month) {
            case 1: case 3: case 5: case 7: case 8: case 10: case 12:
                days = 31;
                break;
            case 4: case 6: case 9: case 11:
                days = 30;
                break;
            case 2:
                days = 28;
                break;
            default:
                days = 0;
        }
        System.out.println("Month " + month + " has " + days + " days");
    }
}
```

Why it works: stacking the seven thirty one day months together, and the four thirty day months together, avoids repeating identical logic seven and four times respectively, while `break` statements throughout correctly prevent any unintended fall through between the differently valued groups.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        char letter = 'e';
        switch (letter) {
            case 'a': case 'e': case 'i': case 'o': case 'u':
            case 'A': case 'E': case 'I': case 'O': case 'U':
                System.out.println(letter + " is a vowel");
                break;
            default:
                System.out.println(letter + " is a consonant");
        }
    }
}
```

Why it works: stacking all ten uppercase and lowercase vowel cases together lets a single shared block correctly handle every vowel possibility, and anything not matching one of those ten stacked labels correctly falls through to `default`, being classified as a consonant.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int dayNumber = 5;
        String dayName = switch (dayNumber) {
            case 1 -> "Monday";
            case 2 -> "Tuesday";
            case 3 -> "Wednesday";
            case 4 -> "Thursday";
            case 5 -> "Friday";
            case 6 -> "Saturday";
            case 7 -> "Sunday";
            default -> "Invalid day";
        };
        System.out.println(dayName);
    }
}
```

Why it works: each arrow branch directly yields its corresponding `String` value with no risk of fall through, and the entire switch expression's result is assigned directly to `dayName` in one clean statement, avoiding the need for a separate variable declared before the switch and then reassigned inside each case.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        int month = 1;
        switch (month) {
            case 12: case 1: case 2:
                System.out.println("Winter");
                break;
            case 3: case 4: case 5:
                System.out.println("Spring");
                break;
            case 6: case 7: case 8:
                System.out.println("Summer");
                break;
            case 9: case 10: case 11:
                System.out.println("Autumn");
                break;
            default:
                System.out.println("Invalid month");
        }
    }
}
```

Why it works: each season's three corresponding months are deliberately stacked together, relying on intentional fall through to share one identical print statement per season, correctly handling all twelve months across exactly four seasonal groups.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        int score = 85;
        String feedback = switch (score / 10) {
            case 10, 9 -> "Excellent";
            case 8 -> {
                String message = "Very good, ";
                message += "keep up the strong performance.";
                yield message;
            }
            case 7 -> "Good";
            default -> "Needs improvement";
        };
        System.out.println(feedback);
    }
}
```

Expected output: `Very good, keep up the strong performance.`

Why it works: `score / 10` gives `8` for a score of `85`, matching the `case 8` branch, which uses a full block body since it needs more than a single expression; the block builds up a `String` across two statements and then uses `yield` to explicitly hand that final built up value back as the result of the entire switch expression, while every other branch remains a simpler single expression arrow form.

---

## 16. Iteration Statement: While Statement

### 16.1 Concept Explanation

The `while` statement is Java's simplest iteration, or looping, construct. It repeatedly executes a block of code for as long as a specified `boolean` condition remains `true`, checking that condition before each iteration, including before the very first one.

**Basic syntax.**

```java
while (condition) {
    // loop body, executes repeatedly while condition is true
}
```

**Entry controlled behavior.** The defining characteristic of `while` is that it is entry controlled, meaning the condition is evaluated before the loop body executes for the first time. If the condition is `false` from the very start, the loop body never executes even once. This distinguishes `while` from `do while`, covered in the next section, which is exit controlled and always executes its body at least once regardless of the condition.

**Loop control variables.** Nearly every practical `while` loop depends on one or more variables that change over the course of the loop's execution, eventually causing the condition to become `false` and the loop to terminate. This variable, or set of variables, must typically be initialized before the loop begins and modified somewhere inside the loop body; forgetting to modify it inside the body is one of the single most common sources of accidental infinite loops.

**Infinite loops.** A `while` loop whose condition never becomes `false`, whether because the condition is a literal `true`, or because the loop body never actually modifies whatever variable the condition depends on, runs forever, or until the program is externally terminated, or until a `break` or `return` statement inside the body is reached. Deliberately infinite loops, written as `while (true) { ... }`, are a legitimate and common pattern specifically when combined with an internal `break` condition, since it allows the loop's exit condition to be checked at a more flexible point within the body rather than strictly at the very top.

**`break` inside a while loop.** The `break` statement immediately terminates the nearest enclosing loop entirely, transferring control to the very next statement after the loop, regardless of what the loop's own condition would otherwise evaluate to.

**`continue` inside a while loop.** The `continue` statement skips the remainder of the current iteration's body and jumps directly back to re evaluating the loop's condition, effectively moving on to the next iteration attempt without executing any code that comes after the `continue` within that same iteration.

**Nested while loops.** A `while` loop can contain another complete `while` loop within its body. `break` and `continue` used inside a nested inner loop, without a label, by default only affect that innermost loop, not any outer loop that contains it. Reaching outer loops from inside a nested structure requires labeled `break` or `continue` statements, a more advanced technique that becomes especially relevant once multiple levels of nesting are involved.

### 16.2 Exam Perspective

1. **Direct conceptual questions** ask you to state whether `while` checks its condition before or after the loop body, and to correctly distinguish this from `do while`.
2. **Output based questions** trace a `while` loop's iterations precisely, tracking a control variable's changing value across every pass through the body.
3. **Debugging questions** present a `while` loop missing the statement that would modify its control variable, testing whether you recognize the resulting infinite loop.
4. **Error identification and tricky questions** hide a stray semicolon immediately after the `while` condition, connecting back to the empty statement trap covered in the Types of Java Statements section, producing either an infinite loop or a loop that spins doing nothing depending on the exact condition used.
5. **Scenario based questions** test `break` and `continue` behavior specifically, asking exactly how many total iterations occur and what gets printed given a `continue` or `break` triggered under a specific condition partway through the loop.
6. **Questions combining multiple concepts** combine `while` loops with earlier arithmetic and relational operators, requiring you to correctly trace both the loop's control flow and the arithmetic happening inside it simultaneously.
7. **Edge case questions** test a `while` loop whose condition is `false` from the very first check, confirming the body never executes even a single time.

### 16.3 Examples

**Example 1: A basic while loop with correct termination.**

```java
public class Demo {
    public static void main(String[] args) {
        int count = 0;
        while (count < 5) {
            System.out.println("Count is " + count);
            count++;
        }
        System.out.println("Loop finished, count = " + count);
    }
}
```

Output:
```
Count is 0
Count is 1
Count is 2
Count is 3
Count is 4
Loop finished, count = 5
```

Explanation: the condition `count < 5` is checked before every single iteration, including the first. The loop runs for values `0` through `4`, five total iterations, and stops as soon as `count` becomes `5`, since `5 < 5` is `false`.

**Example 2: A while loop whose condition is false from the start.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = 10;
        while (x < 5) {
            System.out.println("This never prints");
        }
        System.out.println("Skipped entirely, x is still " + x);
    }
}
```

Output: `Skipped entirely, x is still 10`

Explanation: since `while` is entry controlled, the condition `x < 5` is checked before the body ever runs, and since it is already `false` at that very first check, the body never executes even once.

**Example 3: continue skipping the remainder of an iteration.**

```java
public class Demo {
    public static void main(String[] args) {
        int i = 0;
        while (i < 6) {
            i++;
            if (i % 2 == 0) {
                continue;
            }
            System.out.println("Odd number: " + i);
        }
    }
}
```

Output:
```
Odd number: 1
Odd number: 3
Odd number: 5
```

Explanation: `i` is incremented first at the top of each iteration. When `i` is even, `continue` immediately skips the `System.out.println` call for that iteration and jumps straight back to re evaluate the loop's condition, so only odd values of `i` ever reach the print statement.

**Example 4: break exiting a loop early.**

```java
public class Demo {
    public static void main(String[] args) {
        int i = 0;
        while (true) {
            if (i >= 4) {
                break;
            }
            System.out.println("i = " + i);
            i++;
        }
        System.out.println("Loop exited");
    }
}
```

Output:
```
i = 0
i = 1
i = 2
i = 3
Loop exited
```

Explanation: this uses the deliberately infinite `while (true)` pattern, relying entirely on the internal `break` statement to terminate the loop once `i` reaches `4`, which is a common and legitimate style when the natural exit condition is more conveniently checked partway through the loop body rather than strictly at the top.

### 16.4 Important Notes

- `while` is entry controlled: its condition is checked before the loop body runs, including before the very first iteration, so the body may execute zero times if the condition starts out `false`.
- Forgetting to modify the loop's control variable somewhere inside the body is the single most common cause of an unintentional infinite loop.
- `while (true) { ... }` combined with an internal `break` is a legitimate, common pattern for loops whose natural exit condition is more naturally checked partway through the body.
- `break` immediately and fully terminates the nearest enclosing loop; `continue` skips only the remainder of the current iteration and returns to the condition check for the next iteration.
- A stray semicolon immediately after a `while` condition creates an empty statement as the loop's entire body, which, combined with a condition that never changes as a result, frequently causes an infinite loop.
- `break` and `continue` inside a nested inner loop affect only that innermost loop by default, unless a label is explicitly used to target an outer loop.

### 16.5 Scenario Based Understanding

**Scenario A.** A question shows a `while` loop meant to process items from a data structure until it is empty, but the condition mistakenly checks a variable that is never actually updated inside the loop body, and asks what happens when the program runs.
- What is happening: the loop's exit condition depends on a variable with no path to ever change.
- Which concept is involved: infinite loop caused by a missing control variable update.
- How to identify it: trace every statement inside the loop body and confirm whether any of them actually modifies the variable referenced in the condition.
- Correct reasoning: since nothing inside the body ever changes the variable the condition depends on, the condition remains `true` forever, and the loop runs indefinitely, likely requiring external termination of the program.
- Common mistake: assuming the loop will naturally terminate once the underlying data is conceptually exhausted, without verifying that the code actually contains a statement that updates the tracking variable to reflect that exhaustion.

**Scenario B.** A question shows a `while` loop counting down from a positive number to zero using `count--`, but the condition is written as `count != 0` rather than `count > 0`, and `count` starts at a negative number, and asks what happens.
- What is happening: a negative starting value combined with a `!= 0` condition never actually reaches exactly zero through decrementing, since decrementing from a negative number moves further away from zero, not toward it.
- Which concept is involved: infinite loop caused by a logically flawed exit condition rather than a missing update statement.
- How to identify it: check both the direction the control variable moves, via `++` or `--`, and whether the condition can genuinely ever become `false` given that direction and the starting value.
- Correct reasoning: since `count` starts negative and is decremented, it moves further into negative territory, never landing exactly on `0`, so `count != 0` remains `true` forever, causing an infinite loop despite the control variable being actively updated every single iteration.
- Common mistake: assuming that because the control variable is being modified each iteration, the loop is automatically safe from running forever, without separately verifying that the modification actually moves the variable toward satisfying the exit condition.

### 16.6 Quiz: Iteration Statement While Statement

1. When is a `while` loop's condition checked?
   A. After the loop body executes  B. Before the loop body executes, every time, including the first  C. Only once, before the loop starts  D. Only after an exception occurs

2. If a `while` loop's condition is `false` from the very beginning, how many times does the body execute?
   A. Exactly once  B. Zero times  C. Infinitely  D. Compile error

3. What does `break` do inside a `while` loop?
   A. Skips to the next iteration  B. Terminates the nearest enclosing loop entirely  C. Pauses the loop  D. Restarts the loop

4. What does `continue` do inside a `while` loop?
   A. Terminates the loop  B. Skips the rest of the current iteration and re-checks the condition  C. Skips the entire next iteration  D. Restarts the program

5. What is the output of the following?
   ```java
   int i = 0;
   while (i < 3) {
       System.out.println(i);
   }
   ```
   A. `0 1 2`  B. Infinite loop, printing `0` forever  C. Nothing prints  D. Compile error

6. Given `int x = 5; while (x > 0) { x -= 2; } System.out.println(x);`, what is printed?
   A. `0`  B. `1`  C. `-1`  D. `5`

7. Is `while (true) { }` valid, compilable Java code?
   A. No, infinite loops do not compile  B. Yes, it is valid and produces an infinite loop  C. Only inside a method returning `void`  D. Only with a `break` present

8. What happens with a stray semicolon: `while (x < 5); { x++; }`, assuming `x` starts below 5 and nothing else modifies `x`?
   A. Compiles and behaves as expected, incrementing x until 5  B. Infinite loop, since the semicolon is the loop's entire empty body and x is never modified inside it  C. Compile error  D. Runs the block exactly once

### 16.7 Quiz Answers and Reasoning

1. **Answer: B.** `while` is entry controlled, meaning its condition is evaluated before every single iteration of the body, including the very first one, which is precisely what distinguishes it from `do while`.

2. **Answer: B, zero times.** Since the condition is checked before the body ever runs, a condition that is already `false` at that first check means the body is skipped entirely and never executes even a single time.

3. **Answer: B.** `break` immediately exits the nearest enclosing loop in its entirety, transferring control directly to the first statement following that loop, completely bypassing any remaining iterations regardless of the loop's own condition.

4. **Answer: B.** `continue` skips only whatever code remains in the current iteration's body after the point where `continue` is reached, and then jumps back to re-evaluate the loop's condition to determine whether another iteration should begin.

5. **Answer: B, infinite loop, printing `0` forever.** The loop body never modifies `i`, so `i` remains permanently `0`, and the condition `i < 3` remains permanently `true`, causing the loop to run forever, endlessly printing `0`.

6. **Answer: C, `-1`.** Starting at `5`, subtracting `2` gives `3`, then `1`, then `-1`. At `1`, the condition `x > 0` is still `true`, so the loop runs once more, subtracting `2` again to reach `-1`. At that point, `-1 > 0` is `false`, so the loop stops, leaving `x` at `-1`.

7. **Answer: B.** `while (true)` is entirely valid, compilable Java syntax. The compiler does not reject infinite loops outright; it simply compiles the code as written, and whether the loop actually runs forever at execution time depends on whether any `break`, `return`, or other transfer of control statement exists inside the body to eventually exit it.

8. **Answer: B.** The semicolon directly after `while (x < 5)` is itself the loop's complete, empty body. Since nothing inside that empty body ever modifies `x`, and the problem states nothing else modifies it either, the condition `x < 5` remains permanently `true`, producing an infinite loop that does nothing on every iteration; the block `{ x++; }` afterward is an entirely separate, unrelated statement that is never even reached, since the infinite empty loop before it never terminates.

### 16.8 Programming Practice: Iteration Statement While Statement

1. **Basic.** Write a program using a `while` loop that prints all even numbers from 2 to 20 inclusive.
2. **Intermediate.** Write a program using a `while` loop that computes and prints the sum of the digits of a hardcoded positive integer, repeatedly extracting the last digit using `%` and removing it using integer division by `10`, until the number becomes `0`.
3. **Intermediate.** Write a program using a `while` loop and `continue` that prints every number from 1 to 30 that is NOT divisible by 3, skipping the ones that are.
4. **Advanced.** Write a program using a `while (true)` loop with an internal `break` that simulates searching through a hardcoded array for a target value, printing the index where it was found and immediately breaking out, or printing a not found message if the loop completes without ever finding a match.
5. **Edge case based.** Write a program demonstrating the correct way to count down from a negative starting number to exactly zero using a `while` loop, choosing the correct condition and increment direction so the loop terminates properly, and add a comment explaining what condition would have caused an infinite loop instead.

### 16.9 Programming Solutions: Iteration Statement While Statement

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int num = 2;
        while (num <= 20) {
            System.out.println(num);
            num += 2;
        }
    }
}
```

Why it works: starting at `2` and adding `2` on every iteration naturally produces only even numbers, and the condition `num <= 20` correctly includes `20` itself as the final printed value before the loop terminates.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int number = 4938;
        int sum = 0;
        int original = number;

        while (number > 0) {
            int lastDigit = number % 10;
            sum += lastDigit;
            number /= 10;
        }

        System.out.println("Sum of digits of " + original + " is " + sum);
    }
}
```

Expected output: `Sum of digits of 4938 is 24`

Why it works: `number % 10` isolates the current last digit, adding it to the running total, and `number /= 10` removes that last digit through integer division truncation, so the loop naturally processes one digit per iteration and terminates once `number` has been fully reduced to `0`.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int i = 0;
        while (i < 30) {
            i++;
            if (i % 3 == 0) {
                continue;
            }
            System.out.println(i);
        }
    }
}
```

Why it works: `i` is incremented at the very top of each iteration before the check, and whenever it is evenly divisible by `3`, `continue` skips the print statement for that specific value, correctly leaving only non multiples of `3` printed, from `1` through `30`.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        int[] data = {14, 27, 3, 89, 42, 6};
        int target = 89;
        int index = 0;

        while (true) {
            if (index >= data.length) {
                System.out.println("Target not found");
                break;
            }
            if (data[index] == target) {
                System.out.println("Found target " + target + " at index " + index);
                break;
            }
            index++;
        }
    }
}
```

Expected output: `Found target 89 at index 3`

Why it works: the deliberately infinite `while (true)` loop relies entirely on its two internal `break` conditions, one for successfully finding the target, and one as a safety check for exhausting the entire array without a match, correctly terminating the loop in either possible outcome.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        int count = -5;
        while (count < 0) {
            System.out.println("count = " + count);
            count++;
        }
        System.out.println("Final count: " + count);
        // If the condition had instead been written as "count != 0" while
        // decrementing count with count-- from a negative starting value, the
        // loop would run forever, since decrementing a negative number moves
        // it further away from zero rather than toward it. Using "count < 0"
        // combined with count++ correctly moves count upward toward, and
        // eventually past, zero, guaranteeing termination.
    }
}
```

Expected output:
```
count = -5
count = -4
count = -3
count = -2
count = -1
Final count: 0
```

Why it works: starting at `-5` and incrementing by `1` each iteration correctly moves `count` steadily toward `0`, and the condition `count < 0` correctly stops the loop the moment `count` reaches exactly `0`, guaranteeing proper termination.

---

## 17. Lend A Hand on While Statement

### 17.1 Applied Examples

**Applied Example 1: Nested while loops with break affecting only the inner loop.**

```java
public class Demo {
    public static void main(String[] args) {
        int outer = 1;
        while (outer <= 3) {
            int inner = 1;
            while (inner <= 5) {
                if (inner == 3) {
                    break;
                }
                System.out.println("outer=" + outer + " inner=" + inner);
                inner++;
            }
            outer++;
        }
    }
}
```

Output:
```
outer=1 inner=1
outer=1 inner=2
outer=2 inner=1
outer=2 inner=2
outer=3 inner=1
outer=3 inner=2
```

Explanation: the `break` triggered when `inner == 3` only terminates the inner `while` loop, returning control to the outer loop, which then proceeds normally to its next iteration. This pattern repeats identically for every outer iteration, since the inner loop is freshly reinitialized to `1` each time the outer loop body runs again.

**Applied Example 2: Combining while, continue, and accumulated state.**

```java
public class Demo {
    public static void main(String[] args) {
        int i = 0;
        int sumOfEvens = 0;
        while (i < 10) {
            i++;
            if (i % 2 != 0) {
                continue;
            }
            sumOfEvens += i;
        }
        System.out.println("Sum of even numbers from 1 to 10: " + sumOfEvens);
    }
}
```

Output: `Sum of even numbers from 1 to 10: 30`

Explanation: `continue` skips the accumulation step specifically for odd values of `i`, so only even values, `2, 4, 6, 8, 10`, actually contribute to `sumOfEvens`, and their total is `30`.

### 17.2 Quiz: Lend A Hand on While Statement

1. In Applied Example 1, if `break` were replaced with `continue`, what would change about the output?
   A. Nothing changes  B. `inner=3` would be skipped for each outer value but iterations 4 and 5 would still print  C. The entire program would fail to compile  D. Only one outer iteration would run

2. Given nested while loops where the inner loop uses `continue` when a condition is met, does this affect the outer loop's iteration count?
   A. Yes, it always terminates the outer loop too  B. No, an unlabeled `continue` only affects the loop it is directly inside  C. Only if the condition also checks the outer variable  D. Compile error

3. What is the final value of `sumOfEvens` in Applied Example 2 if the condition were changed to skip even numbers instead of odd numbers?
   A. `30`  B. `25`  C. `55`  D. `0`

### 17.3 Quiz Answers and Reasoning

1. **Answer: B.** With `continue` instead of `break`, when `inner == 3`, only that specific iteration is skipped, jumping back to re-check the inner condition; `inner` continues incrementing normally, so `inner=4` and `inner=5` would still execute and print for each outer value, unlike with `break`, which stopped the inner loop entirely at that point.

2. **Answer: B.** An unlabeled `continue`, exactly like an unlabeled `break`, only affects the loop it is directly and immediately nested within; it has no effect whatsoever on any outer loop unless an explicit label is used to specifically target that outer loop.

3. **Answer: B, `25`.** If odd numbers instead accumulate into the sum, the qualifying values become `1, 3, 5, 7, 9`, which together total `25`.

### 17.4 Programming Practice: Lend A Hand on While Statement

1. Write a program using nested `while` loops to print a simple multiplication table from 1 to 5 for both the outer and inner dimensions, formatted as `outer x inner = result` on each line.
2. Write a program using a `while` loop with `continue` that prints only the prime numbers between 2 and 30, checking each candidate number for divisibility using an inner `while` loop, and skipping non prime candidates via `continue` in the outer loop.

### 17.5 Programming Solutions: Lend A Hand on While Statement

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int outer = 1;
        while (outer <= 5) {
            int inner = 1;
            while (inner <= 5) {
                System.out.println(outer + " x " + inner + " = " + (outer * inner));
                inner++;
            }
            outer++;
        }
    }
}
```

Why it works: the outer loop drives each row of the table from `1` to `5`, and for every single outer value, the inner loop, freshly restarted at `1` each time, drives every column from `1` to `5`, together producing all twenty five combinations of the multiplication table in order.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int candidate = 2;
        while (candidate <= 30) {
            boolean isPrime = true;
            int divisor = 2;
            while (divisor < candidate) {
                if (candidate % divisor == 0) {
                    isPrime = false;
                    break;
                }
                divisor++;
            }
            if (!isPrime) {
                candidate++;
                continue;
            }
            System.out.println(candidate);
            candidate++;
        }
    }
}
```

Why it works: the inner `while` loop tests every possible divisor smaller than the candidate, immediately marking `isPrime` as `false` and breaking out of the inner loop as soon as any exact divisor is found. The outer loop then uses `continue` to skip printing whenever a candidate was found to be non prime, correctly moving straight on to test the next candidate value.

---

## 18. Iteration Statement: Do While Statement

### 18.1 Concept Explanation

The `do while` statement is Java's other basic looping construct, closely related to `while` but with one critical structural difference: it is exit controlled rather than entry controlled, meaning its condition is checked after the loop body executes, not before.

**Basic syntax.**

```java
do {
    // loop body
} while (condition);
```

Notice the required semicolon after the closing parenthesis of the `while` clause; this is mandatory syntax for `do while` and is frequently forgotten, causing a compile error, since `do while` is the only looping construct in Java that requires a trailing semicolon after its condition in this way.

**Guaranteed first execution.** Because the condition is checked only after the body runs, a `do while` loop's body always executes at least once, regardless of whether the condition would have been `true` or `false` if it had been checked beforehand. This is the single most important distinguishing fact about `do while` compared to `while`, and it is the primary reason to choose `do while` over `while` in situations where you specifically need guaranteed at least once execution, such as displaying a menu to a user and then checking whether they want to continue, since the menu must be shown at least one time no matter what.

**Everything else behaves identically to while.** Aside from this entry versus exit controlled distinction, `break`, `continue`, nested loop behavior, and infinite loop risks all work exactly the same way in `do while` as they do in `while`. `continue` inside a `do while` loop jumps to the condition check at the bottom, exactly mirroring how it jumps to the condition check at the top in an ordinary `while` loop.

### 18.2 Exam Perspective

1. **Direct conceptual questions** ask you to state the key structural difference between `while` and `do while`, specifically regarding when the condition is checked.
2. **Output based questions** present a `do while` loop whose condition is `false` from the very start and ask how many times the body executes, testing whether you correctly recognize it still executes exactly once, unlike an equivalent `while` loop, which would execute zero times.
3. **Error identification questions** show a `do while` loop missing its required trailing semicolon after the condition, testing whether you know this specific piece of mandatory syntax.
4. **Scenario based questions** describe a menu display, input validation, or "try at least once" style program requirement, and ask you to identify `do while` as the naturally appropriate looping construct for that specific situation.
5. **Tricky or misleading questions** present a `do while` loop and an equivalent looking `while` loop side by side with a condition that starts out `false`, asking you to correctly predict that their outputs differ specifically in whether the body ran at all.
6. **Questions combining multiple concepts** combine `do while` with `break` and `continue`, requiring the same careful control flow tracing skills developed in the `while` section, applied to the exit controlled structure instead.

### 18.3 Examples

**Example 1: do while executing its body once even with a false initial condition.**

```java
public class Demo {
    public static void main(String[] args) {
        int x = 10;
        do {
            System.out.println("This prints exactly once, x = " + x);
        } while (x < 5);
        System.out.println("Loop finished");
    }
}
```

Output:
```
This prints exactly once, x = 10
Loop finished
```

Explanation: even though `x < 5` is `false` from the very start, since `x` is `10`, the `do while` structure guarantees the body executes at least once before the condition is even checked for the first time. After that single execution, the condition is checked, found `false`, and the loop correctly terminates without any further iterations.

**Example 2: Comparing while and do while with the same false starting condition.**

```java
public class Demo {
    public static void main(String[] args) {
        int a = 20;
        while (a < 5) {
            System.out.println("while body: a = " + a);
        }
        System.out.println("while loop done");

        int b = 20;
        do {
            System.out.println("do while body: b = " + b);
        } while (b < 5);
        System.out.println("do while loop done");
    }
}
```

Output:
```
while loop done
do while body: b = 20
do while loop done
```

Explanation: the `while` loop's condition `a < 5` is checked before any execution and is immediately `false`, so its body never runs at all. The `do while` loop's condition `b < 5` is only checked after its body has already run once, so despite being equally `false`, the body still executes exactly that one guaranteed time before the loop correctly terminates.

**Example 3: Required trailing semicolon.**

```java
public class Demo {
    public static void main(String[] args) {
        int count = 0;
        do {
            System.out.println("count = " + count);
            count++;
        } while (count < 3); // semicolon here is mandatory
    }
}
```

Output:
```
count = 0
count = 1
count = 2
```

Explanation: unlike `while` and `for` loops, whose bodies are simply blocks with no trailing semicolon expected after the condition, `do while` specifically requires a semicolon immediately after the closing parenthesis of its `while (condition)` clause; omitting it is a compile time syntax error.

### 18.4 Important Notes

- `do while` is exit controlled: its condition is checked after the body executes, guaranteeing the body runs at least once, regardless of the condition's initial value.
- A trailing semicolon after the `while (condition)` clause is mandatory syntax for `do while`, unlike for `while` or `for` loops.
- `do while` is the natural, idiomatic choice whenever a task genuinely needs to happen at least one time before any condition based decision about repeating it makes sense, such as menu prompts or initial input validation.
- `break` and `continue` behave in `do while` exactly as they do in `while`, with `continue` jumping down to the condition check at the bottom rather than up to a check at the top.
- Every other looping concept, including nested loop behavior and infinite loop risks from a control variable that never changes appropriately, applies identically to `do while` as it does to `while`.

### 18.5 Scenario Based Understanding

**Scenario A.** A question describes a program that must repeatedly prompt a user to enter a positive number, and must display the prompt at least once even before any input has been checked, and asks which loop construct best fits this requirement.
- What is happening: the prompt must always display at least once, regardless of any condition.
- Which concept is involved: exit controlled looping.
- How to identify it: notice the explicit requirement for guaranteed at least once execution before any condition based decision is even relevant.
- Correct reasoning: `do while` is the naturally fitting construct here, since its body, containing the prompt and input reading logic, always executes first, and only afterward is the condition, whether the input was valid, checked to decide if the prompt needs to repeat.
- Common mistake: using an ordinary `while` loop with a pre-loop initial prompt duplicated before the loop even starts, which technically achieves the same guaranteed first execution but through more awkward, duplicated code, rather than using the construct specifically designed for this exact situation.

**Scenario B.** A question presents a `do while` loop with the mandatory trailing semicolon accidentally omitted and asks what happens when the code is compiled.
- What is happening: a required piece of `do while` specific syntax is missing.
- Which concept is involved: the mandatory trailing semicolon rule unique to `do while`.
- How to identify it: check specifically for a semicolon immediately following the closing parenthesis of the `while (condition)` portion.
- Correct reasoning: the code fails to compile with a syntax error, since this semicolon is not optional or stylistic; it is a required part of the `do while` statement's grammar in Java.
- Common mistake: assuming all looping constructs share identical semicolon requirements, when in fact `while` and `for` loops, whose bodies are simply blocks, have no such trailing semicolon requirement at all.

### 18.6 Quiz: Iteration Statement Do While Statement

1. When is a `do while` loop's condition checked?
   A. Before the body executes  B. After the body executes  C. Both before and after  D. Never, `do while` has no condition

2. If a `do while` loop's condition is `false` from the very beginning, how many times does the body execute?
   A. Zero times  B. Exactly once  C. Infinitely  D. Compile error

3. What is required immediately after the closing parenthesis of a `do while` loop's condition?
   A. Nothing  B. A semicolon  C. An opening brace  D. The keyword `end`

4. Given `int x = 100; do { System.out.println(x); } while (x < 10);`, what is printed?
   A. Nothing  B. `100`  C. Infinite loop  D. Compile error

5. Which scenario is `do while` most naturally suited for?
   A. Iterating a fixed number of times known in advance  B. A situation requiring the body to run at least once before any condition matters, such as a menu prompt  C. Iterating over an array's elements  D. Conditional branching between two blocks

6. What does `continue` do inside a `do while` loop?
   A. Terminates the loop entirely  B. Jumps to the condition check at the bottom of the loop  C. Jumps to the top of the body, skipping the condition check  D. Causes a compile error

7. Which of these correctly completes valid `do while` syntax?
   A. `do { } while (x < 5)`  B. `do { } while (x < 5);`  C. `do while (x < 5) { }`  D. `while (x < 5) do { }`

### 18.7 Quiz Answers and Reasoning

1. **Answer: B.** `do while` is exit controlled, meaning the condition is evaluated only after the loop body has already executed, which is the defining structural difference from `while`.

2. **Answer: B, exactly once.** Regardless of the condition's value, `do while` guarantees the body runs at least one time before the condition is even checked for the first time; only after that first execution does the condition determine whether a second iteration should occur.

3. **Answer: B, a semicolon.** `do while` uniquely requires a trailing semicolon directly after the closing parenthesis of its condition, which is mandatory grammar for this specific looping construct, distinct from `while` and `for`.

4. **Answer: B, `100`.** The body executes once unconditionally, printing `100`, and only afterward is the condition `x < 10` checked, which is `false` since `x` is `100`, so the loop correctly terminates after that single guaranteed execution.

5. **Answer: B.** `do while`'s guaranteed at least once execution makes it the natural, idiomatic fit for situations like displaying a menu or prompt that must appear at least one time regardless of any condition that will only be evaluated afterward.

6. **Answer: B.** `continue` inside a `do while` loop jumps directly to the condition check at the bottom of the loop, mirroring exactly how `continue` in an ordinary `while` loop jumps to the condition check at the top, just relocated to match `do while`'s exit controlled structure.

7. **Answer: B.** `do while` requires the loop body in braces, followed by the `while` keyword and the condition in parentheses, followed by a mandatory trailing semicolon; option A is missing that required semicolon, and options C and D use entirely incorrect keyword ordering that does not match Java's actual grammar for this construct.

### 18.8 Programming Practice: Iteration Statement Do While Statement

1. **Basic.** Write a program using a `do while` loop that prints the numbers 1 through 5.
2. **Intermediate.** Write a program simulating a simple menu system using a `do while` loop, where a hardcoded sequence of menu choices is processed one at a time from an array, printing a different message for each recognized choice, and the loop continues until a designated "exit" choice value is encountered, checked at the bottom of the loop.
3. **Intermediate.** Write a program demonstrating the guaranteed at least once execution property of `do while` by using a starting variable value that would immediately fail the loop's condition if checked first, and confirm through printed output that the body still runs exactly once.
4. **Advanced.** Write a program using a `do while` loop that repeatedly doubles a starting value until it exceeds a hardcoded threshold, printing each doubled value along the way, and add a check to also confirm the loop still executes correctly even if the starting value already exceeds the threshold before the first doubling.

### 18.9 Programming Solutions: Iteration Statement Do While Statement

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int i = 1;
        do {
            System.out.println(i);
            i++;
        } while (i <= 5);
    }
}
```

Why it works: the body prints and increments `i` on every pass, and the condition, checked after each pass, correctly allows exactly five total iterations, covering `i` values `1` through `5`, before `i` becomes `6` and the condition finally fails.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int[] choices = {1, 2, 1, 3};
        int index = 0;
        int choice;

        do {
            choice = choices[index];
            switch (choice) {
                case 1:
                    System.out.println("Viewing profile");
                    break;
                case 2:
                    System.out.println("Viewing settings");
                    break;
                case 3:
                    System.out.println("Exiting menu");
                    break;
                default:
                    System.out.println("Unknown choice");
            }
            index++;
        } while (choice != 3 && index < choices.length);
    }
}
```

Why it works: the loop body always processes at least the first choice before its condition is ever checked, matching the natural feel of a menu that must show something before deciding whether to continue, and the condition checks both for the designated exit choice, `3`, and for safely running out of choices in the array, whichever happens first.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        int x = 50;
        int executionCount = 0;
        do {
            executionCount++;
            System.out.println("Body executed, x = " + x);
        } while (x < 10);
        System.out.println("Total executions: " + executionCount);
    }
}
```

Expected output:
```
Body executed, x = 50
Total executions: 1
```

Why it works: `x < 10` would already be `false` if checked before any execution, since `x` starts at `50`, but because `do while` checks its condition only after the body runs, the body still executes exactly once, which `executionCount` directly confirms.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        int value = 3;
        int threshold = 100;
        do {
            System.out.println("Value: " + value);
            value *= 2;
        } while (value <= threshold);
        System.out.println("Final value exceeding threshold: " + value);
    }
}
```

Expected output:
```
Value: 3
Value: 6
Value: 12
Value: 24
Value: 48
Value: 96
Final value exceeding threshold: 192
```

Why it works: starting from `3` and doubling repeatedly, the loop continues printing and doubling for as long as the resulting value stays at or below the threshold of `100`; once doubling `96` produces `192`, which exceeds the threshold, the condition check after that final doubling correctly stops the loop, having already guaranteed at least that first print of the starting value `3` regardless of the threshold's relationship to it.

---

## 19. Lend A Hand on Do While Statement

### 19.1 Applied Examples

**Applied Example 1: do while combined with break for early exit on invalid data.**

```java
public class Demo {
    public static void main(String[] args) {
        int[] readings = {12, 45, -1, 78, 90};
        int index = 0;
        int sum = 0;

        do {
            int current = readings[index];
            if (current < 0) {
                System.out.println("Invalid reading encountered, stopping");
                break;
            }
            sum += current;
            index++;
        } while (index < readings.length);

        System.out.println("Sum so far: " + sum);
    }
}
```

Output:
```
Invalid reading encountered, stopping
Sum so far: 57
```

Explanation: the loop processes readings one at a time, guaranteed to check the very first reading regardless of any condition, and accumulates valid values into `sum`. As soon as a negative value is encountered, at index `2`, `break` immediately exits the loop entirely, so the readings at index `3` and `4` are never processed, leaving `sum` at `12 + 45`, which is `57`.

**Applied Example 2: Nested do while loops.**

```java
public class Demo {
    public static void main(String[] args) {
        int i = 1;
        do {
            int j = 1;
            do {
                System.out.print(i * j + " ");
                j++;
            } while (j <= 3);
            System.out.println();
            i++;
        } while (i <= 3);
    }
}
```

Output:
```
1 2 3 
2 4 6 
3 6 9 
```

Explanation: each outer iteration guarantees the inner `do while` loop runs at least once, and the inner loop's own condition, checked at its own bottom, independently controls exactly how many times it repeats within that particular outer pass, correctly producing a three by three multiplication grid.

### 19.2 Quiz: Lend A Hand on Do While Statement

1. In Applied Example 1, if the value `-1` at index 2 were instead `33`, so that `readings` had no negative values at all, what would `sum` equal after the loop finishes?
   A. `57`  B. `258`  C. `0`  D. Compile error

2. In Applied Example 2, how many total lines of output are printed?
   A. 1  B. 3  C. 9  D. 6

3. If the outer loop in Applied Example 2 started with `i = 5` and the condition remained `i <= 3`, how many total lines would print?
   A. 0  B. 1  C. 3  D. 5

### 19.3 Quiz Answers and Reasoning

1. **Answer: B, `258`.** With no negative values anywhere in the array, the `break` condition is never triggered, so the loop runs through every single element, guaranteed to check the first one and continuing all the way to the end via its bottom checked condition. The full sum is `12 + 45 + 33 + 78 + 90`, which totals `258`.

2. **Answer: B, 3.** Each full pass of the outer `do while` loop produces exactly one line of printed numbers followed by one `System.out.println()` call, and the outer loop runs for `i` values `1`, `2`, and `3`, producing exactly three total lines.

3. **Answer: B, 1.** Even though `i <= 3` would already be `false` for a starting value of `i = 5`, `do while` guarantees the outer loop's body runs at least once before that condition is ever checked, so exactly one line prints, corresponding to `i = 5`'s inner loop pass, before the outer condition is checked afterward and found `false`, correctly stopping further outer iterations.

### 19.4 Programming Practice: Lend A Hand on Do While Statement

1. Write a program using a `do while` loop combined with `break` that searches through a hardcoded array for the first negative number, printing its index and value as soon as found, or a not found message if the loop completes without ever hitting a negative value.
2. Write a program using nested `do while` loops to print a right triangle pattern of asterisks, with the outer loop controlling the row count from 1 to 5 and the inner loop controlling how many asterisks appear on each row, matching the current row number.

### 19.5 Programming Solutions: Lend A Hand on Do While Statement

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int[] values = {10, 25, 33, -8, 47};
        int index = 0;
        boolean found = false;

        do {
            if (values[index] < 0) {
                System.out.println("First negative value " + values[index] + " found at index " + index);
                found = true;
                break;
            }
            index++;
        } while (index < values.length);

        if (!found) {
            System.out.println("No negative value found");
        }
    }
}
```

Expected output: `First negative value -8 found at index 3`

Why it works: the loop guarantees checking at least the first element, and continues checking subsequent elements until either a negative value triggers the `break`, setting `found` to `true` along the way, or the array is fully exhausted, at which point the `found` flag correctly determines whether the fallback "not found" message should print.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int row = 1;
        do {
            int col = 1;
            StringBuilder line = new StringBuilder();
            do {
                line.append("*");
                col++;
            } while (col <= row);
            System.out.println(line);
            row++;
        } while (row <= 5);
    }
}
```

Expected output:
```
*
**
***
****
*****
```

Why it works: the outer loop guarantees each row from `1` to `5` runs its own inner loop at least once, and the inner loop's condition, `col <= row`, ties the number of asterisks printed on each row directly to that row's own number, correctly building up the right triangle shape one row at a time.

---

## 20. Iteration Statement: For Statement

### 20.1 Concept Explanation

The `for` statement is Java's most structurally compact looping construct, bundling initialization, condition checking, and post iteration update into a single header line. Java actually provides two distinct forms of `for`: the traditional, fully general form, and the enhanced `for each` form introduced in Java 5, specifically designed for iterating over arrays and other collection like structures.

**Traditional for loop syntax.**

```java
for (initialization; condition; update) {
    // loop body
}
```

The three sections inside the parentheses, separated by semicolons, serve distinct roles. The **initialization** section runs exactly once, before the loop begins at all, typically declaring and initializing a loop control variable. The **condition** section is a `boolean` expression checked before every single iteration, including the first, exactly like a `while` loop's condition; if it is `false`, the loop body does not execute for that attempt. The **update** section runs after every complete iteration of the body, typically incrementing or decrementing the control variable, and runs before the condition is checked again for the next potential iteration.

**A traditional for loop is entry controlled, exactly like while.** Since the condition is checked before the very first iteration, a `for` loop whose condition is `false` from the start never executes its body even once, exactly mirroring `while`'s behavior and unlike `do while`.

**Scope of the loop control variable.** A variable declared inside the `for` loop's own initialization section is scoped strictly to the loop itself, including its condition, update, and body, but ceases to exist once the loop finishes entirely; it cannot be referenced afterward.

**Any or all of the three sections can be omitted.** Java allows any of the initialization, condition, or update sections to be left empty, as long as the semicolons separating them remain present. Omitting the condition specifically causes it to be treated as always `true`, which is how the idiom `for (;;) { ... }` creates a deliberately infinite loop, functionally equivalent to `while (true) { ... }`, typically relying on an internal `break` to terminate.

**Multiple variables and multiple update expressions.** The initialization section can declare and initialize more than one variable, as long as they share the same declared type, separated by commas. The update section can likewise contain multiple comma separated update expressions, allowing several control variables to be advanced together within a single loop header.

```java
for (int i = 0, j = 10; i < j; i++, j--) {
    System.out.println("i=" + i + " j=" + j);
}
```

**The enhanced for each loop.** Introduced specifically to simplify iterating over every element of an array or a `Collection` without manually managing an index variable, its syntax is:

```java
for (ElementType element : arrayOrCollection) {
    // uses element
}
```

This form automatically handles retrieving each successive element in order and assigning it to the loop variable, eliminating both the need for an explicit index and the possibility of common off by one indexing mistakes. However, this convenience comes with a meaningful limitation: the enhanced for loop provides no access to the current index itself, and the loop variable it creates is effectively a fresh copy of each element for primitives, meaning modifying that loop variable inside the body does not modify the original array's contents.

### 20.2 Exam Perspective

1. **Direct conceptual questions** ask you to identify the three components of a traditional `for` loop's header and describe each one's specific role and timing.
2. **Output based questions** trace a `for` loop's iterations precisely, including loops with multiple variables or multiple update expressions in the header.
3. **Error identification questions** show a `for` loop referencing its control variable outside the loop's own scope, testing whether you recognize the compile error this causes.
4. **Debugging and tricky questions** present an empty `for (;;)` loop and ask you to correctly identify it as an intentional infinite loop construct rather than a syntax error.
5. **Scenario based questions** compare a traditional indexed `for` loop against an enhanced for each loop solving the same iteration task, testing whether you recognize when the enhanced form is appropriate, specifically when the index itself is not needed, versus when a traditional indexed loop is required, specifically when the index is needed or elements must be modified in place.
6. **Questions combining multiple concepts** test that modifying the loop variable inside an enhanced for each loop's body does not affect the original underlying array, since the loop variable for primitives is a copy of each element's value, not a reference back into the array.
7. **Edge case questions** test a `for` loop with an omitted condition combined with an internal `break`, or a loop with multiple comma separated update expressions advancing in opposite directions simultaneously.

### 20.3 Examples

**Example 1: Standard for loop with a full header.**

```java
public class Demo {
    public static void main(String[] args) {
        for (int i = 1; i <= 5; i++) {
            System.out.println("i = " + i);
        }
    }
}
```

Output:
```
i = 1
i = 2
i = 3
i = 4
i = 5
```

Explanation: `i` is initialized to `1` exactly once, the condition `i <= 5` is checked before every iteration, and `i++` runs after each complete pass through the body, before the condition is checked again for the next potential iteration.

**Example 2: Multiple variables and update expressions in one header.**

```java
public class Demo {
    public static void main(String[] args) {
        for (int i = 0, j = 10; i < j; i++, j--) {
            System.out.println("i=" + i + " j=" + j);
        }
    }
}
```

Output:
```
i=0 j=10
i=1 j=9
i=2 j=8
i=3 j=7
i=4 j=6
```

Explanation: both `i` and `j` are initialized in the same header, and both are updated together on every pass, `i` increasing and `j` decreasing simultaneously, until `i < j` finally becomes `false`, which happens once `i` reaches `5` and `j` reaches `5`.

**Example 3: Enhanced for each loop not modifying the original array.**

```java
public class Demo {
    public static void main(String[] args) {
        int[] numbers = {1, 2, 3};
        for (int n : numbers) {
            n = n * 10;
        }
        for (int n : numbers) {
            System.out.println(n);
        }
    }
}
```

Output:
```
1
2
3
```

Explanation: in the first loop, `n` is a fresh local copy of each array element's value on every iteration; reassigning `n` inside the loop body has no effect whatsoever on the actual elements stored in the `numbers` array itself. The second loop then confirms that the original array remains completely unchanged.

**Example 4: Infinite for loop with internal break.**

```java
public class Demo {
    public static void main(String[] args) {
        int count = 0;
        for (;;) {
            if (count >= 3) {
                break;
            }
            System.out.println("count = " + count);
            count++;
        }
    }
}
```

Output:
```
count = 0
count = 1
count = 2
```

Explanation: omitting all three sections of the `for` header, while still keeping the two required semicolons, produces a deliberately infinite loop, since the missing condition is treated as always `true`. The internal `if` check combined with `break` provides the actual termination logic, exactly mirroring the `while (true)` pattern covered in the previous sections.

### 20.4 Important Notes

- A traditional `for` loop's header has three sections: initialization, runs once; condition, checked before every iteration; and update, runs after every completed iteration.
- `for` is entry controlled, exactly like `while`: a condition that is `false` from the start means the body never executes.
- A variable declared in the `for` loop's own initialization section is scoped strictly to the loop and cannot be accessed after the loop ends.
- Any or all of the three header sections can be omitted; an omitted condition is treated as always `true`, enabling the `for (;;) { ... }` intentional infinite loop idiom.
- Multiple variables of the same type can be declared together in the initialization section, and multiple update expressions can be listed together in the update section, both separated by commas.
- The enhanced for each loop, `for (Type element : arrayOrCollection)`, simplifies element access but provides no index and cannot modify the original array's primitive elements through reassigning the loop variable.
- Use a traditional indexed `for` loop whenever you need the index itself, need to iterate in a non standard order such as backward, or need to actually modify array elements in place; use the enhanced for each loop whenever you simply need to read through every element in order with no index required.

### 20.5 Scenario Based Understanding

**Scenario A.** A question shows an attempt to double every element of an `int` array using an enhanced for each loop, assigning the doubled value back to the loop variable inside the body, and asks whether the array is actually modified.
- What is happening: the loop variable is a copy, not a reference back into the array.
- Which concept is involved: the enhanced for each loop's inability to modify primitive array elements through its loop variable.
- How to identify it: check whether the array itself, using index based access such as `array[i] = ...`, is ever assigned to, versus only the loop variable being reassigned.
- Correct reasoning: since the loop variable holds only a copy of each primitive element's value on every iteration, reassigning that loop variable has no effect on the original array; a traditional indexed `for` loop using `array[i] = array[i] * 2;` would be required to actually achieve the intended modification.
- Common mistake: assuming the enhanced for each loop provides some form of reference back to each array slot, when for primitive element types it fundamentally only ever provides a copied value.

**Scenario B.** A question shows a `for` loop's control variable being referenced in a `System.out.println` statement placed immediately after the closing brace of the loop, and asks whether this compiles.
- What is happening: the loop control variable was declared inside the `for` header itself, scoping it strictly to the loop.
- Which concept is involved: for loop control variable scope.
- How to identify it: check whether the variable was declared inside the `for` header's initialization section specifically, as opposed to being declared before the loop began.
- Correct reasoning: since the variable's scope is limited entirely to the loop, including its condition, update, and body, referencing it in any code after the loop's closing brace is a compile time "cannot find symbol" error.
- Common mistake: assuming a loop control variable behaves like any other local variable declared in the surrounding method and remains accessible afterward, when in fact its scope is specifically and strictly confined to the loop that declared it.

### 20.6 Quiz: Iteration Statement For Statement

1. Which section of a `for` loop's header runs exactly once, before any iteration begins?
   A. Condition  B. Update  C. Initialization  D. Body

2. When is a `for` loop's condition checked?
   A. After the body, every time  B. Before the body, every time, including the first  C. Only once  D. Never

3. What is the output of `for (int i = 5; i < 3; i++) { System.out.println(i); }`?
   A. `5`  B. Infinite loop  C. Nothing prints  D. Compile error

4. Is `for (;;) { }` valid Java syntax?
   A. No, all three sections are mandatory  B. Yes, it produces an infinite loop  C. Only with a single semicolon  D. Only inside a method returning `int`

5. Given `int[] arr = {1, 2, 3}; for (int x : arr) { x++; }`, what are the array's contents after the loop?
   A. `{2, 3, 4}`  B. `{1, 2, 3}`  C. Compile error  D. `{0, 1, 2}`

6. Can a variable declared in a `for` loop's initialization section be accessed after the loop's closing brace?
   A. Yes, always  B. No, its scope is limited to the loop  C. Only if declared `static`  D. Only if the loop runs at least once

7. What is the output of the following?
   ```java
   for (int i = 0, j = 5; i < 3; i++, j -= 2) {
       System.out.println(i + " " + j);
   }
   ```
   A. Three lines: `0 5`, `1 3`, `2 1`  B. Three lines: `0 5`, `1 4`, `2 3`  C. Compile error  D. Infinite loop

8. Which loop form is most appropriate when you need both the current index and the current element's value while iterating an array?
   A. Enhanced for each  B. Traditional indexed for loop  C. do while only  D. Neither can access both

### 20.7 Quiz Answers and Reasoning

1. **Answer: C, initialization.** The initialization section executes exactly one single time, before the loop's condition is ever checked for the first time, typically used to declare and set the starting value of a control variable.

2. **Answer: B.** Exactly like `while`, a traditional `for` loop is entry controlled, meaning its condition is evaluated before every single iteration attempt, including the very first one.

3. **Answer: C, nothing prints.** The condition `i < 3` is checked before the first iteration, and with `i` starting at `5`, this is immediately `false`, so the body never executes even once.

4. **Answer: B.** Omitting all three sections while retaining the two required semicolons is entirely valid Java syntax; the missing condition defaults to always `true`, producing a deliberate infinite loop, typically paired with an internal `break` for controlled termination.

5. **Answer: B, `{1, 2, 3}`, unchanged.** The enhanced for each loop variable `x` holds only a copy of each element's primitive value on every iteration; incrementing `x` inside the body has no effect whatsoever on the actual elements stored in the original `arr` array.

6. **Answer: B.** A variable declared inside a `for` loop's initialization section is scoped strictly to that loop, including its condition, update section, and body, and ceases to exist once the loop's closing brace is reached, making any reference to it afterward a compile time error.

7. **Answer: A.** Tracing carefully: initial values are `i=0, j=5`, printing `0 5`. After the update, `i` becomes `1` and `j -= 2` brings `j` from `5` down to `3`, printing `1 3`. After the next update, `i` becomes `2` and `j` drops from `3` to `1`, printing `2 1`. At that point `i < 3` is still true for `i=2`, but after this iteration's update `i` becomes `3`, which fails the condition, so the loop stops with exactly these three printed lines: `0 5`, `1 3`, `2 1`.

8. **Answer: B, traditional indexed for loop.** Only a traditional indexed `for` loop, using an explicit counter variable such as `i`, provides direct access to both the current index, via the counter variable itself, and the current element's value, via `array[i]`, simultaneously; the enhanced for each loop deliberately omits any index tracking entirely, providing only the element's value on each pass.

### 20.8 Programming Practice: Iteration Statement For Statement

1. **Basic.** Write a program using a traditional `for` loop that prints all multiples of 7 from 7 to 70 inclusive.
2. **Intermediate.** Write a program that uses a traditional indexed `for` loop to find and print both the index and the value of the largest element in a hardcoded `int` array.
3. **Intermediate.** Write a program using an enhanced for each loop to compute and print the sum and average of all elements in a hardcoded `double` array.
4. **Advanced.** Write a program using a single `for` loop header with two variables and two update expressions to print every pair of values as `i` counts up from 0 and `j` counts down from 20, stopping once they meet or cross, on a single loop pass with no additional loops needed.
5. **Edge case based.** Write a program that reverses a hardcoded `int` array in place, meaning without creating a new array, using a traditional indexed `for` loop that swaps elements from opposite ends moving toward the middle, and print the array both before and after the reversal.
6. **Logic intensive.** Write a program using nested traditional `for` loops to print Pascal's triangle up to 6 rows, where each entry is computed using the standard combinatorial relationship rather than a hardcoded lookup, and align the output so it visually resembles a triangle.

### 20.9 Programming Solutions: Iteration Statement For Statement

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        for (int i = 7; i <= 70; i += 7) {
            System.out.println(i);
        }
    }
}
```

Why it works: starting at `7` and incrementing by `7` on every pass naturally produces exactly the multiples of `7`, and the condition `i <= 70` correctly includes `70` itself as the final printed value.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        int[] values = {23, 89, 12, 67, 45, 91, 34};
        int maxIndex = 0;
        for (int i = 1; i < values.length; i++) {
            if (values[i] > values[maxIndex]) {
                maxIndex = i;
            }
        }
        System.out.println("Largest value: " + values[maxIndex] + " at index " + maxIndex);
    }
}
```

Why it works: starting the comparison loop from index `1`, since index `0` is already assumed to be the current best by default, each subsequent element is compared against the current best found so far, updating `maxIndex` whenever a genuinely larger value is encountered, correctly identifying both the value and its position by the time the loop finishes.

**Solution 3.**

```java
public class Solution3 {
    public static void main(String[] args) {
        double[] prices = {19.99, 45.50, 12.25, 88.00, 33.75};
        double sum = 0.0;
        for (double price : prices) {
            sum += price;
        }
        double average = sum / prices.length;
        System.out.println("Sum: " + sum);
        System.out.println("Average: " + average);
    }
}
```

Why it works: the enhanced for each loop is ideal here since only each element's value is needed, with no index required at all, and dividing the accumulated `sum` by `prices.length` correctly computes the average across all elements.

**Solution 4.**

```java
public class Solution4 {
    public static void main(String[] args) {
        for (int i = 0, j = 20; i < j; i++, j--) {
            System.out.println("i=" + i + " j=" + j);
        }
    }
}
```

Why it works: both variables are declared and updated together within one single loop header, `i` counting upward and `j` counting downward simultaneously on every pass, and the loop naturally terminates the moment they meet or cross, requiring no separate nested or nested sequential loop structure at all.

**Solution 5.**

```java
public class Solution5 {
    public static void main(String[] args) {
        int[] data = {10, 20, 30, 40, 50};

        System.out.print("Before: ");
        for (int v : data) {
            System.out.print(v + " ");
        }
        System.out.println();

        int left = 0;
        int right = data.length - 1;
        while (left < right) {
            int temp = data[left];
            data[left] = data[right];
            data[right] = temp;
            left++;
            right--;
        }

        System.out.print("After: ");
        for (int v : data) {
            System.out.print(v + " ");
        }
        System.out.println();
    }
}
```

Expected output:
```
Before: 10 20 30 40 50 
After: 50 40 30 20 10 
```

Thought process: reversing an array in place is most naturally expressed using two pointers moving toward each other from opposite ends, swapping the elements they point to at each step, and stopping once they meet or cross in the middle, which is why a `while` loop with two explicitly tracked index variables reads slightly more naturally here than a single `for` header, though a `for` loop with two update expressions, as shown in Solution 4's technique, would work equally well.

Why it works: each swap correctly exchanges the values at the `left` and `right` positions using a standard temporary variable technique, and advancing `left` inward while retreating `right` inward after every swap ensures every pair of mirrored positions gets swapped exactly once, with the loop stopping naturally once the two pointers meet or cross, at which point the entire array has been fully reversed.

Time complexity: O(n) where n is the array's length, since each element is touched a constant number of times. Space complexity: O(1) beyond the input array itself, since the reversal happens entirely in place using only a single temporary variable per swap.

**Solution 6.**

```java
public class Solution6 {
    public static void main(String[] args) {
        int numRows = 6;
        for (int row = 0; row < numRows; row++) {
            long value = 1;
            for (int col = 0; col <= row; col++) {
                System.out.print(value + " ");
                value = value * (row - col) / (col + 1);
            }
            System.out.println();
        }
    }
}
```

Expected output:
```
1 
1 1 
1 2 1 
1 3 3 1 
1 4 6 4 1 
1 5 10 10 5 1 
```

Thought process: each entry in Pascal's triangle is a binomial coefficient. Rather than computing each coefficient independently from scratch, which would be less efficient, this solution computes each entry incrementally from the previous one within the same row, using the standard relationship where the next binomial coefficient in a row equals the current one multiplied by `(row - col)` and divided by `(col + 1)`.

Why it works: starting each row with `value = 1`, which correctly represents the first entry of every row, and then applying the incremental multiplicative relationship after printing each entry correctly generates every subsequent entry in that same row without needing a separate factorial calculation or a full nested lookup table.

Time complexity: O(n squared) where n is `numRows`, since the inner loop's total work across all outer iterations grows proportionally to the triangular number of entries. Space complexity: O(1) beyond the output itself, since only a single running `value` is maintained per row rather than storing the entire triangle in memory. Edge case to note: using `long` rather than `int` for `value` guards against overflow for a reasonably larger number of rows than requested here, though for very large row counts even `long` would eventually be insufficient.

---

## 21. Lend A Hand on For Statement

### 21.1 Applied Examples

**Applied Example 1: Nested for loops with labeled break reaching an outer loop.**

```java
public class Demo {
    public static void main(String[] args) {
        outerLoop:
        for (int i = 1; i <= 3; i++) {
            for (int j = 1; j <= 3; j++) {
                if (i == 2 && j == 2) {
                    break outerLoop;
                }
                System.out.println("i=" + i + " j=" + j);
            }
        }
        System.out.println("Loops finished");
    }
}
```

Output:
```
i=1 j=1
i=1 j=2
i=1 j=3
i=2 j=1
Loops finished
```

Explanation: an ordinary unlabeled `break` inside the inner loop would only terminate that inner loop, allowing the outer loop to continue to its next iteration. Here, the label `outerLoop:` placed immediately before the outer `for` statement allows `break outerLoop;` to terminate both loops entirely at once, the moment `i` is `2` and `j` is `2`, jumping straight to the statement after the entire labeled outer loop.

**Applied Example 2: Traditional for loop iterating backward.**

```java
public class Demo {
    public static void main(String[] args) {
        int[] items = {10, 20, 30};
        for (int i = items.length - 1; i >= 0; i--) {
            System.out.println("Index " + i + ": " + items[i]);
        }
    }
}
```

Output:
```
Index 2: 30
Index 1: 20
Index 0: 10
```

Explanation: a traditional indexed `for` loop, unlike the enhanced for each form, can freely traverse in any direction, including backward, by initializing the control variable to the last valid index and decrementing down to `0`, something the enhanced for each loop has no built in way to do at all.

### 21.2 Quiz: Lend A Hand on For Statement

1. In Applied Example 1, what would change if `break outerLoop;` were replaced with a plain, unlabeled `break;`?
   A. Nothing changes  B. Only the inner loop would terminate at that point, and the outer loop would continue to `i=3`  C. Compile error  D. Both loops would still terminate identically

2. Can the enhanced for each loop natively iterate an array in reverse order without any additional index tracking?
   A. Yes, using a negative step  B. No, it always iterates forward only  C. Yes, using `break`  D. Only for `String` arrays

3. What must precede a labeled loop for a labeled `break` or `continue` to reference it?
   A. The label followed by a colon, placed directly before the loop statement  B. A `static` keyword  C. Nothing extra is needed  D. An `@Label` annotation

### 21.3 Quiz Answers and Reasoning

1. **Answer: B.** Without the label, `break` only affects the nearest enclosing loop, which is the inner `for (int j ...)` loop. So upon reaching `i=2, j=2`, only the inner loop would terminate for that particular outer pass, and the outer loop would proceed normally to `i=3`, running the inner loop fully again for that iteration, producing considerably more output than the labeled version.

2. **Answer: B.** The enhanced for each loop is specifically designed for straightforward, complete forward traversal only; it provides no mechanism at all for reverse iteration, skipping elements, or any non standard traversal order, all of which require falling back to a traditional indexed `for` loop instead.

3. **Answer: A.** A label consists of a chosen identifier followed immediately by a colon, placed directly before the loop statement it names, which then allows `break labelName;` or `continue labelName;` anywhere inside that loop, including inside nested loops within it, to specifically target that particular labeled loop.

### 21.4 Programming Practice: Lend A Hand on For Statement

1. Write a program using nested `for` loops and a labeled `continue` that searches a small hardcoded two dimensional grid, represented as a 2D array, for the first occurrence of a target value, skipping to the next row entirely as soon as a specific "skip marker" value is found anywhere in the current row, and printing the coordinates where the target was ultimately found.
2. Write a program using a traditional indexed `for` loop to print a hardcoded `String` array both in its original order and in reverse order, without ever creating a second array.

### 21.5 Programming Solutions: Lend A Hand on For Statement

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        int[][] grid = {
            {1, 2, -99, 4},
            {5, 6, 7, 8},
            {9, -99, 11, 12}
        };
        int target = 7;
        boolean found = false;

        rowLoop:
        for (int row = 0; row < grid.length; row++) {
            for (int col = 0; col < grid[row].length; col++) {
                if (grid[row][col] == -99) {
                    continue rowLoop;
                }
                if (grid[row][col] == target) {
                    System.out.println("Found " + target + " at row " + row + ", col " + col);
                    found = true;
                    break rowLoop;
                }
            }
        }

        if (!found) {
            System.out.println("Target not found");
        }
    }
}
```

Expected output: `Found 7 at row 1, col 2`

Why it works: the labeled `continue rowLoop;` immediately abandons the rest of the current row's scanning the moment the skip marker `-99` is encountered anywhere in it, jumping straight to the next row rather than merely skipping to the next column within the same row, which is exactly what an unlabeled `continue` would have done instead. Row `0` contains the skip marker before ever reaching the target, so it is abandoned entirely, and row `1` is then fully scanned and correctly finds the target.

**Solution 2.**

```java
public class Solution2 {
    public static void main(String[] args) {
        String[] words = {"apple", "banana", "cherry", "date"};

        System.out.println("Original order:");
        for (int i = 0; i < words.length; i++) {
            System.out.println(words[i]);
        }

        System.out.println("Reverse order:");
        for (int i = words.length - 1; i >= 0; i--) {
            System.out.println(words[i]);
        }
    }
}
```

Why it works: the first loop traverses the array forward in the conventional manner, while the second loop simply starts its index at the last valid position, `words.length - 1`, and decrements down to `0`, reading and printing the exact same underlying array in the opposite order without ever needing to construct a separate reversed copy of it.

---

## 22. Return Statement with Lend A Hand

### 22.1 Concept Explanation

The `return` statement is Java's primary transfer of control statement for exiting a method and optionally sending a value back to the code that called it. Understanding `return` fully requires connecting it to method return types, control flow inside conditionals and loops, and a few specific interactions with `try finally` blocks that are worth knowing even at this foundational stage.

**Basic syntax.** A method declared with a return type other than `void` must use `return expression;`, where `expression`'s type matches, or can be automatically widened or autoboxed to match, the method's declared return type. A method declared with `void` as its return type either omits `return` entirely, letting execution simply reach the end of the method body naturally, or uses a bare `return;` with no expression, purely to exit the method early.

**Every possible execution path through a non void method must return a value.** The Java compiler performs reachability and definite return analysis on every method body. If there exists even one possible path through the method's control flow, considering every `if`, `else`, loop, and `switch` branch, that does not end in a `return` statement, the compiler rejects the method entirely with a "missing return statement" error. This is why an `if` without a matching `else`, used as the sole content of a method body, generally fails to compile if intended as the method's only logic, since the implicit "condition is false" path would otherwise fall through with no return.

**`return` immediately exits the method, regardless of nesting depth.** Unlike `break`, which only exits the nearest enclosing loop or switch, `return` unconditionally exits the entire method it appears in, no matter how many loops, conditionals, or other nested blocks currently surround it. This makes `return` an effective and common way to exit deeply nested logic early, such as immediately returning as soon as a search loop finds its target, without needing any additional flags or labeled breaks.

**Code after an unconditional `return` is unreachable and a compile error.** If the compiler can prove, through static analysis, that a particular statement can genuinely never be executed because every path leading to it already returned, that statement is flagged as unreachable code, which is a compile time error in Java, not merely a warning as in some other languages.

**Multiple return points within one method are entirely legal.** A method can contain several different `return` statements along different conditional branches, and execution simply exits through whichever one is actually reached at runtime for a given set of inputs. This is a common and often perfectly readable style, sometimes called early return or guard clause style, particularly for validating inputs at the very top of a method before proceeding to its main logic.

**Interaction between `return` and `finally`.** Although exception handling itself belongs to a later topic, it is worth knowing at this stage that a `return` statement inside a `try` block does not immediately exit the method if a `finally` block is also present; the `finally` block's code still executes first, after the `return` expression has been evaluated but before the method actually exits, and in unusual cases a `return` inside `finally` itself can even override an earlier pending return from the `try` block, though writing code that relies on this is considered extremely poor practice.

### 22.2 Exam Perspective

1. **Direct conceptual questions** ask whether a `void` method can use `return;` with no expression, and confirm that it can, purely to exit early.
2. **Error identification questions** show a non void method with a conditional branch that does not return a value on every possible path, testing whether you spot the missing return statement compile error.
3. **Debugging questions** show code placed after an unconditional `return` within the same block, testing whether you recognize this as unreachable code and therefore a compile error rather than simply dead, ignored code.
4. **Scenario based questions** present a search style method using early `return` inside a loop to exit as soon as a match is found, contrasting this with an equivalent version using a flag variable and `break`, testing whether you understand both achieve a similar practical effect but `return` additionally exits the entire method, not just the loop.
5. **Tricky or misleading questions** test whether `return` inside a deeply nested set of loops and conditionals correctly exits all of them at once, unlike `break`, which would need explicit labels to reach beyond its nearest enclosing loop.
6. **Questions combining multiple concepts** combine `return` with `if else if` ladders using multiple return points, requiring you to trace exactly which return statement executes for a given set of input values.

### 22.3 Examples

**Example 1: A void method using return to exit early.**

```java
public class Demo {
    static void printIfPositive(int x) {
        if (x <= 0) {
            System.out.println("Not positive, skipping");
            return;
        }
        System.out.println("Positive value: " + x);
    }

    public static void main(String[] args) {
        printIfPositive(-5);
        printIfPositive(10);
    }
}
```

Output:
```
Not positive, skipping
Positive value: 10
```

Explanation: the bare `return;` inside the `if` block immediately exits the method once a non positive value is detected, preventing the following `System.out.println("Positive value: ...")` line from ever executing for that particular call, without needing an `else` block to achieve the same effect.

**Example 2: A missing return statement failing to compile.**

```java
public class Demo {
    static int classify(int x) {
        if (x > 0) {
            return 1;
        }
        // missing return for the case where x <= 0
    }

    public static void main(String[] args) {
        System.out.println(classify(5));
    }
}
```

Explanation: this fails to compile with a "missing return statement" error, because the compiler recognizes that if `x > 0` is `false`, control reaches the end of the method body with no `return` statement having executed, and since `classify` is declared to return an `int`, every possible path through the method must end in a `return`.

**Example 3: return exiting deeply nested loops immediately.**

```java
public class Demo {
    static int findFirstNegative(int[][] grid) {
        for (int row = 0; row < grid.length; row++) {
            for (int col = 0; col < grid[row].length; col++) {
                if (grid[row][col] < 0) {
                    return grid[row][col];
                }
            }
        }
        return 0;
    }

    public static void main(String[] args) {
        int[][] data = {{1, 2, 3}, {4, -7, 6}, {8, 9, 10}};
        System.out.println(findFirstNegative(data));
    }
}
```

Output: `-7`

Explanation: as soon as the nested loops encounter the first negative value, `return grid[row][col];` immediately exits both the inner and outer loops and the entire method at once, in a single step, without needing any labeled `break` statements at all, directly returning the found value to the caller.

**Example 4: Unreachable code after an unconditional return.**

```java
public class Demo {
    static int alwaysFive() {
        return 5;
        // System.out.println("This line is unreachable"); // would not compile
    }

    public static void main(String[] args) {
        System.out.println(alwaysFive());
    }
}
```

Explanation: any statement placed immediately after an unconditional `return`, within that same reachable block with no intervening branch, can never possibly execute, and the Java compiler specifically detects and rejects this as an "unreachable statement" compile time error, rather than silently allowing dead code to exist uncompiled or simply ignored.

### 22.4 Important Notes

- A `void` method may use a bare `return;` with no expression to exit early; a non void method must use `return expression;` where the expression's type matches or can convert to the declared return type.
- Every possible execution path through a non void method must end in a `return` statement, or the compiler rejects the method with a missing return statement error.
- `return` exits the entire enclosing method immediately, regardless of how many loops, conditionals, or other nested blocks currently surround it, unlike `break`, which only affects its nearest enclosing loop or switch.
- Code that the compiler can prove is never reachable, such as any statement immediately following an unconditional `return` within the same block, is a compile time error in Java, not merely a warning.
- Multiple return points within a single method are entirely legal and are a common, often clear style, particularly for validating inputs early before the method's main logic proceeds.
- A `return` inside a `try` block still allows an accompanying `finally` block's code to execute before the method actually exits; this deeper interaction becomes more fully relevant once exception handling is covered as its own topic.

### 22.5 Scenario Based Understanding

**Scenario A.** A question presents a method meant to search an array and return the index of a target value, using a loop with an `if` check inside, but the `return` statement for the found case is mistakenly placed outside the loop entirely, after it, and asks what the method actually does.
- What is happening: the return statement's placement relative to the loop determines when, and with what value, the method actually exits.
- Which concept is involved: return statement placement and its effect on when a method exits.
- How to identify it: trace exactly which block, the loop body itself versus the code that follows the loop, contains the return statement in question.
- Correct reasoning: if the return meant to report a found index is placed after the loop rather than inside it, the loop will run to completion regardless of whether a match was found partway through, since nothing inside the loop body actually exits the method early, and whatever value ends up being returned will not correctly reflect an early match found during the loop's execution.
- Common mistake: assuming that simply having correct comparison logic somewhere inside the loop is sufficient, without separately verifying that a `return` statement capable of actually exiting the method with the intended value is placed at the correct point within that logic.

**Scenario B.** A question presents a method with an `if else if` ladder, each branch ending in its own `return` statement, and no code at all following the ladder, and asks whether this compiles successfully as a non void method.
- What is happening: every branch of the ladder, including an implicit consideration of whether all possible input cases are truly covered, needs to guarantee a return.
- Which concept is involved: definite return analysis across a full if else if ladder.
- How to identify it: check specifically whether a final, unconditional `else` branch is present, since without one, there could exist an input value for which none of the conditions match.
- Correct reasoning: if the ladder includes a final `else` with its own `return`, every possible path is covered and the method compiles successfully. If the ladder has no final `else`, and only a series of `else if` conditions each with a `return`, the compiler cannot prove that some condition will always match, and will reject the method with a missing return statement error, even if the conditions happen to be logically exhaustive in the programmer's own mind.
- Common mistake: assuming the compiler performs deep logical reasoning about whether a set of conditions is truly exhaustive, when in fact its definite return analysis is comparatively mechanical and specifically requires an unconditional final `else` branch to be satisfied for this kind of ladder structure.

### 22.6 Quiz: Return Statement

1. Can a `void` method contain a `return;` statement with no expression?
   A. No, `void` methods cannot use `return` at all  B. Yes, to exit the method early  C. Only as the very last line  D. Only inside a loop

2. What happens if a non void method has a branch where no return statement is reached?
   A. Returns `null` automatically  B. Returns the default value of the return type  C. Compile time error  D. Runtime exception

3. Does `return` inside a nested loop exit just that loop, or the entire method?
   A. Just the nearest loop  B. The entire method  C. Just the nearest two loops  D. Depends on whether it is labeled

4. What happens when a statement is placed immediately after an unconditional `return` within the same block?
   A. It executes normally  B. It is silently ignored at runtime  C. Compile time "unreachable statement" error  D. Runtime warning only

5. Is it legal for a method to contain more than one `return` statement along different branches?
   A. No, only one return is allowed per method  B. Yes, this is entirely legal and common  C. Only in `void` methods  D. Only inside `switch` statements

6. Given an `if else if` ladder inside a non void method where every branch returns a value but there is no final unconditional `else`, does this compile?
   A. Always compiles  B. Generally does not compile, since the compiler cannot prove every path returns  C. Only compiles for `boolean` return types  D. Compiles only with at least three branches

### 22.7 Quiz Answers and Reasoning

1. **Answer: B.** A `void` method can absolutely use a bare `return;` with no accompanying expression, and doing so is a common, legitimate technique for exiting the method early once some condition has been satisfied, without needing an `else` block to contain the remaining logic.

2. **Answer: C, compile time error.** Java's compiler performs definite return analysis on every non void method, and any possible execution path that does not end in a `return` statement causes a compile time "missing return statement" error; there is no automatic fallback value returned at runtime for such a case.

3. **Answer: B, the entire method.** `return` always exits the entire enclosing method immediately, completely regardless of how many loops or other nested blocks currently surround it, which is a key distinction from `break`, which only ever affects its nearest enclosing loop or switch statement unless explicitly labeled.

4. **Answer: C.** The Java compiler specifically detects code that can be proven, through static analysis, to be genuinely unreachable, such as any statement directly following an unconditional `return` within that same block, and rejects it as a compile time error rather than allowing it to exist silently.

5. **Answer: B.** Multiple return statements within a single method, placed along different conditional branches, are entirely legal Java syntax and a common, often clear coding style, frequently used for early validation or guard clause patterns.

6. **Answer: B.** Without a final unconditional `else` branch, the compiler cannot mechanically prove that every possible input value is guaranteed to match at least one of the `if` or `else if` conditions, so even if the conditions happen to be logically exhaustive from the programmer's perspective, the method generally fails to compile due to the missing return statement rule, unless some other statement, such as an unconditional return, follows the entire ladder.

### 22.8 Programming Practice: Return Statement

1. **Basic.** Write a method that takes an `int` and returns `"Even"` or `"Odd"` as a `String`, using two separate `return` statements along an `if else` structure.
2. **Intermediate.** Write a method using early return guard clauses that validates a hardcoded set of three input parameters representing a rectangle's width, height, and a scaling factor, returning `-1` immediately if any parameter is zero or negative, and otherwise returning the correctly scaled area.
3. **Intermediate.** Write a method that searches a two dimensional `int` array for a target value and returns a two element `int` array containing the row and column where it was found, using `return` to exit immediately upon finding a match, or returns `{-1, -1}` if the value is never found anywhere in the grid.
4. **Advanced.** Write a method classifying a hardcoded triangle's three side lengths into `"Invalid"`, `"Equilateral"`, `"Isosceles"`, or `"Scalene"` using an `if else if` ladder with a return statement in every single branch, including a final unconditional `else`, ensuring the method compiles cleanly with no missing return statement error.

### 22.9 Programming Solutions: Return Statement

**Solution 1.**

```java
public class Solution1 {
    static String classifyParity(int n) {
        if (n % 2 == 0) {
            return "Even";
        } else {
            return "Odd";
        }
    }

    public static void main(String[] args) {
        System.out.println(classifyParity(7));
        System.out.println(classifyParity(12));
    }
}
```

Why it works: the `if else` structure guarantees exactly one of the two return statements always executes for any given `int` input, satisfying the compiler's definite return requirement while keeping the logic simple and directly readable.

**Solution 2.**

```java
public class Solution2 {
    static double scaledArea(double width, double height, double scale) {
        if (width <= 0) {
            return -1;
        }
        if (height <= 0) {
            return -1;
        }
        if (scale <= 0) {
            return -1;
        }
        return width * height * scale;
    }

    public static void main(String[] args) {
        System.out.println(scaledArea(5, 4, 2));
        System.out.println(scaledArea(-5, 4, 2));
    }
}
```

Expected output:
```
40.0
-1.0
```

Why it works: each guard clause independently checks one specific invalid condition and returns immediately if triggered, so by the time execution reaches the final `return width * height * scale;` line, every parameter is already guaranteed to be strictly positive, making the main calculation safe and straightforward without needing any nested `if else` structure at all.

**Solution 3.**

```java
public class Solution3 {
    static int[] findPosition(int[][] grid, int target) {
        for (int row = 0; row < grid.length; row++) {
            for (int col = 0; col < grid[row].length; col++) {
                if (grid[row][col] == target) {
                    return new int[]{row, col};
                }
            }
        }
        return new int[]{-1, -1};
    }

    public static void main(String[] args) {
        int[][] grid = {{4, 8, 15}, {16, 23, 42}, {1, 2, 3}};
        int[] position = findPosition(grid, 23);
        System.out.println("Found at row " + position[0] + ", col " + position[1]);

        int[] notFound = findPosition(grid, 99);
        System.out.println("Result for missing value: " + notFound[0] + ", " + notFound[1]);
    }
}
```

Expected output:
```
Found at row 1, col 1
Result for missing value: -1, -1
```

Why it works: the moment a match is found anywhere in the nested loop traversal, `return new int[]{row, col};` immediately exits both loops and the method itself in one step, directly returning the coordinates; if the entire grid is fully traversed with no match ever found, control naturally reaches the final `return new int[]{-1, -1};` statement after both loops, correctly signaling that the target was not present anywhere in the grid.

**Solution 4.**

```java
public class Solution4 {
    static String classifyTriangle(int a, int b, int c) {
        if (a + b <= c || a + c <= b || b + c <= a) {
            return "Invalid";
        } else if (a == b && b == c) {
            return "Equilateral";
        } else if (a == b || b == c || a == c) {
            return "Isosceles";
        } else {
            return "Scalene";
        }
    }

    public static void main(String[] args) {
        System.out.println(classifyTriangle(3, 3, 3));
        System.out.println(classifyTriangle(4, 4, 7));
        System.out.println(classifyTriangle(5, 6, 7));
        System.out.println(classifyTriangle(1, 2, 10));
    }
}
```

Expected output:
```
Equilateral
Isosceles
Scalene
Invalid
```

Why it works: the ladder checks the triangle inequality validity first, since an invalid combination should be reported before any further classification is attempted, then proceeds through equilateral, isosceles, and finally the unconditional `else` covering scalene, guaranteeing every single possible combination of three side lengths reaches exactly one `return` statement, which is precisely what allows this method to compile cleanly with no missing return statement error.

---

## 23. Final Revision Section

This section consolidates every topic covered in this chunk into one compact reference. Use it for rapid pre exam review, not as your first introduction to any concept; if anything here feels unfamiliar, return to that topic's full section above.

### 23.1 Core Facts Checklist

**Keywords**
- 8 primitive type keywords, plus access, non access, control flow, and exception handling keywords; `true`, `false`, `null` are reserved literals, not technically keywords.
- `goto` and `const` are reserved but unimplemented; using either as an identifier is a compile error.
- `var`, `yield`, `record`, `sealed`, `permits`, `non-sealed` are context sensitive; valid as identifiers outside their special grammatical position.

**Primitive types**

| Type | Size | Default | Notes |
|---|---|---|---|
| `byte` | 8 bit | 0 | -128 to 127 |
| `short` | 16 bit | 0 | -32768 to 32767 |
| `int` | 32 bit | 0 | ±2147483647 range |
| `long` | 64 bit | 0L | needs `L` suffix for literals beyond int range |
| `float` | 32 bit | 0.0f | needs `f` suffix |
| `double` | 64 bit | 0.0d | default for decimal literals |
| `char` | 16 bit | '\u0000' | unsigned, 0 to 65535 |
| `boolean` | JVM dependent | false | no int/boolean conversion in Java |

- Integer overflow wraps silently, two's complement; no exception thrown.
- Fields get default values automatically; local variables never do, and must be definitely assigned before use.
- `byte`/`short`/`char` arithmetic promotes to `int`.

**Variables and literals**
- Instance (per object), static (per class, shared), local (per block, no default, strict scope).
- Leading `0` means octal; `0x` hexadecimal; `0b` binary. Underscores allowed only strictly between digits.
- Shadowing: local name hides outer field name; `this.field` or `ClassName.field` reaches the hidden one. Two locals cannot shadow each other in overlapping scope.

**Casting**
- Widening (small to large): automatic, no cast needed. Chain: `byte→short→int→long→float→double`; `char` widens only to `int`/`long`/`float`/`double`.
- Narrowing: requires explicit cast; truncates fractional part (never rounds) for float/double to int; clamps to MAX/MIN for out of range float to int, NaN becomes 0; wraps around (keeps low bits) for int to int narrowing.
- Cast operator binds only to the immediately following operand.

**Arithmetic, unary, relational**
- Integer division truncates; division by int zero throws `ArithmeticException`; float division by zero gives `Infinity`/`NaN`, no exception.
- `%` result takes the sign of the dividend.
- `+` concatenates when either operand is a `String`; evaluation is strictly left to right, so operand order changes the outcome.
- Postfix `x++` yields old value then updates; prefix `++x` updates then yields new value.

**Logical and bitwise**
- `&&`/`||` short circuit; `&`/`|` never short circuit, even on `boolean` operands.
- `&`, `|`, `^` are boolean operators on `boolean` operands and bitwise operators on integer operands.
- `~x` equals `-(x + 1)`. `x ^ x` is `0`; `x ^ 0` is `x`.

**Shift and assignment**
- `<<` fills with 0 from the right; multiply by 2^n.
- `>>` sign extends (preserves sign bit); `>>>` always fills with 0, discarding sign.
- Shift amount is taken modulo bit width: mod 32 for `int`, mod 64 for `long`.
- Every compound assignment operator (`+=`, `-=`, etc.) implicitly inserts a narrowing cast back to the left operand's type.

**Precedence** (high to low, abbreviated): postfix `++`/`--` → unary/cast → `* / %` → binary `+ -` → shift → relational/`instanceof` → `== !=` → `&` → `^` → `|` → `&&` → `||` → `?:` → assignment.
- Among `&`, `^`, `|`: `&` binds tightest, then `^`, then `|`.
- Assignment and `?:` are right to left associative; almost everything else is left to right.

**Statements**
- Only assignment, `++`/`--`, method call, and object creation expressions can stand alone as expression statements.
- A stray `;` is a valid, empty statement; commonly causes silent logic bugs after `if` or loop headers.
- Braceless bodies cover only the single next statement.

**If / Switch**
- `if` condition must be genuine `boolean`; no int-as-truthy.
- First matching condition in an `if else if` ladder wins; later conditions never evaluated.
- Order overlapping range checks from most to least restrictive.
- `else` binds to the nearest unmatched `if` (dangling else) when braces are omitted.
- Switch permits `byte`/`short`/`char`/`int` (+wrappers), `String`, `enum`; never `long`/`float`/`double`/`boolean`.
- `case` labels must be compile time constants.
- Traditional switch falls through without `break`; arrow (`->`) switch does not fall through and can yield a value (`yield` inside a block branch).
- Switching on `null` String throws `NullPointerException`.

**While / Do While / For**
- `while` and `for` are entry controlled (condition checked before body, may run zero times); `do while` is exit controlled (condition checked after body, always runs at least once, requires trailing `;`).
- `break` exits the nearest enclosing loop entirely; `continue` skips to the next condition check.
- Labeled `break`/`continue` can target an outer loop.
- `for` loop control variable is scoped strictly to the loop.
- Enhanced for each provides no index and cannot modify primitive array elements through its loop variable.

**Return**
- Every path through a non void method must return a value, or it is a compile error.
- `return` exits the entire method immediately, regardless of nesting depth, unlike `break`.
- Code immediately after an unconditional `return` in the same block is unreachable and a compile error.

### 23.2 High Frequency Traps Table

| Trap | What actually happens |
|---|---|
| `if (x = 5)` | Compile error in Java (assignment yields `int`, not `boolean`) |
| `010` as a literal | Octal, equals decimal 8, not 10 |
| `float f = 3.5;` | Compile error, decimal literals default to `double` |
| `0.1 + 0.2 == 0.3` | `false`, floating point imprecision |
| `Math.abs(Integer.MIN_VALUE)` | Still negative, overflow |
| `byte b = 10; b += 300;` | Compiles (hidden cast), wraps to `54` |
| `x << 32` for `int x` | No effect, shift amount mod 32 is 0 |
| Missing `break` in switch | Falls through to next case(s) |
| `if (cond);` | Semicolon is the empty body; block after runs unconditionally |
| `for (int i=0; ...) {} ... println(i);` after loop | Compile error, `i` out of scope |
| `&` vs `&&` with a risky right operand | `&` always evaluates the right side; can throw when `&&` would have short circuited |

### 23.3 Exam Strategy Tips

- For any expression mixing `+` with `String` and numeric operands, physically trace left to right, writing down each intermediate result; never trust a single mental pass.
- Whenever `++`/`--` appears more than once on the same variable in one expression, write out each sub step separately before combining them.
- When you see `&` or `|` next to a risky right operand (division, array access, method call with a side effect), check specifically whether short circuiting would have mattered.
- For `if else if` ladders with overlapping numeric ranges, always check whether conditions are ordered from most to least restrictive.
- For switch questions, scan every case from the matching label downward until you hit a `break`, `return`, or the end of the switch, since fall through is the default in the traditional form.
- For loop questions, first identify whether the loop is entry controlled (`while`, `for`) or exit controlled (`do while`) before predicting how many times the body runs.
- For casting questions, first identify whether the conversion is widening or narrowing, then whether both types are integer, both floating point, or mixed, since the specific truncation, wraparound, or clamping rule depends on that combination.