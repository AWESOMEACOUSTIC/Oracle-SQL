# Java Competitive Exam Preparation
## Chunk 9: Collections Framework, Generics, and Loops Revisited

Okay, new topic, and it's a big one: **Collections**. If you've been using arrays this whole time and wondering "isn't there something better than this," yes, there is, and that's exactly what this chunk is about.

We'll keep things simple and casual here. No need to overcomplicate this stuff, it's actually pretty intuitive once you see it in action. And since this is the kind of thing that shows up a LOT in coding assessments (the kind companies like Cognizant use to test you), most of the practice questions in this chunk are written like real coding problems, with a proper problem statement, examples, and constraints, so you get comfortable with that format too.

---

## Table of Contents

1. What are Collections
2. Collections Framework and its Benefits
3. Collections Interfaces
4. Collection Framework Components
5. List Interface with Lend a Hand
6. Set Interface with Lend a Hand on HashSet
7. What is Generics
8. Using Generics with Collections
9. Lend a Hand on Generics
10. Lend a Hand on the for Loop
11. For-Each Loop with Lend a Hand
12. Iterator with Lend a Hand
13. Map Interface with Lend a Hand on HashMap and TreeMap
14. Lend a Hand on Adding User Defined Objects to a Collection
15. Consolidated Quiz
16. Revision Summary

---

## 1. What are Collections

So you already know arrays. Arrays are great, but they have a big annoying limitation: once you say `int[] arr = new int[5];`, that's it, it's stuck at size 5 forever. Want to add a 6th item? Too bad, you have to make a whole new array and copy everything over.

A **Collection** is basically Java's answer to "I want a group of things, but I don't want to deal with fixed sizes." Think of it like a shopping cart. You don't decide upfront "I will buy exactly 7 items today." You just keep adding stuff, and if you change your mind, you remove stuff. That's a collection.

In plain terms: a Collection is an object that holds a bunch of other objects (called elements), and it knows how to add, remove, and go through them, without you needing to worry about resizing anything yourself.

```java
List<String> watchlist = new ArrayList<>();
watchlist.add("Ocean Deep");
watchlist.add("Skyline");
watchlist.add("Cosmic Journey");
System.out.println(watchlist);
```

Output: `[Ocean Deep, Skyline, Cosmic Journey]`

No size limit, no manual resizing, and it even prints nicely by default. That's the whole pitch.

### Quick Check

Why not just use a bigger array from the start, like `int[100]`, so you never run out of space?

Because you'd be wasting memory if you don't use all 100 slots, and you'd still break if someone needs a 101st item. Collections grow exactly as needed, no guessing required.

---

## 2. Collections Framework and its Benefits

The **Collections Framework** is just the name for the whole family of ready-made classes and interfaces Java gives you for storing groups of objects: `List`, `Set`, `Map`, and all their different flavors like `ArrayList`, `HashSet`, `HashMap`, and so on. It's all sitting in `java.util`, ready to use.

Here's why it's genuinely useful, not just "nice to have":

- **You don't reinvent the wheel.** Need a resizable list? Don't write your own. `ArrayList` already exists, already works, already got tested by millions of people before you.
- **Built in methods for common stuff.** Sorting, searching, checking if something exists, removing duplicates, all of it has ready-made methods.
- **Consistent way of doing things.** Once you know how `List` works, `Set` feels familiar too, because they share a lot of the same method names and patterns.
- **Performance choices.** Need fast lookups? Use a `HashSet`. Need things sorted automatically? Use a `TreeSet`. The framework gives you options depending on what you actually need.

### Quick Check

If arrays already exist, why would anyone bother learning the whole Collections Framework?

Because arrays are rigid (fixed size) and don't come with built in tools like "remove this specific item" or "check if this value already exists." Collections do all that for you, out of the box.

---

## 3. Collections Interfaces

At the top of the framework, there's a bunch of interfaces. Remember interfaces from Chunk 4, just a contract, no actual implementation? Same idea here.

Here's the basic family tree:

```
Collection (interface)
 ├── List   (ordered, allows duplicates)
 ├── Set    (no duplicates)
 └── Queue  (first-in-first-out style)

Map (interface, separate family, not a Collection)
```

Quick note: `Map` is technically NOT part of the `Collection` interface family. It stores key-value pairs instead of just single elements, so it lives in its own little branch. We'll get to it in Section 13.

- **`List`**: an ordered group where duplicates are totally fine, and you can access things by index, just like an array, but resizable.
- **`Set`**: a group where duplicates are NOT allowed. Adding the same thing twice? It just gets ignored.
- **`Queue`**: stuff waiting in line, first one in is usually the first one out.

### Quick Check

If you tried to add `"Ocean Deep"` twice to a `Set`, what would happen?

Nothing bad, it just wouldn't add it a second time. Sets automatically block duplicates.

---

## 4. Collection Framework Components

Basically, the whole framework has three kinds of pieces:

1. **Interfaces** — the contracts: `List`, `Set`, `Map`, `Queue`.
2. **Implementations** — the actual classes that do the work: `ArrayList`, `LinkedList`, `HashSet`, `TreeSet`, `HashMap`, `TreeMap`.
3. **Algorithms** — helper methods, mostly static ones sitting in a class called `Collections` (yes, confusingly similar name), like `Collections.sort(list)` or `Collections.reverse(list)`.

