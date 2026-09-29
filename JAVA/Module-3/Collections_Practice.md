# Java Collections: Assessment Practice Set

**Focus:** `HashMap` · `HashSet` · `TreeMap` · `ArrayList`
**Style:** company scenario → requirements → given driver code → samples → answer → reasoning

## How to use this set

* Each question mirrors the assessment format you described: the **driver code is given** and you write only the classes or methods named in the requirements. The driver is shown in each question so you can run it locally.
* **Attempt each question before scrolling to its answer.** Samples cover the normal path; the hidden test cases in an assessment will hit the edge cases listed in each *Reasoning* section (empty input, ties, case sensitivity, boundaries, `null` returns).
* Every reference solution was compiled and run against the sample inputs, and the sample outputs in this file are the real program output.
* Java's natural `String` ordering compares character codes, so **uppercase letters sort before lowercase** (`"Z" < "a"`).
* Difficulty rises inside each part. If you are short on time, do the **Medium** and above.

## Contents

| # | Question | Collection | Level |
|---|---|---|---|
| 1 | [CityMart Stock Tracker](#q1-citymart-stock-tracker) | HashMap | Easy |
| 2 | [QuickRead Keyword Analysis](#q2-quickread-keyword-analysis) | HashMap | Easy |
| 3 | [GrandStay Booking Cancellations](#q3-grandstay-booking-cancellations) | HashMap | Easy-Medium |
| 4 | [CityCourier Customer Registry](#q4-citycourier-customer-registry) | HashMap (with ArrayList values) | Medium |
| 5 | [CipherWorks Message Analysis](#q5-cipherworks-message-analysis) | HashMap + ArrayList (sorting entries) | Medium |
| 6 | [PayTrack Settlement Pair](#q6-paytrack-settlement-pair) | HashMap (lookup by complement) | Medium-Hard |
| 7 | [TechNova Training Batches](#q7-technova-training-batches) | HashSet (set operations) | Easy-Medium |
| 8 | [SafeBank Duplicate Transactions](#q8-safebank-duplicate-transactions) | HashSet (add() return value) | Medium |
| 9 | [AirNest Passenger Check-in](#q9-airnest-passenger-check-in) | HashSet (custom equals/hashCode) | Medium-Hard |
| 10 | [Bharat Savings Interest Slabs](#q10-bharat-savings-interest-slabs) | TreeMap (floorEntry / tailMap) | Medium |
| 11 | [BrainBowl Quiz Leaderboard](#q11-brainbowl-quiz-leaderboard) | TreeMap (case-insensitive comparator, subMap) | Medium-Hard |
| 12 | [Sunrise Retail Regional Sales](#q12-sunrise-retail-regional-sales) | TreeMap (aggregation + parsing) | Hard |
| 13 | [SkyWays Baggage Screening](#q13-skyways-baggage-screening) | ArrayList (safe removal) | Medium |
| 14 | [MetroCab Ride ID Consolidation](#q14-metrocab-ride-id-consolidation) | ArrayList (two-pointer merge, subList) | Medium |
| 15 | [Prime Infotech Payroll Ranking](#q15-prime-infotech-payroll-ranking) | ArrayList (objects + Comparator) | Medium-Hard |

---

# Part A: HashMap

## Q1. CityMart Stock Tracker

**Collection:** HashMap  ·  **Level:** Easy

CityMart Stores, a supermarket chain headquartered in Bengaluru, wants to keep track of the stock available for each product code across its warehouse deliveries. Develop a Java program based on the following requirements.

**Requirement 1**
Create a `StockManager` class containing a `HashMap` named `stockMap`.
Implement:

```java
public void addStock(String productCode, int quantity)
```

The map should use `productCode -> quantity`.
If the `productCode` is already present, the new quantity must be **added to the existing quantity**. Otherwise a new entry is created.

Constraint: `productCode` is case-sensitive.

**Requirement 2**
Implement:

```java
public List<String> findLowStockProducts(int threshold)
```

The method should return the product codes whose quantity is **strictly less than** `threshold`, in ascending (natural) order.
If no such product exists, the main method should display:

```
No low stock products found for <threshold>
```

**Restrictions**

* Edit only the `StockManager` class.
* Attributes must be `private`.
* Constructor and methods must be `public`.
* Do not change the specified class, attribute, or method names.
* Do not use `System.exit(0)`.

**Input Format**
First line: integer `n` (number of deliveries). Next `n` lines: `productCode quantity`. Last line: the `threshold`.

**Output Format**
Each product code on a separate line, or the "No low stock..." message.

**Given driver code** (already provided; do not modify)

```java
// Main.java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        StockManager sm = new StockManager();
        int n = sc.nextInt();
        for (int i = 0; i < n; i++) {
            String code = sc.next();
            int qty = sc.nextInt();
            sm.addStock(code, qty);
        }
        int threshold = sc.nextInt();
        List<String> result = sm.findLowStockProducts(threshold);
        if (result.isEmpty()) {
            System.out.println("No low stock products found for " + threshold);
        } else {
            for (String s : result) {
                System.out.println(s);
            }
        }
    }
}
```

**Sample Input 1**

```text
6
P101 20
P205 5
P101 10
p205 3
P330 50
P205 4
25
```

**Sample Output 1**

```text
P205
p205
```

**Sample Input 2**

```text
6
P101 20
P205 5
P101 10
p205 3
P330 50
P205 4
2
```

**Sample Output 2**

```text
No low stock products found for 2
```

### Answer

```java
// StockManager.java
import java.util.*;

public class StockManager {
    private HashMap<String, Integer> stockMap;

    public StockManager() {
        this.stockMap = new HashMap<>();
    }

    public void addStock(String productCode, int quantity) {
        stockMap.merge(productCode, quantity, Integer::sum);
    }

    public List<String> findLowStockProducts(int threshold) {
        List<String> result = new ArrayList<>();
        for (Map.Entry<String, Integer> entry : stockMap.entrySet()) {
            if (entry.getValue() < threshold) {
                result.add(entry.getKey());
            }
        }
        Collections.sort(result);
        return result;
    }
}
```

### Reasoning

* **`merge(key, value, Integer::sum)`** does "insert if absent, otherwise combine" in one call. The long-hand equivalent is `stockMap.put(code, stockMap.getOrDefault(code, 0) + quantity)`. Forgetting the default (`stockMap.get(code) + quantity`) throws a `NullPointerException` the first time a code is seen.
* **Case sensitivity is free**: `HashMap` uses `equals()`, so `"P205"` and `"p205"` are different keys. In the sample, `P205` = 9 and `p205` = 3 are two separate products, and both are below 25.
* **`HashMap` has no ordering**, so the result must be sorted explicitly. `Collections.sort` on `String` uses natural order, where uppercase comes before lowercase (`"P205"` before `"p205"`).
* **Strictly less than** means `<`, not `<=`. A product with quantity exactly equal to the threshold must not appear.
* Return an **empty list, not `null`**, when nothing matches. The driver calls `isEmpty()`, so `null` would crash.
* `entry.getValue() < threshold` auto-unboxes the `Integer`. That is safe here because the values are never null.


---

## Q2. QuickRead Keyword Analysis

**Collection:** HashMap  ·  **Level:** Easy

QuickRead Publications, based in Kolkata, collects one-word keywords from reader feedback forms. The editorial team wants to know which keyword(s) are used most often. Write a Java program to find them.

The `UserMainCode` class should contain a static method named `findMostFrequentWords` that accepts the array of keywords and returns the keyword(s) with the highest frequency.

The `UserInterface` class accepts the number of keywords and the keywords, calls the method, and displays each keyword of the result on a separate line.

**Rules**

* Frequency counting must be **case-insensitive** (`"Java"` and `"java"` are the same keyword).
* The returned keywords must be in **lowercase**.
* If more than one keyword shares the highest frequency, return all of them in ascending order.

Method signature:

```java
public static String[] findMostFrequentWords(String[] words)
```

**Input Format**
An integer `n`, followed by `n` keywords.

**Output Format**
The most frequent keyword(s), one per line.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        String[] words = new String[n];
        for (int i = 0; i < n; i++) {
            words[i] = sc.next();
        }
        String[] result = UserMainCode.findMostFrequentWords(words);
        for (String w : result) {
            System.out.println(w);
        }
    }
}
```

**Sample Input 1**

```text
8
Java java Python PYTHON c Java go C
```

**Sample Output 1**

```text
java
```

**Sample Input 2**

```text
6
apple Banana APPLE banana cherry Cherry
```

**Sample Output 2**

```text
apple
banana
cherry
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static String[] findMostFrequentWords(String[] words) {
        Map<String, Integer> freq = new HashMap<>();
        int max = 0;

        for (String w : words) {
            String key = w.toLowerCase();
            int count = freq.getOrDefault(key, 0) + 1;
            freq.put(key, count);
            if (count > max) {
                max = count;
            }
        }

        List<String> result = new ArrayList<>();
        for (Map.Entry<String, Integer> e : freq.entrySet()) {
            if (e.getValue() == max) {
                result.add(e.getKey());
            }
        }

        Collections.sort(result);
        return result.toArray(new String[0]);
    }
}
```

### Reasoning

* **Normalise before counting**: `toLowerCase()` on the key, not on the final output. Counting first and lowercasing later would keep `"Java"` and `"java"` as separate entries.
* **`getOrDefault(key, 0) + 1`** is the standard frequency-count idiom. `freq.merge(key, 1, Integer::sum)` is the equivalent one-liner.
* **Track `max` while counting** so you need only one extra pass (to collect the words equal to `max`) instead of a separate pass to find the maximum.
* **Ties**: Sample 2 has three words with frequency 2, so all three are returned. A common mistake is returning only the first word found at the maximum.
* **Sort the result**: `HashMap` iteration order is unspecified, and the problem demands ascending order.
* **`e.getValue() == max`** works only because one side is a primitive `int`, which forces unboxing. If both sides were `Integer` objects, `==` would compare references and break for values above 127. Use `.equals()` in that case.
* `toArray(new String[0])` is the idiomatic way to convert a `List<String>` to `String[]`.


---

## Q3. GrandStay Booking Cancellations

**Collection:** HashMap  ·  **Level:** Easy-Medium

GrandStay Hotels, a hotel chain operating in Goa, wants to manage room bookings and handle cancellations efficiently. Develop a Java program based on the following requirements.

**Requirement 1**
Create a `RoomBooking` class containing a `HashMap` named `bookingMap` (`bookingNumber -> guestName`).
Implement:

```java
public void addBooking(String bookingNumber, String guestName)
```

If a booking with the same `bookingNumber` already exists, the **existing booking must be kept** and the new one ignored.
Constraint: `bookingNumber` is case-sensitive.

**Requirement 2**
Implement:

```java
public boolean cancelBooking(String bookingNumber)
```

Remove the booking. Return `true` if a booking was removed, `false` if the booking number does not exist.

**Requirement 3**
Implement:

```java
public int cancelAllByGuest(String guestName)
```

Remove **every** booking made by the given guest and return the number of bookings removed. The guest name comparison must be **case-insensitive**.

**Requirement 4**
Implement:

```java
public List<String> getBookingNumbers()
```

Return the remaining booking numbers in ascending order.

**Restrictions**

* Edit only the `RoomBooking` class.
* Attributes must be `private`.
* Constructor and methods must be `public`.
* Do not change the specified class, attribute, or method names.
* Do not use `System.exit(0)`.

**Input Format**
`n`, then `n` lines of `bookingNumber guestName`, then the booking number to cancel, then the guest name whose bookings must all be cancelled.

**Given driver code** (already provided; do not modify)

```java
// Main.java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        RoomBooking rb = new RoomBooking();
        int n = sc.nextInt();
        for (int i = 0; i < n; i++) {
            String bookingNumber = sc.next();
            String guest = sc.next();
            rb.addBooking(bookingNumber, guest);
        }
        String toCancel = sc.next();
        String guestName = sc.next();

        if (rb.cancelBooking(toCancel)) {
            System.out.println("Booking " + toCancel + " cancelled");
        } else {
            System.out.println("Booking " + toCancel + " not found");
        }

        int removed = rb.cancelAllByGuest(guestName);
        System.out.println("Bookings cancelled for " + guestName + ": " + removed);

        List<String> remaining = rb.getBookingNumbers();
        if (remaining.isEmpty()) {
            System.out.println("No bookings remaining");
        } else {
            System.out.println("Remaining bookings:");
            for (String b : remaining) {
                System.out.println(b);
            }
        }
    }
}
```

**Sample Input 1**

```text
5
B101 Meera
B102 Karthik
B101 Zaheer
B103 meera
B104 Nikhil
B102
MEERA
```

**Sample Output 1**

```text
Booking B102 cancelled
Bookings cancelled for MEERA: 2
Remaining bookings:
B104
```

**Sample Input 2**

```text
2
B201 Ravi
B202 Sita
B999
Zed
```

**Sample Output 2**

```text
Booking B999 not found
Bookings cancelled for Zed: 0
Remaining bookings:
B201
B202
```

**Sample Input 3**

```text
2
B301 Anil
B302 anil
B301
ANIL
```

**Sample Output 3**

```text
Booking B301 cancelled
Bookings cancelled for ANIL: 1
No bookings remaining
```

### Answer

```java
// RoomBooking.java
import java.util.*;

public class RoomBooking {
    private HashMap<String, String> bookingMap;

    public RoomBooking() {
        this.bookingMap = new HashMap<>();
    }

    public void addBooking(String bookingNumber, String guestName) {
        bookingMap.putIfAbsent(bookingNumber, guestName);
    }

    public boolean cancelBooking(String bookingNumber) {
        return bookingMap.remove(bookingNumber) != null;
    }

    public int cancelAllByGuest(String guestName) {
        int before = bookingMap.size();
        bookingMap.values().removeIf(g -> g.equalsIgnoreCase(guestName));
        return before - bookingMap.size();
    }

    public List<String> getBookingNumbers() {
        List<String> numbers = new ArrayList<>(bookingMap.keySet());
        Collections.sort(numbers);
        return numbers;
    }
}
```

### Reasoning

* **`put` vs `putIfAbsent`**: plain `put` overwrites, so `B101` would end up belonging to Zaheer. `putIfAbsent` keeps the first booking, which is what the requirement asks for.
* **`remove(key)` returns the previous value** (or `null` if the key was absent), so `remove(key) != null` doubles as the "did it exist" check without a separate `containsKey` call. This only works because `null` is never stored as a value here. If null values were allowed, use `containsKey` first.
* **Removing many entries safely**: `bookingMap.values().removeIf(...)` removes the matching entries from the map itself, because `values()` is a live view. The tempting alternative is a for-each over `entrySet()` that calls `bookingMap.remove(...)` inside the loop, which throws `ConcurrentModificationException`. The other safe route is an explicit `Iterator` with `it.remove()`.
* **Counting removals**: comparing `size()` before and after avoids writing a counter into the lambda (lambdas cannot modify local `int` variables).
* **Mixed case rules**: booking numbers are compared with `equals` (the map does this), guest names with `equalsIgnoreCase`. Sample 3 shows why `B302 anil` is removed by `ANIL` while `B301` was already removed by number.
* **`keySet()` is a view**. Sorting requires copying it into an `ArrayList` first; you cannot call `Collections.sort` on a `Set`.


---

## Q4. CityCourier Customer Registry

**Collection:** HashMap (with ArrayList values)  ·  **Level:** Medium

CityCourier, a parcel delivery company, wants to group its customers by the city they belong to and quickly find its busiest city. Develop a Java program based on the following requirements.

**Requirement 1**
Create a `CustomerRegistry` class containing a `HashMap` named `cityMap` (`city -> list of customer names`).
Implement:

```java
public void addCustomer(String customerName, String city)
```

Constraints:

* `city` is **case-insensitive**. Store city keys in **lowercase**.
* `customerName` is case-sensitive.
* The same customer must not be added twice to the same city.

**Requirement 2**
Implement:

```java
public List<String> getCustomersByCity(String city)
```

Return the customer names of that city in ascending order (city lookup is case-insensitive). If the city is unknown, return an empty list. In that case the main method displays:

```
No customers found in <city>
```

**Requirement 3**
Implement:

```java
public String findBusiestCity()
```

Return the (lowercase) city having the highest number of customers. If two cities tie, return the one that comes first alphabetically. If the registry is empty, return `"NONE"`.

**Restrictions**

* Edit only the `CustomerRegistry` class.
* Attributes must be `private`.
* Constructor and methods must be `public`.
* Do not change the specified class, attribute, or method names.
* Do not use `System.exit(0)`.

**Input Format**
`n`, then `n` lines of `customerName city` (no spaces inside either), then the city to query.

**Given driver code** (already provided; do not modify)

```java
// Main.java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        CustomerRegistry reg = new CustomerRegistry();
        int n = sc.nextInt();
        for (int i = 0; i < n; i++) {
            String name = sc.next();
            String city = sc.next();
            reg.addCustomer(name, city);
        }
        String query = sc.next();

        List<String> customers = reg.getCustomersByCity(query);
        if (customers.isEmpty()) {
            System.out.println("No customers found in " + query);
        } else {
            for (String c : customers) {
                System.out.println(c);
            }
        }
        System.out.println("Busiest city: " + reg.findBusiestCity());
    }
}
```

**Sample Input 1**

```text
7
Ravi Chennai
Anu chennai
Ravi CHENNAI
Mohan Delhi
Sita delhi
Vikram Mumbai
Latha DELHI
chennai
```

**Sample Output 1**

```text
Anu
Ravi
Busiest city: delhi
```

**Sample Input 2**

```text
7
Ravi Chennai
Anu chennai
Ravi CHENNAI
Mohan Delhi
Sita delhi
Vikram Mumbai
Latha DELHI
Pune
```

**Sample Output 2**

```text
No customers found in Pune
Busiest city: delhi
```

**Sample Input 3**

```text
3
A X
B Y
C Z
X
```

**Sample Output 3**

```text
A
Busiest city: x
```

**Sample Input 4**

```text
0
chennai
```

**Sample Output 4**

```text
No customers found in chennai
Busiest city: NONE
```

### Answer

```java
// CustomerRegistry.java
import java.util.*;

public class CustomerRegistry {
    private HashMap<String, List<String>> cityMap;

    public CustomerRegistry() {
        this.cityMap = new HashMap<>();
    }

    public void addCustomer(String customerName, String city) {
        List<String> customers =
                cityMap.computeIfAbsent(city.toLowerCase(), k -> new ArrayList<>());
        if (!customers.contains(customerName)) {
            customers.add(customerName);
        }
    }

    public List<String> getCustomersByCity(String city) {
        List<String> customers = cityMap.get(city.toLowerCase());
        if (customers == null) {
            return new ArrayList<>();
        }
        List<String> sorted = new ArrayList<>(customers);
        Collections.sort(sorted);
        return sorted;
    }

    public String findBusiestCity() {
        String busiest = "NONE";
        int max = 0;
        for (Map.Entry<String, List<String>> e : cityMap.entrySet()) {
            int size = e.getValue().size();
            if (size > max || (size == max && e.getKey().compareTo(busiest) < 0)) {
                busiest = e.getKey();
                max = size;
            }
        }
        return busiest;
    }
}
```

### Reasoning

* **Map of lists = grouping.** `computeIfAbsent(key, k -> new ArrayList<>())` returns the existing list or creates and stores a new one, so you never write the "if list is null, create it, put it back" boilerplate. It returns the list, so you can call `add` on the result directly.
* **Normalise the key at the boundary.** Lowercase the city in both `addCustomer` and `getCustomersByCity`. Forgetting either side makes lookups miss.
* **Duplicate customers**: `List.contains` is `O(n)`, which is fine at this scale. If the lists were huge, the values would be `Set<String>` instead. Note that `"Ravi"` added twice to `chennai` (once as `Chennai`, once as `CHENNAI`) is only stored once, which is why Chennai has 2 customers, not 3.
* **Return a copy when sorting.** Sorting the internal list in place would silently mutate the registry's stored order, and returning the internal list lets callers modify your data.
* **Tie-break logic**: `size > max` handles a strictly larger city; `size == max && key < busiest` handles alphabetical ties. Sample 3 has three cities with one customer each, so the answer is `x`. Since the initial `busiest` is `"NONE"` and `max` is `0`, the first city examined always wins the first comparison, so the tie-break never wrongly compares against `"NONE"`.
* **Alternative**: declaring `cityMap` as a `TreeMap` would make iteration alphabetical, so a plain `size > max` check would give the tie-break for free. The question demands `HashMap`, so the explicit comparison is needed.
* **Empty registry**: returns `"NONE"` because the loop never runs (sample 4).


---

## Q5. CipherWorks Message Analysis

**Collection:** HashMap + ArrayList (sorting entries)  ·  **Level:** Medium

CipherWorks, a cyber-security firm in Chennai, studies intercepted messages. Analysts want two pieces of information for a given message. Write a Java program based on the following requirements. The `UserMainCode` class must contain the two static methods below.

**Requirement 1**

```java
public static List<String> sortByFrequency(String input)
```

Count every **letter** in the message (ignore case; ignore digits, spaces and symbols). Return a list of strings in the format `letter=count` (letter in lowercase), sorted by:

1. count, **highest first**
2. letter, alphabetically (ascending) when counts are equal

**Requirement 2**

```java
public static char findFirstNonRepeating(String input)
```

Return the first character (scanning left to right) that appears **exactly once** in the message. This check is **case-sensitive** and considers every character, including spaces and digits. If there is no such character, return `'-'`.

The `UserInterface` class reads the whole line, calls both methods and prints the results. If Requirement 1 returns an empty list it prints `No letters found`.

**Input Format**
A single line containing the message (it may contain spaces).

**Output Format**
Each `letter=count` on a separate line, followed by `First non-repeating: <char>`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String input = sc.nextLine();

        List<String> freq = UserMainCode.sortByFrequency(input);
        if (freq.isEmpty()) {
            System.out.println("No letters found");
        } else {
            for (String s : freq) {
                System.out.println(s);
            }
        }
        System.out.println("First non-repeating: " + UserMainCode.findFirstNonRepeating(input));
    }
}
```

**Sample Input 1**

```text
Hello World
```

**Sample Output 1**

```text
l=3
o=2
d=1
e=1
h=1
r=1
w=1
First non-repeating: H
```

**Sample Input 2**

```text
aabbCC
```

**Sample Output 2**

```text
a=2
b=2
c=2
First non-repeating: -
```

**Sample Input 3**

```text
1234 5
```

**Sample Output 3**

```text
No letters found
First non-repeating: 1
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static List<String> sortByFrequency(String input) {
        Map<Character, Integer> freq = new HashMap<>();
        for (char ch : input.toCharArray()) {
            if (Character.isLetter(ch)) {
                freq.merge(Character.toLowerCase(ch), 1, Integer::sum);
            }
        }

        List<Map.Entry<Character, Integer>> entries = new ArrayList<>(freq.entrySet());
        entries.sort((a, b) -> {
            int byCount = b.getValue().compareTo(a.getValue());   // higher count first
            if (byCount != 0) {
                return byCount;
            }
            return a.getKey().compareTo(b.getKey());              // then letter ascending
        });

        List<String> result = new ArrayList<>();
        for (Map.Entry<Character, Integer> e : entries) {
            result.add(e.getKey() + "=" + e.getValue());
        }
        return result;
    }

    public static char findFirstNonRepeating(String input) {
        Map<Character, Integer> count = new HashMap<>();
        for (char ch : input.toCharArray()) {
            count.merge(ch, 1, Integer::sum);
        }
        for (char ch : input.toCharArray()) {
            if (count.get(ch) == 1) {
                return ch;
            }
        }
        return '-';
    }
}
```

### Reasoning

* **A `HashMap` cannot be sorted.** The pattern is: count with the map, copy `entrySet()` into an `ArrayList`, sort that list with a comparator, then read it in order. This "map to list of entries to sort" pattern shows up constantly in assessments.
* **Two-level comparator**: for descending count, compare `b` to `a` (swapping the operands reverses the order). Only when counts tie do you fall through to the alphabetical comparison. Getting the swap direction wrong is the most common bug. An equivalent modern form is `Map.Entry.<Character,Integer>comparingByValue().reversed().thenComparing(Map.Entry.comparingByKey())`.
* **Use `compareTo` on the boxed values**, not `b.getValue() - a.getValue()`. Subtraction can overflow for extreme values and is a habit worth avoiding.
* **Two different normalisations**: Requirement 1 lowercases (so `H` and `h` merge), Requirement 2 does not (so `H` and `h` stay distinct). In `"Hello World"` the uppercase `H` appears once and is the first unique character, even though `e` also appears once.
* **Two passes for "first unique"**: pass one counts, pass two scans the original string in order and returns the first character with count `1`. Iterating the map instead would lose the original order.
* **Sample 3** (`"1234 5"`) has no letters, so Requirement 1 yields an empty list and the driver prints the fallback message. Requirement 2 treats the digit `1` as the first unique character.
* **`e.getKey() + "=" + e.getValue()`** works because the `String` literal in the middle turns the `+` into concatenation. Without any `String` operand, `char + int` would be numeric addition.


---

## Q6. PayTrack Settlement Pair

**Collection:** HashMap (lookup by complement)  ·  **Level:** Medium-Hard

PayTrack Finance, a payments company in Hyderabad, reconciles transactions at the end of the day. Given the amounts of the transactions in the order they were posted, the finance team wants to find two different transactions whose amounts add up exactly to a settlement target.

The `UserMainCode` class must contain a static method:

```java
public static int[] findPair(ArrayList<Integer> amounts, int target)
```

Return an `int` array of size 2 containing the indices `{i, j}` of the two transactions, where `i < j` and `amounts.get(i) + amounts.get(j) == target`.

If several pairs exist, return the pair whose **second index `j` is the smallest**. If there is still more than one choice for `i`, choose the **smallest `i`**.

If no such pair exists, return `{-1, -1}`.

A transaction cannot be paired with itself. Amounts may be zero or negative.

The `UserInterface` class prints `Indices: <i> <j>` or `No pair found`.

**Input Format**
`n`, then `n` amounts, then the `target`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        ArrayList<Integer> amounts = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            amounts.add(sc.nextInt());
        }
        int target = sc.nextInt();

        int[] r = UserMainCode.findPair(amounts, target);
        if (r[0] == -1) {
            System.out.println("No pair found");
        } else {
            System.out.println("Indices: " + r[0] + " " + r[1]);
        }
    }
}
```

**Sample Input 1**

```text
6
3 8 5 1 4 6
9
```

**Sample Output 1**

```text
Indices: 1 3
```

**Sample Input 2**

```text
3
4 4 1
8
```

**Sample Output 2**

```text
Indices: 0 1
```

**Sample Input 3**

```text
1
5
10
```

**Sample Output 3**

```text
No pair found
```

**Sample Input 4**

```text
4
-3 0 3 0
0
```

**Sample Output 4**

```text
Indices: 0 2
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static int[] findPair(ArrayList<Integer> amounts, int target) {
        Map<Integer, Integer> firstIndex = new HashMap<>();   // value -> first index seen

        for (int j = 0; j < amounts.size(); j++) {
            int need = target - amounts.get(j);
            Integer i = firstIndex.get(need);
            if (i != null) {
                return new int[]{i, j};
            }
            firstIndex.putIfAbsent(amounts.get(j), j);        // add AFTER checking
        }
        return new int[]{-1, -1};
    }
}
```

### Reasoning

* **Brute force is `O(n^2)`** (two nested loops). A `HashMap` from value to index turns it into a single `O(n)` pass: for each element ask "have I already seen `target - current`?"
* **Order of operations matters.** Check the map first, then insert the current element. If you insert first, an element can match itself: in Sample 3, `5 + 5 = 10` would wrongly report a pair from a single transaction.
* **Why this gives the required pair**: scanning left to right, the first `j` for which the complement is already in the map is by definition the smallest possible `j`. Storing only the **first** index for each value (`putIfAbsent`) guarantees the smallest `i` for that `j`. Plain `put` would keep overwriting with the latest index and return a larger `i` when values repeat.
* **Duplicates that must pair with each other** (Sample 2, `4 + 4 = 8`) work correctly: at `j = 1` the first `4` is already in the map from `j = 0`.
* **Zero and negatives** need no special handling (Sample 4: `-3 + 3` at indices `0, 2` is found before `0 + 0` at `1, 3`, because `j = 2` comes before `j = 3`).
* **`Integer i = firstIndex.get(need)`**: keep the result as an `Integer` (not `int`) so the `null` check works. Assigning a missing key to an `int` triggers a `NullPointerException` on unboxing.
* **Sorting the list first is a trap here**: it destroys the original indices the answer must report.


---


---

# Part B: HashSet

## Q7. TechNova Training Batches

**Collection:** HashSet (set operations)  ·  **Level:** Easy-Medium

TechNova Solutions, an IT services company in Hyderabad, ran two training batches. The employee IDs attending each batch were recorded from sign-in sheets, and some employees signed in more than once. HR wants to know who attended **both** batches and who attended **only the first** batch.

The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static String[] findCommonIds(String[] batch1, String[] batch2)
```

Return the IDs present in both batches, without duplicates, sorted in ascending order.

**Requirement 2**

```java
public static String[] findExclusiveIds(String[] batch1, String[] batch2)
```

Return the IDs present in `batch1` but **not** in `batch2`, without duplicates, sorted in ascending order.

Constraint: IDs are case-sensitive (`"E10"` and `"e10"` are different employees). The result may be empty. The `UserInterface` prints `None` for an empty result.

**Input Format**
`n`, then `n` IDs of batch 1, then `m`, then `m` IDs of batch 2.

**Output Format**
`Common IDs:` followed by the IDs, then `Only in Batch 1:` followed by the IDs. Each ID on a separate line.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        String[] batch1 = new String[n];
        for (int i = 0; i < n; i++) {
            batch1[i] = sc.next();
        }
        int m = sc.nextInt();
        String[] batch2 = new String[m];
        for (int i = 0; i < m; i++) {
            batch2[i] = sc.next();
        }

        System.out.println("Common IDs:");
        print(UserMainCode.findCommonIds(batch1, batch2));
        System.out.println("Only in Batch 1:");
        print(UserMainCode.findExclusiveIds(batch1, batch2));
    }

    private static void print(String[] arr) {
        if (arr.length == 0) {
            System.out.println("None");
        } else {
            for (String s : arr) {
                System.out.println(s);
            }
        }
    }
}
```

**Sample Input 1**

```text
5
E102 E101 E105 E101 E103
4
E103 E109 E101 E103
```

**Sample Output 1**

```text
Common IDs:
E101
E103
Only in Batch 1:
E102
E105
```

**Sample Input 2**

```text
2
e10 E10
2
E10 E11
```

**Sample Output 2**

```text
Common IDs:
E10
Only in Batch 1:
e10
```

**Sample Input 3**

```text
2
A1 A2
1
B1
```

**Sample Output 3**

```text
Common IDs:
None
Only in Batch 1:
A1
A2
```

**Sample Input 4**

```text
1
X1
1
X1
```

**Sample Output 4**

```text
Common IDs:
X1
Only in Batch 1:
None
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static String[] findCommonIds(String[] batch1, String[] batch2) {
        Set<String> common = new HashSet<>(Arrays.asList(batch1));
        common.retainAll(new HashSet<>(Arrays.asList(batch2)));   // intersection
        return sortedArray(common);
    }

    public static String[] findExclusiveIds(String[] batch1, String[] batch2) {
        Set<String> only = new HashSet<>(Arrays.asList(batch1));
        only.removeAll(new HashSet<>(Arrays.asList(batch2)));     // difference
        return sortedArray(only);
    }

    private static String[] sortedArray(Set<String> set) {
        String[] result = set.toArray(new String[0]);
        Arrays.sort(result);
        return result;
    }
}
```

### Reasoning

* **Sets do the de-duplication for you.** Wrapping the array in a `HashSet` removes repeated sign-ins automatically, so there is no manual duplicate-checking loop.
* **Set algebra methods**: `retainAll` = intersection, `removeAll` = difference, `addAll` = union. All three **mutate the set they are called on**, which is why each method builds its own fresh `HashSet` first. Calling `retainAll` directly on a set you still need later is a classic bug.
* **Why wrap the argument in a `HashSet` too**: `retainAll` and `removeAll` call `contains` on the argument. A `HashSet` gives `O(1)` lookups; passing a `List` would make the whole operation `O(n*m)`.
* **`Arrays.asList(array)`** gives a fixed-size list view that is fine as a constructor argument. Never try to `add` or `remove` on that list itself.
* **Sort at the end**: `HashSet` iteration order is unspecified. Copy to an array, then `Arrays.sort` (natural order, uppercase before lowercase).
* **Case sensitivity** (Sample 2): `e10` is exclusive to batch 1 while `E10` is common. No `toLowerCase()` should be applied anywhere.
* **Empty results** (Samples 3 and 4) must return an empty array, not `null`, so `arr.length` in the driver works.


---

## Q8. SafeBank Duplicate Transactions

**Collection:** HashSet (add() return value)  ·  **Level:** Medium

SafeBank, a private bank in Mumbai, screens each batch of transaction reference numbers for duplicates, because a repeated reference number may indicate a fraudulent replay. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static String findFirstDuplicate(String[] refs)
```

Scan the array from left to right and return the reference number whose **second occurrence appears earliest**. If there are no duplicates, return `"NO DUPLICATES"`.

**Requirement 2**

```java
public static List<String> findAllDuplicates(String[] refs)
```

Return every reference number that occurs more than once, **each exactly once**, in the order in which their **second occurrence** appears in the array. Return an empty list if there are none.

Constraint: reference numbers are case-sensitive.

The `UserInterface` prints the first duplicate, then either the list of duplicates or `No duplicates found`.

**Input Format**
`n`, then `n` reference numbers.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        String[] refs = new String[n];
        for (int i = 0; i < n; i++) {
            refs[i] = sc.next();
        }

        System.out.println(UserMainCode.findFirstDuplicate(refs));

        List<String> dups = UserMainCode.findAllDuplicates(refs);
        if (dups.isEmpty()) {
            System.out.println("No duplicates found");
        } else {
            for (String d : dups) {
                System.out.println(d);
            }
        }
    }
}
```

**Sample Input 1**

```text
8
TX1 TX2 TX3 TX2 TX1 TX4 TX2 TX3
```

**Sample Output 1**

```text
TX2
TX2
TX1
TX3
```

**Sample Input 2**

```text
4
a b c d
```

**Sample Output 2**

```text
NO DUPLICATES
No duplicates found
```

**Sample Input 3**

```text
5
TX1 tx1 TX1 tx1 TX1
```

**Sample Output 3**

```text
TX1
TX1
tx1
```

**Sample Input 4**

```text
1
only
```

**Sample Output 4**

```text
NO DUPLICATES
No duplicates found
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static String findFirstDuplicate(String[] refs) {
        Set<String> seen = new HashSet<>();
        for (String ref : refs) {
            if (!seen.add(ref)) {      // add() returns false if already present
                return ref;
            }
        }
        return "NO DUPLICATES";
    }

    public static List<String> findAllDuplicates(String[] refs) {
        Set<String> seen = new HashSet<>();
        Set<String> reported = new HashSet<>();
        List<String> result = new ArrayList<>();

        for (String ref : refs) {
            if (!seen.add(ref) && reported.add(ref)) {
                result.add(ref);
            }
        }
        return result;
    }
}
```

### Reasoning

* **`Set.add()` returns a `boolean`**: `true` if the element was newly added, `false` if it was already present. That makes "seen before?" a single call, `if (!seen.add(x))`, instead of `contains` followed by `add`.
* **Why "second occurrence" order**: as soon as an element fails to be added, that position is its second occurrence. Scanning left to right, the first failure is the earliest second occurrence, so `findFirstDuplicate` can return immediately. In Sample 1, `TX2` (index 3) is found before `TX1` (index 4), even though `TX1` appears first in the array.
* **Reporting each duplicate once**: `TX2` appears three times, so it fails `seen.add` twice. The second set `reported` filters the repeat. The expression `!seen.add(ref) && reported.add(ref)` relies on **short-circuit evaluation**: `reported.add` runs only when `ref` was already seen, and it returns `true` only the first time.
* **Why not use a `HashSet` for the result**: it would lose the required order. An `ArrayList` keeps the insertion order; a `LinkedHashSet` would also work.
* **Order of the operands matters**: writing `reported.add(ref) && !seen.add(ref)` would add every element to `reported` before checking `seen`, giving wrong output.
* **Case sensitivity** (Sample 3): `TX1` and `tx1` are different, and each is a duplicate in its own right. The expected order is `TX1` first (its second occurrence is at index 2) and `tx1` second (index 3).
* **Edge cases**: a single element and an all-unique array both return the fallback values.


---

## Q9. AirNest Passenger Check-in

**Collection:** HashSet (custom equals/hashCode)  ·  **Level:** Medium-Hard

AirNest Airlines, based in Mumbai, noticed that its check-in kiosks sometimes register the same passenger more than once, with names typed slightly differently. A passenger is uniquely identified by the **passport number**, not by the name. Develop a Java program based on the following requirements.

**Requirement 1**
Create a `Passenger` class with two `private` attributes, `name` and `passportNumber`, a `public` constructor `Passenger(String name, String passportNumber)`, and getters `getName()` and `getPassportNumber()`.

Two `Passenger` objects must be considered **equal when their passport numbers match, ignoring case**. The name must not affect equality.
Override `equals` and `hashCode` accordingly.

**Requirement 2**
Create a `PassengerManager` class containing a `HashSet<Passenger>` named `passengerSet`.
Implement:

```java
public boolean registerPassenger(Passenger passenger)
```

Add the passenger to the set. Return `true` if the passenger was newly registered and `false` if it is a duplicate.

```java
public int getUniquePassengerCount()
```

Return the number of distinct passengers registered so far.

**Restrictions**

* Edit only the `Passenger` and `PassengerManager` classes.
* Attributes must be `private`.
* Constructors and methods must be `public`.
* Do not change the specified class, attribute, or method names.
* Do not use `System.exit(0)`.

**Input Format**
`n`, then `n` lines of `name passportNumber` (no spaces inside either).

**Given driver code** (already provided; do not modify)

```java
// Main.java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        PassengerManager pm = new PassengerManager();
        int n = sc.nextInt();
        for (int i = 0; i < n; i++) {
            String name = sc.next();
            String passport = sc.next();
            Passenger p = new Passenger(name, passport);
            if (pm.registerPassenger(p)) {
                System.out.println("Registered: " + p.getName());
            } else {
                System.out.println("Duplicate passenger: " + p.getName());
            }
        }
        System.out.println("Unique passengers: " + pm.getUniquePassengerCount());
    }
}
```

**Sample Input 1**

```text
5
Arun K123A
Bina M998Z
Arun-Kumar k123a
Chitra P554Q
Bina M998Z
```

**Sample Output 1**

```text
Registered: Arun
Registered: Bina
Duplicate passenger: Arun-Kumar
Registered: Chitra
Duplicate passenger: Bina
Unique passengers: 3
```

**Sample Input 2**

```text
3
Ravi X1
Ravi x1
Ravi X2
```

**Sample Output 2**

```text
Registered: Ravi
Duplicate passenger: Ravi
Registered: Ravi
Unique passengers: 2
```

### Answer

```java
// Passenger.java
import java.util.Locale;

