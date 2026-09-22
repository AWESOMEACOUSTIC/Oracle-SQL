# Java Competitive Exam Preparation
## Chunk 3: Objects as Method Arguments, Returning Objects, and Inheritance

This chunk continues directly from Chunk 2's `BankAccount` class. First it fills in a gap left open there: how objects behave when passed into or returned from methods. Then it grows the scenario into a small class hierarchy, `SavingsAccount` and `CheckingAccount` both extending `BankAccount`, which is what naturally introduces inheritance, the `Object` class, `super`, overriding, polymorphism, and method hiding. Every new idea is shown reusing or reshaping something you already built in Chunk 2, since that connection is exactly what makes these concepts click.

---

## Table of Contents

1. Passing Objects as Method Arguments
2. Returning Objects from a Method
3. Lend A Hand on Objects as Method Arguments
4. What is Inheritance and How to Perform Inheritance
5. The Object Class, Subclass, and Superclass
6. Super Constructor with Lend A Hand
7. Lend A Hand on Inheritance
8. Method Overriding
9. Run Time Polymorphism with Lend A Hand
10. Hiding Methods
11. Lend A Hand on Method Overriding
12. Inheritance Activity
13. Programming Practice and Solutions
14. Revision Summary

---

## 1. Passing Objects as Method Arguments

### Concept Explanation

Java is always strictly pass by value, for both primitives and objects, with no exceptions. What that means differs depending on what kind of value is being passed. For a primitive like `int`, the actual value itself is copied into the parameter. For an object, what gets copied is the reference, the thing that points to the object, not the object itself. The method receives its own copy of that reference, but that copy still points to the exact same object in memory as the original.

This has two consequences that are easy to confuse with each other, so it is worth stating both precisely.

- **Modifying the object's state through the parameter is visible to the caller.** Since the parameter reference points to the same object as the caller's original reference, calling a method on it or changing its fields through a setter genuinely changes that one shared object, and the caller will see that change afterward.
- **Reassigning the parameter to point to a different object is NOT visible to the caller.** Since only the reference was copied, pointing that local copy somewhere else has no effect whatsoever on the caller's original variable, which still points to whatever it always pointed to.

**Demonstrating this with `BankAccount`.**

```java
public class Demo {
    static void addBonus(BankAccount acc) {
        acc.deposit(100); // modifies the shared object's state
    }

    static void tryToReplace(BankAccount acc) {
        acc = new BankAccount("Someone Else", 999999); // only reassigns the local copy
    }

    public static void main(String[] args) {
        BankAccount original = new BankAccount("Tara", 500);

        addBonus(original);
        System.out.println("After addBonus: " + original.getBalance());

        tryToReplace(original);
        System.out.println("After tryToReplace: " + original.getAccountHolder());
    }
}
```

Output:
```
After addBonus: 600.0
After tryToReplace: Tara
```

`addBonus` calls `deposit` on the same object `original` refers to, so the balance change is genuinely visible afterward. `tryToReplace` only changes what its own local parameter `acc` points to; `original` in `main` never pointed anywhere else and is completely unaffected, still referring to Tara's account with its original balance intact.

### Important Notes

- There is no such thing as "pass by reference" in Java; object references themselves are always passed by value, which is a specific and precise claim, not a loose approximation.
- A method receiving an object parameter can freely call any of that object's `public` methods, including ones that modify its private state internally, exactly as Chunk 2's encapsulation relied on.
- Reassigning a parameter inside a method is a purely local operation with zero effect on the caller.
- This is exactly why `withdraw` and `deposit` from Chunk 2 work correctly when called from anywhere: the object they operate on is the one, single, shared object, regardless of how many different references throughout a program might point to it.

### Quick Check

If `addBonus` had instead been written to accept a `double` parameter and reassign it, such as `amount = amount + 100;`, would that have affected any variable back in `main`?

No, and for a related but distinct reason: `double` is a primitive, so its actual value is copied, and reassigning a local copy of a primitive value never affects the caller's original variable either. Objects and primitives actually agree on this point, reassignment of a local parameter never propagates back; the only difference between them is that an object's shared state can still be modified through the copied reference, which a primitive has no equivalent for at all.

---

## 2. Returning Objects from a Method

### Concept Explanation

A method's return type can be a class type just as easily as a primitive type, in which case the method returns a reference to an object rather than a primitive value. This unlocks two extremely common and useful patterns.

**Factory style methods that build and return a new object.** A `static` method can construct a new object internally and return a reference to it, giving callers a convenient, named way to create objects with particular setup logic, without needing a public constructor for every single variation.

```java
public static BankAccount openWithWelcomeBonus(String accountHolder) {
    BankAccount acc = new BankAccount(accountHolder, 0.0);
    acc.deposit(50, "Welcome bonus");
    return acc;
}
```

**Method chaining by returning `this`.** A method can return a reference to the very object it was called on, by writing `return this;`, which lets the caller immediately call another method on the result, chaining several calls together in a single expression. This directly reuses the `this` reference concept from Chunk 2, applied to a new purpose.

```java
public BankAccount deposit(double amount) {
    if (amount > 0) {
        balance += amount;
    }
    return this;
}
```

With this change, `deposit`'s return type has changed from `void` to `BankAccount`, and it now returns a reference to the same account it was called on, letting calls be chained:

```java
BankAccount acc = new BankAccount("Ishaan", 0);
acc.deposit(100).deposit(200).deposit(50);
System.out.println(acc.getBalance());
```

Output: `350.0`

Each `deposit` call returns `this`, the very same `acc` object, so the next `.deposit(...)` in the chain is called directly on that returned reference, which is really just `acc` again. All three deposits accumulate onto the exact same account.

### Important Notes

- A method returning an object type follows the exact same definite return rules from Chunk 1: every path through the method must return a compatible reference, or `null`, or the code fails to compile.
- Returning `this` only makes sense for an instance method, since `this` refers to whichever specific object the method was invoked on; a `static` method has no `this` at all.
- A factory method that returns a newly constructed object is a common alternative, or complement, to constructors, especially useful when the "recipe" for building an object is more elaborate than a constructor alone reads well expressing.
- Chaining reads naturally but every intermediate call in the chain must itself return the right type for the next call to be valid; a chain breaks, with a compile error, the moment one method along the way returns something the next call cannot be made on.

### Quick Check

If `deposit` still validated `amount > 0` and rejected an invalid amount, but still always executed `return this;` regardless of whether the deposit actually happened, would `acc.deposit(-50).deposit(100);` still compile and run without crashing?