| Interface | Common Implementations |
|---|---|
| `List` | `ArrayList`, `LinkedList` |
| `Set` | `HashSet`, `TreeSet`, `LinkedHashSet` |
| `Map` | `HashMap`, `TreeMap`, `LinkedHashMap` |

You'll mostly stick to `ArrayList`, `HashSet`, and `HashMap` for everyday stuff. The others show up when you need something specific, like sorted order.

### Quick Check

What's the difference between an interface like `List` and an implementation like `ArrayList`?

`List` is just the plan, the "what it should be able to do." `ArrayList` is the actual working class that does it. You write `List<String> x = new ArrayList<>();` because you're declaring the type as the interface, but building the real object from the implementation.

---

## 5. List Interface with Lend a Hand

### The Basics

A `List` is your go-to when order matters and duplicates are okay. Two popular flavors:

- **`ArrayList`**: backed by an array internally, super fast for reading by index, a bit slower for inserting/removing in the middle.
- **`LinkedList`**: backed by a chain of nodes, faster for inserting/removing at the start or middle, a bit slower for random access.

Honestly, for most everyday use, `ArrayList` is the default choice.

```java
List<String> movies = new ArrayList<>();
movies.add("Skyline");
movies.add("Ocean Deep");
movies.add("Skyline"); // duplicates? totally fine

System.out.println(movies.get(0));      // Skyline
System.out.println(movies.size());      // 3
System.out.println(movies.contains("Ocean Deep")); // true

movies.remove("Skyline"); // removes the FIRST match only
System.out.println(movies);
```

Output:
```
Skyline
3
true
[Ocean Deep, Skyline]
```

Notice `remove("Skyline")` only killed the first one it found, the second `"Skyline"` is still hanging around.

### Handy List Methods

| Method | What it does |
|---|---|
| `add(item)` | adds to the end |
| `add(index, item)` | inserts at a specific spot |
| `get(index)` | grabs the item at that index |
| `set(index, item)` | replaces the item at that index |
| `remove(index)` or `remove(item)` | removes by position or by value |
| `size()` | how many items total |
| `contains(item)` | true or false, is it in there |
| `indexOf(item)` | where is it (or -1 if not found) |
| `isEmpty()` | true if size is 0 |

### Lend a Hand: Quiz

1. What does `list.remove(2)` do if `list` is `List<Integer>` containing `[10, 20, 30, 40]`?
   A. Removes the value `2`  B. Removes whatever is at index `2`, which is `30`  C. Removes the last item  D. Compile error

2. Can a `List` hold the same value more than once?
   A. No, never  B. Yes, duplicates are allowed  C. Only for `String`  D. Only if sorted

3. What's the go-to choice between `ArrayList` and `LinkedList` for everyday use?
   A. `LinkedList`, always  B. `ArrayList`, usually  C. They're identical  D. Neither, use arrays

### Answers

1. **B.** For a `List<Integer>`, `remove(int index)` is overloaded to take the index, so `remove(2)` removes whatever sits at position `2`, not the value `2` itself. This is honestly a classic gotcha, worth remembering.

2. **B.** `List` is totally fine with duplicates, that's actually one of its defining traits versus `Set`.

3. **B.** `ArrayList` is the practical default for most situations, quick lookups by index, decent overall performance.

### Practice Problem

Given an array of integers, find the second largest distinct number in it.

**Example 1:**

```
Input: nums = [10, 20, 4, 45, 99]
Output: 45
Explanation: The largest is 99, and the next distinct value below it is 45.
```

**Example 2:**

```
Input: nums = [5, 5, 5, 5]
Output: -1
Explanation: There is no second distinct value since every number is the same.
```

**Constraints:**

* `1 <= nums.length <= 1000`
* `-10^6 <= nums[i] <= 10^6`

### Solution

```java
import java.util.List;
import java.util.ArrayList;

public class SecondLargest {
    static int secondLargest(int[] nums) {
        List<Integer> distinct = new ArrayList<>();
        for (int n : nums) {
            if (!distinct.contains(n)) {
                distinct.add(n);
            }
        }

        if (distinct.size() < 2) {
            return -1;
        }

        int largest = Integer.MIN_VALUE;
        int second = Integer.MIN_VALUE;

        for (int val : distinct) {
            if (val > largest) {
                second = largest;
                largest = val;
            } else if (val > second) {
                second = val;
            }
        }

        return second;
    }

    public static void main(String[] args) {
        System.out.println(secondLargest(new int[]{10, 20, 4, 45, 99}));
        System.out.println(secondLargest(new int[]{100, 50, 30}));
        System.out.println(secondLargest(new int[]{5, 5, 5, 5}));
    }
}
```

Why this works: we first build a `List` of distinct values using `contains()` as our duplicate filter, super handy method to have on hand. Then we do one pass, starting both `largest` and `second` as low as possible, so every real value naturally slots in correctly, even when the biggest number shows up first in the list. No sorting needed, just one clean pass.

A quick word on why we start both at `Integer.MIN_VALUE` instead of grabbing `distinct.get(0)` as a head start: if you seed `largest` with the first element and then still loop over that same first element again, you can accidentally compare a value against itself and let it slip into `second` by mistake. Starting both trackers below any possible real value sidesteps that trap entirely.

