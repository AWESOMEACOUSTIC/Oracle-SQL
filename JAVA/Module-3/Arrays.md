# Java Competitive Exam Preparation
## Chunk 10: Arrays

Alright, time to go back to basics for a bit, arrays. You've actually been using these since Chunk 1 without us stopping to properly break them down. Now we're doing that properly: what they actually are, the different flavors, and honestly, why Chunk 9 spent so much time convincing you to use `ArrayList` instead half the time.

Same deal as last chunk, keeping things casual and simple, and the practice questions follow that same coding-test style with proper examples and constraints.

---

## Table of Contents

1. What is an Array and Types of Array
2. One Dimensional Array with Lend a Hand
3. Multi-Dimensional Array
4. Advantages and Disadvantages of Arrays
5. Consolidated Quiz
6. Additional Practice Problems
7. Revision Summary

---

## 1. What is an Array and Types of Array

### The Basics

An array is a fixed-size container that holds a bunch of values, all of the SAME type, sitting one after another, and you grab any of them instantly using an index number.

Think of it like a row of lockers. Every locker is the same size, they're all lined up in order, and each one has a number on it. Locker 0, locker 1, locker 2, and so on. Want what's in locker 3? You just go straight there, no searching needed.

```java
int[] scores = new int[5]; // 5 lockers, all currently holding 0 (default value)
scores[0] = 90;
scores[1] = 85;
System.out.println(scores[0]); // 90
```

That "fixed size" part is the big deal here. Once you say `new int[5]`, you get exactly 5 slots, forever. No 6th slot magically appears later.

### Types of Arrays

There are a couple of ways people talk about "types" of arrays:

**By dimension:**
- **One-dimensional array**: a simple straight line of values. `int[] scores`.
- **Multi-dimensional array**: an array of arrays, think rows and columns, like a grid or table. `int[][] grid`. We'll dig into this properly in Section 3.

**By what they hold:**
- **Array of primitives**: `int[]`, `double[]`, `char[]`, and so on. The actual values sit directly in the array's slots.
- **Array of objects**: `String[]`, `Movie[]`, `BankAccount[]`. Here's the interesting bit, the array doesn't hold the actual objects, it holds references to them, exactly the stack-and-heap idea from Chunk 7. Each slot points to an object living somewhere on the heap.

```java
BankAccount[] accounts = new BankAccount[3];
accounts[0] = new BankAccount("Priya", 500);
```

`accounts[0]` isn't the actual account object sitting inside the array, it's a reference to it, same rules as any other object reference.

One more thing worth knowing: the array itself, `scores` or `accounts`, is ALSO an object. It lives on the heap too, and the variable holding it is just a reference, exactly like every other object we've dealt with since Chunk 3.

### Quick Check

If `String[] names = new String[3];`, what's sitting in each slot right after creation, before you assign anything?

`null`. Remember default values from Chunk 1? Arrays get default values too. For reference types like `String`, that default is `null`. For `int[]`, it'd be `0` instead.

---

## 2. One Dimensional Array with Lend a Hand

### Declaring and Creating

There's a few ways to make a 1D array:

```java
int[] a = new int[5];              // 5 slots, all default value 0
int[] b = {10, 20, 30};            // literal shortcut, size decided by how many values you list
int[] c = new int[]{10, 20, 30};   // same thing, just written out more explicitly
```

### The length Property

Here's a classic gotcha: for arrays, it's `arr.length`, no parentheses. It's a property, not a method call.

Compare that to `String`'s `.length()`, WITH parentheses (that one's a method, from Chunk 5), and `List`'s `.size()` (also a method, different name entirely, from Chunk 9). Three different data structures, three different ways to ask "how big are you." Annoying, but it's just something you memorize.

```java
int[] nums = {5, 10, 15};
System.out.println(nums.length); // 3, no parens!
```

### Looping Through a 1D Array

Both loop styles from Chunk 1 work great here.

```java
int[] nums = {5, 10, 15, 20};

for (int i = 0; i < nums.length; i++) {
    System.out.println("Index " + i + ": " + nums[i]);
}

for (int n : nums) {
    System.out.println(n);
}
```

