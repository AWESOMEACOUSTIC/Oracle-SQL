# Java Competitive Exam Preparation
## Chunk 8: Exception Handling

This chunk introduces a new running scenario: **StreamFlix**, a live streaming platform in the spirit of Jio Hotstar. The switch away from the banking scenario is deliberate: exception handling is best taught through situations that genuinely go wrong in unpredictable ways, and a streaming platform naturally has plenty of them, an expired subscription, a device limit reached, age restricted content, a malformed rating input. Every class built here still uses the full object oriented toolkit from Chunks 2 through 4: encapsulation, inheritance, abstract classes, interfaces, and composition, so this chunk also doubles as a fresh, integrated review of that material in a new setting.

---

## Table of Contents

1. Introduction to Exceptions with Lend A Hand
2. Exception Hierarchy
3. Exception Handling and Try-Catch-Finally
4. Execution Flow When Exceptions Are Raised
5. Multiple Catch Block and Nested Try Block
6. Throws Keyword with Lend A Hand
7. Throw Keyword with Lend A Hand
8. Print Stack Trace and How to Analyze It
9. User Defined Exception with Lend A Hand
10. Consolidated Quiz: All Eight Chunks Together
11. Programming Practice and Solutions
12. Revision Summary

---

## 1. Introduction to Exceptions with Lend A Hand

### Setting Up the Scenario

StreamFlix needs a content hierarchy, exactly the kind of design Chunk 4 covered. `Content` is naturally abstract: nothing is ever just a bare "content" in the app, it is always specifically a movie, a live match, or some other concrete kind.

```java
public abstract class Content {
    private String title;
    private int durationMinutes;
    protected boolean ageRestricted;

    public Content(String title, int durationMinutes, boolean ageRestricted) {
        this.title = title;
        this.durationMinutes = durationMinutes;
        this.ageRestricted = ageRestricted;
    }

    public String getTitle() { return title; }
    public int getDurationMinutes() { return durationMinutes; }
    public boolean isAgeRestricted() { return ageRestricted; }

    public abstract String getContentType();
}

public interface Downloadable {
    void download();
}

public class Movie extends Content implements Downloadable {
    public Movie(String title, int durationMinutes, boolean ageRestricted) {
        super(title, durationMinutes, ageRestricted);
    }

    @Override
    public String getContentType() { return "Movie"; }

    @Override
    public void download() {
        System.out.println(getTitle() + " downloaded for offline viewing.");
    }
}

public class LiveMatch extends Content {
    private String teams;

    public LiveMatch(String title, int durationMinutes, String teams) {
        super(title, durationMinutes, false);
        this.teams = teams;
    }

    @Override
    public String getContentType() { return "Live Match"; }
}
```

`Movie` implements `Downloadable`, since movies can be saved offline; `LiveMatch` does not, since a live broadcast cannot meaningfully be downloaded, exactly the kind of capability based design Chunk 4's interface coverage was built around. A `User` HAS-A `Subscription`, a composition relationship, also from Chunk 4.

```java
public class Subscription {
    private String tier;
    private boolean active;
    private int maxConcurrentStreams;

    public Subscription(String tier, boolean active, int maxConcurrentStreams) {
        this.tier = tier;
        this.active = active;
        this.maxConcurrentStreams = maxConcurrentStreams;
    }

    public boolean isActive() { return active; }
    public int getMaxConcurrentStreams() { return maxConcurrentStreams; }
}

public class User {
    private String username;
    private int age;
    private Subscription subscription;
    private int activeStreams;

    public User(String username, int age, Subscription subscription) {
        this.username = username;
        this.age = age;
        this.subscription = subscription;
        this.activeStreams = 0;
    }

    public String getUsername() { return username; }
    public int getAge() { return age; }
    public Subscription getSubscription() { return subscription; }
    public int getActiveStreams() { return activeStreams; }
    public void incrementStreams() { activeStreams++; }
    public void decrementStreams() { activeStreams--; }
}
```

### Concept Explanation

An exception is an object representing an abnormal condition that disrupts a program's normal, expected flow of execution. Earlier chunks already used a simple, manual way to signal failure: Chunk 2's `BankAccount.withdraw` returned `false` when a withdrawal could not proceed. That approach works well when a method's caller is expected to routinely check for an ordinary, anticipated failure. It breaks down for problems that can occur almost anywhere, unpredictably, deep inside unrelated code, such as dereferencing a reference that turns out to be `null`, or dividing by a value that turns out to be zero. Java's own built in operations already raise exceptions for exactly this kind of problem, entirely without any custom code asking them to.

```java
public class Demo {
    public static void main(String[] args) {
        Content content = null;
        System.out.println(content.getTitle());
    }
}
```

Output:
```
Exception in thread "main" java.lang.NullPointerException
	at Demo.main(Demo.java:4)
```

`content` is `null`, so attempting `content.getTitle()` cannot possibly succeed; there is no object there to call a method on. The JVM itself detects this and raises a `NullPointerException`, an object representing exactly this specific failure, entirely without any explicit code in `Demo` asking for it. Since nothing in this program handles that exception, the program terminates abruptly, printing the error to the console. The rest of this chunk covers how to detect and respond to situations like this deliberately, rather than letting them crash the program outright.

### Important Notes

- An exception is an object, an instance of a class, representing a specific abnormal condition.
- Some exceptions, like `NullPointerException`, are raised automatically by the JVM when an operation cannot possibly succeed; others, covered later in this chunk, are raised deliberately by a program's own code for its own business specific error conditions.
- An unhandled exception terminates the thread it occurred on, printing diagnostic information to the console by default; for a single threaded program's `main` thread, this means the entire program stops.
- Exceptions exist specifically to handle problems that can arise unpredictably, in contrast to the manual, boolean return based signaling from earlier chunks, which suits problems a caller specifically expects to check for at one particular point.

### Lend A Hand: Quiz

1. What kind of thing is a Java exception, fundamentally?
   A. A special kind of primitive  B. An object, an instance of a class representing an abnormal condition  C. A compiler directive  D. A keyword

2. What happens to a single threaded program if an exception is thrown and never handled anywhere?
   A. The line is skipped and execution continues  B. The program terminates, printing diagnostic information  C. It compiles into a warning only  D. The exception is silently ignored

3. Why did `content.getTitle()` in the example above throw an exception rather than simply doing nothing?
   A. `getTitle()` is broken  B. `content` was `null`, so there was no actual object to call a method on, and the JVM detects this automatically  C. `Content` is abstract  D. Compile error

### Answers