---

## 6. Set Interface with Lend a Hand on HashSet

### The Basics

A `Set` is like a `List` that just refuses duplicates. Try adding the same thing twice, and it politely ignores the second attempt.

The most common flavor is `HashSet`. It's crazy fast for checking "does this exist?" (technically constant time on average), but it does NOT keep things in the order you added them. If you need order, there's `LinkedHashSet` (keeps insertion order) or `TreeSet` (keeps sorted order).

```java
Set<String> genres = new HashSet<>();
genres.add("Action");
genres.add("Drama");
genres.add("Action"); // ignored, already there

System.out.println(genres.size());        // 2
System.out.println(genres.contains("Drama")); // true
genres.remove("Drama");
System.out.println(genres);
```

Output:
```
2
true
[Action]
```

### Why HashSet is so Fast at Lookups

Here's the thing that makes `HashSet` special: it uses `hashCode()` behind the scenes (remember Chunk 5?) to figure out roughly where to store each item, so checking "is this already in here" doesn't need to check every single element one by one, like `List.contains()` would. That's exactly why `HashSet` is the go-to tool when you just need to check membership fast.

### Lend a Hand: Quiz

1. What happens when you `add()` the same value twice to a `HashSet`?
   A. It gets added twice  B. Second one is silently ignored  C. Compile error  D. Runtime exception

2. Does `HashSet` keep items in the order you added them?
   A. Yes, always  B. No, order isn't guaranteed  C. Only for numbers  D. Only if sorted first

3. Why is `HashSet.contains()` generally faster than `ArrayList.contains()` for large amounts of data?
   A. It's not actually faster  B. `HashSet` uses hashing to jump close to where an item should be, instead of checking every element one at a time  C. `HashSet` only holds numbers  D. `ArrayList` doesn't have `contains()`

### Answers

1. **B.** Sets are all about "no repeats." The duplicate attempt just quietly does nothing.

2. **B.** `HashSet` doesn't promise any particular order. If order matters, reach for `LinkedHashSet` or `TreeSet` instead.

3. **B.** This is the whole point of hashing, from Chunk 5's `hashCode()` discussion. It lets `HashSet` skip straight to roughly the right spot instead of scanning everything.

### Practice Problem

Given an array of integers `nums`, return `true` if any value appears at least twice in the array, and `false` if every element is distinct.

**Example 1:**

```
Input: nums = [1, 2, 3, 1]
Output: true
Explanation: 1 appears twice.
```

**Example 2:**

```
Input: nums = [1, 2, 3, 4]
Output: false
Explanation: Every value shows up exactly once.
```

**Example 3:**

```
Input: nums = [1, 1, 1, 3, 3, 4, 3, 2, 4, 2]
Output: true
```

**Constraints:**

* `1 <= nums.length <= 10^5`
* `-10^9 <= nums[i] <= 10^9`

### Solution

```java
import java.util.HashSet;
import java.util.Set;

public class ContainsDuplicate {
    static boolean containsDuplicate(int[] nums) {
        Set<Integer> seen = new HashSet<>();
        for (int n : nums) {
            if (!seen.add(n)) {
                return true; // add() returns false if it was ALREADY there
            }
        }
        return false;
    }

    public static void main(String[] args) {
        System.out.println(containsDuplicate(new int[]{1, 2, 3, 1}));
        System.out.println(containsDuplicate(new int[]{1, 2, 3, 4}));
    }
}
```

Why this works: here's a neat little trick, `Set.add()` actually returns a `boolean` telling you whether the item got added or not. If it returns `false`, that means it was already in the set, so we found our duplicate right there. One pass, done.

---

## 7. What is Generics

### The Problem Generics Solve

Picture this: before generics existed (old Java), collections just stored plain `Object`. That meant you could accidentally throw anything into a list, mix numbers and strings together, and the compiler wouldn't even blink. You'd only find out something went wrong at runtime, usually as a `ClassCastException`, remember that one from Chunk 8?

Generics fix this by letting you say upfront: "this list only holds `String`s" or "this list only holds `Integer`s."

```java
List rawList = new ArrayList(); // old style, no type safety
rawList.add("Skyline");
rawList.add(42); // compiler allows this, uh oh

List<String> safeList = new ArrayList<>(); // generics, type locked in
safeList.add("Skyline");
// safeList.add(42); // this line won't even compile
```

The `<String>` part is the generic type. It tells the compiler "only `String` objects allowed here," and the compiler actually enforces it for you, at compile time, before your program even runs.

### Why This Matters

- **Catches mistakes early.** Compile time errors are way better than runtime crashes.
- **No manual casting needed.** With the old raw style, you'd have to cast every single thing you pulled out: `String s = (String) rawList.get(0);`. With generics, `safeList.get(0)` just hands you a `String` directly, no casting required.
- **Cleaner code overall.** Less clutter, more clarity about what's supposed to go where.

### Quick Check

If `List<String> names` already exists, and you try `names.add(100);`, what happens?

It won't even compile. `100` is an `int`, which autoboxes to `Integer`, and `Integer` isn't a `String`, so the compiler stops you right there.

---

## 8. Using Generics with Collections

Every collection type supports generics the same basic way: stick the type in angle brackets right after the collection name.