Yes. Since `deposit` always returns `this` no matter which branch executes, the chain remains valid even when one particular call in the middle is rejected internally; the invalid deposit simply has no effect on the balance, while the chain itself continues to work, since a valid `BankAccount` reference is still returned either way.

---

## 3. Lend A Hand on Objects as Method Arguments

### Applied Example: Combining argument passing with overloading and static from Chunk 2

```java
public class Bank {
    static boolean transfer(BankAccount from, BankAccount to, double amount) {
        boolean withdrawn = from.withdraw(amount);
        if (!withdrawn) {
            return false;
        }
        to.deposit(amount);
        return true;
    }
}

public class Demo {
    public static void main(String[] args) {
        BankAccount a = new BankAccount("Nikhil", 1000);
        BankAccount b = new BankAccount("Aisha", 200);

        boolean success = Bank.transfer(a, b, 300);
        System.out.println("Transfer succeeded: " + success);
        System.out.println("Nikhil: " + a.getBalance());
        System.out.println("Aisha: " + b.getBalance());
    }
}
```

Output:
```
Transfer succeeded: true
Nikhil: 700.0
Aisha: 500.0
```

`transfer` is `static`, exactly like `BankAccount.getTotalAccountsCreated()` from Chunk 2, since it is a general purpose operation not tied to any one particular account. It receives two separate `BankAccount` references as arguments, both pointing to the caller's actual objects, so `withdraw` and `deposit` called inside `transfer` genuinely modify `a` and `b` themselves, visible in `main` immediately afterward.

### Quiz

1. If `Bank.transfer` reassigned its own `from` parameter partway through, such as `from = to;`, would this affect which object `a` refers to back in `main`?
   A. Yes  B. No, only the local parameter changes  C. Only if `transfer` is not `static`  D. Compile error

2. Why is `transfer` declared `static` rather than as an instance method on `BankAccount` itself?
   A. Static methods run faster  B. It operates on two separate accounts rather than belonging to any single one, mirroring the general purpose static utility pattern from Chunk 2  C. Instance methods cannot take object parameters  D. There is no real reason

3. In the `deposit` chaining example from Section 2, what is the actual type of the value returned by `acc.deposit(100)`?
   A. `void`  B. `double`  C. `BankAccount`  D. `int`

### Answers

1. **B.** Exactly as covered in Section 1, only the local copy of the reference inside `transfer` would change; `a` in `main` keeps pointing to whatever object it always pointed to, completely unaffected by any reassignment happening inside the method.

2. **B.** A transfer fundamentally involves two accounts working together, neither of which is more naturally "the" object the method belongs to, which is exactly the kind of situation, mirroring `getTotalAccountsCreated` tracking state across every account rather than one, that calls for a `static` method instead of an instance method.

3. **C.** Since `deposit` was changed in Section 2 to return `this`, and `this` inside a `BankAccount` method always refers to a `BankAccount`, the expression `acc.deposit(100)` evaluates to a `BankAccount` reference, specifically the very same object `acc` already pointed to.

### Programming Practice

1. Write a static method `printSummary(BankAccount acc)` that prints the account holder and balance, and call it on an account, then modify the account's balance afterward via `deposit`, printing it again directly to confirm the object was never copied.
2. Write a static factory method `openJointDefault()` that creates and returns a `BankAccount` for `"Joint Account"` with an initial balance of `0`, then immediately deposits a fixed `100` welcome bonus into it before returning it.

### Solutions

**Solution 1.**

```java
public class Solution1 {
    static void printSummary(BankAccount acc) {
        System.out.println(acc.getAccountHolder() + ": " + acc.getBalance());
    }

    public static void main(String[] args) {
        BankAccount acc = new BankAccount("Farah", 250);
        printSummary(acc);
        acc.deposit(75);
        printSummary(acc);
    }
}
```

Why it works: `printSummary` receives a reference to the exact same object `main` holds in `acc`, so any change made to that object through `main`'s own code, such as the `deposit` call, is fully visible the next time `printSummary` reads from it, confirming no copy of the underlying account was ever made.

**Solution 2.**

```java
public class Solution2 {
    static BankAccount openJointDefault() {
        BankAccount acc = new BankAccount("Joint Account", 0);
        acc.deposit(100);
        return acc;
    }

    public static void main(String[] args) {
        BankAccount joint = openJointDefault();
        System.out.println(joint.getAccountHolder() + ": " + joint.getBalance());
    }
}
```

Why it works: the factory method builds the object entirely inside itself, applies its own setup logic, here a fixed welcome bonus, and returns the finished reference, giving the caller a fully prepared object through one simple call, without the caller needing to know or repeat the setup steps itself.

---

## 4. What is Inheritance and How to Perform Inheritance

### Growing the Scenario

`BankAccount` works well as a general purpose account, but a real bank offers different kinds of accounts. A savings account earns interest. A checking account allows a limited overdraft. Both are still fundamentally bank accounts: both have a holder, a balance, deposit and withdraw behavior. Inheritance lets you express exactly this relationship in code: "a `SavingsAccount` is a `BankAccount`, plus a bit more."

### Concept Explanation

Inheritance lets one class acquire the fields and methods of another class, using the `extends` keyword. The class doing the inheriting is called the **subclass**, also called the derived or child class. The class being inherited from is called the **superclass**, also called the base or parent class. Java supports single inheritance for classes, meaning one class can extend at most one other class directly, unlike some other languages that allow inheriting from multiple classes at once.

```java
public class SavingsAccount extends BankAccount {
    private double interestRate;
}
```

`SavingsAccount` now automatically has access to every `public` and `protected` member `BankAccount` defines, including `deposit`, `withdraw`, `getBalance`, and `getAccountHolder`, without rewriting any of that code. This is the core payoff of inheritance: shared behavior is written exactly once, in the superclass, and every subclass gets it for free while adding only what genuinely makes it different.

**What inheritance does NOT give a subclass direct access to.** `BankAccount`'s fields, `balance`, `accountHolder`, and `accountNumber`, are `private`, so even though `SavingsAccount` inherits the account object's overall state, it cannot reference `balance` by name directly inside its own code; it must go through the inherited public methods like `getBalance()`, exactly the same restriction any other outside class would face. This is precisely why choosing `protected` instead of `private` sometimes matters specifically for classes designed to be extended, since `protected` members remain hidden from unrelated outside classes but do become directly accessible to subclasses.

| Modifier | Accessible directly inside a subclass? |
|---|---|
| `private` | No |
| default (no keyword) | Only if the subclass is in the same package |
| `protected` | Yes, always, even across packages |
| `public` | Yes, always |