1. **B.** Every exception in Java is a genuine object, an instance of some class in the `Throwable` hierarchy, covered fully in Section 2, carrying information about what went wrong.

2. **B.** With nothing to catch it, an exception propagates all the way up, terminating the thread it occurred on; for `main`, that means the whole program stops, after the JVM prints diagnostic information about the failure.

3. **B.** `null` means "no object here at all"; calling any method through a `null` reference is fundamentally impossible to carry out, and the JVM raises `NullPointerException` specifically to signal this, entirely automatically, with no custom code required.

### Programming Practice

1. Write a short program that declares a `Content` reference, leaves it `null`, and calls `getContentType()` on it, then run it mentally and describe, in a comment, exactly what output you expect to see.

### Solution

```java
public class Solution1 {
    public static void main(String[] args) {
        Content content = null;
        System.out.println(content.getContentType());
        // Expected: a NullPointerException is thrown and printed to the
        // console, terminating the program before "after" would ever print,
        // since content has no actual object to call getContentType() on.
    }
}
```

Why it works: this directly reproduces the exact failure pattern from the concept explanation, confirming that calling any method, including one declared `abstract` in `Content`, on a `null` reference always fails the same way, regardless of which specific method is called.

---

## 2. Exception Hierarchy

### Concept Explanation

Every exception in Java is an object belonging to a class within a single, specific hierarchy rooted at `Throwable`, which, exactly as covered in Chunk 3, itself ultimately extends `Object`.

```
Object
 └── Throwable
      ├── Error
      │    └── (StackOverflowError, OutOfMemoryError, and others)
      └── Exception
           ├── RuntimeException (unchecked)
           │    └── (NullPointerException, ArrayIndexOutOfBoundsException,
           │         ArithmeticException, ClassCastException,
           │         NumberFormatException, and others)
           └── (IOException and other checked exceptions)
```

**`Error` versus `Exception`.** `Error` represents serious problems generally considered outside a normal program's ability to meaningfully recover from, such as `StackOverflowError`, from runaway recursion, or `OutOfMemoryError`, from the JVM genuinely running out of heap space. Ordinary application code is not expected to catch these. `Exception` represents conditions a well written program can reasonably anticipate and respond to, and is the branch this entire chunk focuses on.

**Checked versus unchecked exceptions.** `Exception` itself splits further into two practically very different categories. Any subclass of `RuntimeException` is an **unchecked** exception: the compiler does not require calling code to handle or declare it in any way. Every other subclass of `Exception`, meaning anything that is not also a `RuntimeException`, is a **checked** exception: the compiler actively enforces that calling code either catches it or explicitly declares that it might propagate further, using the `throws` keyword covered in Section 6. `NullPointerException` from Section 1, along with `ArrayIndexOutOfBoundsException`, `ArithmeticException`, `ClassCastException`, and `NumberFormatException` from Chunk 6, are all unchecked, all being subclasses of `RuntimeException`. `IOException`, representing failures reading or writing files or network connections, is a classic checked exception.

### Important Notes

- Every exception, and every error, is ultimately a `Throwable`, which is ultimately an `Object`.
- `Error` represents serious, generally unrecoverable JVM level problems; application code is not expected to catch it.
- `RuntimeException` and its subclasses are unchecked: the compiler places no requirement on handling them.
- Any `Exception` subclass that is not also a `RuntimeException` subclass is checked: the compiler requires calling code to either handle it or declare it.
- `NullPointerException`, `ArrayIndexOutOfBoundsException`, `ArithmeticException`, `ClassCastException`, and `NumberFormatException` are all unchecked, all descending from `RuntimeException`.

### Quick Check

Is `NullPointerException` checked or unchecked, and how can you tell from its position in the hierarchy?

Unchecked. It is a subclass of `RuntimeException`, and every subclass of `RuntimeException`, no matter how many levels removed, is unchecked by definition; the compiler never requires it to be caught or declared.

---

## 3. Exception Handling and Try-Catch-Finally

### Concept Explanation

Exception handling means writing code that anticipates a specific exception might occur and responds to it deliberately, rather than letting it propagate uncontrolled. The core tool for this is the `try` block, paired with one or more `catch` blocks, and optionally a `finally` block.

```java
try {
    // code that might throw an exception
} catch (SomeExceptionType e) {
    // code that runs only if that specific exception type was thrown
} finally {
    // code that always runs, whether an exception occurred or not
}
```

**Basic handling.**

```java
public class Demo {
    public static void main(String[] args) {
        Content content = null;
        try {
            System.out.println(content.getTitle());
        } catch (NullPointerException e) {
            System.out.println("Could not read content: it does not exist.");
        }
        System.out.println("Program continues normally.");
    }
}
```

Output:
```
Could not read content: it does not exist.
Program continues normally.
```

Instead of crashing, the exception is caught, a friendly message is printed, and execution continues normally afterward, exactly the behavior a real application needs.

**The `finally` block: guaranteed cleanup.** Code inside `finally` runs no matter what happens in the `try` block, whether it completes normally, throws an exception that gets caught, or even throws an exception that is never caught at all. This makes `finally` the reliable tool for cleanup work, in sharp contrast to Chunk 7's `finalize()`, which was explicitly shown to be unreliable and unpredictably timed. `finally` has none of those problems: it always runs, deterministically, immediately after the `try` or `catch` block finishes.

```java
public class Demo {
    public static void main(String[] args) {
        System.out.println("Connecting to stream...");
        try {
            int[] episodeIndices = {0, 1, 2};
            System.out.println(episodeIndices[5]); // throws ArrayIndexOutOfBoundsException
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Requested episode does not exist.");
        } finally {
            System.out.println("Connection closed.");
        }
    }
}
```

Output:
```
Connecting to stream...
Requested episode does not exist.
Connection closed.
```

"Connection closed." prints unconditionally, regardless of whether the array access succeeded or failed, exactly the deterministic guarantee `finally` provides.

### Important Notes

- `try` wraps code that might throw; `catch` specifies how to respond to a specific exception type; `finally` always runs afterward, regardless of outcome.
- `finally` is the reliable alternative to Chunk 7's `finalize()`: it runs deterministically and immediately, never depending on garbage collection timing.
- A `try` block can exist with just a `catch`, just a `finally`, or both together; at least one of the two must be present.
- Catching an exception stops it from propagating further up the call chain, letting the program continue past the `try` block normally.

### Quick Check

If the array access in the example above had succeeded instead, such as `episodeIndices[1]`, would `"Connection closed."` still print?

