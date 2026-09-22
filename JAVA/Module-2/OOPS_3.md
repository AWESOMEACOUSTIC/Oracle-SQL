# Java Competitive Exam Preparation
## Chunk 4: Abstract Classes, Interfaces, and IS-A / HAS-A Relationships

This chunk continues the banking scenario from Chunks 2 and 3. It adds two new tools for expressing design intent: abstract classes, for when a superclass should force its subclasses to each provide their own version of some behavior, and interfaces, for when unrelated classes need to share a capability without sharing an inheritance tree. Every example below deliberately reuses fields, methods, and classes built in the earlier chunks, and the closing quiz and activity pull concepts from all four chunks together at once.

---

## Table of Contents

1. Introduction to Abstract Classes
2. Lend A Hand on Abstract Class
3. Introduction to Interfaces
4. Why Do We Use Interfaces
5. Lend A Hand on Interfaces
6. Interfaces vs Abstract Classes
7. Interface as a Type with Lend A Hand
8. Interface vs Class
9. Inheritance among Interfaces
10. Interface and Polymorphism
11. IS-A and HAS-A Relationships
12. Consolidated Quiz: All Four Chunks Together
13. Programming Practice and Solutions
14. Revision Summary

---

## 1. Introduction to Abstract Classes

### Growing the Scenario

So far, every `BankAccount` operation has happened through direct method calls like `acc.deposit(100)`. A real bank also needs to record what happened as its own thing: a transaction, something with an amount and a description, that can later be applied to an account. But a bare, generic "transaction" makes no real sense on its own; every actual transaction is specifically a deposit, a withdrawal, or some other concrete kind, each applying itself to an account differently. This is exactly the situation abstract classes are built for.

### Concept Explanation

An abstract class is a class declared with the `abstract` keyword that cannot be instantiated directly with `new`, even though it can otherwise look and behave much like an ordinary class: it can have fields, constructors, and fully implemented, ordinary methods. What makes it different is that it can also declare **abstract methods**, methods with a signature but no body at all, ending in a semicolon instead of `{ }`. Any concrete, that is, non abstract, subclass of an abstract class is required to provide an actual implementation for every abstract method it inherits, or the subclass itself must also be declared `abstract`.

**Abstract in the real world.** Think of "vehicle" as a concept. You have certainly seen cars, motorcycles, and trucks, but you have never seen a bare, generic "vehicle" that is not specifically one of those. "Vehicle" is a useful category for grouping shared traits, like having a speed and being able to accelerate, but it only ever exists in the world through one of its concrete forms. An abstract class captures exactly this relationship in code: `Transaction` is a meaningful category with shared traits, but every transaction that actually exists in a running program is specifically a `DepositTransaction`, a `WithdrawalTransaction`, or some other concrete kind.

**Building `Transaction`.**

```java
public abstract class Transaction {
    private final double amount;
    private final String description;

    public Transaction(double amount, String description) {
        this.amount = amount;
        this.description = description;
    }

    public double getAmount() {
        return amount;
    }

    public abstract boolean execute(BankAccount account);

    @Override
    public String toString() {
        return description + " of " + amount;
    }
}
```

`Transaction` has a constructor, two ordinary fields, and a fully implemented `toString()`, exactly like any other class from Chunks 2 and 3. `execute(BankAccount account)`, however, is declared `abstract`, with no body, because how a transaction actually affects an account genuinely differs by transaction type, and `Transaction` itself has no sensible single answer to give.

**Concrete subclasses providing the missing implementation.**

```java
public class DepositTransaction extends Transaction {
    public DepositTransaction(double amount) {
        super(amount, "Deposit");
    }

    @Override
    public boolean execute(BankAccount account) {
        account.deposit(getAmount());
        return true;
    }
}

public class WithdrawalTransaction extends Transaction {
    public WithdrawalTransaction(double amount) {
        super(amount, "Withdrawal");
    }

    @Override
    public boolean execute(BankAccount account) {
        return account.withdraw(getAmount());
    }
}
```

Both subclasses call `super(amount, description)`, exactly the constructor chaining pattern from Chunk 3's `super(...)` section, to let `Transaction`'s own constructor handle the shared setup. Each then provides its own `execute`, satisfying the abstract method's requirement. `DepositTransaction` always succeeds and returns `true`; `WithdrawalTransaction` defers to `BankAccount`'s own `withdraw`, which already returns a meaningful `boolean`, from Chunk 2's return value design.

### When to Use an Abstract Class

Reach for an abstract class specifically when a group of related classes share both common state or behavior, which the abstract class can implement once, and at least one piece of behavior that every subclass must provide its own version of, which the abstract class leaves as an abstract method. If every subclass would genuinely behave identically for some method, put that method's real implementation directly in the abstract class instead, exactly as `toString()` was written once in `Transaction` rather than repeated in every subclass. If there is no shared state or behavior at all, and classes only need to share a capability's signature, an interface, covered starting in Section 3, is usually the better fit.

### Important Notes

- `abstract class Transaction` cannot be instantiated: `new Transaction(10, "test")` is a compile time error.
- An abstract method has no body and ends with a semicolon; it can only exist inside an abstract class.
- A concrete subclass must override every abstract method it inherits, using the exact same overriding rules from Chunk 3, or the subclass itself must also be declared `abstract`.
- An abstract class can still have constructors, even though it can never be instantiated directly; those constructors exist specifically to be called via `super(...)` from a concrete subclass's own constructor.
- An abstract class can freely mix abstract methods with fully implemented, ordinary methods and fields, exactly like `Transaction` does.

