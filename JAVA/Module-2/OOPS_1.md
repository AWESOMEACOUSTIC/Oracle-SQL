# Java Competitive Exam Preparation
## Chunk 2: Access Modifiers, Methods, Encapsulation, Overloading, Static, and Constructors

This chunk covers the foundations of object oriented programming in Java. Rather than treating each topic in isolation, every concept here is introduced through one continuous, realistic scenario: building a `BankAccount` class step by step. Each new topic adds one more capability to this same class, so by the end you will have watched a complete, properly encapsulated, well designed Java class emerge naturally from the concepts themselves. This mirrors how these ideas actually get used together in real code, and it keeps the connections between topics visible rather than treating them as disconnected facts to memorize.

**How this document is organized.** Each section builds directly on the class from the previous section. Read them in order. A single quiz near the end tests the whole scenario together, since by that point the concepts are meant to be understood as one coherent design rather than separate topics. A programming practice set with full solutions follows, and a compact revision table closes the chunk.

---

## Table of Contents

1. Access Modifiers
2. What is a Method and How to Declare Methods
3. Encapsulation with Lend A Hand
4. Returning Values from Methods with Lend A Hand
5. Method Overloading with Lend A Hand
6. Static Keyword and Static Methods with Lend A Hand
7. Constructor Overloading
8. `this` Reference and Constructor Chaining
9. Consolidated Quiz: The Full BankAccount Scenario
10. Programming Practice and Solutions
11. Revision Summary

---

## 1. Access Modifiers

### The Scenario Begins

Imagine you are building a small banking application. The very first design decision you face is: which parts of a `BankAccount` should the rest of the program be allowed to touch directly, and which parts should be protected from outside interference? This is exactly the problem access modifiers solve.

### Concept Explanation

An access modifier controls where in a program a class, field, method, or constructor can be referenced from. Java has four levels of access, and importantly, one of them, called default or package private access, has no keyword at all; it is simply what you get when no modifier is written.

| Modifier | Same class | Same package | Subclass in different package | Different package, not a subclass |
|---|---|---|---|---|
| `private` | Yes | No | No | No |
| default (no keyword) | Yes | Yes | No | No |
| `protected` | Yes | Yes | Yes | No |
| `public` | Yes | Yes | Yes | Yes |

`private` is the most restrictive: only code inside the exact same class can access a `private` member. Default access, used when you write no modifier at all, opens access to any class in the same package but nothing outside it. `protected` extends default access to also include subclasses even if they live in a different package. `public` is the least restrictive: accessible from anywhere.

**Applying this to `BankAccount`.** The fields that hold sensitive state, the account holder's name, the balance, and the account number, should be `private`. There is no good reason for arbitrary outside code to reach in and directly overwrite `balance` without going through the account's own rules. The methods that the rest of the program is meant to actually use, like depositing or checking a balance, should be `public`, since they form the class's intended interface to the outside world.

```java
public class BankAccount {
    private String accountHolder;
    private double balance;
    private int accountNumber;
}
```

Marking the class itself `public` means other classes, in any package, can create `BankAccount` objects. Marking the fields `private` means no other class, including classes in the same package, can read or write `accountHolder`, `balance`, or `accountNumber` directly; they will need to go through methods that this class chooses to expose, which is the whole point, and which the Encapsulation section builds out fully.

### Important Notes

- A top level class can only be `public` or default access; it cannot be `private` or `protected`.
- Fields and methods can use all four levels.
- Choosing `private` for fields and `public` for the methods meant to be used externally is the standard, idiomatic starting point for well encapsulated classes, and is exactly the pattern this entire chunk builds toward.
- Default access is easy to apply by accident, simply by forgetting to write a modifier at all; always be deliberate about which level you intend.

### Quick Check

Given the `BankAccount` class above, can a class in a completely different package directly write `account.balance = 1000000;`?

No. `balance` is `private`, so only code inside `BankAccount` itself can reference it directly, regardless of what package the calling code lives in. This is precisely the protection access modifiers are designed to provide.

---

## 2. What is a Method and How to Declare Methods

### Growing the Scenario

A class with only private fields and no way to interact with them is useless. The next step is giving `BankAccount` behavior: actions it can perform, expressed as methods.

### Concept Explanation

A method is a named block of code that performs a specific action, optionally accepts input in the form of parameters, and optionally produces output in the form of a return value. Every method declaration has the same anatomy:

```
accessModifier returnType methodName(parameterList) {
    // method body
}
```

- **Access modifier** controls who can call the method, exactly as covered in the previous section.
- **Return type** states what kind of value the method sends back to its caller, or `void` if it sends nothing back.
- **Method name** follows the same identifier rules as variable names, and by convention starts with a lowercase letter and uses camelCase for multiple words.
- **Parameter list** declares zero or more typed inputs the method needs to do its job, separated by commas if there is more than one; each parameter is a local variable scoped to that method.
- **Method body** is the block of code that actually executes when the method is called.