Yes. `finally` runs after the `try` block regardless of whether an exception occurred at all; a successful `try` block simply skips every `catch` clause and proceeds directly to `finally`, which still executes exactly as it would if an exception had been caught.

---

## 4. Execution Flow When Exceptions Are Raised

### Concept Explanation

Understanding precisely what the JVM does the instant an exception is thrown is essential for correctly tracing more complex code. The sequence is exact and mechanical.

1. An exception object is created, either automatically by the JVM or explicitly by code using `throw`, covered in Section 7.
2. Execution of the current method stops immediately at that point; no further statements in that method run, not even ones that appear to come "right after" on the same line.
3. The JVM searches the enclosing `try` block, if one exists, for a `catch` clause matching the exception's type.
4. If a matching `catch` is found, its code runs, then any `finally` block runs, and execution continues with whatever follows the entire `try`-`catch`-`finally` structure.
5. If no matching `catch` exists in the current method, any `finally` block still runs first, and then the exception propagates to the caller, the method that called this one, exactly as if it had been thrown from the point of that call instead.
6. This propagation, called stack unwinding, repeats up through the entire chain of method calls until a matching `catch` is found somewhere, or the exception reaches the very top, `main`, uncaught, terminating the program.

**Demonstrating propagation through three levels of calls.**

```java
public class Demo {
    static void validateSubscription(User user) {
        if (!user.getSubscription().isActive()) {
            throw new IllegalStateException(user.getUsername() + "'s subscription is inactive.");
        }
    }

    static void startStream(User user) {
        validateSubscription(user);
        System.out.println("Streaming started for " + user.getUsername());
    }

    static void playContent(User user) {
        startStream(user);
    }

    public static void main(String[] args) {
        User user = new User("meera", 25, new Subscription("Basic", false, 1));
        try {
            playContent(user);
        } catch (IllegalStateException e) {
            System.out.println("Playback failed: " + e.getMessage());
        }
    }
}
```

Output: `Playback failed: meera's subscription is inactive.`

`validateSubscription` throws the exception. It has no `try` block of its own, so it propagates immediately to `startStream`, which also has no relevant `try` block, so it propagates again to `playContent`, which likewise has none, propagating once more to `main`, which finally has a matching `catch`. Every intermediate method's remaining code, the `System.out.println` inside `startStream` included, never runs at all, since execution left each of those methods the instant the exception was thrown, well before reaching that line.

### Important Notes

- Throwing an exception immediately halts the current method's execution at that exact point; nothing after it in that method runs.
- If no `catch` matches within the current method, the exception propagates to the caller, exactly as if it had originated from that call site instead.
- `finally` blocks along the way still run during propagation, even though the corresponding `try`'s own `catch` clauses did not match.
- An exception that propagates all the way out of `main` uncaught terminates the program.

### Quick Check

In the three level example above, does `startStream`'s `System.out.println("Streaming started for ...")` line ever execute?

No. `validateSubscription`, called as the very first statement inside `startStream`, throws before control ever returns to `startStream`, so the line after that call is never reached at all.

---

## 5. Multiple Catch Block and Nested Try Block

### Concept Explanation

**Multiple catch blocks.** A single `try` can be followed by several `catch` clauses, each handling a different exception type, letting a program respond differently depending on exactly what went wrong.

```java
public static void requestRating(String rawRating) {
    try {
        int rating = Integer.parseInt(rawRating);
        int scaled = 100 / rating;
        System.out.println("Scaled rating: " + scaled);
    } catch (NumberFormatException e) {
        System.out.println("Rating must be a number.");
    } catch (ArithmeticException e) {
        System.out.println("Rating cannot be zero.");
    }
}
```

The JVM checks each `catch` clause in order, top to bottom, and runs the first one whose type matches the thrown exception.

**Order matters: most specific first.** If one `catch` clause's type is a superclass of another's, the more specific, subclass type must be listed first. Listing a superclass catch before a subclass catch makes the subclass catch unreachable, and is a compile time error, a direct application of the class hierarchy reasoning from Chunk 3 and Chunk 4.

```java
try {
    // ...
} catch (RuntimeException e) {
    // catches everything below
} catch (NumberFormatException e) { // compile error: unreachable, already caught above
    // ...
}
```

Since `NumberFormatException` is a subclass of `RuntimeException`, from Section 2's hierarchy, the first, broader `catch` would already handle every `NumberFormatException` too, making the second `catch` clause impossible to ever reach, which the compiler rejects outright.

**Multi-catch syntax.** When two or more exception types should be handled identically, they can be combined into a single `catch` clause using `|`, avoiding duplicated code.

```java
try {
    int rating = Integer.parseInt(rawRating);
    int scaled = 100 / rating;
} catch (NumberFormatException | ArithmeticException e) {
    System.out.println("Invalid rating input: " + e.getMessage());
}
```

**Nested try blocks.** A `try` block can contain another complete `try` block inside it. The inner `try`'s own `catch` clauses handle problems local to that inner section without disturbing the outer flow at all; only an exception the inner `try` does not catch propagates outward to the outer `try`'s own `catch` clauses, following the exact same propagation rule from Section 4.

```java
public static void processWatchSession(String[] rawIndices) {
    try {
        System.out.println("Session starting.");
        for (String raw : rawIndices) {
            try {
                int index = Integer.parseInt(raw);
                System.out.println("Watched episode index " + index);
            } catch (NumberFormatException e) {
                System.out.println("Skipping invalid entry: " + raw);
            }
        }
        System.out.println("Session complete.");
    } finally {
        System.out.println("Session resources released.");
    }
}
```

The inner `try` handles a malformed individual entry locally, letting the loop continue to the next entry rather than aborting the entire session, while the outer `try`'s `finally` still guarantees session cleanup regardless of how the inner loop went.

### Important Notes

- Multiple `catch` clauses are checked top to bottom; the first matching one runs.
- A more specific, subclass exception type must be caught before a more general, superclass type, or the compiler rejects the unreachable subclass catch.
- Multi-catch, using `|`, handles several exception types identically in one clause, avoiding duplicated handling code.
- A nested `try` handles local problems without disturbing the surrounding flow; only exceptions it does not itself catch propagate outward to any enclosing `try`.

### Quick Check

Why must `NumberFormatException` be caught before `RuntimeException` if both appear as separate `catch` clauses for the same `try`?

Because `NumberFormatException` is a subclass of `RuntimeException`, as shown in Section 2's hierarchy; a `catch (RuntimeException e)` clause would already match any `NumberFormatException` too, so placing it first would make a later, more specific `NumberFormatException` clause unreachable, which the compiler does not allow.

---

