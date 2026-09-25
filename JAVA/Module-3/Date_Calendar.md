# Java Competitive Exam Preparation
## Chunk 11: Dates, Calendar, and Formatting

Time to talk about handling dates in Java. Fair warning upfront: the classes covered in this chunk, `Date`, `Calendar`, `SimpleDateFormat`, are the OLD, classic Java date tools. Java 8 later introduced a much nicer `java.time` package (`LocalDate`, `LocalDateTime`, and friends), which is what you'd actually reach for in modern code. But these older classes still show up constantly in exams, older codebases, and legacy systems, so it's genuinely worth knowing them properly. We'll flag the "this is the old way" bits as we go.

Same casual style, same practice question format as the last couple chunks.

---

## Table of Contents

1. Dates and Calendar Objects
2. Date Class with Lend a Hand
3. Date Class API with Lend a Hand
4. Formatting Date in Java
5. Date Format with Lend a Hand
6. Simple Date Format with Lend a Hand
7. Lend a Hand on String to Date
8. Calendar Class with Lend a Hand
9. Consolidated Quiz
10. Revision Summary

---

## 1. Dates and Calendar Objects

So why do we even need special classes for dates? Because a date isn't just a number, it needs to handle things like "how many days are in February this year" or "what day of the week does this fall on," and doing all that math by hand every time would be a nightmare.

Java gives you two main legacy tools for this:

- **`java.util.Date`**: represents one specific instant in time, basically just a big number, milliseconds since January 1, 1970 (this moment is called "the epoch," if you ever hear that term). It doesn't really know about years, months, or days on its own, it's just a timestamp.
- **`java.util.Calendar`**: the smarter, more flexible cousin. It knows how to break a moment in time down into fields, year, month, day, hour, and lets you do date arithmetic (like "add 10 days") without you having to worry about month or year rollovers yourself.

Basically: `Date` is "a specific point in time." `Calendar` is "a toolkit for working with that point in time in calendar terms."

### Quick Check

If you just need to store "when did this event happen" and don't need to do any date math on it, which is simpler, `Date` or `Calendar`?

`Date`. It's just a timestamp, nice and simple. You'd reach for `Calendar` specifically when you need to extract fields (like "what year is this") or do arithmetic (like "what's 30 days from now").

---

## 2. Date Class with Lend a Hand

### Creating a Date

```java
Date now = new Date(); // current date and time, right now
Date specific = new Date(1735689600000L); // a specific instant, given as milliseconds since the epoch
```

Most of `Date`'s OTHER constructors, the ones that take a year, month, and day directly, are deprecated. Remember Chunk 6, where `new Integer(int)` was deprecated in favor of `Integer.valueOf()`? Same deal here, these old constructors were replaced by `Calendar`, which does a much better job of handling year/month/day construction properly.

### Printing a Date

```java
Date now = new Date();
System.out.println(now);
```

Something like: `Thu Sep 25 01:30:45 UTC 2025`

That's `Date`'s default `toString()` format, and yeah, it's not exactly pretty or flexible. That's exactly why Section 4 onward is all about formatting dates properly.

### Lend a Hand: Quiz

1. What does `new Date()` give you?
   A. Midnight, January 1, 1970  B. The current date and time, right now  C. An empty date  D. Compile error

2. What is "the epoch" in Java date terms?
   A. January 1, 1970  B. The current year  C. A random reference point  D. Nothing, made up term

3. Why are most of `Date`'s year/month/day constructors deprecated?
   A. They never worked  B. They were replaced by better tools, namely `Calendar`, similar to how `new Integer(int)` was deprecated in favor of `Integer.valueOf()`  C. `Date` itself is deprecated entirely  D. Java doesn't allow deprecated constructors to run

### Answers

1. **B.** No arguments means "give me right now," captured as a `Date` object.

2. **A.** January 1, 1970, 00:00:00 UTC. Every `Date` under the hood is really just "milliseconds since this moment."

3. **B.** Exactly the same deprecation pattern from Chunk 6. The old constructors are still technically usable, but `Calendar` does the job properly and is the recommended path instead.

### Practice Problem

Given two `Date` objects represented as milliseconds since the epoch, determine which one is earlier.

**Example 1:**

