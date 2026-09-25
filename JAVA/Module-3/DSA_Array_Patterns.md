# Array DSA Patterns: The Recognition Playbook

## How This Guide Works (Read This First)

Here's the honest truth about DSA problems: there aren't actually thousands of different problems out there. There are maybe 10 to 15 core **patterns**, and almost every array or string problem you'll ever face is just one of those patterns wearing a different costume.

The skill that actually matters isn't "can I solve this exact problem." It's "can I recognize which pattern this problem is secretly asking for." Once you spot the pattern, the code basically writes itself, because you've already got the template in your head.

So that's how this guide is built. For every pattern, you'll get:

- **The Core Idea** — what's actually going on, in plain language, no jargon.
- **When You Should Suspect This Pattern** — the actual signal words and situations that should make a lightbulb go off in your head. This is the part to really burn into memory.
- **The Template** — a generic skeleton you can adapt to basically any problem using that pattern.
- **Two solved problems** — worked all the way through, with an explanation of exactly which signals gave the pattern away.
- **Recognition Recap** — a quick "if you see this, think this" cheat list at the end of each section.

Don't just read the solutions. Every time, before you look at the answer, ask yourself "what clues in this problem are pointing me toward a specific pattern?" That habit is genuinely the whole game.

---

## Table of Contents

1. How This Guide Works
2. Two Pointers
3. Sliding Window
4. Prefix Sum
5. Monotonic Stack
6. Kadane's Algorithm (Maximum Subarray Pattern)
7. Binary Search on Arrays
8. Cyclic Sort
9. Merge Intervals
10. Fast and Slow Pointers
11. The Master Cheat Sheet
12. Mixed Practice: Name That Pattern

---

## 2. Two Pointers

### The Core Idea

Instead of using one index and nested loops (which usually means O(n²)), you use TWO indices moving through the array in a coordinated way, often starting from opposite ends and walking toward each other, sometimes both starting from the same end at different speeds. Because both pointers only ever move forward (never backward), the whole thing finishes in one pass, O(n).

Think of it like two people searching a hallway of lockers from opposite ends, walking toward the middle, instead of one person checking every single pair of lockers against each other.

### When You Should Suspect This Pattern

- The array is **sorted** (or can easily be sorted) and you're looking for a pair or triplet matching some condition (like a target sum).
- You need to compare elements from both ends of the array at the same time.
- Classic "container," "area," or "capacity" style problems.
- You're removing duplicates or rearranging elements in place.
- Checking if something is a palindrome.
- The brute force solution is an obvious nested loop, O(n²), and the problem hints it wants something faster.

### The Template

```java
int left = 0;
int right = arr.length - 1;

while (left < right) {
    // check something using arr[left] and arr[right]
    if (/* condition met */) {
        // do something, maybe return
    } else if (/* need a bigger value */) {
        left++;
    } else {
        right--;
    }
}
```

### Problem 1: Two Sum II — Input Array Is Sorted

Given a 1-indexed array of integers `numbers` that is already sorted in ascending order, find two numbers that add up to a specific `target`. Return the indices of the two numbers (1-indexed) as an array of length 2.

You may assume each input has exactly one solution, and you can't use the same element twice.

**Example 1:**

```
Input: numbers = [2,7,11,15], target = 9
Output: [1,2]
Explanation: numbers[0] + numbers[1] = 2 + 7 = 9, so the indices are 1 and 2.
```

**Example 2:**

```
Input: numbers = [2,3,4], target = 6
Output: [1,3]
Explanation: 2 + 4 = 6.
```

**Constraints:**

* `2 <= numbers.length <= 3 * 10^4`
* `-1000 <= numbers[i] <= 1000`
* `numbers` is sorted in non-decreasing order.
* `-1000 <= target <= 1000`

**Solution:**

```java
public class TwoSumSorted {
    static int[] twoSum(int[] numbers, int target) {
        int left = 0;
        int right = numbers.length - 1;

        while (left < right) {
            int sum = numbers[left] + numbers[right];
            if (sum == target) {
                return new int[]{left + 1, right + 1};
            } else if (sum < target) {
                left++;
            } else {
                right--;
            }
        }
        return new int[]{-1, -1};
    }

    public static void main(String[] args) {
        int[] result = twoSum(new int[]{2, 7, 11, 15}, 9);
        System.out.println(result[0] + ", " + result[1]);
    }
}
```

**Why this is Two Pointers:** the array being sorted is the huge giveaway here. Since it's sorted, if the current sum is too small, we KNOW moving `left` forward (to a bigger number) is the only way to increase the sum. If the sum is too big, moving `right` backward is the only way to shrink it. That certainty is exactly what makes two pointers work, sorted order gives us a clear direction to move in.

---

### Problem 2: Container With Most Water

You're given an array `height` where `height[i]` is the height of a vertical line at position `i`. Find two lines that, together with the x-axis, form a container that holds the most water. Return the maximum amount of water it can hold.

**Example 1:**

```
Input: height = [1,8,6,2,5,4,8,3,7]
Output: 49
Explanation: The lines at index 1 (height 8) and index 8 (height 7) form a container of width 7 and height min(8,7)=7, giving 49.
```

**Example 2:**

```
Input: height = [1,1]
Output: 1
```

**Constraints:**

* `2 <= height.length <= 10^5`
* `0 <= height[i] <= 10^4`

**Solution:**

```java
public class ContainerWithMostWater {
    static int maxArea(int[] height) {
        int left = 0;
        int right = height.length - 1;
        int best = 0;

        while (left < right) {
            int h = Math.min(height[left], height[right]);
            int width = right - left;
            best = Math.max(best, h * width);

            if (height[left] < height[right]) {
                left++;
            } else {
                right--;
            }
        }
        return best;
    }

    public static void main(String[] args) {
        System.out.println(maxArea(new int[]{1, 8, 6, 2, 5, 4, 8, 3, 7}));
    }
}
```