## 6. Throws Keyword with Lend A Hand

### Concept Explanation

The `throws` keyword appears in a method's signature, exactly the signature concept from Chunk 2, to declare that this method might propagate a checked exception without handling it itself, placing the responsibility on whoever calls it instead.

```java
public class SubscriptionExpiredException extends Exception {
    public SubscriptionExpiredException(String message) {
        super(message);
    }
}
```

```java
public class StreamingService {
    public void startStream(User user, Content content) throws SubscriptionExpiredException {
        if (!user.getSubscription().isActive()) {
            throw new SubscriptionExpiredException(user.getUsername() + "'s subscription has expired.");
        }
        user.incrementStreams();
        System.out.println("Streaming " + content.getTitle() + " for " + user.getUsername());
    }
}
```

Since `SubscriptionExpiredException` extends `Exception` directly, not `RuntimeException`, it is checked, exactly the classification from Section 2. Because `startStream` might throw it without catching it internally, the method signature must declare `throws SubscriptionExpiredException`, or the code fails to compile.

**The compiler enforces handling at every call site.** Any code calling `startStream` must either catch `SubscriptionExpiredException` itself, or declare `throws SubscriptionExpiredException` on its own enclosing method too, propagating the same obligation further outward.

```java
public class Demo {
    public static void main(String[] args) {
        StreamingService service = new StreamingService();
        User user = new User("arjun", 22, new Subscription("Basic", false, 1));
        Movie movie = new Movie("Nature's Wonders", 90, false);

        try {
            service.startStream(user, movie);
        } catch (SubscriptionExpiredException e) {
            System.out.println("Cannot start stream: " + e.getMessage());
        }
    }
}
```

Without this `try catch`, or without `main` itself declaring `throws SubscriptionExpiredException`, this code would simply fail to compile, since a checked exception can never be silently ignored by the compiler the way an unchecked one can.

### Important Notes

- `throws ExceptionType` in a method signature declares that the method might propagate that exception without handling it internally.
- This is only ever required, or even meaningful, for checked exceptions; declaring `throws` for an unchecked exception is legal but has no enforcement effect, since the compiler never required it in the first place.
- Every caller of a method declaring `throws SomeCheckedException` must either catch it or declare the same `throws` themselves, or the code fails to compile.
- `throws` in a signature is a declaration; it does not itself cause an exception to be thrown, which is the job of the `throw` keyword, covered next.

### Lend A Hand: Quiz

1. Why does `startStream` need `throws SubscriptionExpiredException` in its signature?
   A. It is optional stylistic decoration  B. Because `SubscriptionExpiredException` is checked, and the method does not catch it internally, so the compiler requires this declaration  C. All methods require `throws`  D. Because the method is `public`

2. What happens if `Demo.main` calls `service.startStream(user, movie)` without a `try catch` and without declaring `throws SubscriptionExpiredException` on `main` itself?
   A. Compiles fine, exception ignored  B. Compile time error  C. Runtime exception only  D. Warning only

3. Is `throws` ever required for an unchecked exception like `NullPointerException`?
   A. Yes, always  B. No, the compiler places no such requirement on unchecked exceptions  C. Only in `main`  D. Only for custom exceptions

### Answers

1. **B.** Checked exceptions are precisely the category the compiler actively enforces handling for; since `startStream` throws one without catching it, declaring `throws` is mandatory here.

2. **B.** Leaving a checked exception neither caught nor declared anywhere along the call chain is a compile time error, exactly the enforcement Section 6 describes as unique to checked exceptions.

3. **B.** Unchecked exceptions, being subclasses of `RuntimeException`, carry no compiler enforced handling requirement at all; `throws` can technically still be written for one, purely as documentation, but it changes nothing about compilation.

### Programming Practice

1. Write a method `validateAge(User user, Content content) throws AgeRestrictedException` (declare a simple checked `AgeRestrictedException` extending `Exception` for this exercise) that throws it if `content.isAgeRestricted()` is `true` and `user.getAge()` is under `18`, and a `main` method that calls it inside a `try catch`, testing both an underage and an adult user.

### Solution

```java
public class AgeRestrictedException extends Exception {
    public AgeRestrictedException(String message) {
        super(message);
    }
}
```

```java
public class Solution1 {
    static void validateAge(User user, Content content) throws AgeRestrictedException {
        if (content.isAgeRestricted() && user.getAge() < 18) {
            throw new AgeRestrictedException(user.getUsername() + " is not old enough for " + content.getTitle());
        }
        System.out.println("Access granted for " + user.getUsername());
    }

    public static void main(String[] args) {
        Movie restricted = new Movie("Late Night Thriller", 110, true);
        User teen = new User("kiran", 15, new Subscription("Premium", true, 2));
        User adult = new User("divya", 30, new Subscription("Premium", true, 2));

        try {
            validateAge(teen, restricted);
        } catch (AgeRestrictedException e) {
            System.out.println("Blocked: " + e.getMessage());
        }

        try {
            validateAge(adult, restricted);
        } catch (AgeRestrictedException e) {
            System.out.println("Blocked: " + e.getMessage());
        }
    }
}
```

Expected output:
```
Blocked: kiran is not old enough for Late Night Thriller
Access granted for divya
```

Why it works: `validateAge` declares `throws AgeRestrictedException` since it is checked and not caught internally, forcing every caller, here `main`, to handle it explicitly, exactly the enforcement described in this section.

---

## 7. Throw Keyword with Lend A Hand

### Concept Explanation

While `throws` declares that a method might propagate an exception, `throw` is the actual statement that raises one, at the precise point where a problem is detected.

```java
throw new StreamLimitExceededException("Concurrent stream limit reached for " + user.getUsername());
```

`throw` always operates on a single, specific exception object, constructed with `new` exactly like any other object from Chunk 2, and immediately triggers the exact propagation behavior described in Section 4: the current method halts at that point, and the JVM begins searching for a matching `catch`.

**`throw` versus `throws`.** These are easy to visually confuse but serve entirely different roles: `throws` is a method signature declaration, stating what might happen; `throw` is an executable statement, an action that actually makes it happen, appearing inside a method's body, not its signature.

```java
public class StreamLimitExceededException extends RuntimeException {
    public StreamLimitExceededException(String message) {
        super(message);
    }
}
```

```java
public void startStream(User user, Content content) {
    if (user.getActiveStreams() >= user.getSubscription().getMaxConcurrentStreams()) {
        throw new StreamLimitExceededException("Concurrent stream limit reached for " + user.getUsername());
    }
    user.incrementStreams();
}
```