public class Passenger {
    private String name;
    private String passportNumber;

    public Passenger(String name, String passportNumber) {
        this.name = name;
        this.passportNumber = passportNumber;
    }

    public String getName() {
        return name;
    }

    public String getPassportNumber() {
        return passportNumber;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        Passenger other = (Passenger) o;
        return passportNumber.equalsIgnoreCase(other.passportNumber);
    }

    @Override
    public int hashCode() {
        return passportNumber.toLowerCase(Locale.ROOT).hashCode();
    }
}
```

```java
// PassengerManager.java
import java.util.HashSet;

public class PassengerManager {
    private HashSet<Passenger> passengerSet;

    public PassengerManager() {
        this.passengerSet = new HashSet<>();
    }

    public boolean registerPassenger(Passenger passenger) {
        return passengerSet.add(passenger);
    }

    public int getUniquePassengerCount() {
        return passengerSet.size();
    }
}
```

### Reasoning

* **How `HashSet` decides two objects are the same**: it first compares `hashCode()` to pick a bucket, then calls `equals()` inside that bucket. Without overriding both, the default `Object` versions compare **memory addresses**, so two `Passenger` objects with identical passports would both be stored.
* **The contract**: if `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` **must** hold. Here `equals` ignores case, so `hashCode` must normalise case too. If `hashCode` used the raw passport string, `"K123A"` and `"k123a"` would land in different buckets, `equals` would never even be consulted, and the duplicate would slip through. This is the most tested trap in this topic.
* **Both methods must use the same fields.** The name is excluded from both, because it must not affect equality.
* **`toLowerCase(Locale.ROOT)`** avoids locale surprises (for example the Turkish dotted/dotless `i`). Plain `toLowerCase()` also passes here, but the `Locale.ROOT` habit is safer.
* **`add()` already returns the answer**: `true` for a new element, `false` for an existing one. No `contains` call is needed.
* **The `equals` template**: check `this == o`, then `null` and class mismatch, then cast and compare the relevant fields. Use `getClass() != o.getClass()` (or `instanceof`) and never assume `o` is a `Passenger`.
* **Mutable key warning**: if you changed `passportNumber` after inserting the object into a `HashSet`, its stored bucket would be stale and the object could become unfindable. That is why the fields here are set once in the constructor and have no setters.


---


---

# Part C: TreeMap

## Q10. Bharat Savings Interest Slabs

**Collection:** TreeMap (floorEntry / tailMap)  ·  **Level:** Medium

Bharat Savings Bank, headquartered in Pune, pays interest on savings accounts according to balance slabs. Each slab is defined by its **minimum balance**, and a customer earns the rate of the highest slab whose minimum balance does not exceed their balance. Develop a Java program based on the following requirements.

**Requirement 1**
Create an `InterestSlabManager` class containing a `TreeMap` named `slabMap` (`minBalance -> rate`).
Implement:

```java
public void addSlab(int minBalance, double rate)
```

If a slab with the same `minBalance` already exists, its rate must be **replaced**.

**Requirement 2**
Implement:

```java
public Double findInterestRate(int balance)
```

Return the rate of the slab with the **highest** `minBalance` that is **less than or equal to** `balance`. If the balance is below the lowest slab, return `null`. In that case the main method displays:

```
No slab applicable for balance <balance>
```

**Requirement 3**
Implement:

```java
public List<Integer> findHigherSlabs(int balance)
```

Return the `minBalance` values of all slabs **strictly greater than** `balance`, in ascending order. If there are none, return an empty list. The main method then displays `No higher slabs available`.

**Restrictions**

* Edit only the `InterestSlabManager` class.
* Attributes must be `private`.
* Constructor and methods must be `public`.
* Do not change the specified class, attribute, or method names.
* Do not use `System.exit(0)`.

**Input Format**
`n`, then `n` lines of `minBalance rate`, then the customer's `balance`.

**Given driver code** (already provided; do not modify)

```java
// Main.java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        InterestSlabManager mgr = new InterestSlabManager();
        int n = sc.nextInt();
        for (int i = 0; i < n; i++) {
            int minBalance = sc.nextInt();
            double rate = sc.nextDouble();
            mgr.addSlab(minBalance, rate);
        }
        int balance = sc.nextInt();

        Double rate = mgr.findInterestRate(balance);
        if (rate == null) {
            System.out.println("No slab applicable for balance " + balance);
        } else {
            System.out.println("Interest rate: " + rate);
        }

        List<Integer> higher = mgr.findHigherSlabs(balance);
        if (higher.isEmpty()) {
            System.out.println("No higher slabs available");
        } else {
            for (int b : higher) {
                System.out.println(b);
            }
        }
    }
}
```

**Sample Input 1**

```text
5
10000 3.5
50000 4.5
100000 5.5
25000 4.0
50000 4.75
60000
```

**Sample Output 1**

```text
Interest rate: 4.75
100000
```

**Sample Input 2**

```text
5
10000 3.5
50000 4.5
100000 5.5
25000 4.0
50000 4.75
5000
```

**Sample Output 2**

```text
No slab applicable for balance 5000
10000
25000
50000
100000
```

**Sample Input 3**

```text
5
10000 3.5
50000 4.5
100000 5.5
25000 4.0
50000 4.75
100000
```

**Sample Output 3**

```text
Interest rate: 5.5
No higher slabs available
```

**Sample Input 4**

```text
5
10000 3.5
50000 4.5
100000 5.5
25000 4.0
50000 4.75
25000
```

**Sample Output 4**

```text
Interest rate: 4.0
50000
100000
```

### Answer

```java
// InterestSlabManager.java
import java.util.*;

