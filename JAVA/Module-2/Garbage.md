# Java Competitive Exam Preparation
## Chunk 7: JVM Memory Architecture, Object Creation, and Garbage Collection

Every chunk so far has treated `new BankAccount(...)` as a single, simple step. This chunk opens that step up: where does the object actually live, what exactly happens between the moment `new` is written and the moment a usable reference comes back, and what happens later when that object is no longer needed. Understanding this machinery retroactively explains several rules from earlier chunks that were previously just stated as facts, particularly Chunk 3's claim that objects are passed by reference value, and Chunk 2's claim that static fields live for the life of the class.

---

## Table of Contents

1. JVM Runtime Memory Architecture
2. What Happens When an Object Is Created
3. What Is Garbage Collection
4. What Is a Memory Leak
5. System.gc() and finalize()
6. Consolidated Quiz: All Seven Chunks Together
7. Programming Practice and Solutions
8. Revision Summary

---

## 1. JVM Runtime Memory Architecture

### Concept Explanation

When a Java program runs, the JVM divides the memory it manages into several distinct regions, each with a different purpose, a different lifetime, and different rules about who can access it. Understanding this division is what finally explains, precisely, why an object reference behaves the way it has behaved throughout this entire course.

| Region | Shared or per-thread | What it stores |
|---|---|---|
| Heap | Shared across the whole JVM | Every object and array ever created with `new`, including every instance field's actual value |
| Method Area (Metaspace) | Shared across the whole JVM | Per class information: the class's structure, its method bytecode, and its `static` fields |
| Java Stack | One per thread | Local variables and method call frames for that thread, including object references themselves |
| PC Register | One per thread | The address of the instruction that thread is currently executing |
| Native Method Stack | One per thread | Support for calls into non-Java, native code |

**The heap.** Every object created anywhere in a program, regardless of which method or thread created it, lives in the heap, a single shared memory region. This is also exactly where garbage collection, covered in Section 3, does its work.

**The method area, or metaspace.** Each class, not each object, has exactly one entry here, created the first time that class is loaded. This is where a class's `static` fields genuinely live, which is precisely why `BankAccount.totalAccountsCreated` from Chunk 2 is shared across every single `BankAccount` object: there is only one copy of it, sitting in the method area associated with the `BankAccount` class itself, never duplicated per object.

**The Java stack, and why object passing works the way Chunk 3 described.** Each thread gets its own private stack, made up of one frame per currently active method call. A frame holds that method's local variables, including its parameters. Crucially, when a local variable's type is a class, such as `BankAccount acc`, what is actually stored in that stack slot is only a reference, essentially an address pointing into the heap, not the object's actual data. The object's real fields, `accountHolder`, `balance`, `accountNumber`, live on the heap; the stack only ever holds a pointer to where that data is.

This is the exact mechanical reason behind Chunk 3's pass by value rule: when `acc` is passed into a method, the value copied into the new parameter slot is that same reference, that same heap address, not the object itself. Two different stack slots can hold the same heap address, which is exactly why modifying the object through either one is visible to both, while reassigning one slot to point elsewhere never affects the other.

**Where `String` literals live.** Chunk 5 introduced the string pool without specifying exactly where it physically resides. In modern Java, since Java 7, the string pool lives inside the ordinary heap, not in the method area, specifically so that pooled `String` objects are subject to the same garbage collection rules as any other heap object, rather than persisting for the entire life of the program the way method area content effectively does.

### Important Notes

- The heap is shared across the entire program; the stack is private, one per thread.
- `static` fields live in the method area, associated with the class, exactly once, which is why Chunk 2's `totalAccountsCreated` behaves as shared state.
- A local variable of a class type stores only a reference on the stack; the actual object always lives on the heap.
- A method's stack frame, including all of its local variables and parameters, is destroyed the moment that method returns, exactly matching the local variable scope rules from Chunk 1.
- The string constant pool lives inside the heap in modern Java, making pooled `String` objects eligible for garbage collection like any other heap object.

### Quick Check

When `SavingsAccount sa = new SavingsAccount("Om", 2000, 0.04);` executes inside `main`, where does the reference `sa` live, and where does the actual object it points to live?

`sa` itself, the reference, lives in `main`'s stack frame, since it is a local variable. The actual `SavingsAccount` object, holding `accountHolder`, `balance`, `accountNumber`, and `interestRate`, lives on the heap; `sa` merely stores the heap address where that object can be found.