Since `StreamLimitExceededException` extends `RuntimeException`, it is unchecked, so `startStream`'s signature needs no `throws` declaration at all for it, even though the method body genuinely does `throw` it under the right condition.

### Important Notes

- `throw` is a statement that raises one specific exception object at the exact point it appears; `throws` is a signature level declaration that a checked exception might propagate.
- `throw` requires an actual `Throwable` object, almost always constructed fresh with `new` at the point of the throw.
- Throwing an unchecked exception requires no corresponding `throws` declaration, though throwing a checked exception without catching it internally does.
- Once `throw` executes, the rest of that method's body is skipped entirely, exactly the halt-and-propagate behavior from Section 4.

### Lend A Hand: Quiz

1. What is the key difference between `throw` and `throws`?
   A. They are interchangeable  B. `throw` is a statement that raises a specific exception; `throws` is a signature declaration stating a checked exception might propagate  C. `throws` raises exceptions; `throw` declares them  D. `throw` only works with unchecked exceptions

2. Does `startStream`'s use of `throw new StreamLimitExceededException(...)` require the method to also declare `throws StreamLimitExceededException`?
   A. Yes, always required  B. No, since `StreamLimitExceededException` is unchecked, no `throws` declaration is required  C. Only if called from `main`  D. Compile error without it

3. What must follow the `throw` keyword?
   A. A `String` message only  B. An actual `Throwable` object, typically constructed with `new`  C. A class name only  D. Nothing, `throw` alone is valid

### Answers

1. **B.** This is precisely the distinction Section 7 draws: `throw` acts, raising one specific exception instance; `throws` merely declares a possibility in a method's signature.

2. **B.** Unchecked exceptions carry no compiler enforced declaration requirement at all, exactly as covered in Section 6; `StreamLimitExceededException` being a `RuntimeException` subclass means `throw`ing it needs no accompanying `throws`.

3. **B.** `throw` must be followed by an expression evaluating to an actual `Throwable` object; a bare `throw;` with nothing after it is not valid Java syntax outside of a very specific, unrelated rethrow context involving `catch` parameters.

### Programming Practice

1. Write a method `checkStreamLimit(User user)` that throws `StreamLimitExceededException` if the user's active streams already equal or exceed their subscription's limit, and demonstrate it being called without any `try catch` at all, since the exception is unchecked, letting it propagate uncaught to observe the default crash output.

### Solution

```java
public class Solution1 {
    static void checkStreamLimit(User user) {
        if (user.getActiveStreams() >= user.getSubscription().getMaxConcurrentStreams()) {
            throw new StreamLimitExceededException("Limit reached for " + user.getUsername());
        }
        System.out.println("Stream allowed for " + user.getUsername());
    }

    public static void main(String[] args) {
        User user = new User("sana", 28, new Subscription("Basic", true, 1));
        user.incrementStreams();
        checkStreamLimit(user);
        System.out.println("This line never runs.");
    }
}
```

Expected output:
```
Exception in thread "main" StreamLimitExceededException: Limit reached for sana
	at Solution1.checkStreamLimit(Solution1.java:5)
	at Solution1.main(Solution1.java:12)
```

Why it works: since `StreamLimitExceededException` is unchecked, `main` compiles perfectly fine without any `try catch` at all; at runtime, once `user`'s single stream slot is already used, `checkStreamLimit` throws, and with nothing to catch it, the program terminates exactly as Section 1 first demonstrated, with `"This line never runs."` correctly never printing.

---

## 8. Print Stack Trace and How to Analyze It

### Concept Explanation

When an exception is printed, either automatically by an uncaught exception terminating the program, or explicitly via `e.printStackTrace()` inside a `catch` block, Java prints a stack trace: a structured report showing exactly where the exception occurred and the full chain of method calls that led there.

```java
public class Demo {
    static void validateSubscription(User user) {
        if (!user.getSubscription().isActive()) {
            throw new IllegalStateException("Subscription inactive for " + user.getUsername());
        }
    }

    static void startStream(User user) {
        validateSubscription(user);
    }

    public static void main(String[] args) {
        User user = new User("neel", 20, new Subscription("Basic", false, 1));
        try {
            startStream(user);
        } catch (IllegalStateException e) {
            e.printStackTrace();
        }
    }
}
```

Sample output:
```
java.lang.IllegalStateException: Subscription inactive for neel
	at Demo.validateSubscription(Demo.java:4)
	at Demo.startStream(Demo.java:9)
	at Demo.main(Demo.java:14)
```

**Reading a stack trace.** The very first line states the exception's full class name and its message, exactly the message passed to its constructor. Each following line, prefixed `at`, is one stack frame, and together they list the call chain, top to bottom, from where the exception was actually created and thrown down to the entry point that ultimately led there. The topmost `at` line is always exactly where the exception originated, `validateSubscription`, at line 4. The bottommost is the outermost call in the chain that led there, `main`, at line 14.

**How to use this to debug.** When investigating a real failure, start reading from the top: the first `at` line belonging to your own code, as opposed to a library's internal code, is almost always the most direct clue to the actual bug's location. Reading further down traces exactly how execution arrived there, which is often just as useful for understanding why the problem occurred, not merely where.

**`Caused by`: chained exceptions.** Sometimes one exception occurs while handling another, and a stack trace will include a `Caused by:` section beneath the main one, showing the original, underlying exception that ultimately triggered the one actually being reported.

**Other ways to access this information.** `e.getMessage()` returns just the message string, without the full trace. `e.getStackTrace()` returns the frame information as an array of `StackTraceElement` objects, for programmatic inspection, rather than the human readable printed format `printStackTrace()` produces directly.

### Important Notes

- The first line of a stack trace is the exception's class and message; every following `at` line is one stack frame, ordered from where the exception was thrown, at the top, down to the outermost calling context, at the bottom.
- `printStackTrace()` sends this report to the console; `getMessage()` returns just the message text; `getStackTrace()` returns the frame data as an array for programmatic use.
- The topmost frame belonging to your own application code is usually the most direct clue to a bug's actual location.
- `Caused by:` sections indicate one exception occurred while another was already being handled, showing the original underlying failure beneath the one actually caught.

### Quick Check

In the sample stack trace above, which line corresponds to where the exception was actually thrown, and which corresponds to where the outermost, originating call happened?

`at Demo.validateSubscription(Demo.java:4)`, the topmost line, is exactly where `throw new IllegalStateException(...)` executed. `at Demo.main(Demo.java:14)`, the bottommost line, is where the very first call in this particular chain, `startStream(user)`, was made from.

---

## 9. User Defined Exception with Lend A Hand