public class InterestSlabManager {
    private TreeMap<Integer, Double> slabMap;

    public InterestSlabManager() {
        this.slabMap = new TreeMap<>();
    }

    public void addSlab(int minBalance, double rate) {
        slabMap.put(minBalance, rate);
    }

    public Double findInterestRate(int balance) {
        Map.Entry<Integer, Double> entry = slabMap.floorEntry(balance);
        if (entry == null) {
            return null;
        }
        return entry.getValue();
    }

    public List<Integer> findHigherSlabs(int balance) {
        return new ArrayList<>(slabMap.tailMap(balance, false).keySet());
    }
}
```

### Reasoning

* **`TreeMap` keeps keys sorted**, which unlocks the navigation methods a `HashMap` cannot offer. "Highest key not exceeding X" is exactly `floorEntry(X)`.
* **The four neighbours, and what they include**:
  * `floorEntry(k)`: greatest key **<= k**
  * `lowerEntry(k)`: greatest key **< k**
  * `ceilingEntry(k)`: smallest key **>= k**
  * `higherEntry(k)`: smallest key **> k**

  Choosing `lowerEntry` here would be wrong: a balance of exactly `25000` must earn the `25000` slab (Sample 4), and `lowerEntry` would skip it.
* **Return `null` when nothing qualifies**: `floorEntry` returns `null` if every key is greater than the argument (Sample 2). Check for it before calling `getValue()`, or you get a `NullPointerException`. This is why the return type is the wrapper `Double` rather than the primitive `double`.
* **`tailMap(key, inclusive)`** returns the portion of the map from `key` onward. Passing `false` makes the bound exclusive, giving "strictly greater". The one-argument `tailMap(key)` is **inclusive**, which would wrongly include a slab equal to the balance (Sample 3 and 4 show the boundary).
* **Views vs copies**: `tailMap(...)` and `keySet()` are live views backed by the original map. Wrapping the keys in `new ArrayList<>(...)` returns an independent snapshot, so callers cannot accidentally change the slab map.
* **Overwrite behaviour**: `put` on an existing key replaces the value. The `50000` slab ends up with `4.75`, not `4.5`.
* **Related safe-vs-unsafe pair**: `firstKey()` and `lastKey()` throw `NoSuchElementException` on an empty map, while `firstEntry()` and `lastEntry()` return `null`.


---

## Q11. BrainBowl Quiz Leaderboard

**Collection:** TreeMap (case-insensitive comparator, subMap)  ·  **Level:** Medium-Hard

BrainBowl, an inter-college quiz competition held in Chennai, records a score every time a participant plays a round. A participant may play several rounds and the organisers typed the names inconsistently (`Ravi`, `RAVI`, `ravi`). Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static TreeMap<String, Integer> buildLeaderboard(String[] names, int[] scores)
```