### Quick Check

Why does `Transaction` bother having a constructor at all, if it can never be instantiated directly?

Because a subclass's constructor still needs a way to initialize the fields declared in `Transaction`, `amount` and `description`, both of which are `private` and therefore not directly settable from `DepositTransaction`'s own code. Calling `super(amount, description)` delegates that initialization to `Transaction`'s constructor, exactly the same reason `SavingsAccount` called `super(accountHolder, initialBalance)` in Chunk 3, even though `Transaction` itself is never the object actually being constructed at runtime.

---

## 2. Lend A Hand on Abstract Class

### Applied Example: Polymorphism over an abstract type

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount("Wren", 500);

        Transaction[] transactions = {
            new DepositTransaction(200),
            new WithdrawalTransaction(1000),
            new WithdrawalTransaction(100)
        };

        for (Transaction t : transactions) {
            boolean success = t.execute(acc);
            System.out.println(t + " -> success: " + success);
        }

        System.out.println("Final balance: " + acc.getBalance());
    }
}
```

Output:
```
Deposit of 200.0 -> success: true
Withdrawal of 1000.0 -> success: false
Withdrawal of 100.0 -> success: true
Final balance: 600.0
```

`Transaction[]` holds a mix of concrete subclass objects, exactly like `BankAccount[]` held `SavingsAccount` and `CheckingAccount` objects in Chunk 3. Calling `t.execute(acc)` dispatches, at runtime, to whichever concrete subclass's own `execute` the actual object is, following the exact same dynamic method dispatch rules from Chunk 3's polymorphism section, even though `Transaction` itself never provides any implementation for `execute` to fall back on.

### Quiz

1. Why can `transactions` be declared as `Transaction[]` even though every actual element is some concrete subclass?
   A. `Transaction[]` secretly creates `Transaction` objects  B. Since each subclass genuinely is a `Transaction` through inheritance, a `Transaction[]` array can legally hold references to any of them, mirroring `BankAccount[]` in Chunk 3  C. Arrays cannot hold abstract types at all  D. Compile error

2. Could `new Transaction(50, "Generic")` ever compile?
   A. Yes, always  B. No, abstract classes cannot be instantiated directly  C. Only inside `main`  D. Only if `Transaction` has no constructor

3. If a new `TransferTransaction` class extended `Transaction` but forgot to override `execute`, what would happen?
   A. It inherits a default no-op implementation  B. Compile error, unless `TransferTransaction` is itself also declared `abstract`  C. Runtime exception when `execute` is called  D. It silently compiles fine

### Answers

1. **B.** Exactly as with `BankAccount[]` holding `SavingsAccount` and `CheckingAccount` objects, any array typed as a superclass, abstract or not, can hold references to any of its concrete subclasses, since each one genuinely is that type through inheritance.

2. **B.** `Transaction` is declared `abstract`, and Java flatly forbids instantiating an abstract class with `new`, regardless of whether it has a constructor; the constructor exists only to be called from a subclass via `super(...)`.

3. **B.** Since `execute` is abstract in `Transaction` and `TransferTransaction` provides no implementation, `TransferTransaction` has not fully satisfied `Transaction`'s contract; the compiler requires either an implementation or that `TransferTransaction` itself also be marked `abstract`, deferring the obligation further down the hierarchy.

### Programming Practice

1. Add a `TransferTransaction` class extending `Transaction`, taking a source `BankAccount`, a destination `BankAccount`, and an amount in its constructor, whose `execute(BankAccount account)` ignores its own `account` parameter and instead withdraws from the source and deposits into the destination, returning whether the withdrawal succeeded.

### Solution

```java
public class TransferTransaction extends Transaction {
    private BankAccount source;
    private BankAccount destination;

    public TransferTransaction(BankAccount source, BankAccount destination, double amount) {
        super(amount, "Transfer");
        this.source = source;
        this.destination = destination;
    }

    @Override
    public boolean execute(BankAccount account) {
        boolean withdrawn = source.withdraw(getAmount());
        if (!withdrawn) {
            return false;
        }
        destination.deposit(getAmount());
        return true;
    }
}
```

Why it works: `super(amount, "Transfer")` reuses `Transaction`'s constructor exactly as the other subclasses do, and the two extra fields, `source` and `destination`, are genuinely new state specific to this subclass, storing object references exactly as covered in Chunk 3's discussion of passing objects. `execute` still matches the abstract method's required signature even though its own `account` parameter goes unused here, since the real accounts involved were already captured in the constructor.

---

## 3. Introduction to Interfaces

### Growing the Scenario

The bank now wants a feature completely unrelated to the account type hierarchy: every account, and even a transaction, should be able to produce an audit log entry, a short text summary for compliance review. `BankAccount` and `Transaction` are not related to each other by inheritance at all, and forcing them into a shared superclass just for one method would be poor design. This is exactly what interfaces solve: sharing a capability across otherwise unrelated classes.

### Concept Explanation

An interface is declared with the `interface` keyword and defines a contract: a set of method signatures that any implementing class promises to provide real implementations for. Unlike an abstract class, a traditional interface has no fields holding instance state, no constructors, and, prior to Java 8, every method was implicitly abstract with no body at all.

**Developing an interface.**

```java
public interface Auditable {
    String getAuditLog();
}
```

This declares a single method, `getAuditLog()`, with no body. Any class that wants to be `Auditable` must implement it.

**Implementing an interface.** A class uses the `implements` keyword, rather than `extends`, to state that it fulfills an interface's contract.

```java
public class BankAccount implements Auditable {
    // ... existing fields and methods from Chunks 2 and 3 ...