### Concept Explanation

A custom, user defined exception is an ordinary class, following exactly the same class design principles from Chunks 2 through 4, that extends either `Exception`, to create a checked exception, or `RuntimeException`, to create an unchecked one. Extending `Throwable` or `Error` directly is possible but essentially never appropriate for application level custom exceptions.

**Building StreamFlix's exception types.**

```java
public class SubscriptionExpiredException extends Exception {
    public SubscriptionExpiredException(String message) {
        super(message);
    }
}

public class StreamLimitExceededException extends RuntimeException {
    public StreamLimitExceededException(String message) {
        super(message);
    }
}

public class AgeRestrictedContentException extends RuntimeException {
    public AgeRestrictedContentException(String message) {
        super(message);
    }
}
```

Every one of these constructors calls `super(message)`, exactly the constructor chaining pattern from Chunk 3, delegating to `Exception`'s, or `RuntimeException`'s, own constructor, which ultimately stores the message inside `Throwable` itself, making it retrievable later through `getMessage()`, exactly as demonstrated throughout this chunk.

**Choosing checked versus unchecked for a custom exception.** This is a genuine design decision, not an arbitrary one. `SubscriptionExpiredException` was made checked specifically because it represents an expected, recoverable business condition that every caller of `startStream` genuinely ought to be forced to consider handling, exactly the compiler enforcement from Section 6. `StreamLimitExceededException` and `AgeRestrictedContentException` were made unchecked because they represent conditions closer to a caller's own logic error, calling `startStream` without first checking eligibility, where forcing every single caller everywhere to explicitly handle or declare them would add more ceremony than genuine value.

**Putting the full `StreamingService` together.**

```java
public class StreamingService {
    public void startStream(User user, Content content) throws SubscriptionExpiredException {
        if (!user.getSubscription().isActive()) {
            throw new SubscriptionExpiredException(user.getUsername() + "'s subscription has expired.");
        }
        if (content.isAgeRestricted() && user.getAge() < 18) {
            throw new AgeRestrictedContentException(content.getTitle() + " is age restricted.");
        }
        if (user.getActiveStreams() >= user.getSubscription().getMaxConcurrentStreams()) {
            throw new StreamLimitExceededException("Concurrent stream limit reached for " + user.getUsername());
        }
        user.incrementStreams();
        System.out.println("Streaming " + content.getTitle() + " for " + user.getUsername());
    }

    public void stopStream(User user) {
        user.decrementStreams();
    }
}
```

```java
public class Demo {
    public static void main(String[] args) {
        StreamingService service = new StreamingService();
        User user = new User("tanvi", 16, new Subscription("Premium", true, 2));
        Movie movie = new Movie("Midnight Heist", 120, true);

        try {
            service.startStream(user, movie);
        } catch (SubscriptionExpiredException e) {
            System.out.println("Subscription problem: " + e.getMessage());
        } catch (AgeRestrictedContentException e) {
            System.out.println("Age restriction problem: " + e.getMessage());
        } finally {
            service.stopStream(user);
            System.out.println("Stream slot released. Active streams: " + user.getActiveStreams());
        }
    }
}
```

Output:
```
Age restriction problem: Midnight Heist is age restricted.
Stream slot released. Active streams: 0
```

`SubscriptionExpiredException`, being checked, must be explicitly caught, exactly as `throws` on `startStream` requires. `AgeRestrictedContentException`, being unchecked, could technically have been left uncaught, but is caught anyway here since the program specifically wants to handle it gracefully. `stopStream` in `finally` runs regardless of which, if any, exception occurred, correctly releasing the stream slot even though `incrementStreams()` was never actually reached for this particular attempt, since `activeStreams` starts at `0` and no successful start ever occurred here to increment it, so `decrementStreams()` here simply keeps it safely at `0`.

### Important Notes

- A custom exception is an ordinary class extending `Exception`, for checked, or `RuntimeException`, for unchecked; extending `Throwable` or `Error` directly is essentially never appropriate.
- Custom exception constructors typically accept a message and forward it via `super(message)`, exactly the constructor chaining principle from Chunk 3, making it available later through `getMessage()`.
- Choosing checked versus unchecked is a genuine design decision: checked for conditions every caller should be forced to consider, unchecked for conditions closer to a caller's own logic error.
- Multiple custom exception types can be handled with separate `catch` clauses on the same `try`, exactly the multiple catch block mechanism from Section 5, letting each failure mode receive its own tailored response.

### Lend A Hand: Quiz

1. Why does `SubscriptionExpiredException extends Exception` rather than `RuntimeException`?
   A. Arbitrary choice  B. It represents an expected, recoverable business condition every caller of `startStream` should be forced to consider handling, which checked exception enforcement provides  C. `Exception` is required for all custom exceptions  D. `RuntimeException` cannot have constructors

2. What does calling `super(message)` inside a custom exception's constructor accomplish?
   A. Nothing, it is optional decoration  B. It delegates to `Exception`'s or `RuntimeException`'s own constructor, ultimately storing the message in `Throwable` so `getMessage()` can retrieve it later  C. It prints the message immediately  D. It creates a new exception object

3. In the `Demo` example, why does `stopStream(user)` run even though `service.startStream(...)` threw an exception?
   A. It should not run, this is a bug  B. Because it is called inside `finally`, which always runs regardless of whether an exception occurred, exactly as covered in Section 3  C. Coincidence  D. Compile error

### Answers

1. **B.** This is precisely the design reasoning from this section: checked status forces every caller to genuinely consider this specific, recoverable business scenario, which is exactly the intent behind making it checked rather than unchecked.

2. **B.** `super(message)` forwards the message up through the constructor chain to `Throwable`'s own storage for it, which is what every later call to `getMessage()` or `printStackTrace()` actually reads from.

3. **B.** `stopStream(user)` sits inside the `finally` block, which Section 3 established always runs after a `try`, regardless of whether the `try` completed normally or an exception was thrown and caught.

### Programming Practice

1. Write a method `addRating(Content content, String rawRating)` that parses `rawRating` with `Integer.parseInt`, throws a new custom checked exception `InvalidRatingException` if the parsed value is outside `1` to `5` inclusive, and otherwise prints a confirmation message, then write a `main` method testing it with a valid rating, an out of range rating, and a non numeric rating, handling all outcomes appropriately.

### Solution

```java
public class InvalidRatingException extends Exception {
    public InvalidRatingException(String message) {
        super(message);
    }
}
```

