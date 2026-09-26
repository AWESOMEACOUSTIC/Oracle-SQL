# Java Competitive Exam Preparation
## Chunk 12: I/O Streams

Time to talk about how Java actually reads and writes files. This is one of those topics that sounds scary ("streams," "buffers," all these classes wrapping other classes) but once you get the core picture, it's actually pretty simple. We'll go slow, use a lot of examples, and end with one proper big scenario problem that ties everything together.

Same casual style as the last few chunks.

---

## Table of Contents

1. Introduction to I/O Streams and Its Types
2. InputStream Class and Its Hierarchy
3. FileInputStream with Example
4. BufferedInputStream with Lend a Hand
5. FileOutputStream
6. BufferedOutputStream with Lend a Hand
7. The Big Scenario Question
8. Consolidated Quiz
9. Revision Summary

---

## 1. Introduction to I/O Streams and Its Types

### The Core Idea

Picture a garden hose. Water doesn't teleport from the tap to the bucket, it flows through the hose, bit by bit, from one end to the other. A **stream**, in Java, is exactly that idea, but for data. Data flows from a SOURCE (like a file on your disk) to a DESTINATION (like your program), or the other way around, bit by bit, instead of all at once.

That's genuinely the whole concept. Everything else in this chunk is just "okay, but what kind of hose, and what's flowing through it."

### Two Big Categories

- **Byte streams**: move raw bytes, one at a time (or in chunks). Good for ANY kind of file, text, images, videos, doesn't matter. The classes here all end in `InputStream` or `OutputStream`. This is what this whole chunk focuses on.
- **Character streams**: move actual characters (text), and handle text encoding properly for you. Classes here end in `Reader` or `Writer`. Not our focus this chunk, but worth knowing they exist for when you're specifically working with text and want cleaner, character-aware handling.

### Input vs Output