```
Input: date1Millis = 1000000, date2Millis = 2000000
Output: "date1 is earlier"
```

**Example 2:**

```
Input: date1Millis = 5000000, date2Millis = 5000000
Output: "same instant"
```

### Solution

```java
import java.util.Date;

public class CompareDates {
    static String compareDates(long millis1, long millis2) {
        Date date1 = new Date(millis1);
        Date date2 = new Date(millis2);

        if (date1.before(date2)) {
            return "date1 is earlier";
        } else if (date1.after(date2)) {
            return "date2 is earlier";
        } else {
            return "same instant";
        }
    }

    public static void main(String[] args) {
        System.out.println(compareDates(1000000, 2000000));
        System.out.println(compareDates(5000000, 5000000));
    }
}
```

Why this works: `Date` gives you `before()` and `after()` right out of the box for exactly this kind of comparison, no need to manually compare the raw millisecond numbers yourself.

---

## 3. Date Class API with Lend a Hand

### The Full Toolkit

| Method | What it does |
|---|---|
| `getTime()` | returns the millisecond value as a `long` |
| `setTime(long)` | changes this `Date` to represent a different instant |
| `before(Date other)` | true if this date comes before `other` |
| `after(Date other)` | true if this date comes after `other` |
| `equals(Object other)` | true if both represent the exact same instant |
| `compareTo(Date other)` | negative, zero, or positive, same idea as `Integer.compare` from Chunk 6 |

`Date` actually implements `Comparable<Date>`, which is why `compareTo` works. That's the same interface idea from Chunk 4, a contract saying "objects of this type know how to compare themselves to each other."

### A Realistic Use: Checking Subscription Expiry

Remember Chunk 8's `Subscription` class, which just used a plain `boolean active` flag? A real system would actually store an expiry date instead, and compute "is it active" by comparing that date to right now.

```java
import java.util.Date;

public class SubscriptionCheck {
    static boolean isExpired(Date expiryDate) {
        Date now = new Date();
        return expiryDate.before(now);
    }

    public static void main(String[] args) {
        Date pastDate = new Date(1000000000000L); // way back in 2001
        System.out.println("Expired: " + isExpired(pastDate));
    }
}
```

### Lend a Hand: Quiz

1. What does `date1.compareTo(date2)` return if `date1` is earlier than `date2`?
   A. `0`  B. A positive number  C. A negative number  D. Throws an exception

2. What interface does `Date` implement that gives it `compareTo()`?
   A. `Serializable` only  B. `Comparable<Date>`  C. `Iterable`  D. None, it's built in specially

3. What does `getTime()` return?
   A. A `String` of the current time  B. A `long`, milliseconds since the epoch  C. A `Calendar` object  D. `void`

### Answers

1. **C.** Just like `Integer.compare` and `Double.compare` from Chunk 6, `compareTo` follows the same negative/zero/positive convention.

2. **B.** `Comparable<Date>` is the interface contract, from Chunk 4's interface coverage, that gives every `Date` object the ability to compare itself against another.

3. **B.** It's the raw underlying value every `Date` is built on, milliseconds since the epoch, as a `long`.

### Practice Problem

Given a subscription's expiry date and a "today" date, both as milliseconds since the epoch, determine if the subscription is expired or still active.

**Example 1:**

```
Input: expiryMillis = 1000000, todayMillis = 2000000
Output: "Expired"
```

**Example 2:**

```
Input: expiryMillis = 5000000, todayMillis = 2000000
Output: "Active"
```

### Solution

```java
import java.util.Date;

public class SubscriptionStatus {
    static String checkStatus(long expiryMillis, long todayMillis) {
        Date expiry = new Date(expiryMillis);
        Date today = new Date(todayMillis);
        return expiry.before(today) ? "Expired" : "Active";
    }

    public static void main(String[] args) {
        System.out.println(checkStatus(1000000, 2000000));
        System.out.println(checkStatus(5000000, 2000000));
    }
}
```

Why this works: same `before()` trick as the last section, we're just wrapping it in a friendlier method name that matches a real business rule.

---

## 4. Formatting Date in Java

`Date.toString()` gives you that clunky `Thu Sep 25 01:30:45 UTC 2025` format, and honestly, real apps almost never want that exact look. You want to CONTROL the format, maybe `25/09/2025`, maybe `September 25, 2025`, maybe `2025-09-25`.