`names[i]` scored `scores[i]`. Build the leaderboard as follows:

* Names are **case-insensitive**: `Ravi` and `RAVI` are the same participant.
* Keep only the **highest** score of each participant.
* The name shown on the leaderboard is the spelling that appeared **first** in the input.
* The leaderboard must be **sorted alphabetically, ignoring case**.

**Requirement 2**

```java
public static List<String> getParticipantsBetween(TreeMap<String, Integer> board, String from, String to)
```

Return the names on the leaderboard that fall alphabetically between `from` and `to`, **both inclusive** (case-insensitive), in ascending order. If `from` comes after `to`, return an empty list (do **not** throw an exception).

**Input Format**
`n`, then `n` lines of `name score`, then `from` and `to`.

**Output Format**
Each leaderboard entry as `name score`, then `Between <from> and <to>:` followed by the matching names (or `No participants in range`).

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        String[] names = new String[n];
        int[] scores = new int[n];
        for (int i = 0; i < n; i++) {
            names[i] = sc.next();
            scores[i] = sc.nextInt();
        }
        String from = sc.next();
        String to = sc.next();

        TreeMap<String, Integer> board = UserMainCode.buildLeaderboard(names, scores);
        for (Map.Entry<String, Integer> e : board.entrySet()) {
            System.out.println(e.getKey() + " " + e.getValue());
        }

        List<String> range = UserMainCode.getParticipantsBetween(board, from, to);
        System.out.println("Between " + from + " and " + to + ":");
        if (range.isEmpty()) {
            System.out.println("No participants in range");
        } else {
            for (String s : range) {
                System.out.println(s);
            }
        }
    }
}
```

**Sample Input 1**

```text
6
Ravi 70
anita 85
RAVI 90
Zoya 60
Anita 80
bala 75
Bala RAVI
```

**Sample Output 1**

```text
anita 85
bala 75
Ravi 90
Zoya 60
Between Bala and RAVI:
bala
Ravi
```

**Sample Input 2**

```text
6
Ravi 70
anita 85
RAVI 90
Zoya 60
Anita 80
bala 75
zoya anita
```

**Sample Output 2**

```text
anita 85
bala 75
Ravi 90
Zoya 60
Between zoya and anita:
No participants in range
```

**Sample Input 3**

```text
3
kiran 40
KIRAN 40
Kiran 10
kiran kiran
```

**Sample Output 3**

```text
kiran 40
Between kiran and kiran:
kiran
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static TreeMap<String, Integer> buildLeaderboard(String[] names, int[] scores) {
        TreeMap<String, Integer> board = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);

        for (int i = 0; i < names.length; i++) {
            Integer existing = board.get(names[i]);
            if (existing == null || scores[i] > existing) {
                board.put(names[i], scores[i]);   // replaces the value, keeps the original key
            }
        }
        return board;
    }

    public static List<String> getParticipantsBetween(TreeMap<String, Integer> board,
                                                      String from, String to) {
        if (board.comparator().compare(from, to) > 0) {
            return new ArrayList<>();
        }
        return new ArrayList<>(board.subMap(from, true, to, true).keySet());
    }
}
```

### Reasoning

* **A `TreeMap` decides key equality with its comparator, not with `equals()`.** With `String.CASE_INSENSITIVE_ORDER`, `"Ravi"`, `"RAVI"` and `"ravi"` all compare as `0`, so they are the **same key**. That single constructor argument gives you the case-insensitive merge for free.
* **`put` on an existing key replaces the value but keeps the original key object.** This is exactly why the first-seen spelling survives: `Ravi` is inserted first, and the later `RAVI 90` only updates the value (the leaderboard still prints `Ravi 90`). Likewise `anita` stays lowercase.
* **Keep the highest score**: fetch the current value, and only `put` if it is absent or the new score is greater. `board.merge(name, score, Math::max)` is an equivalent one-liner.
* **Sorting is automatic**: iterating a `TreeMap` yields keys in comparator order. The output starts with `anita` and `bala` and not with `Ravi` because the comparison ignores case. With natural ordering, `Ravi` and `Zoya` would have jumped ahead of the lowercase names.
* **`subMap(from, fromInclusive, to, toInclusive)`** returns the range view. The 4-argument version is needed because the 2-argument `subMap(from, to)` is inclusive at the start but **exclusive at the end**.
* **The trap in Sample 2**: `subMap` throws `IllegalArgumentException` if `from > to`. Guard first using the map's own comparator (`board.comparator().compare(from, to)`) so the same case-insensitive rule applies to the guard as to the range.
* **The bounds need not exist as keys.** `subMap` works on the ordering, so `from`/`to` can be any strings. Watch the end bound: with `to = "r"`, the name `Ravi` sorts **after** `"r"` (a longer string with the same prefix is greater), so it would be excluded. Pick bounds carefully when designing tests.
* **Sample 3** confirms that three spellings collapse into one entry with the highest score.


---

## Q12. Sunrise Retail Regional Sales

**Collection:** TreeMap (aggregation + parsing)  ·  **Level:** Hard

Sunrise Retail, a retail chain based in Jaipur, receives sales records from its stores as text in the format `region:amount`, for example `north:1200`. The head office wants the total sales per region, listed alphabetically, and wants to know which regions performed **above average**. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static TreeMap<String, Integer> totalByRegion(String[] records)
```