```java
List<Integer> scores = new ArrayList<>();
Set<String> usernames = new HashSet<>();
Map<String, Integer> ratings = new HashMap<>();
```

Notice `Map` takes TWO type parameters, one for the key, one for the value. Makes sense, since a map stores pairs.

**Only wrapper classes and reference types work here, not primitives.** You can't write `List<int>`, only `List<Integer>`. That's exactly why Chunk 6's wrapper classes matter so much, they're what lets primitive-ish data actually live inside a collection.

```java
List<Integer> episodeCounts = new ArrayList<>();
episodeCounts.add(10); // autoboxed to Integer automatically
int first = episodeCounts.get(0); // auto-unboxed back to int
```

You barely even notice the boxing and unboxing happening, Java handles it quietly in the background, exactly as covered in Chunk 6.

### Quick Check

Why can't you write `List<int>` in Java?

Generics only work with object types, not primitives. You have to use the wrapper class instead, `List<Integer>`, and autoboxing/unboxing takes care of the conversion for you behind the scenes.

---

## 9. Lend a Hand on Generics

### Quiz

1. What's the main benefit of using `List<String>` instead of a raw `List`?
   A. It runs faster at runtime  B. The compiler catches type mistakes before the program even runs  C. It uses less memory  D. There's no real benefit

2. Why does `Map<String, Integer>` need two types inside the brackets?
   A. Typo, should only be one  B. One type is for the key, one is for the value  C. It doubles the storage  D. It's optional

3. Can you write `Set<double>`?
   A. Yes  B. No, use `Set<Double>` instead, since generics need object types  C. Only with a cast  D. Only for `TreeSet`

### Answers

1. **B.** That's the whole point, generics push type errors to compile time, way before they'd otherwise blow up your program at runtime.

2. **B.** A `Map` stores key-value pairs, so it makes sense it needs to know the type of both halves.

3. **B.** Primitives don't work with generics at all. You always need the wrapper class version.

### Practice Problem

You're given a `List<Integer>` representing scores. Write a method that returns a new `List<Integer>` containing only the scores that are 50 or above (passing scores).

**Example 1:**

```
Input: scores = [45, 67, 89, 32, 50, 99]
Output: [67, 89, 50, 99]
```

**Example 2:**

```
Input: scores = [10, 20, 30]
Output: []
Explanation: Nobody passed.
```

**Constraints:**

* `0 <= scores.length <= 1000`
* `0 <= scores[i] <= 100`

### Solution

```java
import java.util.List;
import java.util.ArrayList;

public class PassingScores {
    static List<Integer> getPassingScores(List<Integer> scores) {
        List<Integer> passing = new ArrayList<>();
        for (Integer s : scores) {
            if (s >= 50) {
                passing.add(s);
            }
        }
        return passing;
    }

    public static void main(String[] args) {
        List<Integer> scores = List.of(45, 67, 89, 32, 50, 99);
        System.out.println(getPassingScores(scores));
    }
}
```

Why this works: nothing fancy, just a `List<Integer>` in, a filtered `List<Integer>` out. Generics keep everything type-safe the whole way through, no casting headaches at all.

---

## 10. Lend a Hand on the for Loop

Quick refresher from Chunk 1: the plain old counting `for` loop. It's still very useful with collections, especially when you need the index itself, not just the value.

```java
List<String> movies = new ArrayList<>();
movies.add("Skyline");
movies.add("Ocean Deep");
movies.add("Cosmic Journey");

for (int i = 0; i < movies.size(); i++) {
    System.out.println(i + ": " + movies.get(i));
}
```

Output:
```
0: Skyline
1: Ocean Deep
2: Cosmic Journey
```

Use this style whenever you actually need the position, like printing a numbered list, or looping backward, or skipping every other item.

### Quiz

1. Why use a regular `for` loop instead of a for-each loop when working with a `List`?
   A. For-each is broken  B. Sometimes you need the index itself, which for-each doesn't give you  C. `for` is required by `List`  D. No difference at all

2. What does `movies.size()` return in the loop above?
   A. `2`  B. `3`  C. The last element  D. Compile error

### Answers

1. **B.** For-each just hands you each value, one at a time, no index attached. If you need to know "which position is this," the regular `for` loop is the way to go.

2. **B.** There are 3 movies in the list, so `size()` returns `3`.

### Practice Problem

Given a `List<Integer>`, print every element along with its index, but only for elements at even index positions (0, 2, 4, and so on).

**Example 1:**

```
Input: nums = [10, 20, 30, 40, 50]
Output:
Index 0: 10
Index 2: 30
Index 4: 50
```

### Solution

```java
import java.util.List;

public class EvenIndexPrinter {
    static void printEvenIndices(List<Integer> nums) {
        for (int i = 0; i < nums.size(); i += 2) {
            System.out.println("Index " + i + ": " + nums.get(i));
        }
    }

    public static void main(String[] args) {
        printEvenIndices(List.of(10, 20, 30, 40, 50));
    }
}
```

Why this works: we increment `i` by `2` each time instead of `1`, so we naturally skip odd positions. This is exactly the kind of thing where a for-each loop just wouldn't cut it, since it has no idea what "index" even means.

---

## 11. For-Each Loop with Lend a Hand

The for-each loop, also from Chunk 1, is usually the cleanest choice when you just want to go through every item and you don't care about the index at all.