**Adding a first method to `BankAccount`.**

```java
public class BankAccount {
    private String accountHolder;
    private double balance;
    private int accountNumber;

    public void deposit(double amount) {
        balance = balance + amount;
        System.out.println("Deposited " + amount + ". New balance: " + balance);
    }
}
```

`deposit` is `public`, so external code can call it. It takes one parameter, `amount`, of type `double`. Its return type is `void`, meaning it performs an action but sends no value back to the caller. Inside the body, it directly modifies the private field `balance`, which is exactly the point: external code cannot touch `balance` directly, but it can call `deposit`, and `deposit`, being part of the class itself, is fully permitted to modify `balance` on the object's behalf.

**Calling a method.** Once an object exists, its public methods are invoked using dot notation.

```java
BankAccount acc = new BankAccount();
acc.deposit(500.0);
```

### Important Notes

- A method's parameters are local variables that exist only for the duration of that method call.
- The number, order, and types of arguments supplied at a call site must match some declared method's parameter list; this becomes especially important once Method Overloading is introduced.
- `void` methods perform an action but return nothing; attempting to use a `void` method's call as though it produced a value, such as `int x = someVoidMethod();`, is a compile error.
- A method can be called only through an object reference for instance methods, or through the class name for static methods, a distinction the Static Keyword section covers fully.

### Quick Check

Why can `deposit` modify `balance` directly, when the earlier section established that `balance` is `private`?

Because `private` restricts access to code outside the class, not code inside it. `deposit` is a method defined inside `BankAccount` itself, so it has full access to every field of that class, private or otherwise. This is exactly how encapsulation is meant to work: the class controls its own state internally, while the outside world is limited to whatever public methods the class chooses to expose.

---

## 3. Encapsulation with Lend A Hand

### Growing the Scenario

`deposit` works, but there is still no way for outside code to check the current balance, and nothing is stopping `deposit` from accepting a negative amount, which would make no real world sense. Encapsulation is the principle that formalizes fixing both problems: hide the data, and control access to it entirely through methods that can enforce rules.

### Concept Explanation

Encapsulation means bundling an object's data together with the methods that operate on that data, while restricting direct outside access to the data itself. In practice, in Java, this almost always looks like: private fields, plus public methods, conventionally called getters and setters, that provide controlled read and write access.

- A **getter**, conventionally named `getFieldName()`, returns the current value of a field, with no parameters.
- A **setter**, conventionally named `setFieldName(newValue)`, updates a field's value, typically after validating the incoming value is acceptable.

**Encapsulating `BankAccount` fully.**

```java
public class BankAccount {
    private String accountHolder;
    private double balance;
    private int accountNumber;

    public void deposit(double amount) {
        if (amount <= 0) {
            System.out.println("Deposit amount must be positive.");
            return;
        }
        balance = balance + amount;
    }

    public double getBalance() {
        return balance;
    }

    public String getAccountHolder() {
        return accountHolder;
    }

    public void setAccountHolder(String accountHolder) {
        this.accountHolder = accountHolder;
    }
}
```

`getBalance` is a getter: it exposes the balance for reading, but crucially there is no `setBalance` method at all here, meaning the only way to change `balance` is through `deposit`, which validates the amount first. This is the real payoff of encapsulation: it is not just about hiding data for its own sake, it is about guaranteeing that a field can only ever change through code that enforces the class's own rules. `setAccountHolder` does exist, since there is no similar concern about validating a name, though a real system might still validate that it is not blank.

Notice `setAccountHolder` uses `this.accountHolder`. The parameter `accountHolder` shadows the field of the same name inside this method, exactly as covered in Chunk 1's discussion of shadowing; `this.accountHolder` explicitly reaches the field, while the unqualified `accountHolder` refers to the parameter. The full behavior of `this` is covered in depth in its own section later in this chunk.