### Important Notes

- `extends` establishes an "is a" relationship: a `SavingsAccount` is a `BankAccount`, which should genuinely make logical sense, not merely be convenient code reuse; inheritance used purely to reuse code between unrelated concepts is considered poor design.
- A class can only `extend` one other class directly, though that superclass can itself extend another class, forming a chain.
- A subclass inherits all non `private` members of its superclass automatically; it does not need to redeclare them.
- Private members of the superclass are still part of every subclass object's actual state in memory, but are only reachable through inherited public or protected methods, never by direct name from the subclass's own code.

### Quick Check

If `SavingsAccount` needs to read the current balance inside one of its own methods, how does it do so, given `balance` is `private` in `BankAccount`?

It calls the inherited public method `getBalance()`, exactly the same way any other class would, since `private` fields are never directly reachable from a subclass, regardless of the inheritance relationship.

---

## 5. The Object Class, Subclass, and Superclass

### Concept Explanation

Every class in Java, if it does not explicitly `extend` some other class, implicitly extends `java.lang.Object`. This means `Object` sits at the very top of every single class hierarchy in Java; every object of every type is, indirectly if not directly, an `Object`. Since `BankAccount` did not write `extends` anything, it implicitly extends `Object`. Since `SavingsAccount` extends `BankAccount`, and `BankAccount` implicitly extends `Object`, `SavingsAccount` is, by chain, also an `Object`.

**What `Object` provides to every class.** `Object` defines several methods every single Java object automatically has, whether or not the class that defines it ever mentions them. The most commonly encountered ones are `toString()`, which returns a `String` representation of the object, `equals(Object other)`, which compares two objects for equality, and `getClass()`, which returns runtime type information about the object.

**The default `toString()`.** Unless a class overrides it, `Object`'s own `toString()` returns a string made up of the class's fully qualified name, an `@` symbol, and the object's hash code in hexadecimal, which is rarely useful on its own and is exactly why classes commonly override `toString()` to produce something meaningful, a topic covered fully in the next section on overriding.

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount("Test", 100);
        System.out.println(acc.toString());
    }
}
```

Sample output: `BankAccount@1b6d3586`

The exact hash code portion varies between runs, but the shape, class name, `@`, then a hex code, is the guaranteed default format.

**Subclass and superclass terminology, precisely.** In `class SavingsAccount extends BankAccount`, `SavingsAccount` is the subclass and `BankAccount` is its superclass. `SavingsAccount` is also, simultaneously, a superclass of anything that might further extend it, and `BankAccount` is a subclass of `Object`. These terms are always relative to a specific pair of classes in a hierarchy, not absolute labels.

### Important Notes

- `Object` is the implicit root of every class hierarchy in Java; there is no way to write a class that does not, directly or indirectly, extend `Object`.
- `toString()`, `equals()`, `hashCode()`, and `getClass()` are available on every object automatically, inherited from `Object`, even if never explicitly written by the programmer.
- The default `toString()` format is class name, `@`, hexadecimal hash code; it is rarely useful as is and is one of the most commonly overridden methods in practice.
- "Subclass" and "superclass" describe a relationship between two specific classes, and the same class can be a subclass with respect to one class and a superclass with respect to another.

### Quick Check

Is `SavingsAccount` an instance of `Object`?

Yes. Since `SavingsAccount extends BankAccount`, and `BankAccount` implicitly extends `Object`, every `SavingsAccount` object is, through that chain, also genuinely an `Object`, which is why calling `.toString()` or `.equals(...)` on a `SavingsAccount` reference always compiles, even before `SavingsAccount` defines any such methods itself.

---

## 6. Super Constructor with Lend A Hand

### Concept Explanation

A subclass's constructor is responsible for initializing the subclass's own added fields, but the superclass portion of the object still needs proper initialization too, exactly the fields and setup logic that already exist in the superclass's own constructors. The `super(...)` call invokes a constructor of the immediate superclass, and it plays exactly the same structural role for inheritance that `this(...)` played for constructor chaining within one class back in Chunk 2.

**The rule.** If used, `super(...)` must be the very first statement in a subclass constructor, exactly mirroring the rule for `this(...)`. If a subclass constructor contains neither an explicit `super(...)` call nor an explicit `this(...)` call, the compiler automatically inserts an implicit call to the superclass's no argument constructor, `super()`, as the first statement. If the superclass has no accessible no argument constructor, that implicit insertion fails and the subclass fails to compile unless it provides an explicit `super(...)` call matching some constructor the superclass actually has.

**Writing `SavingsAccount`'s constructor.**

```java
public class SavingsAccount extends BankAccount {
    private double interestRate;

    public SavingsAccount(String accountHolder, double initialBalance, double interestRate) {
        super(accountHolder, initialBalance);
        this.interestRate = interestRate;
    }