Java gives you two classes for this, and they're related exactly the way you'd expect from Chunk 4:

- **`java.text.DateFormat`**: an abstract class. It defines the general idea of "something that can format a date into text, and parse text back into a date," but doesn't nail down exactly how.
- **`java.text.SimpleDateFormat`**: extends `DateFormat`, and is the concrete class you'll actually use almost every time. It lets you define your OWN custom pattern for exactly how dates should look.

That's a real, genuine abstract-class-and-concrete-subclass pair straight out of the JDK, same relationship as `Content` and `Movie` from Chunk 8, or `Number` and `Integer` from Chunk 6.

### Quick Check

Why can't you just do `new DateFormat()` directly?

Because `DateFormat` is abstract, exactly the Chunk 4 rule, abstract classes can never be instantiated directly. You either use one of its provided static factory methods, or reach for a concrete subclass like `SimpleDateFormat`.

---

## 5. Date Format with Lend a Hand

### Using DateFormat's Built-In Styles

`DateFormat` gives you ready-made formatting styles through static factory methods, similar spirit to `Calendar.getInstance()` coming up in Section 8, a factory method handing you back a ready-to-use object.

```java
import java.text.DateFormat;
import java.util.Date;

DateFormat shortForm = DateFormat.getDateInstance(DateFormat.SHORT);
DateFormat longForm = DateFormat.getDateInstance(DateFormat.LONG);

Date now = new Date();
System.out.println(shortForm.format(now)); // something like 9/25/25
System.out.println(longForm.format(now));  // something like September 25, 2025
```

The exact output depends on your system's locale, but the general idea is: `SHORT`, `MEDIUM`, `LONG`, and `FULL` give you increasingly detailed, pre-built styles, without you needing to define your own pattern.

### Lend a Hand: Quiz

1. Can you write `new DateFormat()` directly?
   A. Yes  B. No, `DateFormat` is abstract  C. Only with a `String` argument  D. Only inside `main`

2. What do `DateFormat.SHORT`, `MEDIUM`, `LONG`, and `FULL` represent?
   A. Time zones  B. Pre-built formatting detail levels  C. Error codes  D. Date ranges

3. What does `DateFormat.getDateInstance()` give you back?
   A. A `Date` object  B. A ready-to-use `DateFormat` object you can call `.format()` on  C. A `String`  D. Compile error

### Answers

1. **B.** Straight from Chunk 4's abstract class rules, no direct instantiation allowed.

2. **B.** They're constants representing how much detail the formatted output should include, from a brief `SHORT` style up to a very descriptive `FULL` style.

3. **B.** It's a factory method, similar spirit to the factory methods from Chunk 3, handing you a ready-made formatter object rather than a formatted `String` directly.

---

## 6. Simple Date Format with Lend a Hand

### Building Your Own Pattern

This is the one you'll actually use constantly. `SimpleDateFormat` lets you define an exact custom pattern using specific letters.

| Letter | Means |
|---|---|
| `yyyy` | 4-digit year |
| `MM` | month (01-12) |
| `dd` | day of month (01-31) |
| `HH` | hour, 24-hour format (00-23) |
| `mm` | minute (00-59) |
| `ss` | second (00-59) |
| `EEE` | day of week, short form (Mon, Tue...) |
| `a` | AM/PM marker |

```java
import java.text.SimpleDateFormat;
import java.util.Date;

SimpleDateFormat sdf = new SimpleDateFormat("dd-MM-yyyy");
Date now = new Date();
System.out.println(sdf.format(now)); // e.g. 25-09-2025
```

### The Classic Trap: MM vs mm

This one gets EVERYONE at least once. `MM` (capital) means month. `mm` (lowercase) means minute. Mix them up, and your date comes out looking completely wrong, but it still compiles and runs just fine, no error at all, just silently wrong output.

```java
SimpleDateFormat wrong = new SimpleDateFormat("dd-mm-yyyy"); // oops, lowercase mm
SimpleDateFormat right = new SimpleDateFormat("dd-MM-yyyy"); // correct, capital MM
```