Parse each record and add its amount to the total of its region.

* Region names are **case-insensitive** and must be stored in **uppercase**.
* Amounts are integers (they may be negative for returns).
* Records that are **badly formatted** (no `:` or more than one `:`) or whose amount is **not a valid integer** must be **ignored**.
* The returned map must be sorted by region name.

**Requirement 2**

```java
public static List<String> regionsAboveAverage(TreeMap<String, Integer> totals)
```

Return, in the map's order, the regions whose total is **strictly greater** than the average of all region totals. Return an empty list if the map is empty or no region qualifies.

The `UserInterface` prints each total as `REGION=total` (or `No valid records`), followed by `Above average:` and the regions (or `No region above average`).

**Input Format**
`n`, then `n` records (no spaces inside a record).

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        String[] records = new String[n];
        for (int i = 0; i < n; i++) {
            records[i] = sc.next();
        }

        TreeMap<String, Integer> totals = UserMainCode.totalByRegion(records);
        if (totals.isEmpty()) {
            System.out.println("No valid records");
        } else {
            for (Map.Entry<String, Integer> e : totals.entrySet()) {
                System.out.println(e.getKey() + "=" + e.getValue());
            }
        }

        List<String> above = UserMainCode.regionsAboveAverage(totals);
        if (above.isEmpty()) {
            System.out.println("No region above average");
        } else {
            System.out.println("Above average:");
            for (String r : above) {
                System.out.println(r);
            }
        }
    }
}
```

**Sample Input 1**

```text
7
north:1200
South:800
NORTH:300
east:500
west:x50
south:400
East:100
```

**Sample Output 1**

```text
EAST=600
NORTH=1500
SOUTH=1200
Above average:
NORTH
SOUTH
```

**Sample Input 2**

```text
2
a:10
b:10
```

**Sample Output 2**

```text
A=10
B=10
No region above average
```

**Sample Input 3**

```text
3
abc
north:5:6
south:12.5
```

**Sample Output 3**

```text
No valid records
No region above average
```

**Sample Input 4**

```text
4
north:100
south:-40
north:-30
east:25
```

**Sample Output 4**

```text
EAST=25
NORTH=70
SOUTH=-40
Above average:
EAST
NORTH
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static TreeMap<String, Integer> totalByRegion(String[] records) {
        TreeMap<String, Integer> totals = new TreeMap<>();

        for (String record : records) {
            String[] parts = record.split(":");
            if (parts.length != 2) {
                continue;                               // badly formatted
            }
            try {
                int amount = Integer.parseInt(parts[1].trim());
                totals.merge(parts[0].trim().toUpperCase(), amount, Integer::sum);
            } catch (NumberFormatException ex) {
                // invalid amount: ignore this record
            }
        }
        return totals;
    }

    public static List<String> regionsAboveAverage(TreeMap<String, Integer> totals) {
        List<String> result = new ArrayList<>();
        if (totals.isEmpty()) {
            return result;
        }

        long sum = 0;
        for (int value : totals.values()) {
            sum += value;
        }
        double average = (double) sum / totals.size();

        for (Map.Entry<String, Integer> e : totals.entrySet()) {
            if (e.getValue() > average) {
                result.add(e.getKey());
            }
        }
        return result;
    }
}
```

### Reasoning

* **Normalise the key, then aggregate.** Uppercasing the region *before* `merge` makes `north`, `North` and `NORTH` one entry. Here the map uses natural ordering, so storing uppercase keys is what makes the case-insensitive grouping work (contrast with Q11, where a comparator did that job).
* **`merge(key, amount, Integer::sum)`** is the cleanest "running total per key" idiom: it inserts the amount when the key is new and otherwise adds to the existing total.
* **Validate before you trust the input.** `split(":")` on `"abc"` gives one part, on `"north:5:6"` gives three, so `parts.length != 2` rejects both. One subtlety: `"north:".split(":")` gives just `["north"]` (trailing empty strings are dropped), so it is rejected too.
* **`Integer.parseInt` throws `NumberFormatException`** for `"x50"` and for `"12.5"`. Catch it and skip the record. The `try` block wraps both the parse and the merge, so nothing is added for a bad record.
* **Average with the right types**: sum into a `long`, then divide as a `double`. Integer division truncates toward zero, and that matters as soon as returns make totals negative. With two regions totalling `-2` and `-3`, the true average is `-2.5`, so the region with `-2` **is** above average. Integer division gives `-5 / 2 = -2`, and `-2 > -2` is false, so the region would be wrongly dropped. (For all-positive totals truncation happens not to change the result, which is why this bug survives casual testing.) In Sample 4 the totals are `EAST=25`, `NORTH=70`, `SOUTH=-40`, so the average is `55 / 3 = 18.33...` and both `EAST` and `NORTH` qualify.
* **Strictly greater** means a region equal to the average does not qualify. Sample 2 has both totals equal to the average, so the result is empty.
* **Guard the empty case** before dividing: `0 / 0` in `double` gives `NaN` and in `int` throws `ArithmeticException`.
* **Order for free**: iterating the `TreeMap` gives alphabetical regions, so no sorting is needed for the result.


---


---

# Part D: ArrayList

## Q13. SkyWays Baggage Screening

**Collection:** ArrayList (safe removal)  ·  **Level:** Medium

SkyWays Airlines, operating from Delhi, screens checked-in baggage before loading. The weight of each bag (in kg) is stored in an `ArrayList<Integer>`. Two clean-up steps are needed. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static List<Integer> removeBagsOfWeight(List<Integer> weights, int target)
```