    public void applyInterest() {
        double interest = getBalance() * interestRate;
        deposit(interest);
    }
}
```

`super(accountHolder, initialBalance)` calls `BankAccount`'s own two parameter constructor from Chunk 2, which handles the account number generation and sets `balance` and `accountHolder`, exactly the same setup work that constructor already correctly performs. `SavingsAccount`'s constructor then only needs to handle what is genuinely new here, setting `interestRate`. `applyInterest` calls the inherited `getBalance()` and `deposit(...)` methods, since it has no direct access to the private `balance` field itself, exactly as covered in the previous section.

### Applied Example

```java
public class Demo {
    public static void main(String[] args) {
        SavingsAccount sa = new SavingsAccount("Devika", 1000, 0.05);
        sa.applyInterest();
        System.out.println(sa.getAccountHolder() + ": " + sa.getBalance());
        System.out.println("Total accounts created: " + BankAccount.getTotalAccountsCreated());
    }
}
```

Output:
```
Devika: 1050.0
Total accounts created: 1
```

Constructing a `SavingsAccount` still runs `BankAccount`'s own constructor via `super(...)`, which means the static `totalAccountsCreated` counter from Chunk 2 still correctly increments, exactly as it would for any ordinary `BankAccount`, since a `SavingsAccount` genuinely is one, underneath its added behavior.

### Lend A Hand: Quiz

1. What happens if a subclass constructor calls neither `super(...)` nor `this(...)` explicitly?
   A. Compile error, always  B. The compiler automatically inserts an implicit `super()` call as the first statement  C. The superclass portion is left uninitialized  D. The subclass simply has no superclass state at all

2. Why does `SavingsAccount`'s constructor call `super(accountHolder, initialBalance)` rather than setting those values directly?
   A. It cannot access `BankAccount`'s private fields directly, and reusing the existing constructor logic avoids duplicating it  B. It is required syntax with no real reason  C. `SavingsAccount` has no fields of its own  D. `super` is faster than direct assignment

3. What would happen if `BankAccount` had no accessible no argument constructor, since Chunk 2 gave it constructors that always take at least a name, and `SavingsAccount`'s constructor did not include any explicit `super(...)` call?
   A. Compiles fine, using default values  B. Compile error, since the implicit `super()` call would not match any constructor `BankAccount` actually provides  C. Runtime exception  D. `SavingsAccount` is simply not allowed to extend `BankAccount`

### Answers

1. **B.** Exactly mirroring how a missing explicit `this(...)` still leaves a constructor's own body intact, a missing explicit `super(...)` does not leave the superclass uninitialized; the compiler inserts a call to the superclass's no argument constructor automatically, which must actually exist and be accessible for this to succeed.

2. **A.** `balance` and `accountHolder` are `private` in `BankAccount`, so `SavingsAccount` cannot assign them directly by name at all; calling `super(...)` delegates that initialization to code that does have access, the constructor already written inside `BankAccount` itself, avoiding any duplication of that logic.

3. **B.** Since `BankAccount`, per Chunk 2, actually does provide a genuine no argument constructor, `BankAccount()`, this specific scenario would not occur for it; but in general, if a superclass provided no accessible no argument constructor at all, a subclass constructor without an explicit matching `super(...)` call would fail to compile, since the automatically inserted implicit call would have no matching constructor to invoke.

### Programming Practice

1. Write a `CheckingAccount` class extending `BankAccount` with an added `private double overdraftLimit` field, whose constructor takes an account holder, initial balance, and overdraft limit, correctly using `super(...)` to initialize the inherited state.
2. Add a method `getEffectiveAvailable()` to `CheckingAccount` that returns `getBalance() + overdraftLimit`, representing the true amount available to withdraw including the overdraft allowance.

### Solutions

**Solution 1.**

```java
public class CheckingAccount extends BankAccount {
    private double overdraftLimit;