### Applied Example

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount();
        acc.setAccountHolder("Priya Nair");
        acc.deposit(1000);
        acc.deposit(-50);
        System.out.println(acc.getAccountHolder() + "'s balance: " + acc.getBalance());
    }
}
```

Output:
```
Deposit amount must be positive.
Priya Nair's balance: 1000.0
```

The negative deposit attempt is rejected by `deposit`'s own validation logic, and the balance remains unaffected, demonstrating exactly why keeping `balance` private and routing every change through validated methods matters.

### Lend A Hand: Quiz

1. Why does `BankAccount` deliberately not provide a `setBalance` method?
   A. Because `double` cannot be a setter's parameter type  
   B. Because the only intended way to change the balance is through validated methods like `deposit`  
   C. Because getters and setters must always come in pairs  
   D. Because `balance` is `public`

2. What would happen if `balance` were instead declared `public`?
   A. Nothing changes  B. Outside code could set it to any value directly, bypassing the deposit validation entirely  C. `deposit` would stop compiling  D. The class would no longer compile

3. In `setAccountHolder(String accountHolder)`, what does the unqualified `accountHolder` inside the method body refer to?
   A. The field  B. The parameter, since it shadows the field  C. Both simultaneously  D. Compile error due to ambiguity

### Answers

1. **B.** Encapsulation's real purpose is enforcing rules around how state changes, not merely hiding data. By exposing only `deposit`, which validates its input, `BankAccount` guarantees `balance` can never become invalid through some uncontrolled direct write.

2. **B.** A `public` field can be read and written by any code anywhere with no validation whatsoever, completely defeating the purpose of the checks written inside `deposit`; someone could simply write `acc.balance = -9999;` directly.

3. **B.** The parameter shadows the field within the method body, so the unqualified name resolves to the nearer, shadowing parameter; `this.accountHolder` is required to reach the field itself.

---

## 4. Returning Values from Methods with Lend A Hand

### Growing the Scenario

`getBalance` already returns a value. Now consider withdrawing money: this needs to both change the state and tell the caller whether the withdrawal actually succeeded, since a withdrawal larger than the current balance should not be allowed.

### Concept Explanation

A method's return type declares what kind of value, if any, it sends back to its caller. A non `void` method must use `return expression;`, and the expression's type must match, or be convertible to, the declared return type. As covered in Chunk 1, every possible execution path through a non `void` method must reach a `return`, or the code fails to compile, and a method may contain multiple `return` statements along different branches.

**Adding a `withdraw` method that returns success or failure.**

```java
public boolean withdraw(double amount) {
    if (amount <= 0) {
        return false;
    }
    if (amount > balance) {
        return false;
    }
    balance = balance - amount;
    return true;
}
```

`withdraw` returns a `boolean`, letting the caller check whether the withdrawal actually happened, rather than blindly assuming it did. This is a significant improvement over a `void` method, which would leave the caller with no way to know if something went wrong.

### Applied Example

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount();
        acc.deposit(200);

        boolean result1 = acc.withdraw(50);
        boolean result2 = acc.withdraw(1000);

        System.out.println("First withdrawal succeeded: " + result1);
        System.out.println("Second withdrawal succeeded: " + result2);
        System.out.println("Final balance: " + acc.getBalance());
    }
}
```

Output:
```
First withdrawal succeeded: true
Second withdrawal succeeded: false
Final balance: 150.0
```

The first withdrawal is within the balance, so it succeeds, reduces `balance`, and returns `true`. The second exceeds the remaining balance, so `withdraw` returns `false` immediately without touching `balance` at all, exactly as the guard clause intends.

### Lend A Hand: Quiz

1. Why is `boolean` a better return type than `void` for `withdraw`?
   A. `boolean` methods run faster  B. It lets the caller know whether the withdrawal actually happened  C. `void` methods cannot contain `if` statements  D. There is no real difference

2. Given the `withdraw` code above, what happens if `amount` equals exactly `balance`?
   A. Returns `false`, since `amount > balance` is `false` here, so the withdrawal proceeds and succeeds  B. Always returns `false`  C. Throws an exception  D. Compile error

3. If a third guard clause were added to reject withdrawals above `10000` but the method's final `return true;` were accidentally deleted, what would happen?
   A. Compiles fine, just never confirms success  B. Compile error, missing return statement on the path where all checks pass  C. Runtime exception  D. Defaults to returning `false`

### Answers

1. **B.** A `void` version could silently do nothing when the amount is invalid, leaving the caller with no way to distinguish a successful withdrawal from a silently rejected one; returning `boolean` makes that outcome explicit and checkable.

2. **A.** `amount > balance` is `false` when they are exactly equal, so neither guard clause triggers, the withdrawal proceeds, and the method reaches `balance = balance - amount; return true;`, correctly succeeding and leaving `balance` at exactly `0`.

3. **B.** Without a final `return true;`, the path where both guard clauses fail to trigger reaches the end of the method with no return statement, which Java's compiler rejects as a missing return statement error, exactly as covered in Chunk 1.

---

## 5. Method Overloading with Lend A Hand

### Growing the Scenario

Sometimes a deposit comes with a note, like "Salary" or "Gift", and sometimes it does not. Rather than inventing two differently named methods, Java lets you give two methods the same name as long as their parameter lists genuinely differ.

### Concept Explanation

Method overloading means defining multiple methods in the same class that share the same name but differ in their parameter list, either in the number of parameters, the types of parameters, or the order of parameter types. The compiler decides which overloaded version to call based on the arguments supplied at each call site, matched against each available signature.