Use the regular `for` loop when you need the index. Use for-each when you just want the values, nice and clean.

### Common Things You'll Do With Arrays

```java
int[] nums = {5, 2, 9, 1, 7};

// find the max
int max = nums[0];
for (int n : nums) {
    if (n > max) {
        max = n;
    }
}

// sum everything
int sum = 0;
for (int n : nums) {
    sum += n;
}

System.out.println("Max: " + max + ", Sum: " + sum);
```

Output: `Max: 9, Sum: 24`

### Going Out of Bounds

Try to access an index that doesn't exist, and you get `ArrayIndexOutOfBoundsException`, straight from Chunk 8's exception coverage.

```java
int[] nums = {1, 2, 3};
System.out.println(nums[5]); // boom
```

That throws `ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 3` at runtime. No compile error, Java has no way to know this is wrong until the program actually runs and tries it.

### Lend a Hand: Quiz

1. What's the correct way to get an array's size?
   A. `arr.length()`  B. `arr.size()`  C. `arr.length`  D. `arr.getSize()`

2. What's stored in `int[] nums = new int[4];` right after creation?
   A. Garbage values  B. `{0, 0, 0, 0}`  C. `null`  D. Nothing, it's empty

3. What happens with `int[] nums = {1,2,3}; System.out.println(nums[3]);`?
   A. Prints `0`  B. Throws `ArrayIndexOutOfBoundsException`  C. Compile error  D. Prints `null`

### Answers

1. **C.** No parentheses, it's a property, not a method. This trips people up constantly, so it's worth burning into memory.

2. **B.** `int` arrays default every slot to `0`, exactly the primitive default value rules from Chunk 1.

3. **B.** Valid indexes here are `0`, `1`, `2`. Index `3` doesn't exist, so it blows up at runtime with `ArrayIndexOutOfBoundsException`.

### Practice Problem

You're given an array containing `n` distinct numbers, each one somewhere in the range `0` to `n` (inclusive), but exactly one number from that range is missing. Find the missing number.

**Example 1:**

```
Input: nums = [3, 0, 1]
Output: 2
Explanation: n = 3 since there are 3 numbers, so the range is [0,3]. 2 is the only number in that range not in nums.
```

**Example 2:**

```
Input: nums = [0, 1]
Output: 2
Explanation: n = 2, range is [0,2]. 2 is missing.
```

**Example 3:**

```
Input: nums = [9, 6, 4, 2, 3, 5, 7, 0, 1]
Output: 8
```

**Constraints:**

* `1 <= nums.length <= 10^4`
* `0 <= nums[i] <= nums.length`
* All values in `nums` are distinct.

### Solution

```java
public class MissingNumber {
    static int findMissing(int[] nums) {
        int n = nums.length;
        int expectedSum = n * (n + 1) / 2; // sum of 0 to n
        int actualSum = 0;
        for (int num : nums) {
            actualSum += num;
        }
        return expectedSum - actualSum;
    }

    public static void main(String[] args) {
        System.out.println(findMissing(new int[]{3, 0, 1}));
        System.out.println(findMissing(new int[]{0, 1}));
        System.out.println(findMissing(new int[]{9, 6, 4, 2, 3, 5, 7, 0, 1}));
    }
}
```