```java
List<String> genres = new ArrayList<>();
genres.add("Action");
genres.add("Comedy");
genres.add("Drama");

for (String g : genres) {
    System.out.println(g);
}
```

Output:
```
Action
Comedy
Drama
```

Way cleaner than writing `genres.get(i)` everywhere, right? This works for `List`, `Set`, and arrays too. It does NOT directly work on `Map`, though, we'll deal with that in Section 13.

### Lend a Hand: Quiz

1. Can you use a for-each loop on a `Set`?
   A. No  B. Yes, since `Set` is a `Collection` too  C. Only `HashSet`  D. Only if sorted

2. What's the downside of for-each compared to a regular `for` loop?
   A. It's slower  B. You don't get access to the index  C. It can't loop over a `List`  D. There's no real downside ever

### Answers

1. **B.** Both `List` and `Set` are collections, and for-each works on anything that implements `Iterable`, which every collection does.

2. **B.** That's really the one tradeoff, you lose the index. If you need it, go back to the regular `for` loop.

### Practice Problem

Given a `List<String>` of movie titles, count how many titles contain the word `"the"` (case-insensitive, anywhere in the title).

**Example 1:**

```
Input: titles = ["The Great Escape", "Skyline", "Into the Wild", "Ocean Deep"]
Output: 2
Explanation: "The Great Escape" and "Into the Wild" both contain "the" (ignoring case).
```

### Solution

```java
import java.util.List;

public class TitleCounter {
    static int countWithThe(List<String> titles) {
        int count = 0;
        for (String title : titles) {
            if (title.toLowerCase().contains("the")) {
                count++;
            }
        }
        return count;
    }

    public static void main(String[] args) {
        List<String> titles = List.of("The Great Escape", "Skyline", "Into the Wild", "Ocean Deep");
        System.out.println(countWithThe(titles));
    }
}
```

Why this works: we lean on `toLowerCase()` and `contains()` from Chunk 5's `String` methods, combined with a plain for-each loop since we just need every value, no index needed here at all.

---

## 12. Iterator with Lend a Hand

### The Problem

Here's a classic trap: what if you want to remove items from a `List` WHILE looping through it with a for-each loop?

```java
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4, 5, 6));
for (int n : nums) {
    if (n % 2 == 0) {
        nums.remove(Integer.valueOf(n)); // DON'T do this
    }
}
```

This actually throws a `ConcurrentModificationException` at runtime. Java gets upset when you modify a collection's structure while a for-each loop is in the middle of going through it.

### Enter the Iterator

An `Iterator` is a special object that knows how to safely walk through a collection, and, importantly, it has its own `remove()` method that's actually safe to use mid-loop.

```java
import java.util.Iterator;

List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4, 5, 6));
Iterator<Integer> it = nums.iterator();

while (it.hasNext()) {
    int n = it.next();
    if (n % 2 == 0) {
        it.remove(); // this is totally safe
    }
}

System.out.println(nums);
```

Output: `[1, 3, 5]`

### The Three Main Methods

| Method | What it does |
|---|---|
| `hasNext()` | is there another item left to look at |
| `next()` | gives you the next item, and moves forward |
| `remove()` | safely removes the item you just got from `next()` |

Honestly, under the hood, a for-each loop is basically just an `Iterator` in disguise, Java writes all that `hasNext()`/`next()` code for you automatically. You only need to grab the `Iterator` yourself when you need that safe `remove()`.

### Lend a Hand: Quiz

1. What happens if you try to `remove()` from a `List` directly inside a for-each loop?
   A. Works fine, no issue  B. Throws `ConcurrentModificationException` at runtime  C. Compile error  D. Silently does nothing

2. What does `it.next()` do?
   A. Removes the current item  B. Returns the next item and moves the iterator forward  C. Checks if more items exist  D. Restarts the loop

3. Why is `Iterator.remove()` safe when a plain `List.remove()` inside a for-each loop is not?
   A. They're actually the same thing  B. The `Iterator` keeps track of exactly where it is in the collection, so it can adjust safely; a for-each loop has no such awareness  C. `Iterator.remove()` doesn't actually remove anything  D. It's not actually safe either

### Answers

1. **B.** Modifying a collection's structure mid for-each loop confuses Java's internal bookkeeping, and it throws that exception to protect you from weird, inconsistent results.

2. **B.** `next()` does two things at once: hands you the current item, and advances the iterator to the following one.

3. **B.** The `Iterator` object itself is the thing walking through the list, so when you call its own `remove()`, it can update its internal position correctly. A for-each loop doesn't give you that same direct control.

### Practice Problem

Given a `List<Integer>`, remove every number that is divisible by 3, using an `Iterator`, and return the modified list.

**Example 1:**

```
Input: nums = [3, 4, 6, 7, 9, 10, 12]
Output: [4, 7, 10]
Explanation: 3, 6, 9, and 12 are all divisible by 3, so they get removed.
```

### Solution