This is basically the date-formatting version of Chunk 1's case sensitivity lesson, Java cares about capitalization everywhere, and pattern letters are no exception.

### Lend a Hand: Quiz

1. What does the pattern `"yyyy/MM/dd"` produce for September 25, 2025?
   A. `25/09/2025`  B. `2025/09/25`  C. `09/25/2025`  D. `2025-09-25`

2. What's the difference between `MM` and `mm` in a pattern?
   A. No difference  B. `MM` is month, `mm` is minute, totally different fields  C. Both mean month  D. Both mean minute

3. If you accidentally use `mm` instead of `MM` for the month, what happens when you compile and run?
   A. Compile error  B. Runtime exception  C. Compiles and runs fine, but the output is silently wrong  D. `SimpleDateFormat` auto-corrects it

### Answers

1. **B.** `yyyy` gives the 4-digit year first, then `MM` the month, then `dd` the day, exactly matching the pattern's order: `2025/09/25`.

2. **B.** Capital `MM` is month, lowercase `mm` is minute. Totally unrelated fields that just happen to share a letter with different casing.

3. **C.** No error at all, it's a silent logic bug, which honestly makes it worse. Always double check your pattern letters carefully.

### Practice Problem

Given a date string in `dd-MM-yyyy` format, convert it into `yyyy/MM/dd` format.

**Example 1:**

```
Input: dateStr = "25-12-2025"
Output: "2025/12/25"
```

**Example 2:**

```
Input: dateStr = "01-01-2000"
Output: "2000/01/01"
```

### Solution

```java
import java.text.SimpleDateFormat;
import java.text.ParseException;
import java.util.Date;

public class ReformatDate {
    static String reformat(String dateStr) throws ParseException {
        SimpleDateFormat inputFormat = new SimpleDateFormat("dd-MM-yyyy");
        SimpleDateFormat outputFormat = new SimpleDateFormat("yyyy/MM/dd");

        Date date = inputFormat.parse(dateStr);
        return outputFormat.format(date);
    }

    public static void main(String[] args) throws ParseException {
        System.out.println(reformat("25-12-2025"));
        System.out.println(reformat("01-01-2000"));
    }
}
```

Why this works: this is the classic "parse then format" combo, use one `SimpleDateFormat` (matching the INPUT pattern) to turn the `String` into an actual `Date` object, then use a second `SimpleDateFormat` (matching the OUTPUT pattern you want) to turn that same `Date` back into text, just shaped differently. Note the `throws ParseException` on `main`, we'll get into exactly why that's required in the next section.

---

## 7. Lend a Hand on String to Date

### Parsing: Text Going INTO a Date

`format()` goes Date-to-String. `parse()` goes the other way, String-to-Date. And here's the important bit: `parse()` throws `ParseException` if the text doesn't actually match the expected pattern, and it's a CHECKED exception, straight from Chunk 8's classification.

```java
import java.text.SimpleDateFormat;
import java.text.ParseException;
import java.util.Date;

SimpleDateFormat sdf = new SimpleDateFormat("dd-MM-yyyy");

try {
    Date parsed = sdf.parse("25-12-2025");
    System.out.println(parsed);
} catch (ParseException e) {
    System.out.println("That doesn't look like a valid date: " + e.getMessage());
}
```

Since `ParseException` is checked, you MUST either catch it (like above) or declare `throws ParseException` on your enclosing method, exactly the compiler enforcement Chunk 8 covered. No skipping this one.

### Lend a Hand: Quiz

1. Is `ParseException` checked or unchecked?
   A. Unchecked, like `NumberFormatException`  B. Checked, must be caught or declared  C. Neither, it's a warning  D. It doesn't exist

2. What happens if you call `sdf.parse("not a date")` with no `try-catch` and no `throws` declaration?
   A. Compiles fine, throws at runtime  B. Compile error  C. Returns `null`  D. Returns today's date

3. What's the correct order for the "reformat a date" trick?
   A. Format first, then parse  B. Parse with the input's pattern first, getting a `Date`, then format that `Date` with the output pattern  C. You can't reformat dates  D. Parse and format do the same thing

### Answers

1. **B.** Unlike `NumberFormatException` from Chunk 6 (which is unchecked), `ParseException` is genuinely checked, so the compiler enforces handling it.