**What does NOT count as a valid overload.** Changing only the return type, while keeping the exact same parameter list, is not a valid overload; it is actually a compile error, since the compiler cannot distinguish which method a given call site intends purely by looking at the return type, especially when the return value is not used at all. Changing only a parameter's name, with the type and order unchanged, is also not a valid overload, since parameter names play no role in matching a call site to a method signature.

**Adding an overloaded `deposit` to `BankAccount`.**

```java
public void deposit(double amount) {
    if (amount <= 0) {
        System.out.println("Deposit amount must be positive.");
        return;
    }
    balance = balance + amount;
}

public void deposit(double amount, String note) {
    deposit(amount);
    System.out.println("Note: " + note);
}
```

The second `deposit` has a different parameter list, an added `String note`, so this is a valid overload. Its body calls the first version, `deposit(amount)`, to reuse the existing validation and balance update logic, rather than duplicating that logic, and then adds its own extra behavior on top.

### Applied Example

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount();
        acc.deposit(500);
        acc.deposit(1000, "Monthly salary");
        System.out.println("Balance: " + acc.getBalance());
    }
}
```

Output:
```
Note: Monthly salary
Balance: 1500.0
```

The compiler chooses which `deposit` to call based purely on how many arguments are supplied at each call site: one argument matches the first signature, two arguments matches the second.

### Lend A Hand: Quiz

1. Which of these would be a valid overload of `public void deposit(double amount)`?
   A. `public void deposit(double amount)` with a different body only  B. `public int deposit(double amount)`  C. `public void deposit(int amount)`  D. `public void deposit(double sum)`

2. Why is option B in question 1 invalid?
   A. `int` is not a valid return type  B. Changing only the return type with an identical parameter list is not a valid overload  C. It would work fine  D. Methods cannot return `int`

3. Given both `deposit` overloads shown above, what happens with `acc.deposit(500, "Gift");`?
   A. Compile error, no matching overload  B. Calls the two parameter version, which internally reuses the one parameter version  C. Calls the one parameter version twice  D. Runtime exception

### Answers

1. **C.** `deposit(int amount)` has a genuinely different parameter type, `int` instead of `double`, which is sufficient to make it a distinct, valid overload. Option A is an identical signature, which is simply a duplicate method and a compile error. Option B changes only the return type, which is invalid. Option D changes only the parameter's name, which the compiler ignores entirely for overload resolution.

2. **B.** The compiler distinguishes overloaded methods purely by their parameter lists, not by return type, so a return type only change on an otherwise identical signature is ambiguous and rejected at compile time.

3. **B.** Two arguments, a `double` and a `String`, exactly match the `deposit(double amount, String note)` signature, which internally calls the single parameter `deposit(amount)` to perform the actual validation and balance update.

---

## 6. Static Keyword and Static Methods with Lend A Hand

### Growing the Scenario

The bank wants to track how many `BankAccount` objects have been created in total, across the entire program, not per individual account. This is precisely the kind of shared, class wide state that `static` exists for.

### Concept Explanation

A `static` member belongs to the class itself rather than to any individual object. Exactly one copy of a static field exists, shared by every instance of the class, and it can be accessed even with zero objects created. A static method can be called directly through the class name, without needing any object at all, and, critically, a static method cannot directly access instance fields or instance methods, since it has no particular object to operate on.

**Adding a static counter to `BankAccount`.**

```java
public class BankAccount {
    private static int totalAccountsCreated = 0;

    private String accountHolder;
    private double balance;
    private int accountNumber;

    public BankAccount() {
        totalAccountsCreated++;
    }

    public static int getTotalAccountsCreated() {
        return totalAccountsCreated;
    }
}
```

`totalAccountsCreated` is `static`, so it is shared across every `BankAccount` object rather than each object getting its own separate copy. Every time the constructor runs, whichever specific object is being built increments this one shared counter. `getTotalAccountsCreated` is also `static`, so it can be called as `BankAccount.getTotalAccountsCreated()`, directly through the class name, with no need to have any particular account object on hand first.

### Applied Example

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount a = new BankAccount();
        BankAccount b = new BankAccount();
        BankAccount c = new BankAccount();
        System.out.println("Total accounts created: " + BankAccount.getTotalAccountsCreated());
    }
}
```

Output: `Total accounts created: 3`

Each construction increments the one shared static field, so by the time all three objects exist, the count correctly reflects the total, regardless of which specific object's constructor happened to run.

### Lend A Hand: Quiz

1. Why must `totalAccountsCreated` be `static` rather than an ordinary instance field for this purpose?
   A. Static fields are faster  B. An instance field would give each `BankAccount` its own separate counter starting at 0, unable to track a shared total  C. Instance fields cannot be `int`  D. There is no real difference