Why this works: if nothing were missing, the numbers `0` through `n` would add up to `n * (n + 1) / 2` (that's just the classic formula for summing consecutive numbers). We compute what the sum SHOULD be, subtract what it ACTUALLY is, and whatever's left over is exactly the missing number. One pass through the array, nice and quick.

---

## 3. Multi-Dimensional Array

### What It Actually Is

A multi-dimensional array is, honestly, just an array of arrays. A 2D array is an array where each slot holds another array. Think rows and columns, like a spreadsheet or a chessboard.

```java
int[][] grid = new int[3][4]; // 3 rows, 4 columns each
grid[0][0] = 1;
grid[1][2] = 99;
System.out.println(grid[1][2]); // 99
```

You can also build one with a literal:

```java
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};
System.out.println(matrix[2][1]); // 8
```

`matrix[2][1]` means "row 2, column 1", and rows and columns both start counting from 0, exactly like regular arrays.

### Looping Through a 2D Array

You need nested loops, from Chunk 1, one for rows, one for columns.

```java
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6}
};

for (int row = 0; row < matrix.length; row++) {
    for (int col = 0; col < matrix[row].length; col++) {
        System.out.print(matrix[row][col] + " ");
    }
    System.out.println();
}
```

Output:
```
1 2 3 
4 5 6 
```

Notice `matrix.length` gives you the number of ROWS, and `matrix[row].length` gives you the number of columns IN THAT SPECIFIC ROW. Which brings us to...

### Jagged Arrays: Java's Rows Don't Have to Match

Here's something kind of unique to Java: a 2D array isn't actually one solid rectangular block underneath. It's really an array where each element happens to be its own separate array. And since each of those inner arrays is independent, they don't all have to be the same length!

```java
int[][] jagged = new int[3][];
jagged[0] = new int[]{1, 2};
jagged[1] = new int[]{3, 4, 5, 6};
jagged[2] = new int[]{7};

for (int[] row : jagged) {
    for (int val : row) {
        System.out.print(val + " ");
    }
    System.out.println();
}
```

Output:
```
1 2 
3 4 5 6 
7 
```

Three rows, three completely different lengths. Totally legal. This is exactly why the nested loop above used `matrix[row].length` instead of just assuming every row is the same size, you genuinely can't assume that in Java.

### 3D Arrays (Briefly)

Yep, you can keep going. `int[][][] cube = new int[2][3][4];` is an array of arrays of arrays. You won't need these often, but just know the same "array of arrays" logic keeps stacking.

### Quick Check

Why doesn't `matrix.length` tell you how many columns a 2D array has?

Because `matrix` itself is just an array of ROWS. `matrix.length` counts how many rows there are. Each individual row is its own separate array with its own `.length`, and since Java allows jagged arrays, those row lengths might not even all match.

### Practice Problem

Given a 2D matrix, return its transpose (basically flip it, rows become columns and columns become rows).

**Example 1:**

```
Input: matrix = [[1,2,3],[4,5,6]]
Output: [[1,4],[2,5],[3,6]]
```

**Example 2:**

```
Input: matrix = [[1,2],[3,4]]
Output: [[1,3],[2,4]]
```

**Constraints:**

* `1 <= matrix.length, matrix[0].length <= 100`
* Every row has the same length (a proper rectangular matrix, not jagged).

### Solution

```java
public class TransposeMatrix {
    static int[][] transpose(int[][] matrix) {
        int rows = matrix.length;
        int cols = matrix[0].length;
        int[][] result = new int[cols][rows]; // dimensions flip!

        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < cols; c++) {
                result[c][r] = matrix[r][c];
            }
        }
        return result;
    }

    public static void main(String[] args) {
        int[][] matrix = {{1, 2, 3}, {4, 5, 6}};
        int[][] result = transpose(matrix);
        for (int[] row : result) {
            for (int val : row) {
                System.out.print(val + " ");
            }
            System.out.println();
        }
    }
}
```

Why this works: the new array's dimensions are swapped, original columns become the new rows. Then it's just one nested loop, and for every `matrix[r][c]`, we drop it into `result[c][r]`, literally swapping the two index positions. That's the whole trick to a transpose.

---

## 4. Advantages and Disadvantages of Arrays

### Advantages

- **Super fast access by index.** `arr[50]` is instant, doesn't matter if the array has 100 items or a million, grabbing any specific index takes the same tiny amount of time.
- **Simple and predictable.** No hidden behavior, no surprises, just a straightforward block of values.
- **Memory efficient.** Arrays don't carry extra overhead the way some collection classes do, they're about as lean as it gets.
- **Great when you know the size upfront.** If you genuinely know you need exactly 7 slots and that'll never change, an array is a perfectly reasonable, lightweight choice.

### Disadvantages

- **Fixed size, forever.** This is the big one. Once created, an array can't grow or shrink. Need more room? You have to make a whole new, bigger array and copy everything over by hand.
- **Inserting or deleting in the middle is a pain.** Want to insert something at index 2? You have to manually shift everything after it over by one, yourself, there's no built in helper for that.
- **Only one type allowed.** An `int[]` can only ever hold `int`s. Fine most of the time, but sometimes limiting.
- **No handy built-in methods.** No `.add()`, no `.contains()`, no `.remove()`. You're writing that logic yourself every time.

### This Is Basically Why Chunk 9 Happened

Remember how Chunk 9 opened with "arrays are great, but they have this one huge annoying limitation"? Yeah, this is that limitation, spelled out properly now. `ArrayList` and friends exist specifically to patch over these exact disadvantages, resizing automatically, giving you `.add()` and `.remove()` for free, handling all the shifting behind the scenes.

So when do you actually pick a plain array over a `List`? Honestly, mostly when you know the size is fixed and won't change, and you want the leanest, fastest option possible, or when you're working with primitives directly and don't want the (very minor) overhead of boxing everything into wrapper objects for a `List<Integer>`.

### Quick Check

If arrays are faster and leaner, why did Chunk 9 push `ArrayList` as the everyday default instead?

Because most real programs don't know their exact size ahead of time, and constantly needing to resize by hand (making a new array, copying everything over) gets old fast. `ArrayList` trades a tiny bit of raw performance for a whole lot of convenience, and for most everyday code, that trade is well worth it.

---

## 5. Consolidated Quiz

Let's tie this back into stuff from earlier chunks too.

1. `BankAccount[] accounts` holds references, not actual objects. Which chunk first explained that object variables always work this way?
   A. Chunk 1  B. Chunk 3  C. Chunk 5  D. Chunk 6

2. `nums.length` has no parentheses, but `someString.length()` does. Which chunk covered `String`'s version?
   A. Chunk 2  B. Chunk 4  C. Chunk 5  D. Chunk 9

3. Accessing `nums[10]` on a 5-element array throws `ArrayIndexOutOfBoundsException`. Which chunk covered exception handling in depth?
   A. Chunk 6  B. Chunk 7  C. Chunk 8  D. Chunk 9

4. A jagged array has rows of different lengths. Does this break the enhanced for-each loop from Chunk 1?
   A. Yes, for-each can't handle jagged arrays  B. No, for-each just visits each row (itself an array), regardless of that row's own length  C. Compile error  D. Only 2D arrays support for-each

5. Why does `int[] arr = new int[5];` immediately have `0`s in every slot, without you setting anything?
   A. Random memory garbage  B. Java default values for primitives, covered back in Chunk 1  C. Compile error without initializing  D. It actually starts empty

6. `Movie[] movies` versus `List<Movie> movies`, which one can you `.add()` a 6th movie to without creating a whole new container?
   A. The array, easily  B. The `List`, since arrays are fixed size and can't grow  C. Both work the same way  D. Neither supports adding

7. If `String[] names` holds `null` by default in every slot, and you call `names[0].length()` without assigning anything first, what happens?
   A. Prints `0`  B. Throws `NullPointerException`, since `names[0]` is `null`, exactly the Chunk 8 style failure  C. Compile error  D. Returns `null`

### Consolidated Quiz Answers

1. **B.** Chunk 3 laid out the whole "objects are accessed through references" idea, and arrays of objects follow that exact same rule, each slot is a reference, not the object itself.

2. **C.** Chunk 5 covered `String.length()` as a method call, parentheses included, which is exactly the contrast worth remembering against an array's parenthesis-free `.length`.

3. **C.** Chunk 8 was the deep dive into exceptions, including exactly this kind of runtime, unchecked exception.

4. **B.** For-each just walks through each element of the outer array, one at a time, and each of those elements happens to be its own row (its own array). It doesn't care whether those rows are all the same length or not.

5. **B.** Straight from Chunk 1: primitive types always get a predictable default value the moment they're created, `0` for `int`.

6. **B.** This is exactly Section 4's whole point. `List` can grow on demand with `.add()`; a plain array is stuck at whatever size you gave it when you made it.

7. **B.** `names[0]` is `null` since nothing was assigned yet, and calling any method on a `null` reference throws `NullPointerException`, exactly the failure pattern first shown back in Chunk 8.

---

## 6. Additional Practice Problems

Here's a proper practice set to close out this chunk, same format as before, mixing 1D and 2D array problems.

### Problem 1: Move Zeroes

Given an array of integers, move all the zeroes to the end of the array while keeping the relative order of the non-zero elements the same. Do this in place, don't make a brand new array to return.

**Example 1:**

```
Input: nums = [0, 1, 0, 3, 12]
Output: [1, 3, 12, 0, 0]
```

**Example 2:**

```
Input: nums = [0, 0, 1]
Output: [1, 0, 0]
```

**Constraints:**

* `1 <= nums.length <= 10^4`
* `-2^31 <= nums[i] <= 2^31 - 1`

**Solution:**

```java
public class MoveZeroes {
    static void moveZeroes(int[] nums) {
        int insertPos = 0;
        for (int num : nums) {
            if (num != 0) {
                nums[insertPos] = num;
                insertPos++;
            }
        }
        while (insertPos < nums.length) {
            nums[insertPos] = 0;
            insertPos++;
        }
    }

    public static void main(String[] args) {
        int[] nums = {0, 1, 0, 3, 12};
        moveZeroes(nums);
        for (int n : nums) {
            System.out.print(n + " ");
        }
    }
}
```

Why this works: first pass, we walk through the array and copy every non-zero value forward, into the earliest open slot (`insertPos`), squishing all the real values to the front in their original order. Second pass, whatever's left over at the end just gets filled with zeroes. Two simple passes, no extra array needed.

---

### Problem 2: Rotate Array

Given an array of integers and a number `k`, rotate the array to the right by `k` steps.

**Example 1:**

```
Input: nums = [1, 2, 3, 4, 5, 6, 7], k = 3
Output: [5, 6, 7, 1, 2, 3, 4]
Explanation: Rotate right by 1: [7,1,2,3,4,5,6]. By 2: [6,7,1,2,3,4,5]. By 3: [5,6,7,1,2,3,4].
```

**Example 2:**

```
Input: nums = [-1, -100, 3, 99], k = 2
Output: [3, 99, -1, -100]
```

**Constraints:**

* `1 <= nums.length <= 10^5`
* `0 <= k <= 10^5`

**Solution:**

```java
public class RotateArray {
    static int[] rotateRight(int[] nums, int k) {
        int n = nums.length;
        k = k % n; // rotating by n is the same as not rotating at all
        int[] result = new int[n];
        for (int i = 0; i < n; i++) {
            result[(i + k) % n] = nums[i];
        }
        return result;
    }

    public static void main(String[] args) {
        int[] result = rotateRight(new int[]{1, 2, 3, 4, 5, 6, 7}, 3);
        for (int n : result) {
            System.out.print(n + " ");
        }
    }
}
```

Why this works: every element just needs to land `k` positions further to the right, wrapping back around to the start once it goes past the end, that's exactly what `(i + k) % n` does for us. `k = k % n` at the start is a nice little safety net too, since rotating a 7-element array by, say, 10, is really the same as rotating it by just 3.

---

### Problem 3: Row With Maximum Sum

Given a 2D matrix of integers, return the index of the row that has the largest sum. If there's a tie, return the smaller index.

**Example 1:**

```
Input: matrix = [[1,2,3],[8,9,1],[4,5,6]]
Output: 1
Explanation: Row sums are 6, 18, and 15. Row index 1 has the biggest sum, 18.
```

**Example 2:**

```
Input: matrix = [[5,5],[3,7],[1,9]]
Output: 0
Explanation: Row sums are 10, 10, and 10. It's a three-way tie, so we return the smallest index, 0.
```

**Constraints:**

* `1 <= matrix.length <= 100`
* `1 <= matrix[i].length <= 100`
* `-1000 <= matrix[i][j] <= 1000`

**Solution:**

```java
public class RowWithMaxSum {
    static int rowWithMaxSum(int[][] matrix) {
        int maxSum = Integer.MIN_VALUE;
        int maxRow = -1;

        for (int r = 0; r < matrix.length; r++) {
            int sum = 0;
            for (int val : matrix[r]) {
                sum += val;
            }
            if (sum > maxSum) {
                maxSum = sum;
                maxRow = r;
            }
        }
        return maxRow;
    }

    public static void main(String[] args) {
        int[][] matrix = {{1, 2, 3}, {8, 9, 1}, {4, 5, 6}};
        System.out.println(rowWithMaxSum(matrix));
    }
}
```

Why this works: for each row, we add up its values with a little inner for-each loop, and keep track of the best one seen so far. Using strictly `>` (not `>=`) when checking for a new best is what naturally keeps the smallest index on a tie, since a later row with an equal sum just won't beat the existing champion.

---

### Problem 4: Spiral Matrix

Given a 2D matrix, return all the elements in spiral order, starting from the top-left corner, going right, then down, then left, then up, and spiraling inward.

**Example 1:**

```
Input: matrix = [[1,2,3],[4,5,6],[7,8,9]]
Output: [1,2,3,6,9,8,7,4,5]
```

**Example 2:**

```
Input: matrix = [[1,2],[3,4]]
Output: [1,2,4,3]
```

**Constraints:**

* `1 <= matrix.length, matrix[0].length <= 10`
* `-100 <= matrix[i][j] <= 100`

**Solution:**

```java
import java.util.List;
import java.util.ArrayList;

public class SpiralMatrix {
    static List<Integer> spiralOrder(int[][] matrix) {
        List<Integer> result = new ArrayList<>();
        int top = 0, bottom = matrix.length - 1;
        int left = 0, right = matrix[0].length - 1;

        while (top <= bottom && left <= right) {
            for (int col = left; col <= right; col++) {
                result.add(matrix[top][col]);
            }
            top++;

            for (int row = top; row <= bottom; row++) {
                result.add(matrix[row][right]);
            }
            right--;

            if (top <= bottom) {
                for (int col = right; col >= left; col--) {
                    result.add(matrix[bottom][col]);
                }
                bottom--;
            }

            if (left <= right) {
                for (int row = bottom; row >= top; row--) {
                    result.add(matrix[row][left]);
                }
                left++;
            }
        }
        return result;
    }

    public static void main(String[] args) {
        int[][] matrix = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
        System.out.println(spiralOrder(matrix));
    }
}
```

Why this works: this one's the trickiest of the bunch, but the idea is actually pretty tidy once you see it. We keep four boundaries, `top`, `bottom`, `left`, `right`, marking the "unvisited" ring of the matrix. Each pass around the loop peels off one full ring: across the top row, down the right column, back across the bottom row, and up the left column, shrinking the boundary inward by one each time a side is finished. The two `if` checks before the bottom and left passes are there so we don't double-count a row or column when the spiral has shrunk down to just a single row or a single column left in the middle.

---

## 7. Revision Summary

| Concept | Key Point | Where It Showed Up |
|---|---|---|
| What an array is | Fixed-size, same-type container, accessed by index | `scores[0]` |
| Types of arrays | 1D (a line), multi-dimensional (a grid), primitives vs. objects | `int[]` vs `BankAccount[]` |
| Arrays of objects | Slots hold references, not the actual objects | `accounts[0]` |
| `.length` | No parentheses, it's a property, not a method | `nums.length` |
| 2D arrays | Really an array of arrays, accessed as `arr[row][col]` | `matrix[2][1]` |
| Jagged arrays | Rows can have different lengths in Java | `jagged[1].length != jagged[2].length` |
| Advantages | Fast index access, simple, lean on memory | Grabbing `arr[50]` instantly |
| Disadvantages | Fixed size, painful inserts/deletes, no built-in helper methods | Exactly why `ArrayList` exists |

**Quick tip for the exam:** whenever you see a question mixing up `.length`, `.length()`, and `.size()`, just remember: arrays use `.length` (no parens), `String` uses `.length()` (method), and `List`/collections use `.size()` (method). Three different data structures, three different habits, that's just something to memorize cold.