---

## 2. What Happens When an Object Is Created

### Concept Explanation

`new SavingsAccount("Priya", 1000, 0.05)` looks like one simple step, but the JVM actually performs a specific, ordered sequence of work behind it.

**Step 1: Class loading, if not already done.** The very first time any class is actively used, whether by constructing an instance of it or accessing one of its static members, the JVM loads it: it locates the compiled `.class` bytecode, verifies it is well formed and safe, and prepares the class's `static` fields in the method area, giving each one its type's default value, such as `0` for an `int` or `null` for a reference. Only after that does the JVM run the class's static initializers, meaning any `static` field's own initializer expression and any static initializer block, in the exact order they appear in the source code, exactly once, ever, for that class, no matter how many objects of it are later constructed. This entire process happens only the first time a class is used; every later `new` on that same class skips straight past it.

**Step 2: Memory allocation on the heap.** Once the class is confirmed loaded, the JVM allocates a block of heap memory large enough to hold one instance of that class, including every inherited field, `SavingsAccount`'s object needs room for `accountHolder`, `balance`, and `accountNumber` inherited from `BankAccount`, plus `interestRate` declared directly on `SavingsAccount` itself.

**Step 3: Instance fields receive their default values.** Immediately upon allocation, every instance field, of every type, is set to its type's default value, exactly the default value rules from Chunk 1: `0` for numeric primitives, `false` for `boolean`, `null` for every reference type field, including `String` fields like `accountHolder`. This happens before any constructor code, or even any field initializer written in the source, has run at all.

**Step 4: The constructor chain executes.** As covered in Chunk 3, a subclass constructor's first action, explicit or implicit, is a call up to its superclass's constructor, and that superclass's constructor does the same thing, all the way up to `Object`'s own constructor at the very top. Conceptually, this chain resolves upward first, reaching `Object`, and then unwinds back downward: at each level, that class's own field initializers run, in the order they appear in the source, immediately followed by the rest of that level's constructor body. For `new SavingsAccount("Priya", 1000, 0.05)`, this means `Object`'s trivial construction happens first, then `BankAccount`'s constructor body runs, correctly setting `accountHolder`, `balance`, and generating `accountNumber`, and only after that fully completes does `SavingsAccount`'s own constructor body run, setting `interestRate`.

**Step 5: The reference is returned.** Once construction fully completes, `new` yields a reference to the freshly built object, which is then stored wherever the surrounding code puts it, here into the local variable `sa` on the stack, exactly as Section 1 described.

### Applied Example

```java
public class Demo {
    public static void main(String[] args) {
        System.out.println("Before construction");
        SavingsAccount sa = new SavingsAccount("Priya", 1000, 0.05);
        System.out.println("After construction: " + sa.getAccountHolder());
    }
}
```

Behind that single middle line, in order: `BankAccount` and `SavingsAccount` are loaded if this is the first time either has been used, since `SavingsAccount extends BankAccount`, both must be loaded, heap space for one `SavingsAccount` object is allocated, every field, `accountHolder`, `balance`, `accountNumber`, and `interestRate`, is set to its default value, `BankAccount`'s constructor then runs to completion setting the first three, `SavingsAccount`'s own constructor then runs to completion setting `interestRate`, and finally the resulting reference is stored into `sa`.

### Important Notes

- Class loading, including running static initializers, happens once per class, the first time it is actively used, never once per object.
- Every instance field gets its default value immediately upon allocation, strictly before any constructor code runs, exactly why a field left unset by every constructor still safely holds a predictable default rather than garbage data.
- Constructor execution conceptually runs from the top of the hierarchy downward, `Object` first, the most specific subclass last, exactly mirroring the `super(...)` chaining behavior from Chunk 3.
- The reference returned by `new` is the only thing ever assigned to a variable; the object itself never moves into the stack.

### Quick Check

If `BankAccount` has never been used anywhere yet in a running program, and the very first line of `main` is `SavingsAccount sa = new SavingsAccount(...)`, does `BankAccount` get loaded at all, even though `main` never directly writes `new BankAccount(...)`?