    public CheckingAccount(String accountHolder, double initialBalance, double overdraftLimit) {
        super(accountHolder, initialBalance);
        this.overdraftLimit = overdraftLimit;
    }
}
```

Why it works: `super(accountHolder, initialBalance)` reuses `BankAccount`'s existing two parameter constructor for everything `CheckingAccount` shares with any ordinary account, and `this.overdraftLimit = overdraftLimit;` handles only what is genuinely new to `CheckingAccount`, following exactly the same pattern established for `SavingsAccount`.

**Solution 2.**

```java
public double getEffectiveAvailable() {
    return getBalance() + overdraftLimit;
}
```

Why it works: since `balance` itself is not directly reachable from `CheckingAccount`, the calculation goes through the inherited `getBalance()` method instead, correctly combining the account's actual balance with its own new `overdraftLimit` field, which `CheckingAccount` does have direct access to, since it declared that field itself.

---

## 7. Lend A Hand on Inheritance

### Applied Example: A small hierarchy working together

```java
public class Demo {
    public static void main(String[] args) {
        SavingsAccount sa = new SavingsAccount("Om", 2000, 0.04);
        CheckingAccount ca = new CheckingAccount("Priyanka", 300, 500);

        sa.deposit(0); // inherited directly from BankAccount, unchanged
        System.out.println(sa.getAccountHolder() + " is a savings account with balance " + sa.getBalance());

        boolean overdrawn = ca.withdraw(600); // inherited withdraw, does not know about overdraft yet
        System.out.println("Withdrawal of 600 from checking succeeded: " + overdrawn);
        System.out.println("Effective available: " + ca.getEffectiveAvailable());
    }
}
```

Output:
```
Om is a savings account with balance 2000.0
Withdrawal of 600 from checking succeeded: false
Effective available: 800.0
```

Notice `ca.withdraw(600)` returns `false`, since `CheckingAccount` has not yet changed `withdraw`'s behavior at all; it is still using `BankAccount`'s original version exactly as inherited, which has no concept of an overdraft allowance yet. This is precisely the gap that Method Overriding, the next major topic, exists to close.

### Quiz

1. Why does `ca.withdraw(600)` still fail even though `getEffectiveAvailable()` reports `800.0` as available?
   A. Bug in `getEffectiveAvailable`  B. `withdraw` is inherited unchanged from `BankAccount` and has no awareness of the overdraft limit at all  C. `600` is too large regardless  D. Compile error

2. Can `sa.deposit(0)` be called on a `SavingsAccount`, even though `deposit` is defined in `BankAccount`?
   A. No, only inherited fields are accessible, not methods  B. Yes, public methods are inherited and callable exactly as if declared directly on the subclass  C. Only if overridden first  D. Only with `super.deposit(0)`

3. What general problem does this example illustrate about inheritance alone, without overriding?
   A. Inheritance is broken  B. A subclass automatically gets the superclass's behavior exactly as is, which is not always the full behavior the subclass actually needs  C. `CheckingAccount` should not extend `BankAccount`  D. There is no problem

### Answers

1. **B.** `CheckingAccount` inherited `withdraw` from `BankAccount` without changing it in any way, so it still only permits withdrawals up to the current `balance`, with no knowledge whatsoever of the separate `overdraftLimit` field `CheckingAccount` itself added.

2. **B.** Public methods, exactly like public fields would be if any existed, are inherited in full; calling `sa.deposit(0)` works precisely as though `deposit` had been written directly inside `SavingsAccount`, since inheritance genuinely grants that behavior, not merely a reference to it.

3. **B.** Inheritance alone only ever reuses a superclass's existing behavior unchanged; when a subclass genuinely needs different behavior for some inherited method, as `CheckingAccount` does for `withdraw`, that requires overriding, not inheritance by itself.

### Programming Practice

1. Write a `main` method that creates one `SavingsAccount` and one `CheckingAccount`, stores both in a single `BankAccount[]` array, and uses an enhanced for loop, from Chunk 1, to print each account holder's name and balance, demonstrating that a `BankAccount[]` array can legally hold subclass objects.

### Solution

```java
public class Solution1 {
    public static void main(String[] args) {
        BankAccount[] accounts = {
            new SavingsAccount("Om", 2000, 0.04),
            new CheckingAccount("Priyanka", 300, 500)
        };

        for (BankAccount acc : accounts) {
            System.out.println(acc.getAccountHolder() + ": " + acc.getBalance());
        }
    }
}
```

Why it works: since both `SavingsAccount` and `CheckingAccount` are, through `extends`, genuinely `BankAccount` objects, an array declared as `BankAccount[]` can legally hold references to either kind, and the enhanced for loop from Chunk 1 iterates over them exactly like any other array, calling the inherited `getAccountHolder()` and `getBalance()` methods on each one in turn. This array holding a mix of subclass objects through a superclass typed reference is the direct setup for the Run Time Polymorphism section coming up shortly.

---

## 8. Method Overriding

### Concept Explanation

Method overriding means a subclass provides its own new implementation of a method it inherited from its superclass, using the exact same method signature, the same name and the same parameter list. This is precisely how `CheckingAccount` can fix the gap seen in Section 7: it can override `withdraw` to account for its overdraft limit, replacing the inherited version's behavior entirely for objects of that specific subclass.

**Rules an overriding method must follow.**

- The method name and parameter list must match the superclass version exactly.
- The return type must be the same type, or, since Java 5, a covariant return type, meaning a subtype of the original return type.
- The access modifier cannot be more restrictive than the superclass version; a `public` method cannot be overridden as `protected`, though it could be widened, such as overriding a `protected` method as `public`.
- `static`, `final`, and `private` methods cannot be overridden at all; a `static` method with a matching signature in a subclass hides rather than overrides, the subject of Section 10, and `private` methods are not inherited or visible to a subclass in the first place, so a matching signature there is simply an unrelated new method.
- The `@Override` annotation, while not strictly mandatory, is strongly recommended directly above an overriding method, since it instructs the compiler to verify the method genuinely does override something, catching accidental typos in the signature that would otherwise silently create an unrelated overload instead of the intended override.

**Overriding `withdraw` in `CheckingAccount`.** `BankAccount`'s own `withdraw` refuses to let `balance` go negative at all, so it cannot simply be reused unchanged once an overdraft should be allowed to push the effective balance below zero. A correct override needs a way to actually adjust `balance` beyond what the inherited public methods alone allow, which is exactly the kind of situation `protected` access exists for: `BankAccount` can expose a `protected` helper specifically for subclasses to use, while still keeping it hidden from unrelated outside code.

```java
// inside BankAccount
protected void adjustBalance(double delta) {
    balance += delta;
}
```

```java
// inside CheckingAccount
@Override
public boolean withdraw(double amount) {
    if (amount <= 0 || amount > getBalance() + overdraftLimit) {
        return false;
    }
    adjustBalance(-amount);
    return true;
}
```

`adjustBalance` is `protected`, so it is directly callable from `CheckingAccount`'s own code, per the access table in Section 4, even though `balance` itself remains `private` and still cannot be named directly. The override's condition allows withdrawals that would push the balance negative, as long as the result does not exceed the overdraft limit, which is exactly the new behavior `CheckingAccount` needed and `BankAccount`'s original version could never provide.

**Calling the overridden version versus the original with `super`.** Inside an overriding method, `super.methodName(...)` explicitly calls the superclass's original version of that method, which is useful when the override wants to extend rather than fully replace the inherited behavior.

```java
@Override
public String getAccountHolder() {
    return super.getAccountHolder() + " (Checking)";
}
```

This override calls `BankAccount`'s original `getAccountHolder()` first, via `super`, and then adds extra text onto the result, rather than reimplementing the whole thing from scratch.

### Important Notes

- Overriding requires an identical signature; overloading, covered in Chunk 2, requires the same name with a genuinely different parameter list. These are easy to confuse by name alone but are entirely different mechanisms with entirely different resolution rules, compile time for overloading, runtime for overriding, which the next section develops fully.
- `@Override` catches a common and otherwise silent mistake: writing a method that looks like it should override something but actually has a slightly different signature, which the compiler would otherwise happily accept as a brand new, unrelated method instead of flagging as an error.
- `super.methodName(...)` reaches the superclass's version of an overridden method from within the override itself.
- Access modifiers can only stay the same or become less restrictive when overriding, never more restrictive.

### Quick Check

If `BankAccount` declares `public boolean withdraw(double amount)`, could a subclass override it as `protected boolean withdraw(double amount)`?

No. `protected` is more restrictive than `public`, and an overriding method is never allowed to narrow the access level of the method it overrides; it may keep it the same or widen it, but never narrow it.

---

## 9. Run Time Polymorphism with Lend A Hand

### Concept Explanation

Polymorphism means "many forms," and in this context it refers to the fact that a single method call, written once in your code, can behave differently at runtime depending on the actual type of the object it is called on. Java achieves this specifically for overridden instance methods through a mechanism called dynamic method dispatch, or virtual method invocation: the JVM decides, at the moment the method is actually called, which version of an overridden method to run, based on the object's real, runtime type, not the type of the reference variable used to call it.

**Demonstrating dynamic dispatch.**

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount[] accounts = {
            new BankAccount("Generic", 1000),
            new SavingsAccount("Om", 2000, 0.04),
            new CheckingAccount("Priyanka", 300, 500)
        };

        for (BankAccount acc : accounts) {
            System.out.println(acc.getAccountHolder());
        }
    }
}
```

Output:
```
Generic
Om
Priyanka (Checking)
```

Every element of `accounts` is declared with the compile time type `BankAccount`, yet calling `acc.getAccountHolder()` on the `CheckingAccount` object correctly runs `CheckingAccount`'s own overridden version, appending `" (Checking)"`, while the same exact line of code, `acc.getAccountHolder()`, runs plain `BankAccount`'s version for the first two elements, since neither is a `CheckingAccount`. The compiler only ever checks that `BankAccount` has a `getAccountHolder()` method at all, since that is all it can know from the declared array type; which specific version actually executes is decided at runtime, individually, for each object.

**Why this matters practically.** Polymorphism lets code written once, operating purely in terms of a superclass type, correctly handle any current or even future subclass without being rewritten, as long as each subclass properly overrides the relevant behavior. The loop above never mentions `SavingsAccount` or `CheckingAccount` by name at all, yet it correctly invokes each one's own specific behavior.

### Lend A Hand: Quiz

1. In the example above, what determines which version of `getAccountHolder()` actually runs for a given array element?
   A. The declared type of the array, `BankAccount[]`  B. The actual runtime type of the specific object stored in that slot  C. The order of the elements  D. Whether `@Override` was used

2. If `SavingsAccount` did NOT override `getAccountHolder()` at all, what would `sa.getAccountHolder()` print, given `sa` is a `SavingsAccount`?
   A. Compile error, since `SavingsAccount` does not define it  B. The inherited `BankAccount` version's plain result, with no `(Checking)` style suffix  C. `null`  D. `"SavingsAccount"`