2. Could `getTotalAccountsCreated` directly access `balance`, an instance field, inside its body?
   A. Yes, always  B. No, a static method has no implicit object to access an instance field through  C. Only if `balance` is also static  D. Only inside `main`

3. What is the correct way to call `getTotalAccountsCreated` from outside the class?
   A. `new BankAccount().getTotalAccountsCreated()` only  B. `BankAccount.getTotalAccountsCreated()`  C. It cannot be called from outside  D. `accounts.getTotalAccountsCreated()` where `accounts` is undeclared

### Answers

1. **B.** If `totalAccountsCreated` were an instance field, every single `BankAccount` object would carry its own independent copy, always starting at `0` and incremented only by that one object's own constructor call, making it structurally impossible to track a running total shared across every account ever created.

2. **B.** A static method belongs to the class, not to any particular object, so it has no implicit `this` and cannot reach an instance field directly; it would need an explicit object reference passed in or available some other way to do so.

3. **B.** Static methods are conventionally, and most clearly, called through the class name itself, `BankAccount.getTotalAccountsCreated()`, though calling it through an object reference is technically also legal Java, if considered poor style since it obscures the fact that the method is static.

---

## 7. Constructor Overloading

### Growing the Scenario

Right now, creating a `BankAccount` leaves every field at its default value; there is no way to supply an account holder's name or a starting balance at the moment of creation. Constructors solve this, and just like methods, constructors can be overloaded to support several different ways of creating an account.

### Concept Explanation

A constructor is a special block of code that runs automatically when an object is created with `new`, responsible for initializing that object's state. A constructor shares its name exactly with the class, has no return type at all, not even `void`, and can be overloaded exactly like a method, meaning a class can define several constructors as long as their parameter lists differ.

**Giving `BankAccount` multiple constructors.**

```java
public BankAccount() {
    accountNumber = generateAccountNumber();
    balance = 0.0;
}

public BankAccount(String accountHolder) {
    accountNumber = generateAccountNumber();
    this.accountHolder = accountHolder;
    balance = 0.0;
}

public BankAccount(String accountHolder, double initialBalance) {
    accountNumber = generateAccountNumber();
    this.accountHolder = accountHolder;
    balance = initialBalance;
}
```

Three constructors exist here, distinguished entirely by their parameter lists: no arguments, a name only, or a name plus a starting balance. When `new BankAccount("Arjun")` is written, the compiler matches this call against the available constructor signatures and selects the one taking a single `String`.

### Important Notes

- A class with no explicitly written constructor at all automatically receives a compiler provided no argument default constructor; as soon as you write even one constructor yourself, that automatic default constructor no longer exists unless you write it yourself too.
- Constructors are matched to `new` calls using the exact same overload resolution rules covered in the Method Overloading section: number, type, and order of arguments.
- Constructors cannot be `static`, `final`, or `abstract`, since they are fundamentally tied to the specific act of building one particular object.

### Quick Check

Given the three constructors above, which one runs for `new BankAccount("Meera", 5000.0);`?

The third constructor, `BankAccount(String accountHolder, double initialBalance)`, since its parameter list, one `String` followed by one `double`, exactly matches the two arguments supplied at this particular call site.

---

## 8. `this` Reference and Constructor Chaining

### Completing the Scenario

The three constructors above have a problem: `accountNumber = generateAccountNumber();` is repeated identically in every single one. This kind of duplication is exactly what `this()` constructor chaining exists to eliminate.

### Concept Explanation

`this` is a reference to the current object, the specific object whose method or constructor is currently executing. It has two closely related uses that have both already appeared earlier in this chunk in passing, and are now covered fully.

**`this` for disambiguating shadowed fields.** As seen in the Encapsulation section's `setAccountHolder` method, when a parameter or local variable shares a name with a field, `this.fieldName` explicitly refers to the field, resolving what would otherwise be ambiguous or, without `this`, would simply resolve to the shadowing parameter instead.

**`this(...)` for constructor chaining.** Within one constructor, the very first statement can be a call to another constructor of the same class, written as `this(arguments)`, rather than `new`. This lets a more specific constructor delegate its shared setup work to a more general one, avoiding duplicated initialization code entirely. `this(...)`, if used at all, must be the first statement in the constructor, and a constructor cannot call itself in a way that creates an infinite chain.

**Refactoring `BankAccount`'s constructors to chain.**

```java
public BankAccount() {
    this("Unknown", 0.0);
}

public BankAccount(String accountHolder) {
    this(accountHolder, 0.0);
}

public BankAccount(String accountHolder, double initialBalance) {
    this.accountNumber = generateAccountNumber();
    this.accountHolder = accountHolder;
    this.balance = initialBalance;
}
```

Now `generateAccountNumber()` and the actual field assignments exist in exactly one place, the three parameter constructor. The other two constructors simply delegate to it with sensible default values, via `this(...)`, rather than repeating the same three lines of setup logic three separate times.