```java
import java.util.Iterator;
import java.util.List;
import java.util.ArrayList;

public class RemoveMultiplesOfThree {
    static List<Integer> removeMultiples(List<Integer> nums) {
        List<Integer> result = new ArrayList<>(nums);
        Iterator<Integer> it = result.iterator();
        while (it.hasNext()) {
            int n = it.next();
            if (n % 3 == 0) {
                it.remove();
            }
        }
        return result;
    }

    public static void main(String[] args) {
        List<Integer> nums = new ArrayList<>(List.of(3, 4, 6, 7, 9, 10, 12));
        System.out.println(removeMultiples(nums));
    }
}
```

Why this works: exactly the pattern from this section, we walk through with `hasNext()`/`next()`, and any time we spot a multiple of 3, we call `it.remove()` right then and there. No exceptions, no drama.

---

## 13. Map Interface with Lend a Hand on HashMap and TreeMap

### The Basics

A `Map` stores **key-value pairs**. Think of it like a real dictionary: you look up a word (the key) and get its meaning (the value). No duplicate keys allowed, but values can repeat just fine.

```java
Map<String, Integer> ratings = new HashMap<>();
ratings.put("Skyline", 8);
ratings.put("Ocean Deep", 9);
ratings.put("Skyline", 7); // overwrites the old value for "Skyline"

System.out.println(ratings.get("Skyline"));       // 7
System.out.println(ratings.containsKey("Ocean Deep")); // true
System.out.println(ratings.size());               // 2
```

Notice putting `"Skyline"` a second time didn't create a duplicate entry, it just updated the value. That's the deal with `Map` keys, always unique.

### Handy Map Methods

| Method | What it does |
|---|---|
| `put(key, value)` | adds or updates a pair |
| `get(key)` | gets the value for that key (or `null` if missing) |
| `containsKey(key)` | true/false |
| `remove(key)` | removes that pair |
| `keySet()` | gives you all the keys, as a `Set` |
| `values()` | gives you all the values, as a `Collection` |
| `entrySet()` | gives you all the key-value pairs together |

### Looping Over a Map

Since `Map` isn't directly a `Collection`, you can't for-each straight over it. You loop over `entrySet()` instead.

```java
for (Map.Entry<String, Integer> entry : ratings.entrySet()) {
    System.out.println(entry.getKey() + " -> " + entry.getValue());
}
```

### HashMap vs TreeMap

- **`HashMap`**: fast, but no guaranteed order for the keys.
- **`TreeMap`**: automatically keeps keys sorted (ascending, by default).

```java
Map<String, Integer> sorted = new TreeMap<>(ratings);
System.out.println(sorted);
```

If `ratings` had `"Skyline"` and `"Ocean Deep"`, a `TreeMap` version would print them in alphabetical order: `{Ocean Deep=9, Skyline=7}`, since `O` comes before `S`.

### Lend a Hand: Quiz

1. What happens if you `put()` a key that already exists in a `Map`?
   A. It's ignored  B. The old value gets overwritten with the new one  C. Compile error  D. Both values are kept

2. What does `map.get("missing")` return if `"missing"` isn't actually a key in the map?
   A. `0`  B. `null`  C. Throws an exception  D. Empty `String`

3. What's the main difference between `HashMap` and `TreeMap`?
   A. `TreeMap` is always faster  B. `TreeMap` keeps keys sorted automatically, `HashMap` doesn't guarantee any order  C. `HashMap` can't hold `String` keys  D. No real difference

4. How do you loop over a `Map` and get both keys and values together?
   A. Regular for-each directly on the map  B. Loop over `entrySet()`, using `Map.Entry`  C. `Map` cannot be looped over  D. Use `keySet()` only

### Answers

1. **B.** `Map` keys are always unique, so a repeat `put()` just replaces whatever value was there before.

2. **B.** Missing key means `null` comes back. Worth checking for this before you use the result, or you might run into a surprise `NullPointerException` from Chunk 8.

3. **B.** That's the whole reason to reach for `TreeMap`, when you specifically want the keys in sorted order without doing the sorting yourself.

4. **B.** `entrySet()` gives you `Map.Entry` objects, each one bundling a key and its value together, perfect for a for-each loop.

### Practice Problem

Given a string `s`, return the first character that does not repeat anywhere else in the string. If every character repeats, return an underscore `_` instead.

**Example 1:**

```
Input: s = "swiss"
Output: 'w'
Explanation: 's' repeats, 'w' does not repeat and comes first among non-repeating characters.
```

**Example 2:**

```
Input: s = "aabbcc"
Output: '_'
Explanation: every character repeats at least once.
```

**Constraints:**

* `1 <= s.length <= 10^5`
* `s` consists of only lowercase English letters.

### Solution

```java
import java.util.Map;
import java.util.LinkedHashMap;

public class FirstUniqueChar {
    static char firstUnique(String s) {
        Map<Character, Integer> counts = new LinkedHashMap<>();

        for (char c : s.toCharArray()) {
            counts.put(c, counts.getOrDefault(c, 0) + 1);
        }

        for (Map.Entry<Character, Integer> entry : counts.entrySet()) {
            if (entry.getValue() == 1) {
                return entry.getKey();
            }
        }

        return '_';
    }

    public static void main(String[] args) {
        System.out.println(firstUnique("swiss"));
        System.out.println(firstUnique("aabbcc"));
    }
}
```

Why this works: first pass builds a count of every character using the map (`getOrDefault` is a nice shortcut, if the key isn't there yet, it just uses `0` instead of crashing). Second pass walks through in the same order we inserted things (that's why we picked `LinkedHashMap`, it remembers insertion order, unlike plain `HashMap`), and returns the first character whose count is exactly `1`.