Yes. Since `SavingsAccount extends BankAccount`, constructing a `SavingsAccount` requires `BankAccount` to be loaded too, because `SavingsAccount`'s constructor must call up to `BankAccount`'s constructor as part of the chain described in Step 4; the JVM loads every class involved in that chain, not only the one directly named after `new`.

---

## 3. What Is Garbage Collection

### Concept Explanation

Garbage collection, often abbreviated GC, is the JVM's automatic process for reclaiming heap memory occupied by objects that a running program can no longer possibly use. Unlike languages that require manually freeing memory, Java never gives the programmer a `delete` or `free` operation at all; instead, the JVM periodically identifies which heap objects are still reachable and treats everything else as eligible for collection.

**Reachability.** An object is considered reachable if there exists some chain of references leading to it, starting from a "GC root," which includes any local variable currently on any thread's active stack, and any `static` field anywhere in the method area. An object with no such chain leading to it, no matter how it got that way, is unreachable, and therefore eligible for garbage collection.

**How an object becomes unreachable.**

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount temp = new BankAccount("Temporary", 0);
        temp.deposit(100);
        temp = null; // the object temp used to point to is now unreachable, if nothing else references it
    }
}
```

Setting `temp = null;` removes the only reference this code was holding to that particular `BankAccount` object. If nothing else in the program, no other variable, no static field, still refers to it, that object becomes eligible for garbage collection immediately after this line runs.

**A method returning is another common path to unreachability.**

```java
static void processOneTimeBonus() {
    BankAccount bonusAccount = new BankAccount("Bonus", 50);
    bonusAccount.deposit(25);
}
```

Once `processOneTimeBonus()` returns, its entire stack frame is destroyed, exactly as Section 1 described, which destroys the local variable `bonusAccount` along with it. If nothing else ever captured a reference to that object, for instance by returning it or storing it in a field, it becomes unreachable the instant the method returns.

**Eligibility is not immediate collection.** Becoming eligible only means the JVM is now permitted to reclaim an object's memory; it does not mean this happens at any particular guaranteed moment. The JVM decides, based on its own internal algorithms and current memory pressure, exactly when to actually run a collection cycle, which might be soon, might be much later, or, in a very short lived program, might never happen at all before the program simply exits.

### Important Notes

- An object is eligible for garbage collection once no reachable chain of references, starting from any GC root, leads to it any longer.
- Reassigning a reference variable, such as setting it to `null` or to a different object, can make the previously referenced object unreachable, if it was the only reference.
- A method returning destroys its entire stack frame, which can make any object only referenced by that frame's now destroyed local variables unreachable.
- Becoming eligible for collection and actually being collected are two different moments; the JVM controls the timing of the second entirely on its own.
- Java has no manual memory deallocation at all; garbage collection is the sole mechanism for reclaiming heap memory.

### Quick Check

In the `processOneTimeBonus()` example, if the method were changed to `return bonusAccount;` and the caller stored that returned reference in a variable of its own, would the object still become unreachable once `processOneTimeBonus()` returns?

No. Returning the reference, exactly the object returning mechanism from Chunk 3, hands that same reference out to the caller before the method's frame is destroyed. As long as the caller stores it somewhere reachable, such as its own local variable, the object remains reachable through that new location, entirely unaffected by the original local variable inside `processOneTimeBonus()` ceasing to exist.

---

## 4. What Is a Memory Leak

### Concept Explanation

Despite automatic garbage collection, a Java program can still leak memory. A memory leak here means an object that the program's actual logic no longer needs remains reachable anyway, through some reference the program simply forgot to clear, which prevents the garbage collector from ever reclaiming it, even though it genuinely should be able to.

**This is different from memory leaks in languages without garbage collection.** In C or C++, a leak typically means memory was allocated and the pointer to it was lost entirely, with no way to free it at all. In Java, a leak instead means a reference to the object is still very much present and reachable, just no longer meaningfully needed, so the object is never even considered eligible for collection in the first place.

**A classic pattern: an ever growing static collection.**

```java
import java.util.ArrayList;
import java.util.List;

public class AccountRegistry {
    private static List<BankAccount> allAccountsEver = new ArrayList<>();