    @Override
    public String getAuditLog() {
        return "Account #" + accountNumber + " (" + accountHolder + "): balance " + balance;
    }
}
```

`BankAccount` now both has all its existing behavior from earlier chunks and fulfills the `Auditable` contract. `@Override` is used here exactly as it was for overriding, since implementing an interface method is, mechanically, extremely similar to overriding: both require matching an inherited or promised signature exactly.

**A class can implement multiple interfaces, unlike extending multiple classes.** Java allows a class to `implements` any number of interfaces, separated by commas, even though it can `extends` at most one class. This is one of interfaces' most important practical advantages over abstract classes, developed further in Section 6.

```java
public class Transaction implements Auditable {
    // abstract class body...
}
```

Since `Transaction`'s own `toString()` already produces a reasonable summary, `Transaction` can implement `getAuditLog()` by simply delegating to it:

```java
@Override
public String getAuditLog() {
    return "Transaction: " + toString();
}
```

### Important Notes

- Interfaces use `implements`, not `extends`, when a class fulfills them; a class extends at most one class but can implement any number of interfaces.
- A traditional interface method with no body is implicitly `public` and `abstract`, even without those keywords written explicitly.
- Implementing an interface's method follows the same signature matching discipline as overriding, and `@Override` is equally recommended here to catch signature mistakes.
- Interfaces cannot be instantiated directly, exactly like abstract classes: `new Auditable()` is a compile error.

### Quick Check

Can `BankAccount` implement `Auditable` while also being a superclass for `SavingsAccount` and `CheckingAccount`, exactly as it already is from Chunk 3?

Yes. Implementing an interface has no effect at all on a class's own ability to be extended or to extend something else; `BankAccount` still `extends` nothing explicitly, so it implicitly extends `Object` as always, and it now additionally `implements Auditable`, and `SavingsAccount extends BankAccount` continues to work completely unchanged, automatically inheriting the `Auditable` implementation too.

---

## 4. Why Do We Use Interfaces

### Concept Explanation

Interfaces solve a problem inheritance alone cannot: expressing that unrelated classes share a capability, without forcing them into an artificial common superclass. `BankAccount` and `Transaction` have essentially nothing in common structurally, one holds a balance and a holder's name, the other holds an amount and a description, yet both can meaningfully produce an audit log. Making both extend some shared `Auditable` superclass would be forced and would waste Java's single inheritance allowance on a relationship that is really just "can do this one thing," not "is fundamentally this kind of thing."

**Interfaces enable writing code against a capability, not a concrete type.** A method that accepts an `Auditable` parameter can operate on a `BankAccount`, a `Transaction`, or any future class that ever chooses to implement `Auditable`, without that method needing to know or care which concrete class it actually received.

```java
public static void printAuditEntry(Auditable item) {
    System.out.println(item.getAuditLog());
}
```

This single method now works for every current and future `Auditable` type, exactly analogous to how a method accepting `BankAccount` in Chunk 3 worked for every subclass, except here the classes involved need share no inheritance relationship whatsoever.

**Interfaces also support Java's version of multiple inheritance of type.** Since a class can implement several interfaces at once, a single class can promise to fulfill several unrelated capabilities simultaneously, something ordinary single class inheritance could never allow directly.

### Quick Check

If a new, completely unrelated class `LoanApplication`, sharing no inheritance relationship with `BankAccount` or `Transaction` at all, also implemented `Auditable`, would `printAuditEntry` work on it without any changes?

Yes. `printAuditEntry` only requires its parameter to be `Auditable`, and since `LoanApplication` would fulfill that contract regardless of its own unrelated class hierarchy, the exact same method continues to work, unmodified, for this entirely new kind of object too.

---

## 5. Lend A Hand on Interfaces

### Applied Example: One class, multiple interfaces

```java
public interface Reportable {
    String generateReport();
}
```

```java
public class BankAccount implements Auditable, Reportable {
    // ... existing members ...

    @Override
    public String getAuditLog() {
        return "Account #" + accountNumber + " (" + accountHolder + "): balance " + balance;
    }

    @Override
    public String generateReport() {
        return "Report for " + accountHolder + ": current balance is " + balance;
    }
}
```

`BankAccount` now implements two separate interfaces at once, `Auditable` and `Reportable`, each representing a distinct, independent capability, listed together after `implements`, separated by a comma.

### Quiz

1. Could `SavingsAccount`, which extends `BankAccount`, also be passed to `printAuditEntry(Auditable item)`?
   A. No, only direct implementers count  B. Yes, since it inherits `BankAccount`'s implementation of `Auditable` through inheritance, it genuinely is `Auditable` too  C. Only if it implements `Auditable` again itself  D. Compile error

2. Why does `implements Auditable, Reportable` use a comma rather than requiring two separate `implements` clauses?
   A. Java requires exactly this comma syntax for implementing multiple interfaces in one clause  B. It is only stylistic, both are equally valid  C. Only one interface can actually be implemented despite the syntax  D. This is a compile error

3. What would happen if `BankAccount` implemented `Auditable` but never actually provided a `getAuditLog()` method anywhere in its own body or inherited from elsewhere?
   A. Compiles fine, uses a default empty implementation  B. Compile error, since `BankAccount` is a concrete class and must fulfill every method the interface requires  C. Runtime exception only when called  D. `Auditable` becomes optional

### Answers

1. **B.** `SavingsAccount` inherits `BankAccount`'s full implementation of `Auditable`, including its `getAuditLog()` method, exactly as it inherits any other public method; since it therefore genuinely fulfills the `Auditable` contract, it can be passed anywhere an `Auditable` is expected, without needing to mention `Auditable` itself at all.

2. **A.** Java's actual required syntax for implementing multiple interfaces is a single `implements` clause with the interface names separated by commas; there is no alternate form using repeated `implements` keywords.

3. **B.** Unlike an abstract class, which permits leaving abstract methods unimplemented as long as the class itself is also declared `abstract`, a concrete class, one not declared `abstract`, must provide a genuine implementation for every method any interface it implements requires, or the code fails to compile.

### Programming Practice

1. Make `Transaction` also implement `Reportable`, with `generateReport()` returning a short summary using the existing `toString()`, and demonstrate calling `generateReport()` on both a `BankAccount` and a `Transaction` through a shared `Reportable[]` array.

### Solution

```java
public abstract class Transaction implements Auditable, Reportable {
    // ... existing members ...