**Why this is Two Pointers:** notice this array is NOT sorted, so this shows two pointers isn't only for sorted arrays. The key insight is: the container's width only shrinks as the pointers move inward, so we should always move the pointer at the SHORTER line (since that's the one limiting our height, and keeping it means every future container is guaranteed smaller). We start with the widest possible container and greedily narrow it, checking every "best possible at this width" along the way.

### Recognition Recap

- Sorted array + "find a pair/triplet that..." → two pointers, from both ends.
- Not sorted, but "maximize/minimize something between two positions" → two pointers might still work if there's a greedy reason to move one side.
- "Remove duplicates in place" or "partition an array" → two pointers, usually both starting from the same end (one slow, one fast).
- If you catch yourself writing a nested loop to check every pair, stop and ask: can I sort this, or is there a direction I can move one pointer with certainty?

---

## 3. Sliding Window

### The Core Idea

A "window" is just a contiguous chunk of the array, marked by a `left` and `right` boundary. Instead of recalculating something from scratch for every possible window (which is slow), you slide the window forward one step at a time, updating your running answer incrementally, add what just entered the window on the right, remove what just left on the left.

There are two flavors:

- **Fixed size window:** the window size `k` is given upfront, and it just slides across the whole array.
- **Variable size window:** the window grows and shrinks depending on whether some condition is currently satisfied.

### When You Should Suspect This Pattern

- The problem mentions a **contiguous** subarray or substring (this word is your biggest clue).
- "Maximum/minimum sum (or average) of a subarray of size `k`."
- "Longest/shortest substring or subarray that satisfies some condition."
- Words like "consecutive," "window," "at most," "at least," combined with a subarray or substring.
- You'd otherwise be tempted to check every possible subarray, which screams "there's a smarter incremental way to do this."

### The Template (Fixed Size)

```java
int windowSum = 0;
for (int i = 0; i < k; i++) {
    windowSum += arr[i];
}
int best = windowSum;

for (int i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k]; // add new, remove the one that fell off
    best = Math.max(best, windowSum);
}
```

### The Template (Variable Size)

```java
int left = 0;
// some tracking state here, e.g. a running sum, or a Set/Map

for (int right = 0; right < arr.length; right++) {
    // add arr[right] into the window / update tracking state

    while (/* window is currently invalid */) {
        // remove arr[left] from the window / update tracking state
        left++;
    }
    // window [left, right] is now valid, update your answer here
}
```

### Problem 1: Maximum Sum Subarray of Size K

Given an array of integers and a number `k`, find the maximum sum of any contiguous subarray of exactly size `k`.

**Example 1:**

```
Input: nums = [2,1,5,1,3,2], k = 3
Output: 9
Explanation: The subarray [5,1,3] has the largest sum, 9.
```

**Example 2:**

```
Input: nums = [2,3,4,1,5], k = 2
Output: 7
Explanation: [3,4] sums to 7.
```

**Constraints:**

* `1 <= k <= nums.length <= 10^5`
* `-10^4 <= nums[i] <= 10^4`

**Solution:**

```java
public class MaxSumSubarraySizeK {
    static int maxSum(int[] nums, int k) {
        int windowSum = 0;
        for (int i = 0; i < k; i++) {
            windowSum += nums[i];
        }
        int best = windowSum;

        for (int i = k; i < nums.length; i++) {
            windowSum += nums[i] - nums[i - k];
            best = Math.max(best, windowSum);
        }
        return best;
    }

    public static void main(String[] args) {
        System.out.println(maxSum(new int[]{2, 1, 5, 1, 3, 2}, 3));
    }
}
```

**Why this is Sliding Window:** "sum of a contiguous subarray of size k" is about as textbook a sliding window signal as it gets. The naive approach recomputes the sum of every window from scratch, O(n*k). Sliding window keeps a running sum and just swaps one element in, one out, at each step, O(n) total.

---

### Problem 2: Longest Substring Without Repeating Characters

Given a string, find the length of the longest substring without any repeating characters.

**Example 1:**

```
Input: s = "abcabcbb"
Output: 3
Explanation: The answer is "abc", length 3.
```

**Example 2:**

```
Input: s = "bbbbb"
Output: 1
Explanation: The answer is "b", length 1.
```

**Example 3:**

```
Input: s = "pwwkew"
Output: 3
Explanation: The answer is "wke", length 3. Note that "pwke" is a subsequence, not a substring.
```

**Constraints:**

* `0 <= s.length <= 5 * 10^4`
* `s` consists of English letters, digits, symbols, and spaces.

**Solution:**

```java
import java.util.HashSet;
import java.util.Set;

public class LongestSubstringNoRepeat {
    static int lengthOfLongestSubstring(String s) {
        Set<Character> window = new HashSet<>();
        int left = 0;
        int best = 0;

        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);

            while (window.contains(c)) {
                window.remove(s.charAt(left));
                left++;
            }
            window.add(c);
            best = Math.max(best, right - left + 1);
        }
        return best;
    }

    public static void main(String[] args) {
        System.out.println(lengthOfLongestSubstring("abcabcbb"));
        System.out.println(lengthOfLongestSubstring("pwwkew"));
    }
}
```

**Why this is Sliding Window:** this is the variable size flavor. "Longest substring that satisfies a condition" (here, no repeats) is exactly the phrasing to watch for. We grow the window by moving `right` forward, and whenever the window becomes invalid (a repeat shows up), we shrink from `left` until it's valid again. The `HashSet` from Chunk 9 is doing the "is this already in my window" check in constant time, which is what keeps the whole thing fast.

### Recognition Recap

- "Contiguous subarray/substring" is the number one keyword to watch for.
- Fixed number `k` given, asking for a sum/average/max over every window of that size → fixed window.
- "Longest/shortest ... that satisfies condition X" → variable window, grow with `right`, shrink with `left` while invalid.
- If your brute force involves checking every possible start and end point of a subarray, that's your cue: there's probably a sliding window way to do it in one pass instead.

---

## 4. Prefix Sum

### The Core Idea

If you need to answer "what's the sum from index `i` to index `j`" over and over again, recalculating that sum every single time by looping through the range is wasteful. Instead, you precompute a **prefix sum array**, where `prefix[i]` holds the sum of everything from the start up to index `i`. Once you have that, any range sum becomes a single subtraction: `sum(i, j) = prefix[j+1] - prefix[i]`.

It's the same idea as knowing your bank balance on day 1 and day 30, if you know both, you instantly know your net change over the month without re-adding every single day's transaction.

### When You Should Suspect This Pattern

- The problem asks about the **sum of a range or subarray**, especially if it's asked **multiple times** with different ranges.
- "How many subarrays have a sum equal to X" style problems.
- Anything where you'd otherwise be tempted to re-sum a chunk of the array repeatedly.
- Words like "range sum," "cumulative," "running total."

### The Template

```java
int[] prefix = new int[arr.length + 1];
for (int i = 0; i < arr.length; i++) {
    prefix[i + 1] = prefix[i] + arr[i];
}
// sum of arr[i..j] inclusive is now:
int rangeSum = prefix[j + 1] - prefix[i];
```

### Problem 1: Range Sum Query — Immutable

Given an integer array `nums`, handle multiple queries of the type: calculate the sum of elements between indices `i` and `j` (inclusive).

**Example 1:**

```
Input: nums = [-2,0,3,-5,2,-1]
Queries: sumRange(0,2), sumRange(2,5), sumRange(0,5)
Output: 1, -1, -3
Explanation: sumRange(0,2) = -2+0+3 = 1. sumRange(2,5) = 3-5+2-1 = -1. sumRange(0,5) = sum of everything = -3.
```

**Constraints:**

* `1 <= nums.length <= 10^4`
* `-10^5 <= nums[i] <= 10^5`
* `0 <= i <= j < nums.length`
* Up to `10^4` calls to `sumRange`.

**Solution:**

```java
public class RangeSumQuery {
    private int[] prefix;

    RangeSumQuery(int[] nums) {
        prefix = new int[nums.length + 1];
        for (int i = 0; i < nums.length; i++) {
            prefix[i + 1] = prefix[i] + nums[i];
        }
    }

    int sumRange(int i, int j) {
        return prefix[j + 1] - prefix[i];
    }

    public static void main(String[] args) {
        RangeSumQuery rsq = new RangeSumQuery(new int[]{-2, 0, 3, -5, 2, -1});
        System.out.println(rsq.sumRange(0, 2));
        System.out.println(rsq.sumRange(2, 5));
        System.out.println(rsq.sumRange(0, 5));
    }
}
```

**Why this is Prefix Sum:** "handle multiple queries about range sums" is the exact signal. Building the `prefix` array once costs O(n), and every query after that is O(1), instead of O(n) per query if you summed the range fresh each time. Notice the constructor does the setup work once (compare to Chunk 2's constructor discussion), and every later call reuses that saved work.

---

### Problem 2: Subarray Sum Equals K

Given an array of integers `nums` and an integer `k`, return how many contiguous subarrays sum up to exactly `k`.

**Example 1:**

```
Input: nums = [1,1,1], k = 2
Output: 2
Explanation: The subarrays [1,1] (first two) and [1,1] (last two) both sum to 2.
```

**Example 2:**

```
Input: nums = [1,2,3], k = 3
Output: 2
Explanation: [1,2] sums to 3, and [3] on its own sums to 3.
```

**Constraints:**

* `1 <= nums.length <= 2 * 10^4`
* `-1000 <= nums[i] <= 1000`
* `-10^7 <= k <= 10^7`

**Solution:**

```java
import java.util.HashMap;
import java.util.Map;

public class SubarraySumEqualsK {
    static int subarraySum(int[] nums, int k) {
        Map<Integer, Integer> prefixCounts = new HashMap<>();
        prefixCounts.put(0, 1); // an empty prefix (sum 0) has occurred once
        int prefixSum = 0;
        int count = 0;

        for (int n : nums) {
            prefixSum += n;
            if (prefixCounts.containsKey(prefixSum - k)) {
                count += prefixCounts.get(prefixSum - k);
            }
            prefixCounts.put(prefixSum, prefixCounts.getOrDefault(prefixSum, 0) + 1);
        }
        return count;
    }

    public static void main(String[] args) {
        System.out.println(subarraySum(new int[]{1, 1, 1}, 2));
        System.out.println(subarraySum(new int[]{1, 2, 3}, 3));
    }
}
```

**Why this is Prefix Sum:** this is the trickier, leveled-up version of the pattern, prefix sum combined with a `HashMap` from Chunk 9. The idea: if the running `prefixSum` at some point minus `k` matches a prefix sum we've already seen before, then everything between those two points must sum to exactly `k`. Instead of storing the whole prefix array, we just keep a running total plus a map counting how many times each prefix sum value has shown up so far. This is a genuinely common combo, prefix sum + hashmap, worth remembering as its own mini-pattern.

### Recognition Recap

- "Sum of a range, asked repeatedly" → build a prefix array once, O(1) per query after that.
- "Count subarrays whose sum equals something" → prefix sum + a hashmap tracking how often each running total has occurred.
- If you're about to write a loop inside a loop just to keep re-summing chunks of the array, that's your cue to precompute a prefix sum instead.

---

## 5. Monotonic Stack

### The Core Idea

A monotonic stack is just a stack (last in, first out) that you keep either strictly increasing or strictly decreasing from bottom to top, by popping off anything that would break that order before pushing the new element. It's the go-to trick whenever you need to find, for every element, the nearest OTHER element to its left or right that's bigger or smaller than it.

Java's built in `java.util.Stack` (or `Deque`, often preferred these days) works fine here. Push adds to the top, `pop()` removes and returns the top, `peek()` looks at the top without removing it.

### When You Should Suspect This Pattern

- "Next greater element" / "next smaller element" / "previous greater" / "previous smaller."
- "How many days until a warmer temperature" style problems.
- "Largest rectangle in a histogram" style problems.
- Anything asking, for every position, "what's the closest position to the left/right where some comparison flips."
- If your brute force would be "for every element, scan forward/backward until you find X," that's a strong hint a monotonic stack can do it in one pass instead.

### The Template

```java
Deque<Integer> stack = new ArrayDeque<>(); // holds INDICES

for (int i = 0; i < arr.length; i++) {
    while (!stack.isEmpty() && arr[stack.peek()] < arr[i]) {
        int idx = stack.pop();
        // arr[i] is the "next greater element" for arr[idx]
        result[idx] = arr[i];
    }
    stack.push(i);
}
// whatever's left in the stack at the end has no next greater element
```

### Problem 1: Next Greater Element

Given an array of integers, for each element, find the next element to its right that is greater than it. If there isn't one, use `-1`.

**Example 1:**

```
Input: nums = [2,1,2,4,3]
Output: [4,2,4,-1,-1]
Explanation: For 2 (index 0), the next greater is 4. For 1, it's 2. For 2 (index 2), it's 4. For 4 and 3, there's nothing bigger after them.
```

**Constraints:**

* `1 <= nums.length <= 10^4`
* `-10^9 <= nums[i] <= 10^9`

**Solution:**

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class NextGreaterElement {
    static int[] nextGreater(int[] nums) {
        int[] result = new int[nums.length];
        java.util.Arrays.fill(result, -1);
        Deque<Integer> stack = new ArrayDeque<>();

        for (int i = 0; i < nums.length; i++) {
            while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
                int idx = stack.pop();
                result[idx] = nums[i];
            }
            stack.push(i);
        }
        return result;
    }

    public static void main(String[] args) {
        int[] result = nextGreater(new int[]{2, 1, 2, 4, 3});
        for (int r : result) {
            System.out.print(r + " ");
        }
    }
}
```

**Why this is Monotonic Stack:** "next greater element" is basically the textbook name for this pattern, it's almost always the giveaway phrase itself. The stack holds indices of numbers still "waiting" for their next greater element. Every time a bigger number shows up, it resolves everyone in the stack that it beats, then joins the stack itself, waiting for someone even bigger.

---

### Problem 2: Daily Temperatures

Given an array of daily temperatures, return an array where each position tells you how many days you'd have to wait until a warmer temperature. If there's no future day that's warmer, put `0`.

**Example 1:**

```
Input: temperatures = [73,74,75,71,69,72,76,73]
Output: [1,1,4,2,1,1,0,0]
Explanation: Day 0 (73) waits 1 day for 74. Day 2 (75) waits 4 days for 76. The last two days never get warmer, so 0.
```

**Constraints:**

* `1 <= temperatures.length <= 10^5`
* `30 <= temperatures[i] <= 100`

**Solution:**

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class DailyTemperatures {
    static int[] dailyTemperatures(int[] temperatures) {
        int[] result = new int[temperatures.length];
        Deque<Integer> stack = new ArrayDeque<>();

        for (int i = 0; i < temperatures.length; i++) {
            while (!stack.isEmpty() && temperatures[stack.peek()] < temperatures[i]) {
                int idx = stack.pop();
                result[idx] = i - idx; // how many days later did we find the warmer day
            }
            stack.push(i);
        }
        return result;
    }

    public static void main(String[] args) {
        int[] result = dailyTemperatures(new int[]{73, 74, 75, 71, 69, 72, 76, 73});
        for (int r : result) {
            System.out.print(r + " ");
        }
    }
}
```

**Why this is Monotonic Stack:** this is genuinely the exact same code shape as Problem 1, just storing "how far away" instead of "what value." That's actually a really good sign you've internalized the pattern: once you see it, tons of "next X" problems all reduce to this same handful of lines with one tiny tweak to what gets stored.

### Recognition Recap

- "Next/previous greater or smaller element" is the single biggest keyword combo for this pattern.
- "How many steps/days until condition X happens" (looking forward) → also monotonic stack.
- The stack always holds things that are "waiting" for their answer, and gets resolved from the top down whenever a qualifying element shows up.
- If brute force means "for each element, scan the rest of the array looking for a bigger/smaller one," that's O(n²), and a monotonic stack usually brings it down to O(n).

---

## 6. Kadane's Algorithm (Maximum Subarray Pattern)

### The Core Idea

This one's specifically for "find the best contiguous subarray" problems, most famously maximum sum. The trick: at every position, you ask one simple question, "should I extend the subarray I was already building, or is it better to just start fresh from here?" You keep a running "best sum ending exactly here" and separately track the overall best you've seen anywhere.

It gets its own named pattern because this exact "extend or restart" decision shows up constantly, even though it's really a tiny, elegant form of dynamic programming.

### When You Should Suspect This Pattern

- "Maximum (or minimum) sum of a contiguous subarray."
- "Maximum contiguous product" (a trickier cousin, since negative numbers can flip signs).
- Anything asking for the best contiguous chunk, where elements can be negative (if everything's positive, the answer is trivially the whole array, so negatives are usually what makes this interesting).

### The Template

```java
int currentSum = arr[0];
int best = arr[0];

for (int i = 1; i < arr.length; i++) {
    currentSum = Math.max(arr[i], currentSum + arr[i]); // extend or restart
    best = Math.max(best, currentSum);
}
```

### Problem 1: Maximum Subarray

Given an integer array, find the contiguous subarray with the largest sum, and return that sum.

**Example 1:**

```
Input: nums = [-2,1,-3,4,-1,2,1,-5,4]
Output: 6
Explanation: [4,-1,2,1] has the largest sum, 6.
```

**Example 2:**

```
Input: nums = [1]
Output: 1
```

**Example 3:**

```
Input: nums = [5,4,-1,7,8]
Output: 23
Explanation: The whole array is the best subarray here.
```

**Constraints:**

* `1 <= nums.length <= 10^5`
* `-10^4 <= nums[i] <= 10^4`

**Solution:**

```java
public class MaximumSubarray {
    static int maxSubArray(int[] nums) {
        int currentSum = nums[0];
        int best = nums[0];

        for (int i = 1; i < nums.length; i++) {
            currentSum = Math.max(nums[i], currentSum + nums[i]);
            best = Math.max(best, currentSum);
        }
        return best;
    }

    public static void main(String[] args) {
        System.out.println(maxSubArray(new int[]{-2, 1, -3, 4, -1, 2, 1, -5, 4}));
    }
}
```

**Why this is Kadane's:** "maximum sum of a contiguous subarray" is basically this pattern's name tag, it's the exact phrase to watch for. At each step, `currentSum` answers "what's the best subarray ending RIGHT HERE," and if that running total ever drops below just starting over at the current element alone, we restart. One single pass, no nested loops needed.

---

### Problem 2: Maximum Product Subarray

Given an integer array, find a contiguous subarray with the largest product, and return that product.

**Example 1:**

```
Input: nums = [2,3,-2,4]
Output: 6
Explanation: [2,3] has the largest product, 6.
```

**Example 2:**

```
Input: nums = [-2,0,-1]
Output: 0
Explanation: The subarray [0] has the largest product, since -2 and -1 alone are negative, and 0 beats them.
```

**Constraints:**

* `1 <= nums.length <= 2 * 10^4`
* `-10 <= nums[i] <= 10`

**Solution:**

```java
public class MaximumProductSubarray {
    static int maxProduct(int[] nums) {
        int maxSoFar = nums[0];
        int minSoFar = nums[0];
        int best = nums[0];

        for (int i = 1; i < nums.length; i++) {
            int n = nums[i];
            if (n < 0) {
                int temp = maxSoFar;
                maxSoFar = minSoFar;
                minSoFar = temp;
            }
            maxSoFar = Math.max(n, maxSoFar * n);
            minSoFar = Math.min(n, minSoFar * n);
            best = Math.max(best, maxSoFar);
        }
        return best;
    }

    public static void main(String[] args) {
        System.out.println(maxProduct(new int[]{2, 3, -2, 4}));
        System.out.println(maxProduct(new int[]{-2, 0, -1}));
    }
}
```

**Why this is Kadane's, with a twist:** same "extend or restart" spirit, but products have a sneaky trap sums don't: a negative number can flip a very small (very negative) product into the new maximum. So we track BOTH the max and min running product at each step, since a swap of sign can turn today's minimum into tomorrow's maximum. This is a good example of the same core pattern needing a genuine extra layer of thinking once the operation changes from addition to multiplication.

### Recognition Recap

- "Maximum/minimum sum of a contiguous subarray" → Kadane's, extend-or-restart, one pass.
- If it's a product instead of a sum, and negatives are allowed → still Kadane's, but track both a running max AND min.
- The giveaway is always "contiguous" plus "best/largest/maximum" combined with a subarray, not the whole array.

---

## 7. Binary Search on Arrays

### The Core Idea

If your data is sorted, or if you can define a clean "yes/no" condition that flips exactly once as you move across a range of possible answers, you can cut your search space in half every single step instead of checking one element at a time. That's O(log n) instead of O(n), a massive difference for large inputs.

There's a classic flavor (search a sorted array for a target) and a leveled up flavor called "binary search on the answer," where you're not searching the array itself, you're searching a RANGE OF POSSIBLE ANSWERS for the smallest or largest one that satisfies some condition.

### When You Should Suspect This Pattern

- The array is sorted, or the problem says so, or it's a rotated version of a sorted array.
- The constraints hint at needing O(log n), like "the array size can be up to 10^7" and a linear scan would be too slow.
- "Find the minimum/maximum value such that some condition holds" (this phrasing is the biggest hint for "binary search on the answer," a genuinely different feeling problem than a normal search, but the same mechanic underneath).

### The Template

```java
int lo = 0, hi = arr.length - 1;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2; // avoids overflow vs (lo+hi)/2
    if (arr[mid] == target) {
        return mid;
    } else if (arr[mid] < target) {
        lo = mid + 1;
    } else {
        hi = mid - 1;
    }
}
return -1; // not found
```

### Problem 1: Binary Search

Given a sorted array of integers and a target value, return the index of the target, or `-1` if it's not present.

**Example 1:**

```
Input: nums = [-1,0,3,5,9,12], target = 9
Output: 4
```

**Example 2:**

```
Input: nums = [-1,0,3,5,9,12], target = 2
Output: -1
```

**Constraints:**

* `1 <= nums.length <= 10^4`
* Array is sorted in ascending order, all values distinct.
* `-10^4 <= nums[i], target <= 10^4`

**Solution:**

```java
public class BinarySearch {
    static int search(int[] nums, int target) {
        int lo = 0, hi = nums.length - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] == target) {
                return mid;
            } else if (nums[mid] < target) {
                lo = mid + 1;
            } else {
                hi = mid - 1;
            }
        }
        return -1;
    }

    public static void main(String[] args) {
        System.out.println(search(new int[]{-1, 0, 3, 5, 9, 12}, 9));
        System.out.println(search(new int[]{-1, 0, 3, 5, 9, 12}, 2));
    }
}
```

**Why this is Binary Search:** "sorted array, find a target" is the most classic signal there is. Every comparison at `mid` cuts the remaining search space in half, so instead of possibly checking all `n` elements, we're done in roughly `log2(n)` steps.

---

### Problem 2: Find Minimum in Rotated Sorted Array

A sorted array has been "rotated" at some unknown pivot (for example `[0,1,2,4,5,6,7]` becomes `[4,5,6,7,0,1,2]`). Given the rotated array, find the minimum element. Assume no duplicate values.

**Example 1:**

```
Input: nums = [3,4,5,1,2]
Output: 1
```

**Example 2:**

```
Input: nums = [4,5,6,7,0,1,2]
Output: 0
```

**Example 3:**

```
Input: nums = [11,13,15,17]
Output: 11
Explanation: The array wasn't actually rotated at all, still counts.
```

**Constraints:**

* `1 <= nums.length <= 5000`
* All values are unique.

**Solution:**

```java
public class FindMinRotated {
    static int findMin(int[] nums) {
        int lo = 0, hi = nums.length - 1;

        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] > nums[hi]) {
                // the minimum must be somewhere to the right of mid
                lo = mid + 1;
            } else {
                // the minimum is at mid, or somewhere to its left
                hi = mid;
            }
        }
        return nums[lo];
    }

    public static void main(String[] args) {
        System.out.println(findMin(new int[]{3, 4, 5, 1, 2}));
        System.out.println(findMin(new int[]{4, 5, 6, 7, 0, 1, 2}));
        System.out.println(findMin(new int[]{11, 13, 15, 17}));
    }
}
```

**Why this is Binary Search:** even though the array isn't fully sorted anymore, there's still a clean rule we can check at each `mid`: if `nums[mid] > nums[hi]`, the rotation point (and the minimum) must be somewhere to the right, otherwise it's at `mid` or to the left. That "clean yes/no rule that lets you eliminate half the search space" is really the true heart of binary search, way more than "the array has to be fully sorted." Whenever you can find a rule like that, binary search is on the table, rotated array or not.

### Recognition Recap

- Sorted array, looking for a specific value → classic binary search.
- Rotated sorted array → binary search still works, you just need a smarter comparison rule at each step.
- "Find the smallest/largest value such that condition X holds true" → binary search on the answer, search over a RANGE of possible answers, not the array itself.
- Big hint in the constraints (huge input size, but an O(log n) or O(n log n) solution is expected) → binary search is probably involved somewhere.

---

## 8. Cyclic Sort

### The Core Idea

This one's a specialist tool, but when it fits, it's beautifully simple. If you're given an array of `n` numbers that are all supposed to fall inside a known small range, like `1` to `n`, then each number actually has ONE correct home: the value `v` belongs at index `v - 1`. You can walk through the array once, and every time a number isn't at its correct home, swap it there. This sorts the whole thing in O(n), without any comparisons at all, and it makes spotting missing or duplicate numbers trivial afterward, just look for indices holding the wrong value.

### When You Should Suspect This Pattern

- The problem says the array holds numbers in the range `1` to `n` (or `0` to `n-1`), where `n` is the array's own length.
- "Find the missing number(s)" or "find the duplicate number(s)" combined with that range restriction.
- The problem wants an O(n) time, O(1) extra space solution, and a sort or a set-based solution feels too heavyweight for what's actually being asked.

### The Template

```java
int i = 0;
while (i < arr.length) {
    int correctIndex = arr[i] - 1; // assuming values are 1..n
    if (arr[i] != arr[correctIndex]) {
        // swap arr[i] and arr[correctIndex]
        int temp = arr[i];
        arr[i] = arr[correctIndex];
        arr[correctIndex] = temp;
    } else {
        i++;
    }
}
// now arr[i] should equal i+1 for every i, if nothing was missing or duplicated
```

### Problem 1: Find All Numbers Disappeared in an Array

Given an array of `n` integers, where each value is between `1` and `n` (inclusive), some numbers appear twice and others don't appear at all. Find all the numbers in the range `1` to `n` that are missing.

**Example 1:**

```
Input: nums = [4,3,2,7,8,2,3,1]
Output: [5,6]
Explanation: The array has 8 elements, so the full range is 1 to 8. 5 and 6 never show up.
```

**Example 2:**

```
Input: nums = [1,1]
Output: [2]
```

**Constraints:**

* `n == nums.length`
* `1 <= n <= 10^5`
* `1 <= nums[i] <= n`

**Solution:**

```java
import java.util.List;
import java.util.ArrayList;

public class FindDisappearedNumbers {
    static List<Integer> findDisappearedNumbers(int[] nums) {
        int i = 0;
        while (i < nums.length) {
            int correctIndex = nums[i] - 1;
            if (nums[i] != nums[correctIndex]) {
                int temp = nums[i];
                nums[i] = nums[correctIndex];
                nums[correctIndex] = temp;
            } else {
                i++;
            }
        }

        List<Integer> missing = new ArrayList<>();
        for (int j = 0; j < nums.length; j++) {
            if (nums[j] != j + 1) {
                missing.add(j + 1);
            }
        }
        return missing;
    }

    public static void main(String[] args) {
        System.out.println(findDisappearedNumbers(new int[]{4, 3, 2, 7, 8, 2, 3, 1}));
        System.out.println(findDisappearedNumbers(new int[]{1, 1}));
    }
}
```

**Why this is Cyclic Sort:** "numbers between 1 and n, array length n" is the exact signature to watch for. Instead of using a `HashSet` from Chunk 9 to track what we've seen (which works fine but costs extra space), cyclic sort places every number in its rightful home directly inside the array itself. Once the placing pass is done, any index NOT holding its own correct value is instantly a giveaway that the "real" occupant of that spot was missing.

### Recognition Recap

- Array of size `n`, values restricted to the range `1` to `n` (or `0` to `n-1`) → cyclic sort is worth considering.
- "Find the missing number" or "find the duplicate" with that specific range restriction → classic cyclic sort territory.
- If you want O(1) extra space instead of reaching for a `HashSet`, cyclic sort is often the trick that gets you there.

---

## 9. Merge Intervals

### The Core Idea

When you're dealing with a bunch of ranges (intervals), like meeting times or date ranges, and you need to combine the overlapping ones, the trick is almost always the same: sort the intervals by their start value first, then walk through them once, merging as you go. Once sorted, you only ever need to compare each interval to the LAST one you've already merged, no need to compare every pair against every other pair.

### When You Should Suspect This Pattern

- The problem literally talks about "intervals," "ranges," "meetings," or "schedules."
- You need to merge, insert, or find overlaps between ranges.
- Each item in the input has a "start" and an "end."

### The Template

```java
Arrays.sort(intervals, (a, b) -> a[0] - b[0]); // sort by start
List<int[]> result = new ArrayList<>();
result.add(intervals[0]);

for (int i = 1; i < intervals.length; i++) {
    int[] last = result.get(result.size() - 1);
    int[] current = intervals[i];
    if (current[0] <= last[1]) {
        last[1] = Math.max(last[1], current[1]); // overlap, merge
    } else {
        result.add(current); // no overlap, it's a new interval
    }
}
```

### Problem 1: Merge Intervals

Given an array of intervals where `intervals[i] = [start, end]`, merge all overlapping intervals and return the result.

**Example 1:**

```
Input: intervals = [[1,3],[2,6],[8,10],[15,18]]
Output: [[1,6],[8,10],[15,18]]
Explanation: [1,3] and [2,6] overlap, so they merge into [1,6].
```

**Example 2:**

```
Input: intervals = [[1,4],[4,5]]
Output: [[1,5]]
Explanation: These are considered overlapping since they touch at 4.
```

**Constraints:**

* `1 <= intervals.length <= 10^4`
* `intervals[i].length == 2`
* `0 <= start <= end <= 10^4`

**Solution:**

```java
import java.util.List;
import java.util.ArrayList;
import java.util.Arrays;

public class MergeIntervals {
    static int[][] merge(int[][] intervals) {
        Arrays.sort(intervals, (a, b) -> a[0] - b[0]);

        List<int[]> result = new ArrayList<>();
        result.add(intervals[0]);

        for (int i = 1; i < intervals.length; i++) {
            int[] last = result.get(result.size() - 1);
            int[] current = intervals[i];

            if (current[0] <= last[1]) {
                last[1] = Math.max(last[1], current[1]);
            } else {
                result.add(current);
            }
        }
        return result.toArray(new int[result.size()][]);
    }

    public static void main(String[] args) {
        int[][] result = merge(new int[][]{{1, 3}, {2, 6}, {8, 10}, {15, 18}});
        for (int[] interval : result) {
            System.out.print(java.util.Arrays.toString(interval) + " ");
        }
    }
}
```

**Why this is Merge Intervals:** "merge overlapping intervals" is literally this pattern's name. Sorting by start position first is what makes the single pass afterward work correctly, once sorted, any interval that overlaps with what you've built so far can ONLY overlap with the most recently merged one, never an earlier one, so you only ever need to check against the last item in your result list.

### Recognition Recap

- Input made up of `[start, end]` pairs, and the question involves overlaps, merging, or scheduling → sort by start, then one pass.
- Always sort first. Almost every interval problem falls apart without that sort, and comes together easily once you have it.
- After merging, you only ever compare a new interval against the LAST merged one, never the whole list.

---

## 10. Fast and Slow Pointers

### The Core Idea

Two pointers again, but this time moving at different SPEEDS through the same sequence, usually one step at a time (slow) and two steps at a time (fast). This is the classic way to detect a cycle: if there's a loop somewhere, the fast pointer will eventually lap the slow one and they'll land on the same spot. If there's no cycle, the fast pointer just reaches the end first.

You'll usually meet this with linked lists, but it applies to arrays too, in a clever way: if an array's values can be treated as "pointers" to other indices (like, the value at index `i` tells you which index to jump to next), you can run the exact same trick directly on the array.

### When You Should Suspect This Pattern

- "Detect a cycle" in a sequence.
- "Find the duplicate number," specifically when the array has `n+1` numbers all in the range `1` to `n`, which guarantees a "cycle" must exist if you treat values as jump targets.
- Anything hinting you should find something "without using extra space," where a `HashSet` would be the obvious-but-not-optimal answer.

### The Template

```java
int slow = start;
int fast = start;

while (true) {
    slow = nums[slow];          // one step
    fast = nums[nums[fast]];    // two steps
    if (slow == fast) {
        break; // they've met, a cycle exists
    }
}

// second phase: find the actual entry point of the cycle
int slow2 = start;
while (slow2 != slow) {
    slow2 = nums[slow2];
    slow = nums[slow];
}
// slow2 (== slow) is now the cycle's starting point
```

### Problem 1: Find the Duplicate Number

Given an array of `n + 1` integers where every value is between `1` and `n` (inclusive), there's guaranteed to be exactly one duplicate number, though it might repeat more than once. Find it, without modifying the array, and using only O(1) extra space.

**Example 1:**

```
Input: nums = [1,3,4,2,2]
Output: 2
```

**Example 2:**

```
Input: nums = [3,1,3,4,2]
Output: 3
```

**Constraints:**

* `1 <= n <= 10^5`
* `nums.length == n + 1`
* `1 <= nums[i] <= n`
* There is exactly one repeated number.

**Solution:**

```java
public class FindTheDuplicate {
    static int findDuplicate(int[] nums) {
        int slow = nums[0];
        int fast = nums[0];

        do {
            slow = nums[slow];
            fast = nums[nums[fast]];
        } while (slow != fast);

        int slow2 = nums[0];
        while (slow2 != slow) {
            slow2 = nums[slow2];
            slow = nums[slow];
        }
        return slow;
    }

    public static void main(String[] args) {
        System.out.println(findDuplicate(new int[]{1, 3, 4, 2, 2}));
        System.out.println(findDuplicate(new int[]{3, 1, 3, 4, 2}));
    }
}
```

**Why this is Fast and Slow Pointers:** here's the clever bit: since every value points to another valid index (values are 1 to n, array has n+1 slots), you can treat the array exactly like a linked list where `nums[i]` tells you "go to index `nums[i]` next." Because there are more numbers than possible values, at least one value must repeat, which forces a cycle to exist in this "jump around the array" sequence. Finding the duplicate is then exactly the same as finding where a linked list cycle begins, no extra space needed beyond a couple of index variables, which is exactly what the "O(1) extra space" constraint is nudging you toward. Worth comparing this against the Cyclic Sort solution to this same style of problem, same family of problems, genuinely different technique depending on exactly what's being asked and what constraints apply.

### Recognition Recap

- "Detect a cycle," or the constraints demand O(1) extra space where a `HashSet` would otherwise be the easy way out → fast and slow pointers.
- On arrays specifically, this trick applies when array values can be treated as "jump to this index next."
- Two phases: first find WHETHER a cycle exists (pointers meet), second find WHERE it starts (reset one pointer to the beginning, move both one step at a time until they meet again).

---

## 11. The Master Cheat Sheet

Here's the whole guide condensed into one table. This is what you glance at 5 minutes before an assessment.

| Pattern | Biggest Signal Words | Typical Time Complexity | Core Trick |
|---|---|---|---|
| Two Pointers | Sorted array, pair/triplet sum, "container," palindrome | O(n) | Two indices moving toward each other, or one fast/one slow from the same end |
| Sliding Window | "Contiguous," "subarray/substring," fixed size `k`, "longest/shortest" | O(n) | Grow/shrink a window instead of recomputing from scratch |
| Prefix Sum | "Range sum," repeated sum queries, "count subarrays with sum =" | O(n) setup, O(1) per query | Precompute cumulative sums, subtract to get any range |
| Monotonic Stack | "Next/previous greater/smaller," "days until warmer" | O(n) | Keep a stack in sorted order, pop when the order breaks |
| Kadane's Algorithm | "Maximum/minimum sum of contiguous subarray" | O(n) | At each step, extend or restart, whichever is bigger |
| Binary Search | Sorted (or rotated sorted) array, "find min/max value such that..." | O(log n) | Cut the search space in half every step |
| Cyclic Sort | Values restricted to range `1..n` or `0..n-1`, "find missing/duplicate" | O(n), O(1) space | Place every number directly at its correct index |
| Merge Intervals | "Intervals," "meetings," "overlapping ranges" | O(n log n) | Sort by start, merge in one pass |
| Fast and Slow Pointers | "Detect a cycle," O(1) space constraint on a duplicate-finding problem | O(n) | One pointer moves 1 step, the other 2, see where they meet |

---

## 12. Mixed Practice: Name That Pattern

Okay, here's the real test. Below are ten problem descriptions with the pattern deliberately NOT named. Before you scroll down to the answer key, try to identify which pattern each one is screaming for, and jot down WHY, which keyword or constraint gave it away.

This is genuinely the most important exercise in this whole guide. Solving problems is good practice, but training yourself to recognize the pattern from the description alone, before you've written a single line of code, is the actual skill that carries over to every future problem you'll ever see.

1. Given a sorted array of integers, determine if there exist two numbers whose sum equals a given target value.

2. Given a string `s` and a string `t`, find the smallest substring of `s` that contains every character of `t`.

3. Given an array of daily stock prices, for each day, find how many days you'd have to wait to see a higher price than that day's.

4. Given an array of integers, find the maximum sum obtainable from any contiguous run of elements within it.

5. You're given an array and asked to answer thousands of queries, each one asking for the sum of elements between two given indices.

6. A sorted array has been rotated an unknown number of times at an unknown pivot. Find whether a given target value exists in it.

7. An array of size `n` contains distinct integers, each one from `0` to `n`, except exactly one number in that range is missing. Find it.

8. You're given a list of employee meeting time ranges for the day. Determine whether any employee has two meetings that overlap.

9. Given a circular array where each value tells you how many steps to jump forward (or backward) from your current position, determine if there's a cycle in the sequence of positions you visit.

10. Given an array of integers, find the length of the longest subarray where all elements are the same, allowing yourself to change at most one element to extend it.

### Answer Key

1. **Two Pointers.** "Sorted array" plus "two numbers that sum to a target" is close to a dead giveaway. Start from both ends, move inward based on whether the current sum is too big or too small.

2. **Sliding Window (variable size).** "Smallest substring that contains X" is exactly the "shrink while still valid, grow while invalid" shape. Grow the window until it contains everything needed, then shrink from the left as much as possible while it's still valid.

3. **Monotonic Stack.** "How many days until a higher price" is the Daily Temperatures problem in disguise. Keep a stack of days still waiting for their answer, resolve them the moment a higher price shows up.

4. **Kadane's Algorithm.** "Maximum sum from any contiguous run" is this pattern's calling card word for word. Extend or restart at every position.

5. **Prefix Sum.** "Thousands of queries, each asking for a range sum" is exactly the setup where precomputing a prefix sum array pays off massively, O(1) per query instead of re-summing each time.

6. **Binary Search (modified for rotation).** Sorted, even if rotated, plus "find whether a target exists," is a search space you can still cut in half each step, you just need the comparison rule adjusted for the rotation.

7. **Cyclic Sort.** "Size `n`, values from `0` to `n`, one missing" is the exact signature. Place each number at its rightful index, then scan for the slot that doesn't match.

8. **Merge Intervals.** "Meeting time ranges" plus "determine if any overlap" is a textbook interval problem. Sort by start time, then a single pass checking each against the last one kept.

9. **Fast and Slow Pointers.** "Circular array, values tell you how many steps to jump, is there a cycle" is precisely the array-as-linked-list cycle detection setup. Two pointers at different speeds, see if they ever meet.

10. **Sliding Window (variable size), with a twist.** This one's a bit sneaky, "longest subarray, allowed one change" is still fundamentally a window that grows while a condition holds (in this case, "at most one element differs from the majority in the window") and shrinks when it's violated. Worth noticing that sliding window shows up in more disguises than any other pattern in this list, so when in doubt about a "longest/shortest contiguous X" problem, it's always worth considering first.

---