    public static void register(BankAccount acc) {
        allAccountsEver.add(acc);
    }
}
```

`ArrayList` here is a simple resizable list from `java.util`; `add(acc)` appends `acc` onto the end of it. Since `allAccountsEver` is `static`, it lives in the method area for the entire life of the program, exactly as Section 1 established, meaning any object added to it remains reachable, through that static field, for as long as the program runs, even long after every other part of the program has finished using that particular account and every local variable that once pointed to it has gone out of scope.

```java
public class Demo {
    public static void main(String[] args) {
        for (int i = 0; i < 1_000_000; i++) {
            BankAccount temp = new BankAccount("Temp" + i, 0);
            AccountRegistry.register(temp);
        }
        // every single one of these million accounts remains reachable through
        // AccountRegistry.allAccountsEver, so none of them are ever eligible
        // for garbage collection, even though this loop no longer needs any
        // of them once it moves on to the next iteration
    }
}
```

Even though `temp` itself goes out of scope and is reused on every iteration, exactly as Chunk 1 covered for loop variable scope, each individual object it briefly pointed to remains reachable afterward, since `AccountRegistry.register` stored a second, separate reference to it inside the static list. The heap steadily fills with a million objects that nothing in the program's actual logic still needs, yet none of them can ever be reclaimed.

**Other common causes, briefly.** Beyond static collections, unintentional retention commonly happens through objects registered as listeners or callbacks that are never later removed, and through caches that grow without any eviction policy, both of which follow the exact same underlying pattern: a reference persists somewhere longer than the program's actual logic requires it to.

### Important Notes

- A Java memory leak means an object remains reachable, and therefore ineligible for collection, even though the program no longer actually needs it.
- Static fields are a particularly common source, since they persist for the entire life of the program, exactly as covered in Chunk 2 and reinforced by Section 1's method area discussion.
- The fix for this class of leak is almost always the same: explicitly remove the no longer needed reference, such as calling a corresponding `remove` method, once an object genuinely is done being needed.
- A memory leak in Java is a logic problem, not a missing language feature; the garbage collector is working exactly as designed the entire time, correctly refusing to collect something that is still, technically, reachable.

### Quick Check

If `AccountRegistry` provided a matching `unregister(BankAccount acc)` method that removed `acc` from `allAccountsEver`, and every part of the program correctly called it once an account was genuinely finished with, would this resolve the leak?

Yes. Removing the reference from the static list eliminates that particular path to reachability. If no other reference to that account exists elsewhere at that point, the object becomes eligible for garbage collection exactly as Section 3 described, resolving the leak, since the static field itself is no longer the thing keeping the object artificially alive.

---

## 5. System.gc() and finalize()

### Concept Explanation

Java provides two closely related, and both commonly misunderstood, tools connected to garbage collection: a way to request that a collection cycle run, and a way for an object to run cleanup code just before it is collected. Both are worth understanding specifically so you know their real, limited guarantees rather than an idealized version of what they might seem to promise.

**`System.gc()`.** Calling `System.gc()` does not force garbage collection to happen. It is only a hint, a suggestion to the JVM that now might be a reasonable time to run a collection cycle. The JVM is completely free to ignore this hint entirely, and in practice often does, particularly if it has already determined a collection is not currently worthwhile. `Runtime.getRuntime().gc()` is an equivalent, lower level way to make the same request.

```java
BankAccount temp = new BankAccount("Temp", 0);
temp = null;
System.gc(); // only a request; the JVM may or may not actually collect anything here
```

Relying on `System.gc()` to guarantee an object is collected by any particular point in a program is a mistake; no such guarantee exists anywhere in the language specification.

**`finalize()`.** `Object`, exactly the same class discussed in Chunk 3 and again in Chunk 5's `equals`/`hashCode` coverage, also historically provided a method called `finalize()`, which the garbage collector may call on an object at most once, at some point before actually reclaiming its memory, intended to give that object one last chance to release any resources it might be holding.

```java
public class BankAccount {
    // ... existing members ...