3. Can a `BankAccount` reference variable be used to call a method that only `CheckingAccount` defines and does not override from `BankAccount`, such as `getEffectiveAvailable()`?
   A. Yes, always  B. No, not without an explicit cast to `CheckingAccount` first, since the compiler only allows calls that `BankAccount`'s own declared type supports  C. Only inside a loop  D. Only if `@Override` is used

### Answers

1. **B.** Dynamic method dispatch, the mechanism underlying runtime polymorphism, resolves an overridden instance method call based on the actual object's real type at runtime, completely independent of what type the reference variable holding it was declared as.

2. **B.** Without an override, `SavingsAccount` simply uses the inherited `BankAccount` version unchanged, exactly as `CheckingAccount` originally did for `withdraw` back in Section 7, before it was given its own override; there is no automatic special behavior just from being a subclass, only from actually overriding.

3. **B.** The compiler checks method calls against the reference variable's declared, compile time type, which here is `BankAccount`; since `BankAccount` does not declare `getEffectiveAvailable()` at all, calling it through a plain `BankAccount` reference is a compile error, regardless of what the object's actual runtime type happens to be, unless the reference is first explicitly cast to `CheckingAccount`.

### Programming Practice

1. Write a program with a `BankAccount[]` array mixing all three classes from this chunk, and use an enhanced for loop to call an overridden `getAccountHolder()` style method on each, confirming through the printed output that each object's own specific version ran.

### Solution

```java
public class Solution1 {
    public static void main(String[] args) {
        BankAccount[] accounts = {
            new BankAccount("Wei", 500),
            new SavingsAccount("Anaya", 1500, 0.03),
            new CheckingAccount("Diego", 400, 200)
        };

        for (BankAccount acc : accounts) {
            System.out.println(acc.getAccountHolder());
        }
    }
}
```

Why it works: only `CheckingAccount` overrides `getAccountHolder()` in this chunk's design, so its element alone prints with the `(Checking)` suffix, while the plain `BankAccount` and the `SavingsAccount`, which does not override this particular method, both print their holder's name unchanged, directly confirming that each call was resolved individually based on each object's actual runtime type.

---

## 10. Hiding Methods

### Concept Explanation

Method hiding looks superficially similar to overriding but behaves fundamentally differently, and the distinction is one of the most commonly tested traps in this entire topic area. Hiding applies specifically to `static` methods. When a subclass declares a `static` method with the exact same signature as a `static` method already declared in its superclass, this does not override the superclass version at all; it hides it. The critical difference is how the call is resolved: an overridden instance method is resolved at runtime based on the object's actual type, but a hidden static method is resolved at compile time, based purely on the declared, compile time type of the reference used to call it.

**Demonstrating hiding.**

```java
public class BankAccount {
    // ... existing members ...
    public static String getBankName() {
        return "Central Community Bank";
    }
}

public class SavingsAccount extends BankAccount {
    // ... existing members ...
    public static String getBankName() {
        return "Central Community Bank - Savings Division";
    }
}
```

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount ref1 = new SavingsAccount("Test", 0, 0.01);
        SavingsAccount ref2 = new SavingsAccount("Test", 0, 0.01);

        System.out.println(ref1.getBankName());
        System.out.println(ref2.getBankName());
    }
}
```

Output:
```
Central Community Bank
Central Community Bank - Savings Division
```

Both `ref1` and `ref2` point to the exact same kind of object, a `SavingsAccount`, yet calling `getBankName()` on them produces two different results. This is the defining signature of hiding rather than overriding: `ref1`'s compile time declared type is `BankAccount`, so the compiler resolves `ref1.getBankName()` to `BankAccount`'s own version, completely regardless of what actual object `ref1` happens to point to at runtime. `ref2`'s declared type is `SavingsAccount`, so its call resolves to `SavingsAccount`'s version instead. Contrast this directly with Section 9's `getAccountHolder()` example, where every element, regardless of its declared array type, correctly ran its own actual object's overridden version; that difference is exactly what separates overriding from hiding.

**Calling a static method through an instance reference is legal but discouraged.** Writing `ref1.getBankName()` works, since Java permits calling a static method through an instance reference, but it is misleading exactly because of the behavior just demonstrated; the far clearer, and idiomatically correct, style is always `BankAccount.getBankName()` or `SavingsAccount.getBankName()`, calling explicitly through the class name, which makes the compile time nature of the resolution visually obvious.

### Important Notes

- Hiding applies to `static` methods; overriding applies to instance methods. The two mechanisms look similar in code but resolve completely differently.
- A hidden static method is resolved using the reference variable's compile time declared type; an overridden instance method is resolved using the object's actual runtime type.
- Calling a static method through an object reference, though legal, is poor style specifically because it visually resembles an instance method call while actually behaving according to entirely different, compile time rules.
- Fields, incidentally, behave like hiding rather than overriding too: Java has no concept of polymorphic instance fields at all, so if a subclass declares a field with the same name as one in its superclass, access through a superclass typed reference always resolves to the superclass's field, exactly mirroring the static method hiding example above, which is worth remembering as a related trap even though this chunk's scenario has not needed to demonstrate it directly, since `BankAccount`'s fields are all private.

### Quick Check

If `getBankName()` had been declared as an ordinary instance method rather than `static` in both classes, with an identical body otherwise, what would `ref1.getBankName()` print?

`"Central Community Bank - Savings Division"`, the `SavingsAccount` version, since removing `static` would turn this into ordinary overriding, resolved by the object's actual runtime type, exactly as `getAccountHolder()` was resolved in Section 9, rather than by the reference variable's declared, compile time type.

---

## 11. Lend A Hand on Method Overriding

### Applied Example: Overriding toString, connecting back to the Object class

```java
public class BankAccount {
    // ... existing members ...
    @Override
    public String toString() {
        return "BankAccount[holder=" + getAccountHolder() + ", balance=" + getBalance() + "]";
    }
}
```

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount acc = new SavingsAccount("Lucas", 800, 0.02);
        System.out.println(acc);
    }
}
```

Output: `BankAccount[holder=Lucas, balance=800.0]`

`System.out.println(acc)` implicitly calls `acc.toString()`. Since `SavingsAccount` does not override `toString()` itself, it inherits `BankAccount`'s overridden version unchanged, which produces a far more useful result than the default `Object` provided format from Section 5, `BankAccount@somehashcode`, demonstrating exactly why overriding `toString()` is one of the most common and genuinely useful overrides in ordinary Java code.

### Quiz