Remove **all** bags whose weight is exactly `target` from the given list and return the same list.

**Requirement 2**

```java
public static List<Integer> removeOverweightBags(List<Integer> weights, int limit)
```

Remove all bags whose weight is **strictly greater than** `limit` from the given list and return the same list.

**Rules**

* Both methods must modify and return the list that was passed in (in place).
* The methods must not throw `ConcurrentModificationException`.
* The weight value `target` may also be a valid index in the list. The method must remove **values**, never positions.

The `UserInterface` prints the list after each step, with the weights separated by a space, or `Empty` if no bag remains.

**Input Format**
`n`, then `n` weights, then `target`, then `limit`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;
import java.util.stream.Collectors;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        List<Integer> weights = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            weights.add(sc.nextInt());
        }
        int target = sc.nextInt();
        int limit = sc.nextInt();

        List<Integer> step1 = UserMainCode.removeBagsOfWeight(weights, target);
        print("After removing weight " + target + ":", step1);

        List<Integer> step2 = UserMainCode.removeOverweightBags(weights, limit);
        print("After removing bags above " + limit + ":", step2);
    }

    private static void print(String label, List<Integer> list) {
        String text = list.isEmpty()
                ? "Empty"
                : list.stream().map(String::valueOf).collect(Collectors.joining(" "));
        System.out.println(label + " " + text);
    }
}
```

**Sample Input 1**

```text
8
5 2 7 2 30 2 18 40
2
20
```

**Sample Output 1**

```text
After removing weight 2: 5 7 30 18 40
After removing bags above 20: 5 7 18
```

**Sample Input 2**

```text
3
1 1 1
1
10
```

**Sample Output 2**

```text
After removing weight 1: Empty
After removing bags above 10: Empty
```

**Sample Input 3**

```text
4
1 5 7 1
1
100
```

**Sample Output 3**

```text
After removing weight 1: 5 7
After removing bags above 100: 5 7
```

**Sample Input 4**

```text
5
10 20 30 40 50
99
30
```

**Sample Output 4**

```text
After removing weight 99: 10 20 30 40 50
After removing bags above 30: 10 20 30
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static List<Integer> removeBagsOfWeight(List<Integer> weights, int target) {
        weights.removeIf(w -> w == target);
        return weights;
    }

    public static List<Integer> removeOverweightBags(List<Integer> weights, int limit) {
        Iterator<Integer> it = weights.iterator();
        while (it.hasNext()) {
            if (it.next() > limit) {
                it.remove();
            }
        }
        return weights;
    }
}
```

### Reasoning

* **`remove(int)` vs `remove(Object)` is the trap of this question.** On a `List<Integer>`, `weights.remove(target)` with an `int` argument calls `remove(int index)` and deletes the element **at that position**. In Sample 3 (`1 5 7 1`, target `1`), the buggy call would delete `5` (index 1) and leave `1 7 1`. To remove by value, use `weights.remove(Integer.valueOf(target))`, or better, `removeIf`.
* **`removeIf(predicate)`** removes every matching element in one call and handles the shifting internally. Inside the lambda, `w == target` compares an `Integer` with an `int`, so `w` is unboxed and the comparison is numeric. This is safe.
* **Why not a for-each loop with `weights.remove(...)`?** Structurally modifying a list while a for-each (which uses an iterator behind the scenes) is running throws `ConcurrentModificationException`. The safe options are `removeIf`, an explicit `Iterator` with `it.remove()` (used in Requirement 2), or a backwards index loop `for (int i = size - 1; i >= 0; i--)`.
* **Why not a forwards index loop?** After `remove(i)` all later elements shift left, so the next element moves into slot `i` and the loop's `i++` skips it. With input `2 2 7`, a forwards loop removing the value `2` would leave one `2` behind.
* **Comparing `Integer` objects**: `it.next() > limit` auto-unboxes, which is fine. But never compare two `Integer` objects with `==` (it compares references and only appears to work for small cached values, -128 to 127). Use `.equals()` or unbox first.
* **In-place contract**: the driver passes the same list to both methods, so Requirement 2 sees the result of Requirement 1. Building a new list instead would still print correctly here, but it would break the stated contract.
* **Immutable lists**: `removeIf` on `List.of(...)` or `Arrays.asList(...)` fails (`UnsupportedOperationException`). The driver uses `new ArrayList<>()`, so it is fine here, but keep it in mind.
* **Sample 4**: the target `99` is not present, so nothing is removed in step 1, and `30` (not strictly greater than the limit `30`) survives step 2.


---

## Q14. MetroCab Ride ID Consolidation

**Collection:** ArrayList (two-pointer merge, subList)  ·  **Level:** Medium

MetroCab, a cab aggregator in Bengaluru, receives ride IDs from two zone servers. Each zone sends its IDs as an already sorted (ascending) `ArrayList<Integer>`, but IDs can repeat inside a list and across the two lists. The operations team needs one clean consolidated list, and a rotated view of it for shift planning. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static ArrayList<Integer> mergeSortedUnique(ArrayList<Integer> zoneA, ArrayList<Integer> zoneB)
```