    @Override
    protected void finalize() throws Throwable {
        System.out.println("Finalizing account for " + accountHolder);
        super.finalize();
    }
}
```

**Why `finalize()` is unreliable and discouraged.** Several serious problems make `finalize()` a poor tool for anything a program genuinely depends on. It is never guaranteed to run at all, since a program might exit before the garbage collector ever gets around to calling it on a given object. Its timing, even when it does run, is entirely unpredictable, controlled by the same JVM discretion covered in Section 3. Overriding it can also measurably slow down garbage collection for that class of object. For all of these reasons, `finalize()` was formally deprecated starting in Java 9, and modern Java code relies instead on explicit, deterministic cleanup mechanisms, such as calling a `close()` method directly or using try-with-resources, a construct belonging to exception handling and outside this chunk's scope, but worth knowing exists as the actual recommended replacement.

### Important Notes

- `System.gc()` only requests garbage collection; it never guarantees that a collection cycle actually runs, or that any specific object gets collected as a result.
- `finalize()`, inherited from `Object`, may be called once before an object's memory is reclaimed, but is never guaranteed to run at all, and its timing is never predictable even when it does.
- `finalize()` is deprecated since Java 9; real resource cleanup, such as closing a file or network connection, should never depend on it.
- Neither tool changes the fundamental rule from Section 3: the JVM alone decides when, or whether, to actually collect any given eligible object.

### Quick Check

If a program calls `System.gc()` immediately after setting a reference to `null`, is it guaranteed that the object's `finalize()` method, if overridden, runs before the very next line of code executes?

No, on two separate counts. First, `System.gc()` is only a request, so the JVM might not run a collection cycle at that point at all. Second, even during an actual collection cycle, `finalize()`'s timing relative to the rest of the program is never guaranteed by the language specification; there is no reliable way to make finalization happen synchronously with any specific point in a program's execution.

---

## 6. Consolidated Quiz: All Seven Chunks Together

1. `BankAccount.totalAccountsCreated`, from Chunk 2, is `static`. Based on Section 1's memory architecture, where does this field actually live?
   A. On the heap, once per object  B. In the method area, associated with the `BankAccount` class itself, exactly once  C. On the stack  D. In the string pool

2. When a `BankAccount` reference is passed as a method argument, exactly as covered in Chunk 3, what is actually being copied, based on this chunk's memory model?
   A. The entire object's field data  B. The reference itself, a heap address stored in a stack slot, is copied, not the object's data  C. Nothing is copied  D. A new object is always created

3. `Transaction`, from Chunk 4, is `abstract` and declares `execute` as an abstract method. During the class loading step from Section 2, are `Transaction`'s static initializers run once, or once per concrete subclass instantiated?
   A. Once per subclass instantiated  B. Once total, the first time `Transaction`, or any class requiring it to be loaded, is actively used, regardless of how many concrete subclasses are later instantiated  C. Never, since it is abstract  D. Once per method call

4. `String` literals, from Chunk 5, are pooled. Based on Section 1, where does that pool actually live in modern Java?
   A. The method area  B. The heap, since Java 7  C. The stack  D. It does not exist in modern Java

5. In `SavingsAccount`'s constructor chain from Chunk 3, which class's constructor logic conceptually runs first when a new `SavingsAccount` is constructed?
   A. `SavingsAccount`'s own  B. `BankAccount`'s  C. `Object`'s, at the very top of the chain, before either subclass's own logic runs  D. None run automatically

6. A `Branch` object, from Chunk 4, holds a `BankAccount[]` as a field, a HAS-A relationship. If every reference to a particular `Branch` object is set to `null`, does its `BankAccount[]` array, and every account inside it, immediately become eligible for garbage collection too?
   A. No, arrays are never collected  B. Yes, if nothing else independently references that array or those specific accounts, they too lose their only remaining path to reachability and become eligible alongside the `Branch` object itself  C. Only the `Branch` object itself becomes eligible, never its contents  D. Compile error

7. If `AccountRegistry.allAccountsEver`, from Section 4, used a plain instance field rather than a `static` one, would the same unbounded leak pattern still occur across the entire program's lifetime in the same way?
   A. Yes, identically  B. Not in the same way; an instance field's lifetime is tied to its own containing object's reachability, not automatically to the entire program's lifetime the way a static field's is  C. Instance fields cannot hold collections  D. Compile error

8. Does overriding `equals()` and `hashCode()`, from Chunk 5, have any effect on whether an object is reachable for garbage collection purposes?
   A. Yes, always  B. No; reachability, covered in Section 3, is determined purely by reference chains from GC roots, entirely unrelated to how `equals()` or `hashCode()` are implemented  C. Only if `hashCode()` returns `0`  D. Compile error

### Consolidated Quiz Answers

1. **B.** As established in Section 1, `static` fields live in the method area, associated with the class itself rather than any individual object, which is exactly why `totalAccountsCreated` is shared across every `BankAccount` instance rather than duplicated per object.

2. **B.** This is the precise mechanical explanation, from Section 1, for Chunk 3's pass by value rule: the stack slot holding an object typed parameter stores only a heap address, and it is that address, the reference, which gets copied, never the object's actual field data.

3. **B.** Class loading and static initialization, covered in Section 2, happen exactly once per class, the first time it is genuinely needed, entirely independent of how many concrete subclass objects are later constructed; `Transaction` itself never needs to be instantiated for this to occur, only referenced in a way that requires it to be loaded.

4. **B.** As specifically noted in Section 1, the string constant pool has lived inside the regular heap since Java 7, rather than in the method area, making pooled strings subject to ordinary garbage collection rules.

5. **C.** As covered in Section 2 and originally in Chunk 3, the constructor chain always resolves all the way up to `Object` first before any actual field initialization or constructor body logic executes, working back down from there through `BankAccount` and finally `SavingsAccount`.

6. **B.** Setting every reference to the `Branch` object itself to `null` removes the only path of reachability the array had, assuming nothing else in the program independently references that same array or its individual account elements; reachability, as covered in Section 3, propagates through the entire chain of references, not just the outermost object.

7. **B.** A `static` field, as covered in Section 1, persists for the entire life of the class, independent of any individual object; a plain instance field only persists for as long as its own containing object remains reachable, so if that containing object itself eventually becomes unreachable and is collected, its instance field, and whatever it referenced, becomes eligible for collection right along with it, rather than persisting indefinitely on its own.

8. **B.** Reachability, as defined in Section 3, is entirely about whether a chain of references from a GC root leads to an object; it has no relationship whatsoever to that object's `equals()` or `hashCode()` implementations, which only affect how the object compares to others or behaves in hash based contexts.

---

## 7. Programming Practice and Solutions

### Practice Problems

1. **Basic.** Write a short program that creates a `BankAccount`, prints a message, sets the only reference to `null`, and calls `System.gc()`, adding a comment explaining why the printed order of any `finalize()` message, if one were added, cannot be relied upon to appear before the program's own next line.
2. **Intermediate.** Rewrite `AccountRegistry` from Section 4 to include a working `unregister(BankAccount acc)` method using `ArrayList`'s `remove(Object o)` method, and demonstrate registering three accounts, unregistering one, and printing the registry's remaining size using `size()`.
3. **Intermediate.** Write a method `describeReachability()` that creates a `BankAccount`, stores it in a local variable, then explains in comments, at each of three points, whether the object is reachable and why: immediately after creation, after the local variable is reassigned to `null`, and after the method itself returns.
4. **Advanced.** Design a small `Session` class representing a logged in bank session that registers itself into a `static` `List<Session> activeSessions` upon construction, and provide a `logout()` instance method that removes itself from that list, demonstrating the fix for the exact leak pattern described in Section 4, then write a `main` method creating several sessions, logging out only some of them, and printing how many remain active.

### Solutions

**Solution 1.**

```java
public class Solution1 {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount("Temp", 0);
        System.out.println("Account created for " + acc.getAccountHolder());
        acc = null;
        System.gc();
        // System.gc() only requests a collection cycle; the JVM is free to
        // ignore it entirely or run it at some unpredictable later time, so
        // any finalize() message the collected object might have printed
        // could appear before, after, or never relative to this next line.
        System.out.println("Program continuing");
    }
}
```

Why it works: this directly demonstrates the request-not-command nature of `System.gc()` from Section 5, and the comment explicitly states the guarantee, or rather the lack of one, that governs whether any cleanup code would run at a predictable point.

**Solution 2.**

```java
import java.util.ArrayList;
import java.util.List;