    @Override
    public String generateReport() {
        return "Transaction report: " + toString();
    }
}
```

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount("Farid", 300);
        Transaction t = new DepositTransaction(50);

        Reportable[] items = { acc, t };
        for (Reportable r : items) {
            System.out.println(r.generateReport());
        }
    }
}
```

Why it works: both `BankAccount` and `Transaction` implement `Reportable`, despite sharing no inheritance relationship with each other at all, so a single `Reportable[]` array can hold both kinds of object together, and calling `generateReport()` on each dispatches to that specific class's own implementation, exactly the polymorphism principle from Chunk 3, now working across entirely unrelated class hierarchies rather than within one.

---

## 6. Interfaces vs Abstract Classes

### Comparison

| | Abstract Class | Interface |
|---|---|---|
| Keyword | `abstract class` | `interface` |
| A class relates to it via | `extends` | `implements` |
| How many a class can have | One abstract superclass at most | Any number of interfaces |
| Instance fields with real state | Allowed | Not allowed, other than `public static final` constants |
| Constructors | Allowed, called via `super(...)` from a subclass | Not allowed at all |
| Method bodies | Freely mixes abstract and fully implemented methods | Traditionally all abstract; Java 8+ allows `default` and `static` methods with bodies |
| Best fit for | A group of closely related classes sharing both state and some forced, subclass specific behavior | A capability that unrelated classes can all promise to support |

**Applying this to the scenario.** `Transaction` is an abstract class because `DepositTransaction`, `WithdrawalTransaction`, and `TransferTransaction` are closely related, they share real fields and a real constructor, and only differ in one specific piece of behavior. `Auditable` and `Reportable` are interfaces because `BankAccount` and `Transaction` share no state or inheritance relationship at all, only a capability.

### Quick Check

Could `Auditable` have been designed as an abstract class instead of an interface?

Not without real cost: since a class can extend only one class, making `Auditable` an abstract class would force every implementer to spend their one allowed superclass slot on it, which would make it impossible for `BankAccount` to simultaneously extend nothing else meaningful and still be extended by `SavingsAccount`, and impossible for `Transaction`, which genuinely needs `abstract class Transaction` as its actual superclass for shared state, to also be `Auditable` through inheritance. The interface design avoids this conflict entirely, since implementing any number of interfaces costs nothing toward the single class inheritance allowance.

---

## 7. Interface as a Type with Lend A Hand

### Concept Explanation

An interface name can be used as a reference type, exactly like a class name, even though it can never be instantiated directly. A variable declared with an interface type can hold a reference to any object whose actual class implements that interface, and calling a method through that reference resolves, at runtime, to whichever concrete class the object actually is, following the exact same dynamic dispatch rules from Chunk 3.

```java
Auditable item = new BankAccount("Zoya", 700);
System.out.println(item.getAuditLog());

item = new DepositTransaction(50);
System.out.println(item.getAuditLog());
```

The same variable, `item`, declared as type `Auditable`, is reassigned across two completely unrelated concrete classes over its lifetime, and each call to `item.getAuditLog()` correctly runs that specific object's own implementation. Only methods declared in `Auditable` itself, here just `getAuditLog()`, can be called through this reference without an explicit cast, exactly mirroring how a `BankAccount` typed reference in Chunk 3 could not call `CheckingAccount` specific methods without first casting.

### Lend A Hand: Quiz

1. Given `Auditable item = new BankAccount(...);`, can `item.deposit(100)` be called directly?
   A. Yes, always  B. No, `Auditable` does not declare `deposit`, so the compiler rejects the call through this reference type, regardless of the object's actual class  C. Only if `BankAccount` is final  D. Runtime exception

2. What determines which class's `getAuditLog()` actually runs when called through an `Auditable` reference?
   A. The declared type, `Auditable`  B. The object's actual runtime class  C. Whichever was implemented first  D. Compile error, ambiguous

3. Could a single `Auditable[]` array hold a `BankAccount`, a `SavingsAccount`, and a `DepositTransaction` all together?
   A. No, they must share a common class  B. Yes, since all three genuinely implement `Auditable`, directly or through inheritance  C. Only two at a time  D. Compile error

### Answers

1. **B.** The compiler only permits calls that the reference's declared type actually promises; `Auditable` promises only `getAuditLog()`, so `deposit`, even though it genuinely exists on the actual `BankAccount` object underneath, is not callable without first casting `item` back to `BankAccount`.