### Applied Example

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount a = new BankAccount();
        BankAccount b = new BankAccount("Karthik");
        BankAccount c = new BankAccount("Divya", 2500.0);

        System.out.println(a.getAccountHolder() + ": " + a.getBalance());
        System.out.println(b.getAccountHolder() + ": " + b.getBalance());
        System.out.println(c.getAccountHolder() + ": " + c.getBalance());
    }
}
```

Output:
```
Unknown: 0.0
Karthik: 0.0
Divya: 2500.0
```

Each constructor, regardless of which one is directly invoked by `new`, ultimately funnels through the same three parameter constructor via chaining, guaranteeing every `BankAccount` object is initialized consistently through exactly one piece of real initialization logic.

### Important Notes

- `this.field = parameter;` resolves a shadowed field; the unqualified name alone would refer to the parameter instead.
- `this(...)` must be the very first statement inside a constructor if used at all; it cannot appear anywhere else.
- Chaining constructors with `this(...)` avoids duplicating shared initialization logic across multiple overloaded constructors, exactly mirroring how the Method Overloading section's two parameter `deposit` reused the one parameter `deposit` rather than duplicating its logic.
- A constructor cannot chain to itself, directly or indirectly through a cycle of `this(...)` calls, since that would create unresolvable infinite recursion; the compiler detects and rejects this.

### Quick Check

Why must `this(...)` be the first statement in a constructor?

Because the chained constructor is responsible for a portion of the object's initialization, and Java requires that delegation happen before any other code in the calling constructor runs, ensuring the object is never left in a partially, inconsistently initialized state partway through construction.

---

## 9. Consolidated Quiz: The Full BankAccount Scenario

This quiz treats `BankAccount`, in its final complete form with all sections applied, as one whole design, testing whether you can reason about access modifiers, methods, encapsulation, return values, overloading, static members, and constructors together, exactly as they interact in the finished class.

**Reference: the complete class.**

```java
public class BankAccount {
    private static int totalAccountsCreated = 0;

    private String accountHolder;
    private double balance;
    private int accountNumber;

    public BankAccount() {
        this("Unknown", 0.0);
    }

    public BankAccount(String accountHolder) {
        this(accountHolder, 0.0);
    }

    public BankAccount(String accountHolder, double initialBalance) {
        this.accountNumber = totalAccountsCreated + 1;
        this.accountHolder = accountHolder;
        this.balance = initialBalance;
        totalAccountsCreated++;
    }

    public void deposit(double amount) {
        if (amount <= 0) {
            return;
        }
        balance += amount;
    }

    public void deposit(double amount, String note) {
        deposit(amount);
        System.out.println("Note: " + note);
    }

    public boolean withdraw(double amount) {
        if (amount <= 0 || amount > balance) {
            return false;
        }
        balance -= amount;
        return true;
    }

    public double getBalance() {
        return balance;
    }

    public String getAccountHolder() {
        return accountHolder;
    }