1. Why does `System.out.println(acc)` print a meaningful summary rather than the default `Object` style output?
   A. `println` is special cased for `BankAccount`  B. `BankAccount` overrides `toString()`, and `println` implicitly calls it  C. `SavingsAccount` overrides `toString()`  D. Compile error

2. If `SavingsAccount` also overrode `toString()` itself, which version would run for a `BankAccount acc = new SavingsAccount(...)` reference calling `acc.toString()`?
   A. `BankAccount`'s version, since `acc`'s declared type is `BankAccount`  B. `SavingsAccount`'s version, since `toString()` is an instance method resolved by the object's actual runtime type  C. Both run  D. Compile error

3. Is overriding `toString()` an example of overriding or hiding?
   A. Hiding, since it is inherited from `Object`  B. Overriding, since `toString()` is an ordinary instance method, not static  C. Neither applies to `Object`'s methods  D. It depends on the access modifier used

### Answers

1. **B.** `println` internally calls `.toString()` on whatever object it is given, and since `BankAccount` overrides that method with its own meaningful implementation, that version runs instead of `Object`'s default, unhelpful format.

2. **B.** `toString()` is an ordinary instance method, so exactly like `getAccountHolder()` in Section 9, it is resolved through dynamic dispatch based on the object's actual runtime type, `SavingsAccount` here, completely regardless of `acc`'s declared compile time type.

3. **B.** `toString()` is a perfectly ordinary, non static instance method inherited from `Object`; overriding it follows every single rule covered in Section 8, and calls to it resolve dynamically exactly like any other overridden instance method, with nothing special about the fact that it happens to originate from `Object` rather than from a class the programmer wrote.

### Programming Practice

1. Add an overridden `toString()` to `SavingsAccount` that includes the interest rate, and demonstrate, using a `BankAccount` typed reference pointing to a `SavingsAccount` object, that this more specific version correctly runs instead of `BankAccount`'s own `toString()`.

### Solution

```java
public class SavingsAccount extends BankAccount {
    // ... existing members ...
    @Override
    public String toString() {
        return super.toString() + " [interestRate=" + interestRate + "]";
    }
}
```

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount acc = new SavingsAccount("Yuki", 1200, 0.03);
        System.out.println(acc);
    }
}
```

Expected output: `BankAccount[holder=Yuki, balance=1200.0] [interestRate=0.03]`

Why it works: `SavingsAccount`'s override calls `super.toString()` first to reuse `BankAccount`'s own formatted output rather than duplicating it, then appends the additional interest rate detail. Since `toString()` resolves dynamically by actual object type, this `SavingsAccount` specific version correctly runs even though the reference `acc` is declared as plain `BankAccount`, directly confirming runtime polymorphism at work exactly as demonstrated throughout Section 9.

---

## 12. Inheritance Activity

This activity ties every concept in this chunk, and several from Chunk 2, into one connected exercise. Work through it in order; each part depends on the class hierarchy built by the previous parts.

### The Activity

1. Confirm `BankAccount`, `SavingsAccount`, and `CheckingAccount` all compile together as one coherent hierarchy, with `BankAccount` including the `toString()` override from Section 11, `SavingsAccount` including its own `toString()` override and `applyInterest()` method from Section 6, and `CheckingAccount` including its overridden `withdraw()` and `getAccountHolder()` from Section 8.
2. Create a `BankAccount[]` array holding one instance of each of the three classes.
3. Loop over the array with an enhanced for loop, and for each account, print its `toString()` result, then attempt a `withdraw(10000)` call and print whether it succeeded.
4. Separately, call the static `getBankName()` method three different ways: through `BankAccount.getBankName()`, through a `BankAccount` typed reference pointing to a `SavingsAccount`, and through a `SavingsAccount` typed reference, and explain in a comment why the results are not all identical.
5. Add up the `getTotalAccountsCreated()` count after all objects are built and confirm it reflects every object created, including the `SavingsAccount` and `CheckingAccount` instances, connecting back to the static counter and `super(...)` chaining from Section 6.

### Consolidated Quiz

1. When the enhanced for loop in step 3 calls `withdraw(10000)` on the `CheckingAccount` element, which version of `withdraw` runs?
   A. `BankAccount`'s original version  B. `CheckingAccount`'s overridden version, resolved at runtime regardless of the array's declared element type  C. Both versions run in sequence  D. Compile error, since the array is typed `BankAccount[]`

2. Why does step 5's total correctly include the `SavingsAccount` and `CheckingAccount` objects, even though `totalAccountsCreated` is declared inside `BankAccount`, not inside either subclass?
   A. Each subclass gets its own separate copy of the counter  B. Every subclass constructor eventually calls a `BankAccount` constructor via `super(...)`, which increments the one shared static field regardless of which subclass is actually being built  C. Coincidence  D. It does not actually include them

3. In step 4, why do the three different `getBankName()` calls potentially produce different results?
   A. `getBankName` is overridden, resolved dynamically  B. `getBankName` is static and hidden, resolved by each reference's declared compile time type, not the actual object's runtime type  C. Random behavior  D. Compile error, since static methods cannot be called this way at all

4. If `CheckingAccount` had not overridden `getAccountHolder()`, would step 3's printed `toString()` output for the `CheckingAccount` element still include a `" (Checking)"` style suffix anywhere?
   A. Yes, unrelated to any specific override  B. No, since `toString()` calls `getAccountHolder()` internally within `BankAccount`'s implementation, and without `CheckingAccount`'s own override, the plain inherited version, with no suffix, would be used instead  C. Compile error  D. Only if `@Override` is present

5. Which single concept from Chunk 2 makes it possible for `SavingsAccount`'s constructor to avoid rewriting the account number generation logic that already exists in `BankAccount`?
   A. Method overloading  B. Static methods  C. Constructor chaining, here applied across classes through `super(...)`, mirroring `this(...)` chaining within one class  D. Access modifiers

### Consolidated Quiz Answers

1. **B.** `withdraw` is an ordinary instance method that `CheckingAccount` genuinely overrides, so exactly as covered in Section 9, dynamic method dispatch resolves the call based on the object's actual runtime type, a `CheckingAccount`, entirely regardless of the array being declared as `BankAccount[]`.

2. **B.** As covered in Section 6, every subclass constructor, directly or indirectly, ultimately calls one of `BankAccount`'s own constructors through `super(...)`, and since `totalAccountsCreated` is `static`, exactly one shared copy exists across the entire hierarchy, incremented by that one shared constructor regardless of which specific subclass triggered it.

3. **B.** As covered in Section 10, `getBankName` is `static`, so it is hidden rather than overridden by any redefinition in `SavingsAccount`, and its resolution depends entirely on the compile time declared type of whichever reference is used to call it, not on the actual object underneath.

4. **B.** `BankAccount`'s `toString()`, as written in Section 11, calls `getAccountHolder()` internally; if `CheckingAccount` had never overridden that method, this internal call would simply run the inherited, plain `BankAccount` version instead, with no `(Checking)` suffix anywhere in the resulting output.

5. **C.** This is precisely constructor chaining, using `super(...)` to delegate to an already written constructor rather than duplicating its logic, structurally identical to how `this(...)` let `BankAccount`'s own overloaded constructors delegate to each other back in Chunk 2.

---

## 13. Programming Practice and Solutions

### Practice Problems

1. **Basic.** Add a `protected` helper method `logActivity(String message)` to `BankAccount` that simply prints the account holder's name alongside the message, and call it from within `SavingsAccount`'s `applyInterest()` method to log the interest application, demonstrating that `protected` members, unlike `private` ones, are directly callable from subclass code.
2. **Intermediate.** Give `CheckingAccount` an overloaded constructor, following Chunk 2's overloading rules, that accepts only an account holder and initial balance, chaining via `this(...)` to the existing three parameter constructor with a sensible default overdraft limit.
3. **Intermediate.** Write a method `applyMonthlyMaintenance(BankAccount acc)` that takes any `BankAccount`, including a subclass instance, and calls `acc.withdraw(5)` on it, printing whether the fee was successfully collected, demonstrating that passing subclass objects as superclass typed arguments works exactly as with any other object argument from Section 1.
4. **Advanced.** Add a new `PremiumSavingsAccount` class extending `SavingsAccount`, overriding `applyInterest()` to apply double the inherited interest rate's effect by calling `super.applyInterest()` twice, and demonstrate polymorphic behavior by calling `applyInterest()` on a `SavingsAccount[]` array containing both plain `SavingsAccount` and `PremiumSavingsAccount` objects.

### Solutions

**Solution 1.**

```java
// inside BankAccount
protected void logActivity(String message) {
    System.out.println("[" + getAccountHolder() + "] " + message);
}
```

```java
// inside SavingsAccount
public void applyInterest() {
    double interest = getBalance() * interestRate;
    deposit(interest);
    logActivity("Interest of " + interest + " applied.");
}
```

Why it works: `logActivity` is `protected`, so unlike a `private` member, it is directly callable from `SavingsAccount`'s own code without going through any additional getter, exactly the accessibility difference laid out in the access modifier table back in Section 4.

**Solution 2.**

```java
public CheckingAccount(String accountHolder, double initialBalance) {
    this(accountHolder, initialBalance, 200.0);
}
```

Why it works: this follows exactly the constructor overloading pattern from Chunk 2, delegating via `this(...)` to the existing, more complete constructor with a chosen sensible default for the parameter being omitted, avoiding any duplicated initialization logic.

**Solution 3.**

```java
public class Solution3 {
    static void applyMonthlyMaintenance(BankAccount acc) {
        boolean collected = acc.withdraw(5);
        System.out.println("Fee collected from " + acc.getAccountHolder() + ": " + collected);
    }