Merge the two sorted lists into one list that is sorted ascending and contains **no duplicates**. Either list may be empty. Do not modify the input lists.

**Requirement 2**

```java
public static ArrayList<Integer> rotateRight(ArrayList<Integer> list, int k)
```

Return a **new** list obtained by rotating the given list to the right by `k` positions (`k >= 0`). The last `k` elements move to the front, keeping their order. `k` may be larger than the size of the list. An empty list returns an empty list.

The `UserInterface` merges the two zone lists and then rotates the merged list. It prints each list on one line, elements separated by a space, or `Empty`.

**Input Format**
`n`, then `n` IDs of zone A, then `m`, then `m` IDs of zone B, then `k`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;
import java.util.stream.Collectors;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        ArrayList<Integer> zoneA = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            zoneA.add(sc.nextInt());
        }
        int m = sc.nextInt();
        ArrayList<Integer> zoneB = new ArrayList<>();
        for (int i = 0; i < m; i++) {
            zoneB.add(sc.nextInt());
        }
        int k = sc.nextInt();

        ArrayList<Integer> merged = UserMainCode.mergeSortedUnique(zoneA, zoneB);
        print("Merged:", merged);
        print("Rotated:", UserMainCode.rotateRight(merged, k));
    }

    private static void print(String label, List<Integer> list) {
        String text = list.isEmpty()
                ? "Empty"
                : list.stream().map(String::valueOf).collect(Collectors.joining(" "));
        System.out.println(label + " " + text);
    }
}
```

**Sample Input 1**

```text
5
1 3 3 7 9
4
2 3 8 9
3
```

**Sample Output 1**

```text
Merged: 1 2 3 7 8 9
Rotated: 7 8 9 1 2 3
```

**Sample Input 2**

```text
0
2
4 4
5
```

**Sample Output 2**

```text
Merged: 4
Rotated: 4
```

**Sample Input 3**

```text
0
0
3
```

**Sample Output 3**

```text
Merged: Empty
Rotated: Empty
```

**Sample Input 4**

```text
3
1 2 3
0
6
```

**Sample Output 4**

```text
Merged: 1 2 3
Rotated: 1 2 3
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static ArrayList<Integer> mergeSortedUnique(ArrayList<Integer> zoneA,
                                                       ArrayList<Integer> zoneB) {
        ArrayList<Integer> result = new ArrayList<>();
        int i = 0;
        int j = 0;

        while (i < zoneA.size() || j < zoneB.size()) {
            int next;
            if (j >= zoneB.size() || (i < zoneA.size() && zoneA.get(i) <= zoneB.get(j))) {
                next = zoneA.get(i++);
            } else {
                next = zoneB.get(j++);
            }

            if (result.isEmpty() || result.get(result.size() - 1) != next) {
                result.add(next);
            }
        }
        return result;
    }

    public static ArrayList<Integer> rotateRight(ArrayList<Integer> list, int k) {
        int size = list.size();
        if (size == 0) {
            return new ArrayList<>();
        }
        int r = k % size;
        ArrayList<Integer> result = new ArrayList<>(list.subList(size - r, size));
        result.addAll(list.subList(0, size - r));
        return result;
    }
}
```

### Reasoning

* **Two-pointer merge** exploits the fact that both inputs are already sorted: repeatedly take the smaller of the two current heads. It runs in `O(n + m)` and needs no sorting. The simpler alternative, `TreeSet<Integer> set = new TreeSet<>(zoneA); set.addAll(zoneB); return new ArrayList<>(set);`, is also correct and shorter but `O((n + m) log(n + m))`. Know both.
* **The tricky condition** is `j >= zoneB.size() || (i < zoneA.size() && zoneA.get(i) <= zoneB.get(j))`. It reads: take from A when B is exhausted, or when A still has elements and its head is not larger. The order of the checks prevents an `IndexOutOfBoundsException`, because `j >= size` is tested before `zoneB.get(j)` is ever evaluated.
* **Removing duplicates while merging**: because the output is built in ascending order, any duplicate must equal the *last element added*. So a single comparison with `result.get(result.size() - 1)` is enough, and no `contains` scan is required. `result.get(...) != next` compares an `Integer` with an `int`, which unboxes, so it is a numeric comparison.
* **Do not modify the inputs**: the code only reads `zoneA` and `zoneB`, so the "do not modify" rule holds.
* **Rotation and `k % size`**: rotating by `size` returns the original list, so only `k % size` matters (Sample 2: `k = 5`, one element, so `r = 0`). Without the modulo, `size - k` becomes negative and `subList` throws `IndexOutOfBoundsException`.
* **The empty-list guard is essential**: `k % 0` throws `ArithmeticException` (Sample 3). Returning early avoids it.
* **`subList(from, to)`** is a **view** with an inclusive start and exclusive end. Copying it into a new `ArrayList` (as done here) makes the result independent. Adding to the original list while a `subList` view is alive would cause a `ConcurrentModificationException`.
* **Shortcut**: `Collections.rotate(copy, k)` does the same in place and handles large `k` and empty lists itself. Work on a copy (`new ArrayList<>(list)`) if the original must stay unchanged.
* **`k = 0`** (Sample 4) returns an identical copy: `subList(size, size)` is empty and the second `addAll` copies everything.


---

## Q15. Prime Infotech Payroll Ranking

**Collection:** ArrayList (objects + Comparator)  ·  **Level:** Medium-Hard

Prime Infotech, an IT company in Noida, wants to produce a payroll ranking report. Employee records are held in an `ArrayList<Employee>`. The `Employee` class is already provided. Develop a Java program based on the following requirements.

**Requirement 1**
Create a `PayrollManager` class containing an `ArrayList` named `employees`.
Implement:

```java
public void addEmployee(Employee employee)
```

**Requirement 2**
Implement:

```java
public List<String> getTopEarners(int n)
```

Return the **names** of the top `n` earners. Employees must be ranked by:

1. salary, **highest first**
2. name, ascending, when salaries are equal

If `n` is greater than the number of employees, return all of them. If `n` is zero or negative, return an empty list. The internal `employees` list must **not** be reordered by this call.

**Requirement 3**
Implement:

```java
public double getAverageSalary(String department)
```

Return the average salary of the employees of that department (department comparison is **case-insensitive**). If the department has no employees, return `0.0`. In that case the main method displays:

```
No employees found in <department>
```

**Restrictions**

* Edit only the `PayrollManager` class.
* Attributes must be `private`.
* Constructor and methods must be `public`.
* Do not change the specified class, attribute, or method names.
* Do not use `System.exit(0)`.
* Salaries are always greater than zero.

**Input Format**
`n`, then `n` lines of `name department salary`, then the number of top earners required, then the department to average.

**Given driver code** (already provided; do not modify)

```java
// Main.java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        PayrollManager pm = new PayrollManager();
        int n = sc.nextInt();
        for (int i = 0; i < n; i++) {
            String name = sc.next();
            String dept = sc.next();
            double salary = sc.nextDouble();
            pm.addEmployee(new Employee(name, dept, salary));
        }
        int top = sc.nextInt();
        String department = sc.next();

        System.out.println("Top earners:");
        for (String name : pm.getTopEarners(top)) {
            System.out.println(name);
        }

        double avg = pm.getAverageSalary(department);
        if (avg == 0.0) {
            System.out.println("No employees found in " + department);
        } else {
            System.out.println("Average salary of " + department + ": " + avg);
        }
    }
}
```

```java
// Employee.java
public class Employee {
    private String name;
    private String department;
    private double salary;