---

## 14. Lend a Hand on Adding User Defined Objects to a Collection

### Here's Where It Gets Interesting

So far we've been putting `String`s and `Integer`s into collections. What about your own custom classes, like a `Movie` object?

```java
public class Movie {
    private String title;
    private int year;

    public Movie(String title, int year) {
        this.title = title;
        this.year = year;
    }

    public String getTitle() { return title; }
    public int getYear() { return year; }
}
```

You can absolutely put `Movie` objects into a `List`, no problem:

```java
List<Movie> movies = new ArrayList<>();
movies.add(new Movie("Skyline", 2020));
movies.add(new Movie("Skyline", 2020)); // "duplicate" content, but...

System.out.println(movies.size()); // 2, both got added!
```

Wait, both got added, even though they look identical? Yep. Here's why: `List` doesn't care about duplicates at all, it'll happily hold two objects with the exact same data.

### The Real Trap: Set and Map Need equals() and hashCode()

Now here's where it actually matters. If you try to put `Movie` objects into a `HashSet`, or use them as `Map` keys, Java needs to know: "are these two `Movie` objects actually the same thing, or different?" And by default, remember Chunk 5, `equals()` just checks if it's the literal same object in memory. Two separately created `Movie` objects, even with identical data, are NOT equal by default.

```java
Set<Movie> movieSet = new HashSet<>();
movieSet.add(new Movie("Skyline", 2020));
movieSet.add(new Movie("Skyline", 2020));

System.out.println(movieSet.size()); // 2, not 1!
```

That's probably NOT what you wanted. To fix this, you override `equals()` and `hashCode()` together, exactly the pairing Chunk 5 covered.

```java
public class Movie {
    private String title;
    private int year;

    public Movie(String title, int year) {
        this.title = title;
        this.year = year;
    }

    public String getTitle() { return title; }
    public int getYear() { return year; }

    @Override
    public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof Movie)) return false;
        Movie that = (Movie) other;
        return this.year == that.year && this.title.equals(that.title);
    }

    @Override
    public int hashCode() {
        return title.hashCode() * 31 + year;
    }
}
```

Now try it again:

```java
Set<Movie> movieSet = new HashSet<>();
movieSet.add(new Movie("Skyline", 2020));
movieSet.add(new Movie("Skyline", 2020));

System.out.println(movieSet.size()); // 1, correctly treated as duplicates now
```

Once `equals()` and `hashCode()` are properly overridden, `HashSet` (and `HashMap` keys) will correctly treat two objects with the same data as "the same thing," even though they're technically two separate objects sitting in two separate spots in memory.

### The Golden Rule

If you're putting custom objects into a `HashSet` or using them as `HashMap` keys, and you care about "same data = same object," you MUST override both `equals()` and `hashCode()` together. Skip one, and things break in weird, hard-to-spot ways, exactly the contract violation warning from Chunk 5.

### Lend a Hand: Quiz

1. If `Movie` doesn't override `equals()`, what does `movie1.equals(movie2)` check, even if both have identical `title` and `year`?
   A. Compares all fields automatically  B. Falls back to `Object`'s default, which checks if it's literally the same object in memory  C. Always returns `true`  D. Compile error

2. Why does adding two "identical looking" `Movie` objects to a `List` result in a size of `2`?
   A. Bug in `List`  B. `List` doesn't check for duplicates at all, it just adds whatever you give it  C. `Movie` is broken  D. `add()` failed silently

3. What's required to make a `HashSet<Movie>` correctly treat two same-data `Movie` objects as duplicates?
   A. Nothing, it works automatically  B. Override both `equals()` and `hashCode()` consistently  C. Override only `equals()`  D. Override only `hashCode()`

### Answers

1. **B.** Without an override, `equals()` is just `Object`'s original version, reference comparison, same as `==`. Two separately built objects will never be "equal" this way, no matter how similar their data looks.

2. **B.** `List` simply doesn't do any duplicate checking. It's an ordered bag that holds whatever you put in, as many times as you put it in.

3. **B.** Both together, always. `HashSet` uses `hashCode()` first to find the right bucket, then `equals()` to confirm an actual match. Skipping either one breaks the whole mechanism.

### Practice Problem

You're given a `List<Movie>`, where each `Movie` has a `title` and a `year`. Two movies count as duplicates if they have the exact same title AND year. Return how many unique movies are in the list.

Assume `Movie` already has `equals()` and `hashCode()` properly overridden, based on `title` and `year`.

**Example 1:**

```
Input: movies = [("Skyline", 2020), ("Ocean Deep", 2019), ("Skyline", 2020)]
Output: 2
Explanation: "Skyline" (2020) shows up twice, so only 2 unique movies remain.
```

### Solution

```java
import java.util.List;
import java.util.Set;
import java.util.HashSet;

public class UniqueMovieCounter {
    static int countUnique(List<Movie> movies) {
        Set<Movie> uniqueMovies = new HashSet<>(movies);
        return uniqueMovies.size();
    }

    public static void main(String[] args) {
        List<Movie> movies = List.of(
            new Movie("Skyline", 2020),
            new Movie("Ocean Deep", 2019),
            new Movie("Skyline", 2020)
        );
        System.out.println(countUnique(movies));
    }
}
```