2. **B.** Exactly as with ordinary overriding, an interface method call resolves at runtime based on the actual object's real class, not the declared reference type, which is precisely what "interface as a type" enabling polymorphism means.

3. **B.** `BankAccount` implements `Auditable` directly, `SavingsAccount` inherits that implementation, and `DepositTransaction` inherits it through `Transaction`; since all three genuinely fulfill the `Auditable` contract, one way or another, a shared `Auditable[]` array can legally hold references to all three at once.

### Programming Practice

1. Write a method `logAll(Auditable[] items)` that loops over the array with an enhanced for loop and prints every element's audit log, then call it with an array mixing a `BankAccount`, a `SavingsAccount`, and a `WithdrawalTransaction`.

### Solution

```java
public class Solution1 {
    static void logAll(Auditable[] items) {
        for (Auditable item : items) {
            System.out.println(item.getAuditLog());
        }
    }

    public static void main(String[] args) {
        Auditable[] items = {
            new BankAccount("Marco", 400),
            new SavingsAccount("Elena", 900, 0.03),
            new WithdrawalTransaction(75)
        };
        logAll(items);
    }
}
```

Why it works: `logAll` accepts any array of `Auditable`, regardless of what concrete classes actually populate it, exactly mirroring how methods accepting `BankAccount` in Chunk 3 worked across an entire class hierarchy; here that same idea extends across classes that share no inheritance relationship at all, purely through the shared interface.

---

## 8. Interface vs Class

### Concept Explanation

An ordinary class and an interface both define a type that can be used to declare variables and parameters, but they differ sharply in what they are allowed to contain and how objects relate to them.

| | Class | Interface |
|---|---|---|
| Can be instantiated with `new` | Yes, if not `abstract` | Never |
| Instance fields with real, per object state | Yes | No, other than constants |
| Constructors | Yes | No |
| A type can have how many as its direct supertype | One superclass | Any number of interfaces |
| Default access of members if unspecified | Package private | `public` for methods |

A class describes what an object fundamentally is and holds its actual state. An interface describes only what an object can do, a contract of capability, with no state of its own to carry. This is exactly why `BankAccount` is a class, since it genuinely needs to hold `balance`, `accountHolder`, and `accountNumber` as real per object state, while `Auditable` is an interface, since "being auditable" carries no state of its own at all, only a promise to answer `getAuditLog()`.

### Quick Check

Could `Auditable` have a field `private String lastAuditDate;`?

No, not as ordinary instance state; interfaces cannot declare instance fields carrying per object state at all, only `public static final` constants, which are shared, unchanging values rather than per object data. If tracking a last audit date were genuinely needed, that state would have to live in each implementing class instead, such as a field directly inside `BankAccount`.

---

## 9. Inheritance among Interfaces

### Concept Explanation

An interface can extend another interface, using `extends`, exactly the same keyword a class uses to extend a class, though the relationship works slightly differently: an interface can extend multiple other interfaces at once, unlike a class, which can extend only one class.

**Extending `Auditable` for a stricter guarantee.**

```java
public interface DetailedAuditable extends Auditable {
    String getFullAuditTrail();
}
```

Any class implementing `DetailedAuditable` must provide both `getAuditLog()`, inherited from `Auditable`, and `getFullAuditTrail()`, declared directly on `DetailedAuditable` itself. This mirrors class inheritance in spirit: a more specific interface builds on a more general one, adding further requirements rather than replacing what came before.

```java
public class SavingsAccount extends BankAccount implements DetailedAuditable {
    // ... existing members ...

    @Override
    public String getFullAuditTrail() {
        return getAuditLog() + " | interestRate=" + interestRate;
    }
}
```

`SavingsAccount` already inherits `getAuditLog()` from `BankAccount`, satisfying `Auditable`'s half of the contract through ordinary class inheritance, and provides `getFullAuditTrail()` itself to satisfy the rest, satisfying `DetailedAuditable` as a whole. Notice `SavingsAccount` both `extends BankAccount` and `implements DetailedAuditable` simultaneously, exactly the combination Section 6 highlighted as impossible if `Auditable` had been designed as an abstract class instead.

### Quick Check

Since `DetailedAuditable extends Auditable`, is every `DetailedAuditable` also, automatically, an `Auditable`?

Yes. Exactly as a subclass is automatically an instance of its superclass, any type implementing `DetailedAuditable` automatically satisfies `Auditable` too, so a `SavingsAccount` object can be assigned to an `Auditable` typed reference just as easily as to a `DetailedAuditable` typed one.

---

## 10. Interface and Polymorphism

### Concept Explanation

Everything Chunk 3 established about runtime polymorphism for class based overriding applies identically to interface method implementations. A method declared in an interface and implemented differently by several classes resolves, at the moment it is called, based on the actual object's real class, never the declared reference type, precisely the same dynamic dispatch mechanism, just operating across a capability contract rather than a class hierarchy.

```java
public class Demo {
    public static void main(String[] args) {
        Reportable[] items = {
            new BankAccount("Ken", 600),
            new SavingsAccount("Mia", 1200, 0.02),
            new DepositTransaction(80)
        };

        for (Reportable r : items) {
            System.out.println(r.generateReport());
        }
    }
}
```

Each call to `r.generateReport()` runs whichever specific class's own implementation the actual object has, `BankAccount`'s own version for the first element, the inherited `BankAccount` version again for the `SavingsAccount`, since it never overrides `generateReport()` itself, and `Transaction`'s version for the last, all resolved individually at runtime, exactly the same principle demonstrated with `Transaction[]` in Section 2 and `BankAccount[]` in Chunk 3.