    public Employee(String name, String department, double salary) {
        this.name = name;
        this.department = department;
        this.salary = salary;
    }

    public String getName() {
        return name;
    }

    public String getDepartment() {
        return department;
    }

    public double getSalary() {
        return salary;
    }
}
```

**Sample Input 1**

```text
6
Kiran IT 65000
Asha HR 48000
Bala IT 72000
Deepa Finance 72000
Charu it 55000
Farah HR 39000
3
IT
```

**Sample Output 1**

```text
Top earners:
Bala
Deepa
Kiran
Average salary of IT: 64000.0
```

**Sample Input 2**

```text
6
Kiran IT 65000
Asha HR 48000
Bala IT 72000
Deepa Finance 72000
Charu it 55000
Farah HR 39000
10
Legal
```

**Sample Output 2**

```text
Top earners:
Bala
Deepa
Kiran
Charu
Asha
Farah
No employees found in Legal
```

**Sample Input 3**

```text
2
Meena Ops 30000
Meena Ops 30000
0
Ops
```

**Sample Output 3**

```text
Top earners:
Average salary of Ops: 30000.0
```

### Answer

```java
// PayrollManager.java
import java.util.*;

public class PayrollManager {
    private ArrayList<Employee> employees;

    public PayrollManager() {
        this.employees = new ArrayList<>();
    }

    public void addEmployee(Employee employee) {
        employees.add(employee);
    }

    public List<String> getTopEarners(int n) {
        List<Employee> sorted = new ArrayList<>(employees);      // copy, keep original order
        sorted.sort(Comparator.comparingDouble(Employee::getSalary).reversed()
                              .thenComparing(Employee::getName));

        List<String> names = new ArrayList<>();
        int limit = Math.min(n, sorted.size());
        for (int i = 0; i < limit; i++) {
            names.add(sorted.get(i).getName());
        }
        return names;
    }

    public double getAverageSalary(String department) {
        double total = 0;
        int count = 0;
        for (Employee e : employees) {
            if (e.getDepartment().equalsIgnoreCase(department)) {
                total += e.getSalary();
                count++;
            }
        }
        return count == 0 ? 0.0 : total / count;
    }
}
```

### Reasoning

* **Sort a copy, not the original.** `new ArrayList<>(employees)` protects the internal insertion order, as the requirement demands. Sorting `employees` directly would have the side effect of reordering it on every call.
* **Comparator chaining**: `Comparator.comparingDouble(Employee::getSalary).reversed().thenComparing(Employee::getName)` reads exactly like the rule: salary descending, then name ascending. Note that `.reversed()` applies to the comparator built *so far* (the salary comparison only), and `.thenComparing` is added after, so the name tie-break stays ascending. Calling `.reversed()` at the very end would flip the name order too and produce `Deepa` before `Bala`.
* **The lambda equivalent** (worth knowing when method references are awkward): `(a, b) -> { int c = Double.compare(b.getSalary(), a.getSalary()); return c != 0 ? c : a.getName().compareTo(b.getName()); }`. Always use `Double.compare`, not `a - b`, for doubles.
* **`Math.min(n, sorted.size())`** handles `n` larger than the list (Sample 2) without an `IndexOutOfBoundsException`, and a zero or negative `n` simply skips the loop (Sample 3). Using `sorted.subList(0, n)` would crash in both cases.
* **Average**: accumulate in a `double`, and guard against `count == 0` before dividing. In `double` arithmetic `0.0 / 0` is `NaN` rather than an exception, which would silently print `NaN`. In Sample 1 the `IT` department includes `Charu it` because the department comparison is case-insensitive, giving `(65000 + 72000 + 55000) / 3 = 64000.0`.
* **Why `0.0` works as the "none" signal**: the problem guarantees salaries above zero, so a genuine average can never be `0.0`. In a real design you would return an `OptionalDouble` or throw, but assessments often specify sentinel values like this.
* **Sort stability**: `List.sort` (and `Collections.sort`) is stable. Records that the comparator considers equal (same salary **and** same name) keep their original relative order, so the output is deterministic.


---

---

# Quick Revision: Gotchas That Assessments Love

| Collection | Handy methods | Traps to remember |
|---|---|---|
| `HashMap` | `getOrDefault`, `putIfAbsent`, `merge`, `computeIfAbsent`, `remove(key)` (returns old value), `entrySet()`, `values().removeIf(...)` | No ordering: sort a copy of keys or entries. `get` on a missing key gives `null`, so unboxing it into an `int` throws `NullPointerException`. Removing inside a for-each throws `ConcurrentModificationException`. Comparing `Integer` values with `==` is wrong above 127. |
| `HashSet` | `add` (returns `boolean`), `contains`, `retainAll` (intersection), `removeAll` (difference), `addAll` (union) | Set operations **mutate** the receiver, so copy first. Custom objects need `equals` **and** `hashCode` built from the same fields. No ordering: sort at the end or use `LinkedHashSet` / `TreeSet`. |
| `TreeMap` | `firstKey`, `lastKey`, `floorKey/Entry`, `ceilingKey/Entry`, `lowerKey`, `higherKey`, `headMap`, `tailMap`, `subMap`, `descendingMap` | Key equality comes from `compareTo` / the comparator, not `equals`. `put` on an existing key keeps the **original key object**. `firstKey()` throws on an empty map but `firstEntry()` returns `null`. `subMap` throws if `from > to`. `null` keys throw `NullPointerException`. `headMap`, `tailMap` and `subMap` are **views**. Check the inclusive/exclusive flags. |
| `ArrayList` | `add`, `get`, `set`, `remove(int)`, `remove(Object)`, `removeIf`, `subList`, `sort`, `Collections.sort/reverse/rotate/swap` | `remove(int)` vs `remove(Object)` on `List<Integer>`. Removing inside a for-each throws `ConcurrentModificationException`. `subList` is a view. `Arrays.asList` is fixed-size and `List.of` is immutable. Copy before sorting when the original order must be kept. |

## Comparator cheat sheet

```java
// descending numeric, then ascending name
list.sort(Comparator.comparingDouble(Employee::getSalary).reversed()
                    .thenComparing(Employee::getName));

// map entries: value descending, key ascending
entries.sort((a, b) -> {
    int c = b.getValue().compareTo(a.getValue());
    return c != 0 ? c : a.getKey().compareTo(b.getKey());
});

// case-insensitive ordering
new TreeMap<String, Integer>(String.CASE_INSENSITIVE_ORDER);
```

## Assessment habits

1. Read the **return type** carefully: empty list vs `null`, `String[]` vs `List<String>`, `Double` vs `double`.
2. Ask, for each string: **case-sensitive or not?** Then normalise at the boundary (`toLowerCase()` on the key) and nowhere else.
3. `HashMap` and `HashSet` give **no order**. Whenever the output must be ordered, sort explicitly or choose `TreeMap` / `TreeSet`.
4. Test the boundaries yourself: empty input, one element, all duplicates, all equal, `n` larger than the size, value exactly on the threshold.
5. Return copies of internal collections when the class must protect its own state.