    public static int getTotalAccountsCreated() {
        return totalAccountsCreated;
    }
}
```

### Quiz

1. Why are `balance`, `accountHolder`, and `accountNumber` all declared `private`?
   A. Java requires all fields to be private  B. To ensure they can only be changed through the class's own validated methods, the core idea of encapsulation  C. Because they are all numeric  D. To make the class compile faster

2. Which constructor actually performs the real field assignments?
   A. The no argument constructor  B. The single `String` constructor  C. The two parameter constructor, `BankAccount(String, double)`  D. All three equally

3. What is the return type of `withdraw`, and why is that choice meaningful?
   A. `void`; it does not matter  B. `boolean`; it lets the caller know whether the withdrawal actually succeeded  C. `double`; it returns the new balance  D. `int`; it returns an error code

4. Why does `deposit(double amount, String note)` call `deposit(amount)` rather than repeating the balance update logic itself?
   A. It is required syntax  B. To avoid duplicating the validation and update logic, reusing the existing single parameter version instead  C. Because `note` cannot be used otherwise  D. There is no real reason

5. Why is `totalAccountsCreated` declared `static`?
   A. So each object gets its own independent copy  B. So exactly one shared copy exists and tracks the count across every object ever created  C. Because `int` fields must be static  D. To make it private

6. Can `getTotalAccountsCreated` access `balance` directly?
   A. Yes  B. No, it is static and has no implicit object to access an instance field through  C. Only for the first created object  D. Only if `balance` is also static

7. What does `this("Unknown", 0.0);` inside the no argument constructor do?
   A. Creates a brand new, separate `BankAccount` object  B. Delegates to the two parameter constructor to perform the actual initialization with default values  C. Calls a static method  D. Causes a compile error

8. Given `BankAccount acc = new BankAccount("Zara", 100);` followed by `acc.deposit(-5);`, what is `acc.getBalance()` afterward?
   A. `95.0`  B. `100.0`  C. `-5.0`  D. Compile error

9. Given the class above, is `public void deposit(double amount)` and `public void deposit(double amount, String note)` a valid pair of overloads?
   A. No, same name is never allowed twice  B. Yes, they have different parameter lists  C. No, because both return `void`  D. Only if one is `private`

10. If external code writes `acc.balance = 99999;` directly, what happens?
    A. Compiles and works fine  B. Compile time error, since `balance` is `private`  C. Runtime exception  D. Silently ignored

11. What would happen if the constructor chaining were accidentally written so that `BankAccount()` called `this("Unknown", 0.0);` and the two parameter constructor also somehow called back to `BankAccount()`?
    A. Works fine  B. Compile error, since this would form an unresolvable constructor call cycle  C. Infinite object creation at runtime  D. Silently uses default values

12. Why is `this.accountHolder = accountHolder;` necessary inside the two parameter constructor, rather than just `accountHolder = accountHolder;`?
    A. It is not necessary, both work identically  B. Without `this`, the unqualified name refers to the shadowing parameter, so the field would never actually be set from the parameter's value  C. `this` makes the code run faster  D. Java requires `this` inside every constructor

### Consolidated Quiz Answers

1. **B.** This is the core idea of encapsulation, developed fully in Section 3: hiding fields entirely behind methods guarantees that every change to an object's state passes through logic the class itself controls and can validate.

2. **C.** As shown in Section 8, the two parameter constructor is the only one containing actual field assignments and the account number generation logic; the other two constructors exist purely to supply sensible defaults and delegate via `this(...)`.

3. **B.** As shown in Section 4, returning `boolean` lets the caller distinguish a successful withdrawal from a rejected one, which a `void` return type could never communicate.

4. **B.** As shown in Section 5, reusing `deposit(amount)` from within `deposit(amount, note)` avoids duplicating the validation and balance update logic in two places, which would risk the two copies drifting out of sync if one were ever updated without the other.

5. **B.** As shown in Section 6, `static` is precisely what allows exactly one shared copy to exist across every object, rather than each object tracking its own independent, and therefore useless for this purpose, count.

6. **B.** As shown in Section 6, a static method has no implicit current object, so it cannot reach any instance field like `balance` directly without first being given some specific object reference to read it from.

7. **B.** As shown in Section 8, `this(...)` calls another constructor of the same class, here delegating the no argument case to the fully general two parameter constructor with chosen default values, rather than creating any new, separate object.

8. **B, `100.0`.** `deposit`'s guard clause `if (amount <= 0) { return; }` rejects the negative amount immediately, leaving `balance` completely unchanged from its initial value of `100.0`, exactly mirroring the validation behavior demonstrated back in Section 3.

9. **B.** The two methods share a name but have genuinely different parameter lists, one parameter versus two, which is precisely what makes this a valid overload pair, as covered fully in Section 5.

10. **B.** `balance` is `private`, so any code outside `BankAccount` itself, regardless of package, cannot reference it directly at all; this is rejected by the compiler before the program can even run, exactly as covered in Section 1.

11. **B.** A cycle of `this(...)` calls with no constructor ever reaching actual, non delegating initialization logic is unresolvable, and the Java compiler detects and rejects this specific situation as a compile time error, as noted in Section 8.

12. **B.** The parameter `accountHolder` shadows the field of the same name within the constructor body; without `this`, the unqualified assignment `accountHolder = accountHolder;` would simply assign the parameter to itself, leaving the actual field at its default, uninitialized value, exactly as covered in Section 3's discussion of `setAccountHolder`.

---

## 10. Programming Practice and Solutions

### Practice Problems

1. **Basic.** Add a `getAccountNumber()` getter to `BankAccount` and demonstrate it by printing the account number of a newly created account.
2. **Intermediate.** Add a new overloaded constructor `BankAccount(String accountHolder, double initialBalance, boolean isPremium)` that also sets a new private `boolean isPremium` field, chaining to the existing two parameter constructor via `this(...)` to avoid duplicating the account number and field setup logic.
3. **Intermediate.** Add a `transferTo(BankAccount other, double amount)` method that withdraws `amount` from the current account and, only if that withdrawal succeeds, deposits it into `other`, returning a `boolean` indicating whether the entire transfer succeeded.
4. **Advanced.** Add a static field `interestRate` and a static method `applyInterestToAll` is not practical without tracking every object, so instead add an instance method `applyInterest()` that increases `balance` by `balance * interestRate`, where `interestRate` is a shared static value settable through a static setter method, and demonstrate that changing the static rate affects every account's interest calculation.
5. **Edge case based.** Modify `withdraw` to also handle the case where `amount` is exactly `0`, deciding and clearly commenting on whether this should count as success or failure, and write a short test demonstrating your chosen behavior.

### Solutions

**Solution 1.**

```java
public class BankAccount {
    // ... existing fields and methods ...