public class AccountRegistry {
    private static List<BankAccount> allAccountsEver = new ArrayList<>();

    public static void register(BankAccount acc) {
        allAccountsEver.add(acc);
    }

    public static void unregister(BankAccount acc) {
        allAccountsEver.remove(acc);
    }

    public static int size() {
        return allAccountsEver.size();
    }
}
```

```java
public class Solution2 {
    public static void main(String[] args) {
        BankAccount a = new BankAccount("A", 0);
        BankAccount b = new BankAccount("B", 0);
        BankAccount c = new BankAccount("C", 0);

        AccountRegistry.register(a);
        AccountRegistry.register(b);
        AccountRegistry.register(c);

        AccountRegistry.unregister(b);

        System.out.println("Remaining registered: " + AccountRegistry.size());
    }
}
```

Expected output: `Remaining registered: 2`

Why it works: `remove(Object o)` uses `BankAccount`'s `equals()` behavior to locate and remove the matching entry from the list; since `b` was explicitly unregistered, only `a` and `c` remain, correctly resolving the unbounded growth risk from Section 4 as long as every part of the program that registers an account also remembers to unregister it once finished.

**Solution 3.**

```java
public class Solution3 {
    static void describeReachability() {
        BankAccount acc = new BankAccount("Test", 0);
        // Point 1: reachable, since the local variable acc, on this method's
        // own stack frame, is a GC root, and it directly references the object.

        acc = null;
        // Point 2: the object acc used to reference is now unreachable, assuming
        // nothing else in the program independently references it, since its
        // only reference has just been overwritten with null.

        // Point 3, after this method returns: the entire stack frame, including
        // wherever acc's slot lived, is destroyed; even if acc had not been set
        // to null, returning would have removed this path to reachability too.
    }