### Quick Check

If `SavingsAccount` overrode `generateReport()` itself, adding interest rate details, would the `SavingsAccount` element in the loop above use that override or `BankAccount`'s version?

`SavingsAccount`'s own override, since interface implemented methods are ordinary instance methods once implemented in a class, resolved by the object's actual runtime type exactly like any other overridden method, regardless of whether the array is typed by an interface, `Reportable`, or a class, `BankAccount`.

---

## 11. IS-A and HAS-A Relationships

### Concept Explanation

These two phrases summarize the two fundamental ways one type relates to another in object oriented design, and correctly identifying which one applies in a given situation is one of the most practically useful design skills this entire course builds toward.

**IS-A describes inheritance and interface implementation.** If class `B` extends class `A`, or implements interface `A`, then "a `B` is an `A`" should read as a genuinely true, sensible statement. `SavingsAccount` is a `BankAccount`. `BankAccount` is `Auditable`. `DepositTransaction` is a `Transaction`. Each of these reads naturally and correctly as an IS-A relationship, which is exactly the test for whether inheritance or interface implementation is the appropriate design choice in the first place.

**HAS-A describes composition: one class holding a reference to another as a field.** If class `C` declares a field of type `D`, then "a `C` has a `D`" is the relationship, entirely separate from any inheritance hierarchy.

```java
public class Branch {
    private String branchName;
    private BankAccount[] accounts;

    public Branch(String branchName, BankAccount[] accounts) {
        this.branchName = branchName;
        this.accounts = accounts;
    }

    public double getTotalHoldings() {
        double total = 0;
        for (BankAccount acc : accounts) {
            total += acc.getBalance();
        }
        return total;
    }
}
```

A `Branch` HAS-A `BankAccount[]`; a branch is not itself any kind of account, and there is no IS-A relationship between `Branch` and `BankAccount` at all. `Branch` simply holds references to several accounts and coordinates operations across them, exactly the composition relationship HAS-A describes.

**Why the distinction matters for design.** Choosing inheritance for a relationship that is really HAS-A leads to awkward, overly rigid designs, such as a `Branch` class extending `BankAccount` purely to reuse some method, when a branch is not genuinely a kind of account at all. The IS-A test from the paragraph above is the simplest practical check: if the sentence sounds wrong, composition, a HAS-A field, is almost certainly the better fit than inheritance.

### Quick Check

Using the classes built across this chunk, is the relationship between `Transaction` and `BankAccount` IS-A or HAS-A?

Neither, directly, though it is close to HAS-A in spirit: `Transaction`'s `execute(BankAccount account)` method receives a `BankAccount` as a parameter, temporarily using it, rather than a `Transaction` object permanently holding a `BankAccount` reference as one of its own fields the way `Branch` holds its `accounts` array. `TransferTransaction`, from Section 2's practice problem, does genuinely have a HAS-A relationship with `BankAccount`, since it stores `source` and `destination` as its own fields for the lifetime of the object.

---

## 12. Consolidated Quiz: All Four Chunks Together

This quiz deliberately mixes concepts from every chunk so far: primitives and operators, encapsulation and constructors, inheritance and polymorphism, and now abstract classes and interfaces.

1. Given `abstract class Transaction` with a `private final double amount` field and a `public double getAmount()` getter, why is the field `private` rather than `protected`, given that subclasses need to read it?
   A. It must always be `private`  B. Since a public getter already provides controlled read access, `private` still enforces encapsulation while subclasses use the inherited getter, following the same pattern as `BankAccount`'s fields in Chunk 2  C. `protected` does not work with `abstract`  D. There is no real reason

2. In `DepositTransaction`'s constructor, `super(amount, "Deposit")` is called. What compile time rule from Chunk 3 does this follow?
   A. `super(...)` must be the first statement in the constructor  B. `super(...)` can appear anywhere  C. `super(...)` is optional here  D. This is actually `this(...)`, not `super(...)`

3. Given `Transaction[] transactions` holding a mix of `DepositTransaction` and `WithdrawalTransaction` objects, and a `for (Transaction t : transactions)` loop calling `t.execute(acc)`, which Chunk 1 concept does the loop itself rely on?
   A. The enhanced for each loop  B. The while loop  C. The switch statement  D. Bitwise operators

4. If `WithdrawalTransaction.execute` used `amount > 0 ? account.withdraw(amount) : false` instead of an `if` statement, which Chunk 1 operator is this?
   A. The bitwise OR operator  B. The ternary conditional operator  C. The modulus operator  D. The shift operator

5. Why can a single class, such as `SavingsAccount` from Section 9, both `extends BankAccount` and `implements DetailedAuditable` at the same time, when it could never `extends` two classes at once?
   A. Interfaces do not count as real types  B. Java's single inheritance restriction applies only to classes; a class may implement any number of interfaces alongside its one superclass  C. `DetailedAuditable` is secretly a class  D. This is actually a compile error

6. `BankAccount` overrides `toString()`, from Chunk 3, and separately implements `getAuditLog()` and `generateReport()` from this chunk's interfaces. Are `getAuditLog()` and `generateReport()` examples of overriding or of implementing?
   A. Overriding, since all three are conceptually similar  B. Implementing, since they fulfill interface contracts rather than replacing an inherited class method's behavior, though the mechanics and use of `@Override` are extremely similar  C. Neither, since interfaces have no relation to overriding at all  D. Hiding, since `getAuditLog` could be static