Why this works: dumping the whole `List` straight into a `new HashSet<>(movies)` automatically kicks out duplicates, as long as `equals()` and `hashCode()` are set up right. Then `size()` just tells us how many unique ones survived. Nice and short.

---

## 15. Consolidated Quiz

Let's mix in stuff from earlier chunks too, since that's kind of the whole point of this course.

1. `Movie` from this chunk overrides `equals()` and `hashCode()`. Which earlier chunk first taught you how to do that properly?
   A. Chunk 3  B. Chunk 4  C. Chunk 5  D. Chunk 7

2. `List<Integer> episodeCounts` stores `int` values through autoboxing. Which chunk explained autoboxing in detail?
   A. Chunk 4  B. Chunk 5  C. Chunk 6  D. Chunk 8

3. Trying to modify a `List` mid for-each loop throws `ConcurrentModificationException`. Which chunk's rules about the enhanced for-each loop scope explain why direct removal during that loop is risky?
   A. Chunk 1  B. Chunk 3  C. Chunk 5  D. Chunk 6

4. `Movie` is a plain class with private fields and public getters. What's that pattern called, first taught back in Chunk 2?
   A. Polymorphism  B. Encapsulation  C. Inheritance  D. Overloading

5. If you tried `list.get(100)` on a `List` with only 5 elements, what would happen?
   A. Returns `null`  B. Throws an exception, similar in spirit to `ArrayIndexOutOfBoundsException` from Chunk 1  C. Returns `0`  D. Compile error

6. Why can't you write `List<int>` directly?
   A. Syntax error, no real reason  B. Generics only work with reference types, so you need `Integer`, the wrapper class from Chunk 6, instead  C. `int` doesn't exist in Java  D. You actually can

7. `TreeMap` keeps its keys sorted. Which sorting-adjacent idea from Chunk 6 does `Double.compare`/`Integer.compare` connect to here?
   A. Nothing related  B. Both are about defining a reliable way to order values, which is exactly what a `TreeMap` needs internally to keep keys sorted  C. `TreeMap` never sorts numbers  D. Compile error

8. A `StreamingService` method from Chunk 8 threw `SubscriptionExpiredException`. If we stored a bunch of `User` objects who hit that exception into a `List<User>` for a report, would `List` care whether `User` overrides `equals()`?
   A. Yes, `List` requires it  B. No, `List` allows duplicates regardless, `equals()`/`hashCode()` overrides only really matter for `Set` and `Map` key behavior  C. Compile error without it  D. `User` cannot be stored in a `List`

### Answers

1. **C.** Chunk 5 walked through the whole `equals()`/`hashCode()` contract, and this chunk just reuses that exact idea for a new class, `Movie`.

2. **C.** Chunk 6 covered autoboxing and unboxing in depth, which is exactly what quietly happens every time you add an `int` to a `List<Integer>`.

3. **A.** Chunk 1 covered loop and scope rules, and the general idea that a for-each loop expects the collection to stay stable while it's mid-iteration connects right back to that same "don't mess with what you're iterating over" caution.

4. **B.** Private fields plus public getter methods is the textbook encapsulation pattern, first properly introduced back in Chunk 2.

5. **B.** `List.get()` with an out of range index throws `IndexOutOfBoundsException`, which is basically the `List` world's version of `ArrayIndexOutOfBoundsException` from Chunk 1's array coverage.

6. **B.** Generics need object types, not primitives, exactly why the wrapper classes from Chunk 6 exist and get used everywhere collections are involved.

7. **B.** Both are fundamentally about defining a consistent way to compare and order values, `TreeMap` needs exactly that kind of ordering logic internally to keep its keys sorted.

8. **B.** `List` never cares about `equals()`/`hashCode()` for its own basic behavior, since it allows duplicates freely. Those overrides only start to matter once you move to `Set` or use objects as `Map` keys.

---

## 16. Revision Summary

| Concept | Key Point | Where It Showed Up |
|---|---|---|
| Collections | A flexible, resizable group of objects, unlike fixed-size arrays | `watchlist` example |
| List | Ordered, duplicates allowed, access by index | `ArrayList<String>` |
| Set | No duplicates, `HashSet` doesn't guarantee order | `Set<String> genres` |
| Generics | `<Type>` locks a collection to one type, catches mistakes at compile time | `List<String>` vs raw `List` |
| Using generics with collections | Only object types work, not primitives, that's why wrappers matter | `List<Integer>` |
| for loop | Best when you need the index | Printing numbered movies |
| for-each loop | Cleanest for just going through every value | Looping over `genres` |
| Iterator | Safe way to remove items while looping | `it.remove()` |
| Map | Key-value pairs, unique keys, `HashMap` unordered, `TreeMap` sorted | `ratings.put(...)` |
| User defined objects in collections | `List` doesn't care about duplicates; `Set`/`Map` keys need proper `equals()`/`hashCode()` | `Movie` in a `HashSet` |

**Quick tip for the exam:** whenever a question involves a custom object going into a `HashSet` or being used as a `HashMap` key, your first thought should be "did they override `equals()` and `hashCode()`?" That one question answers almost every tricky "why did this count come out wrong" style question in this whole topic.