    public static void main(String[] args) {
        describeReachability();
    }
}
```

Why it works: this walks through the exact reachability lifecycle from Section 3 at three distinct, clearly marked points, reinforcing that unreachability can arise either from explicit reassignment or from the natural destruction of a stack frame once its method returns.

**Solution 4.**

```java
import java.util.ArrayList;
import java.util.List;

public class Session {
    private static List<Session> activeSessions = new ArrayList<>();
    private String username;

    public Session(String username) {
        this.username = username;
        activeSessions.add(this);
    }

    public void logout() {
        activeSessions.remove(this);
    }

    public static int activeCount() {
        return activeSessions.size();
    }
}
```

```java
public class Solution4 {
    public static void main(String[] args) {
        Session s1 = new Session("priya");
        Session s2 = new Session("arjun");
        Session s3 = new Session("meera");

        System.out.println("Active after login: " + Session.activeCount());

        s2.logout();

        System.out.println("Active after one logout: " + Session.activeCount());
    }
}
```

Expected output:
```
Active after login: 3
Active after one logout: 2
```

Why it works: `activeSessions.add(this);` inside the constructor registers each new `Session` into the shared static list, exactly the `this` reference pattern from Chunk 3, and `logout()` correctly removes that same reference, demonstrating the fix for the Section 4 leak pattern: as long as every session that starts eventually calls `logout()`, none of them linger in the static list, and therefore none of them are artificially kept reachable, once the program's actual logic is genuinely finished with them.

---

## 8. Revision Summary

| Concept | Key Rule | Where It Appeared |
|---|---|---|
| Heap | Shared, holds every object and its instance field data | `BankAccount` objects |
| Method area | Shared, holds class structure, method bytecode, and `static` fields | `totalAccountsCreated` |
| Stack | Per thread, holds local variables and references, destroyed when a method returns | `acc` in `main` |
| Class loading | Happens once per class, the first time it is genuinely needed; runs static initializers exactly once | `Transaction` loaded via `super(...)` chain |
| Object creation order | Load class, allocate heap memory, default values, constructor chain top-down, return reference | `new SavingsAccount(...)` |
| Reachability | An object stays alive only while some GC root can reach it through a reference chain | `temp = null;` |
| Garbage collection | Automatic, JVM-timed reclamation of unreachable heap objects; never manually forced | No `delete`/`free` in Java |
| Memory leak | An object remains reachable, and therefore uncollectable, even though the program no longer needs it | `AccountRegistry` without `unregister` |
| System.gc() | A request only; never a guarantee that collection actually runs | `System.gc();` |
| finalize() | May run once before collection, never guaranteed to run at all or at any predictable time; deprecated since Java 9 | `BankAccount.finalize()` |

**Exam tip.** Whenever a question describes an object and then asks whether it is eligible for garbage collection, trace every single reference to it explicitly: has any variable holding it gone out of scope, been reassigned, or been set to `null`? Is it, or was it ever, stored in a `static` field or any other structure with a longer lifetime than the code that created it? Reachability is the entire question; nothing about an object's own class, fields, or overridden methods changes the answer.