7. A `static int totalAccountsCreated` field exists in `BankAccount`, from Chunk 2. If `Transaction` also had a `static int totalTransactionsCreated` field, incremented in its constructor, would creating a `DepositTransaction` increment it, even though `DepositTransaction` itself declares no such field?
   A. No, since `DepositTransaction` has its own separate copy  B. Yes, since `DepositTransaction`'s constructor calls `super(...)`, running `Transaction`'s constructor, which increments the one shared static field belonging to `Transaction`  C. Compile error  D. Only if `DepositTransaction` is also `abstract`

8. Given `Auditable item = new SavingsAccount("Priya", 500, 0.02);`, can `item` call `applyInterest()` directly?
   A. Yes, since the actual object is a `SavingsAccount`  B. No, since `Auditable` does not declare `applyInterest()`, so the compiler only permits calls it promises, regardless of the object's actual class  C. Only with `super`  D. Runtime exception

9. `CheckingAccount`, from Chunk 3, overrides `withdraw` to account for its overdraft limit. If `CheckingAccount` were passed into `Transaction`'s `execute` via a `WithdrawalTransaction`, which `withdraw` would actually run?
   A. `BankAccount`'s original version, since `execute`'s parameter is typed `BankAccount`  B. `CheckingAccount`'s overridden version, resolved dynamically based on the actual object regardless of the parameter's declared type  C. Compile error  D. Both versions run

10. Is the relationship between `Branch` and `BankAccount[]`, from Section 11, better described as IS-A or HAS-A, and why?
    A. IS-A, since branches process accounts  B. HAS-A, since `Branch` holds an array of `BankAccount` as one of its own fields, rather than `Branch` itself being a kind of `BankAccount`  C. Neither applies  D. IS-A, since `Branch` extends `BankAccount`

### Consolidated Quiz Answers

1. **B.** Exactly as with `BankAccount`'s own fields back in Chunk 2, keeping `amount` `private` and exposing it only through `getAmount()` preserves encapsulation's core guarantee, that state can only be read or changed through controlled methods, while still giving `DepositTransaction` and `WithdrawalTransaction` everything they need via the inherited getter.

2. **A.** As covered in Chunk 3, `super(...)`, if used at all, must be the very first statement in a constructor, and `DepositTransaction`'s constructor correctly follows this rule.

3. **A.** The `for (Transaction t : transactions)` syntax is precisely the enhanced for each loop introduced in Chunk 1, here iterating over an abstract typed array exactly as it iterated over primitive and `BankAccount` typed arrays in earlier chunks.

4. **B.** This is the ternary conditional operator, `condition ? valueIfTrue : valueIfFalse`, from Chunk 1's Operator Precedence section, here choosing between calling `withdraw` and simply yielding `false` based on whether `amount` is positive.

5. **B.** As explained in Section 6, Java's single inheritance restriction, at most one `extends`, applies only to classes; implementing interfaces via `implements` carries no such limit, which is exactly why `SavingsAccount` can combine both relationships at once.

6. **B.** `getAuditLog()` and `generateReport()` fulfill interface contracts, which is technically called implementing rather than overriding, though as noted in Section 3, the mechanics, matching an inherited signature exactly and using `@Override`, are extremely similar to class based overriding from Chunk 3.

7. **B.** Exactly as `totalAccountsCreated` was incremented by every `BankAccount` constructor call regardless of which subclass triggered it in Chunk 3, a hypothetical `totalTransactionsCreated` in `Transaction` would be incremented by `Transaction`'s own constructor, which every concrete subclass's constructor, including `DepositTransaction`'s, calls via `super(...)`.

8. **B.** Exactly as covered in Section 7, the compiler only allows calls that the reference's declared type, `Auditable` here, actually promises; `applyInterest()` is not part of that contract, so it cannot be called through this particular reference without an explicit cast back to `SavingsAccount` first.

9. **B.** This is runtime polymorphism exactly as covered in Chunk 3 and reinforced in Section 10: an overridden instance method resolves based on the actual object's real class at the moment it is called, entirely regardless of what type a method parameter, here `Transaction.execute`'s `BankAccount account` parameter, happens to be declared as.

10. **B.** `Branch` holds a `BankAccount[]` as one of its own fields; it does not extend `BankAccount` or claim to be one, which is precisely the HAS-A, composition relationship described in Section 11, as opposed to the IS-A relationship inheritance and interface implementation both describe.

---

## 13. Programming Practice and Solutions

### Practice Problems

1. **Basic.** Add a `getAmount()`-based validation to `Transaction`'s constructor so that a negative `amount` throws no exception yet, but instead is simply clamped to `0` using `Math.max(0, amount)`, applying the same defensive coding spirit as `BankAccount`'s `deposit` validation from Chunk 2.
2. **Intermediate.** Give `Branch` a method `getAuditReport()` that loops over its `accounts` array and, for each account, checks with `instanceof` whether it also implements `Auditable` (all `BankAccount`s do, so this will always be true here, but write the check anyway as defensive style), appending each account's `getAuditLog()` to one combined `String`.
3. **Intermediate.** Add an interface `Insurable` with a method `double getInsuredAmount()`, have `SavingsAccount` implement it returning the lesser of `getBalance()` and a fixed `100000` cap, and demonstrate polymorphism by calling `getInsuredAmount()` through an `Insurable` typed reference.
4. **Advanced.** Build a small `main` method that creates a `Branch` containing one `BankAccount`, one `SavingsAccount`, and one `CheckingAccount`, applies a `WithdrawalTransaction` to each using a loop, and prints the branch's total holdings before and after, tying together arrays, loops, polymorphism, abstract classes, and encapsulation from across all four chunks in a single program.