```java
public class Solution1 {
    static void addRating(Content content, String rawRating) throws InvalidRatingException {
        int rating;
        try {
            rating = Integer.parseInt(rawRating);
        } catch (NumberFormatException e) {
            throw new InvalidRatingException("Rating must be a number, got: " + rawRating);
        }
        if (rating < 1 || rating > 5) {
            throw new InvalidRatingException("Rating must be between 1 and 5, got: " + rating);
        }
        System.out.println("Rated " + content.getTitle() + " with " + rating + " stars.");
    }

    public static void main(String[] args) {
        Movie movie = new Movie("Ocean Deep", 100, false);
        String[] inputs = {"4", "9", "great"};

        for (String input : inputs) {
            try {
                addRating(movie, input);
            } catch (InvalidRatingException e) {
                System.out.println("Rejected: " + e.getMessage());
            }
        }
    }
}
```

Expected output:
```
Rated Ocean Deep with 4 stars.
Rejected: Rating must be between 1 and 5, got: 9
Rejected: Rating must be a number, got: great
```

Why it works: this combines a nested try, catching `NumberFormatException` internally and converting it into the custom `InvalidRatingException`, with a checked custom exception whose `throws` declaration forces the outer loop's `try catch` to handle every rejection path uniformly, correctly processing all three test inputs through the same consistent error handling structure.

---

## 10. Consolidated Quiz: All Eight Chunks Together

1. `Movie extends Content implements Downloadable`, both from this chunk. Which earlier chunk established that a class can extend one class while implementing any number of interfaces simultaneously?
   A. Chunk 2  B. Chunk 3  C. Chunk 4  D. Chunk 5

2. `User` HAS-A `Subscription`, a composition relationship. Which chunk introduced HAS-A as distinct from IS-A?
   A. Chunk 2  B. Chunk 3  C. Chunk 4  D. Chunk 6

3. `SubscriptionExpiredException`'s constructor calls `super(message)`. Which chunk's constructor chaining concept does this directly reuse?
   A. Chunk 2  B. Chunk 3  C. Chunk 5  D. Chunk 7

4. Section 3 contrasted `finally` with Chunk 7's `finalize()`. Which is the accurate contrast?
   A. Both are equally unreliable  B. `finally` runs deterministically and immediately; `finalize()`'s timing is never guaranteed and depends on garbage collection  C. `finalize()` is more reliable  D. They are unrelated concepts with nothing to compare

5. `Integer.parseInt(rawRating)` inside `addRating` can throw `NumberFormatException`. Which chunk first introduced this specific exception?
   A. Chunk 4  B. Chunk 5  C. Chunk 6  D. Chunk 7

6. If `startStream`'s `if (content.isAgeRestricted() && user.getAge() < 18)` check used `&` instead of `&&`, would this change whether `AgeRestrictedContentException` could ever be thrown for a non age restricted movie?
   A. Yes, significantly  B. No; since neither side of this particular condition has a side effect or risk of throwing, `&` and `&&` would behave identically here, exactly the distinction Chunk 1 drew between short circuit necessity and mere style  C. Compile error  D. `&` is invalid here

7. `Movie`'s `download()` method, from `Downloadable`, is an ordinary instance method. If a `Content` typed reference actually pointing to a `Movie` object called an overridden method, would that resolve via the same dynamic dispatch mechanism from Chunk 3?
   A. No, interfaces are unrelated to that mechanism  B. Yes, interface implemented and class overridden methods both resolve dynamically by the object's actual runtime type, exactly as Chunk 4 established  C. Only for `static` methods  D. Compile error

8. In the `StreamingService.startStream` example, `user.incrementStreams()` modifies the `User` object's own private state from outside the `User` class. How is this possible given `activeStreams` is `private`?
   A. It should not be possible, this is a bug  B. `incrementStreams()` is a public method defined inside `User` itself, so it has full access to `User`'s own private fields, exactly the encapsulation pattern from Chunk 2  C. `private` does not apply here  D. Compile error

### Consolidated Quiz Answers

1. **C.** Chunk 4 established that Java's single inheritance restriction applies only to classes, `extends`, while a class may `implement` any number of interfaces at once, exactly the combination `Movie` uses here.

2. **C.** Chunk 4's IS-A and HAS-A section defined composition, one class holding another as a field, as HAS-A, distinct from the IS-A relationship inheritance and interface implementation describe; `User` holding a `Subscription` field is a direct example.

3. **B.** Chunk 3 introduced `super(...)` constructor chaining, and every custom exception constructor in this chunk reuses that exact mechanism to delegate message storage up to `Throwable`.

4. **B.** This is precisely Section 3's point: `finally` is deterministic and immediate, entirely unlike `finalize()`, whose unreliable, unpredictable timing was established in Chunk 7.

5. **C.** Chunk 6's wrapper class coverage first introduced `NumberFormatException` as the runtime exception thrown by `Integer.parseInt` and similar parsing methods for malformed input.

6. **B.** Both `content.isAgeRestricted()` and `user.getAge() < 18` are plain, side effect free boolean expressions here, so, exactly as Chunk 1 established, `&` and `&&` would produce identical results in this specific case; the short circuiting distinction only matters when the right operand has a side effect or a risk of throwing, neither of which applies here.

7. **B.** Chunk 4 explicitly extended Chunk 3's dynamic dispatch mechanism to interface implemented methods, establishing that both resolve identically, by the object's actual runtime type, regardless of whether the reference used to call them is typed as a class or an interface.

8. **B.** This is ordinary encapsulation from Chunk 2: `incrementStreams()` is a method defined inside `User`'s own class body, so, exactly like `BankAccount`'s own methods accessing its private fields, it has unrestricted access to every field of its own class, entirely independent of where the method is called from.

---

## 11. Programming Practice and Solutions

### Practice Problems

1. **Basic.** Write a method `getEpisode(String[] episodes, int index)` that safely returns the episode at `index`, catching `ArrayIndexOutOfBoundsException` internally and returning `"Episode not found"` instead of letting it propagate.
2. **Intermediate.** Write a method `castToMovie(Content content)` that attempts `(Movie) content`, catching `ClassCastException` and printing a friendly message if `content` is not actually a `Movie`, demonstrating this with both a genuine `Movie` and a `LiveMatch` passed in.
3. **Intermediate.** Write a full `watchSession(User user, Content content, StreamingService service)` method that calls `startStream` inside a `try`, catches `SubscriptionExpiredException`, `AgeRestrictedContentException`, and `StreamLimitExceededException` with three separate, correctly ordered `catch` clauses, each printing a distinct message, and always calls `stopStream` in `finally`.
4. **Advanced.** Extend `StreamingService` with a `batchStart(User user, Content[] items)` method that attempts to start every item in the array in a loop, using a nested `try` per item so that one failure does not stop the rest of the batch from being attempted, and returns a count of how many streams successfully started.