    public static void main(String[] args) {
        applyMonthlyMaintenance(new BankAccount("Plain", 10));
        applyMonthlyMaintenance(new SavingsAccount("Saver", 3, 0.02));
    }
}
```

Expected output:
```
Fee collected from Plain: true
Fee collected from Saver: false
```

Why it works: `applyMonthlyMaintenance` accepts any `BankAccount`, including subclass instances, exactly as Section 1 established for object arguments generally, and since `withdraw` is called on whatever the actual object is, the plain account, with enough balance, succeeds, while the saver, with too little balance and no overdraft allowance, correctly fails.

**Solution 4.**

```java
public class PremiumSavingsAccount extends SavingsAccount {
    public PremiumSavingsAccount(String accountHolder, double initialBalance, double interestRate) {
        super(accountHolder, initialBalance, interestRate);
    }

    @Override
    public void applyInterest() {
        super.applyInterest();
        super.applyInterest();
    }
}
```

```java
public class Demo {
    public static void main(String[] args) {
        SavingsAccount[] accounts = {
            new SavingsAccount("Regular", 1000, 0.05),
            new PremiumSavingsAccount("Premium", 1000, 0.05)
        };

        for (SavingsAccount sa : accounts) {
            sa.applyInterest();
            System.out.println(sa.getAccountHolder() + ": " + sa.getBalance());
        }
    }
}
```

Expected output:
```
Regular: 1050.0
Premium: 1102.5
```

Why it works: `PremiumSavingsAccount` overrides `applyInterest()` to call the inherited version twice through `super.applyInterest()`, compounding the interest effect rather than reimplementing the calculation from scratch. The loop calling `sa.applyInterest()` correctly dispatches to each object's own actual version at runtime, exactly as covered throughout Section 9, even though every element in the array is declared with the shared type `SavingsAccount`.

---

## 14. Revision Summary

| Concept | Key Rule | Where It Appeared |
|---|---|---|
| Passing objects | References are passed by value; state changes through the reference are visible to the caller, reassigning the parameter is not | `addBonus` vs `tryToReplace` |
| Returning objects | A method can return a new object, or return `this` for chaining | `openWithWelcomeBonus`, chained `deposit` |
| Inheritance | `extends` grants non private members automatically; single inheritance only | `SavingsAccount extends BankAccount` |
| `Object` class | Root of every hierarchy; provides `toString()`, `equals()`, `getClass()` by default | Default `toString()` before overriding |
| `super(...)` | Calls a superclass constructor; must be the first statement; mirrors `this(...)` chaining | `SavingsAccount`'s constructor |
| Method overriding | Same signature, same or covariant return type, same or wider access, resolved at runtime | `CheckingAccount.withdraw`, `toString()` |
| Runtime polymorphism | Overridden instance methods resolve by the object's actual type, not the reference's declared type | `BankAccount[]` holding mixed subclasses |
| Method hiding | Static methods with a matching signature hide rather than override; resolved by the reference's declared type | `getBankName()` |

**Exam tip.** Whenever a question involves a superclass typed reference pointing to a subclass object, ask exactly one question first: is the method being called `static` or an instance method? If it is an instance method, resolution follows the object's actual runtime type, runtime polymorphism. If it is `static`, resolution follows the reference variable's declared compile time type, hiding. This single distinction resolves the large majority of tricky inheritance output questions on a competitive exam.