Simple as it sounds:
- **Input stream**: data flowing INTO your program (you're reading something).
- **Output stream**: data flowing OUT of your program (you're writing something).

All of this lives in the `java.io` package.

### Quick Check

If you're reading an image file, would you use a byte stream or a character stream?

Byte stream. Images aren't text, there's no meaningful "character" to read, it's just raw binary data, exactly what byte streams are built for.

---

## 2. InputStream Class and Its Hierarchy

### The Family Tree

`InputStream` sits at the top of every byte-based input class, and, exactly the abstract class idea from Chunk 4, it IS abstract. You never write `new InputStream()` directly.

```
InputStream (abstract)
 ├── FileInputStream        (reads from a file)
 ├── ByteArrayInputStream   (reads from a byte array already in memory)
 └── FilterInputStream (abstract, wraps another InputStream)
      └── BufferedInputStream (adds an internal buffer for speed)
```

### The Core Method: read()

Every `InputStream` gives you a `read()` method that reads a single byte and returns it, as an `int`.

Wait, a single BYTE, but it returns an `int`? Here's the actual reason, and it's a genuinely good "why" to know: a `byte` can represent values `0` to `255` (256 possible values). But `read()` also needs a special signal for "there's nothing left to read, end of stream," and it uses `-1` for that. If `read()` returned a `byte`, there'd be no room left for a distinct `-1` value alongside all 256 real byte values. Returning an `int` gives plenty of extra room, so `-1` can mean "end of stream" without colliding with any actual byte value.

```java
int result = someInputStream.read();
if (result == -1) {
    System.out.println("Nothing left to read");
} else {
    System.out.println("Got a byte: " + result);
}
```

### Other Handy Methods

| Method | What it does |
|---|---|
| `read()` | reads one byte, returns `int` (`0`-`255`, or `-1` for end of stream) |
| `read(byte[] buffer)` | fills the array with as many bytes as it can, returns how many it actually read |
| `available()` | roughly how many bytes are ready to be read right now |
| `close()` | releases the file (or whatever resource) this stream is holding onto |

### Why close() Matters

Every open stream is holding onto a real operating system resource, like a file handle. Forgetting to `close()` it is genuinely similar in spirit to the memory leak discussion from Chunk 7, an unreleased resource just sits there taking up space it doesn't need to anymore. We'll cover the cleanest way to guarantee closing happens, `try`-with-resources, in Section 3.

### Quick Check

Why does `read()` return `-1` specifically, instead of, say, `0`, to mean "end of stream"?

Because `0` is a perfectly valid actual byte value. `-1` is impossible for a real byte (which only ranges `0` to `255`), so it's a safe, unambiguous signal that can never be confused with genuine data.

---

## 3. FileInputStream with Example

### The Basics

`FileInputStream` is the concrete class (remember, `InputStream` itself is abstract) you actually use to read bytes straight from a file.

```java
import java.io.FileInputStream;
import java.io.IOException;

public class ReadFileDemo {
    public static void main(String[] args) {
        try (FileInputStream fis = new FileInputStream("data.txt")) {
            int byteRead;
            while ((byteRead = fis.read()) != -1) {
                System.out.print((char) byteRead);
            }
        } catch (IOException e) {
            System.out.println("Something went wrong reading the file: " + e.getMessage());
        }
    }
}
```

### That try (...) Bit: try-with-resources

Notice the `try (FileInputStream fis = ...)` part, this is called **try-with-resources**, and it's the actual, real answer to something Chunk 7 and Chunk 8 both mentioned but didn't fully show you yet: a reliable way to guarantee cleanup, way better than depending on garbage collection or `finalize()`.

Here's the deal: any class that implements the `Closeable` interface (which `InputStream` and `OutputStream` both do, another genuine interface example, straight from Chunk 4) can be declared inside those parentheses. Java then GUARANTEES `close()` gets called automatically once the `try` block finishes, whether it finished normally OR because an exception was thrown. You don't need a separate `finally` block just to call `close()` yourself, Java does it for you.

```java
// the old, more error-prone way
FileInputStream fis = new FileInputStream("data.txt");
try {
    // use fis
} finally {
    fis.close(); // easy to forget, or to get wrong if multiple resources are involved
}

// the modern way, much safer
try (FileInputStream fis = new FileInputStream("data.txt")) {
    // use fis
} // close() is called automatically here, guaranteed
```

### Why This Throws a Checked Exception

`new FileInputStream("data.txt")` throws `FileNotFoundException` if the file doesn't exist, and `read()` throws `IOException` if something goes wrong mid-read. Both are checked exceptions (`FileNotFoundException` is actually a subclass of `IOException`), exactly Chunk 8's classification, since file operations can fail for all sorts of reasons entirely outside your program's control, missing files, permission issues, a full disk, and the compiler wants you to genuinely plan for that.

### Reading a Whole Small File at Once

For a small file, it's often easier to just grab everything in one shot using a byte array, rather than looping one byte at a time.

```java
import java.io.FileInputStream;
import java.io.IOException;

public class ReadWholeFile {
    public static void main(String[] args) {
        try (FileInputStream fis = new FileInputStream("data.txt")) {
            byte[] buffer = new byte[1024]; // assumes the file is smaller than 1024 bytes
            int bytesRead = fis.read(buffer);
            String content = new String(buffer, 0, bytesRead);
            System.out.println(content);
        } catch (IOException e) {
            System.out.println("Could not read file: " + e.getMessage());
        }
    }
}
```

That `new String(buffer, 0, bytesRead)` is exactly the offset-and-count `String` constructor from Chunk 5, building a `String` from just the portion of the array that actually holds real data.

### Quiz

1. What does `FileInputStream`'s constructor throw if the file doesn't exist?
   A. `NullPointerException`  B. `FileNotFoundException`, a checked exception  C. Nothing, returns `null`  D. `ArrayIndexOutOfBoundsException`

2. What's the benefit of try-with-resources over manually calling `close()` in a `finally` block?
   A. No real benefit  B. Java guarantees `close()` runs automatically, even if an exception occurs, without you writing it yourself  C. It's just shorter to type  D. It skips exception handling entirely

3. If `read()` returns `-1`, what does that mean?
   A. An error occurred  B. End of the stream, nothing left to read  C. The byte value 255  D. The file is corrupted

### Answers

1. **B.** `FileNotFoundException` extends `IOException`, so it's checked, and must be caught or declared, exactly Chunk 8's rules.

2. **B.** That automatic guaranteed cleanup, even on an exception, is the whole point, no more forgetting a `finally` block or writing it incorrectly with multiple resources.

3. **B.** `-1` is reserved specifically to mean "nothing left," as covered in Section 2, since it can never be confused with an actual byte value.

### Practice Problem

Suppose a file named `notes.txt` contains exactly this text: `Hi!`. Write a program using `FileInputStream` that counts how many bytes are in the file.

**Example:**

```
File contents: "Hi!"
Output: 3
```

### Solution

```java
import java.io.FileInputStream;
import java.io.IOException;

public class CountBytes {
    static int countBytes(String filename) throws IOException {
        int count = 0;
        try (FileInputStream fis = new FileInputStream(filename)) {
            while (fis.read() != -1) {
                count++;
            }
        }
        return count;
    }

    public static void main(String[] args) throws IOException {
        System.out.println(countBytes("notes.txt"));
    }
}
```

Why this works: each call to `read()` grabs exactly one byte and advances the stream forward. We just keep looping and counting until we hit that `-1` end-of-stream marker.

---

## 4. BufferedInputStream with Lend a Hand

### The Problem It Solves

Calling `read()` one byte at a time directly on a `FileInputStream` genuinely means the underlying operating system might be asked to fetch data from the disk on every single call, which is slow. Real disk access is expensive compared to just grabbing something already sitting in memory.

`BufferedInputStream` fixes this by reading a big CHUNK of the file into an internal memory buffer all at once, and then serving your individual `read()` calls from that fast in-memory buffer instead. Only once the buffer runs dry does it go back to the disk for another chunk.

### How You Use It

```java
import java.io.FileInputStream;
import java.io.BufferedInputStream;
import java.io.IOException;

public class BufferedReadDemo {
    public static void main(String[] args) {
        try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("data.txt"))) {
            int byteRead;
            while ((byteRead = bis.read()) != -1) {
                System.out.print((char) byteRead);
            }
        } catch (IOException e) {
            System.out.println("Read failed: " + e.getMessage());
        }
    }
}
```

Notice `BufferedInputStream` wraps ANOTHER stream, here a `FileInputStream`, as its constructor argument. This "wrap one stream inside another" style is common throughout Java's I/O classes, `BufferedInputStream` adds buffering ON TOP of whatever stream you hand it.

### It's Still an InputStream

Since `BufferedInputStream` extends `InputStream` (through `FilterInputStream`), it has the exact same `read()` method signature, same `close()`, everything. Any code expecting a plain `InputStream` reference happily accepts a `BufferedInputStream` too, exactly the polymorphism idea from Chunk 3 and Chunk 4. The buffering happens invisibly underneath; your calling code doesn't change at all.

```java
InputStream in = new BufferedInputStream(new FileInputStream("data.txt"));
// 'in' can be used exactly like any other InputStream, buffering is just an invisible bonus
```

### Lend a Hand: Quiz

1. Why is `BufferedInputStream` generally faster than reading directly from a plain `FileInputStream`?
   A. It's not actually faster  B. It reads a large chunk into memory at once, reducing how often the disk is actually touched  C. It skips bytes to save time  D. It uses less memory

2. What does `BufferedInputStream`'s constructor take as an argument?
   A. A filename `String`  B. Another `InputStream` to wrap  C. A byte array  D. Nothing

3. Can a method expecting a plain `InputStream` parameter accept a `BufferedInputStream` argument?
   A. No, types must match exactly  B. Yes, since `BufferedInputStream` IS an `InputStream`, through inheritance  C. Only with a cast  D. Compile error

### Answers

1. **B.** Fewer actual trips to the disk, since most `read()` calls get served straight from the fast in-memory buffer instead.

2. **B.** You wrap an existing stream, like `new BufferedInputStream(new FileInputStream("data.txt"))`, adding buffering on top of it.

3. **B.** Exactly the polymorphism principle from Chunk 3, any subclass reference can be used wherever its superclass type is expected.

### Practice Problem

Using `BufferedInputStream`, write a program that reads a file named `greeting.txt` and prints its entire contents as a `String`.

**Example:**

```
File contents: "Hello there!"
Output: Hello there!
```

### Solution

```java
import java.io.FileInputStream;
import java.io.BufferedInputStream;
import java.io.IOException;

public class ReadWithBuffer {
    static String readFile(String filename) throws IOException {
        try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream(filename))) {
            byte[] buffer = new byte[1024];
            int bytesRead = bis.read(buffer);
            return new String(buffer, 0, bytesRead);
        }
    }

    public static void main(String[] args) throws IOException {
        System.out.println(readFile("greeting.txt"));
    }
}
```

Why this works: same overall shape as the plain `FileInputStream` version from Section 3, just wrapped in a `BufferedInputStream` for the speed benefit. The calling code barely changes at all, which is exactly the point.

---

## 5. FileOutputStream

### The Basics

`FileOutputStream` is `FileInputStream`'s mirror image, it writes bytes TO a file instead of reading them.

```java
import java.io.FileOutputStream;
import java.io.IOException;

public class WriteFileDemo {
    public static void main(String[] args) {
        String message = "Hello, file!";
        try (FileOutputStream fos = new FileOutputStream("output.txt")) {
            fos.write(message.getBytes());
        } catch (IOException e) {
            System.out.println("Write failed: " + e.getMessage());
        }
    }
}
```

`message.getBytes()` converts a `String` into a raw `byte[]`, since `write()` deals in bytes, not `String`s directly.

### Overwrite vs Append

By default, `new FileOutputStream("output.txt")` WIPES OUT any existing content in that file and starts fresh. If you want to add onto the end instead, pass `true` as a second constructor argument.

```java
FileOutputStream overwrite = new FileOutputStream("output.txt");       // erases existing content first
FileOutputStream append = new FileOutputStream("output.txt", true);    // keeps existing content, adds after it
```

This is a genuinely important gotcha, forgetting that second `true` when you meant to append is a classic way to accidentally wipe out data you actually wanted to keep.

### Quiz

1. What happens to `output.txt`'s existing content with `new FileOutputStream("output.txt")` (no second argument)?
   A. It's preserved, new content is added at the end  B. It's erased, the file starts empty  C. Compile error  D. Nothing happens until `close()`

2. How do you open a file in APPEND mode?
   A. It's the default  B. Pass `true` as a second constructor argument  C. Call `append()` after opening  D. Not possible in Java

3. What does `"Hello".getBytes()` return?
   A. A `String`  B. A `byte[]`  C. An `int`  D. A `char[]`

### Answers

1. **B.** Default behavior wipes the file clean first. If you wanted to keep what was already there, you needed the append flag.

2. **B.** `new FileOutputStream(filename, true)`, that boolean is specifically the append toggle.

3. **B.** `getBytes()` converts the `String`'s characters into their raw byte representation, exactly what `write()` needs.

### Practice Problem

Write a program that writes the text `"Order received"` to a new file called `log.txt`, and then, in a second separate step, appends `"Order shipped"` on a new line without erasing the first message.

**Expected final contents of log.txt:**

```
Order received
Order shipped
```

### Solution

```java
import java.io.FileOutputStream;
import java.io.IOException;

public class AppendLogDemo {
    public static void main(String[] args) throws IOException {
        try (FileOutputStream fos = new FileOutputStream("log.txt")) {
            fos.write("Order received\n".getBytes());
        }

        try (FileOutputStream fos = new FileOutputStream("log.txt", true)) {
            fos.write("Order shipped".getBytes());
        }
    }
}
```

Why this works: the first block creates (or wipes) `log.txt` and writes the first line. The second block reopens the SAME file with `true` for append mode, so it adds the second line right after the first, instead of erasing it.

---

## 6. BufferedOutputStream with Lend a Hand

### The Idea

Exactly the same relationship as `BufferedInputStream` had with `FileInputStream`: `BufferedOutputStream` wraps a `FileOutputStream` (or any `OutputStream`) and collects your writes in an internal memory buffer first, only actually pushing data out to the disk once that buffer fills up, or when you explicitly tell it to.

```java
import java.io.FileOutputStream;
import java.io.BufferedOutputStream;
import java.io.IOException;

public class BufferedWriteDemo {
    public static void main(String[] args) {
        try (BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("output.txt"))) {
            bos.write("Buffered writing is faster!".getBytes());
        } catch (IOException e) {
            System.out.println("Write failed: " + e.getMessage());
        }
    }
}
```

### The Big Gotcha: flush() and close()

Here's the classic trap: since `BufferedOutputStream` holds your data in memory before actually writing it to disk, if your program somehow ends WITHOUT properly closing (or flushing) the stream, that buffered data might never actually make it to the file. You'd swear you wrote something, and the file would just be empty, or missing the last bit you wrote.

`close()` (which try-with-resources calls automatically) does flush any remaining buffered data before shutting down, so as long as you're using try-with-resources properly, this is already handled for you. But it's worth knowing `flush()` exists as a manual escape hatch, forcing everything currently buffered out to disk right now, without closing the stream entirely.

```java
bos.write(someBytes);
bos.flush(); // force it out to disk right now, without closing the stream
```

### Lend a Hand: Quiz

1. Why might data written through a `BufferedOutputStream` NOT actually appear in the file right away?
   A. It's a bug in Java  B. It's sitting in an internal memory buffer, not yet pushed out to disk  C. `BufferedOutputStream` doesn't support writing  D. The file is read-only

2. What does `flush()` do?
   A. Deletes the buffer's content  B. Forces any currently buffered data out to disk immediately, without closing the stream  C. Closes the stream  D. Nothing, it's decorative

3. Does try-with-resources handle flushing buffered data for you?
   A. No, you must always call `flush()` yourself  B. Yes, `close()` (called automatically by try-with-resources) flushes remaining data first  C. Only for `BufferedInputStream`  D. Only if you call `flush()` manually first

### Answers

1. **B.** That's the entire point of buffering, data sits in memory temporarily for speed, and only gets pushed to the actual file in batches.

2. **B.** `flush()` is your manual "push everything out right now" button, useful when you want data on disk immediately without ending the stream.

3. **B.** `close()` is documented to flush remaining buffered data before actually shutting the stream down, so try-with-resources genuinely does cover this for you automatically.

### Practice Problem

Explain, in your own words (write it as a code comment), why the following code might result in an empty `output.txt` file, and then fix it.

```java
BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("output.txt"));
bos.write("Important data".getBytes());
// program ends here, no close() or flush() called
```

### Solution

```java
import java.io.FileOutputStream;
import java.io.BufferedOutputStream;
import java.io.IOException;

public class FixedBufferedWrite {
    public static void main(String[] args) throws IOException {
        // The original code never called close() or flush(), so "Important data"
        // may still be sitting in BufferedOutputStream's internal memory buffer
        // when the program ends, never actually reaching the disk. Using
        // try-with-resources guarantees close() runs, which flushes the buffer
        // first, fixing the problem.
        try (BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("output.txt"))) {
            bos.write("Important data".getBytes());
        }
    }
}
```

Why this works: wrapping the stream in try-with-resources guarantees `close()` runs no matter what, and `close()` always flushes any remaining buffered bytes out to disk first, so the data is safely written every time.

---

## 7. The Big Scenario Question

Here's a proper, lengthy scenario problem, the kind you'd see in an actual assessment, pulling together everything from this chunk plus a good chunk of earlier ones.

### Scenario: ShopEase Order Log Processor

You're working on the backend for **ShopEase**, an online store. Every order placed on the site gets appended, as a single line of raw text, to a file called `orders.txt`. Each line follows this format:

```
orderId,customerName,amount
```

For example:

```
101,Priya,499.50
102,Arjun,0
103,Meera,899.00
104,BadLine
105,Karan,-50.00
106,Divya,1200.75
```

Some lines in this file are malformed, either because they don't have exactly three comma-separated fields (like line 104), or because the amount isn't a valid positive number (lines 102 and 105 both have non-positive amounts, which should be treated as invalid, not just unparseable text).

Your job is to write a program that:

1. Reads `orders.txt` using a buffered byte stream.
2. Parses each line, and classifies it as either a VALID order (exactly 3 fields, and the amount parses as a number greater than `0`) or an INVALID line.
3. Writes every valid order's `orderId` and `amount` to a new file called `valid_orders.txt`, one per line, in the format `orderId: amount`.
4. Writes a summary to a file called `summary.txt`, containing the total number of valid orders, the total number of invalid lines, and the sum of all valid order amounts.

Use proper resource handling (try-with-resources) throughout, and make sure malformed lines are skipped gracefully rather than crashing the whole program.

**Expected `valid_orders.txt` contents (given the sample data above):**

```
101: 499.5
103: 899.0
106: 1200.75
```

**Expected `summary.txt` contents:**

```
Valid orders: 3
Invalid lines: 3
Total revenue: 2599.25
```

### Solution

```java
import java.io.FileInputStream;
import java.io.BufferedInputStream;
import java.io.FileOutputStream;
import java.io.BufferedOutputStream;
import java.io.IOException;

public class ShopEaseOrderProcessor {

    static String readEntireFile(String filename) throws IOException {
        try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream(filename))) {
            byte[] buffer = new byte[4096];
            int totalBytesRead = bis.read(buffer);
            return new String(buffer, 0, totalBytesRead);
        }
    }

    public static void main(String[] args) throws IOException {
        String content = readEntireFile("orders.txt");
        String[] lines = content.split("\n");

        int validCount = 0;
        int invalidCount = 0;
        double totalRevenue = 0.0;

        try (BufferedOutputStream validOut = new BufferedOutputStream(new FileOutputStream("valid_orders.txt"))) {
            for (String line : lines) {
                String trimmedLine = line.trim();
                if (trimmedLine.isEmpty()) {
                    continue;
                }

                String[] parts = trimmedLine.split(",");
                if (parts.length != 3) {
                    invalidCount++;
                    continue;
                }

                String orderId = parts[0];
                double amount;
                try {
                    amount = Double.parseDouble(parts[2]);
                } catch (NumberFormatException e) {
                    invalidCount++;
                    continue;
                }

                if (amount <= 0) {
                    invalidCount++;
                    continue;
                }

                validCount++;
                totalRevenue += amount;
                String outputLine = orderId + ": " + amount + "\n";
                validOut.write(outputLine.getBytes());
            }
        }

        try (BufferedOutputStream summaryOut = new BufferedOutputStream(new FileOutputStream("summary.txt"))) {
            String summary = "Valid orders: " + validCount + "\n"
                    + "Invalid lines: " + invalidCount + "\n"
                    + "Total revenue: " + totalRevenue;
            summaryOut.write(summary.getBytes());
        }

        System.out.println("Processing complete.");
    }
}
```

### Walking Through Why This Works

- **Reading the file:** `readEntireFile` wraps a `FileInputStream` in a `BufferedInputStream`, exactly Section 4's pattern, reads it into a byte array, and converts it to a `String` using the offset-and-count constructor from Chunk 5.
- **Splitting into lines:** `content.split("\n")` (Chunk 5's `String.split`) breaks the whole file into individual order lines.
- **Parsing each line:** `trimmedLine.split(",")` breaks each line into its 3 comma-separated fields. If there aren't exactly 3, straight to `invalidCount++` and `continue` (Chunk 1's `continue`, skip the rest of this iteration).
- **Validating the amount:** `Double.parseDouble` (Chunk 6) is wrapped in its own small `try-catch` for `NumberFormatException` (Chunk 6, Chunk 8). This specifically handles a line that HAS 3 fields but where the third one isn't valid number text at all, a case distinct from the "wrong number of fields" check already handled above.
- **The `amount <= 0` check** catches lines 102 and 105, which parse just fine as numbers, but aren't valid ORDER amounts business-wise.
- **Writing valid orders:** each valid line gets immediately written into `valid_orders.txt` through the buffered output stream, one order at a time, as we go.
- **Writing the summary:** built as one single `String` and written in one shot, once every line has been processed.
- **Resource safety:** every single stream, input and output, is opened inside a try-with-resources block, so every one of them is guaranteed to close properly (and, for the output streams, guaranteed to flush) even if something unexpected goes wrong partway through.

---

## 8. Consolidated Quiz

1. `InputStream` and `OutputStream` are both abstract. Which chunk first taught the general abstract class rule these follow?
   A. Chunk 2  B. Chunk 3  C. Chunk 4  D. Chunk 5

2. `read()` returns an `int`, not a `byte`, specifically to make room for the `-1` end-of-stream signal. What similar "why not just use the smaller type" reasoning appeared back in Chunk 1?
   A. Why `char` is unsigned  B. Why arithmetic on `byte` operands promotes to `int`  C. Why `boolean` isn't numeric  D. None of these are related

3. `BufferedInputStream` can be used anywhere a plain `InputStream` is expected. Which chunk's concept does this directly demonstrate?
   A. Encapsulation, Chunk 2  B. Polymorphism, Chunk 3  C. Static keyword, Chunk 2  D. Generics, Chunk 9

4. `FileNotFoundException` is checked. Which chunk covered the full checked vs. unchecked distinction?
   A. Chunk 6  B. Chunk 7  C. Chunk 8  D. Chunk 9

5. try-with-resources requires the resource to implement `Closeable`. Which earlier chunk's concept does an interface requirement like this rely on?
   A. Chunk 2's encapsulation  B. Chunk 4's interfaces as contracts  C. Chunk 5's `equals`/`hashCode`  D. Chunk 6's wrapper classes

6. In the ShopEase scenario, `Double.parseDouble` can throw `NumberFormatException`. Which chunk first introduced that specific exception?
   A. Chunk 5  B. Chunk 6  C. Chunk 7  D. Chunk 8

7. `content.split("\n")` and `trimmedLine.split(",")` are both used in the scenario solution. Which chunk covered `String.split()`?
   A. Chunk 4  B. Chunk 5  C. Chunk 9  D. Chunk 10

8. Why does forgetting to close a `FileInputStream` connect back to Chunk 7's discussion of resource management?
   A. It doesn't connect at all  B. An unclosed stream holds onto a real system resource (a file handle) it no longer needs, similar in spirit to an object being kept needlessly reachable  C. `FileInputStream` uses the same static counter as Chunk 7's examples  D. Chunk 7 was only about heap memory, nothing else

### Consolidated Quiz Answers

1. **C.** Chunk 4 established that abstract classes can never be instantiated directly, and `InputStream`/`OutputStream` are genuine, real-world JDK examples of exactly that rule.

2. **B.** Same underlying idea as Chunk 1's arithmetic promotion, when a smaller type isn't enough room for everything that needs representing (in that case, the result of arithmetic; here, the extra `-1` sentinel), Java bumps up to a bigger type that has the room.

3. **B.** This is Chunk 3's polymorphism principle in action: any subclass reference (`BufferedInputStream`) can be used wherever its superclass type (`InputStream`) is expected.

4. **C.** Chunk 8 was the deep dive into checked versus unchecked exceptions, and `FileNotFoundException` (extending the checked `IOException`) fits squarely into that checked category.

5. **B.** Chunk 4 covered interfaces as contracts unrelated classes can implement, and `Closeable` is exactly that kind of contract, letting try-with-resources work with any class that fulfills it.

6. **B.** Chunk 6's wrapper class coverage introduced `NumberFormatException` as the exception thrown by parsing methods like `Double.parseDouble` on invalid text.

7. **B.** Chunk 5 covered `String`'s full API, including `split()`, used here to break the file's raw text apart into lines and fields.

8. **B.** While Chunk 7 focused on heap memory and garbage collection specifically, the underlying principle, an unreleased resource sitting around unnecessarily, is the same general idea applied to a different kind of resource, a file handle instead of heap memory.

---

## 9. Revision Summary

| Concept | Key Point | Where It Showed Up |
|---|---|---|
| Stream | Data flowing bit by bit from a source to a destination | The garden hose analogy |
| Byte vs character streams | Byte streams (`InputStream`/`OutputStream`) for any data; character streams (`Reader`/`Writer`) for text specifically | Section 1 |
| InputStream | Abstract root of all byte input classes | `FileInputStream`, `BufferedInputStream` |
| Why read() returns int | Needs room for the `-1` end-of-stream signal, which a `byte` alone can't fit alongside all 256 real byte values | Section 2 |
| FileInputStream | Concrete class, reads bytes from a file, throws checked `FileNotFoundException`/`IOException` | `new FileInputStream("data.txt")` |
| try-with-resources | Guarantees `close()` runs automatically, even on exceptions, for any `Closeable` resource | Every example in this chunk |
| BufferedInputStream | Wraps another `InputStream`, buffers reads in memory to reduce disk access | `new BufferedInputStream(new FileInputStream(...))` |
| FileOutputStream | Writes bytes to a file; overwrites by default, `true` second argument appends instead | `new FileOutputStream("log.txt", true)` |
| BufferedOutputStream | Wraps another `OutputStream`, buffers writes; `close()`/`flush()` push the buffer out to disk | The empty-file gotcha |

**Quick tip for the exam:** anytime a question shows a `BufferedOutputStream` (or any output stream) NOT properly closed or flushed, and asks "what's in the file afterward," the answer is almost always "less than expected, or nothing at all," since buffered data that never gets flushed never actually reaches the disk. And anytime you see `InputStream`/`OutputStream` wrapped inside another stream class, recognize that as the buffering pattern, same original stream capability, just faster underneath.