### Solutions

**Solution 1.**

```java
public Transaction(double amount, String description) {
    this.amount = Math.max(0, amount);
    this.description = description;
}
```

Why it works: `Math.max(0, amount)` guarantees `amount` never stores a negative value, regardless of what was passed in, following the same defensive validation spirit as `BankAccount.deposit`'s own rejection of non positive amounts back in Chunk 2, just implemented as clamping rather than outright rejection here.

**Solution 2.**

```java
public String getAuditReport() {
    StringBuilder report = new StringBuilder();
    for (BankAccount acc : accounts) {
        if (acc instanceof Auditable) {
            Auditable auditable = (Auditable) acc;
            report.append(auditable.getAuditLog()).append("\n");
        }
    }
    return report.toString();
}
```

Why it works: `instanceof`, from Chunk 1's relational operators, checks whether `acc`'s actual runtime type genuinely implements `Auditable` before attempting the cast, which here always succeeds since `BankAccount` implements `Auditable` directly, but the check itself is the defensively correct pattern for any array whose element type might not always guarantee a particular interface.

**Solution 3.**

```java
public interface Insurable {
    double getInsuredAmount();
}
```

```java
public class SavingsAccount extends BankAccount implements Insurable, DetailedAuditable {
    // ... existing members ...

    @Override
    public double getInsuredAmount() {
        return Math.min(getBalance(), 100000);
    }
}
```

```java
public class Demo {
    public static void main(String[] args) {
        Insurable acc = new SavingsAccount("Tomás", 150000, 0.02);
        System.out.println("Insured amount: " + acc.getInsuredAmount());
    }
}
```

Expected output: `Insured amount: 100000.0`

Why it works: `SavingsAccount` now implements three separate things at once, its one superclass `BankAccount` via `extends`, and two interfaces, `Insurable` and `DetailedAuditable`, via `implements`, exactly demonstrating Section 6's point that interfaces cost nothing toward the single class inheritance limit. `getInsuredAmount()` correctly caps the reported figure at the fixed limit using `Math.min`, resolved dynamically through the `Insurable` typed reference exactly as Section 10 covered.

**Solution 4.**

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount[] accountArray = {
            new BankAccount("Plain", 1000),
            new SavingsAccount("Saver", 2000, 0.03),
            new CheckingAccount("Checker", 500, 300)
        };
        Branch branch = new Branch("Main Street", accountArray);

        System.out.println("Total before: " + branch.getTotalHoldings());

        for (BankAccount acc : accountArray) {
            Transaction t = new WithdrawalTransaction(100);
            boolean success = t.execute(acc);
            System.out.println(acc.getAccountHolder() + " withdrawal success: " + success);
        }

        System.out.println("Total after: " + branch.getTotalHoldings());
    }
}
```

Why it works: this single program draws on the enhanced for loop and array declarations from Chunk 1, the encapsulated `BankAccount` and its `getAccountHolder()`/`getBalance()` from Chunk 2, the `SavingsAccount`/`CheckingAccount` hierarchy and `withdraw` overriding from Chunk 3, and the abstract `Transaction`/`WithdrawalTransaction` pairing along with `Branch`'s HAS-A composition from this chunk, all working together in one coherent flow, exactly the kind of integration a competitive exam's later, harder questions tend to demand.

---

## 14. Revision Summary

| Concept | Key Rule | Where It Appeared |
|---|---|---|
| Abstract class | Cannot be instantiated; may mix abstract and concrete methods; subclasses must implement every abstract method or also be abstract | `Transaction`, `DepositTransaction`, `WithdrawalTransaction` |
| When to use abstract class | Related classes sharing both real state/behavior and at least one forced, subclass specific method | `Transaction`'s shared fields plus abstract `execute` |
| Interface | A contract of method signatures; no instance state, no constructors; implemented via `implements` | `Auditable`, `Reportable` |
| Multiple interfaces | A class can implement any number of interfaces, unlike extending only one class | `BankAccount implements Auditable, Reportable` |
| Interface vs abstract class | Interface for unrelated classes sharing a capability; abstract class for related classes sharing state plus forced behavior | `Auditable` vs `Transaction` |
| Interface vs class | A class holds real state and can be instantiated; an interface holds no state and cannot | `BankAccount` vs `Auditable` |
| Interface as a type | A reference of interface type may point to any implementing object; only that interface's methods are callable without a cast | `Auditable item = new BankAccount(...)` |
| Interface inheritance | An interface can `extend` one or more other interfaces, adding further required methods | `DetailedAuditable extends Auditable` |
| Interface and polymorphism | Interface implemented methods resolve dynamically by actual object type, exactly like class overriding | `Reportable[]` mixing unrelated classes |
| IS-A | Inheritance and interface implementation; test by reading it as a sentence | `SavingsAccount` IS-A `BankAccount`; `BankAccount` IS-A `Auditable` |
| HAS-A | Composition; one class holding another as a field | `Branch` HAS-A `BankAccount[]` |

**Exam tip.** When a question presents a new interface or abstract class, ask two things in order. First: does this group of classes share real state and a constructor, or only a behavioral promise? Real shared state points to an abstract class; a bare promise points to an interface. Second, for any method call through a reference of either kind: is the method being resolved based on what the reference type declares is available, compile time, or based on which concrete implementation actually runs, runtime? The first determines what you are even allowed to call; the second, covered fully by polymorphism, determines what code actually executes.