2. **B.** Since it's checked and neither caught nor declared, this fails to compile, exactly the enforcement rule from Chunk 8.

3. **B.** Parse the original string (using a format matching HOW it's currently written) into an actual `Date` object first, then format that `Date` using whatever NEW pattern you want for the output.

### Practice Problem

Given an array of date strings, all in `dd-MM-yyyy` format, find the earliest date among them and return it as a `String` in the same format.

**Example 1:**

```
Input: dates = ["25-12-2025", "01-01-2020", "15-06-2023"]
Output: "01-01-2020"
```

### Solution

```java
import java.text.SimpleDateFormat;
import java.text.ParseException;
import java.util.Date;

public class EarliestDate {
    static String findEarliest(String[] dates) throws ParseException {
        SimpleDateFormat sdf = new SimpleDateFormat("dd-MM-yyyy");
        Date earliest = sdf.parse(dates[0]);
        String earliestStr = dates[0];

        for (String dateStr : dates) {
            Date current = sdf.parse(dateStr);
            if (current.before(earliest)) {
                earliest = current;
                earliestStr = dateStr;
            }
        }
        return earliestStr;
    }

    public static void main(String[] args) throws ParseException {
        String[] dates = {"25-12-2025", "01-01-2020", "15-06-2023"};
        System.out.println(findEarliest(dates));
    }
}
```

Why this works: each date string gets parsed into a real `Date` object, and from there it's just a plain for-each loop from Chunk 1, comparing with `before()` exactly like Section 2's practice problem, keeping track of both the earliest `Date` found and its original `String` form to return.

---

## 8. Calendar Class with Lend a Hand

### Getting a Calendar

You never construct a `Calendar` directly with `new`, because, exactly like `DateFormat`, `Calendar` is abstract. Instead, you use its factory method.

```java
import java.util.Calendar;

Calendar cal = Calendar.getInstance(); // hands you back a ready-to-use, concrete Calendar object
```

`getInstance()` is exactly the same factory method pattern from Chunk 3, it builds and returns a concrete object (technically a `GregorianCalendar`, the standard implementation) without you needing to know or care about that concrete class name.

### Reading Fields

```java
Calendar cal = Calendar.getInstance();
int year = cal.get(Calendar.YEAR);
int month = cal.get(Calendar.MONTH);       // WATCH OUT, this is 0-indexed!
int day = cal.get(Calendar.DAY_OF_MONTH);

System.out.println(year + " " + month + " " + day);
```

### The Big Trap: Calendar.MONTH is 0-Indexed

`Calendar.JANUARY` is `0`. `Calendar.DECEMBER` is `11`. So if `cal.get(Calendar.MONTH)` gives you `8`, that's actually September, not August. This trips up SO many people. It's basically the exact same "arrays start at 0" trap from Chunk 1 and Chunk 10, just wearing a calendar costume this time.

```java
Calendar cal = Calendar.getInstance();
cal.set(2025, Calendar.DECEMBER, 25); // setting December 25, 2025
// notice we used the constant Calendar.DECEMBER (which equals 11) rather than typing 12
```

Always use the named constants (`Calendar.JANUARY`, `Calendar.DECEMBER`, and so on) instead of typing raw numbers for the month, it completely sidesteps the confusion.

### Doing Date Math with add()

```java
Calendar cal = Calendar.getInstance();
cal.set(2025, Calendar.DECEMBER, 28);
cal.add(Calendar.DAY_OF_MONTH, 5); // add 5 days

int year = cal.get(Calendar.YEAR);
int month = cal.get(Calendar.MONTH);
int day = cal.get(Calendar.DAY_OF_MONTH);
System.out.println(year + "-" + (month + 1) + "-" + day); // 2026-1-2
```

Notice `add()` correctly handled rolling over into January of the NEXT year all on its own, you didn't have to manually check "does this cross a month or year boundary." That automatic rollover handling is really the whole reason `Calendar` exists.

### Lend a Hand: Quiz

1. Why can't you write `new Calendar()`?
   A. You can  B. `Calendar` is abstract, exactly like `DateFormat`  C. It requires a `String` argument  D. `Calendar` doesn't exist

2. What does `Calendar.getInstance()` return?
   A. A `Date` object  B. A ready-to-use, concrete `Calendar` object  C. A `String`  D. `null`

3. If `cal.get(Calendar.MONTH)` returns `0`, what month is that?
   A. January  B. December  C. Invalid, months start at 1  D. February

4. What's the advantage of `cal.add(Calendar.DAY_OF_MONTH, 5)` over manually adding 5 to a day number yourself?
   A. No real advantage  B. It automatically handles rolling over into the next month or year correctly  C. It's just shorter to type  D. It only works for single-digit additions

### Answers

1. **B.** Same abstract class rule from Chunk 4, applied to `Calendar` this time, no direct instantiation.

2. **B.** It's a factory method that hands back a working, concrete `Calendar` object, ready to use immediately.

3. **A.** `Calendar.MONTH` is 0-indexed, so `0` is January, not February and definitely not "invalid." This is THE classic `Calendar` gotcha.

4. **B.** Manually adding days yourself means you'd have to check "did this cross into a new month, does that month have 30 or 31 days, did THAT cross into a new year," `add()` handles every bit of that automatically.

### Practice Problem

Given a person's birth year, birth month (1-12, human-friendly), and birth day, along with a "today" reference date (also year, month 1-12, day), calculate their age in completed years.

**Example 1:**

```
Input: birthYear = 2000, birthMonth = 6, birthDay = 15, todayYear = 2025, todayMonth = 6, todayDay = 14
Output: 24
Explanation: Their birthday this year hasn't happened yet (it's tomorrow), so they're still 24.
```

**Example 2:**

```
Input: birthYear = 2000, birthMonth = 6, birthDay = 15, todayYear = 2025, todayMonth = 6, todayDay = 15
Output: 25
Explanation: Today IS their birthday, so they just turned 25.
```

**Example 3:**

```
Input: birthYear = 2000, birthMonth = 6, birthDay = 15, todayYear = 2025, todayMonth = 1, todayDay = 1
Output: 24
Explanation: We're still early in the year, before their June birthday, so they haven't turned 25 yet.
```

### Solution

```java
public class AgeCalculator {
    static int calculateAge(int birthYear, int birthMonth, int birthDay,
                             int todayYear, int todayMonth, int todayDay) {
        int age = todayYear - birthYear;

        boolean birthdayNotYetHappened =
                (todayMonth < birthMonth) ||
                (todayMonth == birthMonth && todayDay < birthDay);

        if (birthdayNotYetHappened) {
            age--;
        }
        return age;
    }

    public static void main(String[] args) {
        System.out.println(calculateAge(2000, 6, 15, 2025, 6, 14));
        System.out.println(calculateAge(2000, 6, 15, 2025, 6, 15));
        System.out.println(calculateAge(2000, 6, 15, 2025, 1, 1));
    }
}
```

Why this works: the naive answer is just `todayYear - birthYear`, but that's wrong whenever this year's birthday hasn't actually happened yet. So we check: has today's month already passed the birth month, or if we're in the exact same month, has today's day already reached the birth day? If NEITHER of those is true yet, the birthday's still coming up this year, so we subtract 1 from the naive answer. This deliberately avoids using `Calendar.MONTH`'s 0-indexing at all, by keeping everything in plain human-friendly 1-12 month numbers throughout, side-stepping that whole trap entirely for this particular calculation.

---

## 9. Consolidated Quiz

Tying this back into earlier chunks too.

1. `DateFormat` is abstract, `SimpleDateFormat` extends it. Which chunk first taught the general rule that abstract classes can't be instantiated directly?
   A. Chunk 2  B. Chunk 3  C. Chunk 4  D. Chunk 5

2. `Calendar.getInstance()` hands you a ready-made object without you calling `new` yourself. Which chunk introduced this exact factory method idea?
   A. Chunk 1  B. Chunk 3  C. Chunk 6  D. Chunk 8

3. `ParseException` must be caught or declared with `throws`. Which chunk explained checked exceptions in full?
   A. Chunk 5  B. Chunk 6  C. Chunk 7  D. Chunk 8

4. `Date` implements `Comparable<Date>`, giving it `compareTo()`. Which chunk covered interfaces as contracts?
   A. Chunk 2  B. Chunk 3  C. Chunk 4  D. Chunk 5

5. The `MM` vs `mm` mixup in `SimpleDateFormat` patterns is fundamentally the same kind of mistake as what earlier, more general Java rule?
   A. Integer overflow  B. Java being case sensitive, from Chunk 1  C. Access modifiers  D. Autoboxing

6. `Calendar.MONTH` being 0-indexed is basically the calendar version of which earlier concept?
   A. String immutability  B. Arrays starting at index 0, from Chunk 1 and Chunk 10  C. Static fields  D. Method overloading

7. `Date`'s old year/month/day constructors are deprecated. Which chunk first showed a deprecated constructor being replaced by a recommended alternative?
   A. Chunk 4  B. Chunk 5  C. Chunk 6  D. Chunk 7

8. In the age calculator practice problem, why did the solution avoid using `Calendar` entirely and just take plain `int` parameters instead?
   A. `Calendar` cannot represent birthdays  B. To keep the logic simple and sidestep the 0-indexed month trap, plain human-friendly 1-12 numbers are less error-prone for this specific calculation  C. `Calendar` is deprecated  D. It's impossible to compare `Calendar` objects

### Consolidated Quiz Answers

1. **C.** Chunk 4 laid out the abstract class rule, and this chunk gave you two brand new, genuine real-world examples of it in the JDK itself, `DateFormat` and `Calendar` both.

2. **B.** Chunk 3 introduced factory methods, a `static` method that builds and returns an object without the caller needing to use `new` directly, exactly what `Calendar.getInstance()` and `DateFormat.getDateInstance()` both do.

3. **D.** Chunk 8 covered the full checked-versus-unchecked distinction, and `ParseException` is a clean, real-world example of a checked exception you're guaranteed to run into.

4. **C.** Chunk 4 covered interfaces as contracts that unrelated classes can implement, and `Comparable` is exactly that kind of contract, giving `Date` its `compareTo()` method.

5. **B.** Chunk 1 established that Java cares about capitalization everywhere, and `SimpleDateFormat` pattern letters are a very real, very common place that rule bites people.

6. **B.** Same zero-based counting idea from arrays, just applied to months instead of array slots, and just as easy to trip over if you're not expecting it.

7. **C.** Chunk 6 covered `new Integer(int)` being deprecated in favor of `Integer.valueOf()`, the exact same kind of "old constructor replaced by a better tool" story as `Date`'s deprecated constructors being replaced by `Calendar`.

8. **B.** Keeping months as plain, ordinary 1-12 numbers sidesteps the whole `Calendar.MONTH` 0-indexing trap entirely for this specific calculation, since there's no actual need to build a full `Calendar` object just to compare month and day numbers.

---

## 10. Revision Summary

| Concept | Key Point | Where It Showed Up |
|---|---|---|
| Date | A single instant in time, milliseconds since the epoch (Jan 1, 1970) | `new Date()` |
| Date's deprecated constructors | Old year/month/day constructors, replaced by `Calendar` | Same deprecation pattern as Chunk 6's `Integer` |
| Date API | `before()`, `after()`, `compareTo()`, `getTime()` | Comparing subscription expiry dates |
| DateFormat | Abstract class, defines the "format and parse" contract | Can't `new DateFormat()` |
| SimpleDateFormat | Concrete subclass, lets you define a custom pattern | `new SimpleDateFormat("dd-MM-yyyy")` |
| MM vs mm | Capital is month, lowercase is minute, silently wrong if swapped | Classic case-sensitivity trap |
| String to Date | `.parse()` throws checked `ParseException` | Must catch or declare `throws` |
| Calendar | Abstract class, factory method `getInstance()`, field-based access | `cal.get(Calendar.YEAR)` |
| Calendar.MONTH | 0-indexed, January is `0`, December is `11` | Always use named constants, not raw numbers |
| Calendar.add() | Handles month/year rollovers automatically | Adding days across a year boundary |

**Quick tip for the exam:** any question showing `SimpleDateFormat` output that looks "off" is almost always testing `MM` vs `mm`. And any question involving `Calendar.get(Calendar.MONTH)` giving a weirdly-off-by-one-looking value is testing whether you remember it's 0-indexed. Those two traps alone cover a huge chunk of what gets asked about this topic.