### Solutions

**Solution 1.**

```java
public class Solution1 {
    static String getEpisode(String[] episodes, int index) {
        try {
            return episodes[index];
        } catch (ArrayIndexOutOfBoundsException e) {
            return "Episode not found";
        }
    }

    public static void main(String[] args) {
        String[] episodes = {"Pilot", "The Rise", "Finale"};
        System.out.println(getEpisode(episodes, 1));
        System.out.println(getEpisode(episodes, 10));
    }
}
```

Expected output:
```
The Rise
Episode not found
```

Why it works: the valid index returns normally, while the out of range index triggers `ArrayIndexOutOfBoundsException`, caught locally and converted into a friendly fallback value rather than propagating and crashing the caller.

**Solution 2.**

```java
public class Solution2 {
    static void castToMovie(Content content) {
        try {
            Movie movie = (Movie) content;
            System.out.println("Cast succeeded: " + movie.getTitle());
        } catch (ClassCastException e) {
            System.out.println("Cannot cast " + content.getContentType() + " to Movie.");
        }
    }

    public static void main(String[] args) {
        Content movie = new Movie("Skyline", 105, false);
        Content match = new LiveMatch("Finals Night", 180, "Team A vs Team B");

        castToMovie(movie);
        castToMovie(match);
    }
}
```

Expected output:
```
Cast succeeded: Skyline
Cannot cast Live Match to Movie.
```

Why it works: casting a reference whose actual object genuinely is a `Movie` succeeds without issue, while attempting the same cast on an object that is actually a `LiveMatch` throws `ClassCastException` at runtime, correctly caught and reported rather than crashing the program.

**Solution 3.**

```java
public class Solution3 {
    static void watchSession(User user, Content content, StreamingService service) {
        try {
            service.startStream(user, content);
        } catch (SubscriptionExpiredException e) {
            System.out.println("Subscription issue: " + e.getMessage());
        } catch (AgeRestrictedContentException e) {
            System.out.println("Age restriction issue: " + e.getMessage());
        } catch (StreamLimitExceededException e) {
            System.out.println("Stream limit issue: " + e.getMessage());
        } finally {
            service.stopStream(user);
        }
    }

    public static void main(String[] args) {
        StreamingService service = new StreamingService();
        User user = new User("rohit", 30, new Subscription("Basic", true, 1));
        Movie movie = new Movie("Cosmic Journey", 95, false);

        watchSession(user, movie, service);
        System.out.println("Active streams after session: " + user.getActiveStreams());
    }
}
```

Expected output:
```
Streaming Cosmic Journey for rohit
Active streams after session: 0
```

Why it works: since `SubscriptionExpiredException` is checked, this `try` block is mandatory for `watchSession` to compile at all; the other two catches handle the unchecked possibilities gracefully as well, and `finally` guarantees `stopStream` runs, correctly returning the active stream count to `0` even after a fully successful stream, since the session has ended.

**Solution 4.**

```java
public class Solution4 {
    static int batchStart(StreamingService service, User user, Content[] items) {
        int successCount = 0;
        for (Content item : items) {
            try {
                try {
                    service.startStream(user, item);
                    successCount++;
                } catch (SubscriptionExpiredException e) {
                    System.out.println("Skipping " + item.getTitle() + ": " + e.getMessage());
                }
            } catch (RuntimeException e) {
                System.out.println("Skipping " + item.getTitle() + ": " + e.getMessage());
            }
        }
        return successCount;
    }

    public static void main(String[] args) {
        StreamingService service = new StreamingService();
        User user = new User("ishita", 15, new Subscription("Premium", true, 5));
        Content[] items = {
            new Movie("Adventure Land", 100, false),
            new Movie("Late Night Thriller", 110, true),
            new LiveMatch("Championship", 150, "Red vs Blue")
        };

        int started = batchStart(service, user, items);
        System.out.println("Successfully started: " + started);
    }
}
```

Expected output:
```
Skipping Late Night Thriller: Late Night Thriller is age restricted.
Successfully started: 2
```

Why it works: the inner `try` catches the checked `SubscriptionExpiredException` specifically, satisfying the compiler's requirement, while the outer `try` catches any remaining unchecked `RuntimeException`, covering both `AgeRestrictedContentException` and `StreamLimitExceededException` together with one broader clause, exactly the multiple catch block reasoning from Section 5. Each item is attempted independently, so one age restricted failure does not prevent the two eligible items from starting successfully.

---

## 12. Revision Summary

| Concept | Key Rule | Where It Appeared |
|---|---|---|
| What an exception is | An object representing an abnormal condition, disrupting normal flow | `NullPointerException` on `content.getTitle()` |
| Exception hierarchy | `Throwable` → `Error` / `Exception`; `RuntimeException` subclasses are unchecked, everything else under `Exception` is checked | The hierarchy diagram |
| try-catch-finally | `try` wraps risky code; `catch` handles a specific type; `finally` always runs, deterministically, unlike `finalize()` | `startStream` with `finally` releasing the slot |
| Execution flow | Throwing halts the current method immediately; unmatched exceptions propagate up the call chain until caught or the program ends | Three level `validateSubscription` propagation |
| Multiple catch | Checked top to bottom; subclass types must precede superclass types, or a compile error results | `NumberFormatException` then `ArithmeticException` |
| Nested try | An inner try's unhandled exceptions propagate to any enclosing try, exactly like ordinary propagation | `processWatchSession`'s per-entry inner try |
| throws | A signature declaration that a checked exception might propagate; enforced by the compiler at every call site | `startStream(...) throws SubscriptionExpiredException` |
| throw | The statement that actually raises one specific exception object | `throw new StreamLimitExceededException(...)` |
| Stack trace | Top line is the exception and message; `at` lines run from where it was thrown down to the outermost call | `e.printStackTrace()` output |
| Custom exceptions | Extend `Exception` for checked, `RuntimeException` for unchecked; call `super(message)` | `SubscriptionExpiredException`, `StreamLimitExceededException` |

**Exam tip.** For any question involving a custom or built in exception, first classify it: is it, or does it extend, `RuntimeException`? If yes, it is unchecked, and the compiler enforces nothing about handling it. If no, it is checked, and every method that can let it escape uncaught must declare `throws` for it, and every caller must in turn catch it or declare it too. This single classification question resolves the large majority of compile-versus-runtime exception questions on a competitive exam.