    public int getAccountNumber() {
        return accountNumber;
    }

    public static void main(String[] args) {
        BankAccount acc = new BankAccount("Rohan", 300);
        System.out.println("Account number: " + acc.getAccountNumber());
    }
}
```

Why it works: this is a straightforward getter, following the exact same pattern as `getBalance` and `getAccountHolder` from Section 3, exposing a private field for reading without allowing it to be modified from outside the class at all, since no corresponding setter is provided.

**Solution 2.**

```java
private boolean isPremium;

public BankAccount(String accountHolder, double initialBalance, boolean isPremium) {
    this(accountHolder, initialBalance);
    this.isPremium = isPremium;
}
```

Why it works: chaining to the existing two parameter constructor via `this(accountHolder, initialBalance);` reuses the account number generation and field setup logic that already exists there, and the new constructor only needs to add the one additional line handling the new `isPremium` field, exactly following the same duplication avoiding pattern established in Section 8.

**Solution 3.**

```java
public boolean transferTo(BankAccount other, double amount) {
    boolean withdrawn = this.withdraw(amount);
    if (!withdrawn) {
        return false;
    }
    other.deposit(amount);
    return true;
}
```

Why it works: the method first attempts the withdrawal from the current account, reusing the existing, already validated `withdraw` method rather than duplicating its balance and amount checks. Only if that withdrawal genuinely succeeds does it proceed to deposit into the other account, and the overall `boolean` return value correctly reflects whether the complete transfer actually happened, connecting directly back to the return value reasoning covered in Section 4.

**Solution 4.**

```java
private static double interestRate = 0.02;

public static void setInterestRate(double rate) {
    interestRate = rate;
}

public void applyInterest() {
    balance += balance * interestRate;
}
```

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount a = new BankAccount("Kiran", 1000);
        BankAccount b = new BankAccount("Neha", 2000);

        BankAccount.setInterestRate(0.05);
        a.applyInterest();
        b.applyInterest();

        System.out.println(a.getBalance());
        System.out.println(b.getBalance());
    }
}
```

Expected output:
```
1050.0
2100.0
```

Why it works: since `interestRate` is `static`, changing it once through `BankAccount.setInterestRate(0.05);` affects the calculation for every single account, since all instance method calls to `applyInterest` read from that one shared field, directly demonstrating the shared, class wide nature of static state covered in Section 6.

**Solution 5.**

```java
public boolean withdraw(double amount) {
    if (amount < 0 || amount > balance) {
        return false;
    }
    // A withdrawal of exactly 0 is treated as a trivial success: it changes
    // nothing, but there is no real reason to reject it as invalid, unlike a
    // genuinely negative amount, which makes no real world sense at all.
    balance -= amount;
    return true;
}
```

```java
public class Demo {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount("Test", 500);
        System.out.println(acc.withdraw(0));
        System.out.println(acc.getBalance());
    }
}
```

Expected output:
```
true
500.0
```

Why it works: changing the guard condition from `amount <= 0` to `amount < 0` specifically allows exactly `0` through as a valid, trivially successful withdrawal, while still correctly rejecting any genuinely negative amount, and the balance is correctly left unchanged since subtracting `0` has no real effect.

---

## 11. Revision Summary

| Concept | Key Rule | Where It Appeared in the Scenario |
|---|---|---|
| Access modifiers | `private` for internal state, `public` for the intended external interface | `balance`, `accountHolder` private; `deposit`, `withdraw` public |
| Method anatomy | modifier, return type, name, parameters, body | `deposit(double amount)` |
| Encapsulation | private fields plus controlled public methods enforce validation | `deposit` rejecting non positive amounts |
| Return values | non void methods must return on every path; `boolean` communicates success or failure | `withdraw` returning `true`/`false` |
| Method overloading | same name, different parameter list; return type alone never distinguishes overloads | `deposit(double)` vs `deposit(double, String)` |
| Static | one shared copy per class, not per object; static methods cannot touch instance members directly | `totalAccountsCreated`, `getTotalAccountsCreated()` |
| Constructor overloading | same rules as method overloading, applied to constructors; no return type at all | three `BankAccount` constructors |
| `this` and chaining | `this.field` resolves shadowing; `this(...)` delegates to another constructor and must be the first statement | `setAccountHolder`, chained `BankAccount` constructors |

**Exam tip.** When a question presents an unfamiliar class, mentally reconstruct it the same way this chunk built `BankAccount`: identify which fields are private and why, trace which constructor actually performs real initialization versus which ones merely delegate, and check whether each method's return type is doing meaningful work for its caller. These four checks resolve the large majority of object oriented design questions on a competitive exam.